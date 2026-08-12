# Lesson 0.8 — MariaDB Migration

This lesson covers **MariaDB database migration from one environment/server to another**, including **Windows ↔ Linux**.

The DBA objective is not simply:

> "Move the database."

It is:

> **Move the database safely, with controlled downtime, complete data, correct configuration, and a tested rollback plan.**

---

# 1. What Is Migration?

Migration means moving a MariaDB workload from one location/version/platform to another.

Examples:

```text
Server A → Server B
MariaDB 10.x → MariaDB 11.x
Windows → Linux
Linux → Windows
On-prem → Cloud
VM → VM
Old hardware → New hardware
```

---

# 2. Migration Architecture

Typical migration:

```text
             SOURCE
        ┌──────────────┐
        │   MariaDB    │
        │   Server A   │
        └──────┬───────┘
               │
               │ Migration
               ▼
        ┌──────────────┐
        │   MariaDB    │
        │   Server B   │
        └──────┬───────┘
               │
               ▼
             TARGET
```

---

# 3. Migration Is More Than Data

A common beginner mistake is:

```text
Database
   ↓
Copy tables
   ↓
DONE
```

A real migration includes:

```text
Data
+
Schema
+
Users
+
Privileges
+
Configuration
+
Stored objects
+
Events
+
Jobs
+
Replication
+
Application configuration
+
Monitoring
+
Backup
```

---

# 4. Migration Types

There are several approaches.

### Logical migration

```text
Source
  ↓
mariadb-dump
  ↓
SQL
  ↓
Target
```

### Physical migration

```text
Source
  ↓
mariadb-backup
  ↓
Backup
  ↓
Target
```

### Replication-based migration

```text
SOURCE
   │
   │ replication
   ▼
TARGET
   │
   ▼
Catch up
   │
   ▼
Cutover
```

Each method has different advantages.

---

# 5. Choosing a Migration Method

| Requirement             | Common Approach               |
| ----------------------- | ----------------------------- |
| Small DB                | Logical dump                  |
| Moderate DB             | Logical or physical           |
| Very large DB           | Physical/replication-oriented |
| Minimal downtime        | Replication-based             |
| Object-level migration  | Logical                       |
| Cross-platform          | Logical often convenient      |
| Full server replacement | Physical can be useful        |

This is a design decision, not a one-command operation.

---

# 6. Migration Planning

Before touching anything:

```text id="x7a8h4"
Source
  ↓
Inventory
  ↓
Dependencies
  ↓
Target design
  ↓
Migration method
  ↓
Testing
  ↓
Cutover
  ↓
Validation
```

---

# 7. Source Inventory

Collect:

```sql id="uh1q2s"
SELECT VERSION();
```

Also determine:

```text id="zly3c1"
Database size
Database names
Tables
Storage engines
Character sets
Users
Privileges
Stored procedures
Functions
Triggers
Events
Replication
Binary logging
```

---

# 8. Check Database Sizes

You can estimate database sizes using:

```sql id="xaxl9m"
SELECT
    table_schema AS database_name,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2)
        AS size_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_mb DESC;
```

This helps determine:

```text id="u5ly7h"
Migration time
Backup size
Network requirements
Target storage
Downtime
```

---

# 9. Check Storage Engines

Run:

```sql id="3s44bw"
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE
FROM information_schema.tables
WHERE TABLE_SCHEMA NOT IN
('information_schema','mysql','performance_schema','sys');
```

You want to know what engines your databases use.

For many modern MariaDB workloads, you'll encounter:

```text id="i4e1kw"
InnoDB
```

but don't assume every environment is identical.

---

# 10. Check Character Sets

Run:

```sql id="m9uh2k"
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA;
```

Character-set differences can create nasty migration problems.

For example:

```text id="e7j4i4"
Source
utf8mb4

Target
different character set/collation
```

Potentially:

```text id="3n1jbx"
Sorting differences
Comparison differences
Encoding problems
Application errors
```

---

# 11. Check Tables

```sql id="52n8fr"
SELECT
    TABLE_SCHEMA,
    COUNT(*) AS table_count
FROM information_schema.tables
WHERE TABLE_TYPE = 'BASE TABLE'
GROUP BY TABLE_SCHEMA;
```

