# Chapter 10 – Database Monitoring, Performance Tuning, and Troubleshooting

## 1. Concept Overview

A database in production is a living, breathing organism. Over time, as data volumes grow and application code changes, performance inevitably degrades. **Database Monitoring** is the practice of establishing a baseline of normal behavior (Observability) so you can detect anomalies. 

When a database slows down, novice administrators blindly throw more hardware (CPU/RAM) at the problem. A true Database Architect relies on **Performance Tuning** and **Wait Statistics**. 
Whenever a query is executing, it is either actively running on a CPU thread, or it is *waiting* for a resource (waiting for a lock to be released, waiting for data to be read from the physical disk, waiting for memory to be allocated). By asking the database exactly *what* it is waiting for, you can diagnose the root cause of the bottleneck with mathematical precision.

## 2. History

In the 1990s, performance troubleshooting was largely guesswork. DBAs monitored OS-level metrics (Task Manager, `top`) and if CPU was high, they bought a bigger server. In the early 2000s, Microsoft revolutionized troubleshooting by introducing **Wait Statistics** and **Dynamic Management Views (DMVs)** in SQL Server 2005. Instead of guessing, DBAs could query the engine to see exactly why it was slow. The rest of the industry, including PostgreSQL, eventually adopted this wait-state methodology as the gold standard for performance engineering.

## 3. Real-world analogy

Imagine a massive **Highway Traffic Jam**.
*   **The OS-Level approach:** You look from a helicopter and see the highway is full (100% CPU utilization). Your solution is to spend $50 million to add two more lanes. But the traffic jam remains.
*   **The Wait-Statistics approach:** You walk down to the cars and ask the drivers *why* they are stopped. They say, "We are waiting for the Toll Booth operator." (Wait Type: I/O Bottleneck). You realize adding lanes (CPU) is useless; you need to add more toll booths (faster Disks / better Indexes).

## 4. Business problem solved

*   **SLA Violations:** Troubleshooting slow databases prevents application timeouts, ensuring the business meets its Service Level Agreements (e.g., all web pages load in under 2 seconds).
*   **Cloud Costs:** Rather than upgrading to a $5,000/month AWS RDS instance to solve a CPU spike, tuning a single missing index can allow the application to run smoothly on a $500/month instance.
*   **Outage Resolution:** When the database completely locks up, knowing exactly which system views to query allows a DBA to resolve the crisis in minutes rather than hours.

---

## 5. Microsoft SQL Server explanation

SQL Server provides the most robust diagnostic toolset in the relational database industry. It exposes its internal memory structures through **Dynamic Management Views (DMVs)**. These are system views that begin with `sys.dm_`.

Key Troubleshooting Tools:
1.  **`sys.dm_os_wait_stats`:** Aggregated historical wait statistics since the server was last restarted.
2.  **`sys.dm_exec_requests`:** Shows exactly what queries are running *right now*, what wait type they are currently experiencing, and their exact SQL text.
3.  **Extended Events (XEvents):** A lightweight tracing system to capture specific events (like queries taking longer than 5 seconds) asynchronously to a file.
4.  **Query Store:** The ultimate historical tuning tool, tracking execution plan regressions over time.

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE master;
GO

-- 1. What is currently running right now? (The Panic Button Query)
SELECT 
    r.session_id,
    r.status,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    t.text AS sql_command,
    p.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
WHERE r.session_id > 50; -- Ignore system background processes
GO

