# Lesson 8 — MariaDB High Availability

**Goal:** Learn how to design MariaDB so that the failure of one server doesn't necessarily become an application outage.

This lesson moves from:

```text
"How do I run MariaDB?"
```

to:

```text
"How do I keep MariaDB available when something fails?"
```

---

# 1. What Is High Availability?

**High Availability (HA)** means designing a system to minimize service interruption when components fail.

A basic single-server architecture is:

```text
Application
     │
     ▼
MariaDB
     │
     ▼
Storage
```

If the MariaDB server fails:

```text
Application
     │
     ▼
MariaDB ❌
```

The application may become unavailable.

HA introduces redundancy:

```text
              Application
                   │
              ┌────┴────┐
              ▼         ▼
           Server A   Server B
              │         │
              └────┬────┘
                   ▼
              HA mechanism
```

---

# 2. HA vs. Backup

These solve different problems.

### HA

Answers:

> **"How do I keep the service running?"**

### Backup

Answers:

> **"How do I recover lost or damaged data?"**

You need to think about both:

```text
High Availability
       +
Backup & Recovery
       =
Resilient Database
```

---

# 3. MariaDB HA Options

MariaDB environments can use different architectures, including:

* Primary/replica replication
* Automated failover systems
* MariaDB Galera Cluster
* Load balancers/proxies
* Multiple application servers
* Distributed storage architectures

For this lesson, we'll focus heavily on **MariaDB Galera Cluster**, because it is an important MariaDB HA technology.

---

# 4. Galera Cluster

Galera provides synchronous, virtually synchronous multi-primary clustering.

Conceptually:

```text
             ┌───────────────┐
             │               │
             ▼               ▼
          Node 1 ◄────────► Node 2
             ▲               ▲
             │               │
             └──────► Node 3
```

The nodes communicate with each other as part of the cluster.

A simplified three-node architecture:

```text
             Galera Cluster

        ┌─────────────────────┐
        │                     │
        ▼                     ▼
     Node 1 ◄──────────────► Node 2
        ▲                     ▲
        │                     │
        └──────── Node 3 ─────┘
```

---

# 5. Why Three Nodes?

A three-node cluster is common because of **quorum**.

Suppose you have:

```text
Node 1
Node 2
Node 3
```

If Node 1 fails:

```text
Node 2 + Node 3
```

Two nodes remain.

They can maintain a majority.

But with only two nodes:

```text
Node 1
Node 2
```

If communication breaks:

```text
Node 1 ←X→ Node 2
```

each side could potentially believe the other has failed.

That creates difficult split-brain scenarios.

---

# 6. Quorum

Quorum means a sufficient number of cluster members agree that the cluster can continue operating.

For three nodes:

```text
3 nodes
↓
majority = 2
```

Therefore:

```text
Node 1 ❌
Node 2 ✓
Node 3 ✓
```

The remaining two have a majority.

But:

```text
Node 1 ✓
Node 2 ❌
Node 3 ❌
```

only one node remains.

The cluster cannot safely assume it has majority authority.

---

# 7. Split-Brain

This is one of the most important HA concepts.

Imagine:

```text
Node 1     X     Node 2
```

The network connection between them fails.

Both nodes are still running.

If both independently accept writes, you can end up with two conflicting views of the system.

That's:

**split-brain.**

Conceptually:

```text
          Network failure
               X
              / \
             /   \
        Node A   Node B
          │         │
       writes     writes
          │         │
       Data A     Data B
```

This is extremely dangerous.

HA systems use quorum and membership protocols to prevent unsafe independent operation.

---

# 8. Synchronous vs. Asynchronous Replication

This distinction is important.

### Asynchronous

```text
Primary
   │
   └────► Replica
```

The primary can continue without waiting for the replica to finish applying the change.

There may be replication lag.

### Synchronous / virtually synchronous

The cluster coordinates transaction certification/replication more tightly among nodes.

Conceptually:

```text
Transaction
     │
     ├──► Node 1
     ├──► Node 2
     └──► Node 3
```

This can reduce some forms of replication lag and improve consistency characteristics, but it introduces network and coordination requirements.

---

# 9. Network Is Critical

A database cluster is only as healthy as its underlying communication.

Galera nodes require reliable network communication.

You need to consider:

* Latency
* Packet loss
* Bandwidth
* Firewall rules
* DNS/hostname resolution
* Network segmentation
* Redundant network paths

A cluster spread across distant geographic locations can have very different performance characteristics from a cluster in one data center.