Compare source and target after migration.

---

# 12. Check Users

Remember from Lesson 5:

```sql id="6pjhja"
SELECT User, Host
FROM mysql.user
ORDER BY User, Host;
```

Users are part of migration planning.

You don't want:

```text id="7c6e7c"
Data migrated successfully
       ↓
Application cannot login
```

---

# 13. Check Grants

For important users:

```sql id="r1s6b4"
SHOW GRANTS FOR 'appuser'@'localhost';
```

Record the privileges.

After migration, compare them.

---

# 14. Stored Objects

Inventory:

```text id="q9cxly"
Tables
Views
Triggers
Procedures
Functions
Events
```

For example:

```sql id="ymq2xq"
SHOW FULL TABLES;
```

Triggers:

```sql id="b71z4s"
SHOW TRIGGERS;
```

Events:

```sql id="2kw8hl"
SHOW EVENTS;
```

Stored procedures/functions can be reviewed through the relevant `information_schema` metadata and `SHOW CREATE` statements.

---

# 15. Migration Dependencies

Your MariaDB database may be connected to:

```text id="4m6j3h"
Application servers
Reporting tools
ETL systems
Monitoring
Backup systems
Replication
BI tools
Batch jobs
APIs
```

Map these dependencies before migration.

---

# 16. Migration Runbook

A production migration should have a written runbook.

Example:

```text id="s3p8f0"
1. Pre-check source
2. Pre-check target
3. Backup source
4. Perform migration
5. Validate target
6. Synchronize changes
7. Freeze writes
8. Final sync
9. Cut over application
10. Validate
11. Monitor
12. Rollback if required
```

---

# 17. Logical Migration

The simplest approach:

```text id="g4n2my"
SOURCE
   │
   ▼
mariadb-dump
   │
   ▼
SQL file
   │
   ▼
TARGET
   │
   ▼
mariadb
```

Example:

```bash id="f08o8p"
mariadb-dump -u root -p salesdb > salesdb.sql
```

Transfer:

```text id="x1ps5v"
salesdb.sql
```

to the target server.

Then:

```bash id="5r5m58"
mariadb -u root -p salesdb < salesdb.sql
```

---

# 18. Why Logical Migration Is Attractive

It can be relatively straightforward:

```text id="wl8y3c"
Export
  ↓
Transfer
  ↓
Import
  ↓
Validate
```

It is also useful when moving between different operating systems.

For example:

```text id="8p7p50"
Windows
  ↓
Logical dump
  ↓
Linux
```

or:

```text id="j9u7nd"
Linux
  ↓
Logical dump
  ↓
Windows
```

---

# 19. Logical Migration Limitations

For very large databases:

```text id="a6l1sy"
Dump
  ↓
Transfer
  ↓
Import
```

can take a long time.

Suppose:

```text id="d4l6vc"
Database = 4 TB
```

You need to consider:

```text id="2q2bdu"
Dump time
Transfer time
Import time
Downtime
Storage
CPU
Network
```

---

# 20. Migration Downtime

Suppose:

```text id="7f4wgn"
Application writes continuously
```

If you simply dump once:

```text id="2ncb2e"
10:00
Dump starts

10:30
New data

11:00
Dump finishes
```

The target may not contain changes made after the relevant snapshot/dump point.

That's why large/minimal-downtime migrations need more sophisticated planning.

---

# 21. Minimal-Downtime Migration

A common strategy:

```text id="o3ap32"
SOURCE
   │
   │ Initial copy
   ▼
TARGET
   │
   │ Keep synchronized
   ▼
Target catches up
   │
   ▼
Short write freeze
   │
   ▼
Final synchronization
   │
   ▼
Application cutover
```

Replication can be used as part of this strategy when the supported topology/version combination permits it.

---

# 22. Replication-Based Migration

Conceptually:

```text id="q7n7q8"
             SOURCE
                │
                │ binlog
                ▼
             TARGET
                │
                ▼
          Replication Lag
                │
                ▼
             Catch Up
                │
                ▼
             Cutover
```

The target receives ongoing changes until it is ready.

---

# 23. Cutover

Cutover means switching the application from:

```text id="3nkn1h"
OLD SERVER
```

to:

```text id="c4wq3q"
NEW SERVER
```

