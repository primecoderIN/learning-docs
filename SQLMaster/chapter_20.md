# Chapter 20: Execution Plans & Query Optimization

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand how the Cost-Based Query Optimizer chooses an Execution Plan.
*   Read Graphical Execution Plans (Right-to-Left, Top-to-Bottom).
*   Visually identify the difference between Index Seeks, Index Scans, and Key Lookups.
*   Understand the "Tipping Point" (why SQL Server sometimes ignores a perfectly good index).
*   Extract raw SQL from EF Core to analyze it in SQL Server Management Studio (SSMS) or Azure Data Studio.

---

## 20.1 The Query Optimizer

When you submit a query (or EF Core submits one for you), SQL Server does not just blindly execute it. It passes the text to the **Query Optimizer**.

SQL Server's Query Optimizer is **Cost-Based**. 
1.  It parses your SQL.
2.  It looks at the available Indexes and Statistics (metadata about data distribution).
3.  It generates dozens of potential execution paths (Plans).
4.  It estimates the CPU and I/O "Cost" for each path.
5.  It selects the plan with the lowest cost and executes it.

*Architect Note:* The Optimizer doesn't look for the *perfect* plan; it looks for a "good enough" plan as quickly as possible. If compiling the perfect plan takes 5 seconds, but a decent plan takes 5 milliseconds, it chooses the decent plan.

---

## 20.2 Reading Execution Plans

You can view Execution Plans in SSMS by clicking **"Include Actual Execution Plan"** (or pressing `Ctrl+M`) before running a query.

### How to Read the Plan
1.  **Read Right-to-Left, Top-to-Bottom.** The operations on the far right (like reading from tables) feed data to the operations on the left (like sorting or joining), ending at the final `SELECT` node on the far left.
2.  **Look at the Line Thickness.** The arrows connecting operators indicate data flow. Thick arrows mean millions of rows are moving between operators. Thin arrows mean few rows. If you see a massively thick arrow feeding into a "Filter" operator, you have a non-SARGable query that is reading too much data from disk.

---

## 20.3 Seeks vs. Scans

The icons on the far right of the plan tell you how data was physically retrieved.

### 1. Index Seek (The Goal)
*   **What it does:** The engine navigated the B-Tree directly to the exact rows requested.
*   **Performance:** $O(\log N)$. Instantaneous.
*   **When it happens:** You filtered using a highly selective, SARGable `WHERE` clause on an indexed column.

### 2. Index Scan / Clustered Index Scan (The Warning)
*   **What it does:** The engine started at page 1 and read every single page until the end of the table.
*   **Performance:** $O(N)$. Slow on large tables. Causes heavy I/O.
*   **When it happens:** You have no index, you used a non-SARGable function (`WHERE YEAR(Date) = 2024`), or you used a leading wildcard (`LIKE '%Admin'`).

*Exception:* An Index Scan on a table with 50 rows is fine. An Index Scan on a 500-million row telemetry table is a critical incident.

---

## 20.4 Key Lookups and Spools

As discussed in Chapter 19, if the Optimizer uses a Non-Clustered index but needs columns that aren't in the index, it performs a **Key Lookup**.
In an Execution Plan, this appears as two operators tied to a **Nested Loops Join**:
1.  Top branch: Index Seek (Non-Clustered).
2.  Bottom branch: Key Lookup (Clustered).

*Fix:* Add an `INCLUDE` clause to the Non-Clustered Index.

### Spools
If you see a **Table Spool** or **Window Spool** icon, SQL Server has stopped processing the pipeline, built a hidden temporary table in TempDB, and written data into it (often to sort it or support a Window Function). Spools are expensive. Fix them by providing indexes that pre-sort the data requested in your `ORDER BY` or `PARTITION BY` clauses.

---

## 20.5 Estimated vs. Actual Execution Plans

*   **Estimated Plan:** Tells you what SQL Server *thinks* it will do. It does not actually execute the query. Fast, but can be inaccurate if statistics are out of date.
*   **Actual Plan:** Executes the query and compares the estimated row counts to the actual row counts. 

**The Missing Index Warning:** If you look at an execution plan (Estimated or Actual), SQL Server might display green text at the top: *"Missing Index (Impact 95%)..."*
*Architect Warning:* Never blindly apply these. The Optimizer only looks at the current query; it does not consider the Write Penalty on the whole system. Review missing index requests holistically.

---

## 20.6 Architect Perspective: The "Tipping Point"

A junior DBA adds a Non-Clustered index on `Status`. They write `SELECT * FROM Sessions WHERE Status = 'Faulted'`. They look at the execution plan, and to their horror, SQL Server ignored the index and did a Clustered Index Scan! Why?

