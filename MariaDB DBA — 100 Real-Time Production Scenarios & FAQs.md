# MariaDB DBA — 100 Real-Time Production Scenarios & FAQs

Below is a **production-oriented MariaDB DBA interview + troubleshooting bank**.
Each scenario has **5–6 practical points**: what to check, commands/concepts, likely cause, action, and validation.

---

## 1. MariaDB service won't start

**Q: MariaDB suddenly fails to start. What do you do?**

1. Check service status: `systemctl status mariadb` on Linux or the MariaDB Windows service.
2. Review the MariaDB error log and OS/Event Viewer logs.
3. Check for configuration errors, port conflicts, permissions, or missing files.
4. Check disk space with `df -h` on Linux or Windows disk monitoring.
5. Don't immediately delete or modify database files.
6. Correct the root cause, start MariaDB, and validate with `mariadb-admin ping`.

---

## 2. MariaDB is running but application cannot connect

**Q: MariaDB is UP, but the application gets connection errors.**

1. Verify MariaDB is listening on the expected port.
2. Check `bind-address` and network/firewall configuration.
3. Verify the application username, password, database, and host.
4. Check the account with `SELECT User, Host FROM mysql.user;`.
5. Test connectivity from the application server.
6. Check MariaDB logs for authentication or connection errors.

---

## 3. "Access denied" error

**Q: Application receives `Access denied for user`.**

1. Identify the exact `User` and `Host` in the error.
2. Check whether that exact account exists.
3. Run `SHOW GRANTS FOR 'user'@'host';`.
4. Verify password/authentication configuration.
5. Check whether the application is connecting from the expected IP/hostname.
6. Correct the account/grants and test again.

---

## 4. Too many connections

**Q: MariaDB reports `Too many connections`.**

1. Check `SHOW VARIABLES LIKE 'max_connections';`.
2. Check `Threads_connected` and `Threads_running`.
3. Inspect `SHOW FULL PROCESSLIST;`.
4. Determine whether connections are leaking from the application.
5. Don't blindly increase `max_connections`.
6. Fix connection pooling/leaks and increase the limit only when justified.

---

## 5. MariaDB is very slow

**Q: Users report that the database is slow.**

1. Check CPU, memory, disk I/O, and network.
2. Run `SHOW FULL PROCESSLIST;`.
3. Look for long-running or waiting queries.
4. Investigate locks and transactions.
5. Use `EXPLAIN` on suspicious SQL.
6. Measure before and after any optimization.

---

## 6. One query takes 30 seconds

1. Capture the exact SQL.
2. Run `EXPLAIN` against it.
3. Check whether appropriate indexes exist.
4. Examine estimated rows and access type.
5. Check whether unnecessary columns/rows are being retrieved.
6. Test the optimized query and compare execution time.

---

## 7. Query suddenly became slow

1. Compare current and historical performance.
2. Run `EXPLAIN` again.
3. Check whether indexes/statistics/data distribution changed.
4. Check server CPU, memory, and I/O.
5. Look for concurrent workload or locking.
6. Determine whether the query plan changed.

---

## 8. Full table scan

1. Use `EXPLAIN`.
2. Check whether `key` is NULL when an index should be useful.
3. Verify the filtering columns.
4. Consider an appropriate index.
5. Check whether functions/casts prevent efficient index use.
6. Re-run `EXPLAIN` after changes.

---

## 9. Should every column be indexed?

1. No.
2. Indexes consume disk and memory.
3. INSERT/UPDATE/DELETE operations incur index-maintenance overhead.
4. Indexes should support real query patterns.
5. Check existing indexes before adding new ones.
6. Remove redundant indexes only after impact analysis.

---

## 10. Composite index isn't helping

1. Check the index column order.
2. Compare it with actual query predicates.
3. Run `EXPLAIN`.
4. Check selectivity and estimated rows.
5. Verify whether another index is being chosen.
6. Test alternative index designs against real workload.

---

# Connections & Sessions

## 11. One session is consuming CPU

1. Run `SHOW FULL PROCESSLIST;`.
2. Identify the connection ID and SQL.
3. Determine whether the query is legitimate.
4. Check execution plan and transaction state.
5. If confirmed harmful, consider `KILL <id>;`.
6. Investigate why the application generated the query.

---

## 12. Can I kill a long-running query?

1. First identify what the query is doing.
2. Determine whether it is blocking other transactions.
3. Check the owning user/application.
4. Understand the consequences of termination.
5. Use `KILL` only when justified.
6. Investigate the underlying reason afterward.

---

## 13. Hundreds of sleeping connections

1. Check `Threads_connected`.
2. Inspect `SHOW FULL PROCESSLIST`.
3. Identify the application creating them.
4. Review connection-pool configuration.
5. Determine whether connections are being reused correctly.
6. Fix the application/pool instead of simply increasing limits.

---

## 14. Threads_running suddenly increases

1. Check CPU and disk I/O.
2. Inspect `SHOW FULL PROCESSLIST`.
3. Identify common SQL statements.
4. Check for locks.
5. Look for a traffic spike.
6. Determine whether one query pattern is responsible.

---

## 15. Application connection pool is exhausted

1. Check MariaDB connection count.
2. Check application pool size.
3. Look for long-running transactions.
4. Check for leaked connections.
5. Verify queries return/close connections correctly.
6. Correct the pool/application behavior.

---

# Locks & Transactions

## 16. Query is waiting for a lock

1. Identify the waiting session.
2. Identify the blocking session.
3. Determine which transaction holds the lock.
4. Check how long the transaction has been open.
5. Avoid killing sessions without understanding consequences.
6. Fix transaction/locking design if recurring.

---

## 17. Deadlock occurred

1. Identify the transactions involved.
2. Review MariaDB/InnoDB deadlock information.
3. Determine which tables/indexes were locked.
4. Check transaction ordering in application code.
5. Ensure the application handles deadlock retry appropriately.
6. Fix recurring transaction-ordering problems.

---

## 18. Long-running transaction

1. Identify the session and transaction.
2. Check whether it holds locks.
3. Determine why it has remained open.
4. Check application transaction handling.
5. Terminate it only after impact assessment.
6. Prevent recurrence through application design/monitoring.

---

## 19. DELETE is taking too long

1. Check `EXPLAIN` for the corresponding selection.
2. Verify filtering columns are indexed appropriately.
3. Check for locks.
4. Check transaction size.
5. Large deletes may create substantial undo/log activity.
6. Consider controlled batching where appropriate.

---

## 20. UPDATE is blocking the application

1. Identify the transaction.
2. Check affected rows and execution plan.
3. Check lock waits.
4. Determine whether the UPDATE is missing an index.
5. Check transaction duration.
6. Optimize and retest before production rollout.

---

# Backup & Recovery

## 21. Backup failed

1. Identify the backup method.
2. Review backup logs.
3. Check disk space.
4. Check permissions and connectivity.
5. Determine whether the previous successful backup is still usable.
6. Fix the failure and immediately establish a verified backup.

---

## 22. Backup completed but restore fails

1. Treat the backup as untrusted until validated.
2. Review restore errors.
3. Verify backup completeness.
4. Test against a compatible MariaDB environment.
5. Identify version/configuration incompatibilities.
6. Update the backup process to include regular restore testing.

---

## 23. Someone accidentally dropped a table

1. Stop actions that could overwrite recovery evidence.
2. Determine when the DROP occurred.
3. Identify the latest usable backup.
4. Use binary logs where applicable to recover to the desired point.
5. Restore into an isolated environment first.
6. Validate the recovered table before returning it to production.

---

## 24. Database was accidentally deleted

