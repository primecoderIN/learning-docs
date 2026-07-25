# Part 9: Monitoring & Maintenance

# Chapter 31: Query Store & Performance Monitoring

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand why SQL Server Profiler is obsolete and dangerous for production monitoring.
*   Enable and configure **Query Store** (the "Flight Data Recorder" for SQL Server).
*   Diagnose **Plan Regressions** (why a query that ran fine yesterday is timing out today).
*   Force Execution Plans to instantly resolve production outages.
*   Analyze **Wait Statistics** to determine exactly what a slow query is waiting for.

---

## 31.1 The Legacy Trap: SQL Server Profiler

For 15 years, when a database was slow, DBAs opened a tool called SQL Server Profiler. They connected to the production server and watched a live stream of every single query executing in real-time.

**Why Profiler is Dead:**
Running Profiler on a high-throughput SaaS database intercepts every transaction, causing massive CPU spikes. It can literally crash a production server. Furthermore, if you weren't watching Profiler at 2:00 AM when the outage occurred, the data is gone forever.

Microsoft formally deprecated Profiler for query performance tuning. The modern standard is **Query Store**.

---

## 31.2 The "Flight Data Recorder": Query Store

Introduced in SQL Server 2016 (and enabled by default in Azure SQL), **Query Store** acts as an airplane's black box.

It runs continuously in the background with negligible overhead (1-2%). It silently captures:
1.  The text of every query executed.
2.  The exact Execution Plan chosen by the Optimizer.
3.  Runtime statistics (How many CPU milliseconds it took, how much memory it used, how many physical reads it performed).

If an outage occurs at 2:00 AM, you can log in at 9:00 AM, open Query Store in SSMS, and see exactly what was burning CPU at that specific minute.

### Enabling Query Store
If you are on-premise, you must enable it manually:
```sql
ALTER DATABASE VoltCore 
SET QUERY_STORE = ON 
    (OPERATION_MODE = READ_WRITE, 
     CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30), 
     DATA_FLUSH_INTERVAL_SECONDS = 900, 
     MAX_STORAGE_SIZE_MB = 1000);
```

---

## 31.3 Identifying CPU Hogs

When the API starts throwing 500 Timeouts, the first place an Architect looks is the **"Top Resource Consuming Queries"** report in Query Store.

Query Store provides a visual bar chart of queries ranked by Total CPU Time, Execution Count, or Logical Reads.
Usually, a 100% CPU spike is caused by a single rogue query. 

*   If a query has a **High Execution Count** and **Low CPU per execution**: The API has an N+1 bug (Chapter 23).
*   If a query has a **Low Execution Count** and **Massive CPU per execution**: The query is doing a Clustered Index Scan on a massive table due to a missing index (Chapter 19) or Parameter Sniffing (Chapter 21).

---

## 31.4 Plan Regression (When good queries go bad)

The most terrifying production issue is when no code was deployed, no traffic spiked, but the database suddenly locks up.
This is almost always **Plan Regression**.

**The Scenario:**
1.  For 6 months, `usp_GetStations` has executed beautifully using Plan A (Index Seek).
2.  Overnight, the Statistics Auto-Update kicks in, or the plan cache is cleared.
3.  The next morning, a massive tenant executes the query first. The Optimizer gets confused and compiles Plan B (Clustered Index Scan).
4.  Plan B is cached. Every tenant is now forced to use the terrible Plan B. The server CPU hits 100%.

Query Store has a specific report called **"Regressed Queries"**. It graphically shows you: "This query used to take 2ms yesterday, and today it takes 40,000ms."

---

## 31.5 Forcing Execution Plans (The Quick Fix)

When Plan Regression causes an outage, you don't have time to write new code, test an `OPTION (RECOMPILE)` hint, push it through CI/CD, and deploy it. You need the API back online in 10 seconds.

Query Store allows you to **Force a Plan**.

In the SSMS GUI (or via T-SQL), you click on the Query. Query Store shows you a history of all plans it has ever used. You click on Plan A (the good one from yesterday), and click "Force Plan".
```sql
-- The T-SQL equivalent of the UI button
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 105;
```

Instantly, the Query Optimizer is overridden. Every time that query executes, it is strictly forced to use Plan A, regardless of parameters or statistics. The CPU drops to 5%, and the outage is resolved.
*(Architect Note: Forcing a plan is a temporary tourniquet, not a cure. You must still fix the underlying missing index or parameter sniffing bug in the next sprint).*

---

## 31.6 Architect Perspective: Wait Statistics

If a query takes 10 seconds to run, but Query Store shows it only consumed 50 milliseconds of CPU, what was it doing for the other 9,950 milliseconds?

It was **Waiting**.

SQL Server tracks exactly why a thread is waiting. In modern versions, Query Store captures these **Wait Statistics**.
*   **LCK_M_X / LCK_M_S:** The query is waiting on a Lock. (Blocking/Deadlocks - Chapter 17).
*   **PAGEIOLATCH_SH:** The query is waiting for the hard drive to read data from the MDF file into RAM. (Missing index causing a table scan, or slow storage).
*   **ASYNC_NETWORK_IO:** The database finished the query instantly, but the C# application is reading the data too slowly over the network (e.g., executing business logic inside a `DataReader` loop).

