# Lesson 7 — MariaDB Performance & Monitoring

**Goal:** Learn how to answer the DBA question:

> **“Why is MariaDB slow?”**

The professional answer is **not** “increase the buffers.”

Instead:

```text
Observe → Measure → Identify bottleneck → Fix → Measure again
```

---

# 1. What Does “Slow” Actually Mean?

A database can be slow for many different reasons:

```text
Application
    │
    ▼
MariaDB
    │
    ├── CPU bottleneck
    ├── RAM pressure
    ├── Disk I/O
    ├── Network
    ├── Locks
    ├── Bad SQL
    ├── Missing indexes
    ├── Too many connections
    └── Replication lag
```

Your first job is to **identify which category the problem belongs to**.

---

# 2. Start With `SHOW PROCESSLIST`

When users report:

> “The database is hanging.”

Start by looking at currently running sessions:

```sql id="gq7m0k"
SHOW FULL PROCESSLIST;
```

Look for:

* Very long-running queries
* Many connections
* Queries waiting for locks
* Queries stuck in unusual states
* Large numbers of sleeping connections
* Unexpected users/applications

Example conceptual output:

```text id="c1s3q6"
Id     User      Time    State       Info
101    appuser   0       executing   SELECT ...
102    appuser   250     Waiting...  UPDATE ...
103    report    900     Sending...  SELECT ...
```

The `Time` and `State` columns can give you valuable clues.

---

# 3. Find Long-Running Queries

You can inspect process information with:

```sql id="j3s9q5"
SHOW FULL PROCESSLIST;
```

A query running for:

```text id="y0p1j8"
2 seconds
```

may be perfectly normal.

A query running for:

```text id="r5g3b7"
3 hours
```

deserves investigation.

But don't automatically kill it.

Ask:

> Why has it been running for three hours?

---

# 4. Killing a Query

If you determine that a query must be terminated, identify its connection ID.

For example:

```text id="j0q1y8"
Id = 123
```

You can terminate the connection:

```sql id="g5z0g7"
KILL 123;
```

This can interrupt the session and potentially cause transaction rollback.

Therefore:

**Never kill a database session simply because it has a large `Time` value.**

---

# 5. Slow Query Log

The slow query log is one of the DBA's most useful performance tools.

Check whether it is enabled:

```sql id="qv3g5z"
SHOW VARIABLES LIKE 'slow_query_log';
```

Check its location:

```sql id="p8j4d6"
SHOW VARIABLES LIKE 'slow_query_log_file';
```

Check the threshold:

```sql id="a9r7v3"
SHOW VARIABLES LIKE 'long_query_time';
```

The slow query log can tell you:

> **Which SQL statements are actually taking too long.**

That's much better than guessing.

---

# 6. Example of a Slow Query

Suppose your application executes:

```sql id="y3z1t8"
SELECT *
FROM employees
WHERE email = 'alice@example.com';
```

If `employees` contains 20 million rows and `email` isn't appropriately indexed, the query may require significant work.

A DBA should investigate the execution plan.

---

# 7. `EXPLAIN`

This is one of the most important commands in SQL performance analysis.

Run:

```sql id="1d3r9f"
EXPLAIN
SELECT *
FROM employees
WHERE email = 'alice@example.com';
```

MariaDB will provide information about how it intends to execute the query.

You may see columns such as:

```text id="n6k2y5"
id
select_type
table
type
possible_keys
key
key_len
ref
rows
Extra
```

Don't memorize all of these immediately.

Focus first on:

```text id="9r8b1w"
possible_keys
key
rows
Extra
```

---

# 8. Why `EXPLAIN` Matters

Suppose you see:

```text id="7x8k2m"
key = NULL
rows = 20,000,000
```

That may indicate MariaDB expects to examine a huge number of rows.

Now create an appropriate index:

```sql id="1g9k5h"
CREATE INDEX idx_employee_email
ON employees(email);
```