1. Do not panic or overwrite the environment.
2. Determine exactly what was deleted.
3. Identify the latest valid backup.
4. Determine whether point-in-time recovery is possible.
5. Restore to a safe environment first.
6. Validate data before production recovery.

---

## 25. How do you perform point-in-time recovery?

1. Start with a valid base backup.
2. Identify the desired recovery timestamp.
3. Obtain the required binary logs.
4. Apply changes up to the required point.
5. Validate the restored state.
6. Redirect the application only after verification.

---

## 26. Backup files are consuming all disk space

1. Identify the largest files/directories.
2. Check backup retention policy.
3. Determine which backups are still required.
4. Don't delete files blindly.
5. Move/archive valid backups to appropriate storage.
6. Correct retention and capacity planning.

---

## 27. Binary logs are consuming huge disk space

1. Determine why binary logs are being retained.
2. Check replication topology.
3. Verify whether replicas are healthy.
4. Review configured retention/purge policy.
5. Never manually delete active database files.
6. Resolve the underlying retention/replication problem first.

---

# Replication

## 28. Replica is lagging

1. Measure the actual lag.
2. Check replica CPU, memory, and disk I/O.
3. Look for long-running transactions.
4. Check network connectivity.
5. Determine whether the replica is overloaded by read/reporting workloads.
6. Fix the bottleneck and monitor recovery.

---

## 29. Replication stopped

1. Check replication status.
2. Read the replication error.
3. Determine whether the SQL/apply or I/O side stopped.
4. Check network and authentication.
5. Verify binary logs and transaction state.
6. Repair replication using a controlled procedure rather than guessing.

---

## 30. Replica has SQL errors

1. Capture the exact error.
2. Identify the transaction causing it.
3. Check whether source and replica schemas/configuration differ.
4. Determine whether data diverged.
5. Don't blindly skip transactions.
6. Correct the root cause and verify consistency.

---

## 31. Replica disk is full

1. Check filesystem usage.
2. Identify large files.
3. Check database and binary logs.
4. Determine whether replication is still healthy.
5. Free space safely according to your retention policy.
6. Resume/validate replication afterward.

---

## 32. Replica is slower than primary

1. Compare CPU/memory/storage.
2. Check replica workload.
3. Check replication lag.
4. Check long-running read queries.
5. Compare configuration and hardware.
6. Optimize the replica without compromising replication.

---

## 33. Can a replica be used for reporting?

1. Yes, depending on workload and consistency requirements.
2. Assess replication lag.
3. Separate reporting workload from critical application traffic.
4. Monitor replica resource consumption.
5. Ensure reports don't interfere with replication.
6. Define acceptable data freshness.

---

## 34. Replica is showing stale data

1. Check replication lag.
2. Confirm the application is reading from the replica.
3. Check whether the latest transaction has reached it.
4. Determine whether read-after-write consistency is required.
5. Route critical reads appropriately.
6. Fix replication lag if unexpectedly high.

---

## 35. Should replicas be backed up?

1. It depends on the backup architecture.
2. A replica is not a substitute for an independent backup.
3. Backups still need separate storage and retention.
4. Consider replica workload and backup impact.
5. Test restoration.
6. Document which server is authoritative for backups.

---

# HA & Failover

## 36. Primary server failed

1. Confirm the primary failure.
2. Check whether the replica is healthy.
3. Determine replication state and lag.
4. Fence/isolate the old primary if necessary.
5. Promote the appropriate replica using the approved runbook.
6. Redirect applications and validate.

---

## 37. When should you fail over?

1. Failover should be based on defined operational criteria.
2. Confirm the primary is actually unavailable or unsafe.
3. Check candidate replica health.
4. Assess data-loss risk.
5. Follow the approved HA procedure.
6. Document the event afterward.

---

## 38. Planned switchover

1. Verify target replica health.
2. Confirm replication is caught up.
3. Stop/freeze writes as required.
4. Perform final synchronization.
5. Promote the target.
6. Redirect and validate the application.

---

## 39. Split brain occurred

1. Immediately determine which nodes are accepting writes.
2. Isolate/fence one side.
3. Prevent further divergent writes.
4. Identify the authoritative data source.
5. Reconcile/rebuild the affected node carefully.
6. Review HA safeguards to prevent recurrence.

---

## 40. Failover succeeded but application still fails

1. Check application database hostname.
2. Check DNS/load balancer/VIP configuration.
3. Check connection pools.
4. Verify user permissions on the new primary.
5. Test connectivity from the application tier.
6. Validate application transactions.

---

# Installation & Configuration

## 41. MariaDB installation on Linux

1. Confirm supported OS/MariaDB versions.
2. Install from an appropriate MariaDB repository/package source.
3. Start and enable the service.
4. Apply initial security configuration.
5. Validate version, port, and data directory.
6. Test local and application connectivity.

---

## 42. MariaDB installation on Windows

1. Confirm OS and MariaDB compatibility.
2. Install MariaDB using the appropriate Windows distribution/package.
3. Configure the service and data directory.
4. Secure the installation.
5. Verify service status and port.
6. Test with the MariaDB client.

---

## 43. MariaDB won't start after configuration change

1. Identify the exact configuration change.
2. Review MariaDB error logs.
3. Check configuration syntax/support.
4. Revert the problematic change if necessary.
5. Start MariaDB and verify.
6. Apply configuration changes one at a time in future.

---

## 44. Port 3306 is unavailable

1. Check which process owns the port.
2. Confirm MariaDB's configured port.
3. Check whether another MariaDB instance is running.
4. Review firewall configuration.
5. Change the port only when architecturally justified.
6. Update clients/application configuration accordingly.

---

## 45. MariaDB uses too much memory

1. Check OS memory usage.
2. Check MariaDB buffer/cache settings.
3. Check number of connections.
4. Check temporary table/workload behavior.
5. Check whether the OS is swapping.
6. Tune based on measurements rather than a generic formula.

---

## 46. MariaDB CPU usage is 100%

1. Identify whether MariaDB is actually consuming the CPU.
2. Check active queries.
3. Look for inefficient SQL.
4. Check concurrent workload.
5. Use `EXPLAIN` on expensive queries.
6. Fix workload/query issues before simply adding CPU.

---

# Patching & Upgrades

## 47. MariaDB patching production

1. Check the exact source and target versions.
2. Review release notes and compatibility.
3. Back up the database.
4. Test the patch in a non-production environment.
5. Prepare rollback and validation procedures.
6. Monitor closely after patching.

---

## 48. Patch caused application errors

1. Capture the exact application/database error.
2. Compare behavior with the previous version.
3. Check changed configuration/authentication behavior.
4. Review MariaDB error logs.
5. Decide whether rollback is safe and necessary.
6. Perform root-cause analysis before retrying.

---

## 49. Can you patch a replica first?

1. In a suitable HA architecture, often yes.
2. Verify replication and compatibility first.
3. Patch the replica.
4. Validate it.
5. Perform controlled switchover if appropriate.
6. Patch the former primary afterward.

---

## 50. Major version upgrade

1. Read the supported upgrade path.
2. Test application compatibility.
3. Review configuration changes.
4. Take and verify backups.
5. Perform a full test upgrade.
6. Execute production upgrade with rollback/recovery planning.

---

# Migration

## 51. Windows → Linux migration

1. Inventory source MariaDB version and databases.
2. Build and validate the Linux target.
3. Use an appropriate logical/physical/replication migration method.
4. Transfer data securely.
5. Validate users, schemas, objects, and data.
6. Perform controlled application cutover.

---

## 52. Linux → Windows migration