-- 2. What is my server historically waiting on?
SELECT TOP 10
    wait_type,
    wait_time_ms / 1000.0 AS WaitS,
    (wait_time_ms - signal_wait_time_ms) / 1000.0 AS ResourceS,
    signal_wait_time_ms / 1000.0 AS SignalS,
    waiting_tasks_count AS WaitCount
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK', 'BROKER_TASK_STOP', 'DIRTY_PAGE_POLL') -- Filter benign waits
ORDER BY wait_time_ms DESC;
GO
```

## 7. SQL Server internals

**Understanding SQL Server Wait Types:**
*   `PAGEIOLATCH_SH`: (Shared Page I/O Latch). A query is waiting for an 8KB data page to be read from the physical disk into RAM. High waits here mean your disks are slow, or (more likely) you are doing massive Table Scans because you are missing an index.
*   `CXPACKET`: The query is running in parallel across multiple CPU cores. Some cores finished faster than others and are waiting for the slow cores to catch up. Often fixed by tuning the "Cost Threshold for Parallelism" setting.
*   `LCK_M_U` or `LCK_M_X`: (Lock Wait). The query is physically blocked by another transaction holding a lock on the data.

## 8. SQL Server execution

When the database freezes in production:
1. Run the `sys.dm_exec_requests` query.
2. Look at the `blocking_session_id` column. If Session 60 is blocked by 55, and 55 is blocked by 52... Session 52 is the **Head Blocker**.
3. Use `DBCC INPUTBUFFER(52)` to see what query Session 52 is running (or failed to commit).
4. If it's an orphaned transaction, the DBA uses the `KILL 52` command. The transaction rolls back, locks release, and the blocking chain clears instantly.

## 9. SQL Server enterprise examples

*   **Memory Pressure (Page Life Expectancy):** Enterprise DBAs obsessively monitor a Performance Counter called **Page Life Expectancy (PLE)**. This represents how long (in seconds) an 8KB data page stays in the RAM Buffer Pool before being flushed out to make room for new data. If PLE suddenly drops from 5,000 seconds to 10 seconds, the server is under severe memory pressure, usually caused by a bad query pulling millions of rows into memory.

## 10. SQL Server performance considerations

*   **Index Maintenance:** Over time, B-Tree indexes become fragmented (due to page splits). Enterprise environments schedule a weekly job (often using Ola Hallengren's famous maintenance scripts) to intelligently `REORGANIZE` indexes with > 5% fragmentation, and `REBUILD` indexes with > 30% fragmentation.

## 11. SQL Server security considerations

*   Querying DMVs requires the `VIEW SERVER STATE` permission. This should be restricted to DBAs and senior engineers, as DMVs can inadvertently expose PII data that is currently sitting in memory buffers or execution plans.

## 12. SQL Server common mistakes

*   **Throwing RAM at a PLE problem:** If Page Life Expectancy is low, junior admins double the server RAM. 90% of the time, the problem isn't a lack of RAM; the problem is a missing index causing a query to scan 50GB of data off the disk into RAM, pushing all the good data out. Fix the index, and the existing RAM is plenty.
*   **Ignoring the Transaction Log:** Leaving databases in the `FULL` recovery model without running Transaction Log backups. The log file grows to consume the entire C: drive, crashing the entire Windows Server.

## 13. SQL Server best practices

*   Change the default "Cost Threshold for Parallelism" from 5 to 50. The default of 5 was set in the 1990s. Today, it causes SQL Server to use parallel CPU threads for tiny, trivial queries, wasting CPU context-switching overhead.
*   Baseline your wait statistics. You cannot know if a `PAGEIOLATCH` wait time of 500ms is "bad" unless you know that your normal baseline is 5ms.

---

## 14. PostgreSQL explanation

PostgreSQL tuning is a blend of internal metrics and heavy reliance on the host OS (Linux). Postgres does not have hundreds of DMVs like SQL Server; instead, it relies on several crucial internal statistical views (starting with `pg_stat_`) and extensions.

The most critical extension in the Postgres ecosystem is **`pg_stat_statements`**. It must be loaded in `postgresql.conf` (`shared_preload_libraries = 'pg_stat_statements'`). It acts as a continuous flight recorder, aggregating the execution count, total time, and disk I/O for every normalized query executed on the server.

## 15. PostgreSQL syntax

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. What is currently running right now? (The Panic Button Query)
SELECT 
    pid, 
    usename, 
    state, 
    wait_event_type, 
    wait_event, 
    query_start, 
    query 
FROM pg_stat_activity 
WHERE state != 'idle' AND pid != pg_backend_pid();

-- 2. Find the historically slowest queries (Requires pg_stat_statements)
SELECT 
    calls,
    total_exec_time / 1000.0 / 60.0 AS total_minutes,
    min_exec_time,
    max_exec_time,
    mean_exec_time,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;

-- 3. Check for severe index bloat / dead tuples
SELECT 
    relname AS table_name,
    n_live_tup,
    n_dead_tup,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

## 16. PostgreSQL internals

Postgres introduced comprehensive Wait Events in version 9.6. When querying `pg_stat_activity`, look at the `wait_event_type` and `wait_event`:
*   **Type: `IO`, Event: `DataFileRead`:** The query is waiting for the OS to fetch an 8KB block from the physical disk. (Equivalent to SQL Server `PAGEIOLATCH`).
*   **Type: `Lock`, Event: `transactionid`:** The query is attempting to update a row that is currently locked by an uncommitted transaction.
*   **Type: `Client`, Event: `ClientRead`:** The database is waiting for the application to send the next command. If a query is stuck here, the problem is the application, not the database.

## 17. PostgreSQL execution

**Diagnosing Postgres Blocking:**
Because Postgres uses MVCC, readers never block writers. If you see severe blocking, it is almost always two transactions attempting to `UPDATE` the same row, or an unindexed Foreign Key triggering a lock escalation during a `DELETE`.
You can query the `pg_locks` view, joined with `pg_stat_activity`, to find exactly which `pid` (Process ID) is holding the lock, and use `pg_terminate_backend(pid)` to kill the offending connection.

## 18. PostgreSQL enterprise examples

*   **Auto_explain:** In enterprise environments where developers complain about intermittent timeouts, DBAs load the `auto_explain` module. It automatically logs the Execution Plan of any query that takes longer than a specified threshold (e.g., 2 seconds) directly to the Postgres text log, allowing post-mortem analysis without having to guess what the parameters were.

## 19. PostgreSQL performance considerations

*   **Cache Hit Ratio:** Postgres relies heavily on the Linux OS Page Cache. DBAs monitor the "Cache Hit Ratio" (Buffer hits / Total reads). A healthy OLTP system should have a Cache Hit Ratio > 99%. If it drops to 80%, the database is heavily reliant on physical disk I/O, indicating a need for more RAM or better indexing.
*   **Autovacuum Tuning:** The #1 performance killer in Postgres is table bloat. If `pg_stat_user_tables` shows millions of `n_dead_tup` (dead tuples), Autovacuum is falling behind. You must tune `autovacuum_vacuum_cost_limit` (increase it) and `autovacuum_naptime` (decrease it) to make Autovacuum run more aggressively.

## 20. PostgreSQL security considerations

*   By default, ordinary users cannot see the SQL text of queries executed by other users in `pg_stat_activity`. They only see `<insufficient privilege>`. Only a superuser or a role granted `pg_read_all_stats` can monitor the entire instance.

## 21. PostgreSQL common mistakes

*   **Setting `shared_buffers` too high:** Because Postgres uses "Double Buffering", if you give 90% of the server's RAM to `shared_buffers`, you starve the Linux OS Page Cache. The standard rule is to set `shared_buffers` to exactly 25% of total system RAM, leaving the rest for the OS.
*   **Ignoring Connection Limits:** Setting `max_connections = 5000` in `postgresql.conf` without using PgBouncer. Postgres will fork 5,000 OS processes, exhaust the CPU context-switching limits, trigger the OOM (Out Of Memory) Killer, and crash the server.

## 22. PostgreSQL best practices

*   Install and monitor `pg_stat_statements` from Day 1. It is the single most important diagnostic tool in the Postgres ecosystem.
*   Routinely `REINDEX` heavily updated tables (using `REINDEX CONCURRENTLY` in modern Postgres to avoid locking) to clear out B-Tree bloat caused by MVCC updates.

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Current Activity** | `sys.dm_exec_requests` | `pg_stat_activity` | Both provide real-time visibility into running queries and their exact wait states. |
| **Historical Tuning** | Query Store (Native) | `pg_stat_statements` (Ext) | Query Store retains full execution plans. `pg_stat_statements` only retains aggregate stats. |
| **Memory Monitoring**| Page Life Expectancy (PLE) | Cache Hit Ratio | SQL Server manages its own memory completely. Postgres relies heavily on the host Linux OS. |
| **Index Fragmentation**| Internal Page Splits | MVCC Dead Tuple Bloat | Both require routine maintenance (`ALTER INDEX REORGANIZE` vs `REINDEX CONCURRENTLY`). |

## 24. Architect recommendations

**The 4 Golden Signals of Database Monitoring**
Whether using Datadog, Prometheus, or native tools, an Architect must build dashboards monitoring these four signals:
1.  **Latency:** The time it takes to service a request (e.g., 99th percentile query time is < 50ms).
2.  **Traffic:** The amount of demand placed on the system (e.g., 5,000 Transactions Per Second).
3.  **Errors:** The rate of requests that fail (e.g., Deadlocks, Constraint Violations).
4.  **Saturation:** How "full" the system is (e.g., CPU %, Disk Queue Length, Memory usage).

## 25. DBA recommendations

*   **Don't panic at 100% CPU.** If the CPU is 100%, but the Latency is low and the application is snappy, congratulations—you are fully utilizing the hardware you paid for. CPU is only a bottleneck if it is accompanied by high Latency (meaning queries are waiting in the CPU runnable queue).
*   **Do panic at 100% Disk I/O.** If disk latency spikes from 2ms to 50ms, the entire database architecture will grind to a halt immediately. I/O is the ultimate bottleneck.

## 26. Developer recommendations

*   **Use `EXPLAIN` in CI/CD:** If a developer submits a PR that introduces a query doing a massive Sequential Scan, the CI pipeline should flag it before it ever reaches production.
*   **Name your connections:** In the connection string, set the `ApplicationName` parameter (e.g., "BillingService"). This string shows up directly in `pg_stat_activity` and `sys.dm_exec_sessions`. When the DBA is troubleshooting a locked table at 2 AM, they instantly know which microservice is responsible.

## 27. Production case study

**The NextEvent "Frozen" Database**

*Scenario:* On a Friday night, the NextEvent application completely froze. The web servers threw 504 Gateway Timeouts. The Cloud dashboard showed CPU utilization dropped to 5%, but active database connections skyrocketed to 5,000/5,000. 

*Diagnosis:* 
1. The DBA ran the Panic Button query (`pg_stat_activity`).
2. They saw 4,999 connections sitting in the state: `wait_event_type = Lock`.
3. They traced the `blocking_session_id` up the tree to a single connection (PID 1050). PID 1050 was running `DELETE FROM core.organizations WHERE org_id = 12;`.
4. Why did a simple delete lock the entire server? The DBA looked at the schema. `core.events` had a Foreign Key pointing to `organizations.org_id` with `ON DELETE CASCADE`. 
5. Crucially, the developer *forgot to put an index* on `events.org_id`.

*RCA:* When the database tried to cascade the delete, it had to verify if any events belonged to `org_id = 12`. Because there was no index, it had to execute a full Table Scan on the massive `events` table. To protect data integrity during a full table scan for a cascading delete, the database escalated to an Exclusive Table Lock on `events`. Every other user on the website trying to buy a ticket (which inserts into `tickets` and checks the `events` table) was instantly blocked. 

*Architectural Fix:* The DBA killed PID 1050, instantly restoring the website. They then executed `CREATE INDEX idx_events_org_id ON core.events(org_id) CONCURRENTLY`. The cascading delete now takes 1 millisecond and requires zero table locks.

## 28. ASCII diagrams wherever helpful

**The Troubleshooting Flowchart**

```text
[ DATABASE IS SLOW ]
         |
         v
