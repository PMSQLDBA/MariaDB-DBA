# Lesson 3 — MariaDB Users & Security

**Goal:** Learn how a MariaDB DBA controls **who can connect, what they can do, and where they can connect from**.

The central security principle is:

> **Give each user only the privileges they actually need.**

---

## 1. MariaDB Authentication vs. Authorization

These are different concepts.

### Authentication

**"Who are you?"**

Example:

```text
username + authentication method
```

### Authorization

**"What are you allowed to do?"**

For example:

```text
alice → SELECT only
bob   → SELECT + INSERT + UPDATE
dba   → administrative privileges
```

Think:

```text
Authentication
      ↓
Who are you?
      ↓
Authorization
      ↓
What can you do?
```

---

# 2. See Existing Users

MariaDB stores account information in its system tables.

You can inspect accounts with:

```sql
SELECT User, Host
FROM mysql.user;
```

You may see something like:

```text
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| root        | localhost |
| mariadb     | localhost |
+-------------+-----------+
```

The important part is that a MariaDB account is associated with both:

```text
User + Host
```

For example:

```text
'alice'@'localhost'
```

and:

```text
'alice'@'%'
```

are different accounts.

---

# 3. Create a User

Create a user:

```sql
CREATE USER 'alice'@'localhost'
IDENTIFIED BY 'StrongPasswordHere';
```

Now Alice exists, but she doesn't automatically have access to everything.

That's intentional.

**Creating a user ≠ granting privileges.**

---

# 4. Grant Privileges

Suppose Alice needs to work with the `company` database.

Give her read-only access:

```sql
GRANT SELECT
ON company.*
TO 'alice'@'localhost';
```

Now she can run queries such as:

```sql
SELECT *
FROM company.employees;
```

But she shouldn't be able to modify the data.

For example, an `UPDATE` should fail if she doesn't have the necessary privilege.

---

# 5. Grant Multiple Privileges

Suppose another application user needs to read and modify data:

```sql
CREATE USER 'appuser'@'localhost'
IDENTIFIED BY 'AnotherStrongPassword';
```

Grant:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON company.*
TO 'appuser'@'localhost';
```

This user can now perform normal CRUD operations.

But notice what we **didn't** give them:

```text
DROP DATABASE
CREATE USER
GRANT
SUPER-level administration
```

That's least privilege.

---

# 6. Database-Level vs. Table-Level Privileges

You can grant privileges at different scopes.

### Entire database

```sql
GRANT SELECT
ON company.*
TO 'alice'@'localhost';
```

The `*` means all tables in that database.

### Specific table

```sql
GRANT SELECT
ON company.employees
TO 'alice'@'localhost';
```

Alice can now read only that table.

### Specific privileges

You can be very granular:

```sql
GRANT SELECT, INSERT
ON company.employees
TO 'alice'@'localhost';
```

This is useful for application accounts that should have very limited access.

---

# 7. Checking a User's Privileges

One of the most important commands for a DBA:

```sql
SHOW GRANTS FOR 'alice'@'localhost';
```

Example result:

```text
GRANT SELECT ON `company`.* TO `alice`@`localhost`
```

This tells you what MariaDB has actually granted to the account.

When troubleshooting permissions, **`SHOW GRANTS` should be one of your first commands.**

---

# 8. REVOKE

Suppose Alice no longer needs `SELECT` access.

Remove it:

```sql
REVOKE SELECT
ON company.*
FROM 'alice'@'localhost';
```

Check:

```sql
SHOW GRANTS FOR 'alice'@'localhost';
```

The privilege should no longer appear.

---

# 9. DROP USER

If an account is no longer needed:

```sql
DROP USER 'alice'@'localhost';
```

This is preferable to simply leaving unused accounts around.

Unused accounts increase your attack surface.

---

# 10. Roles

Imagine you have 20 developers.

Instead of individually granting the same permissions to all 20 users, you can use a **role**.

Create a role:

```sql
CREATE ROLE 'developer';
```

Grant privileges to the role:

```sql
GRANT SELECT, INSERT, UPDATE
ON company.*
TO 'developer';
```

Create users:

```sql
CREATE USER 'alice'@'localhost'
IDENTIFIED BY 'StrongPassword1';

CREATE USER 'bob'@'localhost'
IDENTIFIED BY 'StrongPassword2';
```

Grant the role:

```sql
GRANT 'developer'
TO 'alice'@'localhost';

GRANT 'developer'
TO 'bob'@'localhost';
```

Now the structure becomes:

```text
             developer role
             /            \
            /              \
         Alice             Bob
```

If the required privileges change, you can modify the role instead of individually modifying every user.

---

# 11. Default Roles

You can configure a role to be automatically active for a user.

For example:

```sql
SET DEFAULT ROLE 'developer'
FOR 'alice'@'localhost';
```

This becomes particularly useful in environments with many users and standardized permissions.

---

# 12. Host Restrictions

Remember:

```text
'user'@'host'
```

A local-only account:

```sql
CREATE USER 'report'@'localhost'
IDENTIFIED BY 'StrongPassword';
```

can authenticate locally.

A remote-access account might use a specific host:

```sql
CREATE USER 'report'@'192.168.1.50'
IDENTIFIED BY 'StrongPassword';
```

This is much more restrictive than allowing connections from everywhere.

You may encounter:

```sql
'user'@'%'
```

The `%` represents a wildcard host.

For example:

```sql
CREATE USER 'appuser'@'%'
IDENTIFIED BY 'StrongPassword';
```

This can permit connections from any host that can otherwise reach the server and satisfy the account/authentication rules.

**Don't use `%` casually in production.**

Prefer the narrowest network source that actually meets the requirement.

---

# 13. Why `root` Shouldn't Be Your Application User

A common beginner mistake is:

```text
Application
    ↓
