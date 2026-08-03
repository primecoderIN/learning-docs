# Chapter 5 – Query Optimizer and Execution Plans

## 1. Concept Overview

SQL (Structured Query Language) is a **declarative** language. When you write a SQL query, you are telling the database *what* data you want (e.g., "Give me all VIP users"), but you are explicitly *not* telling the database *how* to get it. 

The **Query Optimizer** is the most complex, highly guarded, and mathematically sophisticated component of any Relational Database Management System (RDBMS). Its job is to take your declarative SQL, analyze the available indexes, evaluate the statistical distribution of the data, calculate the CPU/Memory/IO costs of thousands of potential execution paths, and choose the most efficient physical path to retrieve the data. 

The chosen path is called the **Execution Plan** (or Query Plan). It dictates exactly which indexes to seek, which tables to scan, and which algorithms to use to join the tables together (Nested Loop, Hash Match, or Merge Join). 

## 2. History

Early database systems required developers to manually write the physical navigation path to the data (procedural code). If an index was added or dropped, the application code broke and had to be rewritten. In 1979, Pat Selinger and her team at IBM's System R project published a seminal paper on **Cost-Based Optimization**. They invented the modern approach where the database maintains statistics about the data and mathematically estimates the "cost" of different execution paths. Every modern RDBMS optimizer (including SQL Server and Postgres) is a direct descendant of the Selinger Optimizer.

## 3. Real-world analogy

Imagine using **Google Maps** to drive from New York to Los Angeles.

*   **The Declarative Query:** You type "Los Angeles" into the destination box. You do not tell Google Maps which streets to turn on.
*   **The Statistics:** Google Maps looks at its internal data: distance, speed limits, current traffic jams, and road closures.
*   **The Cost-Based Optimizer:** Google Maps calculates 50 different potential routes. Route A takes 40 hours (Highway). Route B takes 55 hours (Backroads). Route C takes 42 hours but requires tolls.
*   **The Execution Plan:** Google Maps chooses Route A, drawing the blue line on your screen, providing step-by-step physical instructions (Turn left on Main St., Merge onto I-80).
*   **Parameter Sniffing (The Error):** If you asked for a route at 3:00 AM, Google might route you right through downtown Chicago. If you reuse that exact same route at 5:00 PM (Rush Hour), it will be a disaster. The plan was optimized for a different "parameter."

## 4. Business problem solved

The Optimizer abstracts the physical data layer from the application. It allows DBAs to add indexes, partition tables, or upgrade hardware, and the database will automatically adapt its execution plans to become faster *without requiring a single line of application code to change*.

---

## 5. Microsoft SQL Server explanation

The SQL Server Query Optimizer is a cost-based optimizer. It does not guarantee the *absolute best* plan; finding the perfect plan for a 10-table join could take hours of compute time. Instead, it aims to find a *good enough* plan in the shortest amount of time.

SQL Server heavily relies on **Statistics**. A Statistics object is a mathematical histogram that describes the distribution of values in a column. For example, if a column contains `Country`, the histogram tells the Optimizer that 'USA' occurs 50 million times, while 'Vatican City' occurs 3 times.

Based on this Cardinality Estimation (guessing how many rows will be returned), the Optimizer chooses one of three physical Join operators:
1.  **Nested Loops Join:** Excellent for small row counts. For every row in Table A, loop through Table B. (O(N*M))
2.  **Merge Join:** Excellent if both inputs are already sorted (e.g., via Clustered Indexes). Zips them together. (O(N+M))
3.  **Hash Match Join:** Used for massive, unsorted datasets. Builds an in-memory hash table of the smaller input, then probes it with the larger input. High memory requirement.

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Ensure auto-statistics are on (Best Practice)
ALTER DATABASE NextEventDB SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE NextEventDB SET AUTO_UPDATE_STATISTICS ON;
GO

-- 2. Turn on physical execution stats for the current session
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

