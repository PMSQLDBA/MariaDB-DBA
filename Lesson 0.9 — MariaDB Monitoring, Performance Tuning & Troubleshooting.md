# Lesson 0.9 — MariaDB Monitoring, Performance Tuning & Troubleshooting

This is where you start thinking like a **production MariaDB DBA**.

A database can be:

* Running but slow
* Healthy but overloaded
* Low on disk
* Blocked by locks
* Suffering from bad queries
* Having replication lag
* Running out of connections
* Experiencing storage I/O problems

The DBA's job is to **identify the bottleneck before changing anything**.

---

# 1. DBA Troubleshooting Philosophy

When someone says:

> **"MariaDB is slow."**

Don't immediately increase memory or restart MariaDB.

Use:

```text
                    INCIDENT
                       │
                       ▼
                 DEFINE SYMPTOM
                       │
                       ▼
                 COLLECT EVIDENCE
                       │
                       ▼
                 IDENTIFY BOTTLENECK
                       │
                       ▼
                 TEST HYPOTHESIS
                       │
                       ▼
                 APPLY CHANGE
                       │
                       ▼
                  VALIDATE
                       │
                       ▼
                   MONITOR
```

---

# 2. The Four Major Bottlenecks

Most database performance problems eventually point toward one or more of:

```text
CPU
Memory
Disk / I/O
SQL / Locks
```

Think:

```text
                 PERFORMANCE
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
       CPU          MEMORY        STORAGE
                                      │
                                      ▼
                                     SQL
                                      │
                                      ▼
                                    LOCKS
```

---

# 3. First Rule: Establish the Baseline

Before tuning, know what "normal" looks like.

Record:

```text
CPU
Memory
Disk latency
IOPS
Connections
Threads
Query latency
Transactions
Replication lag
Errors
```

Then you can distinguish:

```text
Normal
   vs
Abnormal
```

---

# 4. MariaDB Version

Always begin with:

```sql
SELECT VERSION();
```

Why?

Because troubleshooting advice depends on:

```text
MariaDB version
Storage engine
Configuration
Operating system
```

Never blindly apply a tuning recommendation designed for a different MariaDB release.

---

# 5. Server Status

A useful starting point:

```sql
SHOW GLOBAL STATUS;
```

This produces a large amount of information.

Instead, investigate specific metrics.

For example:

```sql
SHOW GLOBAL STATUS LIKE 'Threads%';
```

---

# 6. Connections

Check:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

and:

```sql
SHOW STATUS LIKE 'Threads_running';
```

Interpretation:

```text
Threads_connected
    ↓
How many connections exist?

Threads_running
    ↓
How many are actively executing?
```

---

# 7. Too Many Connections

Check:

```sql
SHOW VARIABLES LIKE 'max_connections';
```

Suppose:

```text
max_connections = 200
Threads_connected = 198
```

That deserves investigation.

Don't immediately do:

```text
max_connections = 1000
```

Why?

Because every connection consumes resources.

Increasing the limit can simply move the failure somewhere else:

```text
Connection problem
      ↓
Increase connections
      ↓
More memory usage
      ↓
More CPU contention
      ↓
System becomes worse
```

---

# 8. Current Sessions

Use:

```sql
SHOW PROCESSLIST;
```

For more information:

```sql
SHOW FULL PROCESSLIST;
```

You may see:

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

This is one of your most important troubleshooting commands.

---

# 9. What Is a DBA Looking For?

When reading `SHOW FULL PROCESSLIST`, look for:

```text
Long-running queries
Waiting states
Locked queries
Too many connections
Repeated queries
Unexpected users
Unexpected databases
```

Example:

```text
Query running:
00:15:42
```

That's worth investigating.

But:

> **Long-running doesn't automatically mean bad.**

A legitimate report may take several minutes.

---

# 10. Don't Kill Queries Blindly

You might see:

```text
Query
Time = 1800 seconds
```

Don't immediately run:

```sql
KILL 123;
```

First ask:

```text
What query is it?
Who owns it?
Why is it running?
Is it blocking others?
Is it a legitimate job?
What happens if I terminate it?
```

---

# 11. Killing a Session

If you've confirmed a session is problematic:

```sql
KILL 123;
```

The number is the connection/session ID.

Use this carefully in production.

---

# 12. Locks