[ Check Active Connections (DMV / pg_stat_activity) ]
         |
  +------+-------+
  |              |
[ WAITING ]  [ RUNNING ]
  |              |
  v              v
Is it blocked?   Check Execution Plan
(Lock Wait)      (Table Scan? Hash Match?)
  |              |
  v              v
KILL blocker     Missing Index?
OR fix app       Outdated Statistics?
transactions.    Non-SARGable query?
                 |
                 v
            [ IMPLEMENT FIX ]
```

## 29. Enterprise design discussion

**Read-Heavy vs Write-Heavy Monitoring Profiles**
*   **OLTP (Online Transaction Processing):** (e.g., Booking a ticket). You monitor Locks, Deadlocks, Transaction Log write latency, and Buffer Pool churn. Queries should return in milliseconds.
*   **OLAP (Online Analytical Processing):** (e.g., Generating a monthly sales report). You monitor TempDB/Temp file usage (disk spills), Memory Grants (`work_mem`), and CPU parallelization. Queries are expected to take minutes.
*   *Rule:* Never mix them on the same physical server. An OLAP query will flush the Buffer Pool, destroying the cache for the OLTP application. Always route OLAP to a Read-Replica.

## 30. Hands-on exercises

1. In SQL Server, run the `sys.dm_os_wait_stats` query. Research the top 3 wait types you find on Microsoft documentation.
2. In PostgreSQL, modify `postgresql.conf` to add `shared_preload_libraries = 'pg_stat_statements'`. Restart the server. Run a few complex queries, then query `pg_stat_statements` to see them logged.

## 31. Coding exercises

1. Write a diagnostic script that finds the top 5 largest tables in your database by physical disk size. (SQL Server: use `sys.allocation_units`. Postgres: use `pg_total_relation_size()`).
2. Write a script to identify un-used indexes (indexes taking up disk space and slowing down INSERTS, but never being used for read operations).

## 32. Mini project

**Objective:** Performance Tuning the NextEvent Tickets Table.
1. Insert 1,000,000 randomized rows into the `Tickets` table.
2. Run a query: `SELECT * FROM Tickets WHERE UserID = 500`. Note the execution time and the execution plan (it will be a Table Scan/Seq Scan).
3. Create the appropriate Non-Clustered / B-Tree Index.
4. Rerun the query. Compare the new execution time (it should be instant).
5. Open two connections. In Connection 1, `BEGIN TRAN` and `UPDATE` Ticket 1. In Connection 2, try to `UPDATE` Ticket 1. Use the DMVs to identify Connection 2 is blocked by Connection 1.

## 33. Quiz

1. What is the fundamental difference between an OS-level metric (like CPU usage) and a Wait Statistic?
2. What does `PAGEIOLATCH` (SQL Server) or `DataFileRead` (Postgres) indicate about your query performance?
3. Why is table bloat a massive performance issue in PostgreSQL, and what daemon process is responsible for preventing it?

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** How do you find out why a database query is slow?
    *   **A:** I would look at the Execution Plan (using `EXPLAIN ANALYZE` or SQL Server Graphical Plans). I would check if the database is doing a full Table Scan instead of an Index Seek, which usually means an index is missing or the query is not SARGable.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** You see a massive spike in Active Connections, but CPU usage is almost zero. What is the likely cause?
    *   **A:** Severe blocking. Transactions are piling up, waiting for a lock to be released. Because they are in a "Wait" state, they consume zero CPU cycles. I would query the active sessions DMV to find the "Head Blocker" holding the exclusive lock and kill it.
*   **Q:** What is `pg_stat_statements`?
    *   **A:** It is a PostgreSQL extension that records aggregate statistics for all queries executed on the server, tracking execution time, row counts, and disk I/O. It is the primary tool for identifying historically slow queries.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** A specific Stored Procedure in SQL Server is running incredibly slow. You check the Execution Plan and notice it's using a Nested Loop join, but it's processing 10 million rows. You update the Statistics, but the plan doesn't change. What is the root cause?
    *   **A:** The developer is likely dumping the 10 million rows into a Table Variable (`@Table`) instead of a Temporary Table (`#Table`). SQL Server does not maintain statistics for Table Variables and always assumes they contain 1 row, causing the Optimizer to erroneously choose a Nested Loop join. Changing it to a Temp Table will fix the Cardinality Estimation and allow the Optimizer to choose a Hash Match.
