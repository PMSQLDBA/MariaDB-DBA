# Lesson 0.2 — MariaDB Installation on Linux

**Goal:** Install MariaDB correctly on Linux and understand what every major step is doing.

We'll cover:

* Ubuntu/Debian
* RHEL/Rocky/AlmaLinux
* Repository selection
* Package installation
* Service management
* Initial configuration
* Firewall
* Validation
* Common failures

> **Important:** The exact package/repository commands can vary by MariaDB release and OS version. In production, always use a MariaDB release supported for your OS.

---

# 1. Linux Installation Architecture

Think of the installation as:

```text
Linux Server
     │
     ├── MariaDB packages
     │       │
     │       ▼
     │   MariaDB Server
     │
     ├── Configuration
     │       │
     │       ▼
     │    my.cnf
     │
     ├── Data
     │       │
     │       ▼
     │   Data directory
     │
     └── Service
             │
             ▼
          mariadb
```

After installation, you want:

```text
MariaDB installed
      ↓
Service running
      ↓
Database accessible
      ↓
Configuration correct
      ↓
Security configured
      ↓
Monitoring/backup planned
```

---

# 2. Supported Linux Families

The most common enterprise families you'll encounter are:

### Debian family

```text
Ubuntu
Debian
```

Package manager:

```bash
apt
```

### Red Hat family

```text
RHEL
Rocky Linux
AlmaLinux
```

Package manager:

```bash
dnf
```

Older environments may use:

```bash
yum
```

---

# 3. Before Installing — Check the OS

First:

```bash
cat /etc/os-release
```

Example:

```text
NAME="Ubuntu"
VERSION="24.04..."
```

Or:

```text
NAME="Rocky Linux"
VERSION="9..."
```

Then determine:

```text
OS
Version
Architecture
```

Architecture:

```bash
uname -m
```

Common result:

```text
x86_64
```

or:

```text
aarch64
```

---

# 4. Check Available Resources

Before installing:

```bash
free -h
```

CPU:

```bash
nproc
```

Disk:

```bash
df -h
```

Filesystem inodes:

```bash
df -i
```

This gives you a quick picture of whether the machine is suitable.

---

# 5. Check Existing MariaDB/MySQL Installation

Before installing MariaDB on an existing server, check:

```bash
mariadb --version
```

Also:

```bash
mysql --version
```

And:

```bash
systemctl status mariadb
```

You may discover that another database server is already installed.

**Don't blindly install another server.**

---

# 6. Check Port 3306

Before installation:

```bash
ss -lntp | grep 3306
```

If something is already listening:

```text
LISTEN ... :3306
```

investigate it.

You don't want to discover during installation that another database is already using the port.

---

# 7. Ubuntu/Debian — Package Installation

On a Debian-family system, the basic package workflow is:

```bash
sudo apt update
```

Then:

```bash
sudo apt install mariadb-server mariadb-client
```

The package manager installs the required software and dependencies.

After installation:

```bash
mariadb --version
```

---

# 8. Start MariaDB

Check:

```bash
sudo systemctl status mariadb
```

If it isn't running:

```bash
sudo systemctl start mariadb
```

Then:

```bash
sudo systemctl status mariadb
```

You want something conceptually like:

```text
Active: active (running)
```

---

# 9. Enable MariaDB at Boot

You normally want MariaDB to start automatically after the server boots:

```bash
sudo systemctl enable mariadb
```

You can verify:

```bash
systemctl is-enabled mariadb
```

Expected:

```text
enabled
```

---

# 10. Connect Locally

Try:

```bash
sudo mariadb
```

If successful:

```text
MariaDB [(none)]>
```

Congratulations.

You have a running MariaDB server.

Test:

```sql
SELECT VERSION();
```

Then:

```sql
SHOW DATABASES;
```

Exit:

```sql
exit;
```

---

# 11. RHEL/Rocky/AlmaLinux

