# Lesson 6 — MariaDB Replication

**Goal:** Understand how MariaDB copies changes from one server to another, how to configure replication, monitor it, and troubleshoot common failures.

> **Important:** Replication improves availability and can support read scaling and recovery workflows, but **replication is not a backup**.

---

# 1. What Is Replication?

MariaDB replication copies database changes from one MariaDB server to another.

A common architecture is:

```text id="b5g3s1"
              Application
                   │
                   ▼
              PRIMARY
                   │
             Binary Log
                   │
                   ▼
                REPLICA
```

The **primary** accepts changes.

The **replica** receives and applies those changes.

Depending on your MariaDB version and architecture, terminology and supported replication configurations can vary, but the primary/replica model is the foundation we'll use.

---

# 2. Why Use Replication?

Replication can provide:

### High availability

If the primary fails, a replica may be promoted.

### Read scaling

Applications may send suitable read workloads to replicas.

```text id="n5f1e2"
                 Application
                /           \
               ▼             ▼
          PRIMARY          REPLICA
           writes            reads
```

### Disaster recovery

A replica can exist in another location.

### Reporting

Heavy reporting workloads can sometimes be directed to a replica rather than the primary.

---

# 3. Primary vs. Replica

Think of it like:

```text id="4s7b0h"
PRIMARY
  │
  │ writes changes
  ▼
Binary Log
  │
  ▼
REPLICA
  │
  │ reads/applies changes
  ▼
Replica Data
```

The primary records changes in its binary log.

The replica retrieves those changes and applies them locally.

---

# 4. Binary Logging

Replication depends on binary logging.

Check whether it is enabled:

```sql id="td3b9s"
SHOW VARIABLES LIKE 'log_bin';
```

You generally want:

```text id="u6z5hm"
log_bin = ON
```

You can inspect the binary-log configuration:

```sql id="rr3ojy"
SHOW VARIABLES LIKE 'log_bin_basename';
```

and:

```sql id="5zyzje"
SHOW VARIABLES LIKE 'log_bin_index';
```

The exact available variables depend on your MariaDB version/configuration.

---

# 5. GTID

GTID means **Global Transaction ID**.

Instead of relying only on binary-log filenames and positions, GTID provides transaction identifiers that can make replication management and failover workflows easier.

Conceptually:

```text id="y5uh6o"
Transaction A → GTID A
Transaction B → GTID B
Transaction C → GTID C
```

The replica can track which transactions it has received/applied.

This becomes particularly valuable when:

* Promoting replicas
* Reconnecting replicas
* Performing failover
* Reconfiguring replication

MariaDB's GTID implementation has its own syntax and behavior, so always use documentation appropriate to your installed MariaDB version.

---

# 6. Replication User

The replica needs credentials to connect to the primary.

On the primary, create a dedicated replication account.

For example:

```sql id="gk6x6q"
CREATE USER 'repl'@'10.0.0.20'
IDENTIFIED BY 'StrongReplicationPassword';
```

Then grant the appropriate replication privilege:

```sql id="3o5e47"
GRANT REPLICATION SLAVE
ON *.*
TO 'repl'@'10.0.0.20';
```

Depending on MariaDB version and configuration, privilege terminology may differ, so verify against your installed version.

The important principle is:

**Use a dedicated replication account.**

Don't use `root`.

---

# 7. Restrict the Replication User

Notice:

```text id="b3g2cp"
'repl'@'10.0.0.20'
```

This means the account is restricted to a particular source host.

That's much better than:

```text id="p7t7cx"
'repl'@'%'
```

when a specific replica address is known.

Again:

> **Restrict access as much as practical.**

---

# 8. Primary Configuration

A simplified conceptual configuration might include:

```ini id="7ed4qr"
[mariadbd]
server_id=1
log_bin
```

The exact configuration needed depends on MariaDB version and replication design.

`server_id` uniquely identifies a server in replication topology.

For example:

```text id="m2a1o8"
PRIMARY
server_id = 1

REPLICA
server_id = 2
```

Don't use the same server ID for both.

---

# 9. Replica Configuration

The replica might have:

```ini id="1y0d1u"
[mariadbd]
server_id=2
```

The important point is:

```text id="e4g7id"
Primary → unique ID
Replica → different unique ID
```

If you build multiple replicas:

```text id="ttr5kh"
Primary      server_id=1
Replica 1    server_id=2
Replica 2    server_id=3
Replica 3    server_id=4
```

---

# 10. Initial Data

Before replication can work correctly, the replica needs an appropriate starting dataset.

Conceptually:

```text id="h6qg2j"
Primary
   │
   ├── Create consistent backup
   │
   ▼
Backup
   │
   ▼
Restore onto Replica
   │
   ▼
Start replication
```

This is why **Lesson 5 — Backup & Restore** comes before replication.

You need to understand consistent data transfer before building replication.

---

# 11. Configure the Replica

The replica needs to know:

* Primary hostname/IP
* Replication username
* Replication password
* Replication coordinates/GTID information
* SSL/TLS settings when applicable

MariaDB replication configuration syntax can vary by version and setup.

A traditional position-based configuration may use information such as:

```text id="f7yl1w"
MASTER_HOST
MASTER_USER
MASTER_PASSWORD
MASTER_LOG_FILE
MASTER_LOG_POS
```

GTID-based configurations use GTID-related settings instead.

For production, prefer a replication design appropriate to your MariaDB version rather than copying an old tutorial verbatim.

---

# 12. Starting Replication

Once configured, replication is started on the replica.

Depending on your MariaDB version/configuration, commands such as:

```sql id="t8a4r0"
START SLAVE;
```

may be used.

Newer terminology/configurations may expose:

```sql id="j0g9d6"
START REPLICA;
```

The exact command available depends on the MariaDB version.

The key idea is:

```text id="7d8s7b"
Replica
   ↓
Connect to Primary
   ↓
Read binary-log events
   ↓
Apply changes
```

---

# 13. Checking Replication Status

One of your most important DBA commands is:

```sql id="2s2x1e"
SHOW SLAVE STATUS\G
```

On MariaDB versions/configurations supporting newer terminology, you may also encounter:

```sql id="r6p7zt"
SHOW REPLICA STATUS\G
```

The output contains critical replication information.

---

# 14. Important Replication Fields

When troubleshooting replication, pay attention to fields such as:

```text id="98o2vk"
Slave_IO_Running
Slave_SQL_Running
Seconds_Behind_Master
Last_IO_Error
Last_SQL_Error
Master_Host
Master_Log_File
Read_Master_Log_Pos
Exec_Master_Log_Pos
```

Depending on version, newer terminology may use corresponding `Replica_*` names.

---

# 15. Two Important Replication Processes

Conceptually, think of two major stages:

```text id="i1k1gl"
PRIMARY
   │
   ▼
I/O / Receiver
   │
   ▼
Relay Log
   │
   ▼
SQL / Applier
   │
   ▼
Replica Data
```

The receiver gets changes.

The applier executes those changes on the replica.

Therefore, you can have a situation where:

```text id="36gj7a"
Receiver: working
Applier: broken
```

or:

```text id="4u1s6p"
Receiver: broken
Applier: waiting
```

That's why replication troubleshooting requires more than checking one number.

---

# 16. `Seconds_Behind_Master`

You may encounter:

```text id="5m5d8m"
Seconds_Behind_Master
```

It can provide an indication of replication delay.

For example:

```text id="65hl97"
Seconds_Behind_Master = 0
```

suggests the replica is caught up according to that metric.

But:

> **Zero does not prove that replication is perfectly healthy in every sense.**

And:

> **A nonzero value doesn't automatically mean disaster.**

You need to investigate workload, network conditions, transaction size, and replica performance.

---

# 17. Replication Lag

Suppose the primary is processing:

```text id="8nkn8a"
10,000 transactions/minute
```

but the replica can only apply:

```text id="l2ndm8"
7,000 transactions/minute
```

The replica falls behind.

Conceptually:

```text id="pxyqbt"
PRIMARY
████████████████████
          ↓
      transaction stream
          ↓
REPLICA
████████████
          ↑
      can't keep up
```

Possible causes include:

* Slow disk
* Insufficient CPU
* Large transactions
* Lock contention
* Network latency
* Poor queries
* Replica resource constraints
* Heavy reporting workloads

---

# 18. Replication Failure

Imagine:

```text id="r6c7aa"
Slave_IO_Running: No
```

This means the replica's receiver/IO side isn't running.

Possible causes include:

* Network failure
* Authentication failure
* Primary unavailable
* Incorrect hostname
* Incorrect replication credentials
* TLS problems
* Binary-log availability problems

Look at:

```sql id="o6f0qz"
SHOW SLAVE STATUS\G
```

especially:

```text id="0v5x1s"
Last_IO_Error
```

The error message is often the starting point for diagnosis.

---

# 19. SQL/Applier Failure

Suppose:

```text id="4b2q2x"
Slave_IO_Running: Yes
Slave_SQL_Running: No
```

Now the replica can receive changes but can't apply them.

Possible causes:

* Duplicate-key error
* Missing table
* Schema mismatch
* Data inconsistency
* Constraint violation
* Corrupted data
* A problematic transaction

Check:

```text id="h3byyy"
Last_SQL_Error
```

This is one of the most important fields when diagnosing replication.

---

# 20. Don't Immediately Skip Errors

A dangerous beginner response is:

> "Replication is broken. Just skip the error."

That can create silent data inconsistency.

Before considering any error-skipping or recovery action, determine:

1. What caused the error?
2. Why did the primary and replica diverge?
3. Is the replica still trustworthy?
4. Can the transaction be safely repaired?
5. Should the replica be rebuilt instead?

Sometimes rebuilding a replica from a known-good backup is safer than accumulating inconsistencies.

---

# 21. Primary → Multiple Replicas

A common architecture is:

```text id="q6bquv3"
                  PRIMARY
                 /       \
                /         \
               ▼           ▼
          REPLICA 1    REPLICA 2
```

This can support:

* Read scaling
* Reporting
* Disaster recovery
* Operational redundancy

But now the DBA has more things to monitor.

---

# 22. Replication Monitoring

A production DBA should monitor things such as:

```text id="x3k7e5"
Replication running?
        ↓
Replication errors?
        ↓
Replication lag?
        ↓
Binary logs available?
        ↓
Replica disk space?
        ↓
Replica health?
```

A replica that is silently several days behind isn't a healthy disaster-recovery system.

---

# 23. Replication vs. Backup

This deserves repeating.

Imagine:

```text id="l6c4w0"
Primary
   │
   ├────────► Replica
   │
   └────────► Backup
```

If someone executes:

```sql id="qlf0b4"
DROP TABLE customers;
```

the bad operation may replicate.

But a historical backup may allow you to recover the table.

Therefore:

```text id="0f9a6d"
Replication → Availability / redundancy

Backup → Historical recovery
```

You generally want both.

---

# 24. Lesson 6 Lab

For a **safe lab**, use two disposable MariaDB instances/VMs/containers rather than experimenting against production.

### Primary

Configure a unique server ID:

```ini id="c6q5gv"
[mariadbd]
server_id=1
log_bin
```

### Replica

Configure:

```ini id="0g4khr"
[mariadbd]
server_id=2
```

On the primary, create a replication user appropriate for your test network:

```sql id="4ptn8b"
CREATE USER 'repl'@'10.0.0.20'
IDENTIFIED BY 'StrongReplicationPassword';

GRANT REPLICATION SLAVE
ON *.*
TO 'repl'@'10.0.0.20';
```

Then practice identifying:

```sql id="t7c2q0"
SHOW VARIABLES LIKE 'log_bin';

SHOW VARIABLES LIKE 'server_id';
```

On the replica, practice:

```sql id="fr0h6y"
SHOW SLAVE STATUS\G
```

or, where supported by your version:

```sql id="7h2t8n"
SHOW REPLICA STATUS\G
```

Study these fields:

```text id="n9f13v"
Master_Host
Slave_IO_Running
Slave_SQL_Running
Seconds_Behind_Master
Last_IO_Error
Last_SQL_Error
```

**Don't perform replication experiments against a production server.**

---

# Lesson 6 — DBA Knowledge Check

Make sure you understand:

**Replication**
→ Copies changes between MariaDB servers.

**Primary**
→ Source of replicated changes.

**Replica**
→ Receives and applies changes.

**Binary log**
→ Source of replication events.

**GTID**
→ Transaction identification mechanism useful for replication management.

**Replication user**
→ Dedicated account used by the replica.

**`server_id`**
→ Identifies a server in replication topology.

**Replication lag**
→ Replica is behind the primary.

**IO/receiver problem**
→ Replica can't receive changes.

**SQL/applier problem**
→ Replica received changes but can't apply them.

**Replication ≠ Backup**

---

## DBA Troubleshooting Formula

When replication breaks, don't guess.

Use:

```text id="s1k9q2"
1. Check replication status
          ↓
2. Check IO/receiver state
          ↓
3. Check SQL/applier state
          ↓
4. Read the actual error
          ↓
5. Identify the root cause
          ↓
6. Repair safely
          ↓
7. Verify consistency
          ↓
8. Monitor afterward
```

That workflow is far more valuable than memorizing commands.

### Next: Lesson 7 — MariaDB Performance & Monitoring

We'll learn **`EXPLAIN`, slow-query analysis, indexes, locks, `SHOW PROCESSLIST`, performance metrics, CPU/RAM/disk bottlenecks, query optimization, and how to diagnose "MariaDB is slow" like a DBA.**
