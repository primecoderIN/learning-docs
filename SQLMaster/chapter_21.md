# Chapter 21: Statistics & Parameter Sniffing

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand how SQL Server builds **Statistics** (Histograms and Density Vectors) to map data distribution.
*   Identify why stale statistics cause the Query Optimizer to choose disastrous Execution Plans.
*   Diagnose **Parameter Sniffing**, the most notorious performance bug in Multi-Tenant SaaS applications.
*   Implement architectural fixes for Parameter Sniffing, including `RECOMPILE`, `OPTIMIZE FOR UNKNOWN`, and dynamic SQL.

---

## 21.1 What are Statistics?

In Chapter 20, we learned that the Query Optimizer evaluates multiple execution paths and chooses the one with the lowest cost.
But *how* does it calculate the cost? How does it know that `Status = 'Faulted'` will return 5 rows, while `Status = 'Active'` will return 5,000,000 rows?

It uses **Statistics**.
Statistics are hidden BLOB objects in the database that describe the distribution of data in a column.
Every time you create an Index, SQL Server automatically creates a Statistic object for the indexed columns.

### The Histogram
A Statistic object contains a **Histogram**. 
SQL Server scans the column data, sorts it, and breaks it into a maximum of 200 "steps" or "buckets." 
For each bucket, the Histogram records:
*   The upper bound value.
*   How many rows exactly match that value.
*   How many rows fall between this bucket and the previous bucket.

When you run `WHERE TenantId = 'T1-UUID'`, the Optimizer looks at the Histogram, finds the bucket for 'T1-UUID', and instantly knows exactly how many rows it will likely return.

---

## 21.2 Stale Statistics

As your SaaS application inserts and deletes millions of rows, the distribution of data changes. The Histogram becomes inaccurate.
If the Histogram says a Tenant has 5 rows (but they actually have 500,000 rows today), the Optimizer will compile a plan optimized for 5 rows (Nested Loops). When that plan runs against 500,000 rows, it will crash the server.

*   **Auto-Update Statistics:** By default, SQL Server automatically updates statistics in the background when it detects that roughly 20% of the rows in a table have changed.
*   **The Architect's Fix:** In a 100-million row table, 20% is 20 million rows. That means your stats can be horribly wrong for weeks before SQL Server updates them. Enterprise DBAs schedule nightly SQL Agent jobs (using Ola Hallengren's scripts) to manually `UPDATE STATISTICS` on all heavily used tables.

---

## 21.3 Parameter Sniffing (The Multi-Tenant Curse)

