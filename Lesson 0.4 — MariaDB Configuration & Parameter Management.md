# Lesson 0.4 — MariaDB Configuration & Parameter Management

This is one of the **most important DBA lessons**.

Installing MariaDB gives you a working database. **Configuration makes it suitable for your workload.**

We'll cover the same concepts across **Linux and Windows**, because the MariaDB parameters are mostly the same even though the configuration file locations differ.

---

# 1. Configuration Architecture

Think of MariaDB like this:

```text
                    MariaDB
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Memory         Network         Storage
        │              │              │
        ▼              ▼              ▼
   Buffer Pool     Port 3306       Data Dir
   Connections     Bind Address    Temp Files
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                    Logging
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
           Error     Slow     Binary
            Log      Log       Log
```

A DBA needs to understand **what each parameter does, why it matters, and how to change it safely**.

---

# 2. Configuration Files

### Linux

Common configuration locations include:

```text id="9v5dgl"
/etc/my.cnf
/etc/mysql/my.cnf
/etc/mysql/mariadb.conf.d/
```

### Windows

Commonly:

```text id="0f4b4h"
my.ini
```

But don't assume the file location.

Your first job is:

> **Determine which configuration files MariaDB actually reads.**

---

# 3. Static vs Dynamic Configuration

This is extremely important.

Some variables can be changed while MariaDB is running:

```text id="v3p8kg"
Dynamic
   ↓
SET GLOBAL ...
```

Others require a restart:

```text id="w3i2kf"
Static
   ↓
Edit configuration
   ↓
Restart MariaDB
```

Therefore, before changing anything:

> **Know whether the variable is dynamic.**

---

# 4. Check Current Configuration

Connect:

```bash id="o8g8pl"
mariadb
```

Then:

```sql id="i2jvps"
SHOW VARIABLES;
```

That's a lot of information.

Instead:

```sql id="4e0k4f"
SHOW VARIABLES LIKE 'max_connections';
```

Or:

```sql id="wwf7ve"
SHOW VARIABLES LIKE 'innodb%';
```

Or:

```sql id="b8gcqm"
SHOW VARIABLES LIKE '%timeout%';
```

This is much more practical.

---

# 5. `SHOW GLOBAL VARIABLES`

Use:

```sql id="u8z1q3"
SHOW GLOBAL VARIABLES;
```

This shows server-wide configuration variables.

For example:

```sql id="9g0f7k"
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

---

# 6. Session vs Global

MariaDB can have:

```text id="hzz6o4"
GLOBAL
SESSION
```

Example:

```sql id="x7w6nt"
SHOW GLOBAL VARIABLES LIKE 'wait_timeout';
```

versus:

```sql id="dpsxqj"
SHOW SESSION VARIABLES LIKE 'wait_timeout';
```

The distinction matters because a session can sometimes have its own value.

Think:

```text id="7t3rqa"
GLOBAL
  │
  ├── Session A
  ├── Session B
  └── Session C
```

---

# 7. Configuration Change Workflow

Never do:

```text id="4uq8m4"
"I think this value should be bigger."
        ↓
Change production
        ↓
Hope
```

Use:

```text id="u4q68x"
Measure
  ↓
Understand
  ↓
Check documentation
  ↓
Check current value
  ↓
Determine impact
  ↓
Test
  ↓
Change
  ↓
Verify
  ↓
Monitor
```

This is **real DBA behavior**.

---

# 8. `max_connections`

Check:

```sql id="8m8t8q"
SHOW VARIABLES LIKE 'max_connections';
```

This controls the maximum number of client connections MariaDB permits.

Conceptually:

```text id="xg3p0m"
Application
 ├── Connection 1
 ├── Connection 2
 ├── Connection 3
 ├── ...
 └── Connection N
