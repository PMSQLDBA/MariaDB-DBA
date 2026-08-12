# Lesson 9 — MariaDB Maintenance & Troubleshooting

**Goal:** Learn how to troubleshoot MariaDB when something goes wrong—without making the problem worse.

This is where DBA skills become practical.

> **Golden rule:** When production is broken, don't guess. Preserve evidence, identify the failure, make the smallest safe change, and verify recovery.

---

# 1. The DBA Troubleshooting Model

When MariaDB has a problem:

```text
Problem
   ↓
Observe
   ↓
Collect evidence
   ↓
Identify root cause
   ↓
Take controlled action
   ↓
Verify
   ↓
Document
```

Never start with:

```text
"Let's restart MariaDB."
```

A restart sometimes fixes symptoms while hiding the actual cause.

---

# 2. The Most Important Evidence: Error Log

MariaDB's error log is one of your first places to look.

You may find:

* Startup failures
* Shutdown information
* InnoDB errors
* Corruption messages
* Authentication problems
* Replication errors
* Out-of-memory situations
* Configuration errors

On Linux, the location depends on your installation.

You can inspect relevant variables:

```sql
SHOW VARIABLES LIKE 'log_error';
```

If MariaDB isn't running, you'll need to inspect the configured log location from the operating system/configuration.

---

# 3. Read the Error Log

On a Linux system, you might use:

```bash
tail -100 /path/to/mariadb-error.log
```

Or follow new messages:

```bash
tail -f /path/to/mariadb-error.log
```

The exact path varies by installation.

Look for messages such as:

```text
ERROR
Warning
InnoDB
Crash
Corrupt
Can't open
Permission denied
Out of memory
```

Don't just look at the last line.

Look for the **first meaningful error**.

---

# 4. MariaDB Won't Start

Imagine:

```bash
systemctl status mariadb
```

shows failure.

Don't immediately reinstall MariaDB.

First:

```bash
systemctl status mariadb
```

Then:

```bash
journalctl -u mariadb
```

And inspect the MariaDB error log.

Conceptually:

```text
MariaDB won't start
       ↓
systemd status
       ↓
journal
       ↓
MariaDB error log
       ↓
configuration / permissions / storage / InnoDB
```

---

# 5. Common Startup Problems

A failed startup can be caused by:

### Configuration error

For example, invalid configuration syntax.

### Permissions

MariaDB can't access its files.

### Port conflict

Another process is using the configured port.

### Disk full

MariaDB cannot create/write required files.

### InnoDB recovery

MariaDB is recovering after an unexpected shutdown.

### Corruption

Database files may be damaged.

### Resource exhaustion

Memory or other system resources may be insufficient.

---

# 6. Check the Disk First

One of the simplest and most useful commands:

```bash
df -h
```

Suppose:

```text
Filesystem      Size   Used   Avail
/dev/sda1       100G   100G   0G
```

You have a serious problem.

A database server with no free space can behave unpredictably.

Also inspect inode availability:

```bash
df -i
```

A filesystem can have free bytes but still run out of inodes.

---

# 7. Find What Consumed Disk Space

Use:

```bash
du -sh /path/to/directory/*
```

Be careful on production systems with huge directory trees.

You might discover:

```text
binary logs
backups
temporary files
application logs
old files
```

Don't blindly delete database files.

Especially:

> **Never manually delete MariaDB data files because you need disk space.**

Identify what is safe to remove first.

---

# 8. Binary Logs Can Consume Significant Space

Binary logs are important for:

* Replication
* Point-in-time recovery

But they also require retention management.

Check binary-log settings:

```sql
SHOW BINARY LOGS;
```

and:

```sql
SHOW VARIABLES LIKE 'log_bin%';
```

You should have a deliberate retention policy.

Do **not** randomly delete binary-log files from the filesystem.

Use MariaDB's appropriate binary-log expiration/purging mechanisms according to your version and recovery requirements.

---

# 9. Never Delete Database Files Manually

This deserves its own rule.

Don't do things like:

```bash
rm -rf /var/lib/mysql/*
```

or manually remove random `.ibd`, redo, undo, or system files.

You can turn:

```text
Database problem
```

into:

```text
Permanent data loss
```

Instead, determine exactly what the file is and what MariaDB expects.

---

