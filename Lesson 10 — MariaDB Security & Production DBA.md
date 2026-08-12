# Lesson 10 — MariaDB Security & Production DBA

Welcome to the **final lesson** of your 10-lesson MariaDB DBA course.

You've already covered:

1. MariaDB fundamentals
2. Users & privileges
3. Database objects & administration
4. Transactions & storage engines
5. Backup & restore
6. Replication
7. Performance & monitoring
8. High availability
9. Maintenance & troubleshooting
10. **Security + production administration**

Now we're going to combine everything into a real DBA operating model.

---

# 1. The DBA's Job

A MariaDB DBA isn't simply someone who knows SQL.

A production DBA is responsible for:

```text
Availability
     +
Performance
     +
Security
     +
Backup
     +
Recovery
     +
Monitoring
     +
Capacity
     +
Change management
```

Think of the DBA as protecting **three things**:

```text
             DBA
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
     DATA   SERVICE   SECURITY
```

---

# 2. Database Security Starts With Accounts

List MariaDB accounts:

```sql
SELECT User, Host
FROM mysql.user;
```

You might see:

```text
root        localhost
appuser     localhost
reportuser  10.0.0.50
repl        10.0.0.20
```

Each account should have a specific purpose.

For example:

```text
root       → DBA administration
appuser    → application
reportuser → reporting
repl       → replication
```

Don't use one account for everything.

---

# 3. Least Privilege

This is one of the most important security principles.

> **Give an account only the permissions it actually needs.**

Suppose an application only needs:

```sql
SELECT
INSERT
UPDATE
DELETE
```

Don't give it:

```text
ALL PRIVILEGES
```

unless there is a genuine requirement.

---

# 4. Create a Dedicated Application User

Example:

```sql
CREATE USER 'appuser'@'10.0.0.50'
IDENTIFIED BY 'StrongRandomPassword';
```

Then grant only the required permissions:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON company.*
TO 'appuser'@'10.0.0.50';
```

Verify:

```sql
SHOW GRANTS FOR 'appuser'@'10.0.0.50';
```

This is much safer than letting the application connect as `root`.

---

# 5. Why Host Matters

MariaDB accounts include both:

```text
User
+
Host
```

For example:

```text
'appuser'@'localhost'
```

is different from:

```text
'appuser'@'10.0.0.50'
```

and:

```text
'appuser'@'%'
```

The `%` pattern can allow connections from many hosts.

Therefore, where practical:

```text
Specific host
     ↓
Better control
```

rather than:

```text
%
```

---

# 6. Protect the DBA Account

The most powerful account should be the most carefully protected.

Don't use:

```text
root
```

for application connections.

Don't share administrator credentials among multiple people if individual accounts can be used.

Prefer:

```text
Alice → alice_dba
Bob   → bob_dba
```

rather than:

```text
Everyone → root
```

Individual accounts improve:

* Accountability
* Auditing
* Access control
* Credential management

---

# 7. Password Security

Never put real passwords into:

```bash
history
```

or scripts like:

```bash
mariadb -uroot -pMyPassword
```

Avoid committing passwords into:

```text
Git
configuration repositories
application source code
scripts
documentation
```

Use appropriate secret-management mechanisms.

And use strong, unique credentials.

---

# 8. TLS

When MariaDB connections travel across networks, consider encryption with TLS.

Conceptually:

```text
Without TLS:

Application ──────────────► MariaDB
              network
```

With TLS:

```text
Application ══════════════► MariaDB
             encrypted
```

TLS helps protect credentials and data in transit.

Check relevant SSL/TLS variables:

```sql
SHOW VARIABLES LIKE '%ssl%';
```

Your exact configuration depends on MariaDB version and deployment architecture.

---

# 9. Don't Expose MariaDB to the Internet

A common bad architecture is:

```text
Internet
   │
   ▼
MariaDB :3306
```

A much better design is:

```text
Internet
   │
   ▼
Application
   │
   ▼
Private Network
   │
   ▼
MariaDB
```

Use network controls such as:

* Firewalls
* Private subnets
* Security groups
* Network ACLs
* VPNs
* Restricted source addresses

The database should generally not be directly exposed to the public internet.

---

# 10. Port 3306

MariaDB commonly uses:

```text
TCP 3306
```

Check listening ports on Linux:

```bash
ss -lntp
```

You want to understand:

```text
Who can reach MariaDB?
```

not simply:

```text
Is port 3306 open?
```

---

# 11. Firewall Principle

Think:

```text
Application servers
       │
       │ allowed
       ▼
   MariaDB :3306