```

If the application needs more connections than MariaDB allows, users may see connection failures.

---

# 9. But Don't Set It to 10,000

A common beginner mistake:

> "More connections = better performance."

Not necessarily.

Each connection can consume resources.

Increasing:

```text id="4h0gcm"
max_connections
```

without sufficient RAM can make the server worse.

You must consider:

```text id="f5m8bm"
Connections
+
Memory
+
Concurrency
+
Application connection pooling
+
Query workload
```

---

# 10. Connection Monitoring

Check current connections:

```sql id="a7i1h1"
SHOW STATUS LIKE 'Threads_connected';
```

Maximum simultaneous connections observed:

```sql id="ny6i2b"
SHOW STATUS LIKE 'Max_used_connections';
```

This is much better than blindly increasing `max_connections`.

---

# 11. `innodb_buffer_pool_size`

This is one of the most important InnoDB settings.

Check:

```sql id="r3a8y6"
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

Conceptually:

```text id="2v8rtd"
Disk
 │
 │ data
 ▼
InnoDB
 │
 ▼
Buffer Pool
 │
 ▼
RAM
```

The buffer pool caches frequently used InnoDB data and indexes in memory.

---

# 12. Why Buffer Pool Matters

Without enough memory:

```text id="2bjj9y"
Query
 ↓
Disk
 ↓
Read data
```

With effective caching:

```text id="vtrt7r"
Query
 ↓
Buffer Pool
 ↓
RAM
```

RAM is dramatically faster than storage.

But:

> **Don't allocate all server RAM to MariaDB.**

The OS and other processes need memory too.

---

# 13. Memory Planning

Think:

```text id="wzq6ij"
Server RAM
   │
   ├── OS
   ├── MariaDB global buffers
   ├── Connection/session memory
   ├── Temporary operations
   └── Other services
```

Therefore, configuration must be based on:

```text
Actual workload
+
Available RAM
+
Other processes
```

not a random percentage copied from the internet.

---

# 14. `datadir`

Check:

```sql id="x0c7q6"
SHOW VARIABLES LIKE 'datadir';
```

This tells you where MariaDB stores its database files.

Important for:

* Backup
* Migration
* Storage monitoring
* Disaster recovery
* Troubleshooting

---

# 15. `port`

Check:

```sql id="t9krxs"
SHOW VARIABLES LIKE 'port';
```

Common default:

```text id="c0e7ge"
3306
```

Changing the port affects:

```text
Application
Firewall
Monitoring
Replication
Connection strings
Documentation
```

So don't change it casually.

---

# 16. `bind-address`

Check:

```sql id="s2ap2k"
SHOW VARIABLES LIKE 'bind_address';
```

This controls which network interfaces MariaDB listens on.

Conceptually:

```text id="1dfjsg"
127.0.0.1
    ↓
Local access
```

versus:

```text id="4j1kqf"
Server IP
    ↓
Remote network access
```

A remote connection also requires:

```text id="n8t8jc"
Firewall
+
User/Host privileges
+
Authentication
```

---

# 17. `skip-name-resolve`

You may encounter:

```text id="c5eq8t"
skip-name-resolve
```

This can affect how MariaDB handles hostnames for client authentication.

Before enabling it in production, understand whether your existing accounts rely on hostnames.

For example:

```text id="u7x5j1"
'appuser'@'appserver.example.com'
```

may behave differently if hostname resolution is disabled.

This is why configuration changes must be tested.

---

# 18. Character Sets

Check:

```sql id="8n4v8n"
SHOW VARIABLES LIKE 'character_set%';
```

You may encounter:

```text id="b3rye8"
character_set_server
character_set_database
character_set_client
character_set_connection
character_set_results
```

Character sets determine how text is represented.

Incorrect character-set configuration can cause:

```text
Encoding problems
Corrupted-looking text
Application errors
Sorting differences
```

---

# 19. Collation

Check:

```sql id="cby6s5"
SHOW VARIABLES LIKE 'collation%';
```

Collation affects things such as:

```text
Sorting
String comparison
Case sensitivity
Language-specific behavior
```

For example:

```text
ABC
abc
```

may compare differently depending on the collation.

---

# 20. Logging Configuration

A production DBA should understand at least:

```text id="p2i9d0"
Error log
Slow query log
Binary log
```

They serve different purposes.

---

# 21. Error Log

The error log helps answer:

> **Why is MariaDB unhappy?**

Check:

```sql id="2x8r2v"
SHOW VARIABLES LIKE 'log_error';
```

It can contain information about:

* Startup
* Shutdown
* Errors
* Storage engine issues
* Replication problems
* Configuration problems

---

# 22. Slow Query Log

The slow query log helps answer:

> **Which queries are taking too long?**

Typical configuration includes:

```text id="u1c5wp"
slow_query_log
long_query_time
slow_query_log_file
```

Check:

```sql id="q66l8f"
SHOW VARIABLES LIKE 'slow_query%';
```

And:

```sql id="drx3lc"
SHOW VARIABLES LIKE 'long_query_time';
```

---

# 23. Don't Set `long_query_time = 0` Blindly

Setting:

```text id="16i3xe"
long_query_time = 0
```

can effectively capture every query as slow.

That can generate enormous logs.

Useful for controlled troubleshooting in some environments, but dangerous if left running indefinitely.

Always consider:

```text
Disk
IO
Log volume
Privacy
Performance
Retention
```

---

# 24. Binary Logging

Binary logging is fundamental to many MariaDB environments.

It can support:

```text id="frcz6t"
Replication
Point-in-time recovery
Auditing/change analysis
```

Check:

```sql id="0d9t9k"
SHOW VARIABLES LIKE 'log_bin';
```

If enabled:

```text id="lhg8jh"
Application
   ↓
MariaDB
   ↓
Binary Log
   ↓
Replica / Recovery
```

---

# 25. Binary Log Retention

Binary logs consume disk.

Therefore:

```text id="7gh29u"
Binary logs
     ↓
Disk usage
     ↓
Retention policy
```

You need a deliberate retention strategy.

Don't simply delete binary log files from the filesystem.

Use MariaDB's supported mechanisms for managing binary logs.

---

# 26. `server_id`

For replication environments, you'll encounter:

```sql id="4g9k2j"
SHOW VARIABLES LIKE 'server_id';
```

Replication topologies generally require appropriate unique server identity configuration.

Conceptually:

```text id="4g8xgm"
Primary
server_id = 1

Replica 1
server_id = 2

Replica 2
server_id = 3
```

Never accidentally duplicate server identities in a topology where uniqueness is required.

---

# 27. Timeouts

You'll encounter variables such as:

```text id="3l4y71"
wait_timeout
interactive_timeout
connect_timeout
```

Don't change timeouts simply because:

> "The application says timeout."

First determine **which timeout** is occurring and which layer generated it.

Possible layers:

```text id="b6v2a4"
Application
     ↓
Connection pool
     ↓
Network
     ↓
Firewall
     ↓
MariaDB
     ↓
Query execution
```

---

# 28. Temporary Tables

Queries sometimes need temporary resources.

Check relevant variables such as:

```sql id="1txrkw"
SHOW VARIABLES LIKE '%tmp%';
```

Poorly tuned temporary-table behavior can contribute to performance problems.

Again:

> **Measure first.**

---

# 29. Configuration Change Example

Suppose:

```text id="5b2k8x"
max_connections = 151
```

and monitoring shows:

```text id="u9e7v1"
Max_used_connections = 80
```

Don't automatically increase it.

There may be no problem.

But if:

```text id="e4y4z0"
Max_used_connections = 150
```

and applications are frequently hitting connection errors, investigate:

```text id="u2x9df"
Connection pool
Connection leaks
Application behavior
Traffic growth
MariaDB memory
max_connections
```

Then make a controlled change if justified.

---

# 30. Dynamic Change

For a variable that supports dynamic changes, you may be able to use:

```sql id="h82qf3"
SET GLOBAL variable_name = value;
```

Example:

```sql id="b5n0v8"
SET GLOBAL max_connections = 300;
```

**But don't copy this into production blindly.**

You must first verify:

1. The variable supports dynamic changes.
2. The value is appropriate.
3. Memory impact is acceptable.
4. The change meets your version's behavior.
5. You understand whether it persists after restart.

---

# 31. Runtime Change vs Persistent Change

