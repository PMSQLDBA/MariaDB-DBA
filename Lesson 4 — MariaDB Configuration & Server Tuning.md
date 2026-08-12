# Lesson 4 — MariaDB Configuration & Server Tuning

**Goal:** Learn how a MariaDB DBA controls the behavior of the server through configuration, system variables, memory settings, connections, logging, and runtime changes.

This lesson is important because **bad configuration can cause performance problems, crashes, or unnecessary resource consumption**.

---

# 1. MariaDB Configuration Architecture

Think of MariaDB like this:

```text
Operating System
       │
       ▼
MariaDB Server
       │
       ├── Configuration
       │
       ├── Memory
       │
       ├── Connections
       │
       ├── Storage
       │
       └── Logging
```

MariaDB configuration controls things such as:

* Network port
* Data directory
* Maximum connections
* Buffer sizes
* Logging
* Binary logging
* InnoDB behavior
* Character sets
* Timeouts
* Temporary tables
* Replication

---

# 2. Configuration Files

MariaDB reads configuration from configuration files.

A common configuration file is:

```text
my.cnf
```

Depending on the operating system and installation method, the actual location can differ.

Don't assume there is only one configuration file.

To investigate the server's configuration-file search paths, a DBA can use:

```bash
mariadbd --help --verbose
```

Look through the output for the locations where MariaDB searches for option files.

On many Linux systems, you may encounter paths such as:

```text
/etc/my.cnf
/etc/mysql/my.cnf
/etc/mysql/mariadb.conf.d/
```

The exact layout depends on the distribution and installation.

---

# 3. Important Configuration Sections

A configuration file commonly contains a server section such as:

```ini
[mariadbd]
port=3306
max_connections=200
```

The exact settings you should use depend on the workload and available hardware.

Don't copy random production tuning values from the internet.

---

# 4. Find the Current Configuration

You can inspect server variables from SQL.

For example:

```sql
SHOW VARIABLES;
```

That's a lot of information.

Instead, search for a particular variable:

```sql
SHOW VARIABLES LIKE 'port';
```

Check the data directory:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Check maximum connections:

```sql
SHOW VARIABLES LIKE 'max_connections';
```

Check the server's temporary directory:

```sql
SHOW VARIABLES LIKE 'tmpdir';
```

---

# 5. Global vs. Session Variables

This is an important DBA concept.

MariaDB has variables that can exist at different scopes.

### Global

Applies to the server:

```sql
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

### Session

Applies to your current connection:

```sql
SHOW SESSION VARIABLES LIKE 'sql_mode';
```

You can also use:

```sql
SELECT @@global.max_connections;
```

and:

```sql
SELECT @@session.sql_mode;
```

Think:

```text
GLOBAL
   │
   └── Server-wide setting

SESSION
   │
   └── Setting for one connection
```

---

# 6. Dynamic Configuration

Some variables can be changed while MariaDB is running.

For example, depending on the variable:

```sql
SET GLOBAL variable_name = value;
```

For example:

```sql
SET GLOBAL max_connections = 250;
```

Then verify:

```sql
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

But there's an important catch.

A runtime change may **not survive a server restart**.

So distinguish between:

```text
Runtime configuration
        ↓
Effective now
```

and:

```text
Persistent configuration
        ↓
Survives restart
```

For persistent changes, you generally need to modify the appropriate configuration file and restart/reload as required by that setting.

---

# 7. Don't Change Variables Blindly

This is a major DBA lesson.

Suppose someone says:

> "The database is slow. Increase every buffer."

Don't do that.

MariaDB configuration is interconnected.

Increasing memory-heavy settings can cause:

```text
MariaDB
   ↓
Consumes too much RAM
   ↓
Operating system memory pressure
   ↓
Swapping / OOM
   ↓
Performance gets worse
```

**More memory allocation does not automatically mean more performance.**

Always identify the bottleneck first.

---

# 8. `max_connections`

One of the most commonly discussed variables:

```sql
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

It controls the configured maximum number of simultaneous client connections.

Check the current number:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

Check the historical maximum:

```sql
SHOW STATUS LIKE 'Max_used_connections';
```

These three concepts are useful:

```text
max_connections
        ↓
Configured limit

Threads_connected
        ↓
Connections currently connected

Max_used_connections
        ↓