# 10. MariaDB Is Running but Queries Are Slow

Now suppose:

```text
MariaDB = running
```

but:

```text
Application = slow
```

Use:

```sql
SHOW FULL PROCESSLIST;
```

Then investigate:

```text
Slow queries?
Locks?
Deadlocks?
CPU?
Memory?
Disk?
Connections?
Replication lag?
```

Don't confuse:

```text
MariaDB process exists
```

with:

```text
MariaDB is healthy
```

---

# 11. Lock Troubleshooting

Suppose:

```text
Query A
   ↓
holds lock

Query B
   ↓
waiting
```

Find active sessions:

```sql
SHOW FULL PROCESSLIST;
```

Then investigate InnoDB:

```sql
SHOW ENGINE INNODB STATUS\G
```

Depending on MariaDB version, Performance Schema and information-schema views may also provide useful lock/transaction information.

---

# 12. Long Transactions

Long transactions can cause serious problems.

You should investigate transactions that remain open unexpectedly.

The exact diagnostic queries vary by MariaDB version, but InnoDB transaction information can help identify:

* Transaction age
* Active transactions
* Lock activity
* Waiting transactions

A transaction lasting:

```text
5 minutes
```

may be legitimate.

A transaction lasting:

```text
3 days
```

should immediately raise questions.

---

# 13. Deadlocks

Deadlocks are not necessarily a sign that MariaDB is broken.

They can happen in correctly functioning transactional systems.

Example:

```text
Transaction A:
lock Row 1
wait for Row 2

Transaction B:
lock Row 2
wait for Row 1
```

MariaDB detects the cycle and rolls back one transaction.

Your application should generally be designed to handle deadlock-related transaction failures by retrying appropriately.

The DBA should investigate why the transaction pattern creates frequent deadlocks.

---

# 14. Don't Just Disable Safety Mechanisms

When troubleshooting, avoid "fixes" like:

```text
Disable foreign keys
Disable transactions
Disable integrity checks
Turn off logging
Run everything as root
```

These may make a symptom disappear while creating a larger problem.

A DBA should preserve data integrity while troubleshooting.

---

# 15. Database Corruption

Corruption is one of the most serious situations.

Possible symptoms:

```text
Can't read page
InnoDB errors
Checksum errors
Unexpected crashes
Table access failures
```

First rule:

> **Don't start randomly modifying or deleting database files.**

Second:

> **Preserve the current state and examine the evidence.**

Third:

> **Use backups/recovery procedures when appropriate.**

---

# 16. `CHECK TABLE`

For some MariaDB table/storage-engine situations, you can use:

```sql
CHECK TABLE table_name;
```

For example:

```sql
CHECK TABLE employees;
```

But don't assume this is a universal corruption detector for all InnoDB problems.

InnoDB's architecture means that corruption diagnosis often requires examining:

* Error logs
* InnoDB status
* Storage/system errors
* Backups
* Recovery procedures

---

# 17. `REPAIR TABLE`

Be careful here.

`REPAIR TABLE` is associated primarily with storage engines such as MyISAM and is **not a general repair command for InnoDB corruption**.

Don't blindly run:

```sql
REPAIR TABLE employees;
```

because someone on the internet said "repair the table."

First identify the storage engine:

```sql
SHOW TABLE STATUS
WHERE Name = 'employees';
```

or:

```sql
SHOW CREATE TABLE employees;
```

---

# 18. InnoDB Recovery

InnoDB is designed to recover after crashes.

After an unexpected shutdown, MariaDB may perform crash recovery during startup.

Conceptually:

```text
Unexpected shutdown
       ↓
MariaDB starts
       ↓
InnoDB crash recovery
       ↓
Recover committed state
       ↓
Rollback incomplete transactions
       ↓
Database becomes available
```

The duration depends on workload, storage, transaction activity, and database size.

Don't kill the process just because recovery takes time.

---

# 19. Emergency Recovery Mode

MariaDB/InnoDB has recovery mechanisms that can sometimes help extract data from a badly damaged instance.

But emergency recovery settings can have serious consequences.

They should be considered **last-resort recovery tools**, not normal maintenance.

The correct approach is:

```text
Corruption
   ↓
Preserve original state
   ↓
Use copy/snapshot where possible
   ↓
Attempt controlled recovery
   ↓
Extract/recover data
   ↓
Restore/rebuild clean database
```

