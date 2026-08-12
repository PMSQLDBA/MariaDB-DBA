# Lesson 0.5 — MariaDB Security Hardening & User Administration

This lesson is about making MariaDB **secure enough for production**, not merely getting it running.

We will cover:

* Authentication
* Users
* `root`
* `CREATE USER`
* `GRANT`
* `REVOKE`
* Least privilege
* Host restrictions
* Passwords
* Remote access
* TLS
* Firewall
* Linux vs Windows
* Security auditing
* Common DBA mistakes

---

# 1. MariaDB Security Architecture

Think of access as several layers:

```text
                 Client
                   │
                   ▼
             Network/Firewall
                   │
                   ▼
             MariaDB Listener
                   │
                   ▼
          User + Host Matching
                   │
                   ▼
             Authentication
                   │
                   ▼
                Grants
                   │
                   ▼
             Database Objects
```

A successful connection requires the relevant layers to allow it.

---

# 2. Authentication vs Authorization

These are different.

### Authentication

> **Who are you?**

Example:

```text
username + authentication method
```

### Authorization

> **What are you allowed to do?**

Example:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
DROP
```

Think:

```text
Authentication
      ↓
"Who are you?"

Authorization
      ↓
"What can you do?"
```

---

# 3. The MariaDB Account Model

A MariaDB account is commonly represented as:

```text
'user'@'host'
```

For example:

```sql
'appuser'@'localhost'
```

and:

```sql
'appuser'@'10.10.10.50'
```

are different accounts.

This is extremely important.

---

# 4. Why the Host Matters

Suppose you create:

```sql
CREATE USER 'appuser'@'localhost'
IDENTIFIED BY 'StrongPassword';
```

That account is intended for local connections.

If the application server is:

```text
10.10.10.50
```

then the application connecting remotely isn't necessarily represented by:

```text
'appuser'@'localhost'
```

You must design the account correctly.

---

# 5. Check Existing Users

Run:

```sql
SELECT User, Host
FROM mysql.user;
```

You may see:

```text
+----------+-----------+
| User     | Host      |
+----------+-----------+
| root     | localhost |
| appuser  | localhost |
+----------+-----------+
```

This gives you an initial account inventory.

---

# 6. Check Authentication Plugins

Run:

```sql
SELECT User, Host, plugin
FROM mysql.user;
```

You may see different authentication mechanisms depending on the MariaDB release and account configuration.

Don't change authentication plugins simply because you see one you don't recognize.

First understand:

```text
Current authentication
        ↓
Application compatibility
        ↓
Security requirement
        ↓
Supported MariaDB version
```

---

# 7. `root` Is Not an Application Account

A common mistake:

```text
Application
     ↓
root
     ↓
MariaDB
```

❌ Don't do this.

Applications should normally use dedicated accounts.

Instead:

```text
Application
     ↓
appuser
     ↓
Required privileges only
```

---

# 8. Create an Application User

Example:

```sql
CREATE USER 'appuser'@'10.10.10.50'
IDENTIFIED BY 'StrongRandomPassword';
```

Notice:

```text
'appuser'
    +
'10.10.10.50'
```

This restricts the source host.

Your actual design may use a subnet or another controlled host pattern depending on your architecture.

---

# 9. Create a Database

For the lab:

```sql
CREATE DATABASE salesdb;
```

Check:

```sql
SHOW DATABASES;
```

---

# 10. Grant Only What the Application Needs

Suppose the application needs:

```text
SELECT
INSERT
UPDATE
DELETE
```

You could grant those privileges:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON salesdb.*
TO 'appuser'@'10.10.10.50';
```

Then:

```sql
SHOW GRANTS FOR 'appuser'@'10.10.10.50';
```

---

# 11. Why `GRANT ALL` Is Dangerous

A beginner might do:

```sql
GRANT ALL PRIVILEGES
ON *.*
TO 'appuser'@'%';
```

This is usually far more access than an application needs.

You've potentially created:

```text
Application
    ↓
Everything
    ↓
Every database
```

That's a serious security risk.

---

# 12. Least Privilege