Don't assume:

> "More geographically distributed = automatically better."

Distance introduces latency.

---

# 10. Node Failure

Suppose:

```text id="c6m1p8"
Node 1 ✓
Node 2 ✓
Node 3 ✓
```

Node 2 crashes:

```text id="z4q7x1"
Node 1 ✓
Node 2 ❌
Node 3 ✓
```

The remaining nodes can potentially continue because the cluster still has quorum.

The failed node can later rejoin.

The exact recovery process depends on the cluster state and MariaDB/Galera configuration.

---

# 11. Node Rejoining

When a failed node comes back:

```text
Node 2
   │
   ▼
Reconnect to cluster
   │
   ▼
Determine missing state
   │
   ▼
Synchronize
   │
   ▼
Become healthy cluster member
```

Galera can use state-transfer mechanisms to bring a node back into sync.

Two important concepts you'll encounter are:

* **IST** — Incremental State Transfer
* **SST** — State Snapshot Transfer

---

# 12. IST

IST transfers missing write-set information when the joining node is sufficiently close to the cluster state and the required history is available.

Conceptually:

```text
Node was down
     ↓
Missed changes A, B, C
     ↓
Cluster still has required history
     ↓
Transfer A, B, C
     ↓
Node catches up
```

This can be much faster than transferring the entire dataset.

---

# 13. SST

SST is a more substantial state transfer.

Conceptually:

```text
Healthy Node
     │
     │ full state transfer
     ▼
Joining Node
```

It can involve transferring a large amount of data.

Therefore:

> **SST can be expensive on large production databases.**

Network bandwidth, storage, and donor-node impact all matter.

---

# 14. Donor Node

During state transfer, another healthy node may act as the donor.

Conceptually:

```text
Cluster:

Node 1 ─────► Node 3
 donor         joining

Node 2
 healthy
```

The donor's resources can be affected by the transfer.

Therefore, cluster planning should consider which nodes are suitable donors.

---

# 15. Application Connections

Database HA isn't complete if the application only knows one database IP.

Bad design:

```text
Application
      │
      ▼
Node 1
```

Node 1 fails:

```text
Application
      │
      ▼
Node 1 ❌
```

Better:

```text
Application
      │
      ▼
Proxy / Load Balancer
      │
   ┌──┼──┐
   ▼  ▼  ▼
  N1  N2  N3
```

The application connects to an endpoint that can route traffic appropriately.

---

# 16. MariaDB MaxScale

MariaDB environments can use **MariaDB MaxScale** as a database proxy for functions such as:

* Connection routing
* Read/write splitting
* Monitoring
* Failover-related routing
* Query filtering

Conceptually:

```text
Application
     │
     ▼
  MaxScale
   /    \
  ▼      ▼
Node 1  Node 2
```

The proxy can hide some database topology details from applications.

But remember:

**A proxy doesn't magically solve every HA problem.**

You still need a healthy database cluster and carefully designed failure procedures.

---

# 17. Automatic Failover

Imagine:

```text
Primary
   │
   ▼
Replica
```

Primary fails.

An HA system may perform:

```text
Failure detected
       ↓
Select candidate
       ↓
Promote candidate
       ↓
Redirect application
       ↓
Verify service
```

Automatic failover sounds simple, but production failover requires careful handling of:

* Data consistency
* Network partitions
* Stale replicas
* Application connections
* DNS/proxy behavior
* Transactions in progress
* Old primary returning
* Preventing two writable primaries

---

# 18. Fencing

Fencing is an important HA safety mechanism.

Suppose the old primary is unreachable from the application but still running.

If you promote another node while the old primary continues accepting writes, you can create:

```text
Old Primary
     │
   writes
     │
     ▼
   Data A


New Primary
     │
   writes
     │
     ▼
   Data B
```

That's dangerous.

Fencing aims to prevent the old node from continuing to act as an authoritative writable server.

This is one reason production HA requires more than simply running two database servers.

---

# 19. Disaster Recovery vs. HA

Imagine an entire data center is lost:

```text
Data Center A
┌─────────────────┐
│ MariaDB Cluster │
│      ❌         │
└─────────────────┘
```

If all HA nodes were in that same data center, your HA solution may not protect you from the complete site failure.

That's where disaster recovery architecture comes in:

```text
Data Center A
     │
     │ replication/backup
     ▼
Data Center B
```

You need to think about **failure domains**.

---

# 20. Failure Domains

A failure domain is a group of components that can fail together.