Don't experiment with destructive recovery settings on the only copy of your production database.

---

# 20. Replication Troubleshooting

From Lesson 6, start with:

```sql
SHOW SLAVE STATUS\G
```

or, depending on your MariaDB version:

```sql
SHOW REPLICA STATUS\G
```

Check:

```text
Slave_IO_Running
Slave_SQL_Running
Last_IO_Error
Last_SQL_Error
Seconds_Behind_Master
```

Then classify the problem.

```text
Receiver problem?
       ↓
Network / authentication / primary

Applier problem?
       ↓
SQL / schema / data / transaction
```

---

# 21. Replication Error Example

Suppose:

```text
Slave_SQL_Running: No
```

and:

```text
Last_SQL_Error:
Duplicate entry ...
```

Don't immediately skip the transaction.

Ask:

```text
Why does the replica already contain this row?
```

Possible explanations:

* Someone manually changed the replica
* Data diverged
* Application wrote where it shouldn't
* Previous recovery operation was incorrect
* Replication was interrupted and state became inconsistent

The error is a **symptom**.

Find the cause.

---

# 22. When to Rebuild a Replica

Sometimes the safest solution is:

```text
Replica is inconsistent
        ↓
Don't trust it
        ↓
Take fresh consistent backup/snapshot
        ↓
Rebuild replica
        ↓
Start replication again
        ↓
Verify
```

This is often safer than accumulating manual fixes.

A replica's main value is that it should be **trusted**.

---

# 23. Authentication Problems

Suppose the application suddenly reports:

```text
Access denied for user
```

Check:

```sql
SELECT User, Host
FROM mysql.user;
```

Then inspect the account:

```sql
SHOW GRANTS FOR 'appuser'@'localhost';
```

or the appropriate host pattern.

Remember:

```text
'user'@'localhost'
```

and:

```text
'user'@'%'
```

are not the same account definition.

MariaDB authentication includes both username and host matching.

---

# 24. Don't Use Root for Applications

Bad:

```text
Application
    ↓
root
```

Better:

```text
Application
    ↓
Dedicated application account
    ↓
Only required privileges
```

For example, an application might need:

```text
SELECT
INSERT
UPDATE
DELETE
```

but not:

```text
DROP
GRANT OPTION
CREATE USER
```

Follow least privilege.

---

# 25. Configuration Changes

Before changing MariaDB configuration:

```text
Current configuration
       ↓
Understand parameter
       ↓
Check syntax
       ↓
Change one thing
       ↓
Restart/reload if necessary
       ↓
Verify
       ↓
Monitor
```

Don't change 20 parameters simultaneously.

If performance improves—or gets worse—you won't know which change caused it.

---

# 26. Configuration Backup

Before modifying important configuration:

```bash
cp /path/to/my.cnf /path/to/my.cnf.backup
```

Use an appropriately protected location and naming scheme in real environments.

Then document:

```text
Date
Parameter
Old value
New value
Reason
Expected result
Actual result
```

This is professional change management.

---

# 27. Maintenance Tasks

Regular MariaDB administration may include:

```text
Backups
↓
Restore testing
↓
Monitoring
↓
Log review
↓
Disk-space checks
↓
Security updates
↓
Replication checks
↓
Capacity planning
↓
Index/query review
↓
Configuration review
↓
HA testing
```

Maintenance isn't just:

```text
"Restart MariaDB every Sunday."
```

---

# 28. MariaDB Upgrade Strategy

Never treat a production database upgrade like:

```bash
apt upgrade
```

and hope.

A safer process is:

```text
Read release notes
       ↓
Check compatibility
       ↓
Test backup/restore
       ↓
Test application
       ↓
Test upgrade in staging
       ↓
Plan downtime/failover
       ↓
Perform production upgrade
       ↓
Verify
       ↓
Monitor
```

Version-specific behavior matters.

---

# 29. Pre-Upgrade Checklist

Before an important MariaDB upgrade:

```text
☐ Verified backups
☐ Successful test restore
☐ Application compatibility checked
☐ MariaDB compatibility checked
☐ Replication topology reviewed
☐ Disk space checked
☐ Rollback/recovery plan prepared
☐ Maintenance window scheduled
☐ Monitoring active
☐ Configuration backed up
```