-- 3. Execute a query (In SSMS, press Ctrl+M first to include the Actual Execution Plan)
SELECT e.Title, l.Name, l.Capacity
FROM Core.Events e
INNER JOIN Venues.Locations l ON e.LocationID = l.LocationID
WHERE l.Capacity > 5000;
GO
```

## 7. SQL Server internals

**The Plan Cache:** Compiling an Execution Plan takes significant CPU time. To optimize this, SQL Server places the compiled plan into a memory area called the Plan Cache. The next time the exact same query is executed, SQL Server skips the Optimizer entirely and reuses the cached plan. 

**Parameter Sniffing:** This caching behavior is the #1 cause of performance instability in SQL Server. If a Stored Procedure is compiled the very first time using `@EventID = 1` (an event with 5 tickets), the Optimizer caches a plan using a Nested Loop. If the procedure is called 5 minutes later using `@EventID = 99` (the SuperBowl with 100,000 tickets), SQL Server blindly reuses the cached Nested Loop plan, causing the CPU to spike to 100% and the query to hang for 20 minutes.

## 8. SQL Server execution

To read a SQL Server Graphical Execution Plan (in SSMS), you read **Right to Left, Top to Bottom**.

1.  Far Right (Top): `Clustered Index Seek` on `Venues.Locations`. The Optimizer estimates 10 rows.
2.  Far Right (Bottom): `Index Seek` on `Core.Events`.
3.  Middle: The two streams flow into a `Nested Loops (Inner Join)` operator.
4.  Far Left: `SELECT` node. This displays the total cost.

The most critical thing to look for is the difference between **Estimated Number of Rows** and **Actual Number of Rows**. If Estimated is 1, and Actual is 10,000,000, your Statistics are horribly outdated, and the Optimizer chose the wrong algorithm.

## 9. SQL Server enterprise examples

*   **Query Store:** Introduced in SQL Server 2016, this is a revolutionary enterprise feature. It acts as a "flight data recorder" for the database, automatically saving a history of all execution plans. If a DBA notices that a query suddenly slowed down today, they can use Query Store to instantly "Force" the database to use yesterday's execution plan.

## 10. SQL Server performance considerations

*   **Table Variables vs. Temp Tables:** A classic SQL Server trap. Table Variables (`@MyTable`) historically do not maintain statistics; the Optimizer always guesses they contain exactly 1 row. If you insert 100,000 rows into a Table Variable and join it, the Optimizer will choose a Nested Loop and crash your performance. Temporary Tables (`#MyTable`) maintain full statistics and are highly preferred for large intermediate datasets.

## 11. SQL Server security considerations

*   To view Actual Execution Plans, a developer needs the `SHOWPLAN` database permission. This is generally safe to grant in non-production environments, but in production, plans can inadvertently expose sensitive data values embedded in the plan's parameter list.

## 12. SQL Server common mistakes

*   **Blindly adding Missing Indexes:** The Execution Plan GUI often displays green text: *"Missing Index (Impact 95%)"*. Junior DBAs will right-click and create it. Do not do this blindly. The Optimizer only looked at this *single* query; it did not consider that adding this index might slow down 50 other `INSERT` statements.
*   **Query Hints:** Using `WITH (NOLOCK)` or `OPTION (RECOMPILE)` on every query. Hints override the Optimizer's math. `RECOMPILE` defeats the Plan Cache, causing massive CPU spikes from constant recompilation.

## 13. SQL Server best practices

*   Enable **Query Store** on all databases.
*   If you suffer from Parameter Sniffing in a stored procedure, use `OPTION (OPTIMIZE FOR UNKNOWN)` to force the Optimizer to use an average statistical distribution, yielding a safe, middle-of-the-road execution plan.

---

## 14. PostgreSQL explanation

The PostgreSQL Planner/Optimizer is equally sophisticated but handles caching very differently. In Postgres, the planner uses the GEQO (Genetic Query Optimizer) for queries with massive numbers of joins (default > 12 tables), using biological evolutionary algorithms to find a "good enough" plan without exhaustive mathematical calculation.

Postgres does not have a global, shared Plan Cache for ad-hoc queries like SQL Server. If you send the same text string query 1,000 times to Postgres, it compiles a new plan 1,000 times. Plan caching only occurs when you use explicitly **Prepared Statements** at the session level.

## 15. PostgreSQL syntax

In Postgres, you view execution plans using the `EXPLAIN` keyword.

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. View the Estimated Plan (Instant, does not run the query)
EXPLAIN 
SELECT e.title, l.name
FROM core.events e
JOIN venues.locations l ON e.location_id = l.location_id;