Internet
       │
       X
       │
   MariaDB :3306
```

Restrict database access to trusted networks/hosts.

---

# 12. Audit Your Privileges

For important accounts:

```sql
SHOW GRANTS FOR 'appuser'@'10.0.0.50';
```

Check:

* Does the application really need `DELETE`?
* Does it need `CREATE`?
* Does it need `DROP`?
* Does it need administrative privileges?
* Does it have access to unrelated databases?

Over time, privileges tend to accumulate.

Regular reviews prevent **privilege creep**.

---

# 13. Remove Unused Accounts

List users:

```sql
SELECT User, Host
FROM mysql.user;
```

If an account is no longer required, remove it using the appropriate administrative process:

```sql
DROP USER 'olduser'@'localhost';
```

Don't delete accounts merely because you're unfamiliar with them.

First determine:

> **What is this account used for?**

System/application accounts can have legitimate purposes.

---

# 14. Database Security Is Also OS Security

MariaDB doesn't operate in isolation.

Your security layers are:

```text
Application security
        ↓
Network security
        ↓
OS security
        ↓
MariaDB security
        ↓
Filesystem security
        ↓
Backup security
```

If the Linux server is compromised, database security can be undermined.

---

# 15. Backup Security

Remember Lesson 5?

Backups contain valuable data.

Therefore:

```text
Production database
       ↓
Backup
       ↓
Encryption
       ↓
Access control
       ↓
Separate storage
```

Protect:

* Backup files
* Backup credentials
* Encryption keys
* Backup servers
* Cloud storage credentials
* Restore systems

A stolen backup can be as damaging as a compromised database.

---

# 16. Monitoring Security

Monitor for suspicious events such as:

```text
Unexpected accounts
Unexpected privilege changes
Failed authentication
Unusual connections
Unexpected database creation
Unexpected schema changes
Unusual query patterns
```

Depending on your environment and MariaDB capabilities, auditing/logging solutions can help.

The important DBA principle is:

> **You can't investigate an event you never recorded.**

---

# 17. Don't Ignore Logs

A production DBA should know where important logs are.

Conceptually:

```text
MariaDB
 ├── Error log
 ├── Slow query log
 ├── General/audit logging where appropriate
 └── Binary logs
```

Each serves a different purpose.

### Error log

Troubleshooting.

### Slow query log

Performance investigation.

### Binary log

Replication and recovery-related workflows.

### Audit/general logging

Potentially useful for security or activity investigation, depending on configuration and performance requirements.

---

# 18. Change Management

Never make unexplained production changes.

Instead:

```text
Request
   ↓
Understand impact
   ↓
Backup/recovery check
   ↓
Test
   ↓
Schedule
   ↓
Implement
   ↓
Verify
   ↓
Document
```

For example:

> "Increase `innodb_buffer_pool_size`."

Before changing it, ask:

* Why?
* What evidence supports the change?
* How much RAM is available?
* What else consumes memory?
* What is the current value?
* What is the expected result?
* Is a restart required?
* What is the rollback plan?

---

# 19. Capacity Planning

A DBA shouldn't wait until the database runs out of resources.

Monitor trends:

```text
Database size
Connections
CPU
RAM
Disk
IOPS
Query rate
Transaction rate
Replication lag
```

Suppose database size is growing:

```text
January → 100 GB
February → 115 GB
March → 130 GB
April → 150 GB
```

You can estimate when storage will become a problem.

That's capacity planning.

---

# 20. Daily DBA Checklist

A practical daily checklist might look like:

```text
☐ Check MariaDB availability
☐ Check error logs
☐ Check backup status
☐ Check replication/cluster health
☐ Check replication lag
☐ Check disk space
☐ Check critical alerts
☐ Check failed jobs
☐ Check unusual long-running queries
```

Not every environment needs exactly these checks, but this is a good foundation.

---

# 21. Weekly DBA Checklist

Weekly:

```text
☐ Verify backups
☐ Perform/inspect restore tests
☐ Review slow queries
☐ Review disk growth
☐ Review replication health
☐ Review database growth
☐ Review user accounts
☐ Review privileges
☐ Review security events
☐ Review HA status
```

---

# 22. Monthly DBA Checklist

Monthly:

```text
☐ Test disaster-recovery procedures
☐ Review backup retention
☐ Review capacity
☐ Review database configuration
☐ Review indexes/query trends
☐ Review privileged accounts
☐ Review unused accounts
☐ Review patches/security updates
☐ Review HA/failover documentation
☐ Review incident history
```

Again, adapt the schedule to your organization's requirements.

---

# 23. The Production DBA Workflow

Let's put your entire course together.

Imagine:

```text
Application reports:
"Database is slow."
```

Your process:

```text
1. Check application
        ↓