root
    ↓
MariaDB
```

That's dangerous.

If an application is compromised and its database credentials are `root`, the attacker may gain extremely broad database privileges.

A better architecture is:

```text
Application
    ↓
appuser
    ↓
Only required database privileges
```

For example:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON company.*
TO 'appuser'@'localhost';
```

The application doesn't need DBA privileges.

---

# 14. Password Security

Don't use simple passwords such as:

```text
password
123456
mariadb
admin
```

Use strong, unique credentials.

For production systems, also consider:

* Secret management
* Credential rotation
* Restricted network access
* TLS for remote connections
* Account auditing
* Removing unused accounts
* Avoiding shared accounts

And **never put database passwords directly into source code or public Git repositories.**

---

# 15. TLS and Remote Connections

If clients connect remotely, you should consider encrypting the connection using TLS.

Conceptually:

```text
Without TLS:

Client ───────────────► MariaDB
       potentially exposed traffic


With TLS:

Client ═══════════════► MariaDB
          encrypted
```

For production environments, database traffic should be protected according to your organization's security requirements and threat model.

---

# 16. Privilege Escalation

A major security concern is accidentally granting excessive privileges.

For example:

```sql
GRANT ALL PRIVILEGES
ON *.*
TO 'appuser'@'%';
```

This is generally a **terrible application-account design**.

You've potentially given an application account broad access across the server.

Instead:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON company.*
TO 'appuser'@'localhost';
```

The principle is:

**Specific user + specific host + specific database + specific privileges.**

---

# 17. A Practical User Design

Imagine a company database:

```text
company
├── employees
├── departments
├── payroll
└── reports
```

You might have:

### Application account

```text
appuser
```

Privileges:

```text
SELECT
INSERT
UPDATE
DELETE
```

on appropriate application tables.

### Reporting account

```text
reportuser
```

Privileges:

```text
SELECT
```

on reporting data.

### DBA account

A DBA has significantly broader privileges because administration requires them.

The important point is:

**Don't give every account DBA-level access.**

---

# 18. Security Investigation Commands

These commands should become familiar.

### List users

```sql
SELECT User, Host
FROM mysql.user;
```

### Check privileges

```sql
SHOW GRANTS FOR 'alice'@'localhost';
```

### Check current account

```sql
SELECT USER(), CURRENT_USER();
```

These can produce different information.

`USER()` shows the client identity used to connect.

`CURRENT_USER()` shows the MariaDB account used for authentication and privilege checking.

That distinction becomes useful when diagnosing account/host matching issues.

---

# 19. Lesson 3 Lab

Create a practice database:

```sql
CREATE DATABASE security_lab;
```

Create a read-only user:

```sql
CREATE USER 'reportuser'@'localhost'
IDENTIFIED BY 'StrongPracticePassword';
```

Grant read access:

```sql
GRANT SELECT
ON security_lab.*
TO 'reportuser'@'localhost';
```

Check:

```sql
SHOW GRANTS FOR 'reportuser'@'localhost';
```

Create a role:

```sql
CREATE ROLE 'readonly_role';
```

Grant the role access:

```sql
GRANT SELECT
ON security_lab.*
TO 'readonly_role';
```

Create another user:

```sql
CREATE USER 'auditor'@'localhost'
IDENTIFIED BY 'AnotherStrongPracticePassword';
```

Assign the role:

```sql
GRANT 'readonly_role'
TO 'auditor'@'localhost';
```

Then inspect:

```sql
SHOW GRANTS FOR 'auditor'@'localhost';
```

Finally, practice removing an account:

```sql
DROP USER 'reportuser'@'localhost';
```

---

# Lesson 3 — DBA Knowledge Check

Before moving to Lesson 4, make sure you understand these:

**Authentication**
→ Who is connecting?

**Authorization**
→ What can they do?

**`CREATE USER`**
→ Creates an account.

**`GRANT`**
→ Gives privileges.

**`REVOKE`**
→ Removes privileges.

**`SHOW GRANTS`**
→ Shows effective grants for an account.

**`CREATE ROLE`**
→ Creates reusable privilege sets.

**`DROP USER`**
→ Removes an account.

**`'user'@'host'`**
→ MariaDB accounts include a host component.

**Least privilege**
→ Give users only what they need.

**Application accounts ≠ DBA accounts**

---

## DBA Rule to Remember

> **Never solve a permissions problem by blindly granting `ALL PRIVILEGES`.**

First determine **exactly what operation the user needs**, then grant the minimum privilege necessary.

### Next: Lesson 4 — MariaDB Configuration

We'll cover **`my.cnf`, `datadir`, ports, `max_connections`, memory/buffer settings, logging, server variables, dynamic vs. startup configuration, and how a DBA safely changes MariaDB configuration.**
