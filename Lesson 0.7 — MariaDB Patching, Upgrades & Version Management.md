# Lesson 0.7 — MariaDB Patching, Upgrades & Version Management

This lesson covers the **patching and upgrade work of a MariaDB DBA** across **Linux and Windows**.

The most important distinction:

> **Patching is not the same thing as upgrading.**

A DBA must know **what is changing, why it is changing, how to prepare, how to execute it, and how to recover if it fails.**

---

# 1. MariaDB Versioning

First determine your current version:

```sql
SELECT VERSION();
```

You can also use:

```bash
mariadb --version
```

Example:

```text
MariaDB 11.x.x
```

Your exact output depends on the installed release.

---

# 2. Patch vs Upgrade

Think of changes in three broad categories:

```text
MariaDB
   │
   ├── Security/Bug Fix
   │
   ├── Maintenance Release
   │
   └── Major Version Upgrade
```

### Patch/Maintenance Update

Usually intended to address:

```text
Security fixes
Bug fixes
Stability
Minor improvements
```

### Major Upgrade

Can introduce:

```text
New features
Behavior changes
Compatibility changes
Removed features
Changed defaults
Migration requirements
```

Major upgrades therefore require much more testing.

---

# 3. Why DBAs Patch

A production database shouldn't remain unpatched indefinitely.

Reasons include:

```text
Security vulnerabilities
Bug fixes
Data corruption fixes
Replication fixes
Performance/stability fixes
Operating-system compatibility
Vendor support requirements
```

But:

> **"There is a new version" does not mean "upgrade production immediately."**

---

# 4. The DBA Patch Lifecycle

Use this process:

```text
                    PATCH REQUEST
                         │
                         ▼
                  Identify Version
                         │
                         ▼
                  Review Release
                         │
                         ▼
                  Impact Analysis
                         │
                         ▼
                    Test First
                         │
                         ▼
                  Backup / Recovery
                         │
                         ▼
                  Schedule Change
                         │
                         ▼
                    Patch Server
                         │
                         ▼
                 Startup Validation
                         │
                         ▼
                Database Validation
                         │
                         ▼
               Application Validation
                         │
                         ▼
                    Monitoring
                         │
                         ▼
                  Close Change
```

---

# 5. Step 1 — Identify Current Version

Run:

```sql
SELECT VERSION();
```

Record:

```text
Current version
Operating system
Server role
Database size
Replication role
Application dependencies
```

Example:

```text
Server: DB01
OS: Linux
Role: Primary
MariaDB: 11.x
Database size: 2 TB
Replication: Enabled
```

This becomes your baseline.

---

# 6. Step 2 — Determine Target Version

Never say:

> "Let's install the latest MariaDB."

Instead:

```text
Current version
       ↓
Supported upgrade path
       ↓
Target version
       ↓
Application compatibility
       ↓
OS compatibility
```

You should verify the supported upgrade path for your specific MariaDB release before proceeding.

---

# 7. Step 3 — Read Release Notes

Before patching, review the release information for the target version.

Look specifically for:

```text
Security fixes
Breaking changes
Removed variables
Changed defaults
SQL behavior changes
Storage-engine changes
Replication changes
Authentication changes
Known upgrade issues
```

This is where experienced DBAs differ from people who simply run package commands.

---

# 8. Step 4 — Check Application Compatibility

Your database doesn't exist by itself.

Architecture:

```text
Application
     │
     ▼
MariaDB Connector/Driver
     │
     ▼
MariaDB
     │
     ▼
Storage
```

A MariaDB upgrade can affect:

```text
SQL behavior
Authentication
Drivers
Character sets
Reserved words
Connection behavior
Performance
```

Therefore involve the application team when appropriate.

---

# 9. Step 5 — Check Current Health

Never patch an already unhealthy database without understanding the issue.

Check:

```text
CPU
Memory
Disk
Connections
Errors
Replication
Backup status
Storage capacity
Long-running queries
```