Examples:

```text
Same disk
Same server
Same rack
Same power source
Same network switch
Same data center
Same geographic region
```

Three database nodes in the same physical server are not meaningful HA.

Three nodes in the same rack are better, but still share some failure risks.

Good HA design distributes nodes across appropriate independent failure domains.

---

# 21. HA Doesn't Eliminate Backups

Imagine someone runs:

```sql id="j6m3v8"
DELETE FROM customers;
```

If the database cluster replicates the transaction:

```text
Node 1 → DELETE
Node 2 → DELETE
Node 3 → DELETE
```

Your HA cluster has successfully replicated the disaster.

Therefore:

```text
HA protects availability.

Backups protect recoverability.
```

You need both.

---

# 22. Monitoring a Cluster

A DBA should monitor:

```text
Cluster membership
       ↓
Quorum
       ↓
Node health
       ↓
Replication state
       ↓
Flow control
       ↓
Network
       ↓
Disk space
       ↓
CPU / RAM / I/O
       ↓
Application errors
```

A cluster that is technically "running" can still be unhealthy.

---

# 23. Flow Control

In Galera, if one node cannot keep up, cluster-wide flow control can become important.

Conceptually:

```text
Node 1 → fast
Node 2 → fast
Node 3 → slow
```

If Node 3 falls too far behind, the cluster may need to regulate transaction flow so the slow node can catch up.

This means:

> **The slowest node can affect cluster performance.**

That's one reason node sizing and network/storage performance matter.

---

# 24. HA Testing

Don't wait for a real disaster to discover that failover doesn't work.

Perform controlled tests such as:

```text
Test 1 → Stop one node
Test 2 → Restore the node
Test 3 → Test application connectivity
Test 4 → Test proxy routing
Test 5 → Simulate network interruption
Test 6 → Verify recovery
```

Document:

```text
Expected result
Actual result
Recovery time
Problems encountered
Corrective actions
```

This becomes your **failure test evidence**.

---

# 25. Lesson 8 Lab — Architecture Exercise

You don't need to destroy a real server to practice HA concepts.

Design this architecture on paper:

```text
                  Application
                       │
                       ▼
                  DB Proxy
                 /    |    \
                ▼     ▼     ▼
             Node 1 Node 2 Node 3
                \      |      /
                 \     |     /
                  Galera Cluster
```

Answer:

### Question 1

What happens if Node 1 fails?

### Question 2

What happens if Node 2 fails?

### Question 3

What happens if Nodes 2 and 3 fail?

### Question 4

What happens if the network between Node 1 and the other nodes breaks?

### Question 5

Where does the application connect?

### Question 6

Where are backups stored?

### Question 7

What happens if the entire data center fails?

### Question 8

How will you know that the cluster is unhealthy?

---

# 26. Production HA Checklist

Before calling a MariaDB environment "highly available," verify:

```text id="q2c8m7"
☐ Multiple database nodes
☐ Independent failure domains
☐ Quorum understood
☐ Network requirements understood
☐ Application failover path
☐ Proxy/load-balancer configuration
☐ Backup strategy
☐ Disaster-recovery strategy
☐ Monitoring
☐ Alerting
☐ Tested failover
☐ Tested recovery
☐ Documented procedures
```

---

# Lesson 8 — DBA Knowledge Check

You should understand:

**High Availability**
→ Reduce service interruption.

**Galera**
→ MariaDB clustering technology providing virtually synchronous multi-primary replication.

**Quorum**
→ Majority needed for safe cluster operation.

**Split-brain**
→ Multiple isolated components incorrectly acting as independent authorities.

**IST**
→ Incremental state transfer.

**SST**
→ Larger state snapshot transfer.

**Proxy**
→ Provides an application-facing database endpoint/routing layer.

**Fencing**
→ Prevents an old/isolated primary from continuing unsafe writes.

**Failure domains**
→ Separate components so one failure doesn't take everything down.

**HA ≠ Backup**

---

# The Golden HA Rule

> **Don't design for "nothing will fail." Design for "something will fail, and we know exactly what happens next."**

A good DBA can answer:

**"If this MariaDB node dies at 2:00 AM, what happens at 2:01 AM?"**

If nobody knows, the system isn't truly ready for failure.

### Next: Lesson 9 — MariaDB Maintenance & Troubleshooting

We'll cover **database corruption, crashes, disk-full situations, locks, deadlocks, failed startups, replication failures, error logs, emergency troubleshooting, and systematic recovery procedures.**
