# Lesson 0.3 — MariaDB Installation on Windows

Today we'll learn **MariaDB installation and initial administration on Windows Server**, using the same DBA-focused format.

> **Goal:** By the end, you should be able to install MariaDB, configure the Windows service, locate the data/configuration files, configure networking and firewall access, validate the installation, and troubleshoot common failures.

---

# 1. Windows MariaDB Architecture

Think of the Windows installation like this:

```text
                 Windows Server
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   MariaDB Server   Configuration    Data
        │              │              │
        ▼              ▼              ▼
    mariadbd         my.ini       Data Directory
        │
        ▼
 Windows Service
        │
        ▼
     Port 3306
```

Linux and Windows use the same MariaDB database engine.

The difference is mainly in **OS administration**.

---

# 2. What You Need Before Installation

Before installing MariaDB, collect:

```text
OS:
Windows Server version:
MariaDB version:
CPU:
RAM:
Disk:
Hostname:
IP address:
Data directory:
Port:
Application server IPs:
Backup location:
```

For production, also determine:

```text
RPO:
RTO:
Expected database size:
Expected growth:
HA/replication requirement:
Maintenance window:
```

---

# 3. Check Windows Version

Open PowerShell:

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsArchitecture
```

You want to know:

```text
Windows edition
Windows version
Architecture
```

Architecture matters because you need a compatible MariaDB build.

---

# 4. Check CPU and Memory

PowerShell:

```powershell
Get-CimInstance Win32_Processor |
Select-Object NumberOfLogicalProcessors
```

Memory:

```powershell
Get-CimInstance Win32_ComputerSystem |
Select-Object TotalPhysicalMemory
```

You don't necessarily need exact values during a simple lab installation, but a production DBA should document the resources.

---

# 5. Check Disk Space

PowerShell:

```powershell
Get-PSDrive -PSProvider FileSystem
```

For example:

```text
Name   Used     Free
C      ...      ...
D      ...      ...
```

Think carefully about where database data should live.

Don't automatically put a large production database on the OS drive.

---

# 6. Check Existing MariaDB/MySQL

Before installation:

```powershell
Get-Service | Where-Object {
    $_.Name -match 'MariaDB|MySQL'
}
```

You might discover:

```text
MariaDB
MySQL
MariaDB106
MariaDB105
```

Do not install a second database server until you understand what is already there.

---

# 7. Check Port 3306

PowerShell:

```powershell
Get-NetTCPConnection -LocalPort 3306 -ErrorAction SilentlyContinue
```

If there is output, something is already using port 3306.

You can investigate:

```powershell
Get-NetTCPConnection -LocalPort 3306 |
Select-Object LocalAddress,LocalPort,State,OwningProcess
```

Then:

```powershell
Get-Process -Id <PID>
```

Replace `<PID>` with the actual process ID.

---

# 8. MariaDB Windows Installation

MariaDB provides Windows installation packages.

The normal process is:

```text
Download appropriate installer
        ↓
Run installer
        ↓
Select components
        ↓
Choose data directory
        ↓
Configure service
        ↓
Configure root/admin credentials
        ↓
Configure networking
        ↓
Install
        ↓
Validate
```

For production, don't just click **Next → Next → Finish**.

Record the choices you make.

---

# 9. Installation Components

Depending on the installer/version, you may encounter components such as:

```text
MariaDB Server
Client programs
Development components
Documentation
Additional tools
```

For a database server, the important component is:

```text
MariaDB Server
```

The client tools are also useful for administration.

---

# 10. Data Directory

One of the most important installation decisions is:

> **Where will MariaDB store its data?**

You may have something like:

```text
C:\Program Files\MariaDB\...
```

for binaries, while database data is stored elsewhere.

For a production environment, separating:

```text
OS
MariaDB binaries
Database data
Backups
```

can make administration easier.

For example:

```text
C:\
 └── Windows / Applications

D:\
 └── MariaDBData

E:\
 └── MariaDBBackups
```

The actual layout depends on your storage architecture.

---

# 11. Don't Confuse Program Files With Data

Conceptually:

```text
MariaDB installation
        │
        ├── Binaries
        │
        ├── Configuration
        │
        └── Data