1. Inventory source and target versions.
2. Prepare Windows MariaDB.
3. Prefer a migration method appropriate for cross-platform movement.
4. Restore/import data.
5. Validate configuration and permissions.
6. Test application connectivity before cutover.

---

## 53. Migration completed but row counts differ

1. Identify affected tables.
2. Determine whether the difference is expected.
3. Check dump/restore logs.
4. Compare source and target transaction state.
5. Validate critical records independently.
6. Do not cut over until discrepancies are understood.

---

## 54. Migration completed but application can't connect

1. Verify target MariaDB is running.
2. Check hostname/port.
3. Check firewall/network access.
4. Check application account and password.
5. Check grants and host restrictions.
6. Test from the application server.

---

## 55. Migration rollback needed

1. Determine whether new writes occurred on the target.
2. Stop/freeze writes if required.
3. Preserve evidence and transaction information.
4. Determine how target-only changes will be handled.
5. Redirect application only after rollback validation.
6. Perform root-cause analysis before another cutover.

---

# Storage & Operating System

## 56. Filesystem is 95% full

1. Identify the filesystem.
2. Identify the largest consumers.
3. Check database/log/backup growth.
4. Expand storage or safely remove/archive eligible files.
5. Never delete MariaDB data files blindly.
6. Add capacity alerts before the filesystem reaches 100%.

---

## 57. Linux server is swapping

1. Check `free -h`.
2. Check `vmstat`.
3. Review MariaDB memory configuration.
4. Check connection count.
5. Check other processes consuming RAM.
6. Reduce memory pressure before the server becomes unstable.

---

## 58. Disk I/O is extremely high

1. Check OS-level I/O.
2. Identify MariaDB workload.
3. Look for large scans or sorting.
4. Check temporary tables and long-running queries.
5. Check backup/reporting workload.
6. Determine whether storage capacity/performance needs improvement.

---

## 59. Windows disk is full

1. Identify the affected drive.
2. Check MariaDB data/log/backup locations.
3. Review Windows Event Viewer.
4. Review MariaDB logs.
5. Safely archive/delete eligible non-database files.
6. Expand storage and correct retention issues.

---

## 60. MariaDB server unexpectedly rebooted

1. Confirm OS reboot time.
2. Review OS logs.
3. Review MariaDB error logs.
4. Check memory/CPU/disk events.
5. Determine whether crash recovery completed.
6. Validate database/application health afterward.

---

# Security

## 61. How do you secure a new MariaDB server?

1. Remove/disable unnecessary accounts.
2. Apply least-privilege permissions.
3. Restrict network access.
4. Secure authentication and passwords.
5. Configure TLS where required.
6. Keep MariaDB and OS patched.

---

## 62. Application wants root access

1. Do not normally give the application root privileges.
2. Create a dedicated application account.
3. Grant only required privileges.
4. Restrict the account's allowed host.
5. Store credentials securely.
6. Audit the account periodically.

---

## 63. User has excessive privileges

1. Run `SHOW GRANTS`.
2. Identify privileges actually required.
3. Remove unnecessary permissions.
4. Test the application.
5. Document the intended privilege model.
6. Review privileges periodically.

---

## 64. Suspicious database login

1. Identify account, host, and timestamp.
2. Review MariaDB and infrastructure logs.
3. Determine whether the access is legitimate.
4. Disable/rotate credentials if compromise is suspected.
5. Preserve relevant evidence.
6. Follow your security incident process.

---

## 65. How should database passwords be handled?

1. Never hard-code credentials in scripts or source code.
2. Use an approved secrets-management mechanism.
3. Restrict who can retrieve secrets.
4. Rotate credentials according to policy.
5. Avoid exposing passwords in logs.
6. Test credential rotation before production use.

---

# Monitoring

## 66. What should you monitor?

1. MariaDB availability.
2. CPU/memory/disk.
3. Connections and threads.
4. Query latency/errors.
5. Replication health and lag.
6. Backup success/failure.

---

## 67. Disk alert keeps firing

1. Check actual disk growth.
2. Identify which files are growing.
3. Determine whether growth is expected.
4. Review retention policies.
5. Increase capacity if required.
6. Avoid simply increasing the alert threshold.

---

## 68. Backup alert says failure

1. Check the backup job log.
2. Determine whether failure is transient or persistent.
3. Check storage, credentials, permissions, and connectivity.
4. Verify the last successful backup.
5. Correct the problem and rerun.
6. Validate the resulting backup.

---

## 69. Monitoring says MariaDB is down

1. Verify from the server itself.
2. Check service status.
3. Test with `mariadb-admin ping`.
4. Check network/monitoring connectivity.
5. Review MariaDB and OS logs.
6. Avoid restarting until you understand the state when possible.

---

## 70. Alert storm during incident

1. Identify the primary incident.
2. Group dependent alerts.
3. Avoid reacting independently to every symptom.
4. Fix the underlying failure.
5. Review monitoring dependencies afterward.
6. Improve alert deduplication.

---

# Logs

## 71. Error log is growing rapidly

1. Check current errors/warnings.
2. Identify repeating messages.
3. Determine whether the messages indicate a real problem.
4. Check disk space.
5. Correct the underlying issue.
6. Implement appropriate log retention.

---

## 72. Slow query log is huge

1. Check whether logging is intentionally enabled.
2. Review threshold/configuration.
3. Analyze queries instead of just deleting the log.
4. Ensure sufficient disk space.
5. Archive/rotate according to policy.
6. Disable/reduce logging only when appropriate.

---

## 73. Need to investigate an outage from yesterday

1. Establish exact outage time.
2. Correlate MariaDB logs with OS/application logs.
3. Review monitoring graphs.
4. Check configuration/change history.
5. Examine backup/replication events.
6. Build a timeline before concluding root cause.

---

# Performance

## 74. Database is fast at night but slow during business hours

1. Compare workload volume.
2. Check CPU, memory, and I/O during peak hours.
3. Identify peak SQL.
4. Check locks and connections.
5. Compare query latency over time.
6. Address the actual peak-hour bottleneck.

---

## 75. Reporting query hurts production

1. Identify resource consumption.
2. Check execution plan.
3. Consider a replica/reporting database.
4. Optimize the query.
5. Schedule expensive reporting where appropriate.
6. Ensure reporting doesn't compromise transactional workload.

---

## 76. Index improved SELECT but INSERT became slower

1. Review added index count.
2. Measure write performance.
3. Determine whether the index is necessary.
4. Check index usage.
5. Remove redundant indexes if justified.
6. Balance read and write performance.

---

## 77. Temporary tables are consuming lots of resources

1. Identify queries causing large temporary operations.
2. Use `EXPLAIN`.
3. Check sorting/grouping patterns.
4. Review relevant MariaDB configuration.
5. Optimize the query where possible.
6. Monitor disk usage for temporary files.

---

## 78. Large JOIN is very slow

1. Run `EXPLAIN`.
2. Check indexes on join/filter columns.
3. Verify join conditions are correct.
4. Check estimated rows.
5. Reduce unnecessary rows/columns before the join where possible.
6. Test with production-like data volume.

---

## 79. Query works quickly on small data but slowly in production

1. Compare table sizes.
2. Compare data distribution.
3. Compare indexes.
4. Run `EXPLAIN` on production-like data.
5. Test realistic concurrency.
6. Design performance tests around production scale.

---

# Schema & Data

## 80. Table is corrupted/suspect

1. Stop making assumptions about the cause.
2. Review MariaDB error logs.
3. Identify the storage engine.
4. Take/secure an appropriate backup before repair actions.
5. Follow MariaDB-supported recovery procedures.
6. Validate recovered data thoroughly.