2. Check MariaDB availability
        ↓
3. Check processlist
        ↓
4. Check CPU/RAM/disk
        ↓
5. Check locks
        ↓
6. Find slow SQL
        ↓
7. EXPLAIN suspicious queries
        ↓
8. Fix root cause
        ↓
9. Verify performance
        ↓
10. Document
```

That's DBA thinking.

---

# 24. Production Incident: Database Down

Imagine:

```text
MariaDB ❌
```

Your response:

```text
1. Confirm outage
        ↓
2. Check server
        ↓
3. Check network
        ↓
4. Check MariaDB service
        ↓
5. Check disk
        ↓
6. Check memory
        ↓
7. Read error log
        ↓
8. Identify failure
        ↓
9. Recover
        ↓
10. Verify application
        ↓
11. Verify replication/HA
        ↓
12. Document
```

Don't randomly restart multiple components.

---

# 25. Production Incident: Data Accidentally Deleted

Suppose someone executes:

```sql
DELETE FROM customers;
```

and forgets the `WHERE`.

Your response should be:

```text
STOP FURTHER DAMAGE
        ↓
Preserve evidence
        ↓
Determine transaction state
        ↓
Determine exact time
        ↓
Identify backups
        ↓
Identify binary logs
        ↓
Determine recovery point
        ↓
Perform controlled recovery
        ↓
Validate data
        ↓
Document incident
```

This is where Lessons 5 and 6 become extremely important.

---

# 26. Production Incident: Replica Broken

You learned:

```sql
SHOW SLAVE STATUS\G
```

or the corresponding replica-status command for your version.

Then:

```text
Receiver?
   ↓
Network/authentication/primary

Applier?
   ↓
SQL/data/schema problem
```

Then:

```text
Root cause
    ↓
Repair
    ↓
Verify
```

If the replica is no longer trustworthy:

```text
Fresh backup
     ↓
Rebuild replica
     ↓
Verify
```

---

# 27. Production Incident: Storage Full

Your first instinct should be:

```bash
df -h
```

Then:

```bash
df -i
```

Then identify what is consuming space.

Potential sources:

```text
Binary logs
Backups
Logs
Temporary files
Other applications
Database growth
```

Do not randomly remove MariaDB files.

---

# 28. The DBA's Recovery Hierarchy

When data is damaged, think in this order:

```text
1. Existing transaction rollback?
          ↓
2. Replica/HA recovery?
          ↓
3. Backup restore?
          ↓
4. Point-in-time recovery?
          ↓
5. Specialized emergency recovery?
```

The best option depends entirely on what happened.

---

# 29. The Most Important Numbers

A production DBA should understand:

### RPO

```text
How much data can we lose?
```

### RTO

```text
How long can we be down?
```

### Database size

```text
How much data do we have?
```

### Growth rate

```text
How quickly is it increasing?
```

### Query latency

```text
How quickly are queries completing?
```

### Replication lag

```text
How far behind is the replica?
```

These numbers help turn vague requirements into engineering decisions.

---

# 30. Your Complete MariaDB Architecture

After 10 lessons, you should be able to understand an architecture like:

```text
                    USERS
                      │
                      ▼
                APPLICATION
                      │
                      ▼
               LOAD BALANCER
                      │
                      ▼
                DB PROXY/HA
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       MariaDB     MariaDB     MariaDB
        Node 1      Node 2      Node 3
          │           │           │
          └───────────┼───────────┘
                      │
                Cluster/Replication
                      │
                      ▼
                 BACKUP SYSTEM
                  /         \
                 ▼           ▼
            Local copy    Off-site copy
```

Around everything:

```text
Security
Monitoring
Alerting
Logging
Documentation
```

---

# 31. Your 10-Lesson DBA Skill Map

You've now covered:

| Lesson | Skill                                |
| ------ | ------------------------------------ |
| 1      | MariaDB fundamentals                 |
| 2      | Users & privileges                   |
| 3      | Database/table administration        |
| 4      | Transactions & storage engines       |
| 5      | Backup & restore                     |
| 6      | Replication                          |
| 7      | Performance & monitoring             |
| 8      | High availability                    |
| 9      | Maintenance & troubleshooting        |
| 10     | Security & production administration |

---

# 32. The 20 Commands You Should Know

These are worth becoming comfortable with:

```sql
SHOW DATABASES;