Run:

```sql id="z8s3p2"
EXPLAIN
SELECT *
FROM employees
WHERE email = 'alice@example.com';
```

The plan may now show that the index is being used and that far fewer rows need to be examined.

---

# 9. Table Scan

A table scan means MariaDB examines many or all rows to find matching data.

Conceptually:

```text id="x7q4m1"
Table:

1
2
3
4
5
...
20,000,000

Query
  ↓
Check rows
  ↓
Check rows
  ↓
Check rows
  ↓
...
```

For some queries, a full scan is perfectly reasonable.

For example:

```sql id="7h4f2a"
SELECT COUNT(*)
FROM small_table;
```

A scan of a tiny table isn't a problem.

The DBA must distinguish between:

**Expected work** and **unnecessary work**.

---

# 10. Indexes

An index provides another access path to data.

Conceptually:

```text id="r3c8w4"
Without index:

Query → scan many rows


With index:

Query → index → matching rows
```

Create an index:

```sql id="7t5m9x"
CREATE INDEX idx_department
ON employees(department);
```

Then:

```sql id="f4z1p7"
EXPLAIN
SELECT *
FROM employees
WHERE department = 'IT';
```

Check indexes:

```sql id="w2c9s6"
SHOW INDEX FROM employees;
```

---

# 11. Composite Indexes

Suppose you frequently run:

```sql id="y1x5v8"
SELECT *
FROM employees
WHERE department = 'IT'
AND salary > 70000;
```

You might investigate a composite index such as:

```sql id="4v8c2n"
CREATE INDEX idx_department_salary
ON employees(department, salary);
```

Order matters.

An index on:

```text id="x6w2h1"
(department, salary)
```

is not equivalent to:

```text id="n3f8k5"
(salary, department)
```

The useful index order depends on the workload and query patterns.

Don't create indexes simply because two columns appear in a query.

---

# 12. Index Selectivity

Imagine:

```text id="v5y9t2"
gender
------
M
F
M
F
M
F
...
```

An index on a column with only a couple of possible values may not always provide much benefit.

Compare that with:

```text id="f8n2x6"
email
-----
alice@example.com
bob@example.com
carol@example.com
...
```

Email values may be much more selective.

The usefulness of an index depends on:

* Data distribution
* Query patterns
* Table size
* Cardinality
* Selectivity
* Sort/group operations
* Join conditions

---

# 13. Don't Over-Index

Suppose a table has:

```text id="v2c8k7"
20 indexes
```

That doesn't mean it is optimized.

Every additional index can increase:

* Disk usage
* Memory requirements
* `INSERT` cost
* `UPDATE` cost
* `DELETE` cost
* Maintenance overhead

The goal is:

> **Useful indexes, not maximum indexes.**

---

# 14. `EXPLAIN` for Joins

Consider:

```sql id="k7q4w3"
SELECT e.first_name,
       e.last_name,
       d.department_name
FROM employees e
JOIN departments d
    ON e.department_id = d.department_id
WHERE d.department_name = 'IT';
```

Analyze it:

```sql id="j2r5c9"
EXPLAIN
SELECT e.first_name,
       e.last_name,
       d.department_name
FROM employees e
JOIN departments d
    ON e.department_id = d.department_id
WHERE d.department_name = 'IT';
```

Now you can investigate whether MariaDB has appropriate access paths for the join.

---

# 15. `EXPLAIN FORMAT=JSON`

For deeper analysis, MariaDB supports more detailed explain formats depending on version.

For example:

```sql id="d8v2q1"
EXPLAIN FORMAT=JSON
SELECT *
FROM employees
WHERE department = 'IT';
```

This can provide additional optimizer information.

Don't worry about understanding every field yet.

Start with normal `EXPLAIN`.

---

# 16. Locks

Performance problems aren't always caused by bad queries.

Consider:

```text id="s4w8n2"
Transaction A
    │
    └── locks row

Transaction B
    │
    └── wants same row
            ↓
          WAITING
```

Transaction B may appear "slow," but the real problem is **locking/blocking**.

This is why database performance troubleshooting requires understanding transactions.

---

# 17. Transactions That Stay Open Too Long

A transaction such as:

```sql id="k2d9f7"
START TRANSACTION;

UPDATE employees
SET salary = salary + 1000
WHERE department = 'IT';

-- application waits for 30 minutes

COMMIT;
```

can hold locks much longer than intended.

Long-running transactions can cause:

* Blocking
* Increased undo/history
* Storage growth
* Delayed cleanup
* Poor concurrency

A DBA should investigate transactions that remain open unexpectedly.

---

# 18. InnoDB Status

For InnoDB-specific diagnostics:

```sql id="0q7p4x"
SHOW ENGINE INNODB STATUS\G
```

This can provide information about:

* Transactions
* Locks
* Deadlocks
* Buffer pool
* I/O
* Background activity

It is a powerful troubleshooting command.

---

# 19. Deadlocks

A deadlock can happen when two transactions wait for each other.

Example:

```text id="x4r7k2"
Transaction A
locks Row 1
   ↓
wants Row 2


Transaction B
locks Row 2
   ↓
wants Row 1
```

Result:

```text id="p8k2w5"
A waits for B
B waits for A
```

The database must break the deadlock by rolling back one transaction.

The DBA should investigate the underlying transaction/query pattern rather than merely treating the symptom.

---

# 20. CPU Bottleneck

Suppose MariaDB is consuming very high CPU.

Possible causes:

* Expensive queries
* Large joins
* Sorting
* Aggregation
* Too many concurrent queries
* Poor execution plans
* CPU-intensive functions

The operating system matters too.

MariaDB performance cannot be diagnosed entirely from SQL.

You may need tools such as:

```bash id="w6c2p1"
top
```

or:

```bash id="k5x8m3"
htop
```

depending on the system.

---

# 21. Memory Bottleneck

If the system is running out of memory:

```text id="c8m1y4"
MariaDB
   ↓
RAM pressure
   ↓
Operating system
   ↓
Swap / memory reclaim
   ↓
Performance degradation
```

Check operating-system memory usage as well as MariaDB variables.

MariaDB's memory isn't simply:

```text id="g7f3q9"
innodb_buffer_pool_size
```

Other memory consumers include:

* Connections
* Per-session buffers
* Temporary operations
* Sorts
* Joins
* Network buffers
* Other server components
* The operating system

This is why simply increasing the buffer pool can be dangerous.

---

# 22. Disk I/O

Databases are heavily dependent on storage performance.

Potential symptoms:

```text id="j9m3x7"
High disk utilization
Slow queries
High I/O wait
Long transaction times
```

Possible causes:

* Slow storage
* Insufficient IOPS
* Large table scans
* Excessive temporary tables
* Heavy writes
* Checkpoint pressure
* Poor query design

The database may be waiting for storage rather than CPU.

---

# 23. Connection Pressure

Check:

```sql id="f2k8v4"
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

and:

```sql id="u3p6r9"
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
```

Also:

```sql id="c1v5z8"
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

If:

```text id="z8y2m6"
Threads_connected ≈ max_connections
```

you have reason to investigate connection pressure.

But remember:

**Raising `max_connections` is not automatically the correct solution.**

---

# 24. Performance Counters

You can inspect broad server activity:

```sql id="q4r8v2"
SHOW GLOBAL STATUS LIKE 'Questions';
```

and:

```sql id="m5y1x9"
SHOW GLOBAL STATUS LIKE 'Queries';
```

You can inspect temporary-table activity:

```sql id="x3v7n2"
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

And connection/thread activity:

```sql id="p6c9k4"
SHOW GLOBAL STATUS LIKE 'Threads%';
```

These counters are useful for understanding trends.

A single snapshot is often less valuable than a time series.

---

# 25. Monitoring Over Time

Imagine:

```text id="d9x2p6"
08:00 → 100 connections
09:00 → 150
10:00 → 200
11:00 → 280
12:00 → 400
```

A trend is much more useful than simply knowing:

```text id="y8k4m2"
Current connections = 400
```

Production monitoring should track metrics over time.

Common categories include:

* CPU
* Memory
* Disk
* Connections
* Query latency
* Query throughput
* Lock waits
* Deadlocks
* Temporary tables
* Replication lag
* Disk space
* Error rates

---

# 26. Disk Space

A database can become unavailable simply because the filesystem fills up.

Check from the OS:

```bash id="q9k3f2"
df -h
```

You want to monitor:

```text id="v6m1r8"
Database filesystem
Binary logs
Backup storage
Temporary filesystem
```

A database with:

```text id="r7x2c5"
0 bytes free
```

is a serious operational problem.

---

# 27. A Real DBA Performance Workflow

Suppose users say:

> **"The application is extremely slow."**

Don't immediately change configuration.

Use:

```text id="z1c5m8"
1. Confirm the problem
        ↓
2. Check CPU / RAM / disk
        ↓
3. Check connections
        ↓
4. Check processlist
        ↓
5. Check slow queries
        ↓
6. Examine suspicious SQL with EXPLAIN
        ↓
7. Check locks/deadlocks
        ↓
8. Identify root cause
        ↓
9. Apply targeted fix
        ↓
10. Measure results
```

That's the process you want to develop.

---

# 28. Lesson 7 Lab

Use your `dba_lab` database.

Check active sessions:

```sql id="p3n7w5"
SHOW FULL PROCESSLIST;
```

Check connections:

```sql id="k6r2x9"
SHOW GLOBAL STATUS LIKE 'Threads_connected';

SHOW GLOBAL STATUS LIKE 'Max_used_connections';

SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

Check InnoDB:

```sql id="n8c4q2"
SHOW ENGINE INNODB STATUS\G
```

Create an index:

```sql id="v5x7m1"
CREATE INDEX idx_employee_department
ON employees(department);
```

Then inspect a query:

```sql id="r2k9c6"
EXPLAIN
SELECT *
FROM employees
WHERE department = 'IT';
```

Inspect the indexes:

```sql id="h7m4q8"
SHOW INDEX FROM employees;
```

Check slow-query configuration:

```sql id="z3p8w6"
SHOW VARIABLES LIKE 'slow_query%';

SHOW VARIABLES LIKE 'long_query_time';
```

---

# Lesson 7 — DBA Knowledge Check

You should now understand:

**`SHOW FULL PROCESSLIST`**
→ See active sessions and queries.

**Slow query log**
→ Find expensive/slow SQL.

**`EXPLAIN`**
→ Understand how MariaDB plans to execute a query.

**Indexes**
→ Provide alternative access paths to data.

**Over-indexing**
→ Can hurt write performance and consume resources.

**Locks**
→ Can make otherwise fast queries wait.

**Deadlocks**
→ Transactions can block each other cyclically.

**InnoDB status**
→ Useful for deep storage-engine troubleshooting.

**CPU/RAM/Disk**
→ Database performance depends on the entire system.

**Monitoring**
→ Trends are more useful than isolated measurements.

---

# The Golden Performance Rule

> **Never optimize what you haven't measured.**

If MariaDB is slow, your first response should be:

```text
"What evidence do we have?"
```

—not—

```text
"Which configuration value should I increase?"
```

That distinction separates **DBA troubleshooting** from guesswork.

### Next: Lesson 8 — MariaDB High Availability

We'll cover **HA architecture, failover, Galera Cluster, quorum, split-brain, synchronous replication concepts, node failure, recovery, and how to design MariaDB systems that survive server failures.**