*   **Q:** In Postgres, a developer complains that a query is occasionally taking 5 seconds, but when they run `EXPLAIN ANALYZE` in their terminal, it takes 5 milliseconds. What is happening?
    *   **A:** If the application uses an ORM, it is likely using Prepared Statements. After 5 executions, Postgres switches to a "Generic Plan" that ignores the specific parameter values. When the developer runs it manually, they are sending a raw ad-hoc string, forcing Postgres to compile a "Custom Plan" optimized perfectly for that specific value. This is a classic Parameter Sniffing discrepancy.

## 35. Chapter summary

### Learning Summary
We concluded our architectural journey by mastering the art of Observability and Troubleshooting. We learned to abandon guesswork and OS-level metrics in favor of mathematically precise Wait Statistics. We navigated SQL Server's extensive Dynamic Management Views (DMVs) and PostgreSQL's powerful `pg_stat_activity` and `pg_stat_statements`. We explored how missing indexes cause catastrophic locking cascades, why memory pressure is often a symptom rather than a root cause, and how to systematically diagnose a frozen database in production.

### Key Takeaways
*   A query is always either running on the CPU or waiting for a resource. Find out what it is waiting for.
*   High CPU is not necessarily bad; High Disk I/O Wait is catastrophic.
*   SQL Server relies heavily on DMVs and Query Store; Postgres relies on `pg_stat_statements` and OS tuning.
*   Blocking (Lock Waits) will lock up a server while consuming 0% CPU.
*   Performance tuning is a continuous lifecycle: Baseline, Monitor, Analyze, Tune, Repeat.