---

## 81. Someone wants to change a production table

1. Understand the requested change.
2. Estimate impact and duration.
3. Determine locking/availability implications.
4. Test the change on representative data.
5. Prepare backup/rollback strategy.
6. Execute during an approved change window.

---

## 82. Large index creation is requested

1. Determine table size.
2. Estimate resource usage.
3. Understand locking/availability implications.
4. Test on a similar environment.
5. Schedule appropriately.
6. Monitor CPU, I/O, and application performance during the operation.

---

## 83. Character encoding problem

1. Identify database/table/column character sets.
2. Check connection character-set configuration.
3. Compare source and target settings.
4. Identify whether data is already corrupted.
5. Fix configuration/migration process.
6. Validate multilingual data after correction.

---

## 84. Collation behavior changed after migration

1. Compare source and target collations.
2. Check database/table/column definitions.
3. Test sorting and comparison behavior.
4. Review application expectations.
5. Correct definitions where required.
6. Validate before production cutover.

---

# Production Operations

## 85. DBA receives a change request

1. Understand exactly what will change.
2. Determine business and technical impact.
3. Assess risk.
4. Create implementation steps.
5. Define rollback and validation.
6. Obtain required approval.

---

## 86. Emergency production change

1. Confirm the emergency is real.
2. Stabilize service first.
3. Record what is being changed and why.
4. Use the safest reversible action available.
5. Validate immediately.
6. Complete documentation/RCA afterward.

---

## 87. DBA accidentally changed the wrong server

1. Stop further changes.
2. Determine exactly what changed.
3. Assess data/service impact.
4. Restore the intended configuration/state where safe.
5. Validate the correct environment.
6. Improve server-identification/change-control procedures.

---

## 88. Two DBAs are making simultaneous changes

1. Stop conflicting work.
2. Identify the active change owners.
3. Establish one source of truth.
4. Determine whether changes conflict.
5. Continue under coordinated change control.
6. Document the final state.

---

## 89. Production migration has exceeded the window

1. Determine current migration state.
2. Identify whether the application can remain online safely.
3. Assess rollback vs continuation.
4. Communicate status clearly.
5. Protect data consistency.
6. Continue only according to the approved decision criteria.

---

## 90. Application deployment requires a schema change

1. Review application/database compatibility.
2. Test the schema change.
3. Consider backward compatibility.
4. Coordinate application and DBA deployment order.
5. Have rollback/recovery procedures.
6. Monitor after deployment.

---

# Disaster Recovery

## 91. Entire primary server is lost

1. Confirm infrastructure failure.
2. Activate the DR/failover procedure.
3. Identify the latest consistent recovery source.
4. Restore/promote according to architecture.
5. Validate data and application.
6. Document RPO/RTO achieved.

---

## 92. Data center is unavailable

1. Activate the DR plan.
2. Confirm DR MariaDB infrastructure.
3. Verify replication/backup freshness.
4. Promote/restore according to procedure.
5. Redirect applications.
6. Monitor closely and document recovery.

---

## 93. Ransomware affects the database environment

1. Isolate affected systems according to security procedures.
2. Do not destroy forensic evidence.
3. Protect known-good backups.
4. Determine the compromise scope.
5. Recover using trusted infrastructure/backups.
6. Coordinate DBA, security, infrastructure, and management teams.

---

## 94. Backup and replica are both unavailable

1. Determine whether any additional backup copies exist.
2. Search approved backup/DR locations.
3. Establish the last known recoverable state.
4. Preserve any remaining binary logs/data.
5. Recover into isolated infrastructure.
6. Perform a full incident review afterward.

---

# Advanced DBA Thinking

## 95. Management asks: "Can we guarantee zero downtime?"

1. Don't promise without architecture and testing.
2. Define what "zero downtime" means.
3. Identify application/database dependencies.
4. Design HA, replication, and connection failover appropriately.
5. Test the entire process.
6. Communicate realistic RTO/RPO expectations.

---

## 96. Management asks: "Why do we need backups if we have replication?"

1. Replication copies changes.
2. Accidental DELETE/UPDATE can replicate too.
3. Replicas may contain the same logical mistake.
4. Backups provide an independent recovery mechanism.
5. Point-in-time recovery can address logical mistakes.
6. Therefore replication and backup solve different problems.

---

## 97. Management asks: "Why is the database slow? The server has 64 CPUs."

1. More CPU does not automatically fix inefficient SQL.
2. Queries may be I/O-bound.
3. Queries may be lock-bound.
4. Poor indexing can dominate performance.
5. Application concurrency can create contention.
6. Measure the bottleneck before adding hardware.

---

## 98. DBA is asked to fix production immediately

1. First establish the actual symptom.
2. Protect data and availability.
3. Gather minimum critical evidence.
4. Apply the safest evidence-based fix.
5. Validate service recovery.
6. Perform RCA afterward.

---

## 99. What separates a junior DBA from a senior DBA?

1. Junior DBAs often focus on commands.
2. Senior DBAs focus on impact, risk, and root cause.
3. Senior DBAs understand application dependencies.
4. Senior DBAs plan rollback before changes.
5. Senior DBAs automate repetitive operational work.
6. Senior DBAs think about HA, DR, security, capacity, and business requirements.

---

# 100. Final Scenario — Production MariaDB Completely Down

**Q: You are the on-call DBA. Production MariaDB is unavailable. What do you do?**

### 1. Confirm the incident

```text id="t1g8r3"
Application failure
       ↓
Database connectivity test
       ↓
MariaDB service/status
       ↓
OS health
```

Don't assume the database itself is the cause.

### 2. Determine the failure

Check:

```text id="3t0qhz"
MariaDB service
Error log
Disk space
Memory
CPU
Network
Recent changes
```

### 3. Protect the data

Don't randomly:

```text
Delete files
Initialize a new data directory
Reinstall MariaDB
Overwrite the server
```

Those actions can make recovery worse.

### 4. Determine recovery path

Depending on architecture:

```text id="8p2kz8"
Restart safely
       OR
Recover service
       OR
Fail over to replica
       OR
Restore backup
       OR
Activate DR
```

### 5. Validate

After recovery:

```text id="4f0r8f"
MariaDB
   ↓
Replication
   ↓
Users
   ↓
Critical queries
   ↓
Application
   ↓
Monitoring
```

### 6. Complete the incident

After service is stable:

```text id="h0xj7k"
Timeline
Root cause
RPO/RTO
What worked
What failed
Corrective actions
Preventive actions
Documentation
```

---

# 🏆 100-Scenario DBA Cheat Sheet

When facing **ANY MariaDB production problem**, remember:

```text id="8c8y9n"
             INCIDENT
                 │
                 ▼
          1. CONFIRM
                 │
                 ▼
          2. SCOPE IMPACT
                 │
                 ▼
          3. PROTECT DATA
                 │
                 ▼
          4. COLLECT EVIDENCE
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
       OS      MariaDB    SQL
        │        │        │
       CPU     Sessions  Queries
       RAM     Locks     EXPLAIN
       Disk    Errors    Indexes
        │        │        │
        └────────┼────────┘
                 ▼
            ROOT CAUSE
                 │
                 ▼
          5. CONTROLLED FIX
                 │
                 ▼
          6. VALIDATE
                 │
                 ▼
          7. MONITOR
                 │
                 ▼
          8. DOCUMENT
                 │
                 ▼
             RCA / CAPA
```

## The 10 things a MariaDB DBA should always know