The principle is:

> **Give an account only the permissions required to perform its job.**

For example:

```text
Reporting user
   ↓
SELECT only

Application user
   ↓
SELECT
INSERT
UPDATE
DELETE

DBA
   ↓
Administrative privileges
```

This reduces the blast radius if an account is compromised.

---

# 13. Reporting User Example

Suppose a reporting application only needs to read:

```sql
GRANT SELECT
ON salesdb.*
TO 'reportuser'@'10.10.10.60';
```

It cannot modify data merely because it can read it.

That's good security design.

---

# 14. Read-Only Means Read-Only

Test the account rather than assuming.

Connect as:

```text
reportuser
```

Then:

```sql
SELECT * FROM salesdb.some_table;
```

should work.

But:

```sql
DELETE FROM salesdb.some_table;
```

should fail if the account only has `SELECT`.

Always validate security controls.

---

# 15. `SHOW GRANTS`

This command should become second nature:

```sql
SHOW GRANTS FOR 'appuser'@'10.10.10.50';
```

It tells you what privileges the account has.

Use it during:

```text
Access reviews
Troubleshooting
Security audits
Application onboarding
Migration
Incident response
```

---

# 16. Revoke Privileges

Suppose you accidentally granted:

```sql
GRANT UPDATE
ON salesdb.*
TO 'reportuser'@'10.10.10.60';
```

Remove it:

```sql
REVOKE UPDATE
ON salesdb.*
FROM 'reportuser'@'10.10.10.60';
```

Then verify:

```sql
SHOW GRANTS FOR 'reportuser'@'10.10.10.60';
```

---

# 17. Drop an Unused User

If an account is no longer needed:

```sql
DROP USER 'olduser'@'10.10.10.70';
```

Don't simply change its password and leave unused accounts around indefinitely.

Unused accounts increase attack surface.

---

# 18. Password Security

Never use:

```text
password
Password123
MariaDB123
admin123
```

Use strong, unique credentials.

For production:

```text
Random
Unique
Long
Stored securely
Rotated according to policy
```

Don't put production passwords into:

```text
Git
Shell history
Tickets
Plain-text documentation
Chat messages
```

---

# 19. Password Rotation

A mature DBA environment has a process:

```text
Credential created
       ↓
Stored securely
       ↓
Application configured
       ↓
Rotation schedule
       ↓
Old credential retired
       ↓
Validation
```

Password rotation must account for application connection pools and deployment procedures.

---

# 20. Remote Access

Remote MariaDB access requires several things to align:

```text
Application
    │
    ▼
Network route
    │
    ▼
Firewall
    │
    ▼
MariaDB bind/listener
    │
    ▼
User@Host
    │
    ▼
Authentication
    │
    ▼
Privileges
```

If any layer blocks the connection, it fails.

---

# 21. Don't Use `'%'` Automatically

You may encounter:

```sql
'appuser'@'%'
```

This means the account can potentially authenticate from many source hosts, subject to other controls.

Sometimes wildcard hosts are appropriate.

But don't use them merely because:

> "It makes remote connections easier."

Prefer the narrowest practical source restriction.

---

# 22. Firewall + MariaDB Permissions

You want **defense in depth**.

Example:

```text
Only application subnet
        │
        ▼
Windows/Linux firewall
        │
        ▼
MariaDB 3306
        │
        ▼
'appuser'@'10.10.10.%'
        │
        ▼
salesdb privileges
```

Even if one layer is misconfigured, other controls can reduce exposure.

---

# 23. TLS

For sensitive production traffic, consider encrypted client/server communication.

Without TLS:

```text
Application
      │
      │ network traffic
      ▼
MariaDB
```

With TLS:

```text
Application
      │
      │ encrypted
      ▼
MariaDB
```

TLS helps protect data in transit.

---

# 24. Why TLS Matters

Consider:

```text
Application Server
       │
       │
       ▼
Network
       │
       │
       ▼
Database Server
```

Traffic crossing networks may be exposed to interception if not appropriately protected.

TLS provides encryption and, when configured appropriately, server/client identity verification.

---

