# Lesson 0.10 — MariaDB High Availability, Replication & Production DBA Operations

🎯 **Final lesson of the 10-lesson MariaDB DBA course**

Now we'll combine:

```text
Installation
Configuration
Security
Backup
Patching
Migration
Monitoring
Performance
Replication
High Availability
Disaster Recovery
Production Operations
```

The goal is to understand how a **production MariaDB DBA operates a database environment end-to-end**.

---

# 1. What Is High Availability?

High Availability (HA) means designing the database environment so that a failure of one component does not cause unacceptable application downtime.

Basic idea:

```text
             APPLICATION
                  │
                  ▼
          ┌───────────────┐
          │    PRIMARY    │
          │    MariaDB    │
          └───────┬───────┘
                  │
             Replication
                  │
                  ▼
          ┌───────────────┐
          │    REPLICA    │
          │    MariaDB    │
          └───────────────┘
```

If the primary fails, the replica may be promoted depending on your HA design.

---

# 2. HA vs Backup

These are **not the same thing**.

### High Availability

Protects against:

```text
Server failure
Service failure
Some infrastructure failures
```

### Backup

Protects against:

```text
Accidental deletion
Data corruption
Bad deployment
Logical errors
Ransomware scenarios
Disaster recovery
```

Think:

```text
HA       → Keep service available
Backup   → Recover data
```

You generally need both.

---

# 3. Replication

MariaDB replication allows changes from one server to be transmitted to another.

Conceptually:

```text
             PRIMARY
                │
                │ Binary Log
                ▼
             REPLICA
```

The primary records changes in binary logs.

The replica retrieves and applies those changes.

---

# 4. Binary Log

The binary log records database changes in a form used for replication and other operational purposes.

Conceptually:

```text
Client
  │
  ▼
PRIMARY
  │
  ├── Data change
  │
  └── Binary Log
          │
          ▼
       REPLICA
```

Binary logging is therefore extremely important in many replication architectures.

---

# 5. Why Replication Is Useful

Replication can provide:

```text
Read scaling
Reporting
Backup assistance
Disaster recovery
Migration
Failover capability
```

But replication itself is **not automatically a complete HA solution**.

You still need:

```text
Monitoring
Failover procedures
Application handling
Testing
Operational discipline
```

---

# 6. Primary and Replica

Traditional terminology:

```text
PRIMARY
   │
   │ replication
   ▼
REPLICA
```

The primary generally handles writes.

The replica receives changes.

Depending on architecture, replicas may also serve read workloads.

---

# 7. Replication Flow

At a high level:

```text
1. Application writes to PRIMARY
              │
              ▼
2. PRIMARY records changes
              │
              ▼
3. Binary log
              │
              ▼
4. REPLICA receives changes
              │
              ▼
5. REPLICA applies changes
```

The exact implementation details depend on MariaDB version and replication configuration.

---

# 8. Replication Lag

One of the most important DBA metrics.

Example:

```text
PRIMARY
  │
  │ transaction
  ▼
REPLICA
  │
  │ still applying
  ▼
Lag
```

If:

```text
Primary position = current
Replica position = behind
```

the replica is lagging.

---

# 9. Why Replication Lag Matters

Suppose:

```text
Customer creates order
        │
        ▼
PRIMARY
        │
        │ replication delayed
        ▼
REPLICA
```

A report reading from the replica may temporarily not see the new order.

Therefore applications need to understand replication consistency.

---

# 10. Common Causes of Replication Lag

Potential causes include:

```text
Heavy writes
Slow replica storage
Large transactions
Long-running queries
CPU saturation
Network problems
Lock contention
Insufficient resources
Replica workload
```

Don't assume:

> "Replication is broken."

First measure the lag and investigate why.

---

# 11. Replication Monitoring

A DBA should regularly monitor:

```text
Replication status
Replication errors
Lag
Binary log position/GTID state
IO health
SQL/apply health
Disk space
```

The exact commands depend on the MariaDB replication configuration and version.

---

# 12. GTID

GTID means:

> **Global Transaction ID**

GTIDs provide a way to identify transactions across replication.

Conceptually:

```text
Transaction A → GTID A
Transaction B → GTID B
Transaction C → GTID C
```

The replica can use GTID information to determine which transactions it has already processed and which remain.

---

# 13. Why DBAs Like GTID

GTID-based replication can simplify operational tasks such as:

```text
Failover
Re-pointing replicas
Replication recovery
Tracking transaction state
```

But:

> **GTID must be designed and configured correctly for your MariaDB version/topology.**

Don't copy a configuration from another environment blindly.

---

# 14. Switchover vs Failover