One query may be waiting for another.

Think:

```text
Transaction A
     │
     ▼
Locks row
     │
     X
Transaction B
     │
     ▼
Waiting
```

The user experiences:

```text
"Database is slow."
```

But the actual problem is:

> **Lock contention.**

---

# 13. Deadlock

A deadlock is different.

Imagine:

```text
Transaction A
   │
   ├── locks Row 1
   │
   └── waits for Row 2

Transaction B
   │
   ├── locks Row 2
   │
   └── waits for Row 1
```

Result:

```text
A waits for B
B waits for A
```

MariaDB/InnoDB detects the deadlock and resolves it by rolling back one transaction.

---

# 14. Deadlocks Aren't Necessarily a MariaDB Bug

Deadlocks can result from application transaction design.

Example:

```text
Transaction A:
Update customer
Update order

Transaction B:
Update order
Update customer
```

Different lock ordering can produce deadlocks.

Therefore DBAs often need to work with developers.

---

# 15. Slow Query Problem

Suppose the application says:

> "The customer search page takes 10 seconds."

Don't start by changing server memory.

Find the query.

Example:

```sql
SELECT *
FROM customers
WHERE email = 'alice@example.com';
```

Then investigate:

```text
Index?
Rows examined?
Execution plan?
Table size?
Data distribution?
```

---

# 16. `EXPLAIN`

One of your most important tools:

```sql
EXPLAIN
SELECT *
FROM customers
WHERE email = 'alice@example.com';
```

It helps MariaDB show how it plans to execute the query.

---

# 17. What `EXPLAIN` Tells You

Important concepts include:

```text
table
type
possible_keys
key
rows
Extra
```

You don't need to memorize everything immediately.

Focus initially on:

```text
Is an index being used?
How many rows are estimated?
Is MariaDB scanning the whole table?
```

---

# 18. Full Table Scan

Suppose:

```text
customers = 20 million rows
```

Query:

```sql
SELECT *
FROM customers
WHERE email = 'alice@example.com';
```

If `email` isn't indexed, MariaDB may need to inspect many rows.

Conceptually:

```text
20,000,000 rows
      ↓
Search one by one
      ↓
Slow
```

An appropriate index can dramatically reduce the work.

---

# 19. Index

An index is a data structure that helps MariaDB find rows more efficiently.

Conceptually:

```text
Without index:

Table
 │
 ├── Row
 ├── Row
 ├── Row
 ├── Row
 └── ...
      ↓
Search many rows


With index:

Index
  ↓
Matching row locations
  ↓
Table rows
```

---

# 20. Create an Index

Example:

```sql
CREATE INDEX idx_customers_email
ON customers(email);
```

Then:

```sql
EXPLAIN
SELECT *
FROM customers
WHERE email = 'alice@example.com';
```

Compare the execution plan.

---

# 21. Don't Index Everything

Indexes aren't free.

Every index consumes:

```text
Disk
Memory
Write overhead
Maintenance
```

When you perform:

```sql
INSERT
UPDATE
DELETE
```

indexes may also need to be maintained.

So:

```text
More indexes
      ≠
Always better performance
```

---

# 22. Composite Indexes

Suppose:

```sql
SELECT *
FROM orders
WHERE customer_id = 100
AND order_date >= '2026-01-01';
```

A composite index may be appropriate:

```sql
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date);
```

The **column order matters**.

Index design should follow actual query patterns.

---

# 23. `SELECT *`

Avoid using:

```sql
SELECT *
```

when you don't need every column.

Instead:

```sql
SELECT id, name, email
FROM customers
WHERE id = 100;
```

Benefits can include:

```text
Less data read
Less network traffic
Less application processing
Potentially better use of covering indexes
```

---

# 24. Query Filtering

Bad:

```sql
SELECT *
FROM orders;
```

when the application only needs today's orders.

Better:

```sql
SELECT id, customer_id, order_date
FROM orders
WHERE order_date >= CURRENT_DATE;
```

The application should retrieve the data it actually needs.

---

# 25. Functions on Indexed Columns

Be careful with queries like:

```sql
WHERE YEAR(order_date) = 2026
```

Depending on the query and available indexes, applying a function to a column can prevent efficient use of a normal index.

A range predicate may be preferable:

```sql
WHERE order_date >= '2026-01-01'
  AND order_date <  '2027-01-01'
```