Conceptually:

```text
Healthy
   ↓
Patch

Unhealthy
   ↓
Investigate first
```

---

# 10. Check Connections

Run:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

and:

```sql
SHOW STATUS LIKE 'Threads_running';
```

You want to understand the workload before maintenance.

---

# 11. Check Replication

If the server participates in replication, don't start patching until you understand the topology.

Think:

```text
             Primary
                │
       ┌────────┴────────┐
       ▼                 ▼
    Replica 1         Replica 2
```

Your patching strategy should account for the role of each server.

---

# 12. Check Disk Space

Before an upgrade:

```text
Data
+
Logs
+
Temporary files
+
Package files
+
Backup
```

can require additional disk space.

A database upgrade failing because the filesystem fills up is an avoidable DBA mistake.

---

# 13. Check Backups

Before a major change:

```text
Latest backup
      ↓
Backup verified
      ↓
Recovery procedure known
```

Don't just ask:

> "Do we have a backup?"

Ask:

> **"Can we recover from it?"**

---

# 14. Create a Rollback Plan

This is critical.

Your plan should answer:

```text
What if MariaDB won't start?

What if replication breaks?

What if the application fails?

What if performance degrades?

What if the upgrade changes data behavior?

How do we restore service?
```

---

# 15. Snapshot ≠ Complete Backup Strategy

Some environments use storage or VM snapshots.

They can be useful, but don't assume:

```text
Snapshot = guaranteed database recovery
```

You need to understand:

```text
Crash consistency
Storage consistency
Database state
Snapshot retention
Restore capability
```

Use snapshots according to your infrastructure and MariaDB recovery design.

---

# 16. Linux Patching

On Linux, package management depends on the distribution.

Common examples:

```text
Debian / Ubuntu
    ↓
APT

RHEL-family
    ↓
DNF/YUM
```

The exact MariaDB repository and commands depend on your operating system and target version.

---

# 17. Ubuntu/Debian Concept

First inspect installed packages:

```bash
dpkg -l | grep -i mariadb
```

Check:

```bash
mariadb --version
```

Then review the configured repository.

Conceptually:

```text
Repository
    ↓
Available MariaDB version
    ↓
Package manager
    ↓
Upgrade
```

Don't blindly change repositories on production.

---

# 18. RHEL-family Concept

Check:

```bash
rpm -qa | grep -i mariadb
```

Then:

```bash
mariadb --version
```

Inspect configured repositories.

Conceptually:

```text
MariaDB repository
       ↓
DNF
       ↓
Packages
       ↓
MariaDB
```

---

# 19. Linux Service Status

Before patching:

```bash
systemctl status mariadb
```

After patching:

```bash
systemctl status mariadb
```

Also verify:

```bash
mariadb-admin ping
```

A successful service status alone isn't enough.

---

# 20. Linux Patch Workflow

A controlled workflow:

```text
1. Verify current version
2. Check OS/version
3. Review target release
4. Check backups
5. Check replication
6. Check disk space
7. Stop application if required
8. Patch packages
9. Start MariaDB
10. Verify version
11. Check logs
12. Validate database
13. Validate application
14. Monitor
```

---

# 21. Windows Patching

Windows is similar conceptually but different operationally.

You need to consider:

```text
MariaDB installation
Windows service
my.ini
Data directory
NTFS permissions
Windows Firewall
MariaDB binaries
PATH/environment
Service account
```

---

# 22. Identify MariaDB Service

PowerShell:

```powershell
Get-Service *MariaDB*
```

You may see something like:

```text
Status    Name
------    ----
Running   MariaDB
```

The exact service name can differ.

---

# 23. Check MariaDB Version on Windows

Use:

```powershell
mariadb --version
```

If the command isn't in PATH, use the MariaDB installation's executable path.

---

# 24. Windows Upgrade Workflow

Conceptually:

```text
Current MariaDB
       ↓
Backup
       ↓
Application maintenance
       ↓
Stop MariaDB service
       ↓
Upgrade/install target version
       ↓
Verify configuration
       ↓
Start service
       ↓
Check error log
       ↓
Check version
       ↓
Validate database
       ↓
Validate application
```

---

# 25. Windows Service Management

Stop:

```powershell
Stop-Service MariaDB
```

Start:

```powershell
Start-Service MariaDB
```

Check:

```powershell
Get-Service MariaDB
```

Only use the actual service name on your server.

---

# 26. Linux vs Windows

| Area        | Linux                    | Windows                     |
| ----------- | ------------------------ | --------------------------- |
| Service     | `systemctl`              | Windows Service             |
| Packages    | APT/DNF/etc.             | Installer/package mechanism |
| Config      | `my.cnf` variants        | `my.ini`                    |
| Logs        | Files/journal            | Files/Event Viewer          |
| Firewall    | nftables/firewalld/other | Windows Firewall            |
| Permissions | Unix permissions/ACLs    | NTFS ACLs                   |
| Automation  | shell/systemd/cron       | PowerShell/Task Scheduler   |

The **database principles remain the same**.

---

# 27. Configuration Compatibility

This is a major upgrade risk.

Suppose your old configuration contains:

```text
old_parameter=value
```

The new MariaDB version may:

```text
Support it
Change its meaning
Deprecate it
Remove it
Change its default
```

Therefore:

> **Never blindly carry an old configuration file into a new major version.**

---

# 28. Configuration Review

Before upgrade:

```text
my.cnf / my.ini
       ↓
Inventory custom settings
       ↓
Compare against target version
       ↓
Remove obsolete settings
       ↓
Validate new configuration
```

Pay particular attention to:

```text
Authentication
InnoDB
Replication
Logging
Networking
Character sets
Deprecated variables
```

---

# 29. Patch vs Major Upgrade Risk

Think:

```text
Maintenance patch
      │
      ▼
Usually lower change risk

Major upgrade
      │
      ▼
Higher compatibility risk
```

But don't interpret this as:

> "Maintenance updates are always safe."

They still require testing and change control.

---

# 30. Testing Environment

The best approach:

```text
Production
    │
    │ representative backup/data
    ▼
Test Environment
    │
    ▼
Target MariaDB Version
    │
    ▼
Application Tests
```

Test:

```text
Login
Queries
Writes
Reports
Stored procedures
Jobs
Backups
Replication
Monitoring
```

---

# 31. Upgrade Testing

At minimum:

```text
Application starts
        ↓
Database connection works
        ↓
SELECT works
        ↓
INSERT/UPDATE works
        ↓
Critical business transactions work
        ↓
Reports work
        ↓
Backup works
```

---

# 32. Performance Testing

An upgrade can be technically successful but operationally bad.

Example:

```text
Before:
Query = 100 ms

After:
Query = 700 ms
```

MariaDB started successfully.

But the upgrade may still be unacceptable.

Compare:

```text
CPU
Memory
IO
Connections
Query latency
Throughput
Errors
```

before and after.

---

# 33. Replication Upgrade

For replication environments, patching can be performed in a controlled sequence depending on topology and supported upgrade approach.

Conceptually:

```text
Primary
   │
   ├── Replica A
   └── Replica B
```

You might use a replica to validate the new version before changing the primary, depending on your architecture and compatibility constraints.

But:

> **Replication topology alone does not make every upgrade path safe.**

Version compatibility and MariaDB documentation must be checked.

---

# 34. High Availability

If you have HA:

```text
Node A
Node B
Node C
```

you may be able to perform maintenance with reduced downtime.

But HA does **not** eliminate upgrade risk.

You still need:

```text
Compatibility
Failover testing
Quorum awareness
Replication health
Rollback planning
Application validation
```

---

# 35. Zero-Downtime Is Not Magic

Marketing language sometimes says:

> "This can be upgraded with zero downtime."

A DBA asks:

```text
Under what architecture?

What traffic?

What replication mode?

What version combination?

What failure scenarios?

What rollback?

What application behavior?
```

Always understand the actual mechanics.

---

# 36. Post-Patch Validation

After the upgrade:

```sql
SELECT VERSION();
```

Then check:

```sql
SHOW VARIABLES LIKE 'datadir';
```

```sql
SHOW VARIABLES LIKE 'port';
```

```sql
SHOW VARIABLES LIKE 'max_connections';
```

Compare important settings against your baseline.

---

# 37. Check Error Logs

Immediately inspect the MariaDB error log.

Look for:

```text
Errors
Warnings
Plugin failures
Storage-engine problems
Replication errors
Authentication issues
Configuration problems
```

Don't wait for users to report problems.

---

# 38. Check Application Connectivity

A DBA should test:

```text
Application
    ↓
Database connection
    ↓
Authentication
    ↓
Query
    ↓
Transaction
```

For example:

```text
Login
Create transaction
Read data
Modify test/appropriate data
Commit
```

Use your application's normal health-check process where available.

---

# 39. Check Replication After Patching

If applicable, verify:

```text
Replication connected
Replication lag acceptable
No SQL errors
No IO errors
GTID/binlog state expected
```

The exact commands depend on your MariaDB replication configuration.

---

# 40. Check Backups After Patching

This is frequently forgotten.

After upgrading:

```text
Old backup process
       ↓
New MariaDB version
       ↓
Run backup
       ↓
Verify backup
```

Your backup tool and scripts must remain compatible.

---

# 41. Rollback

Suppose:

```text
Upgrade
   ↓
MariaDB won't start
```

Your rollback might involve:

```text
Stop service
   ↓
Restore previous packages/configuration
   ↓
Restore database if required
   ↓
Start previous version
   ↓
Validate
```

But rollback isn't always as simple as reinstalling the old package.

Database format/data changes may make downgrade unsafe or unsupported.

Therefore:

> **Always establish the supported rollback strategy before upgrading.**

---

# 42. Why Downgrade Can Be Dangerous

Imagine:

```text
Old MariaDB
     ↓
Upgrade
     ↓
Data/metadata changes
     ↓
New MariaDB
```

Then:

```text
New MariaDB
     ↓
Install old version
```

The old version may not understand the modified state.

Therefore:

```text
Upgrade ≠ guaranteed reversible
```

A tested backup/restore path is often much safer than assuming a package downgrade will work.

---

# 43. Change Management

Production patching should have a change record.

Include:

```text
Change ID
Server
Current version
Target version
Reason
Risk
Impact
Backup
Maintenance window
Execution steps
Validation
Rollback
Owner
Approval
Result
```

This is standard DBA discipline.

---

# 44. Patch Runbook Example

```text
CHANGE:
Upgrade MariaDB DB01

CURRENT:
MariaDB X.Y

TARGET:
MariaDB X.Z

PRECHECK:
☐ Backup verified
☐ Disk checked
☐ Replication healthy
☐ Application owner notified
☐ Config backed up
☐ Rollback plan ready

EXECUTION:
☐ Stop application
☐ Stop MariaDB if required
☐ Apply package update
☐ Start MariaDB

VALIDATION:
☐ Version
☐ Service
☐ Logs
☐ Database
☐ Replication
☐ Application
☐ Backup

RESULT:
SUCCESS / FAILURE
```

---

# 45. Emergency Security Patching

Sometimes you cannot wait for the normal patch cycle.

For example:

```text
Critical vulnerability
       ↓
Security assessment
       ↓
Emergency change
       ↓
Accelerated testing
       ↓
Patch
       ↓
Validation
```

Even emergency changes should be documented and tested as far as practical.

---

# 46. Common Patching Mistakes

### ❌ "Just run update."

Not enough.

### ❌ No backup verification

Dangerous.

### ❌ No application testing

Database can be healthy while application is broken.

### ❌ Copy old `my.cnf` blindly