```text
1. Where is my database?
2. Is MariaDB healthy?
3. Is the OS healthy?
4. Are users able to connect?
5. Are queries healthy?
6. Are locks/deadlocks occurring?
7. Is replication healthy?
8. Did the latest backup succeed?
9. Can I recover if this server disappears?
10. What is my rollback/recovery plan?
```

**If you can answer those 10 questions confidently in production, you're thinking like a MariaDB DBA—not just someone who knows MariaDB commands.**

# MariaDB DBA — Another 100 Real-Time Scenarios & FAQs

## Scenarios 101–200

These are deliberately **different from the first 100**, with more real-world DBA situations involving **SQL, InnoDB, security, monitoring, replication, Linux/Windows, upgrades, automation, capacity, incidents, and interviews**.

---

## 101. MariaDB starts but immediately stops

1. Check `systemctl status mariadb` or the Windows service.
2. Read the MariaDB error log for the exact failure.
3. Check whether the configured data directory exists and is accessible.
4. Check disk space and filesystem permissions.
5. Check whether another MariaDB process is already using required resources.
6. Fix the root cause and restart; don't repeatedly restart blindly.

---

## 102. MariaDB process is running but port isn't listening

1. Check the configured `port`.
2. Verify the `bind-address`.
3. Check with `ss -lntp` or an equivalent Windows network command.
4. Confirm the MariaDB process owns the expected port.
5. Check firewall/network configuration.
6. Test locally before testing remotely.

---

## 103. Remote connection works from one server but not another

1. Compare source IP addresses.
2. Check the MariaDB account's `Host` restriction.
3. Check network/firewall rules.
4. Test TCP connectivity to the MariaDB port.
5. Review MariaDB authentication errors.
6. Correct the specific network or account restriction.

---

## 104. Local root login works but remote root login fails

1. This may be intentional security behavior.
2. Check root-related `User`/`Host` entries.
3. Avoid enabling unrestricted remote root access.
4. Create a dedicated administrative account if remote administration is required.
5. Restrict its source hosts.
6. Use secure authentication/TLS where appropriate.

---

## 105. MariaDB client says "Unknown database"

1. Verify the database name.
2. Run `SHOW DATABASES;`.
3. Check spelling/case considerations.
4. Confirm the account has access.
5. Check whether the application points to the correct server.
6. Create/restore the database only if it is genuinely missing.

---

## 106. Table exists but application says it doesn't

1. Check the database/schema selected by the application.
2. Verify the exact table name.
3. Check case sensitivity differences between environments.
4. Confirm the application user has access.
5. Compare source and target schemas.
6. Test the query using the same application credentials.

---

## 107. Application says "Column doesn't exist"

1. Capture the exact SQL and column name.
2. Check the table definition.
3. Compare application version with database schema version.
4. Check whether a deployment missed a migration.
5. Verify the application is connected to the expected database.
6. Apply the missing schema change only after testing.

---

## 108. Application deployment broke because of a schema mismatch

1. Stop or control the deployment if necessary.
2. Identify application and database versions.
3. Determine whether the schema migration ran.
4. Check migration history.
5. Restore compatibility using the approved deployment strategy.
6. Improve deployment sequencing to prevent recurrence.

---

## 109. Database size suddenly increased

1. Identify which databases/tables grew.
2. Compare current size with historical trends.
3. Check large transactions and bulk loads.
4. Check binary logs and temporary files.
5. Determine whether growth is legitimate.
6. Plan capacity before the filesystem becomes critical.

---

## 110. One table suddenly became huge

1. Identify the table's growth rate.
2. Check recent application releases.
3. Check whether a batch process inserted unexpected data.
4. Inspect row counts and indexes.
5. Determine whether data cleanup/retention exists.
6. Fix the source of uncontrolled growth.

---

# SQL Troubleshooting

## 111. `EXPLAIN` shows an unexpected index

1. Review all available indexes.
2. Examine the query predicates.
3. Check estimated rows.
4. Determine why the optimizer selected that index.
5. Test alternative plans safely.
6. Don't force an index unless testing proves it is beneficial.

---

## 112. Query optimizer chooses no index

1. Run `EXPLAIN`.
2. Check table size and estimated selectivity.
3. Check whether an index actually exists.
4. Examine expressions/functions applied to indexed columns.
5. Check statistics/data distribution.
6. Test an appropriate index against realistic data.

---

## 113. `SELECT *` is causing performance problems

1. Identify the consuming application/report.
2. Determine which columns are actually needed.
3. Replace `SELECT *` with required columns where practical.
4. Reduce network and memory usage.
5. Check whether a covering index could help.
6. Measure the improvement.

---

## 114. Query uses a function on an indexed column

1. Example: applying a function in a predicate can reduce index efficiency.
2. Examine the execution plan.
3. Determine whether the predicate can be rewritten.
4. Consider an appropriate alternative/indexing strategy.
5. Test using realistic data.
6. Validate both performance and correctness.

---

## 115. `ORDER BY` query is slow

1. Run `EXPLAIN`.
2. Check whether sorting is required.
3. Review filtering and ordering columns.
4. Consider an index aligned with the query pattern.
5. Check temporary/disk-based sorting.
6. Test under production-like workload.

---

## 116. `GROUP BY` query consumes huge resources

1. Identify the exact query.
2. Run `EXPLAIN`.
3. Check rows processed.
4. Review filtering before aggregation.
5. Consider appropriate indexing.
6. Reduce unnecessary data processed by the aggregation.

---

## 117. Query returns duplicate rows

1. Verify whether duplicates are expected.
2. Review JOIN conditions.
3. Check one-to-many relationships.
4. Don't solve it blindly with `DISTINCT`.
5. Correct the underlying query if the JOIN is wrong.
6. Test against representative data.

---

## 118. Query suddenly returns zero rows

1. Verify application parameters.
2. Run the query manually.
3. Check database/schema selection.
4. Check recent data changes.
5. Review transaction isolation/visibility if relevant.
6. Compare behavior with the last known-good version.

---

## 119. Query results differ between environments

1. Compare MariaDB versions.
2. Compare SQL modes.
3. Compare character sets/collations.
4. Compare schema/index definitions.
5. Compare data distribution.
6. Reproduce using the same SQL and representative data.

---

## 120. SQL mode differs between servers

1. Run `SELECT @@sql_mode;`.
2. Compare server and session settings.
3. Identify whether application behavior depends on them.
4. Standardize configuration where appropriate.
5. Test application compatibility.
6. Document environment-specific differences.

---

# InnoDB / Storage Engine

## 121. Why is InnoDB generally used for transactional workloads?

1. It supports transactions.
2. It provides crash recovery mechanisms.
3. It supports row-level locking.
4. It supports foreign keys.
5. It is designed for concurrent workloads.
6. Always verify engine-specific behavior for your MariaDB version.

---

## 122. Transaction rollback is taking a long time

1. Large transactions can take significant time to undo.
2. Check the affected session/transaction.
3. Monitor CPU, I/O, and locks.
4. Avoid repeatedly killing/restarting the server.
5. Allow recovery/rollback to complete when safe.
6. Prevent recurrence with smaller transactions.

---

## 123. Large transaction is hurting production

1. Identify transaction size and duration.
2. Check locks and resource consumption.
3. Determine why the application performs such a large transaction.
4. Consider batching work.
5. Test smaller transaction boundaries.
6. Monitor impact after changes.

---

## 124. Transaction isolation is causing unexpected behavior

1. Identify current isolation settings.
2. Understand the application's consistency requirements.
3. Reproduce the behavior with concurrent sessions.
4. Check locking and visibility behavior.
5. Change isolation only after understanding the business impact.
6. Test thoroughly before production deployment.

---

## 125. Foreign-key operation is failing