On RHEL-family systems, package management commonly uses:

```bash
dnf
```

The general workflow is:

```bash
sudo dnf install mariadb-server
```

However, **don't assume the OS repository contains the exact MariaDB release you want**.

This is where repository planning becomes important.

---

# 12. OS Repository vs MariaDB Repository

You may have:

```text
Operating system repository
          ↓
       MariaDB
```

or:

```text
MariaDB official repository
          ↓
       MariaDB
```

The second option may be appropriate when you need a specific MariaDB release series.

Before production installation, determine:

```text
Required MariaDB version
        ↓
Supported OS
        ↓
Appropriate repository
```

---

# 13. Why Repository Selection Matters

Suppose your application requires:

```text
MariaDB release X
```

but the OS repository provides:

```text
MariaDB release Y
```

Installing Y without checking compatibility can create problems later.

Repository selection affects:

* Version lifecycle
* Security updates
* Package dependencies
* Upgrade paths
* Supportability

---

# 14. RHEL-Based Service Management

After installation:

```bash
sudo systemctl enable --now mariadb
```

This combines:

```text
enable
+
start
```

Then verify:

```bash
sudo systemctl status mariadb
```

---

# 15. Linux Service Commands

Memorize these:

```bash
sudo systemctl start mariadb
```

Start.

```bash
sudo systemctl stop mariadb
```

Stop.

```bash
sudo systemctl restart mariadb
```

Restart.

```bash
sudo systemctl status mariadb
```

Status.

```bash
sudo systemctl enable mariadb
```

Start automatically at boot.

```bash
sudo systemctl disable mariadb
```

Don't start automatically at boot.

---

# 16. First Security Step

After installation, review the initial security configuration.

On many Linux installations you may encounter:

```bash
sudo mariadb-secure-installation
```

Depending on your MariaDB release/package, the exact prompts can differ.

This can help with tasks such as:

* Reviewing anonymous accounts
* Removing unnecessary test databases
* Reviewing administrative access
* Improving initial security

Don't blindly accept every prompt in production.

Understand what each change does.

---

# 17. Local Authentication

Modern MariaDB installations may use authentication mechanisms that allow administrative access through the local operating system account.

For example:

```bash
sudo mariadb
```

may work without entering a MariaDB root password.

This is not necessarily a problem.

It can be intentional.

The important question is:

> **What authentication mechanism is configured?**

---

# 18. Check Authentication

After connecting:

```sql
SELECT User, Host, plugin
FROM mysql.user;
```

This helps you understand how accounts authenticate.

You may see authentication plugins such as:

```text
unix_socket
mysql_native_password
```

depending on the MariaDB version and account configuration.

Don't change authentication mechanisms just because you see something unfamiliar.

---

# 19. Create an Administrative User

For production administration, you should understand individual DBA accounts.

For example:

```sql
CREATE USER 'dba_admin'@'localhost'
IDENTIFIED BY 'StrongRandomPassword';
```

Then assign the required administrative privileges according to your organization's security model.

Afterward:

```sql
SHOW GRANTS FOR 'dba_admin'@'localhost';
```

Avoid sharing a single administrator credential among multiple people.

---

# 20. Check the Data Directory

Run:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Example:

```text
/var/lib/mysql/
```

This is extremely important.

You need to know:

> **Where does MariaDB actually store its data?**

Don't assume the location.

---

# 21. Check Configuration Files

MariaDB configuration can be distributed across several files/directories depending on the Linux distribution.

Common locations include:

```text
/etc/my.cnf
/etc/mysql/my.cnf
/etc/mysql/mariadb.conf.d/
```

But don't assume your system uses exactly one of these.

You can inspect the installation and package configuration.

The goal is to know:

```text
Which configuration files are read?
```

---

# 22. Check Important Variables

Connect:

```bash
sudo mariadb
```

Then:

```sql
SHOW VARIABLES LIKE 'datadir';
```

