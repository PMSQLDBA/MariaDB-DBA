**MariaDB installation, configuration, patching, upgrades, migration, and OS-specific administration**.

# MariaDB DBA — Installation, Configuration, Patching & Migration

**10 practical lessons**:

| Lesson | Topic                                          |
| ------ | ---------------------------------------------- |
| 1      | MariaDB Installation Fundamentals              |
| 2      | MariaDB Installation on Linux                  |
| 3      | MariaDB Installation on Windows                |
| 4      | MariaDB Configuration & `my.cnf` / `my.ini`    |
| 5      | Security Hardening & Initial Configuration     |
| 6      | Patching MariaDB on Linux                      |
| 7      | Patching MariaDB on Windows                    |
| 8      | Patching on Other OS / Platform Considerations |
| 9      | MariaDB Migration & Upgrade                    |
| 10     | Production Patch & Migration Runbook           |

---

# Lesson 0.1 — MariaDB Installation Fundamentals

## Goal

By the end of this lesson, you should understand:

* What gets installed
* MariaDB packages/components
* Installation methods
* OS differences
* Version selection
* Repository/package management
* Installation planning
* Post-installation validation

---

# 1. What Does "Install MariaDB" Actually Mean?

A database installation isn't simply:

```text
Download
   ↓
Next
   ↓
Finish
```

A DBA needs to think about:

```text
Operating System
       ↓
MariaDB Version
       ↓
Packages
       ↓
Configuration
       ↓
Data Directory
       ↓
Service
       ↓
Security
       ↓
Backup
       ↓
Monitoring
```

A proper installation produces a **manageable database server**, not merely a running process.

---

# 2. MariaDB Server Architecture

At a simplified level:

```text
                MariaDB Server
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     SQL Layer   Storage Engine   Security
        │            │            │
        ▼            ▼            ▼
    Queries       InnoDB        Users
    Optimizer     Files          Grants
```

The server provides:

* SQL processing
* Authentication
* Authorization
* Transaction management
* Storage-engine management
* Replication
* Logging
* Backup-related functionality
* Client connectivity

---

# 3. Main Components

A MariaDB installation can contain several components.

### Server

The actual database engine:

```text
mariadbd
```

Depending on MariaDB version/platform, you may encounter the older `mysqld` naming as well.

### Client

Command-line client:

```bash
mariadb
```

### Configuration

Linux commonly uses:

```text
my.cnf
```

Windows commonly uses:

```text
my.ini
```

### Data directory

Contains database-related files.

Conceptually:

```text
MariaDB
   │
   ├── Configuration
   ├── Data
   ├── Logs
   └── Runtime/service
```

---

# 4. Installation Methods

MariaDB can be installed using different methods.

Common approaches include:

### Linux packages

For example:

```text
APT
DNF/YUM
```

### MariaDB repositories

Useful when you need a specific MariaDB release/version stream.

### Windows installer

Typically used for Windows environments.

### Containers

For example:

```text
Docker / OCI container
```

### Source compilation

Used in specialized environments.

For most enterprise DBAs:

> **Use the vendor-supported package/repository method unless there is a specific reason not to.**

---

# 5. Why Version Selection Matters

Before installation, determine:

```text
Which MariaDB version?
Which OS?
Which architecture?
Which application version?
Which storage engine?
Which replication technology?
Which support lifecycle?
```

Don't simply install:

```text
"the newest thing available"
```

without checking application compatibility.

---

# 6. Version Strategy

Imagine you have:

```text
Application
     │
     ▼
MariaDB 10.x
```

and you're considering:

```text
MariaDB newer release
```

You need to determine:

```text
Application compatibility
       ↓
SQL compatibility
       ↓
Configuration compatibility
       ↓
Authentication compatibility
       ↓
Replication compatibility
       ↓
Backup/restore compatibility
       ↓
Upgrade path
```

A database upgrade is an engineering change, not just a package update.

---

# 7. Production vs. Lab Installation

A lab server might be:

```text
1 VM
2 CPU
4 GB RAM
50 GB disk
```

Production might be:

```text
16+ CPU
64+ GB RAM
High-performance storage
Redundant networking
Monitoring
Backup
HA
```

Therefore:

> **Never copy a lab configuration directly into production.**

---

# 8. Plan the Filesystem

Before installation, decide where database data belongs.

Conceptually:

```text
Operating System
       │
       ├── OS filesystem
       │
       ├── MariaDB data
       │
       ├── Logs
       │
       └── Backup storage
```

For larger systems, database storage should be deliberately designed rather than simply accepting whatever defaults the installer chooses.

---

# 9. Storage Planning

Database performance depends heavily on storage.

Think about:

```text
Capacity
IOPS
Latency
Throughput
Reliability
Filesystem
RAID/storage redundancy
Backup space
Growth
```

For example:

```text
Database = 500 GB
```

does **not** mean:

```text
Disk = 500 GB
```

You need room for:

```text
Database
+
Temporary space
+
Logs
+
Binary logs
+
Backups
+
Growth
+
Maintenance operations
```

---

# 10. Memory Planning

MariaDB needs memory for:

```text
Operating system
MariaDB buffer pool
Connections
Temporary operations
Caches
Other services
```

A common DBA mistake is:

> "The server has 64 GB RAM, so I'll give MariaDB 64 GB."

Don't do that.

The OS needs memory too.

---

# 11. CPU Planning

MariaDB performance isn't simply:

```text
More CPUs = always faster
```

You need to consider:

* Query concurrency
* Query complexity
* Storage latency
* Parallelism
* Connections
* Background tasks
* Replication

CPU should be sized based on workload.

---

# 12. Network Planning

Before installation, determine:

```text
Application → MariaDB
```

and, if applicable:

```text
MariaDB → Replica
```

or:

```text
MariaDB Node 1 ↔ Node 2 ↔ Node 3
```

You should know:

* IP address
* DNS/hostname
* Database port
* Firewall requirements
* Application source addresses
* Replication traffic
* Monitoring access

---

# 13. Default Port

MariaDB commonly uses:

```text
TCP 3306
```

After installation, verify what the server is listening on.

Linux:

```bash
ss -lntp
```

Windows PowerShell:

```powershell
Get-NetTCPConnection -LocalPort 3306
```

Don't assume that because the installation completed, the service is reachable.

---

# 14. Installation Lifecycle

A professional installation follows:

```text
Requirement
    ↓
Version selection
    ↓
OS preparation
    ↓
Storage preparation
    ↓
Package installation
    ↓
Configuration
    ↓
Service startup
    ↓
Security hardening
    ↓
Validation
    ↓
Backup
    ↓
Monitoring
```

---

# 15. Post-Installation Validation

After installing MariaDB, verify:

### Service

Is MariaDB running?

### Version

```sql
SELECT VERSION();
```

### Connectivity

Can you connect?

### Databases

```sql
SHOW DATABASES;
```

### Port

Is the expected port listening?

### Configuration

Are the intended settings active?

### Storage

Is the data directory correct?

### Logs

Are there errors?

---

# 16. First DBA Validation

After installation:

```sql
SELECT VERSION();
```

Then:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Then:

```sql
SHOW VARIABLES LIKE 'port';
```

Then:

```sql
SHOW VARIABLES LIKE 'socket';
```

And:

```sql
SHOW VARIABLES LIKE 'character_set%';
```

These commands begin telling you what kind of MariaDB installation you actually have.

---

# 17. Linux Installation — High-Level Process

For a Linux server:

```text
Linux OS
   ↓
Configure repositories
   ↓
Install MariaDB packages
   ↓
Enable service
   ↓
Start service
   ↓
Validate
   ↓
Secure configuration
```

For Debian/Ubuntu-style systems, package management commonly uses:

```bash
apt
```

For RHEL-family systems:

```bash
dnf
```

or, on older systems:

```bash
yum
```

The exact commands depend on the OS and MariaDB release.

---

# 18. Windows Installation — High-Level Process

Windows follows a different model:

```text
Windows Server
      ↓
MariaDB installer/package
      ↓
Install binaries
      ↓
Configure data directory
      ↓
Configure service
      ↓
Configure networking
      ↓
Start Windows service
      ↓
Validate
```

The MariaDB service can be managed through:

```text
Services
```

or PowerShell/service-management commands.

---

# 19. Linux vs. Windows

| Area               | Linux                           | Windows                            |
| ------------------ | ------------------------------- | ---------------------------------- |
| Package management | APT/DNF/YUM                     | Installer/package tools            |
| Service            | systemd commonly                | Windows Service                    |
| Config             | `my.cnf`                        | `my.ini`                           |
| Logs               | Files/journal depending config  | Files/Event Viewer depending setup |
| Firewall           | firewalld/ufw/iptables/nftables | Windows Firewall                   |
| Automation         | Bash/Ansible/etc.               | PowerShell/automation tools        |
| Filesystem         | Linux filesystem                | NTFS commonly                      |

The **DBA concepts are the same**.

The operating-system administration is different.

---

# 20. "Other OS"

For other platforms, the exact installation mechanism can vary significantly.

The principle remains:

```text
Check supported platform
       ↓
Use supported MariaDB distribution method
       ↓
Install correct version
       ↓
Configure
       ↓
Validate
       ↓
Secure
```

Don't blindly apply Linux commands to another operating system.

---

# 21. Configuration Comes After Installation

Installation gives you:

```text
MariaDB = installed
```

Configuration makes it:

```text
MariaDB = suitable for workload
```

Important areas include:

```text
Networking
Memory
Connections
Character sets
Collation
Logging
Binary logging
Replication
Storage
Security
Timeouts
SQL behavior
```

We'll cover these in detail in **Lesson 4**.

---

# 22. Patching vs. Upgrading

This distinction is essential.

### Patching

Usually means applying fixes within an established release/version line.

Example conceptually:

```text
Version A
   ↓
Version A + security/bug fixes
```

### Upgrade

Moving between larger release lines or versions.

Conceptually:

```text
Version A
   ↓
Version B
```

An upgrade may introduce:

* Compatibility changes
* New features
* Configuration changes
* SQL behavior changes
* Authentication changes
* Replication considerations

Therefore:

> **Patching and upgrading require different levels of testing.**

---

# 23. Patch Management Lifecycle

A proper patch cycle:

```text
Vendor release
      ↓
Review security/bug fixes
      ↓
Check compatibility
      ↓
Test in lab
      ↓
Test in staging
      ↓
Backup
      ↓
Maintenance window
      ↓
Patch production
      ↓
Validate
      ↓
Monitor
```

---

# 24. Never Patch Production First

Bad:

```text
Vendor releases update
        ↓
Production
        ↓
"Let's see what happens."
```

Better:

```text
Vendor update
      ↓
Development
      ↓
Test
      ↓
Staging
      ↓
Production
```

If you have replication/HA, you can often design a rolling maintenance strategy—but that still requires testing and a clear failover plan.

---

# 25. Migration

MariaDB migration means moving a database workload from:

```text
Source
  ↓
Target
```

Examples:

```text
Server A → Server B
Windows → Linux
Linux → Linux
Older MariaDB → Newer MariaDB
Physical server → VM
VM → Cloud
MySQL → MariaDB
```

Migration planning depends heavily on the source and target.

---

# 26. Basic Migration Process

A generic migration:

```text
Assess source
     ↓
Prepare target
     ↓
Install MariaDB
     ↓
Configure target
     ↓
Transfer schema/data
     ↓
Validate
     ↓
Test application
     ↓
Cutover
     ↓
Monitor
```

---

# 27. Migration Is Not Just Data Copy

You must consider:

```text
Data
+
Users
+
Privileges
+
Stored procedures
+
Triggers
+
Events
+
Views
+
Indexes
+
Configuration
+
Replication
+
Application connection strings
```

A migration that copies tables but forgets users and permissions isn't complete.

---

# 28. Migration Validation

After migration, compare:

```text
Database count
Table count
Row counts
Indexes
Constraints
Users
Privileges
Stored routines
Triggers
Events
Application functionality
Performance
```

You should have a documented validation checklist.

---

# 29. Cutover

Cutover is the moment production traffic moves from:

```text
OLD
```

to:

```text
NEW
```

A controlled cutover:

```text
Prepare target
      ↓
Synchronize data
      ↓
Freeze/stop writes if required
      ↓
Final synchronization
      ↓
Validate target
      ↓
Change application endpoint
      ↓
Test
      ↓
Monitor
```

---

# 30. Rollback Plan

Every production migration needs:

> **"What if the new server doesn't work?"**

Example:

```text
OLD SERVER
    │
    │ production
    ▼

NEW SERVER
    │
    │ problem
    ▼

ROLLBACK
    │
    ▼
OLD SERVER
```

But rollback isn't always simple if new writes have already occurred.

Therefore:

> **Your rollback plan must account for data written after cutover.**

---

# 31. Your First Practical Assignment

For **Lesson 1**, create this document for your lab environment:

```text
MariaDB Installation Plan
-------------------------

OS:
MariaDB version:
CPU:
RAM:
Disk:
Data directory:
Log directory:
Port:
Hostname:
IP:
Firewall:
Backup location:
Monitoring:
Application:
Expected database size:
Expected growth:
```

Then answer:

### Question 1

Why shouldn't you install MariaDB before selecting the required version?

### Question 2

What's the difference between patching and upgrading?

### Question 3

Why should production patches be tested before deployment?

### Question 4

Why isn't copying database files alone necessarily a complete migration?

### Question 5

What must a DBA have before performing a production migration?

**Answer:** A tested **recovery/rollback plan**, not just a migration plan.

---

# Lesson 1 — Golden Rule

> **Installation is the beginning of database administration, not the end.**

A professional DBA thinks:

```text
Install
  ↓
Configure
  ↓
Secure
  ↓
Monitor
  ↓
Backup
  ↓
Patch
  ↓
Upgrade
  ↓
Migrate
  ↓
Recover
```

## Next: Lesson 0.2 — MariaDB Installation on Linux

We'll do it **OS-by-OS and command-by-command**, including:

**Ubuntu/Debian → RHEL/Rocky/AlmaLinux → repository setup → package installation → service management → data directory → firewall → initial login → validation → common installation failures.**