### Glossary
*   **Wait Statistics:** Metrics detailing exactly what resources a database engine is waiting for.
*   **DMV (Dynamic Management View):** System views in SQL Server exposing internal memory structures.
*   **Head Blocker:** The transaction at the root of a blocking chain holding the initial exclusive lock.
*   **Cache Hit Ratio:** The percentage of times the database found data in RAM without having to read from physical disk.
*   **PLE (Page Life Expectancy):** How long data pages survive in SQL Server memory before being flushed.

### Common Mistakes
*   Adding RAM to solve an I/O problem caused by a missing index.
*   Using `SELECT *` without filtering, forcing the database to push valuable data out of the Buffer Pool.
*   Ignoring Autovacuum warnings in PostgreSQL until the database shuts down.

### Best Practices
*   Enable `pg_stat_statements` (Postgres) and Query Store (SQL Server) in all production environments.
*   Set up alerts for high concurrency locks, low cache hit ratios, and long-running active transactions.
*   Always include the Application Name in your connection strings to make troubleshooting easier.

### Concluding Remarks
This concludes the comprehensive guide to Enterprise Database Architecture. From the mathematical foundations of Relational Theory, through the physical geometry of B-Trees and Storage Engines, into the complexities of ACID transactions, Distributed Sharding, and Security, you are now equipped with the deep internal knowledge required to design, scale, and rescue world-class database systems. 

**End of Book.**