1. Check the exact error.
2. Verify referenced table/column definitions.
3. Check data type compatibility.
4. Look for orphan/inconsistent data.
5. Check constraints and indexes.
6. Correct the data/schema issue rather than disabling constraints blindly.

---

## 126. DELETE fails because of foreign keys

1. Identify the referencing tables.
2. Determine the intended business behavior.
3. Check whether child rows must be removed first.
4. Review `ON DELETE` behavior.
5. Use controlled transactions.
6. Never disable constraints simply to force deletion without impact analysis.

---

## 127. INSERT fails with duplicate key

1. Capture the duplicate key.
2. Identify the unique/primary index.
3. Determine whether the duplicate is expected.
4. Check application retry behavior.
5. Decide whether INSERT, UPDATE, or an appropriate upsert pattern is intended.
6. Fix the business/application logic.

---

## 128. Auto-increment behavior is unexpected

1. Check the column definition.
2. Check current values and inserts.
3. Review failed/rolled-back inserts.
4. Understand that gaps can occur.
5. Don't assume IDs must be gap-free.
6. If business requires sequential numbers, use a separate controlled mechanism.

---

## 129. Table has excessive indexes

1. List indexes with `SHOW INDEX`.
2. Identify redundant or overlapping indexes.
3. Review actual query workload.
4. Measure write overhead.
5. Remove indexes only after testing.
6. Document why remaining indexes exist.

---

## 130. Table has no primary key

1. Determine whether the table is intentionally designed that way.
2. Understand consequences for data integrity and operations.
3. Identify a suitable candidate key.
4. Test adding a primary key carefully.
5. Assess replication/maintenance implications.
6. Implement only after application impact analysis.

---

# Replication Advanced

## 131. Replica authentication fails

1. Read the replication error.
2. Verify replication credentials.
3. Check source/replica connectivity.
4. Check account host restrictions.
5. Rotate credentials carefully if required.
6. Restart/repair replication using the documented process.

---

## 132. Replica cannot reach primary

1. Test network connectivity.
2. Check DNS/hostname resolution.
3. Check firewall rules.
4. Verify primary is listening.
5. Check replication connection settings.
6. Resolve network connectivity before changing replication state.

---

## 133. Replica is connected but not applying transactions

1. Check replication status.
2. Determine whether the SQL/apply process is stopped.
3. Inspect the error.
4. Check locks and resource pressure.
5. Identify the transaction causing the problem.
6. Correct the error and validate that replication resumes.

---

## 134. Replication lag occurs only during batch jobs

1. Identify the batch workload.
2. Check transaction sizes.
3. Monitor replica I/O and CPU.
4. Determine whether batch transactions can be broken into smaller units.
5. Schedule or throttle workload where appropriate.
6. Establish acceptable lag thresholds.

---

## 135. Replica lag occurs every evening

1. Compare the timing with scheduled jobs.
2. Identify backup/reporting/ETL workloads.
3. Check resource utilization.
4. Identify recurring SQL patterns.
5. Separate workloads if necessary.
6. Fix the recurring bottleneck instead of manually restarting replication.

---

## 136. Replica has more disk usage than primary

1. Compare database files and logs.
2. Check whether reporting creates temporary files.
3. Check binary-log/relay-log retention.
4. Check backup files accidentally stored locally.
5. Compare filesystem layout.
6. Clean up only according to documented retention rules.

---

## 137. Need to rebuild a replica

1. Confirm rebuilding is actually necessary.
2. Take a consistent source backup or use the approved cloning process.
3. Provision the replica correctly.
4. Restore/synchronize data.
5. Configure replication.
6. Verify transaction synchronization before returning it to service.

---

## 138. Replica was offline for several days

1. Determine whether required binary logs still exist.
2. Check how far behind it is.
3. Verify whether incremental recovery is possible.
4. If required, rebuild from a fresh backup.
5. Re-establish replication carefully.
6. Validate data consistency.

---

## 139. Can a replica be promoted immediately?

1. First verify it is healthy.
2. Check replication state.
3. Determine lag/data-loss risk.
4. Fence the failed primary where required.
5. Follow the promotion runbook.
6. Validate application behavior after promotion.

---

## 140. Replication topology has multiple replicas

1. Document which node replicates from which source.
2. Monitor every replication link.
3. Understand failover dependencies.
4. Avoid creating uncontrolled replication chains.
5. Test promotion paths.
6. Keep topology documentation current.

---

# Linux Administration

## 141. MariaDB cannot write to its data directory

1. Check filesystem permissions.
2. Check ownership of the data directory.
3. Check whether the filesystem is mounted read-only.
4. Check SELinux/AppArmor where applicable.
5. Review MariaDB error logs.
6. Correct security/context settings rather than using unsafe permission changes.

---

## 142. Filesystem became read-only

1. Check `mount` status.
2. Review kernel/system logs.
3. Check underlying storage health.
4. Determine whether filesystem corruption occurred.
5. Protect database data before attempting repairs.
6. Follow OS/storage recovery procedures.

---

## 143. Linux server has high load average

1. Determine whether CPU or I/O is causing the load.
2. Use tools such as `top`, `vmstat`, and `iostat` where available.
3. Identify MariaDB processes/queries.
4. Check disk latency.
5. Check concurrent workload.
6. Fix the actual resource bottleneck.

---

## 144. MariaDB service fails after reboot

1. Check service enablement.
2. Review boot-time logs.
3. Check filesystem/data directory availability.
4. Verify configuration.
5. Check dependency/mount ordering.
6. Correct the boot configuration and retest.

---

## 145. Log rotation deleted a file MariaDB still expects

1. Check MariaDB logging configuration.
2. Review rotation configuration.
3. Determine whether the process still has an open file handle.
4. Correct the rotation method.
5. Restart/reopen logs only when appropriate.
6. Verify logging afterward.

---

## 146. NTP/time synchronization is wrong

1. Check server time.
2. Check time synchronization service.
3. Compare DB/application server clocks.
4. Determine whether timestamps affect replication, monitoring, or application behavior.
5. Correct time synchronization.
6. Review events that occurred during the time discrepancy.

---

## 147. Server hostname changed

1. Check MariaDB configuration and scripts.
2. Check replication configuration.
3. Check monitoring and backup jobs.
4. Check application connection settings.
5. Update DNS/service discovery as needed.
6. Test all dependent services.

---

## 148. Server was cloned from another MariaDB server

1. Do not treat the clone as production-ready automatically.
2. Check server identity/configuration.
3. Review replication settings.
4. Check unique IDs/hostnames where relevant.
5. Remove environment-specific secrets/configuration.
6. Validate before connecting it to production topology.

---

## 149. Cron backup job didn't run

1. Check cron configuration.
2. Check execution permissions.
3. Review environment/path differences.
4. Check output/error logs.
5. Run the script manually in a controlled manner.
6. Add monitoring for backup-job failure.

---

## 150. DBA script works manually but fails in cron

1. Cron has a different environment.
2. Use absolute paths.
3. Check PATH and environment variables.
4. Check credentials/secrets access.
5. Capture stdout/stderr.
6. Test under the same execution context.

---

# Windows Administration

## 151. MariaDB Windows service won't start

1. Open Windows service information.
2. Check MariaDB error logs.
3. Check Event Viewer.
4. Verify data directory and permissions.
5. Check port/service conflicts.
6. Correct the cause and restart.

---

## 152. Windows server has high memory usage

1. Identify top memory-consuming processes.
2. Check MariaDB memory settings.
3. Check connection count.
4. Check other services running on the server.
5. Determine whether paging is occurring.
6. Tune the workload/configuration based on evidence.