```sql
SHOW VARIABLES LIKE 'port';
```

```sql
SHOW VARIABLES LIKE 'bind_address';
```

```sql
SHOW VARIABLES LIKE 'max_connections';
```

```sql
SHOW VARIABLES LIKE 'character_set_server';
```

These provide a basic configuration inventory.

---

# 23. Network Binding

One important setting is:

```text
bind-address
```

Conceptually:

```text
bind-address = 127.0.0.1
```

means MariaDB listens locally.

Whereas:

```text
bind-address = server_IP
```

can allow network clients to connect, assuming firewall and MariaDB permissions also permit it.

Don't expose MariaDB broadly just to make remote connections work.

---

# 24. Remote Connectivity

A common mistake:

```text
Application
     │
     X
     │
MariaDB
```

Admin says:

> "I'll just bind MariaDB to `0.0.0.0`."

That can expose MariaDB on every network interface.

Instead:

```text
Application subnet
       │
       ▼
 Firewall
       │
       ▼
 MariaDB
```

Use a restricted network design.

---

# 25. Linux Firewall

Depending on your distribution, you may use:

```text
firewalld
ufw
nftables
```

For example, on systems using UFW:

```bash
sudo ufw status
```

Don't blindly open:

```text
3306/tcp
```

to the entire internet.

If remote access is required, restrict it to the application network/source addresses.

---

# 26. SELinux

On RHEL-family systems, you may encounter:

```text
SELinux
```

Check:

```bash
getenforce
```

Possible output:

```text
Enforcing
```

Don't immediately disable SELinux because MariaDB isn't behaving as expected.

Instead investigate:

* SELinux denials
* File contexts
* Service policies
* Correct MariaDB paths

Disabling security controls is not a proper troubleshooting strategy.

---

# 27. File Ownership

MariaDB's files need appropriate ownership and permissions.

You can inspect:

```bash
ls -ld /var/lib/mysql
```

and:

```bash
ls -l /var/lib/mysql
```

The exact owner/group depends on your installation.

Never casually run:

```bash
chmod -R 777 /var/lib/mysql
```

That is **not** a solution.

---

# 28. Logs

Find the configured error log:

```sql
SHOW VARIABLES LIKE 'log_error';
```

Then inspect it from Linux.

For systemd-managed environments, also use:

```bash
journalctl -u mariadb
```

For recent entries:

```bash
journalctl -u mariadb -n 100
```

Follow live messages:

```bash
journalctl -u mariadb -f
```

---

# 29. Common Installation Failure #1

### MariaDB won't start.

Don't immediately reinstall.

Run:

```bash
systemctl status mariadb
```

Then:

```bash
journalctl -u mariadb -n 100
```

Then inspect the MariaDB error log.

Possible cause:

```text
Configuration
Permissions
Disk
Port
InnoDB
SELinux
```

---

# 30. Common Failure #2 — Port Already Used

Check:

```bash
ss -lntp | grep 3306
```

You may discover:

```text
Another MariaDB
MySQL
Container
Other application
```

Find the process before changing anything.

---

# 31. Common Failure #3 — Disk Full

Check:

```bash
df -h
```

Then:

```bash
df -i
```

If the database filesystem is full:

```text
STOP
   ↓
Investigate
   ↓
Identify consumer
   ↓
Safely free space
   ↓
Verify MariaDB
```

Don't delete random database files.

---

# 32. Common Failure #4 — Permission Denied

If MariaDB can't access a file:

```text
Permission denied
```

Check:

```bash
ls -l
```

and:

```bash
namei -l /path/to/file
```

On SELinux systems also investigate:

```text
SELinux context/policy
```

Do not simply do:

```bash
chmod 777
```

---

# 33. Common Failure #5 — Wrong Configuration

Suppose you edited:

```text
my.cnf
```

and MariaDB won't start.

Your troubleshooting path:

```text
Configuration changed
       ↓
MariaDB fails
       ↓
Check error log
       ↓
Identify invalid setting
       ↓
Correct/revert
       ↓
Restart
       ↓
Verify
```

This is why configuration changes should be made one at a time.

---

# 34. Installation Validation Checklist

After installation:

```text
☐ Correct OS
☐ Correct MariaDB version
☐ Correct repository
☐ Packages installed
☐ Service running
☐ Service enabled
☐ Local login works
☐ Port verified
☐ Data directory verified
☐ Configuration identified
☐ Error log checked
☐ Firewall reviewed
☐ SELinux reviewed where applicable
☐ Authentication reviewed
☐ Administrative access tested
☐ Backup plan created
```

---

# 35. Lab Exercise

If you have an Ubuntu/Debian VM, practice:

```bash
cat /etc/os-release
```

```bash
uname -m
```

```bash
free -h
```

```bash
df -h
```

Then install MariaDB using the appropriate package/repository for your OS and desired supported version.

After installation:

```bash
sudo systemctl status mariadb
```

Then:

```bash
sudo mariadb
```

Inside MariaDB:

```sql
SELECT VERSION();

SHOW DATABASES;

SHOW VARIABLES LIKE 'datadir';

SHOW VARIABLES LIKE 'port';

SHOW VARIABLES LIKE 'bind_address';

SELECT User, Host, plugin
FROM mysql.user;
```

Exit:

```sql
exit;
```

Then:

```bash
sudo journalctl -u mariadb -n 50
```

---

# 36. Your DBA Challenge

Don't just run commands.

Answer these:

### Q1

Why should you check `/etc/os-release` before installing?

### Q2

Why should you verify the MariaDB version before production installation?

### Q3

Why shouldn't you automatically expose port 3306?

### Q4

Why shouldn't you use `chmod 777` on the MariaDB data directory?

### Q5

Why should you check `journalctl` when MariaDB won't start?

### Q6

Why should you know the `datadir`?

### Q7

What is the difference between:

```bash
systemctl start mariadb
```

and:

```bash
systemctl enable mariadb
```

### Q8

Why shouldn't you disable SELinux just because MariaDB has an access problem?

---

# Answers

### Q1

Because installation packages and supported MariaDB versions depend on the OS.

### Q2

Because applications, replication, configuration, and upgrade paths can depend on the MariaDB version.

### Q3

Because that can expose your database to unauthorized network access.

### Q4

Because it creates dangerous filesystem permissions and violates security principles.

### Q5

Because systemd/journal logs can reveal why the service failed.

### Q6

Because it tells you where MariaDB stores its database files and is critical for storage, backup, migration, and troubleshooting.

### Q7

`start` starts MariaDB **now**.

`enable` configures MariaDB to start automatically during boot.

### Q8

Because SELinux is a security control. You should diagnose and correctly configure it rather than simply disabling it.

---

# ⭐ Lesson 2 Golden Rule

> **Never consider MariaDB "installed" just because the package installation succeeded.**

Your real installation completion point is:

```text
Packages
   ↓
Service
   ↓
Configuration
   ↓
Security
   ↓
Connectivity
   ↓
Storage
   ↓
Logs
   ↓
Backup
   ↓
Validation
```

That's the difference between **installing MariaDB** and **installing MariaDB like a DBA**.

---

# Next — Lesson 0.3

## **MariaDB Installation on Windows**

We'll cover the Windows Server DBA workflow in the same format:

```text
Windows Server
      ↓
Prerequisites
      ↓
MariaDB MSI/installer
      ↓
Installation options
      ↓
Data directory
      ↓
my.ini
      ↓
Windows Service
      ↓
PowerShell
      ↓
Firewall
      ↓
Service accounts
      ↓
Remote connectivity
      ↓
Security
      ↓
Validation
      ↓
Troubleshooting
```

And we'll specifically compare **Windows vs Linux MariaDB administration**, because that difference is very important for a DBA.