In a Multi-Tenant SaaS, some customers are massive (Acme Corp has 5,000 Stations), and some are tiny (Bob's Coffee has 1 Station). This data skew introduces a nightmare scenario known as **Parameter Sniffing**.

### The Scenario
You have a Stored Procedure (or an EF Core parameterized query):
```sql
CREATE PROCEDURE usp_GetStations @TenantId UNIQUEIDENTIFIER
AS
    SELECT * FROM core.Stations WHERE TenantId = @TenantId;
```

1.  **Monday 8:00 AM:** The SQL Server cache is cleared (server reboot).
2.  **Monday 8:01 AM:** Bob's Coffee (`@TenantId = 'Tiny'`) logs in.
    *   The Optimizer compiles the Stored Procedure.
    *   It "sniffs" the parameter (`'Tiny'`).
    *   It checks the Statistics: "Ah, 'Tiny' only has 1 row."
    *   It compiles Plan A (Index Seek + Key Lookup) and saves Plan A in the RAM cache.
    *   The query takes 2 milliseconds.
3.  **Monday 8:05 AM:** Acme Corp (`@TenantId = 'Massive'`) logs in.
    *   SQL Server sees the procedure is already in the RAM cache! It does *not* recompile it.
    *   It executes Plan A (Index Seek + Key Lookup) for Acme Corp.
    *   Acme has 5,000 stations. SQL Server executes 5,000 random I/O Key Lookups. 
    *   The query takes 45 seconds and times out the API.

*The DBA looks at the database: CPU is at 100%. The DBA runs `EXEC usp_GetStations 'Massive'` in SSMS, and it runs in 1 second! Why? Because the DBA's SSMS connection uses different `SET` options, forcing a new compilation.*

This is Parameter Sniffing. The cached plan was optimized for the *first* parameter it saw, which is terrible for subsequent, vastly different parameters.

---

## 21.4 Fixing Parameter Sniffing

There are several ways to fix this, depending on your architecture.

### Fix 1: `OPTION (RECOMPILE)`
You can instruct SQL Server to *never* cache the plan. It will compile a brand new, perfect plan every single time the query runs.
*   **Pros:** Guarantees the perfect plan for every tenant.
*   **Cons:** Compiling queries takes CPU. If this query runs 10,000 times a second, recompiling it will burn out your CPU.
```sql
SELECT * FROM core.Stations WHERE TenantId = @TenantId 
OPTION (RECOMPILE);
```

### Fix 2: `OPTION (OPTIMIZE FOR UNKNOWN)`
This tells the Optimizer: "Do not sniff the parameter. Ignore it. Look at the total density of the table and guess an average plan."
*   **Pros:** Provides a stable, mediocre plan that won't timeout for massive tenants. Uses the cache.
*   **Cons:** Bob's Coffee might get a Clustered Index Scan instead of a Seek.
```sql
SELECT * FROM core.Stations WHERE TenantId = @TenantId 
OPTION (OPTIMIZE FOR UNKNOWN);
```

### Fix 3: Dynamic SQL (The Architect's Choice)
If you build the SQL string dynamically (as shown in Chapter 14), SQL Server caches the plan based on the literal string. 
This is why EF Core rarely suffers from Parameter Sniffing on simple filters—if you pass a new parameter value but use `IQueryable` chaining, EF Core often manages parameterization efficiently, though EF Core 5+ queries *can* sniff. To be perfectly safe across tenants, some architects inject the `TenantId` as a literal (for trusted integers) rather than a parameter, forcing a unique plan per tenant.

---

## 21.5 The Code: EF Core Query Hints

In modern EF Core, you can inject SQL Server query hints directly into your LINQ queries using taggers or raw SQL.

In EF Core 8 (and via third-party interceptors), you can force a recompile for massive reporting queries:
```csharp
// (Requires EF Core Interceptor or raw SQL in older versions)
// In EF 8, you can sometimes achieve this via raw SQL views or tag with:
var query = await context.Stations
    .TagWith("OPTION(RECOMPILE)") // A DBA interceptor can replace this tag with the actual hint
    .Where(s => s.TenantId == tenantId)
    .ToListAsync();
```

---

## 21.6 Performance & Security Analysis

### Performance Analysis: Ascending Keys
A massive problem with Statistics occurs on Ascending Keys (like `Identity` or `DateTime2` columns). If your statistics update at midnight, the histogram only knows about dates up to midnight. At 2:00 PM, a query asks for data `WHERE StartTime > 1:00 PM`. The Histogram thinks **0 rows** exist for that time, because it hasn't been updated yet! 
The Optimizer chooses a terrible plan (Estimate 1 row, Actual 50,000 rows). 
*Fix:* Enable Trace Flag 2389, or in modern SQL Server (2014+), the New Cardinality Estimator handles this ascending key problem automatically.

### Security Implications
*   None directly related to statistics, but remember that Statistics contain a sampling of your actual data. If you send a database backup or export statistics to a vendor for tuning, you are leaking actual customer data values (e.g., Tenant UUIDs, exact timestamps).

---

## 21.7 Common Mistakes & Production Pitfalls

1.  **Local Variables in Stored Procedures:** 
    A common "hack" to fix Parameter Sniffing is to copy the parameter into a local variable:
    ```sql
    CREATE PROC usp_Get (@Param INT) AS
    DECLARE @Local INT = @Param;
    SELECT * FROM Table WHERE Col = @Local;
    ```
    Developers think this is clever. It is not. SQL Server cannot sniff local variables. This has the exact same effect as `OPTIMIZE FOR UNKNOWN`. It forces a generic, often poor plan. Only do this intentionally.

---

## 21.8 Production Checklist

*   [ ] Nightly database maintenance jobs are scheduled to `UPDATE STATISTICS` with a `FULLSCAN` on highly volatile tables during off-peak hours.
*   [ ] Multi-tenant reporting queries (which vary wildly based on the tenant size) utilize `OPTION (RECOMPILE)` to guarantee accurate plans for massive tenants.
*   [ ] If a Stored Procedure times out in the API but runs instantly in SSMS, the team immediately diagnoses it as Parameter Sniffing.

---

## 21.9 Exercises

1.  **Diagnosis:** A user reports that running the "Monthly Revenue Report" for the month of January takes 1 second. Running the exact same report for the month of July (which had a huge marketing campaign and 100x more sessions) takes 5 minutes. The DBA runs `DBCC FREEPROCCACHE` (which clears all cached plans), and suddenly July takes 1 second. Explain exactly what happened.
2.  **Fixing the Code:** Write the T-SQL required to append the "Optimize for Unknown" hint to a `SELECT` statement querying `core.Sessions` by `TenantId`.

---

## 21.10 Interview Questions

**Q1: What is Parameter Sniffing, and why is it a massive problem in Multi-Tenant SaaS databases?**
*Answer:* Parameter Sniffing occurs when SQL Server compiles an execution plan for a parameterized query based on the *first* parameter value it receives, and then caches that plan for all future executions. In a multi-tenant SaaS, data distribution is highly skewed (some tenants have 1 row, some have 1,000,000 rows). If a small tenant triggers the compilation, the cached plan (e.g., Key Lookups) will be catastrophically slow when a massive tenant subsequently runs the exact same query.

**Q2: How does `OPTION (RECOMPILE)` fix Parameter Sniffing, and what is its architectural trade-off?**
*Answer:* `OPTION (RECOMPILE)` instructs the Query Optimizer to completely bypass the plan cache. It will look at the exact parameters provided, read the current Statistics, compile a perfect plan for those specific parameters, execute it, and then throw the plan away. The trade-off is CPU utilization. Compiling plans is CPU-intensive. If the query is executed 5,000 times a second (e.g., a simple lookup), recompiling it every time will burn out the database CPU. It should only be used for complex, varying reports.

---
**Next up in Chapter 22:** We will conclude Part 6 by discussing how to manage multi-terabyte tables. We will explore Table Partitioning, Sliding Windows, and Archiving strategies for massive IoT data.