By looking at Wait Stats, you stop guessing and immediately know if you have a CPU problem, a Disk I/O problem, or an Application Code problem.

---

## 31.7 The Code: Querying Query Store via T-SQL

While the SSMS GUI is fantastic, Architects often write raw SQL to monitor Query Store programmatically or feed data into dashboards (like Grafana).

```sql
-- Find the top 5 most expensive queries by Total CPU in the last hour
SELECT TOP 5
    q.query_id,
    t.query_sql_text,
    rs.count_executions,
    rs.avg_cpu_time / 1000.0 AS avg_cpu_ms,
    (rs.count_executions * rs.avg_cpu_time) / 1000.0 AS total_cpu_ms
FROM sys.query_store_query q
JOIN sys.query_store_query_text t ON q.query_text_id = t.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval rsi ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(HOUR, -1, GETUTCDATE())
ORDER BY (rs.count_executions * rs.avg_cpu_time) DESC;
```

---

## 31.8 Performance & Security Analysis

### Performance Analysis: Plan Forcing Failures
What happens if you Force Plan A (which relies on `IX_Stations_TenantId`), and a junior DBA accidentally drops `IX_Stations_TenantId` next week?
SQL Server is smart. If a Forced Plan becomes physically impossible to execute (because an index is missing), SQL Server will silently un-force the plan, compile a new one, and generate a Warning Event. The query will not fail, but performance may regress.

### Security Implications
*   **Data Masking in Query Store:** Query Store captures the exact parameterized SQL text submitted by EF Core. If your API passes sensitive PII (like Credit Card numbers or plain-text passwords) in the `WHERE` clause instead of hashing them in the application layer, those values might be visible in the Query Store views. Always encrypt or hash sensitive data before it leaves the C# application.

---

## 31.9 Common Mistakes & Production Pitfalls

1.  **Over-allocating Query Store Max Size:** Query Store lives entirely inside your Primary Filegroup (MDF). If you set `MAX_STORAGE_SIZE_MB` to 50,000 (50GB), and you run out of disk space, your database will crash. Keep the size reasonable (e.g., 1GB - 5GB) and let the `CLEANUP_POLICY` automatically purge data older than 30 days.
2.  **Using Profiler for Tuning:** If a DBA opens SQL Server Profiler on a high-throughput production database in 2026, they are committing architectural malpractice. Use Query Store or Extended Events (XEvents).

---

## 31.10 Production Checklist

*   [ ] Query Store is explicitly enabled on all on-premise databases (it is default on Azure SQL).
*   [ ] The "Regressed Queries" report is checked weekly to catch degrading execution plans before they cause outages.
*   [ ] `sp_query_store_force_plan` is utilized as a first-line defense during severe, unexplainable CPU spikes.
*   [ ] Wait Statistics are analyzed to differentiate between Network I/O, Disk I/O, and CPU bottlenecks.

---

## 31.11 Exercises

1.  **Diagnosis:** An API endpoint starts timing out at 3:00 PM. You check Query Store. The query execution time jumped from 5ms to 12,000ms. However, the CPU time is only 15ms. You check the Wait Statistics for that query, and it shows 11,985ms of `LCK_M_X`. What is the exact root cause of the timeout?
2.  **Mitigation:** A query that normally does an Index Seek has randomly regressed into a Clustered Index Scan, consuming 100% of the server CPU. Write the steps you would take to resolve the outage within 60 seconds without altering any application code.

---

## 31.12 Interview Questions

**Q1: Contrast SQL Server Profiler with Query Store. Why has the industry abandoned Profiler for performance tuning?**
*Answer:* SQL Server Profiler works by intercepting and logging every single event in real-time, which introduces massive "Observer Overhead" and can severely degrade or crash a production server under heavy load. Furthermore, its data is ephemeral; if you aren't running it when the issue occurs, you miss it. Query Store is integrated directly into the database engine with negligible overhead. It constantly records query texts, execution plans, and runtime statistics to disk, allowing architects to retroactively analyze performance regressions hours or days after they occur.

**Q2: What is "Plan Regression" and how does Query Store allow you to mitigate it instantly?**
*Answer:* Plan Regression occurs when the Query Optimizer discards a highly efficient execution plan and compiles a new, highly inefficient plan (often due to stale statistics or Parameter Sniffing). The query execution time "regresses" from milliseconds to seconds, causing outages. Query Store mitigates this by maintaining a historical record of all plans previously used by the query. An architect can use the UI or `sp_query_store_force_plan` to instantly override the Optimizer and force it to revert to the previously known good plan.

---
**Next up in Chapter 32:** We will dive into Database Maintenance. We will learn how to automate Index Rebuilds, manage Transaction Log growth, and implement Ola Hallengren's legendary maintenance scripts.