Example:

```text id="r5e3g7"
Application
     │
     ▼
OLD DB
```

becomes:

```text id="6o9yjr"
Application
     │
     ▼
NEW DB
```

---

# 24. DNS Cutover

Some environments use a DNS name:

```text id="5y17q8"
db-prod.company.internal
```

Application:

```text id="kj5xj1"
db-prod.company.internal
```

DNS points to:

```text id="1e7bpp"
Old DB
```

After migration:

```text id="ydl26r"
db-prod.company.internal
        ↓
New DB
```

But DNS changes are not instant magic. TTL, client caching, connection pools, and application behavior must be considered.

---

# 25. Connection String Cutover

Another approach is changing:

```text id="i70ov5"
DB_HOST=old-db
```

to:

```text id="j2f23v"
DB_HOST=new-db
```

Then restart/reload the application as appropriate.

This is often straightforward but requires application coordination.

---

# 26. Windows → Linux Migration

Example:

```text id="e4r9t6"
Windows MariaDB
       │
       ▼
Logical backup
       │
       ▼
Transfer
       │
       ▼
Linux MariaDB
       │
       ▼
Restore
```

This is often attractive because logical SQL is less tied to the operating system's filesystem layout.

---

# 27. Linux → Windows Migration

Same principle:

```text id="7dxhpa"
Linux
  │
  ▼
Logical backup
  │
  ▼
Transfer
  │
  ▼
Windows
  │
  ▼
Restore
```

Don't simply copy a Linux MariaDB data directory onto Windows and expect it to work.

---

# 28. Why Data Directory Copying Is Risky

You may see advice like:

```text id="v6b2za"
"Just copy the MariaDB data directory."
```

Don't do this casually.

The data directory depends on:

```text id="h6uk0j"
MariaDB version
Storage engine
Configuration
Filesystem behavior
Permissions
Platform
Clean shutdown/backup consistency
```

For cross-platform migration, logical migration is often easier to reason about.

---

# 29. Physical Migration

A physical migration may use:

```text id="0z5s2s"
mariadb-backup
```

Conceptually:

```text id="8m2z7r"
Source
  ↓
mariadb-backup
  ↓
Physical backup
  ↓
Target
  ↓
Prepare/restore
  ↓
MariaDB
```

This can be highly effective for large environments.

---

# 30. Physical Migration Consideration

Physical backups are generally more sensitive to:

```text id="2k1p99"
MariaDB version
Platform
Storage layout
Configuration
Backup consistency
```

Therefore:

> **Use physical migration according to the supported MariaDB procedure for your specific versions and platform.**

Don't treat a physical backup as a generic ZIP file.

---

# 31. Users Are Not Automatically "Just There"

When migrating database data, make sure you account for:

```text id="l8y5b6"
Users
Hosts
Authentication
Privileges
Roles, where used
```

Your application could fail even though:

```text id="0z2g6b"
All tables migrated successfully.
```

---

# 32. Application Account Example

Source:

```text id="pmz7j3"
'appuser'@'10.10.10.50'
```

Target needs an appropriate account configuration.

Then verify:

```sql id="fnc9jt"
SHOW GRANTS FOR 'appuser'@'10.10.10.50';
```

---

# 33. Configuration Migration

Don't blindly copy:

```text id="qfmy60"
my.cnf
my.ini
```

Instead:

```text id="7n8kq7"
Old configuration
      ↓
Review
      ↓
Target requirements
      ↓
Create target configuration
```

Important areas:

```text id="z9ehxb"
datadir
port
bind-address
character set
collation
buffer settings
logging
binary logging
replication
TLS
authentication
```

---

# 34. Validate Target Configuration

Before application cutover:

```text id="j3aq9s"
Target MariaDB
       ↓
Configuration validated
       ↓
Service starts
       ↓
Database accessible
       ↓
Security validated
```

---

# 35. Data Validation

Never trust:

```text id="m5d1q3"
"Import completed."
```

Compare source and target.

For example:

```sql id="a2y2zj"
SELECT COUNT(*) FROM customers;
```

Run it on both systems.

Compare:

```text id="j65e6e"
SOURCE:  1,000,000
TARGET:  1,000,000
```

Do this for critical tables.

