# Lesson 5 — MariaDB Backup & Restore

**Goal:** Learn how a MariaDB DBA protects data, creates backups, restores databases, and verifies that backups are actually usable.

The most important principle of this lesson:

> **A backup that has never been successfully restored is not a proven backup.**

---

# 1. Why Backups Matter

Imagine this:

```text
Monday
   ↓
Database working normally

Tuesday
   ↓
Someone accidentally deletes data

Wednesday
   ↓
Disk failure

Thursday
   ↓
Application discovers missing records
```

Without a usable backup, the DBA may have no way to recover.

A proper backup strategy protects against:

* Accidental `DELETE`
* Accidental `DROP`
* Hardware failure
* Filesystem corruption
* Human mistakes
* Malware/ransomware
* Application bugs
* Database crashes
* Disaster

---

# 2. Backup vs. High Availability

These are **not the same thing**.

### Backup

Allows you to recover previous data.

```text
Database
   ↓
Backup
   ↓
Stored separately
```

### High Availability

Keeps service available when a server fails.

```text
Primary
   │
   ▼
Replica
```

You generally want both.

A replica is **not a substitute for backups**.

Why?

If someone accidentally runs:

```sql
DROP TABLE customers;
```

the destructive change may propagate to the replica.

So:

```text
Replication ≠ Backup
```

---

# 3. Logical vs. Physical Backups

There are two broad categories.

## Logical backup

Exports database objects/data into a logical representation, commonly SQL.

Example:

```bash
mariadb-dump
```

Conceptually:

```text
MariaDB
   ↓
SQL statements/data
   ↓
backup.sql
```

Advantages:

* Portable
* Easy to inspect
* Useful for migrations
* Can restore selected databases/tables depending on how the dump was made

Disadvantages:

* Restore can be slower for very large databases
* Requires recreating objects and loading data

---

## Physical backup

Copies database files/data structures using a backup method appropriate to the MariaDB storage environment.

Conceptually:

```text
MariaDB data
     ↓
Physical backup
     ↓
Backup storage
```

Physical backup approaches can be much more suitable for large databases and faster recovery scenarios, but require more careful handling of consistency, storage engines, versions, and backup tooling.

---

# 4. `mariadb-dump`

One of the most important tools for MariaDB administrators is:

```bash
mariadb-dump
```

You may also encounter the older name:

```bash
mysqldump
```

For MariaDB environments, use the MariaDB-supported tooling appropriate to your installed version.

---

# 5. Back Up One Database

Suppose we have:

```text
company
```

Create a logical backup:

```bash
mariadb-dump -u root -p company > company_backup.sql
```

You'll be prompted for the password.

The resulting file:

```text
company_backup.sql
```

contains SQL needed to reconstruct the dumped database objects/data, subject to the dump options used.

---

# 6. Inspect the Backup

You can inspect the beginning of the file:

```bash
head company_backup.sql
```

You might see statements resembling:

```sql
CREATE TABLE ...
```

and:

```sql
INSERT INTO ...
```

The exact output depends on the dump options and MariaDB version.

This is one advantage of a logical backup:

**You can inspect it.**

---

# 7. Back Up Multiple Databases

You can specify multiple databases:

```bash
mariadb-dump -u root -p \
    company inventory > databases_backup.sql
```

For all databases:

```bash
mariadb-dump -u root -p --all-databases > all_databases.sql
```

Be careful with all-database dumps because they can include system databases and account-related objects depending on the MariaDB version/options.

For production, understand exactly what your dump contains before relying on it for recovery.

---

# 8. Back Up a Specific Table

You can dump specific tables:

```bash
mariadb-dump -u root -p company employees > employees_backup.sql
```

This can be useful when you need a logical backup of only selected objects.

---

# 9. Important Dump Options

One commonly used option for transactional InnoDB databases is:

```bash
--single-transaction
```

Example:

```bash
mariadb-dump -u root -p \
    --single-transaction \
    company > company_backup.sql
```

For suitable transactional workloads, this can provide a consistent logical snapshot without locking the entire database in the same way some older dump approaches might.

But understand the limitation:

**`--single-transaction` is primarily useful for transactional tables such as InnoDB.**

It doesn't magically make every possible table/storage engine consistent.

---

# 10. Include Stored Objects

Depending on your database, you may need to account for objects such as:

* Stored procedures
* Functions
* Events
* Triggers

Your backup strategy must ensure that required objects are included.

For example, dump options can include routines and events when appropriate.

Don't assume:

> "The tables are backed up, therefore the entire application database is backed up."

A production database can contain much more than tables.

---

# 11. Restore a Database

Suppose you have:

```text
company_backup.sql
```

Create the target database if necessary:

```sql
CREATE DATABASE company;
```

Then restore:

```bash
mariadb -u root -p company < company_backup.sql
```

Conceptually:

```text
company_backup.sql
       ↓
    MariaDB
       ↓
company database
```

---

# 12. Restoring an All-Database Dump

If the dump contains database creation statements and the appropriate objects, you can restore it through the MariaDB client:

```bash
mariadb -u root -p < all_databases.sql
```

Again, **understand what the dump contains before restoring it**.

Never test restoration against production unless you have deliberately planned the operation.

---

# 13. Backup Verification

This is where many inexperienced administrators fail.

They do:

```text
Backup completed successfully
       ↓
"Great!"
```

That's not enough.

A better process is:

```text
Backup
  ↓
Verify file exists
  ↓
Verify size/integrity
  ↓
Restore into isolated environment
  ↓
Check tables
  ↓
Check row/data integrity
  ↓
Test application
```

A backup job saying "success" doesn't prove that you can recover.

---

# 14. The Test Restore

Suppose:

```text
Production
    ↓
backup.sql
    ↓
Test MariaDB server
    ↓
Restore
    ↓
Validation
```

Check databases:

```sql
SHOW DATABASES;
```

Check tables:

```sql
USE company;

SHOW TABLES;
```

Check important data:

```sql
SELECT COUNT(*) FROM employees;
```

You can also compare expected counts and application-level checks.

---

# 15. Backup Retention

Don't keep only one backup.

Imagine:

```text
Monday backup
Tuesday backup
Wednesday backup
Thursday backup
Friday backup
```

If Friday's backup is corrupt, you may still have Thursday's.

Retention policies depend on business requirements.

For example:

```text
Daily backups → 30 days
Weekly backups → 3 months
Monthly backups → 1 year
```

Those are merely examples, not universal requirements.

The correct policy depends on:

* Business requirements
* Compliance
* Storage cost
* Recovery objectives
* Data change rate

---

# 16. RPO and RTO

These two concepts are essential for DBAs.

## RPO — Recovery Point Objective

**How much data can the business afford to lose?**

Example:

```text
RPO = 15 minutes
```

means the recovery strategy should aim to limit data loss to roughly 15 minutes or less, depending on the design.

## RTO — Recovery Time Objective

**How quickly must the service be restored?**

Example:

```text
RTO = 1 hour
```

means the organization targets restoring service within about one hour.

Think:

```text
RPO → How much data can we lose?

RTO → How much time can we be down?
```

---

# 17. Binary Logs and Point-in-Time Recovery

Now we reach an advanced DBA concept.

Suppose you have a full backup from:

```text
Monday 00:00
```

Someone accidentally deletes important data at:

```text
Wednesday 14:32
```

If binary logging has been properly configured and retained, you may be able to restore the Monday backup and then replay changes up to a desired point before the accidental operation.

Conceptually:

```text
Full Backup
Monday
   │
   ▼
Restore
   │
   ▼
Binary Logs
Monday → Tuesday → Wednesday
   │
   ▼
Replay changes
   │
   ▼
Desired recovery point
```

This is called:

**Point-in-Time Recovery (PITR).**

---

# 18. Why Binary Logs Matter

Check:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

If enabled:

```text
MariaDB
   │
   ├── Data
   │
   └── Binary Log
          │
          ├── Events
          ├── Replication
          └── Recovery support
```

Binary logs become especially important in:

* Replication
* Point-in-time recovery
* Disaster recovery

We'll go much deeper into them in **Lesson 6 — Replication**.

---

# 19. Backup Storage

Don't store your only backup on the same disk as the database.

Bad:

```text
Server
├── MariaDB
└── backup.sql
```

If the server's disk fails:

```text
MariaDB ❌
backup.sql ❌
```

Better:

```text
Database Server
      │
      ▼
Backup System
      │
      ├── Local backup
      ├── Separate storage
      └── Off-site/cloud copy
```