Never upgrade a database when your only recovery plan is:

> "Hopefully it works."

---

# 30. The Incident Runbook

A DBA should maintain written procedures.

Example:

```text
INCIDENT: MariaDB unavailable

1. Confirm outage
2. Check application
3. Check host
4. Check MariaDB service
5. Check error log
6. Check disk
7. Check memory/CPU
8. Identify failure
9. Apply approved recovery procedure
10. Verify database
11. Verify application
12. Check replication/HA
13. Document incident
```

This reduces panic during real incidents.

---

# 31. Lesson 9 Lab

Create a troubleshooting checklist and practice the commands below on your lab server.

### Service

```bash
systemctl status mariadb
```

### Logs

```bash
journalctl -u mariadb
```

### Disk

```bash
df -h
df -i
```

### MariaDB processes

```sql
SHOW FULL PROCESSLIST;
```

### InnoDB

```sql
SHOW ENGINE INNODB STATUS\G
```

### Database engines

```sql
SHOW ENGINES;
```

### Table definition

```sql
SHOW CREATE TABLE employees;
```

### Table status

```sql
SHOW TABLE STATUS
WHERE Name = 'employees';
```

### Replication

```sql
SHOW SLAVE STATUS\G
```

or:

```sql
SHOW REPLICA STATUS\G
```

depending on your version.

---

# 32. Troubleshooting Scenarios

Try solving these **before looking at the answers**.

### Scenario A

MariaDB suddenly won't start.

You discover:

```text
Disk usage = 100%
```

**What should you investigate first?**

---

### Scenario B

Application queries are extremely slow.

MariaDB CPU is low, but one query has been running for 45 minutes.

**What do you check?**

---

### Scenario C

Replica reports:

```text
Slave_IO_Running = Yes
Slave_SQL_Running = No
```

**Which side should you investigate first?**

---

### Scenario D

MariaDB starts after a crash and logs show InnoDB recovery.

**Should you immediately kill MariaDB?**

---

### Scenario E

A replica has repeated duplicate-key replication errors.

**Should you simply skip every error?**

---

# Answers

### A

Investigate filesystem usage and determine what consumed the space.

Do **not** blindly delete MariaDB files.

### B

Investigate:

```text
SHOW FULL PROCESSLIST;
EXPLAIN;
locks;
slow query log;
```

and system resource usage.

### C

The receiver is working but the SQL/applier isn't.

Investigate:

```text
Last_SQL_Error
```

and the underlying cause.

### D

No.

Allow legitimate crash recovery to proceed unless there is evidence of a different problem.

### E

No.

Repeated errors may indicate data divergence. Determine the root cause and consider rebuilding the replica if necessary.

---

# Lesson 9 — DBA Knowledge Check

You should now understand:

**Error logs**
→ First-class troubleshooting evidence.

**`systemctl` / `journalctl`**
→ Useful for service-level failures on Linux systems.

**Disk-full problems**
→ Can cause severe database failures.

**Locks/deadlocks**
→ Can cause queries to wait or transactions to roll back.

**InnoDB recovery**
→ Normal part of recovering from certain crashes.

**Corruption**
→ Requires controlled recovery, not random file manipulation.

**Replication errors**
→ Diagnose the actual error before attempting repair.

**Least privilege**
→ Applications should use dedicated accounts.

**Configuration management**
→ Change one thing at a time and document it.

**Upgrade planning**
→ Test recovery and application compatibility before production changes.

---

# The Golden Troubleshooting Rule

> **Preserve evidence before attempting a fix.**

Think like this:

```text
             DON'T PANIC
                  │
                  ▼
          What exactly failed?
                  │
                  ▼
           What does the log say?
                  │
                  ▼
          What does the system say?
                  │
                  ▼
        What does MariaDB say?
                  │
                  ▼
           What is the root cause?
                  │
                  ▼
             Fix safely
                  │
                  ▼
              Verify
```

That is the mindset of a real DBA.

---

# Next: Lesson 10 — MariaDB DBA Security + Production Administration

This is the **final lesson**. We'll put everything together: **users and privileges, authentication, least privilege, TLS, auditing, secure configuration, backups, monitoring, HA, disaster recovery, daily/weekly/monthly DBA checklists, and a complete production DBA workflow.**