---

# 36. Check Important Tables

Don't necessarily compare every table manually.

Identify:

```text id="a2zqce"
Critical tables
Large tables
Transactional tables
Reference tables
Audit tables
```

Then validate:

```text id="q2y5r0"
Row counts
Sample records
Checksums/hashes where appropriate
Application queries
```

---

# 37. Application Validation

This is the most important final test.

Test:

```text id="7a7s1x"
Login
Read
Insert
Update
Delete
Reports
Transactions
Background jobs
APIs
```

The database can look perfect while the application is broken.

---

# 38. Migration Rollback

Before cutover:

```text id="44t9g4"
OLD DB
  │
  │ still available
  ▼
NEW DB
```

After cutover, don't immediately destroy the old system.

Keep it available according to your rollback policy.

If:

```text id="q3l5vr"
New DB fails
```

you may need:

```text id="0dbd1d"
Application
    ↓
Old DB
```

But rollback becomes complicated if writes have occurred on the new database.

Therefore:

> **Define rollback before cutover.**

---

# 39. Rollback Is Harder After Writes

Suppose:

```text id="h5ryo7"
10:00
Cutover to NEW

10:05
New transactions

10:10
Problem discovered
```

You cannot simply point the application back to OLD without accounting for those new transactions.

This is why migration rollback planning must address **data divergence**.

---

# 40. Migration Validation Checklist

```text id="h0c1u6"
☐ MariaDB version
☐ OS/platform
☐ Databases
☐ Tables
☐ Row counts
☐ Critical records
☐ Character sets
☐ Collations
☐ Users
☐ Grants
☐ Procedures
☐ Functions
☐ Triggers
☐ Events
☐ Configuration
☐ TLS
☐ Firewall
☐ Application connectivity
☐ Backup
☐ Monitoring
☐ Replication
```

---

# 41. Migration on Windows

Before migration:

```powershell id="d8s9hm"
Get-Service MariaDB
```

Check:

```text id="l8r4a7"
MariaDB version
Service
my.ini
Data directory
Disk space
Firewall
Backup location
```

After migration:

```powershell id="u6i1gp"
Get-Service MariaDB
```

Then:

```sql id="c6uj8r"
SELECT VERSION();
```

and test application connectivity.

---

# 42. Migration on Linux

Before migration:

```bash id="8w4lgu"
systemctl status mariadb
```

Check:

```bash id="p4njcb"
mariadb --version
```

and:

```bash id="f2e8x9"
df -h
```

After migration:

```bash id="m2y1n7"
systemctl status mariadb
```

Then:

```bash id="8r2c3u"
mariadb-admin ping
```

And validate the database/application.

---

# 43. Migration Timing

Estimate:

```text id="nh7d5w"
Backup time
+
Transfer time
+
Restore time
+
Validation time
+
Cutover time
```

For example:

```text id="f4t5la"
Backup:       2 hours
Transfer:     1 hour
Restore:      2 hours
Validation:   30 min
Cutover:      15 min
```

Don't tell management:

> "Migration takes 15 minutes."

because the final cutover may be 15 minutes while the entire migration takes several hours.

---

# 44. Migration Performance

Consider:

```text id="qbyk74"
Network bandwidth
Disk throughput
CPU
Compression
Encryption
Database size
Number of tables
Indexes
Concurrent workload
```

For a large migration:

```text id="xkgzv4"
Network = bottleneck
```

can dominate the migration duration.

---

# 45. Migration Security

Protect migration data.

A database dump can contain:

```text id="qmyyqe"
Customer information
Passwords/hashes
Financial information
Personal information
Business data
```

Therefore:

```text id="0n4q3y"
Encrypt transfer
Restrict access
Secure backup
Delete temporary dumps securely
Audit access
```

---

# 46. Common Migration Mistakes

### ❌ Copy data directory between platforms

Risky.

### ❌ Forget users

Application can't authenticate.

### ❌ Forget grants

Application gets access denied.

### ❌ Forget events

Scheduled jobs stop.

### ❌ Forget triggers

Business logic can break.

### ❌ Forget character sets

Data can become corrupted/misinterpreted.

### ❌ No application testing

Production incident.

### ❌ No rollback plan

Very dangerous.