```

The executable files and database data serve different purposes.

This matters during:

* Upgrades
* Migration
* Backup
* Disaster recovery
* Troubleshooting

---

# 12. Windows Service

On Windows, MariaDB normally runs as a **Windows service**.

You can view services with:

```powershell
Get-Service | Where-Object {
    $_.Name -match 'MariaDB|MySQL'
}
```

You may see something like:

```text
Status   Name
------   ----
Running  MariaDB
```

The exact service name depends on the installation.

---

# 13. Start the Service

PowerShell:

```powershell
Start-Service -Name MariaDB
```

If the service has a different name, use that actual service name.

Check:

```powershell
Get-Service -Name MariaDB
```

---

# 14. Stop the Service

```powershell
Stop-Service -Name MariaDB
```

Check:

```powershell
Get-Service -Name MariaDB
```

Expected:

```text
Stopped
```

**Production warning:** Never stop a production database casually.

First determine:

```text
Why?
Impact?
Connections?
Replication?
HA?
Maintenance window?
Rollback?
```

---

# 15. Restart the Service

```powershell
Restart-Service -Name MariaDB
```

Again, don't use restart as your first troubleshooting action.

First inspect the problem.

---

# 16. Automatic Startup

A production MariaDB service should generally start automatically after Windows reboots.

Check:

```powershell
Get-Service -Name MariaDB |
Select-Object Name,Status,StartType
```

You generally want:

```text
StartType = Automatic
```

The exact service configuration should match your organization's standards.

---

# 17. MariaDB Configuration — `my.ini`

On Windows, you will commonly encounter:

```text
my.ini
```

This is the primary configuration file you should learn to understand.

Conceptually:

```text
my.ini
   │
   ├── Server settings
   ├── Networking
   ├── Logging
   ├── Memory
   ├── Storage
   └── Replication
```

The exact path depends on how MariaDB was installed.

---

# 18. Find the Active Configuration

Don't assume:

```text
C:\my.ini
```

is the configuration being used.

You need to determine which configuration files MariaDB reads.

One useful approach is to inspect the installation/service configuration and MariaDB's startup behavior.

The DBA principle is:

> **Always verify the active configuration instead of assuming the file you edited is being used.**

---

# 19. Important Configuration Categories

You'll eventually configure things such as:

```text
[mariadb]

port
bind-address

datadir

max_connections

character-set-server
collation-server

innodb_buffer_pool_size

log_error

slow_query_log

server_id
```

Don't copy these blindly into production.

We'll study configuration properly in **Lesson 4**.

---

# 20. Check the Server From SQL

Connect using the MariaDB client.

Then:

```sql
SELECT VERSION();
```

Check data directory:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Port:

```sql
SHOW VARIABLES LIKE 'port';
```

Bind address:

```sql
SHOW VARIABLES LIKE 'bind_address';
```

Error log:

```sql
SHOW VARIABLES LIKE 'log_error';
```

Character set:

```sql
SHOW VARIABLES LIKE 'character_set_server';
```

---

# 21. Check the Windows Firewall

MariaDB normally uses:

```text
TCP 3306
```

Check firewall rules:

```powershell
Get-NetFirewallRule |
Where-Object {$_.Enabled -eq 'True'} |
Select-Object DisplayName,Direction,Action
```

You don't need to memorize every firewall command.

The important concept is:

```text
MariaDB listening
       ≠
MariaDB reachable
```

Both MariaDB configuration **and** Windows Firewall matter.

---

# 22. Creating a Firewall Rule

If remote application access is required, create a narrowly scoped firewall rule.

Conceptually:

```text
Application Server
       │
       │ TCP 3306
       ▼
Windows Firewall
       │
       ▼
MariaDB
```

Avoid:

```text
Internet
   │
   │ TCP 3306
   ▼
MariaDB
```

A production firewall rule should restrict the source network where practical.

---

# 23. Testing the Port

From the application server or another authorized machine:

```powershell
Test-NetConnection <MariaDB-IP> -Port 3306
```

For example:

```powershell
Test-NetConnection 10.10.10.20 -Port 3306
```

Look for:

```text
TcpTestSucceeded : True
```

This tests network connectivity.

It does **not** prove MariaDB authentication works.

---

# 24. Network Testing Layers

When an application can't connect, test in this order:

```text
DNS/hostname
     ↓