-- 2. View the Actual Plan (Runs the query, returns exact timing and IO)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.title, l.name
FROM core.events e
JOIN venues.locations l ON e.location_id = l.location_id;

-- 3. Update Statistics manually (Postgres Autovacuum usually handles this)
ANALYZE core.events;
```

## 16. PostgreSQL internals

When `EXPLAIN ANALYZE` is run, Postgres outputs a text-based tree (Bottom-Up execution). 

*   **Seq Scan:** Sequential Scan (Table Scan). Reads the entire heap.
*   **Index Scan:** Reads the B-Tree, then fetches the tuple from the Heap.
*   **Index Only Scan:** Reads the B-Tree, and the data is satisfied completely by the index (Covering Index).
*   **Bitmap Heap Scan:** Reads the index, builds a bitmap in memory, sorts it, and fetches from the heap sequentially.

The Postgres Optimizer heavily relies on the `pg_statistic` system catalog, which is populated by the `ANALYZE` command (usually run automatically by the Autovacuum daemon).

## 17. PostgreSQL execution

Example Output of `EXPLAIN ANALYZE`:
```text
Hash Join  (cost=28.50..156.00 rows=1000 width=64) (actual time=0.05..1.20 rows=1000 loops=1)
  Hash Cond: (e.location_id = l.location_id)
  ->  Seq Scan on events e  (cost=0.00..120.00 rows=5000 width=36) (actual time=0.01..0.50 rows=5000 loops=1)
  ->  Hash  (cost=25.00..25.00 rows=200 width=36) (actual time=0.02..0.02 rows=200 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 16kB
        ->  Seq Scan on locations l  (cost=0.00..25.00 rows=200 width=36) (actual time=0.00..0.01 rows=200 loops=1)
```
*How to read this:* Read from the most indented node outwards. 
1. It did a Sequential Scan on `locations` (estimated 200 rows, actually got 200).
2. It built a Hash Table in memory (Memory Usage: 16kB).
3. It did a Sequential Scan on `events`.
4. It joined them using a Hash Join.
*Notice: `cost=0.00..25.00`. Cost is an arbitrary unit (representing disk page fetches). The first number is the startup cost; the second is the total cost.*

## 18. PostgreSQL enterprise examples

*   **Enterprise Monitoring:** Because there is no global plan cache GUI, enterprise Postgres DBAs rely strictly on the `pg_stat_statements` extension. This extension hooks into the executor and records the aggregated execution time, CPU usage, and buffer hits for every query executed on the server, serving as the primary tool for finding slow queries.

## 19. PostgreSQL performance considerations

*   **work_mem:** This is the most critical setting for Postgres execution plans. It dictates how much RAM a *single operation* (like a Sort or Hash Join) can use before it spills to disk (Temp files). If `EXPLAIN ANALYZE` shows `Sort Method: external merge Disk`, your query is writing to disk because `work_mem` is too small. However, setting it too high causes OOM (Out of Memory) crashes because `work_mem` is allocated *per node, per query, per connection*.
*   **Prepared Statements:** ORMs (like Prisma or Hibernate) often use Prepared Statements. Postgres will optimize the plan based on the parameters 5 times. On the 6th time, it switches to a "Generic Plan" (ignoring specific parameter values). This can sometimes cause a massive, sudden performance drop (Postgres's version of Parameter Sniffing).

## 20. PostgreSQL security considerations

*   Like SQL Server, running `EXPLAIN ANALYZE` actually executes the query. If you run `EXPLAIN ANALYZE DELETE FROM Users`, **it will delete the users.** Always wrap modifying queries in a transaction if you are just analyzing them: `BEGIN; EXPLAIN ANALYZE DELETE...; ROLLBACK;`

## 21. PostgreSQL common mistakes

*   **Failing to tune Cost Constants:** The Postgres planner uses configuration variables like `seq_page_cost = 1.0` and `random_page_cost = 4.0`. These defaults assume you are running on a 1990s spinning hard drive. If you are on a modern cloud SSD, `random_page_cost` must be lowered to 1.1, or the Optimizer will wrongly favor Sequential Scans over Index Scans.

## 22. PostgreSQL best practices

*   Always read `EXPLAIN (ANALYZE, BUFFERS)`. The `BUFFERS` keyword is critical—it tells you exactly how many 8KB blocks were read from RAM (`hit`) vs read from Disk (`read`). High disk reads mean your memory tuning (`shared_buffers`) is inadequate or your indexes are missing.
*   If statistics are wrong, do not use hints. Instead, increase the statistics target for the specific column: `ALTER TABLE events ALTER COLUMN location_id SET STATISTICS 1000;`, then run `ANALYZE`.

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Plan Caching** | Global, shared cache | Session-level (Prepared Statements) | SQL Server is highly susceptible to Parameter Sniffing. Postgres avoids it for ad-hoc queries but uses more CPU compiling. |
| **Execution Plan Format**| Graphical (SSMS), XML | Text Tree, JSON | SQL Server GUI is vastly superior for visual analysis. Postgres text trees require experience to read. |
| **Optimizer Hints** | Extensive (`OPTION (HASH JOIN)`) | None natively (Requires `pg_hint_plan` extension) | Postgres forces DBAs to fix statistics rather than patching bad queries with hints. |
| **Historical Tuning** | Query Store (Native) | `pg_stat_statements` (Extension) | SQL Server Query Store is currently the industry gold standard for plan regression analysis. |

## 24. Architect recommendations

**The ORM Problem (Object-Relational Impedance Mismatch)**
Modern developers use ORMs (Entity Framework, Hibernate, Prisma). ORMs generate dynamic SQL. 
*   **Architectural Rule:** The Optimizer cannot optimize what it does not understand. ORMs frequently generate "N+1 query" problems (executing 1 query for a parent, and 1,000 separate queries for its children in a loop). 
*   **Solution:** As an Architect, you must mandate that developers profile the SQL generated by the ORM. If an ORM generates a 15-table join that takes 30 seconds, you must intercept that call, replace it with a highly tuned Stored Procedure or a Database View, and map the ORM to that object instead.

## 25. DBA recommendations

*   Do not trust the "Cost" percentages blindly. In SQL Server, a plan might say Node A is 90% of the cost, and Node B is 10%. However, if Node B is a scalar user-defined function (UDF), the Optimizer notoriously under-costs it. Node B might actually be the sole reason the query is slow. Always look at actual I/O and Time metrics, not just estimated costs.

## 26. Developer recommendations

*   **Avoid functions on the left side of the equals sign:** `WHERE UPPER(Email) = 'TEST@TEST.COM'`. This completely disables the Optimizer's ability to use a B-Tree index (Index Seek), forcing an Index Scan.
*   **Avoid wildcard prefixes:** `WHERE Name LIKE '%Smith'`. The B-Tree is sorted alphabetically. If you start with a wildcard, the Optimizer cannot navigate the tree.

## 27. Production case study

**The NextEvent VIP Ticket Outage (Parameter Sniffing)**

*Scenario:* The NextEvent platform relies on a stored procedure `GetTicketsByEventID(@EventID)`. 
At 9:00 AM, a user queried a small local comedy show (`@EventID = 5`). The show had 10 tickets. SQL Server compiled the plan, choosing a **Nested Loop Join**, which is extremely fast for 10 rows. The plan was cached.

At 10:00 AM, the Taylor Swift concert went on sale (`@EventID = 99`). This event had 100,000 tickets. Millions of users called the stored procedure. SQL Server reused the cached **Nested Loop Join** plan. 
Looping 100,000 times for millions of users caused CPU utilization to hit 100% instantly. The database locked up.

*Architectural Fix:* The DBA team identified the Parameter Sniffing issue using Query Store. They altered the stored procedure to include `OPTION (OPTIMIZE FOR UNKNOWN)`. This forced SQL Server to stop caching plans based on the first parameter provided, and instead use a statistically average plan (likely a **Hash Match Join**), which scales perfectly for both 10 rows and 100,000 rows. The CPU instantly dropped to 15%.

## 28. ASCII diagrams wherever helpful

**Execution Plan Tree (Bottom-Up vs Right-to-Left)**

```text
[ SQL SERVER (Read Right to Left) ]
SELECT (Cost 0%) 
  <-- Hash Match Inner Join (Cost 45%) 
      <-- Clustered Index Scan [Locations] (Cost 30%)
      <-- Index Seek [Events] (Cost 25%)

[ POSTGRESQL (Read Bottom-Up, Indented) ]
Hash Join  (cost=...
  Hash Cond: (e.location_id = l.location_id)
  ->  Index Scan on events e (cost=...
  ->  Hash  (cost=...
        ->  Seq Scan on locations l (cost=...
```
*Both represent the exact same mathematical path: Read the locations table, put it in a Hash memory structure, read the events index, and probe the hash table for matches.*

## 29. Enterprise design discussion

**Dynamic SQL vs. Stored Procedures**

*   **Stored Procedures:** Pre-compiled. SQL Server explicitly caches the execution plan based on the parameters passed during the first compilation. This provides immense performance benefits for OLTP but opens the door to Parameter Sniffing.
*   **Dynamic SQL (`sp_executesql`):** Often generated by ORMs or search screens with optional filters. Every unique string of Dynamic SQL generates a brand new execution plan, filling up the Plan Cache with single-use plans (Plan Cache Bloat). 

*Enterprise Standard:* For strict, high-volume transactional endpoints (Booking a Ticket), use Stored Procedures. For highly variable analytical search screens (Filter by Date, OR Filter by Name, OR Filter by Price), use parameterized Dynamic SQL to ensure the Optimizer calculates a fresh plan specifically suited for those specific filters.

## 30. Hands-on exercises

1. Open SSMS. Press `Ctrl + M` to enable Actual Execution Plans. Run a `SELECT *` on a large table. Review the graphical plan. Hover over the right-most node and compare "Estimated Number of Rows" to "Actual Number of Rows".
2. In pgAdmin or psql, run `EXPLAIN ANALYZE SELECT * FROM your_table;`. Identify the `Seq Scan` node and note the execution time.

## 31. Coding exercises

1. Write a complex SQL query that joins `Users`, `Tickets`, and `Events`.
2. Add a `WHERE` clause that forces the Optimizer to filter out 99% of the rows (e.g., `WHERE Price > 100000`). Run the Execution Plan. Note the Join type (likely Nested Loop).
3. Change the `WHERE` clause to return 99% of the rows (e.g., `WHERE Price > 0`). Run the Execution Plan. Note the Join type (likely Hash Match or Merge). Notice how the Optimizer dynamically changes the algorithm based on Cardinality Estimation.

## 32. Mini project

**Objective:** Optimize the NextEvent Analytics Query.
1. The Marketing team runs this query: 
   `SELECT e.Title, COUNT(t.TicketID) FROM core.events e JOIN core.tickets t ON e.event_id = t.event_id WHERE YEAR(t.purchase_date) = 2026 GROUP BY e.Title;`
2. Run the Execution Plan. Identify the non-SARGable bottleneck (the `YEAR` function).
3. Rewrite the query to be completely SARGable (using `>= '2026-01-01' AND < '2027-01-01'`).
4. Create the optimal Covering Index for this query.
5. Rerun the Execution Plan and prove that your optimization eliminated the Table Scan.

## 33. Quiz

1. What is the fundamental difference between declarative and procedural code in the context of databases?
2. What are the three primary physical join operators used by the Query Optimizer?
3. Explain what "Parameter Sniffing" is and why it causes performance instability.

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is an Execution Plan?
    *   **A:** An Execution Plan is the physical roadmap generated by the Query Optimizer. It details exactly which indexes, tables, and algorithms (like joins and sorts) the database engine will use to execute a declarative SQL query.
*   **Q:** What does `EXPLAIN` do in PostgreSQL?
    *   **A:** `EXPLAIN` shows the estimated execution plan calculated by the planner. `EXPLAIN ANALYZE` actually executes the query and shows both the estimates and the real-world execution metrics (time, rows, buffers).

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** You notice the database chose a "Nested Loop Join", but the query is taking 30 minutes to complete. What is likely wrong?
    *   **A:** Nested Loops are only efficient for very small datasets. If it's taking 30 minutes, it means the Optimizer drastically underestimated the number of rows (Cardinality Estimation error). This is almost always caused by outdated Statistics. Updating the statistics should allow the Optimizer to choose a Hash Match join instead.
*   **Q:** What is a Hash Match Join?
    *   **A:** A physical join algorithm where the database takes the smaller of the two tables, hashes the join keys, and builds a hash table in memory. It then scans the larger table, hashes its keys, and probes the memory table for matches. It is highly efficient for large, unsorted datasets but requires significant memory (`work_mem`).

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** In SQL Server, a specific Stored Procedure runs fast in SSMS for you, but the application team complains it constantly times out for them in production. What is the diagnosis and the cure?
    *   **A:** This is a classic symptom of Parameter Sniffing coupled with different connection `SET` options. When you run it in SSMS, your connection settings (like `ARITHABORT ON`) are different than the application's connection pool. SQL Server generates a separate plan cache entry for different `SET` options. Your SSMS compiled a fresh, fast plan based on your parameters. The application is stuck using a terrible cached plan compiled days ago with different parameters. The cure is to update statistics, or alter the SP to use local variables, `OPTION (RECOMPILE)`, or `OPTION (OPTIMIZE FOR UNKNOWN)`.
*   **Q:** In Postgres, you see `Sort Method: external merge Disk` in your `EXPLAIN ANALYZE` output. Why is this bad, and how do you fix it globally vs locally?
    *   **A:** This means the query required a sort (e.g., `ORDER BY`), but the dataset was too large to fit into the memory allocated by `work_mem`. The database had to spill the data to temporary files on the physical disk, which is orders of magnitude slower than RAM. You can fix it globally by altering `work_mem` in `postgresql.conf`, but that risks OOM errors across the server. The architectural best practice is to fix it locally by setting it just for that session: `SET work_mem = '256MB';` before running the specific heavy analytical query.

## 35. Chapter summary

### Learning Summary
We unraveled the mystery of the Query Optimizer, learning how it translates declarative SQL into physical execution plans. We explored the three primary join algorithms (Nested Loop, Merge, Hash) and how the Optimizer relies entirely on Cardinality Estimation (Statistics) to choose between them. We diagnosed the most notorious database issue—Parameter Sniffing—and learned how SQL Server's global plan cache differs from PostgreSQL's session-level caching. 

### Key Takeaways
*   The Optimizer uses mathematical statistics to estimate row counts and calculate physical I/O costs.
*   Nested Loops are for small data; Hash Matches are for large data.
*   Parameter Sniffing occurs when a cached plan, optimized for one specific value, is violently reused for a vastly different value.
*   SARGability is mandatory; functions on the left side of a `WHERE` clause blind the Optimizer.
*   Never blindly apply query hints or missing index recommendations without understanding the holistic architectural impact.

### Glossary
*   **Execution Plan:** The physical sequence of operations chosen by the Optimizer.
*   **Statistics:** Histograms describing the data distribution within a column.
*   **Cardinality Estimation:** The Optimizer's guess of how many rows an operation will return.
*   **Parameter Sniffing:** The act of caching an execution plan based on the parameters passed during first compilation.
*   **Query Store:** A SQL Server feature that records historical execution plans for regression analysis.

### Common Mistakes
*   Using Table Variables for large datasets in SQL Server (Optimizer guesses 1 row).
*   Forgetting to update Statistics after a massive bulk data import.
*   Ignoring the `BUFFERS` output in Postgres `EXPLAIN ANALYZE`.

### Best Practices
*   Write SARGable queries to ensure the Optimizer can utilize index seeks.
*   Rely on Temporary Tables (`#Temp`) rather than Table Variables for complex intermediate data processing.
*   Force application developers to review the Execution Plans generated by their ORMs.

### Further Reading
*   *SQL Server Execution Plans* by Grant Fritchey.
*   *The Art of PostgreSQL* by Dimitri Fontaine.
*   System R Optimizer Paper (Selinger, 1979).

### Preparation for Next Chapter
In Chapter 6, we will shift focus from internal architecture to advanced programming. We will master **Window Functions, CTEs (Common Table Expressions), and Advanced T-SQL / PL/pgSQL**. We will learn how to write complex analytical queries (running totals, ranking, moving averages) that execute natively in the database engine, outperforming application-tier processing by an order of magnitude.