---

## 153. Windows firewall blocks MariaDB

1. Verify MariaDB is listening.
2. Test local connection.
3. Test remote TCP connectivity.
4. Review inbound firewall rules.
5. Allow only required source networks.
6. Retest application connectivity.

---

## 154. Windows Event Viewer shows MariaDB errors

1. Capture timestamp and event details.
2. Correlate with MariaDB error logs.
3. Determine whether it was a service, OS, disk, or configuration issue.
4. Check recent changes.
5. Fix the underlying problem.
6. Monitor for recurrence.

---

## 155. Windows disk is filling with database logs

1. Identify which logs are growing.
2. Review rotation/retention.
3. Check whether logging level changed.
4. Archive safely where required.
5. Ensure sufficient free space.
6. Fix retention/monitoring configuration.

---

# Security & Auditing

## 156. Former employee still has DB access

1. Identify all accounts associated with the user.
2. Disable/remove unauthorized access promptly.
3. Check shared/service accounts.
4. Review recent database activity.
5. Rotate credentials where necessary.
6. Improve offboarding procedures.

---

## 157. Need to audit privilege changes

1. Establish what audit capability is configured.
2. Review relevant logs/audit records.
3. Identify who made the change and when.
4. Compare before/after grants.
5. Determine whether the change was authorized.
6. Implement stronger change auditing if necessary.

---

## 158. Database server is exposed to the internet

1. Treat this as a security concern.
2. Determine whether public exposure is actually required.
3. Restrict network access with firewall/security controls.
4. Use secure authentication/TLS.
5. Remove unnecessary accounts.
6. Monitor and review exposure continuously.

---

## 159. TLS certificate is expiring

1. Check certificate expiration.
2. Obtain the approved replacement certificate.
3. Test certificate/key compatibility.
4. Schedule controlled deployment.
5. Validate client connectivity.
6. Monitor for certificate-related errors.

---

## 160. Application suddenly rejects TLS connection

1. Check certificate validity.
2. Check hostname/SAN expectations.
3. Compare client and server TLS configuration.
4. Review MariaDB logs.
5. Test using the application's connection method.
6. Correct compatibility/configuration issues.

---

# Capacity Planning

## 161. Database storage grows 20 GB every month

1. Calculate historical growth.
2. Identify which tables/databases drive growth.
3. Determine business reason.
4. Forecast future capacity.
5. Review retention/archive policies.
6. Plan storage expansion before reaching critical levels.

---

## 162. Connection count grows every month

1. Track connection trends.
2. Compare with application traffic.
3. Review pool configuration.
4. Identify leaked/idle connections.
5. Forecast future `max_connections` requirements.
6. Fix inefficient connection management.

---

## 163. CPU usage increases every quarter

1. Compare CPU with transaction/query volume.
2. Identify expensive SQL.
3. Check index effectiveness.
4. Review application growth.
5. Forecast CPU requirements.
6. Optimize before scaling hardware.

---

## 164. Database has reached its expected capacity

1. Review actual resource utilization.
2. Identify the limiting resource.
3. Consider query/workload optimization.
4. Evaluate vertical or horizontal scaling.
5. Review archival/retention options.
6. Create a capacity plan with measurable thresholds.

---

## 165. Business expects 10× traffic next year

1. Establish current baseline performance.
2. Model expected transactions/connections.
3. Load-test realistic workload.
4. Identify scaling bottlenecks.
5. Design capacity/HA architecture.
6. Validate the design before traffic arrives.

---

# Automation

## 166. What should a DBA automate first?

1. Backup verification.
2. Health checks.
3. Disk-space alerts.
4. Replication monitoring.
5. Routine reporting.
6. Repetitive maintenance with safe rollback.

---

## 167. Automated script accidentally ran on production

1. Stop the automation immediately.
2. Determine exactly what it executed.
3. Assess database impact.
4. Restore/recover if required.
5. Add environment safeguards.
6. Require production confirmation for dangerous operations.

---

## 168. How can scripts avoid wrong-server execution?

1. Check hostname before execution.
2. Check environment variables.
3. Require explicit production confirmation.
4. Use separate credentials/permissions.
5. Log every execution.
6. Fail closed if environment validation fails.

---

## 169. Backup script reports success but file is missing

1. Check the script's actual exit code.
2. Verify output location.
3. Check filesystem permissions.
4. Check storage/network destination.
5. Validate backup file existence and integrity.
6. Alert based on successful validation, not merely script completion.

---

## 170. DBA wants automatic restart of MariaDB

1. Understand why the service stops.
2. Configure monitoring appropriately.
3. Automatic restart can restore availability in some failures.
4. It can also hide recurring problems.
5. Ensure logs and alerts still capture the failure.
6. Use restart automation together with root-cause analysis.

---

# Monitoring & Observability

## 171. What is the most important MariaDB alert?

1. There isn't one universal alert.
2. Availability is critical.
3. Replication health is critical for replicated systems.
4. Backup failure is critical for recoverability.
5. Disk exhaustion is highly important.
6. Query/resource saturation should be monitored proactively.

---

## 172. Alert threshold is too sensitive

1. Review historical metrics.
2. Determine normal operating range.
3. Distinguish warning from critical thresholds.
4. Avoid excessive alert noise.
5. Test the new thresholds.
6. Review alert effectiveness periodically.

---

## 173. Alert threshold is too high

1. Look for incidents that occurred before the alert triggered.
2. Compare thresholds with actual failure points.
3. Lower warning/critical levels appropriately.
4. Add trend-based alerts where useful.
5. Test alert delivery.
6. Verify that alerts lead to actionable responses.

---

## 174. No monitoring data during outage

1. Determine whether the monitoring system failed too.
2. Use server/application logs as secondary evidence.
3. Check whether agents stopped.
4. Review monitoring architecture.
5. Add independent health checks where appropriate.
6. Ensure critical systems don't rely on one monitoring path.

---

## 175. Metrics look normal but application is slow

1. Infrastructure metrics may hide query-level problems.
2. Check active SQL and latency.
3. Check locks/waits.
4. Check application connection pools.
5. Check database error rates.
6. Add query/application-level observability.

---

# Maintenance

## 176. Should you restart MariaDB every month?

1. Not simply as routine housekeeping.
2. MariaDB does not generally require arbitrary periodic restarts.
3. Restarts should have a reason.
4. Planned maintenance may require one.
5. Diagnose memory/resource problems instead of masking them with restarts.
6. Document necessary restarts.

---

## 177. DBA wants to reboot production server

1. Determine why reboot is required.
2. Check HA/failover options.
3. Assess application impact.
4. Verify backups and recovery.
5. Schedule an approved maintenance window.
6. Validate everything after reboot.

---

## 178. Configuration change requires restart

1. Confirm whether the parameter really requires restart.
2. Determine whether dynamic change is possible.
3. Assess production impact.
4. Schedule if restart is required.
5. Prepare rollback.
6. Validate configuration after restart.

---

## 179. Temporary tablespace/disk usage remains high

1. Determine whether active workload still requires it.
2. Check long-running transactions.
3. Check temporary-table workload.
4. Review filesystem usage.
5. Follow MariaDB-supported maintenance procedures.
6. Avoid deleting internal database files manually.

---

## 180. Database maintenance takes longer than expected

1. Check current operation and progress.
2. Identify locks/resource contention.
3. Compare with test estimates.
4. Determine whether continuing is safer than stopping.
5. Communicate impact.
6. Improve future maintenance estimates.

---

# Data Integrity

## 181. Row counts differ between primary and replica