Network routing
     ↓
Firewall
     ↓
Port 3306
     ↓
MariaDB listening
     ↓
MariaDB user/host permissions
     ↓
Authentication
     ↓
Application
```

This prevents random troubleshooting.

---

# 25. MariaDB User Host Restrictions

Suppose:

```sql
'appuser'@'localhost'
```

exists.

Your application connects from:

```text
10.10.10.50
```

That doesn't necessarily mean the user is allowed to connect remotely.

You need an appropriate account such as:

```sql
'appuser'@'10.10.10.50'
```

or another appropriately restricted host pattern.

This is one of the most common DBA concepts beginners miss.

---

# 26. Windows Service Account

MariaDB can run under a Windows service account.

For production, understand:

```text
Which account runs MariaDB?
What permissions does it have?
What directories can it access?
What happens when passwords/policies change?
```

Don't give a database service unnecessary Windows privileges.

Use an appropriate service identity according to your organization's security requirements.

---

# 27. Windows Event Viewer

Windows administrators should know:

```text
Event Viewer
```

You can launch:

```powershell
eventvwr.msc
```

MariaDB's own error log remains important, but Windows Event Viewer can also help diagnose OS/service-related issues.

---

# 28. MariaDB Error Log

From SQL:

```sql
SHOW VARIABLES LIKE 'log_error';
```

Then inspect the configured file.

When MariaDB fails to start, the error log is one of your first troubleshooting locations.

Typical sequence:

```text
Service fails
    ↓
Check service status
    ↓
Check MariaDB error log
    ↓
Check Event Viewer
    ↓
Check configuration
    ↓
Check filesystem
    ↓
Check permissions
```

---

# 29. Common Failure — Service Won't Start

Suppose:

```powershell
Start-Service MariaDB
```

fails.

Don't immediately reinstall.

Check:

```powershell
Get-Service MariaDB
```

Then investigate the MariaDB error log.

Also check:

```text
my.ini
data directory
permissions
disk space
port conflicts
Windows service configuration
```

---

# 30. Common Failure — Port Conflict

You run:

```powershell
Test-NetConnection localhost -Port 3306
```

but another application is using 3306.

Check:

```powershell
Get-NetTCPConnection -LocalPort 3306
```

Then identify the process.

The solution may be:

```text
Stop conflicting service
```

or:

```text
Use another MariaDB port
```

But don't change MariaDB's port without understanding application and firewall consequences.

---

# 31. Common Failure — Permission Problems

Suppose MariaDB can't read its data directory.

Check:

```text
MariaDB service account
        ↓
Filesystem permissions
        ↓
Data directory
```

Don't solve it with:

```text
Everyone → Full Control
```

That's a serious security mistake.

Grant only the permissions required.

---

# 32. Common Failure — Full Disk

Check:

```powershell
Get-PSDrive -PSProvider FileSystem
```

If the database drive is full:

```text
STOP
 ↓
Identify what's consuming space
 ↓
Check logs
 ↓
Check backups
 ↓
Check binary logs
 ↓
Check database growth
 ↓
Safely resolve
```

Never randomly delete files from the MariaDB data directory.

---

# 33. Common Failure — Remote Connection Doesn't Work

Suppose:

```text
Application
     │
     X
     │