USE database_name;

SHOW TABLES;

DESCRIBE table_name;

SHOW CREATE TABLE table_name;

SELECT ...;

CREATE DATABASE ...;

CREATE USER ...;

ALTER USER ...;

DROP USER ...;

GRANT ...;

REVOKE ...;

SHOW GRANTS FOR ...;

SHOW FULL PROCESSLIST;

SHOW VARIABLES;

SHOW GLOBAL STATUS;

SHOW ENGINE INNODB STATUS\G;

EXPLAIN ...;

SHOW INDEX FROM table_name;

SHOW SLAVE STATUS\G;
```

And remember that replication commands/status terminology can differ between MariaDB versions.

---

# 33. The 10 Linux Commands a DBA Should Know

If you're administering MariaDB on Linux, become comfortable with:

```bash
systemctl status mariadb
systemctl start mariadb
systemctl stop mariadb
systemctl restart mariadb

journalctl -u mariadb

df -h
df -i
du -sh

top
ss -lntp
```

You don't need to memorize hundreds of Linux commands.

But you **must understand the operating system beneath the database**.

---

# 34. Your DBA Mental Model

When something goes wrong, think in layers:

```text
Application
     ↓
Network
     ↓
Operating System
     ↓
MariaDB Server
     ↓
Storage Engine
     ↓
Filesystem
     ↓
Disk
```

A database problem isn't necessarily a SQL problem.

For example:

```text
Slow query
```

could actually be:

```text
Slow disk
```

or:

```text
Lock contention
```

or:

```text
Network latency
```

or:

```text
CPU exhaustion
```

or:

```text
Bad execution plan
```

---

# 35. The DBA Golden Rules

### Rule 1

> **Never make a production change you don't understand.**

### Rule 2

> **Never trust a backup until you've tested a restore.**

### Rule 3

> **Replication is not backup.**

### Rule 4

> **Measure before optimizing.**

### Rule 5

> **Don't give applications unnecessary privileges.**

### Rule 6

> **Never manually delete database files to solve a storage problem.**

### Rule 7

> **Preserve evidence before troubleshooting destructive failures.**

### Rule 8

> **Test failover before you need failover.**

### Rule 9

> **Monitor trends, not just current values.**

### Rule 10

> **Document how to recover the database.**

---

# 🏆 Final DBA Challenge

You now have the knowledge to work through this scenario.

Imagine a production environment:

```text
MariaDB Cluster
3 nodes

Database size:
800 GB

Daily growth:
5 GB

Application:
24/7

RPO:
15 minutes

RTO:
30 minutes
```

At 02:15 AM:

```text
Node 1 crashes.
```

At 02:17:

```text
Node 2 becomes unhealthy.
```

At 02:20:

```text
Application reports database errors.
```

At 02:25:

```text
You discover Node 3 is healthy.
```

Your DBA thinking should now be:

```text
                 INCIDENT
                    │
                    ▼
             Preserve evidence
                    │
                    ▼
             Determine cluster
                  health
                    │
                    ▼
              Verify quorum
                    │
                    ▼
             Protect Node 3
                    │
                    ▼
             Restore service
                    │
                    ▼
             Verify application
                    │
                    ▼
           Investigate Nodes 1/2
                    │
                    ▼
             Verify replication
                    │
                    ▼
             Verify backups
                    │
                    ▼
             Post-incident review
```

You shouldn't panic.

You should have a **runbook**.

---

# 🎓 You Finished the 10 Lessons

You now have the foundation for **MariaDB DBA administration**:

```text
                 MariaDB DBA
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   SECURITY       PERFORMANCE    AVAILABILITY
       │              │              │
       ▼              ▼              ▼
   USERS          EXPLAIN         REPLICATION
   GRANTS         INDEXES         GALERA
   TLS            LOCKS           FAILOVER
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                 RECOVERY
                      │
             ┌────────┴────────┐
             ▼                 ▼
          BACKUPS             PITR
```

## What comes next?

The natural next stage isn't another basic lesson. It's **hands-on DBA training**.

A good next course would be:

### **MariaDB DBA Lab — 10 Real-World Projects**

1. Build a MariaDB server from scratch
2. Create production-style users and privileges
3. Design and secure a database
4. Automate backups
5. Perform a complete restore
6. Build Primary → Replica replication
7. Break replication and repair it
8. Diagnose and optimize slow queries
9. Build a 3-node HA/Galera environment
10. Perform a simulated production disaster and recover it

That would turn the concepts from these 10 lessons into **actual DBA operational skills**.
