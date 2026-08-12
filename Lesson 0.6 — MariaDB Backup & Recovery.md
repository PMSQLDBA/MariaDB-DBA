# Lesson 0.6 — MariaDB Backup & Recovery

This is one of the **most important DBA lessons**.

A DBA is not judged by whether backups exist.

A DBA is judged by whether the database can be **successfully recovered when something goes wrong**.

---

# 1. Backup vs Recovery

These are different concepts.

### Backup

```text
Database
   ↓
Copy of data
   ↓
Backup storage
```

### Recovery

```text
Failure
   ↓
Find backup
   ↓
Restore
   ↓
Apply required changes
   ↓
Validate
   ↓
Application returns
```

Your real objective is:

> **Recover the database within the required RTO with acceptable data loss according to the RPO.**

---

# 2. RPO

**RPO = Recovery Point Objective**

It answers:

> **How much data can we afford to lose?**

Example:

```text
RPO = 15 minutes
```

means the business may accept losing approximately 15 minutes of recent changes, depending on the recovery design.

---

# 3. RTO

**RTO = Recovery Time Objective**

It answers:

> **How quickly must the service be restored?**

Example:

```text
RTO = 1 hour
```

means the recovery process should target restoring service within approximately one hour.

---

# 4. RPO + RTO

Think:

```text
                 Business Requirement
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
             RPO                   RTO
       How much data?         How much time?
              │                     │
              ▼                     ▼
          Backup design        Recovery design
```

Backup strategy should start with these requirements.

---

# 5. MariaDB Backup Types

The major concepts are:

```text
Logical backup
Physical backup
Full backup
Incremental backup
Binary logs
```

---

# 6. Logical Backup

A logical backup represents database objects and data in a logical format.

For example:

```text
Database
   ↓
SQL statements
   ↓
Backup file
```

Typical tool:

```text
mariadb-dump
```

Older environments may refer to:

```text
mysqldump
```

---

# 7. Logical Backup Example

A simple example:

```bash
mariadb-dump -u root -p trainingdb > trainingdb.sql
```

Conceptually:

```text
trainingdb
     ↓
mariadb-dump
     ↓
trainingdb.sql
```

The output contains SQL that can be used to recreate the database objects/data.

---

# 8. Why Logical Backups Are Useful

They are useful for:

```text
Database migration
Development environments
Small/medium databases
Object-level recovery
Cross-environment transfers
Selective restoration
```

For example:

```text
Production
    ↓
Logical backup
    ↓
Development
```

---

# 9. Logical Backup Disadvantages

For a very large database:

```text
Database = 5 TB
```

a logical dump may take considerable time.

It can involve:

```text
Read data
 ↓
Convert to logical representation
 ↓
Write output
 ↓
Transfer/storage
```

Restoration can also take significant time.

Therefore:

> **Logical backup is not automatically the best backup method for every production database.**

---

# 10. Physical Backup

A physical backup works closer to the database's physical storage representation.

MariaDB provides:

```text
mariadb-backup
```

Conceptually:

```text
MariaDB Data
     ↓
mariadb-backup
     ↓
Physical Backup
```

This is particularly useful for larger production environments.

---

# 11. Logical vs Physical

| Feature           | Logical        | Physical         |
| ----------------- | -------------- | ---------------- |
| Tool              | `mariadb-dump` | `mariadb-backup` |
| Representation    | SQL/logical    | Physical         |
| Small DB          | Excellent      | Good             |
| Large DB          | Can be slower  | Often preferable |
| Selective objects | Easier         | Less convenient  |
| Migration         | Very useful    | Useful           |
| Recovery speed    | Can be slower  | Often faster     |
| Complexity        | Lower          | Higher           |

The right choice depends on the environment.

---

# 12. Full Backup

A full backup captures the required database state at a point in time.

Conceptually:

```text
Sunday
  │
  ▼
FULL BACKUP
```

Then:

```text
Monday
Tuesday
Wednesday
```

changes occur.

---

# 13. Incremental Backup

An incremental backup captures changes since a previous relevant backup.

Conceptually:

```text
Sunday
   │
   ▼
Full Backup
   │
   ├── Monday → Incremental
   │
   ├── Tuesday → Incremental
   │
   └── Wednesday → Incremental
```

Recovery may require:

```text
Full backup
    +
Incrementals
    +
Required logs
```

---

# 14. Binary Logs

Binary logs are extremely important for recovery and replication.

Think:

```text
Database
   │
   ▼
Changes
   │
   ▼
Binary Log
```

They record database changes in a format used by MariaDB for purposes including replication and recovery workflows.

---

# 15. Point-in-Time Recovery