### ❌ Destroy old server immediately

Removes an important fallback.

---

# 47. Production Migration Runbook

```text id="9kjd0k"
PHASE 1 — DISCOVERY
☐ Source version
☐ Target version
☐ Database inventory
☐ Dependencies

PHASE 2 — PREPARATION
☐ Target installed
☐ Configuration prepared
☐ Storage ready
☐ Network ready
☐ Security ready

PHASE 3 — TEST
☐ Test migration
☐ Data validation
☐ Application testing
☐ Performance testing

PHASE 4 — MIGRATION
☐ Initial copy
☐ Synchronization
☐ Final validation

PHASE 5 — CUTOVER
☐ Freeze writes
☐ Final sync
☐ Switch application
☐ Validate

PHASE 6 — POST-CUTOVER
☐ Monitor
☐ Backup
☐ Replication
☐ Application
☐ Keep rollback option

PHASE 7 — CLOSE
☐ Document
☐ Confirm success
☐ Retire old server according to policy
```

---

# 48. DBA Challenge

### Q1

What is the difference between logical and physical migration?

### Q2

Why is logical migration often convenient for Windows ↔ Linux?

### Q3

Why shouldn't you blindly copy the MariaDB data directory across platforms?

### Q4

Name five things besides table data that should be considered during migration.

### Q5

Why is rollback more difficult after new transactions start on the target?

### Q6

Why should you validate character sets and collations?

### Q7

Why should the old server not immediately be destroyed after cutover?

### Q8

What is the purpose of replication-based migration?

---

# Answers

### Q1

Logical migration moves database structure/data through a logical representation such as SQL, while physical migration works with the database's physical storage representation.

### Q2

SQL/logical data is less dependent on the underlying operating system filesystem structure.

### Q3

Because the data directory is dependent on MariaDB version, storage engines, configuration, filesystem behavior, permissions, and platform considerations.

### Q4

For example:

```text
Users
Privileges
Triggers
Procedures
Events
Configuration
Character sets
Replication
TLS
Monitoring
```

### Q5

Because the target now contains changes that the old source may not have.

### Q6

Because differences can cause encoding, comparison, sorting, and application behavior problems.

### Q7

It can provide a rollback option if a serious problem is discovered after cutover.

### Q8

It allows the target to continuously receive changes from the source, potentially reducing the final downtime/cutover window.

---

# ⭐ Lesson 8 Golden Rules

> **1. Migration is a project, not a copy command.**

> **2. Inventory everything before moving anything.**

> **3. Data is only one part of a migration.**

> **4. Users and privileges matter.**

> **5. Windows ↔ Linux migrations are often well suited to logical migration.**

> **6. Always validate the target.**

> **7. Define rollback before cutover.**

> **8. Don't destroy the old environment immediately.**

> **9. For large databases, design around transfer time, restore time, and downtime requirements.**

> **10. The migration isn't successful until the application works.**

---

# Your Migration Mental Model

```text id="d3p0c1"
                    SOURCE
                       │
                       ▼
                  INVENTORY
                       │
                       ▼
                  BACKUP
                       │
                       ▼
               MIGRATION METHOD
              ┌────────┼────────┐
              ▼        ▼        ▼
           Logical  Physical  Replication
              │        │        │
              └────────┼────────┘
                       ▼
                     TARGET
                       │
                       ▼
                  VALIDATION
                       │
              ┌────────┴────────┐
              ▼                 ▼
          DATABASE           APPLICATION
              │                 │
              └────────┬────────┘
                       ▼
                    CUTOVER
                       │
                       ▼
                   MONITOR
                       │
                       ▼
                  RETIRE OLD
```

## Next — Lesson 0.9: MariaDB Monitoring, Performance Tuning & Troubleshooting

We'll cover:

```text
CPU / Memory / Disk
Connections
Threads
Slow queries
Locks
Deadlocks
InnoDB
Indexes
Buffer pool
Query analysis
EXPLAIN
Performance Schema
Error logs
MariaDB logs
Linux troubleshooting
Windows troubleshooting
Replication troubleshooting
"Database is slow" troubleshooting
Production incident workflow
```

This will be the **hands-on troubleshooting lesson** where we learn how a DBA actually investigates a MariaDB performance incident.