# 25. Don't Disable TLS Verification Blindly

A common application workaround is:

```text
"Certificate problem? Turn verification off."
```

That's not a good security strategy.

Instead investigate:

```text
Certificate
CA
Hostname
Trust store
Expiration
Server configuration
Client configuration
```

---

# 26. Administrative Accounts

A mature MariaDB environment should distinguish:

```text
DBA
Application
Reporting
Monitoring
Backup
Replication
```

These shouldn't all use the same account.

Think:

```text
                  MariaDB
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
      DBA          App User     Reporting
  Administration  Application    Read-only
```

---

# 27. Monitoring Accounts

Monitoring often requires specific privileges.

Don't give the monitoring system:

```text
ALL PRIVILEGES
```

just because it needs metrics.

Design the account around the monitoring requirements.

This is an example of **least privilege**.

---

# 28. Backup Accounts

Backup tooling may need particular privileges depending on:

```text
Backup method
MariaDB version
Logical vs physical backup
Storage engine
Replication configuration
```

Don't blindly reuse:

```text
root
```

for every backup system.

Design dedicated backup credentials.

---

# 29. Linux Security

On Linux, MariaDB security includes:

```text
MariaDB accounts
+
Filesystem permissions
+
Linux service account
+
Firewall
+
SELinux/AppArmor where applicable
+
TLS
```

Security is not just SQL.

---

# 30. Windows Security

On Windows:

```text
MariaDB accounts
+
Windows service identity
+
NTFS permissions
+
Windows Firewall
+
TLS
+
Windows security policy
```

Again:

> **Database security includes the operating system.**

---

# 31. Filesystem Security

MariaDB's data directory contains sensitive database information.

Therefore:

```text
Data directory
       ↓
Restricted filesystem access
```

Don't allow arbitrary users to read or modify it.

Never solve a permissions issue by making the directory universally writable.

---

# 32. Protect Configuration Files

Configuration files may contain sensitive information depending on your environment.

Protect:

```text
my.cnf
my.ini
credential files
TLS private keys
certificates
backup configuration
```

Especially protect **private keys and credentials**.

---

# 33. Remove Unnecessary Accounts

Perform an account review:

```sql
SELECT User, Host
FROM mysql.user
ORDER BY User, Host;
```

Ask:

```text
Who owns this account?
Why does it exist?
Where is it used?
What privileges does it have?
Is it still required?
```

This is an important DBA security habit.

---

# 34. Security Audit Example

Suppose you find:

```text
root@localhost
appuser@%
reportuser@%
olduser@localhost
test@%
```

Don't immediately delete everything.

Investigate:

```text
root
   ↓
Administrative

appuser
   ↓
Application

reportuser
   ↓
Reporting

olduser
   ↓
Potentially obsolete

test
   ↓
Potentially dangerous
```

Then validate ownership and usage before modifying accounts.

---

# 35. Security Incident Mindset

Suppose you discover:

```text
'appuser'@'%'
```

with excessive privileges.

Don't panic.

Follow:

```text
Identify
   ↓
Determine usage
   ↓
Determine exposure
   ↓
Review logs
   ↓
Reduce privilege
   ↓
Rotate credentials if necessary
   ↓
Validate application
   ↓
Document
```

Security changes should be controlled.

---

# 36. Common Security Mistakes

### ❌ Application uses root

Never make this your normal application architecture.

### ❌ `GRANT ALL ON *.*`

Usually excessive.

### ❌ `'user'@'%'` everywhere

Too broad unless deliberately designed.

### ❌ Shared DBA credentials

Makes accountability difficult.

### ❌ Weak passwords

Easy attack vector.

### ❌ No TLS for sensitive network traffic

Potential exposure.

### ❌ Publicly exposing 3306

Unnecessary attack surface.

### ❌ World-readable data directory

Severe filesystem security issue.

---

# 37. Security Validation Checklist

After hardening:

```text
☐ Root/admin access reviewed
☐ Application accounts separated
☐ Reporting accounts separated
☐ Monitoring accounts reviewed
☐ Backup accounts reviewed
☐ Unused accounts removed
☐ Host restrictions reviewed
☐ Grants reviewed
☐ Least privilege applied
☐ Password policy followed
☐ TLS requirement evaluated
☐ Firewall restricted
☐ Data directory protected
☐ Configuration protected
☐ Service account reviewed
☐ Security logging/monitoring planned
```

---

# 38. Hands-On Lab

Create a test database:

```sql
CREATE DATABASE trainingdb;
```

Create an application account:

```sql
CREATE USER 'training_app'@'localhost'
IDENTIFIED BY 'Use-A-Strong-Lab-Password';
```

Grant only application-level privileges:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON trainingdb.*
TO 'training_app'@'localhost';
```

Verify:

```sql
SHOW GRANTS FOR 'training_app'@'localhost';
```

---

# 39. Create a Read-Only User

```sql
CREATE USER 'training_report'@'localhost'
IDENTIFIED BY 'Use-Another-Strong-Lab-Password';
```

Grant:

```sql
GRANT SELECT
ON trainingdb.*
TO 'training_report'@'localhost';
```

Verify:

```sql
SHOW GRANTS FOR 'training_report'@'localhost';
```

You now have:

```text
training_app
     │
     ├── SELECT
     ├── INSERT
     ├── UPDATE
     └── DELETE

training_report
     │
     └── SELECT
```

---

# 40. Test Least Privilege

As the reporting user:

```sql
SELECT *
FROM trainingdb.some_table;
```

should work if the table exists.

But:

```sql
DELETE FROM trainingdb.some_table;
```

should fail because the account wasn't granted `DELETE`.

That's the point.

---

# 41. DBA Challenge

### Q1

What is the difference between:

```text
'appuser'@'localhost'
```

and:

```text
'appuser'@'%'
```

### Q2

Why should applications not normally use `root`?

### Q3

What does the principle of least privilege mean?

### Q4

What command shows an account's privileges?

### Q5

What command removes a privilege?

### Q6

Why is TLS useful?

### Q7

Why isn't configuring MariaDB alone enough to secure a database server?

### Q8

A reporting application only needs to read data. Should you grant:

```sql
GRANT ALL PRIVILEGES
```

Why or why not?

---

# Answers

### Q1

They represent different host scopes. `localhost` restricts the account to local connections, while `%` is a wildcard host pattern and can allow connections from many hosts.

### Q2

Because a compromised application would potentially gain administrative control over the database.

### Q3

Give users and applications only the privileges they actually require.

### Q4

```sql
SHOW GRANTS FOR 'username'@'host';
```

### Q5

```sql
REVOKE ...
```

### Q6

TLS encrypts network communication and can provide identity verification when properly configured.

### Q7

Because security also involves the OS, filesystem, firewall, network, service identity, TLS, credentials, and monitoring.

### Q8

No. Grant only the required `SELECT` privileges.

---

# ⭐ Lesson 5 Golden Rules

> **1. Authentication answers "Who are you?"**

> **2. Authorization answers "What can you do?"**

> **3. Always think in terms of `'user'@'host'`.**

> **4. Never use root as the normal application account.**

> **5. Apply least privilege.**

> **6. Restrict remote accounts to known sources whenever practical.**

> **7. Protect MariaDB at both the database and operating-system layers.**

> **8. Use TLS where appropriate for network traffic.**

---

# Your MariaDB DBA Security Model

```text
                         SECURITY
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       NETWORK             OS              DATABASE
          │                 │                 │
      Firewall          Permissions         Users
      TCP 3306          Service Account     Grants
      TLS               Data Directory     Authentication
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    DEFENSE IN DEPTH
```

## Next — Lesson 0.6: MariaDB Backup & Recovery

This is where DBA administration becomes **critical**.

We'll cover:

```text
Logical backup
Physical backup
mariadb-dump
mariadb-backup
Full backup
Incremental backup
Binary logs
Point-in-time recovery
Restore
Recovery testing
RPO
RTO
Backup retention
Linux
Windows
Backup failures
Production backup strategy
```

And we'll build a **real backup → destroy → restore → validate** DBA lab.