Suppose:

```text
10:00 AM
Full backup

10:30 AM
Important data inserted

11:00 AM
More transactions

11:17 AM
Accidental DELETE

11:20 AM
Problem discovered
```

You may want to recover to:

```text
11:16:59 AM
```

rather than restoring the database all the way back to 10:00.

This is the idea behind:

> **Point-in-time recovery (PITR).**

---

# 16. PITR Concept

Conceptually:

```text
Full Backup
     │
     ▼
10:00
     │
     ├── Binary log changes
     ├── Binary log changes
     ├── Binary log changes
     │
     ▼
11:16:59
     │
     ▼
Recovered database
```

This is why backup strategy and binary-log management need to be designed together.

---

# 17. Backup Storage

Never keep the only backup on the same disk as the database.

Bad:

```text
C:\
 ├── MariaDB
 └── Backups
```

If the disk fails:

```text
Database ❌
Backup ❌
```

Much better:

```text
Database Server
      │
      ▼
Backup Storage
      │
      ▼
Separate failure domain
```

Even better for critical environments:

```text
Primary
   ↓
Local backup
   ↓
Remote/offsite backup
   ↓
Immutable/controlled backup copy
```

The exact architecture depends on business requirements.

---

# 18. The 3-2-1 Principle

A commonly used backup strategy is the **3-2-1 approach**:

```text
3 copies of data
2 different storage/media types
1 copy offsite
```

Modern environments may go further with:

```text
Immutable storage
Air-gapped copies
Cloud object storage
Cross-region copies
```

The principle is to avoid a single failure destroying everything.

---

# 19. Backup Permissions

Your backup system needs appropriate privileges.

But don't automatically use:

```text
root
```

for every backup process.

Use a dedicated backup account where appropriate, with permissions suited to the backup method and your MariaDB version.

---

# 20. Simple Logical Backup Lab

Create a test database:

```sql
CREATE DATABASE backup_lab;
```

Create a table:

```sql
USE backup_lab;

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100)
);
```

Insert data:

```sql
INSERT INTO employees
VALUES
(1, 'Alice', 'DBA'),
(2, 'Bob', 'Application'),
(3, 'Carol', 'Security');
```

Verify:

```sql
SELECT * FROM employees;
```

---

# 21. Create a Logical Backup

From the OS:

```bash
mariadb-dump -u root -p backup_lab > backup_lab.sql
```

You now have:

```text
backup_lab.sql
```

Check the file:

```bash
ls -lh backup_lab.sql
```

On Windows PowerShell:

```powershell
Get-Item .\backup_lab.sql
```

---

# 22. Inspect the Backup

You can inspect the beginning:

Linux:

```bash
head backup_lab.sql
```

Windows PowerShell:

```powershell
Get-Content .\backup_lab.sql -TotalCount 30
```

You'll see SQL statements representing the database objects/data.

---

# 23. Never Assume a Backup Is Good

This is one of the biggest DBA lessons:

```text
Backup completed successfully
```

does **not** necessarily mean:

```text
Database can be recovered successfully
```

You need:

```text
Backup
   ↓
Restore test
   ↓
Validation
```

---

# 24. Restore Test

Create a test database:

```sql
CREATE DATABASE backup_restore_test;
```

Then restore the logical dump appropriately.

A typical command is:

```bash
mariadb -u root -p backup_restore_test < backup_lab.sql
```

Then connect:

```bash
mariadb -u root -p
```

Run:

```sql
USE backup_restore_test;

SELECT * FROM employees;
```

You should see:

```text
1 Alice
2 Bob
3 Carol
```

---

# 25. Recovery Validation

Don't stop at:

```text
Restore command completed.
```

Validate:

```text
Database exists
      ↓
Tables exist
      ↓
Row counts correct
      ↓
Important data correct
      ↓
Users/permissions appropriate
      ↓
Application can connect
      ↓
Application functionality works
```

---

# 26. What If the Database Is Huge?

Suppose:

```text
Database size = 3 TB
```

A logical dump may not be ideal as your only production backup method.

You should investigate:

```text
mariadb-backup
Incremental backups
Binary logs
Storage snapshots where appropriate
Backup compression
Parallelism
Backup windows
Recovery testing
```

---

# 27. `mariadb-backup`

For physical backups, you'll encounter:

```bash
mariadb-backup
```

A typical workflow conceptually looks like:

```text
Prepare backup destination
       ↓
Run full backup
       ↓
Prepare backup
       ↓
Validate
       ↓
Store safely
```

The exact command sequence depends on the MariaDB version and backup design.

Don't use a random command copied from an older MariaDB tutorial without checking version compatibility.