MariaDB
```

Check:

### Step 1

Is MariaDB running?

```powershell
Get-Service MariaDB
```

### Step 2

Is it listening?

```powershell
Get-NetTCPConnection -LocalPort 3306
```

### Step 3

Does Windows Firewall allow the connection?

### Step 4

Is `bind-address` appropriate?

### Step 5

Does the MariaDB account permit the application's source host?

### Step 6

Does authentication succeed?

---

# 34. Windows vs Linux

Here's your DBA comparison:

| Task           | Linux                       | Windows                   |
| -------------- | --------------------------- | ------------------------- |
| Service status | `systemctl status mariadb`  | `Get-Service`             |
| Start          | `systemctl start mariadb`   | `Start-Service`           |
| Stop           | `systemctl stop mariadb`    | `Stop-Service`            |
| Restart        | `systemctl restart mariadb` | `Restart-Service`         |
| Configuration  | `my.cnf`                    | `my.ini`                  |
| Firewall       | UFW/firewalld/nftables      | Windows Firewall          |
| Logs           | journal/files               | MariaDB logs/Event Viewer |
| Processes      | `ps`, `top`                 | Task Manager/PowerShell   |
| Network        | `ss`                        | `Get-NetTCPConnection`    |
| Disk           | `df -h`                     | `Get-PSDrive`             |

The **MariaDB SQL administration is largely the same**.

The OS commands are different.

---

# 35. Installation Validation

After installation, verify all of this:

```text
☐ MariaDB version
☐ Windows version
☐ MariaDB service
☐ Automatic startup
☐ Data directory
☐ Configuration file
☐ Port
☐ Bind address
☐ Firewall
☐ Error log
☐ Authentication
☐ Local connectivity
☐ Remote connectivity if required
☐ Filesystem permissions
☐ Backup plan
```

---

# 36. Your Windows Lab

If you have a Windows VM, perform:

### Step 1

Check OS:

```powershell
Get-ComputerInfo |
Select-Object WindowsProductName,WindowsVersion,OsArchitecture
```

### Step 2

Check MariaDB services:

```powershell
Get-Service |
Where-Object {$_.Name -match 'MariaDB|MySQL'}
```

### Step 3

Check port:

```powershell
Get-NetTCPConnection -LocalPort 3306 -ErrorAction SilentlyContinue
```

### Step 4

Install the appropriate MariaDB release.

### Step 5

Verify:

```powershell
Get-Service MariaDB
```

### Step 6

Connect:

```text
mariadb
```

or use the installed MariaDB client.

### Step 7

Run:

```sql
SELECT VERSION();

SHOW VARIABLES LIKE 'datadir';

SHOW VARIABLES LIKE 'port';

SHOW VARIABLES LIKE 'bind_address';

SHOW VARIABLES LIKE 'log_error';
```

---

# 37. DBA Challenge

Answer these before moving on.

### Q1

What is the Windows equivalent of:

```bash
systemctl status mariadb
```

### Q2

What is commonly used as the MariaDB configuration filename on Windows?

### Q3

What PowerShell command can test TCP port 3306?

### Q4

MariaDB is running locally, but a remote application can't connect. Name **four things** you would check.

### Q5

Why shouldn't you simply open port 3306 to all networks?

### Q6

Why is the MariaDB service account important?

### Q7

What is the difference between:

```text
MariaDB listening on 3306
```

and:

```text
Application successfully connecting to MariaDB
```

---

# Answers

### Q1

```powershell
Get-Service MariaDB
```

### Q2

```text
my.ini
```

### Q3

```powershell
Test-NetConnection <server> -Port 3306
```

### Q4

For example:

```text
MariaDB listening
Windows Firewall
bind-address
MariaDB user/host permissions
```

### Q5

It unnecessarily exposes the database to unauthorized networks.

### Q6

Because the service account determines what operating-system resources MariaDB can access.

### Q7

Listening only proves that MariaDB has a network socket open. Successful connectivity also requires network routing, firewall access, correct binding, MariaDB host permissions, and authentication.

---

# ⭐ Lesson 3 Golden Rule

> **On Windows, MariaDB administration is a combination of SQL administration + Windows Server administration.**

Think:

```text
                 WINDOWS DBA
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     Windows Layer          MariaDB Layer
          │                     │
      Services               SQL
      Firewall              Users
      Storage               Grants
      PowerShell            Tables
      Event Logs            Performance
          │                     │
          └──────────┬──────────┘
                     ▼
              Production DBA
```

You should now be comfortable with the **installation concepts on Linux and Windows**.

## Next: Lesson 0.4 — MariaDB Configuration

We'll go deep into:

```text
my.cnf / my.ini
     ↓
datadir
port
bind-address
max_connections
innodb_buffer_pool_size
character set
collation
error log
slow query log
binary log
server_id
timeouts
temporary tables
file locations
```

and, most importantly, **how to change configuration safely without breaking production**.