For serious disaster recovery, you want backup copies separated from the primary failure domain.

---

# 20. The 3-2-1 Backup Principle

A commonly used backup strategy is:

```text
3 copies of data
2 different types/media/storage locations
1 copy off-site
```

The exact implementation varies, but the principle is valuable:

> **Don't let one disaster destroy both the production database and every backup.**

---

# 21. Encryption

Backups can contain your entire database.

Therefore:

```text
Production security
       +
Backup security
```

are both important.

If backups contain sensitive data, consider:

* Encryption at rest
* Encryption in transit
* Access controls
* Secure backup credentials
* Key management
* Audit logging

A perfectly secure production database with an unprotected backup is still a serious security problem.

---

# 22. Backup Monitoring

A DBA shouldn't simply create a cron job and forget about it.

A production backup system should provide evidence such as:

```text
Backup started
      ↓
Backup completed
      ↓
Expected file/object exists
      ↓
Expected size/checks passed
      ↓
Retention applied
      ↓
Restore test periodically performed
      ↓
Alert if anything fails
```

You want to know about backup failures **before** you need the backup.

---

# 23. A Practical Backup Script Concept

A basic conceptual command might look like:

```bash
mariadb-dump -u backupuser -p \
    --single-transaction \
    company > /backup/company_$(date +%F).sql
```

This creates a dated filename such as:

```text
company_2026-08-12.sql
```

But don't put a production password directly into scripts like:

```bash
-pMyPassword
```

That's poor credential hygiene.

Production automation should use an appropriate secure authentication mechanism and restricted permissions.

---

# 24. Lesson 5 Lab

### Step 1 — Create a practice database

```sql
CREATE DATABASE backup_lab;

USE backup_lab;

CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150)
);
```

Insert test data:

```sql
INSERT INTO customers (name, email)
VALUES
('Alice', 'alice@example.com'),
('Bob', 'bob@example.com'),
('Carol', 'carol@example.com');
```

Check:

```sql
SELECT * FROM customers;
```

### Step 2 — Dump the database

From the operating system:

```bash
mariadb-dump -u root -p \
    --single-transaction \
    backup_lab > backup_lab.sql
```

### Step 3 — Create a test restore database

```sql
CREATE DATABASE backup_restore_lab;
```

### Step 4 — Restore

```bash
mariadb -u root -p \
    backup_restore_lab < backup_lab.sql
```

### Step 5 — Validate

```sql
USE backup_restore_lab;

SHOW TABLES;

SELECT * FROM customers;

SELECT COUNT(*) FROM customers;
```

You should see the three test customers.

---

# 25. Disaster-Recovery Exercise

Imagine this scenario:

```text
Monday 01:00
Full backup completed.

Tuesday 12:00
Another backup completed.

Wednesday 14:00
Application accidentally deletes 50,000 customer records.

Wednesday 14:05
DBA discovers the mistake.
```

Ask yourself:

### Question 1

Which backup would you start with?

### Question 2

Could you recover to a point immediately before 14:00?

### Question 3

What information would you need?

### Question 4

Where are the binary logs?

### Question 5

Have you actually tested the restoration procedure?

These are real DBA questions.

---

# Lesson 5 — DBA Knowledge Check

You should now understand:

**Logical backup**
→ `mariadb-dump`

**Physical backup**
→ Backup of database storage using appropriate physical-backup tooling

**Restore**
→ Load backup into MariaDB

**Backup verification**
→ Actually perform test restores

**RPO**
→ Maximum acceptable data-loss window

**RTO**
→ Maximum acceptable recovery time

**Binary logs**
→ Important for replication and point-in-time recovery

**PITR**
→ Restore a base backup and replay changes to a desired point

**Backup retention**
→ Keep appropriate historical recovery points

**3-2-1 principle**
→ Multiple copies across separate failure domains

---

## The Golden DBA Rule

> **Backup is not the objective. Recovery is the objective.**

A production DBA should always be able to answer:

**"If the database disappears right now, exactly how do I recover it?"**

If the answer is unclear, the backup strategy isn't finished.

### Next: Lesson 6 — MariaDB Replication

We'll build the concepts behind **Primary → Replica replication, binary logs, GTID, replication users, replica configuration, monitoring replication, replication lag, and troubleshooting broken replication.**