---

# 28. Full Physical Backup Concept

Conceptually:

```text
MariaDB
   │
   ▼
mariadb-backup
   │
   ▼
/backup/full/
   │
   ├── data
   ├── metadata
   └── backup information
```

The backup should be prepared appropriately before it is used for recovery.

---

# 29. Why `mariadb-backup` Matters

For larger environments, physical backup can provide advantages such as:

```text
Faster backup/restore workflows
Better suitability for large databases
Incremental backup capabilities
Production-oriented recovery workflows
```

But it requires a stronger understanding of backup preparation and restore procedures.

---

# 30. Backup Retention

Suppose you create:

```text
Daily backups
```

How long do you keep them?

You need a policy.

Example:

```text
Daily: 7 days
Weekly: 4 weeks
Monthly: 12 months
```

That's just an example.

Your actual policy comes from:

```text
Business
Compliance
Storage
Recovery requirements
```

---

# 31. Binary Log Retention

Binary logs also require retention management.

Think:

```text
Backup
+
Binary logs
=
Recovery capability
```

But binary logs consume disk.

Therefore:

```text
Retention
      +
Disk capacity
      +
Recovery requirements
```

must be balanced.

---

# 32. Backup Monitoring

Your monitoring should know:

```text
Last successful backup
Backup duration
Backup size
Backup location
Backup failures
Storage capacity
Binary log growth
Restore test status
```

Don't wait for a disaster to discover:

```text
"Backups have been failing for three months."
```

---

# 33. Backup Alerting

Good:

```text
Backup failed
     ↓
Alert DBA
     ↓
Investigate
```

Bad:

```text
Backup failed
     ↓
Nobody knows
     ↓
Disaster
     ↓
Restore attempt
     ↓
Backup doesn't exist
```

---

# 34. Recovery Runbook

A production DBA should have a written recovery procedure.

Example:

```text
1. Declare incident
2. Stop application writes if required
3. Identify failure point
4. Identify latest valid backup
5. Identify required binary logs
6. Restore backup
7. Apply required logs
8. Validate data
9. Validate application
10. Re-enable application
11. Monitor
12. Document incident
```

---

# 35. Accidental DELETE Scenario

Imagine:

```sql
DELETE FROM customers;
```

was accidentally executed.

Don't immediately restore the production database over itself.

First:

```text
STOP
  ↓
Determine scope
  ↓
Determine exact time
  ↓
Preserve evidence/logs
  ↓
Identify recovery point
  ↓
Build recovery environment if possible
  ↓
Recover required data
  ↓
Validate
```

This is much safer.

---

# 36. Recovery Environment

A strong recovery strategy often uses a separate server:

```text
Production
    │
    │ Backup
    ▼
Recovery Server
    │
    ▼
Restore
    │
    ▼
Validate
    │
    ▼
Extract required data
```

This avoids experimenting directly on production.

---

# 37. Backup Security

Backups contain database data.

Therefore:

```text
Production data
     ↓
Backup
     ↓
Sensitive information
```

Protect backups with:

```text
Encryption
Access control
Secure storage
Retention policies
Audit logging
```

Don't treat backups as harmless files.

---

# 38. Backup Encryption

If your database contains sensitive information, evaluate encryption for backup data.

Think:

```text
Database
   ↓
Encrypted backup
   ↓
Backup storage
```

Also protect the encryption keys.

> **Encrypted backup + lost key = unusable backup.**

Key management is part of backup management.

---

# 39. Windows Backup Considerations

On Windows:

```text
MariaDB
   ↓
mariadb-dump / mariadb-backup
   ↓
Backup location
```

Also consider:

```text
NTFS permissions
Windows service account
Windows Task Scheduler
PowerShell automation
Windows Defender/antivirus exclusions where appropriate
Disk space
```

Be careful with antivirus/security software on database and backup paths; changes should follow your organization's security policy and MariaDB/vendor guidance.

---

# 40. Linux Backup Considerations

On Linux:

```text
MariaDB
   ↓
mariadb-dump / mariadb-backup
   ↓
Filesystem
   ↓
cron/systemd automation
```

Also consider:

```text
Filesystem permissions
SELinux/AppArmor
Mount points
Storage capacity
Backup service account
```

---

# 41. Logical Backup Automation

Conceptually:

```bash
#!/bin/bash

BACKUP_DIR=/backup/mariadb

mariadb-dump -u backup_user -p backup_lab \
    > "$BACKUP_DIR/backup_lab.sql"
```

But **do not put production passwords directly into scripts**.

Credential handling should use an appropriate secure mechanism.

---

# 42. Backup Script Design

A production backup script should consider:

```text
Logging
Exit codes
Error handling
Timestamp
Compression
Encryption
Retention
Storage verification
Alerts
```

Not just:

```text
dump > file
```

---

# 43. Backup File Naming

Use meaningful names.

For example:

```text
backup_lab_2026-08-12_0200.sql
```

or:

```text
mariadb_full_2026-08-12_0200
```

Good naming helps during recovery.

---

# 44. Compression

Logical backups can often be compressed.

For example:

```text
SQL dump
   ↓
gzip
   ↓
smaller backup
```

But compression uses CPU.

Therefore consider:

```text
CPU
Backup window
Storage
Network
Recovery speed
```

---

# 45. Backup Verification

At minimum:

```text
Backup file exists
+
Non-zero size
+
Expected completion status
```

Better:

```text
Backup
   ↓
Integrity checks
   ↓
Test restore
   ↓
Validation
```

The second approach gives much more confidence.

---

# 46. The DBA Backup Pyramid

Think of backup maturity like this:

```text
Level 1
Backup exists

Level 2
Backup is monitored

Level 3
Backup is verified

Level 4
Restore is tested

Level 5
Recovery is rehearsed

Level 6
RPO/RTO are demonstrated
```

Production DBA teams should aim well beyond Level 1.

---

# 47. Backup Checklist

```text
☐ RPO defined
☐ RTO defined
☐ Backup method selected
☐ Full backup schedule
☐ Incremental strategy if required
☐ Binary log strategy
☐ Backup retention
☐ Backup storage
☐ Offsite copy
☐ Encryption
☐ Backup credentials
☐ Monitoring
☐ Alerting
☐ Restore procedure
☐ Restore testing
☐ Recovery runbook
☐ Documentation
```

---

# 48. DBA Challenge

### Q1

What is the difference between RPO and RTO?

### Q2

What is the difference between a logical and physical backup?

### Q3

Which MariaDB tools are commonly used for logical and physical backups?

### Q4

Why aren't binary logs simply "another backup"?

### Q5

Why should you test restores?

### Q6

Why shouldn't backups be stored only on the database server?

### Q7

If a backup job says **SUCCESS**, does that prove recovery will work?

### Q8

What should you do before restoring over a production database?

---

# Answers

### Q1

**RPO** defines acceptable data loss.

**RTO** defines acceptable recovery time.

### Q2

Logical backup represents database objects/data logically, while physical backup operates closer to the database's physical storage representation.

### Q3

Logical:

```text
mariadb-dump
```

Physical:

```text
mariadb-backup
```

### Q4

Binary logs record changes and are particularly useful for replication and point-in-time recovery. They are normally part of a broader backup/recovery strategy.

### Q5

Because a backup that cannot be restored is not a reliable recovery mechanism.

### Q6

A server/storage failure could destroy both the production database and its only backup.

### Q7

No.

You need verification and ideally a successful restore test.

### Q8

Determine the failure scope and recovery point, preserve necessary information/logs, and preferably perform recovery in a separate environment before modifying production.

---

# ⭐ Lesson 6 Golden Rules

> **1. Backup is not the same as recovery.**

> **2. RPO tells you how much data loss is acceptable.**

> **3. RTO tells you how quickly recovery must occur.**

> **4. Always test restores.**

> **5. Don't keep your only backup on the database server.**

> **6. Protect backups like production data.**

> **7. Binary logs are critical for certain recovery strategies.**

> **8. Never experiment with recovery directly on production when a safer recovery environment is available.**

---

# Your DBA Recovery Mental Model

```text
                  DISASTER / FAILURE
                         │
                         ▼
                    Identify Time
                         │
                         ▼
                    Find Backup
                         │
                         ▼
                 Restore Full Backup
                         │
                         ▼
                Apply Required Changes
                         │
                         ▼
                 Point-in-Time Recovery
                         │
                         ▼
                      Validate
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             Database         Application
              valid?             valid?
                 │               │
                 └───────┬───────┘
                         ▼
                    Go Live
```

## Next — Lesson 0.7: MariaDB Patching, Upgrades & Version Management

We'll cover **exactly the patching topic you originally asked for**:

```text
MariaDB versions
     ↓
Patch vs minor upgrade vs major upgrade
     ↓
Security patches
     ↓
Pre-checks
     ↓
Backup
     ↓
Linux patching
     ↓
Windows patching
     ↓
Repository/package management
     ↓
Rolling/HA considerations
     ↓
Upgrade testing
     ↓
Rollback strategy
     ↓
Post-patch validation
     ↓
Common patching failures
```

We'll also compare **Linux vs Windows patching procedures** step-by-step.