**The Tipping Point:**
Because the query uses `SELECT *`, the engine must perform a Key Lookup to get the missing columns. Key Lookups require random I/O. 
If SQL Server estimates that 'Faulted' sessions make up more than ~1% to 3% of the entire table, it calculates that doing 50,000 Key Lookups is actually *slower* than just scanning the entire Clustered Index using fast, sequential I/O. 
Once a query asks for too many rows, the Optimizer "tips" over and chooses a Scan instead of a Seek + Lookup.

*The Fix:* Use a Covering Index, or project fewer columns (no `SELECT *`).

---

## 20.7 The Code: Extracting EF Core SQL

To tune queries, you must intercept the SQL EF Core generates.

**EF Core 5+ `.ToQueryString()`**
You can instantly view the raw SQL of any LINQ query before it executes:
```csharp
var query = _context.Sessions
    .Where(s => s.TenantId == tenantId && s.Status == "Active")
    .Select(s => new { s.SessionId, s.TotalCost });

// Prints the exact T-SQL to the console, perfect for pasting into SSMS
string sql = query.ToQueryString();
Console.WriteLine(sql); 

var results = await query.ToListAsync();
```

---

## 20.8 Performance & Security Analysis

### Performance Analysis: Implicit Conversions
Look at the Execution Plan properties (F4) for your `SELECT` node. If you see a warning for `PlanAffectingConvert`, it means an Implicit Conversion occurred (e.g., comparing a `VARCHAR` column to an `NVARCHAR` string). This silently prevents an Index Seek. Ensure your EF Core column mappings (`HasColumnType`) match your C# data types (`string`) perfectly.

### Security Implications
*   **SHOWPLAN Permissions:** By default, standard application users cannot view execution plans. To allow a developer to debug production (not recommended, but happens), they need the `VIEW SERVER STATE` and `SHOWPLAN` permissions. Be cautious granting these, as execution plans can leak sensitive data (parameter values are visible in the plan XML).

---

## 20.9 Common Mistakes & Production Pitfalls

1.  **Ignoring Warning Triangles:** If an operator in a plan has a yellow warning triangle, hover over it. It usually means a TempDB Spill occurred (a Sort or Hash Match ran out of RAM). This is a massive performance bottleneck.
2.  **Focusing on Percentages:** The plan shows cost percentages (e.g., Index Seek 10%, Key Lookup 90%). These are *estimated* costs. If a query runs in 5 milliseconds, a 90% Key Lookup is irrelevant. Only tune queries that are actually causing CPU/IO bottlenecks (identified via Query Store, covered in Chapter 31).

---

## 20.10 Production Checklist

*   [ ] Developers use `.ToQueryString()` to extract and review the T-SQL generated by complex EF Core LINQ queries.
*   [ ] Queries with high elapsed times are analyzed in SSMS using "Include Actual Execution Plan".
*   [ ] Thick data flow arrows in execution plans are investigated to ensure filtering is happening as early as possible.
*   [ ] Key Lookups on high-traffic queries are eliminated via Covering Indexes (`INCLUDE`).

---

## 20.11 Exercises

1.  **The Tipping Point:** A table has 1,000,000 rows. A Non-Clustered index exists on the `IsActive` column. `990,000` rows are `IsActive = 1`. `10,000` rows are `IsActive = 0`. 
    If you run `SELECT * FROM Users WHERE IsActive = 1`, will SQL Server use an Index Seek or a Clustered Index Scan? Why?
2.  **Plan Analysis:** You are looking at an execution plan. The furthest right operator is an "Index Scan". The arrow leaving it is incredibly thick. It feeds into a "Filter" operator. The arrow leaving the Filter operator is incredibly thin. What architectural mistake does this pattern represent?

---

## 20.12 Interview Questions

**Q1: What is the difference between an Estimated Execution Plan and an Actual Execution Plan, and why might they look different?**
*Answer:* An Estimated Plan is compiled by the Query Optimizer without executing the query, relying purely on Statistics metadata to guess row counts. An Actual Plan executes the query and tracks the exact number of rows that flowed through each operator. They can look wildly different if the table's Statistics are out of date (e.g., the Optimizer estimates 1 row will return, so it chooses a Nested Loop join, but at runtime, 10,000,000 rows return, causing the query to run for hours).

**Q2: You see a "Key Lookup" operator in an execution plan that is consuming 85% of the query cost. How do you eliminate it?**
*Answer:* A Key Lookup means the Query Optimizer used a Non-Clustered index, but that index did not contain all the columns requested in the `SELECT` or `WHERE` clauses. I would eliminate it by altering the Non-Clustered index to include the missing columns using the `INCLUDE` clause, turning it into a Covering Index.

---
**Next up in Chapter 21:** We've mentioned "Statistics" several times. In the next chapter, we will demystify what Statistics are, how the Optimizer uses them, and how to fix the dreaded "Parameter Sniffing" problem.