Always confirm with `EXPLAIN`.

---

# 26. Performance Is Evidence-Driven

Don't say:

> "Increase `innodb_buffer_pool_size`."

until you know:

```text
Is memory actually the bottleneck?
Is disk I/O high?
What is the current buffer pool configuration?
How much RAM exists?
What else is using memory?
```

Tuning should follow evidence.

---

# 27. InnoDB Buffer Pool

For InnoDB workloads, the buffer pool is a major memory component.

Conceptually:

```text
Disk
  │
  │ slow
  ▼
Buffer Pool
  │
  │ fast memory access
  ▼
Queries
```

A well-sized buffer pool can reduce unnecessary disk reads.

But:

> **Do not blindly allocate nearly all server RAM to MariaDB.**

The OS and other processes also need memory.

---

# 28. Check Buffer Pool Configuration

Run:

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

Remember the value may be displayed in bytes.

You can also inspect relevant status metrics for buffer pool activity.

---

# 29. Memory Troubleshooting

If the server is swapping:

```text
MariaDB
   ↓
Memory pressure
   ↓
OS swap
   ↓
Disk I/O
   ↓
Performance degradation
```

This can be severe for database workloads.

Check OS-level memory usage.

---

# 30. Linux Memory

Useful commands:

```bash
free -h
```

and:

```bash
vmstat 1
```

Also:

```bash
top
```

or:

```bash
htop
```

depending on what is installed.

---

# 31. Linux Disk

Check:

```bash
df -h
```

This tells you filesystem capacity.

Also investigate I/O:

```bash
iostat
```

if available.

A database can have plenty of free space but still suffer from slow storage.

---

# 32. Windows Performance

On Windows, use:

```text
Task Manager
Resource Monitor
Performance Monitor
Event Viewer
```

Watch:

```text
CPU
Memory
Disk
Network
Processes
```

For deeper monitoring, Windows Performance Monitor can provide useful counters.

---

# 33. Disk Full Is a DBA Emergency

Suppose:

```text
/var/lib/mysql
95% full
```

Don't ignore it.

MariaDB may need space for:

```text
Data
Temporary files
Logs
Binary logs
Redo/undo
```

A full filesystem can cause severe database problems.

---

# 34. Find Large Files

Linux:

```bash
du -sh /var/lib/mysql/*
```

Use carefully on production because filesystem traversal itself has cost, though normally modest.

Investigate:

```text
Binary logs
Error logs
General logs
Slow query logs
Temporary files
Backups
```

---

# 35. Windows Disk Investigation

Use PowerShell:

```powershell
Get-PSDrive
```

Then inspect the MariaDB data/log directories.

Also check:

```text
Event Viewer
MariaDB error logs
Backup directories
Binary log directories
```

---

# 36. Slow Query Logging

MariaDB can provide slow query logging to help identify expensive queries.

Conceptually:

```text
Application
     │
     ▼
Slow query
     │
     ▼
Slow query log
     │
     ▼
DBA analysis
     │
     ▼
EXPLAIN / optimization
```

Don't enable heavy logging in production without considering:

```text
Disk usage
I/O
Privacy
Performance
Retention
```

---

# 37. Query Optimization Workflow

Use:

```text
Slow query identified
       ↓
Capture SQL
       ↓
EXPLAIN
       ↓
Check indexes
       ↓
Check rows examined
       ↓
Check joins
       ↓
Check filtering
       ↓
Optimize
       ↓
Test
       ↓
Compare performance
```

---

# 38. Before and After

Never say:

> "I added an index, so it must be faster."

Measure.

Before:

```text
Query = 4.2 sec
```

After:

```text
Query = 180 ms
```

Now you have evidence.

---

# 39. Query Plan Regression

Sometimes an index exists but MariaDB chooses a different plan.

That's why you should compare:

```text
EXPLAIN
```

before and after changes.

Also compare:

```text
Execution time
Rows examined
CPU
I/O
```

---

# 40. Transactions

Transactions are fundamental.

Example:

```sql
START TRANSACTION;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

The changes should be treated as one logical unit.

---

# 41. Long Transactions

Long-running transactions can cause problems such as:

```text
Lock retention
Undo growth
Purge delays
Replication impact
Resource consumption
```

Therefore investigate long transactions during performance incidents.

---

# 42. Commit Quickly

Application design should generally avoid unnecessarily long transactions.

Bad pattern:

```text
BEGIN
   ↓