1. Don't immediately assume replication is broken.
2. Identify affected tables.
3. Compare transaction/replication state.
4. Determine whether legitimate filtering or timing explains it.
5. Perform controlled consistency checks.
6. Repair/rebuild the replica if genuine divergence exists.

---

## 182. Data appears missing after migration

1. Identify exact records.
2. Compare source and target.
3. Check migration logs.
4. Check transaction boundaries.
5. Determine whether filtering/conversion caused the loss.
6. Restore/re-migrate missing data using a controlled procedure.

---

## 183. Duplicate business records appeared

1. Identify duplicate pattern.
2. Determine application cause.
3. Check unique constraints.
4. Identify whether retries caused duplicate inserts.
5. Correct application/database design.
6. Clean existing duplicates carefully.

---

## 184. Business says "database lost yesterday's data"

1. Establish exactly which data is missing.
2. Identify the time range.
3. Check application logs.
4. Check transactions/audit records where available.
5. Determine whether data was deleted, never inserted, or not visible.
6. Recover only after identifying the actual failure.

---

## 185. Database values are NULL unexpectedly

1. Identify affected table/column.
2. Check column definition.
3. Review application code.
4. Check recent deployments.
5. Determine whether NULL is valid.
6. Correct data and application logic separately.

---

# Migration / Upgrade Advanced

## 186. Upgrade succeeded but queries became slower

1. Compare query execution plans.
2. Compare optimizer/configuration behavior.
3. Check indexes/statistics.
4. Check workload and data changes.
5. Identify affected queries.
6. Optimize based on evidence rather than immediately rolling back.

---

## 187. Upgrade changed authentication behavior

1. Capture the exact authentication error.
2. Review source and target authentication configuration.
3. Check client compatibility.
4. Verify account definitions.
5. Test using the application driver.
6. Apply a supported compatibility solution.

---

## 188. Migration changes case sensitivity behavior

1. Compare Windows/Linux filesystem characteristics.
2. Compare database/table naming.
3. Check application SQL.
4. Identify tables referenced with inconsistent case.
5. Standardize naming where practical.
6. Test migration on the actual target OS.

---

## 189. Migration takes 12 hours

1. Identify whether export, transfer, or import is the bottleneck.
2. Measure data volume and throughput.
3. Check compression/network/storage.
4. Consider an appropriate faster migration strategy.
5. Test with production-sized data.
6. Recalculate the cutover window.

---

## 190. Need near-zero-downtime migration

1. Define acceptable downtime/RPO.
2. Establish source-to-target synchronization.
3. Validate target continuously.
4. Keep application compatibility during transition.
5. Perform a controlled final cutover.
6. Maintain rollback capability until validation completes.

---

# Incident Response

## 191. Database outage started immediately after deployment

1. Correlate timestamps.
2. Identify database/application changes.
3. Check error logs.
4. Determine whether rollback is safer than troubleshooting live.
5. Restore service first.
6. Perform RCA after stabilization.

---

## 192. Database outage started after OS patching

1. Check service status.
2. Review OS and MariaDB logs.
3. Check kernel/storage/network changes.
4. Verify MariaDB binary/configuration compatibility.
5. Roll back only under an approved recovery procedure.
6. Document the dependency for future patch testing.

---

## 193. Database becomes unavailable every Sunday

1. Identify scheduled jobs.
2. Compare outage time with backups/ETL/maintenance.
3. Check resource spikes.
4. Review logs around the exact time.
5. Identify the recurring root cause.
6. Fix or reschedule the conflicting workload.

---

## 194. Production database was accidentally restarted

1. Determine who/what initiated the restart.
2. Verify crash recovery completed.
3. Check application availability.
4. Review replication state.
5. Check logs for any recovery errors.
6. Improve access/change controls.

---

## 195. User reports intermittent database failures

1. Determine exact timestamps.
2. Correlate with application and MariaDB logs.
3. Check network stability.
4. Check connection pool exhaustion.
5. Check resource spikes.
6. Avoid concluding "database issue" without evidence.

---

# Interview-Level Scenarios

## 196. What happens when MariaDB crashes?

1. The OS/process failure stops normal database processing.
2. InnoDB recovery mechanisms work during startup.
3. Recovery replays/undoes required changes according to its recovery design.
4. Recovery duration depends on workload and state.
5. Logs should be reviewed after startup.
6. Application and data integrity must be validated.

---

## 197. What is the difference between a backup and a snapshot?

1. A backup is designed for recovery and retention.
2. A storage snapshot captures storage state at a point in time.
3. Snapshot consistency must be understood.
4. Snapshots may depend heavily on the underlying storage platform.
5. Neither should be trusted without recovery testing.
6. Use the mechanism that meets RPO/RTO requirements.

---

## 198. Why should DBAs document everything?

1. Production environments are complex.
2. Another DBA must be able to operate the system.
3. Documentation reduces recovery time.
4. It preserves institutional knowledge.
5. It supports audits and change management.
6. Good runbooks reduce human error during incidents.

---

## 199. What is the most dangerous DBA habit?

1. Making changes without understanding impact.
2. Running commands copied from the internet without validation.
3. Working directly in production without a rollback plan.
4. Assuming backups work without restore testing.
5. Ignoring monitoring and logs.
6. Treating symptoms instead of finding root causes.

---

# 200. Ultimate Production Scenario

## Q: You are the senior MariaDB DBA. Production is slow, replication is lagging, disk is 90% full, and management wants an immediate fix. What do you do?

### 1. Stabilize

```text
Application
     ↓
MariaDB
     ↓
OS
     ↓
Storage
```

Determine whether the problem is actively worsening.

### 2. Protect data

```text
Don't:
❌ Delete random DB files
❌ Rebuild replica blindly
❌ Kill random sessions
❌ Restart repeatedly
❌ Disable replication without understanding why
```

### 3. Identify the bottleneck

Check:

```text
CPU
Memory
Disk I/O
Disk space
Connections
Locks
Long queries
Replication
Binary logs
```

### 4. Reduce immediate pressure

Depending on evidence:

```text
Stop unnecessary reporting
Throttle batch workload
Terminate confirmed harmful queries
Free safe disk space
Redirect workload
```

### 5. Restore normal operation

```text
Slow queries
     ↓
Optimize/fix

Disk pressure
     ↓
Capacity/retention correction

Replication lag
     ↓
Remove bottleneck

Application pressure
     ↓
Control workload
```

### 6. Finish properly

```text
Recovery
   ↓
Validation
   ↓
Monitoring
   ↓
Root Cause
   ↓
Preventive Action
   ↓
Runbook Update
```

---

# 🎯 Your 200-Scenario MariaDB DBA Map

You now have **200 real-world scenarios** covering:

| Area                      | Scenarios |
| ------------------------- | --------: |
| Installation & startup    |   101–110 |
| SQL/query troubleshooting |   111–120 |
| InnoDB & data integrity   |   121–130 |
| Replication               |   131–140 |
| Linux administration      |   141–150 |
| Windows administration    |   151–155 |
| Security                  |   156–160 |
| Capacity planning         |   161–165 |
| Automation                |   166–170 |
| Monitoring                |   171–175 |
| Maintenance               |   176–180 |
| Data integrity            |   181–185 |
| Migration & upgrades      |   186–190 |
| Incident response         |   191–195 |
| Senior DBA/interview      |   196–200 |

### 🔥 The production DBA formula

**Don't memorize 200 answers. Memorize this pattern:**

> **Confirm → Scope → Protect → Collect Evidence → Diagnose → Fix → Validate → Monitor → Document → Prevent**

That sequence will help you handle a large percentage of real MariaDB production incidents safely.