These terms are important.

### Switchover

Planned operation:

```text
PRIMARY
   │
   ▼
Controlled transition
   │
   ▼
REPLICA becomes primary
```

Used for:

```text
Maintenance
Patching
Planned migration
Testing
```

### Failover

Unplanned event:

```text
PRIMARY
   │
   X
FAILURE
   │
   ▼
REPLICA promoted
```

Used when the primary has failed.

---

# 15. Planned Switchover

A controlled process might look like:

```text
1. Verify replica health
2. Confirm replication is caught up
3. Stop/freeze writes
4. Ensure final synchronization
5. Promote target
6. Redirect application
7. Validate
8. Resume writes
9. Monitor
```

Exact steps depend on your HA architecture.

---

# 16. Emergency Failover

Suppose:

```text
10:00
PRIMARY fails
```

The DBA must determine:

```text
Is primary really unavailable?
Is replica healthy?
How far behind is replica?
Which transactions are missing?
Can replica safely be promoted?
How will applications reconnect?
```

Then execute the approved failover runbook.

---

# 17. Don't Promote a Broken Replica

Before promotion:

```text
REPLICA
  │
  ├── MariaDB healthy?
  ├── Storage healthy?
  ├── Replication state healthy?
  ├── Data consistent?
  ├── Enough capacity?
  └── Application connectivity?
```

Only then should promotion be considered.

---

# 18. Split Brain

One of the most dangerous HA problems.

Imagine:

```text
       PRIMARY A
        /     \
       /       \
Application   Application
       \       /
        \     /
       PRIMARY B
```

Both servers think they are primary.

Now:

```text
A receives writes
B receives writes
```

You have divergent data.

This is called:

> **Split brain.**

HA systems need mechanisms/procedures to prevent or safely handle this.

---

# 19. Fencing

A key HA concept.

Before promoting a replica, you may need to ensure the old primary cannot continue accepting writes.

Conceptually:

```text
OLD PRIMARY
     │
     X
FENCED / ISOLATED
     │
     ▼
NEW PRIMARY
```

Otherwise both systems could accept writes.

---

# 20. Application Failover

Database failover isn't enough.

Suppose:

```text
Database:
OLD → NEW
```

But application still connects to:

```text
old-db.company.com
```

Then the application is still broken.

Your architecture should consider:

```text
Application connection handling
DNS
VIP/load balancer
Connection pools
Service discovery
Secrets
```

---

# 21. Disaster Recovery

DR means recovering from a major failure.

Examples:

```text
Entire server lost
Storage failure
Data center failure
Ransomware
Major corruption
Human error
Cloud infrastructure failure
```

Typical architecture:

```text
PRIMARY SITE
     │
     │ replication / backup
     ▼
DR SITE
```

---

# 22. RPO

RPO = **Recovery Point Objective**

It answers:

> "How much data can we afford to lose?"

Example:

```text
RPO = 5 minutes
```

means the organization may tolerate up to approximately five minutes of data loss under the defined recovery scenario.

---

# 23. RTO

RTO = **Recovery Time Objective**

It answers:

> "How quickly must the service be restored?"

Example:

```text
RTO = 30 minutes
```

means the recovery process should target restoration within approximately 30 minutes.

---

# 24. RPO + RTO

Think:

```text
RPO
 ↓
How much data loss?

RTO
 ↓
How much downtime?
```

Example:

```text
Business requirement:

RPO = 5 min
RTO = 30 min
```

Your MariaDB architecture must be designed to meet those requirements.

---

# 25. Backup Strategy

A production DBA should know:

```text
What is backed up?
When?
Where?
How long retained?
Encrypted?
Tested?
How quickly restored?
```

A backup that has never been tested is only an assumption.

---

# 26. Backup Types

Common concepts:

```text
Logical backup
Physical backup
Full backup
Incremental backup
Binary logs
```

The right combination depends on your recovery requirements.

---

# 27. Backup Validation

Don't stop at:

```text
Backup completed successfully
```

You also need:

```text
Can we read it?
Can we restore it?
Is the data complete?
How long does restoration take?
Does the restored database start?
Does the application work?
```

---

# 28. Restore Testing

A DBA should periodically perform:

```text
Backup
  ↓
Restore to test environment
  ↓
Start MariaDB
  ↓
Validate databases
  ↓
Validate users
  ↓
Validate application
  ↓
Measure recovery time
```

This gives you evidence that DR actually works.

---

# 29. Patching in Production

You learned patching earlier.

Now combine it with HA.

Instead of:

```text
Stop only database
        ↓
Downtime
        ↓
Patch
        ↓
Start
```

a HA architecture may allow:

```text
PRIMARY
   │
   ▼
REPLICA
   │
   ▼
Patch / maintain replica
   │
   ▼
Switchover
   │
   ▼
Patch old primary
```

This can reduce downtime, depending on architecture and compatibility.

---

# 30. Patch Planning

Before a MariaDB patch:

```text
☐ Version compatibility
☐ Application compatibility
☐ Backup
☐ Rollback plan
☐ Replication health
☐ Disk space
☐ Test environment
☐ Maintenance window
☐ Monitoring
☐ Stakeholder approval
```

---

# 31. Upgrade vs Patch

A patch may be:

```text
Bug fix
Security fix
Maintenance release
```

An upgrade may involve:

```text
Version family change
Major/minor compatibility considerations
Configuration changes
Application testing
```

Always consult the MariaDB documentation for the exact version path and supported upgrade procedure.

---

# 32. Security Operations

A production DBA should regularly review:

```text
Users
Privileges
Authentication
TLS
Network exposure
Firewall
Audit requirements
Passwords/secrets
Unused accounts
```

Example:

```sql id="8es5e2"
SELECT User, Host
FROM mysql.user;
```

Look for unnecessary accounts.

---

# 33. Least Privilege

Application users should receive only the privileges required.

Avoid giving:

```text
ALL PRIVILEGES
```

to every application account.

Think:

```text
Application
   ↓
Required permissions only
```

rather than:

```text
Application
   ↓
Full database control
```

---

# 34. DBA Daily Checklist

A production DBA may review:

```text
☐ MariaDB service status
☐ Replication health
☐ Replication lag
☐ Backup status
☐ Disk space
☐ CPU
☐ Memory
☐ Connections
☐ Long-running queries
☐ Errors
☐ Security alerts
☐ Monitoring alerts
```

---

# 35. DBA Weekly Checklist

```text
☐ Review slow queries
☐ Review capacity
☐ Review backup retention
☐ Review failed jobs
☐ Review replication
☐ Review error logs
☐ Review unused accounts
☐ Review disk growth
☐ Review performance trends
```

---

# 36. DBA Monthly Checklist

```text
☐ Restore test
☐ DR review
☐ Capacity planning
☐ Patch review
☐ Security review
☐ Index review
☐ Configuration review
☐ HA test where appropriate
☐ Documentation update
☐ Runbook review
```

---

# 37. Production Change Management

Never make major production changes like:

```text
Configuration change
Index creation
Upgrade
Migration
Failover
```

without understanding:

```text
Change
Risk
Impact
Rollback
Validation
Monitoring
```

A good change record contains:

```text
WHAT?
WHY?
WHEN?
RISK?
HOW?
ROLLBACK?
VALIDATION?
```

---

# 38. Incident Management

When production breaks:

```text
1. Detect
2. Assess
3. Stabilize
4. Investigate
5. Recover
6. Validate
7. Monitor
8. Document
9. Root-cause analysis
```

Don't immediately focus on:

> "Who caused it?"

Focus first on:

> **"How do we restore service safely?"**

---

# 39. Root Cause Analysis

After recovery, ask:

```text
What happened?
Why did it happen?
Why wasn't it detected earlier?
Why didn't existing controls prevent it?
What will prevent recurrence?
```

Example:

```text
Disk full
   ↓
Binary logs grew
   ↓
Replica was disconnected
   ↓
Monitoring didn't alert
   ↓
Alerting gap
```

The real root cause may be the monitoring/control failure, not merely "disk full."

---

# 40. Complete MariaDB Production Architecture

A mature environment might look like:

```text
                         APPLICATION
                              │
                              ▼
                       LOAD BALANCER
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
                 DB PRIMARY        DB REPLICA
                     │                 │
                     │ replication     │
                     └────────┬────────┘
                              │
                              ▼
                         BACKUP SYSTEM
                              │
                              ▼
                           DR SITE
```

Around all of this:

```text
Monitoring
Security
Alerting
Patching
Documentation
Incident Response
```

---

# 41. Complete DBA Lifecycle

You started this course with installation.

Now connect everything:

```text
INSTALL
   ↓
CONFIGURE
   ↓
SECURE
   ↓
BACKUP
   ↓
MONITOR
   ↓
TUNE
   ↓
PATCH
   ↓
MIGRATE
   ↓
REPLICATE
   ↓
FAILOVER
   ↓
RECOVER
   ↓
DOCUMENT
```

That's the MariaDB DBA lifecycle.

---

# 42. Windows DBA Workflow

For Windows:

```text
MariaDB Service
      ↓
Windows Event Viewer
      ↓
MariaDB Logs
      ↓
Performance Monitor
      ↓
Task Manager / Resource Monitor
      ↓
MariaDB SQL Diagnostics
      ↓
Backup / Replication
      ↓
Application Validation
```

Useful commands include:

```powershell
Get-Service MariaDB
```

and:

```powershell
Get-Process
```

Then investigate MariaDB from SQL.

---

# 43. Linux DBA Workflow

For Linux:

```text
systemctl
   ↓
MariaDB logs
   ↓
CPU / Memory
   ↓
Disk / I/O
   ↓
Network
   ↓
MariaDB SQL diagnostics
   ↓
Backup / Replication
   ↓
Application
```

Useful commands:

```bash
systemctl status mariadb
```

```bash
journalctl -u mariadb
```

```bash
df -h
```

```bash
free -h
```

```bash
vmstat 1
```

---

# 44. The 10-Lesson Course — Complete

You've now covered:

| Lesson | Topic                                     |
| ------ | ----------------------------------------- |
| 1      | MariaDB DBA Fundamentals                  |
| 2      | MariaDB Architecture & Core Concepts      |
| 3      | Installation & Initial Setup              |
| 4      | Configuration & Administration            |
| 5      | Users, Security & Privileges              |
| 6      | Backup, Restore & Recovery                |
| 7      | Patching, Upgrades & Maintenance          |
| 8      | Migration                                 |
| 9      | Monitoring, Performance & Troubleshooting |
| 10     | HA, Replication & Production Operations   |

---

# 45. What You Should Now Be Able to Explain

You should be able to answer:

### Installation

```text
How do I install MariaDB on Linux?
How do I install it on Windows?
Where is the configuration?
How do I start/stop the service?
```

### Configuration

```text
Where is my.cnf/my.ini?
How do I change a setting?
How do I validate configuration changes?
```

### Security

```text
How do I create users?
How do I grant privileges?
How do I remove privileges?
How do I audit accounts?
```

### Backup

```text
How do I back up MariaDB?
How do I restore?
How do I test backups?
```

### Patching

```text
How do I plan a patch?
How do I validate compatibility?
How do I roll back?
```

### Migration

```text
How do I migrate Windows → Linux?
Linux → Windows?
How do I validate migrated data?
```

### Performance

```text
How do I investigate a slow database?
How do I use SHOW PROCESSLIST?
How do I use EXPLAIN?
How do I investigate locks?
```

### HA

```text
What is replication?
What is failover?
What is switchover?
What is replication lag?
What are RPO/RTO?
```

---

# 46. Final DBA Incident Scenario

Imagine you receive this alert:

> **"Production MariaDB application is timing out."**

Your thought process should now be:

```text
             ALERT
               │
               ▼
        Is MariaDB alive?
               │
               ▼
        Check service/status
               │
               ▼
         Check OS health
        ┌──────┼──────┐
        ▼      ▼      ▼
       CPU    RAM    DISK
        │      │      │
        └──────┼──────┘
               ▼
        Check connections
               │
               ▼
        SHOW PROCESSLIST
               │
        ┌──────┴──────┐
        ▼             ▼
      Queries        Locks
        │             │
        ▼             ▼
      EXPLAIN       Investigate
        │             │
        └──────┬──────┘
               ▼
           ROOT CAUSE
               │
               ▼
         CONTROLLED FIX
               │
               ▼
           VALIDATION
               │
               ▼
            MONITOR
```

**That is the DBA mindset.**

---

# 🏆 Final MariaDB DBA Rules

> **1. Protect the data first.**

> **2. Never make production changes without a rollback plan.**

> **3. Backups are useless until restores are tested.**

> **4. HA is not a replacement for backups.**

> **5. Replication is not automatically HA.**

> **6. Monitor before there is an outage.**

> **7. Don't troubleshoot by guessing.**

> **8. Don't optimize without measuring.**

> **9. Don't blindly copy configuration between servers.**

> **10. Always test failover before you need it.**

> **11. Always test migration before production cutover.**

> **12. Security and least privilege are DBA responsibilities.**

> **13. Documentation is part of database administration.**

> **14. The application and database are one production system.**

> **15. A good DBA prevents incidents; a great DBA also knows how to recover from them.**

---

## 🎓 0.10-Lesson Course Complete

You now have the **foundation of MariaDB administration** from installation through production HA/DR.

The natural next step is **hands-on DBA labs**: instead of more theory, we can build a **MariaDB DBA practice environment on Windows + Linux** and work through realistic tickets such as **"MariaDB won't start," "disk full," "user can't connect," "database is slow," "replication lag," "failed backup," "migration," and "emergency failover."**