This is critical.

Suppose you run:

```sql id="3y5qnd"
SET GLOBAL max_connections = 300;
```

You may have changed the **current running server**.

But the configuration file might still contain:

```text id="g3h40d"
max_connections=151
```

After restart:

```text id="d9v3f2"
151
```

could return.

Therefore:

```text id="2x3bq7"
Runtime configuration
       +
Persistent configuration
```

must be managed together when appropriate.

---

# 32. Safe Configuration Procedure

Use:

```text id="f4e0y5"
1. Record current value
        ↓
2. Understand desired value
        ↓
3. Check documentation
        ↓
4. Check dependencies
        ↓
5. Test
        ↓
6. Backup configuration
        ↓
7. Change
        ↓
8. Validate
        ↓
9. Monitor
        ↓
10. Document
```

---

# 33. Always Back Up Configuration

Before changing:

```text id="7opzgo"
my.cnf
```

or:

```text id="b0m0ol"
my.ini
```

make a backup.

Linux example:

```bash id="y71s0e"
sudo cp /etc/my.cnf /etc/my.cnf.bak
```

**Only use the actual path on your server.**

Windows conceptually:

```text id="q2k6me"
my.ini
  ↓
my.ini.backup
```

Don't overwrite your only known-good configuration.

---

# 34. One Change at a Time

Bad:

```text id="7kqf4m"
Change buffer pool
+
max_connections
+
timeouts
+
character set
+
binary logging
```

Then MariaDB behaves differently.

What caused the problem?

You don't know.

Better:

```text id="0k5z7j"
Change A
 ↓
Test
 ↓
Monitor
 ↓
Document
 ↓
Change B
```

This dramatically improves troubleshooting.

---

# 35. Configuration Validation

After changing configuration:

```sql id="49qk8m"
SHOW VARIABLES LIKE 'parameter_name';
```

Don't assume the change worked.

If you changed a network setting, test:

```text
Local connection
Remote connection
Application connection
```

If you changed memory:

```text
Memory usage
Performance
Errors
```

If you changed logging:

```text
Log generation
Disk growth
```

---

# 36. Configuration Rollback

Every change needs a rollback strategy.

```text id="p5h4si"
New configuration
       ↓
Problem
       ↓
Restore known-good config
       ↓
Restart if required
       ↓
Validate
```

This is especially important for:

* Production
* Network changes
* Authentication changes
* Storage changes
* Replication
* Memory settings

---

# 37. Linux Configuration Troubleshooting

If MariaDB won't start after a configuration change:

```bash id="l8d1pm"
sudo systemctl status mariadb
```

Then:

```bash id="08q5g7"
sudo journalctl -u mariadb -n 100
```

Then inspect the MariaDB error log.

Think:

```text
Configuration
     ↓
Syntax
     ↓
Value
     ↓
Permissions
     ↓
Filesystem
     ↓
MariaDB startup
```

---

# 38. Windows Configuration Troubleshooting

If MariaDB won't start:

```powershell id="l9nq6a"
Get-Service MariaDB
```

Then inspect:

```text
MariaDB error log
```

and:

```text
Windows Event Viewer
```

Also verify:

```text
my.ini
Data directory
Service account
Permissions
Port
Disk
```

---

# 39. Configuration Anti-Patterns

### ❌ Copying a random `my.cnf` from the internet

Workloads differ.

### ❌ Setting every parameter you find

More configuration doesn't mean better configuration.

### ❌ Giving MariaDB almost all RAM

The OS and connections also need memory.

### ❌ Increasing `max_connections` to huge values

Could cause memory/resource exhaustion.

### ❌ Disabling security controls to fix configuration issues

Find the actual cause.

### ❌ Changing multiple variables simultaneously

Makes troubleshooting difficult.

---

# 40. Production Configuration Checklist

Before declaring configuration complete:

```text id="qshy6p"
☐ Version documented
☐ Configuration files identified
☐ Data directory verified
☐ Port verified
☐ Bind address verified
☐ Memory configuration reviewed
☐ Connection limits reviewed
☐ Character set reviewed
☐ Collation reviewed
☐ Error logging enabled
☐ Slow query strategy defined
☐ Binary logging reviewed
☐ Replication settings reviewed if applicable
☐ Firewall reviewed
☐ Authentication reviewed
☐ Configuration backed up
☐ Changes documented
```

---

# 41. DBA Lab Exercise

Run:

```sql id="ckd7pr"
SHOW VARIABLES LIKE 'datadir';
```

```sql id="x4p0hz"
SHOW VARIABLES LIKE 'port';
```

```sql id="sv0z2x"
SHOW VARIABLES LIKE 'bind_address';
```

```sql id="q9q9hv"
SHOW VARIABLES LIKE 'max_connections';
```

```sql id="b5kq0r"
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

```sql id="2e0tca"
SHOW VARIABLES LIKE 'log_error';
```

```sql id="j4s3fp"
SHOW VARIABLES LIKE 'slow_query_log';
```

```sql id="mb7tkn"
SHOW VARIABLES LIKE 'long_query_time';
```

```sql id="1wq4xq"
SHOW VARIABLES LIKE 'log_bin';
```

Then:

```sql id="7n0i6s"
SHOW STATUS LIKE 'Threads_connected';
```

and:

```sql id="0o4gvr"
SHOW STATUS LIKE 'Max_used_connections';
```

---

# 42. Your Configuration Challenge

Assume you have:

```text id="c2pprk"
Server RAM: 64 GB

Current:
max_connections = 151

Current:
Max_used_connections = 145

Application:
Connection errors during peak hours
```

The application team says:

> "Increase max_connections to 2000."

Should you immediately do:

```sql id="j9exk1"
SET GLOBAL max_connections = 2000;
```

### ❌ No.

First investigate:

```text id="s0blxj"
Connection pooling
       ↓
Connection leaks
       ↓
Peak traffic
       ↓
Query duration
       ↓
Threads_connected
       ↓
Memory usage
       ↓
MariaDB resource limits
```

Then determine an appropriate value.

---

# 43. Second Challenge

Your application suddenly becomes slow.

You discover:

```text id="kdv4pi"
CPU = 25%
RAM = 40%
Disk latency = HIGH
```

Should you increase:

```text id="j5n7by"
innodb_buffer_pool_size
```

immediately?

### Not necessarily.

The evidence points toward:

```text
Storage latency
```

You need to investigate the storage subsystem and query workload before changing memory.

This is the DBA mindset:

> **Don't tune the parameter that sounds relevant. Tune the bottleneck you've actually identified.**

---

# 44. Third Challenge

You change:

```sql id="5f5i5v"
SET GLOBAL max_connections = 300;
```

Then restart MariaDB.

It returns to:

```text id="q9w3na"
151
```

Why?

Because the runtime change didn't necessarily make the persistent configuration change.

You need to understand:

```text
Runtime value
      +
Configuration file
```

and how your MariaDB version handles persistence.

---

# ⭐ Lesson 4 Golden Rules

Remember these:

> **1. Measure before tuning.**

> **2. Know whether a parameter is dynamic or requires restart.**

> **3. Separate runtime changes from persistent configuration.**

> **4. Back up configuration before modifying it.**

> **5. Change one thing at a time.**

> **6. Never blindly copy configuration from another server.**

> **7. Never allocate all system RAM to MariaDB.**

> **8. Never solve a network problem by exposing the database unnecessarily.**

---

# Your DBA Configuration Mental Model

```text
                  MariaDB Configuration
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
      MEMORY            NETWORK            STORAGE
        │                  │                  │
 Buffer Pool          Port 3306           Data Dir
 Connections          Bind Address        Temp
 Session Memory       Firewall            Logs
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                         LOGGING
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            Error        Slow         Binary
             Log         Log           Log
```

## Next — Lesson 0.5: MariaDB Security Hardening & Initial Configuration

We'll cover:

**users → root/admin security → authentication → grants → least privilege → TLS → Windows/Linux firewall → network restrictions → service accounts → audit/logging → secure configuration → security validation checklist.**