Highest observed concurrent connection usage
```

---

# 9. Connection Problems

Suppose:

```text
max_connections = 100
```

and your application tries to create more connections.

You may eventually encounter connection failures because the server has reached its configured limit.

But don't immediately solve this by setting:

```text
max_connections = 5000
```

You should investigate **why** the application needs so many connections.

Potential causes include:

* Connection leaks
* Poor connection pooling
* Long-running sessions
* Application traffic spikes
* Slow queries
* Poor application design

The DBA should identify the underlying problem.

---

# 10. Threads and Connections

Check connection-related status:

```sql
SHOW STATUS LIKE 'Threads%';
```

You might see:

```text
Threads_connected
Threads_created
Threads_running
```

These can help you understand connection behavior.

For example:

```sql
SHOW STATUS LIKE 'Threads_running';
```

A high number of actively running threads may indicate substantial concurrent workload.

But **one metric alone doesn't diagnose performance**.

---

# 11. InnoDB Buffer Pool

For InnoDB workloads, the buffer pool is one of the most important memory areas.

Conceptually:

```text
Disk
 │
 │ data pages
 ▼
InnoDB Buffer Pool
 │
 ▼
MariaDB queries
```

Frequently accessed data can be kept in memory, reducing disk I/O.

Check its configuration:

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

For a dedicated database server, the buffer pool often represents a substantial portion of available RAM, but the correct value depends on:

* Total RAM
* MariaDB workload
* Other services
* Connections
* Temporary memory
* Operating-system requirements
* MariaDB version/configuration

Don't blindly allocate nearly all system RAM to the buffer pool.

---

# 12. Temporary Tables

MariaDB may need temporary tables for operations such as sorting and grouping.

Inspect:

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

You may see counters such as:

```text
Created_tmp_tables
Created_tmp_disk_tables
Created_tmp_files
```

These metrics can help identify workloads that generate temporary tables.

But again:

**A high counter isn't automatically a problem.**

You need to understand the workload and query patterns.

---

# 13. Character Sets

Character encoding is another important configuration area.

Check the server character set:

```sql
SHOW VARIABLES LIKE 'character_set_server';
```

Check the database:

```sql
SHOW CREATE DATABASE company;
```

Check a table:

```sql
SHOW CREATE TABLE employees;
```

Modern applications commonly use UTF-8-compatible configurations such as `utf8mb4`.

For example:

```sql
CREATE DATABASE company
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

The exact collation choice should match the application's requirements and MariaDB version.

---

# 14. SQL Mode

SQL mode affects how MariaDB interprets and validates SQL.

Check it:

```sql
SELECT @@sql_mode;
```

Or:

```sql
SHOW VARIABLES LIKE 'sql_mode';
```

SQL mode can influence behavior involving:

* Invalid values
* Date handling
* `GROUP BY`
* Strictness
* Zero dates
* Type conversion

This matters when migrating applications or troubleshooting differences between environments.

---

# 15. Logging

Logs are extremely important to a DBA.

MariaDB can provide different types of logging, depending on configuration and workload.

Important categories include:

### Error log

Useful for:

* Startup failures
* Shutdown information
* Crashes
* InnoDB problems
* Replication problems
* Other server errors

### General query log

Records client activity broadly.

This can be useful for troubleshooting, but it can also generate significant overhead and volume.

### Slow query log

Extremely important for performance analysis.

It helps identify queries that take longer than configured thresholds or otherwise meet configured criteria.

---

# 16. Slow Query Log

Check whether it is enabled:

```sql
SHOW VARIABLES LIKE 'slow_query_log';
```

Check the configured file:

```sql
SHOW VARIABLES LIKE 'slow_query_log_file';
```

Check the threshold:

```sql
SHOW VARIABLES LIKE 'long_query_time';
```

For example:

```text
long_query_time = 10
```

means queries meeting the relevant slow-query criteria around 10 seconds or more may be logged.

In performance work, the slow query log is often far more useful than simply saying:

> "MariaDB feels slow."

---

# 17. Binary Logging

Binary logs are extremely important for advanced DBA work.

They are used for purposes including:

* Replication
* Point-in-time recovery
* Auditing certain changes
* Recovery workflows

Check whether binary logging is enabled:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

This becomes particularly important in **Lesson 5 — Backup & Restore** and **Lesson 6 — Replication**.

---

# 18. Check Server Status

A DBA frequently uses:

```sql
SHOW GLOBAL STATUS;
```

You can narrow it down:

```sql
SHOW GLOBAL STATUS LIKE 'Threads%';
```

or:

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

or:

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
```

Status variables provide counters and runtime information about server activity.

---

# 19. `SHOW PROCESSLIST`

This is one of your most useful troubleshooting commands.

Run:

```sql
SHOW PROCESSLIST;
```

For more information:

```sql
SHOW FULL PROCESSLIST;
```

You may see something like:

```text
Id
User
Host
db
Command
Time
State
Info
```

This lets you investigate currently active connections and queries.

For example, if a query has been running for a very long time, the process list may help you identify it.

---

# 20. Killing a Problematic Session

Suppose you identify a session with ID:

```text
12345
```

You can terminate the connection with:

```sql
KILL 12345;
```

This is an **administrative operation** and should not be performed casually.

Before killing a session, determine:

* What is it doing?
* Who owns it?
* Is it blocking other transactions?
* Is it an application connection?
* Is terminating it safe?
* Could it cause a transaction rollback?

A DBA shouldn't simply kill every long-running query.

---

# 21. Configuration Workflow

A professional DBA shouldn't work like this:

```text
Problem
  ↓
Randomly change settings
  ↓
Restart
  ↓
Hope
```

Instead:

```text
Observe
   ↓
Identify bottleneck
   ↓
Measure
   ↓
Change one relevant setting
   ↓
Test
   ↓
Measure again
   ↓
Document
```

This mindset is essential.

---

# 22. Production Change Example

Suppose your team wants to increase `max_connections`.

A responsible process is:

### Step 1 — Check current value

```sql
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

### Step 2 — Check current usage

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

### Step 3 — Check peak usage

```sql
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
```

### Step 4 — Investigate application behavior

Ask:

```text
Why are connections increasing?
```

### Step 5 — Determine whether the change is actually necessary

### Step 6 — Evaluate RAM/resource impact

### Step 7 — Make a controlled change

### Step 8 — Monitor afterward

That's DBA work.

---

# 23. Lesson 4 Lab

On your MariaDB practice server, run:

```sql
SHOW VARIABLES LIKE 'version%';

SHOW VARIABLES LIKE 'datadir';

SHOW VARIABLES LIKE 'port';

SHOW VARIABLES LIKE 'max_connections';

SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_size';

SHOW GLOBAL STATUS LIKE 'Threads%';

SHOW GLOBAL STATUS LIKE 'Max_used_connections';

SHOW GLOBAL STATUS LIKE 'Created_tmp%';

SHOW VARIABLES LIKE 'slow_query%';

SHOW VARIABLES LIKE 'long_query_time';

SHOW VARIABLES LIKE 'log_bin';

SELECT @@sql_mode;

SHOW FULL PROCESSLIST;
```

Record the results.

Then answer these questions:

1. What port is MariaDB using?
2. Where is the data directory?
3. What is `max_connections`?
4. What is the InnoDB buffer pool size?
5. How many connections are currently connected?
6. What has been the maximum number of simultaneous connections?
7. Is the slow query log enabled?
8. What is `long_query_time`?
9. Is binary logging enabled?
10. What SQL mode is active?

---

# Lesson 4 — DBA Knowledge Check

You should now understand:

**Configuration**
→ `my.cnf` / option files

**Runtime variables**
→ `SHOW VARIABLES`

**Runtime statistics**
→ `SHOW STATUS`

**Connections**
→ `Threads_connected`, `Max_used_connections`

**InnoDB memory**
→ `innodb_buffer_pool_size`

**Diagnostics**
→ `SHOW FULL PROCESSLIST`

**Performance logging**
→ Slow query log

**Replication/recovery foundation**
→ Binary logging

**Configuration philosophy**
→ Measure first, change deliberately, measure again.

---

## The Big DBA Lesson

Don't memorize hundreds of MariaDB variables.

Instead, learn this workflow:

> **What is the problem? → What evidence do I have? → Which variable or subsystem is relevant? → What is the safest change? → Did the change actually improve the system?**

That is the beginning of professional database administration.

### Next: Lesson 5 — Backup & Restore

We'll cover **`mariadb-dump`, logical vs. physical backups, full backups, incremental concepts, restore procedures, backup verification, point-in-time recovery, binary logs, and the difference between having a backup and having a recoverable backup.**