Can introduce incompatible parameters.

### ❌ Assume downgrade is easy

It may not be.

### ❌ Ignore replication

Can create topology failures.

### ❌ Ignore disk space

Can cause upgrade/startup failures.

### ❌ Patch production first

Test first whenever practical.

---

# 47. Linux Practical Checklist

Before patch:

```bash
mariadb --version
```

```bash
systemctl status mariadb
```

Check logs and disk space.

Then after patch:

```bash
mariadb --version
```

```bash
systemctl status mariadb
```

```bash
mariadb-admin ping
```

Then:

```sql
SELECT VERSION();
```

---

# 48. Windows Practical Checklist

Before patch:

```powershell
mariadb --version
```

```powershell
Get-Service MariaDB
```

Verify:

```text
my.ini
Data directory
Disk space
Backup
Service account
Firewall
```

After patch:

```powershell
Get-Service MariaDB
```

```powershell
mariadb --version
```

Then:

```sql
SELECT VERSION();
```

Review the MariaDB error log and Windows Event Viewer.

---

# 49. DBA Challenge

### Q1

What is the difference between patching and a major upgrade?

### Q2

Why should you check release notes?

### Q3

Why should configuration files be reviewed before an upgrade?

### Q4

Why is a successful MariaDB startup not enough to declare an upgrade successful?

### Q5

Why should you test an upgrade before production?

### Q6

Why can downgrade be dangerous?

### Q7

Name three things you should validate after patching.

### Q8

Why should application owners be involved in a database upgrade?

---

# Answers

### Q1

Patching generally focuses on maintenance/security/bug fixes, while a major upgrade can introduce significant compatibility and behavioral changes.

### Q2

To identify security fixes, bugs, changed behavior, deprecated settings, compatibility issues, and upgrade-specific requirements.

### Q3

Because parameters can be deprecated, removed, or changed between versions.

### Q4

Because the application, replication, queries, performance, backups, and configuration may still be broken.

### Q5

To discover compatibility and operational problems before they affect production.

### Q6

Because database data/metadata may have changed in ways that the older version cannot safely handle.

### Q7

For example:

```text
Version
MariaDB service
Error logs
Database connectivity
Replication
Application
Backups
```

### Q8

Because applications depend on database behavior, authentication, SQL compatibility, drivers, performance, and schema behavior.

---

# ⭐ Lesson 7 Golden Rules

> **1. Never patch production blindly.**

> **2. Know your current version and target version.**

> **3. Read the release notes.**

> **4. Test before production.**

> **5. Back up and verify recovery before major changes.**

> **6. Review `my.cnf`/`my.ini` for compatibility.**

> **7. Validate the application, not just MariaDB.**

> **8. Never assume downgrade is a safe rollback.**

> **9. Treat replication and HA as part of the upgrade plan.**

> **10. Document every production change.**

---

# The DBA Patching Mental Model

```text
              CURRENT MARIADB
                     │
                     ▼
              Health Check
                     │
                     ▼
             Backup + Recovery
                     │
                     ▼
             Read Release Notes
                     │
                     ▼
          Compatibility Assessment
                     │
                     ▼
               TEST UPGRADE
                     │
             ┌───────┴───────┐
             ▼               ▼
          SUCCESS          FAILURE
             │               │
             ▼               ▼
       Production        Investigate
          Change
             │
             ▼
       Post-Patch Check
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      DB    APP   REPL
       │     │     │
       └─────┼─────┘
             ▼
         MONITOR
```

## Next — Lesson 0.8: MariaDB Migration

We'll cover **migration end-to-end**:

```text
Source → Target
```

including:

* Same-server migration
* Server-to-server migration
* Linux → Linux
* Windows → Windows
* Windows → Linux
* Linux → Windows
* Logical migration
* Physical migration
* Character sets
* Users/grants
* Large databases
* Downtime planning
* Replication-based migration
* Validation
* Cutover
* Rollback
* Migration runbook
* Production migration checklist