Do lots of work
   ↓
Wait for user
   ↓
More work
   ↓
COMMIT
```

Better:

```text
BEGIN
   ↓
Required database work
   ↓
COMMIT quickly
```

Exact transaction design depends on application requirements.

---

# 43. Deadlock Troubleshooting

When deadlocks occur, investigate:

```text
Which transactions?
Which tables?
Which indexes?
Which rows?
What lock order?
What application code?
```

Don't just say:

> "MariaDB has deadlocks."

Often the application transaction pattern needs adjustment.

---

# 44. Error Logs

Always know where your MariaDB error log is.

It can reveal:

```text
Startup errors
Storage-engine issues
Replication errors
Authentication/plugin problems
Disk problems
Corruption warnings
Configuration problems
```

A DBA should check logs during incidents.

---

# 45. Linux Incident Workflow

Suppose application reports:

> "Database is slow."

Start:

```bash
systemctl status mariadb
```

Then:

```bash
df -h
```

Then:

```bash
free -h
```

Then:

```bash
vmstat 1
```

Then investigate MariaDB:

```sql
SHOW FULL PROCESSLIST;
```

and relevant status variables.

---

# 46. Windows Incident Workflow

Start with:

```text
MariaDB service status
        ↓
Task Manager
        ↓
Resource Monitor
        ↓
Disk space
        ↓
Event Viewer
        ↓
MariaDB error log
        ↓
SHOW FULL PROCESSLIST
```

Then investigate the specific bottleneck.

---

# 47. "MariaDB Is Slow" Decision Tree

```text
                MariaDB Slow
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
       CPU         Memory        Disk
        │            │            │
        ▼            ▼            ▼
   Saturated?     Swap?       High latency?
        │            │            │
        └────────────┼────────────┘
                     ▼
                  SQL/Locks
                     │
              ┌──────┴──────┐
              ▼             ▼
          Slow query      Blocking
              │             │
              ▼             ▼
            EXPLAIN       Locks
              │             │
              └──────┬──────┘
                     ▼
                 Root Cause
```

---

# 48. Don't Restart as Your First Fix

A classic mistake:

```text
Database slow
     ↓
Restart MariaDB
     ↓
Problem temporarily disappears
```

Why?

Restarting may clear:

```text
Connections
Locks
Memory state
Caches
Temporary conditions
```

But the underlying problem remains.

After a while:

```text
Problem returns
```

A restart can be appropriate in some incidents, but it should not replace root-cause analysis.

---

# 49. Monitoring Strategy

Monitor at three levels.

### OS

```text
CPU
Memory
Disk
Network
```

### MariaDB

```text
Connections
Threads
Queries
Errors
Locks
Buffer pool
Replication
```

### Application

```text
Request latency
Error rate
Connection failures
Transaction failures
```

---

# 50. Alerting

Good alerts are actionable.

Examples:

```text
Disk > 85%
Connection utilization > threshold
Replication lag > threshold
Backup failed
MariaDB unavailable
Error rate increased
```

Avoid alerting on every harmless warning.

Too many alerts produce:

```text
Alert fatigue
```

---

# 51. Performance Baseline

Create a baseline such as:

```text
Normal CPU: 30–50%
Normal connections: 120
Normal query latency: 80 ms
Normal replication lag: < 5 sec
Normal disk utilization: 50%
```

Then:

```text
Current:
CPU = 95%
Connections = 250
Latency = 5 sec
```

Now you have a meaningful comparison.

---

# 52. Production Incident Example

Application:

> "Checkout is timing out."

DBA investigates:

```text
CPU = 35%
Memory = normal
Disk = normal
Connections = normal
```

Then:

```sql
SHOW FULL PROCESSLIST;
```

You find many sessions:

```text
Waiting for lock
```

Now the investigation moves toward:

```text
Lock contention
```

rather than CPU or memory.

That's real DBA troubleshooting.

---

# 53. Second Incident Example

Application:

> "Reports are extremely slow."

OS:

```text
CPU = 30%
Disk = normal
```

Processlist shows long-running report queries.

Run:

```sql
EXPLAIN ...
```

You discover:

```text
Huge row estimate
No useful index
```

Now the likely solution is:

```text
Query/index optimization
```

—not increasing server RAM.

---

# 54. Third Incident Example

MariaDB becomes unstable.

You discover:

```text
Filesystem = 100%
```

Then:

```text
Binary logs = hundreds of GB
```

Now investigate:

```text
Why are logs growing?
Is replication consuming them?
Is retention configured correctly?
Is a replica disconnected?
```

Do **not** blindly delete database files.

---

# 55. Performance Tuning Golden Rule

> **Measure → Analyze → Change → Measure again.**

Not:

> Change → Hope.

---

# 56. DBA Troubleshooting Checklist

```text
☐ Confirm symptom
☐ Check MariaDB availability
☐ Check CPU
☐ Check memory
☐ Check disk space
☐ Check disk I/O
☐ Check connections
☐ Check processlist
☐ Check locks
☐ Check long transactions
☐ Check slow queries
☐ EXPLAIN suspicious queries
☐ Check replication
☐ Check error logs
☐ Identify root cause
☐ Apply controlled fix
☐ Validate
☐ Monitor
☐ Document
```

---

# 57. DBA Challenge

### Q1

What are the four major areas you should investigate when MariaDB is slow?

### Q2

What command shows currently connected sessions?

### Q3

Why shouldn't you immediately increase `max_connections`?

### Q4

What does `EXPLAIN` help you understand?

### Q5

What is a deadlock?

### Q6

Why shouldn't you immediately restart MariaDB when it becomes slow?

### Q7

Why are indexes useful?

### Q8

Why can too many indexes be harmful?

### Q9

What is the correct performance-tuning mindset?

---

# Answers

### Q1

```text
CPU
Memory
Disk/I/O
SQL/locks
```

### Q2

```sql
SHOW FULL PROCESSLIST;
```

### Q3

Because additional connections consume resources and can make memory/CPU pressure worse.

### Q4

It helps you understand MariaDB's execution plan for a query, including access paths and estimated rows.

### Q5

A situation where transactions wait on each other in a cycle, preventing progress until MariaDB resolves the deadlock.

### Q6

Because restarting may temporarily hide the symptom without fixing the root cause.

### Q7

Indexes can allow MariaDB to find matching rows more efficiently instead of scanning large amounts of data.

### Q8

They consume storage and memory and add overhead to data modifications.

### Q9

```text
Measure
  ↓
Analyze
  ↓
Change
  ↓
Measure again
```

---

# ⭐ Lesson 9 Golden Rules

> **1. Never troubleshoot by guessing.**

> **2. Establish a baseline.**

> **3. Check CPU, memory, disk, SQL, and locks.**

> **4. `SHOW FULL PROCESSLIST` is one of your first investigation tools.**

> **5. Use `EXPLAIN` for query analysis.**

> **6. Don't blindly increase `max_connections` or memory settings.**

> **7. Don't kill sessions without understanding their impact.**

> **8. Don't restart MariaDB just to hide a performance problem.**

> **9. Measure before and after every performance change.**

> **10. Find the root cause, not merely the symptom.**

---

# Production DBA Mental Model

```text
                  USER REPORT
                      │
                      ▼
             "DATABASE IS SLOW"
                      │
                      ▼
                DEFINE SCOPE
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        SERVER      MARIADB       SQL
          │           │           │
       CPU/RAM      Sessions     Queries
       Disk/IO      Locks        EXPLAIN
          │           │           │
          └───────────┼───────────┘
                      ▼
                  ROOT CAUSE
                      │
                      ▼
                 CONTROLLED FIX
                      │
                      ▼
                   VALIDATE
                      │
                      ▼
                   MONITOR
                      │
                      ▼
                 DOCUMENT
```

# Next — Lesson 0.10: MariaDB High Availability, Replication & Production DBA Operations

This is the **final lesson of the 10-lesson series** and brings everything together:

```text
Replication
Primary / Replica
GTID
Binary logs
Failover
Switchover
High Availability
Disaster Recovery
Monitoring
Maintenance
Patching
Backup
Migration
Security
Production runbooks
Incident management
Linux + Windows
Complete DBA daily checklist
MariaDB DBA interview questions
Real-world production scenarios
```

We'll finish with a **complete MariaDB DBA production workflow from Day 1 through incident recovery**.
