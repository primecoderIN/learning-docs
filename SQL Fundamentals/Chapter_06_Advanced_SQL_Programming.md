# Chapter 6 – Window Functions, CTEs, and Advanced SQL Programming

## 1. Concept Overview

While `SELECT`, `JOIN`, and `WHERE` form the foundation of SQL, enterprise applications require complex analytical processing—calculating running totals, moving averages, hierarchal traversal, and deduplication. Historically, developers resorted to extracting raw data into the application layer to process it with procedural loops. 

Advanced SQL allows us to push this logic down to the database engine using two primary concepts:
1.  **Common Table Expressions (CTEs):** A temporary named result set that you can reference within a `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement. They drastically improve query readability and enable recursion.
2.  **Window Functions:** These perform calculations across a set of table rows that are somehow related to the current row. Unlike a standard aggregate function (`GROUP BY`), which collapses multiple rows into a single output row, a Window Function calculates an aggregate value *while retaining the original individual rows*.

## 2. History

In the 1990s, complex analytical queries in SQL were notoriously difficult to write. If you wanted a running total, you had to use a highly inefficient self-join or a procedural cursor. 
*   The **SQL:1999** standard introduced CTEs (including recursive CTEs) to solve hierarchical data traversal (e.g., employee-manager org charts).
*   The **SQL:2003** standard introduced Window Functions, representing the biggest leap forward in SQL's analytical capability. It allowed SQL to compete directly with Excel's ability to easily calculate moving averages and ranks.

## 3. Real-world analogy

*   **CTE Analogy:** Imagine solving a massive algebraic equation. A CTE is like defining a variable `x = (A + B)` at the top of the page, so later you can simply write `x * C` instead of writing out the complex formula every time.
*   **Window Function Analogy:** Imagine a spreadsheet showing monthly sales for an Event Venue. If you use a `GROUP BY`, you squash the 12 months down into 1 line: "Total Sales: $1M". If you use a **Window Function**, you keep all 12 lines, but you add a new column at the end: "Running Total." On January it says $50k. On February it says $120k. The window of calculation "slides" down the rows.

## 4. Business problem solved

*   **Hierarchical Data:** Storing a multi-level Event Category tree (Music -> Rock -> Alternative) in a single table and querying the entire path in one go. (Solved by Recursive CTEs).
*   **Deduplication:** Finding the 5 most recent ticket purchases *per user* without using a loop. (Solved by `ROW_NUMBER() OVER(PARTITION BY...)`).
*   **Time-Series Analysis:** Comparing a ticket's price to the ticket sold immediately prior to it. (Solved by `LAG()`).

---

## 5. Microsoft SQL Server explanation

SQL Server fully supports the ANSI SQL standards for CTEs and Window Functions. 
In T-SQL, a CTE is purely a logical construct—it is essentially an inline View. It does *not* create a temporary table in memory, nor does it store the result set physically (a common misconception). 

For Window Functions, SQL Server uses the `OVER()` clause. Inside the `OVER()` clause, you can define:
1.  `PARTITION BY`: Divides the result set into buckets (like a `GROUP BY`).
2.  `ORDER BY`: Determines the logical order of rows within the bucket.
3.  `ROWS BETWEEN`: Defines the physical frame of rows to include (e.g., "1 row preceding and current row").

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Common Table Expression (CTE)
WITH VIP_Users AS (
    SELECT UserID, Email 
    FROM Core.Users 
    WHERE TotalSpent > 10000
)
SELECT t.TicketID, v.Email 
FROM Core.Tickets t
INNER JOIN VIP_Users v ON t.UserID = v.UserID;
GO

-- 2. Window Function: Row Number & Running Total
SELECT 
    EventID,
    PurchaseDate,
    Price,
    -- Rank tickets chronologically PER EVENT
    ROW_NUMBER() OVER(PARTITION BY EventID ORDER BY PurchaseDate ASC) AS TicketSequence,
    -- Calculate running total of revenue PER EVENT
    SUM(Price) OVER(PARTITION BY EventID ORDER BY PurchaseDate ASC) AS RunningRevenue
FROM Core.Tickets;
GO

-- 3. Recursive CTE (Generating an Org Chart / Hierarchy)
WITH CategoryHierarchy AS (
    -- Anchor Member (The Root)
    SELECT CategoryID, Name, ParentCategoryID, 0 AS Level
    FROM Core.Categories WHERE ParentCategoryID IS NULL
    
    UNION ALL
    
    -- Recursive Member (Joins back to the CTE itself)
    SELECT c.CategoryID, c.Name, c.ParentCategoryID, ch.Level + 1
    FROM Core.Categories c
    INNER JOIN CategoryHierarchy ch ON c.ParentCategoryID = ch.CategoryID
)
SELECT * FROM CategoryHierarchy;
GO
```

## 7. SQL Server internals

When SQL Server processes a Window Function, it relies on a specific execution plan operator called the **Window Spool**. 
Because a Window Function requires calculating data based on surrounding rows (e.g., a moving average), the Optimizer must read a row, store it in a hidden temporary workspace (`TempDB`), read the next row, compare them, and output the result. 

If the data is not indexed according to the `PARTITION BY` and `ORDER BY` clauses of the Window Function, SQL Server must insert an extremely expensive **Sort** operator into the execution plan *before* the Window Spool.

## 8. SQL Server execution

Executing a query with `ROW_NUMBER() OVER(PARTITION BY EventID ORDER BY PurchaseDate)`:
1. The Optimizer checks if an index exists sorted by `EventID, PurchaseDate`.
2. If yes, it does an Index Seek.
3. The data flows into a **Segment Operator** (which groups the rows by `EventID`).
4. The data flows into a **Sequence Project Operator** (which assigns the 1, 2, 3 integers).
5. If no index exists, a massive **Sort Operator** is placed at step 2, destroying performance for millions of rows.

## 9. SQL Server enterprise examples

*   **Pagination:** Before SQL Server 2012 introduced `OFFSET...FETCH`, enterprise applications exclusively used `ROW_NUMBER()` wrapped in a CTE to paginate grid views (e.g., "Show me rows 51 through 100").
*   **Data Cleansing:** Eradicating duplicate rows in massive ETL loads. By partitioning by the unique business key and ordering by the insert date, developers simply write `DELETE FROM CTE WHERE RowNum > 1`.

## 10. SQL Server performance considerations

*   **The CTE Multiple Execution Trap:** Because a CTE is just a logical macro, if you reference the same CTE three times in your main query, SQL Server will literally execute the CTE logic three separate times. If the CTE takes 5 seconds to run, your query now takes 15 seconds. In this scenario, you must dump the CTE data into a `#TempTable` instead.
*   **Infinite Recursion:** A poorly written Recursive CTE can loop forever. SQL Server has a built-in safety net: `MAXRECURSION 100`. It will crash the query if it loops 101 times unless explicitly overridden with `OPTION (MAXRECURSION 0)`.

## 11. SQL Server security considerations

*   Advanced SQL constructs do not bypass Row-Level Security (RLS) or column encryption. The security context is evaluated at the base table level before the CTE or Window Function logic is applied.

## 12. SQL Server common mistakes

*   **Using CTEs for Performance:** Believing that wrapping a slow query in a CTE will somehow magically make it faster. It won't. It is syntactical sugar for an inline subquery.
*   **Omitting the Framing Clause:** When you write `SUM(Price) OVER(PARTITION BY EventID ORDER BY Date)`, SQL Server silently defaults the frame to `RANGE UNBOUNDED PRECEDING`. The `RANGE` keyword is notoriously slow in SQL Server because it creates an on-disk spool. Changing it to `ROWS UNBOUNDED PRECEDING` provides an instant massive performance boost.

## 13. SQL Server best practices

*   Always prefer Window Functions over Self-Joins.
*   Always use `ROWS` instead of `RANGE` in SQL Server window frames when calculating running totals.
*   Provide a perfectly aligned Non-Clustered Index to support the `PARTITION BY` and `ORDER BY` of your most common window functions to eliminate the Sort operator.

---

## 14. PostgreSQL explanation

PostgreSQL's implementation of Window Functions and CTEs is robust and highly standards-compliant. 

A major historical distinction in Postgres was how it handled CTEs. Prior to PostgreSQL 12, CTEs in Postgres were an "Optimization Fence." Postgres would **always materialise** the CTE (run it once, save it in a hidden temporary table, and query the temp table). While great for reusing a CTE multiple times, it prevented the Optimizer from pushing `WHERE` clauses down into the CTE. Since version 12, Postgres behaves like SQL Server (inlining CTEs) by default, but allows you to force materialization using the `MATERIALIZED` keyword.

## 15. PostgreSQL syntax

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. CTE with explicitly forced materialization (PG 12+)
WITH massive_aggregation AS MATERIALIZED (
    SELECT event_id, SUM(price) as total_rev
    FROM core.tickets
    GROUP BY event_id
)
SELECT * FROM massive_aggregation WHERE total_rev > 100000;

-- 2. Window Function: LAG (Compare to previous row)
SELECT 
    ticket_id,
    purchase_date,
    price,
    LAG(price, 1) OVER(PARTITION BY event_id ORDER BY purchase_date) as previous_ticket_price,
    price - LAG(price, 1) OVER(PARTITION BY event_id ORDER BY purchase_date) as price_difference
FROM core.tickets;

-- 3. The FILTER clause (Postgres specific, highly superior to CASE statements)
SELECT 
    event_id,
    COUNT(ticket_id) AS total_tickets,
    COUNT(ticket_id) FILTER (WHERE status = 'VIP') AS vip_tickets
FROM core.tickets
GROUP BY event_id;
```

## 16. PostgreSQL internals

Postgres executes Window Functions using a **WindowAgg** node in the execution plan. It requires the data to be pre-sorted. If the data isn't sorted via an index, Postgres inserts a **Sort** node.

If `work_mem` is not large enough to hold the partitioned bucket in memory during the WindowAgg operation, Postgres will spill the operation to disk (Temp Files), severely degrading performance.

## 17. PostgreSQL execution

Executing a query with `LAG()`:
1. The Planner identifies the `WindowAgg` requirement.
2. It executes an Index Scan (if available) or a Seq Scan followed by a Sort.
3. The Executor steps through the rows. Because `LAG(1)` requires looking 1 row backward, the executor maintains a tuplestore (a small memory buffer) containing the current and previous rows within the current partition.
4. When the partition key changes (e.g., moving to the next `event_id`), the tuplestore is flushed, and the calculation resets.

## 18. PostgreSQL enterprise examples

*   **IoT and Sensor Data:** Systems ingesting millions of temperature readings use Postgres Window Functions to calculate moving averages over 5-minute rolling windows (`ROWS BETWEEN 5 PRECEDING AND CURRENT ROW`) to smooth out graph visualizations instantly at the database layer.
*   **Time-Series Gap Filling:** Using recursive CTEs (`generate_series()` in Postgres) to generate a calendar of dates, and `LEFT JOIN`ing actual sales data to ensure reports show $0 for days with no sales, rather than skipping the day entirely.

## 19. PostgreSQL performance considerations

*   **CTEs as Optimization Fences (Pre-PG12):** If you are on an older version of Postgres (or use `MATERIALIZED`), be highly aware that if your CTE returns 10 million rows, and your outer query says `LIMIT 1`, Postgres will still calculate and materialize all 10 million rows first. Inlining allows the planner to apply the `LIMIT 1` *inside* the CTE.
*   **Work_mem for Sorts:** Window functions inherently rely on sorting. Ensure your `work_mem` is appropriately tuned for analytical sessions.

## 20. PostgreSQL security considerations

*   When using CTEs to perform DML (`INSERT/UPDATE/DELETE ... RETURNING`), be aware of the security context. You can use a CTE to `DELETE` a row, and `RETURNING` to capture the deleted data and `INSERT` it into an audit table, all within a single atomic statement.

## 21. PostgreSQL common mistakes

*   **Procedural Loops:** Developers accustomed to application code often write PL/pgSQL `FOR...LOOP` constructs to calculate running totals row-by-row. This is a severe anti-pattern in Postgres. A Window Function will execute the same logic 100x to 1000x faster by utilizing C-based set operations instead of pl/pgsql context switching.

## 22. PostgreSQL best practices

*   Use the `FILTER` clause. It is an ANSI SQL standard that Postgres supports natively. It replaces the ugly and slow `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` syntax used in other databases, producing much cleaner execution plans.
*   Use `generate_series()` for any recursive date/number generation. It is a highly optimized C function that vastly outperforms recursive CTEs for simple series generation.

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **CTE Execution** | Always inlined (Logical view) | Inlined (v12+), optionally `MATERIALIZED` | Postgres offers more control. If a CTE is incredibly expensive and reused 5 times in a query, `MATERIALIZED` in Postgres is a lifesaver. In SQL Server, you must use `#Temp` tables. |
| **Window Frame Default** | `RANGE` (Slow on-disk spool) | `RANGE` (Standard) | Both default to `RANGE`. DBAs in both systems must explicitly use `ROWS` for high-performance running totals. |
| **Conditional Aggregates**| Requires `CASE WHEN` | Native `FILTER (WHERE...)` | Postgres `FILTER` is vastly superior for readability and execution speed. |
| **Recursive Safety** | `MAXRECURSION` setting | No native max depth setting | Postgres can easily infinite-loop if the recursive join condition is flawed. |

## 24. Architect recommendations

**Set-Based vs. Procedural Architecture**
The defining characteristic of a Database Architect is recognizing when logic belongs in the application (procedural) vs the database (set-based).
*   **Anti-Pattern:** The application pulls 100,000 rows into RAM. It uses a `for` loop to calculate the difference in time between User A's first ticket purchase and second ticket purchase, then pushes the results back to the database. (Massive network payload, high application memory, terrible latency).
*   **Architectural Standard:** The database executes a single `LEAD()` window function, calculating the difference natively in C/C++ inside the database engine memory, returning exactly the 5 rows requested. 
*   **Rule:** If a calculation requires comparing rows within a dataset to each other, use a Window Function.

## 25. DBA recommendations

*   When developers submit code with Window Functions, immediately check the execution plan for the **Sort** node. If the Sort accounts for 95% of the query cost, mandate the creation of a covering index matching the `PARTITION BY` and `ORDER BY` columns.

## 26. Developer recommendations

*   Understand `ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()`.
    *   `ROW_NUMBER()`: 1, 2, 3, 4 (Always sequential, breaks ties arbitrarily).
    *   `RANK()`: 1, 2, 2, 4 (Ties get the same number, next number is skipped).
    *   `DENSE_RANK()`: 1, 2, 2, 3 (Ties get the same number, next number is NOT skipped).

## 27. Production case study

**The NextEvent Gamification Leaderboard**

*Scenario:* NextEvent introduced a feature to reward the top 10 ticket buyers of the year. The initial implementation used a database Cursor. It selected every user, looped through all their tickets, calculated their total spend, ordered them, and updated a leaderboard table. This nightly job took 4.5 hours to run, causing severe blocking during maintenance windows.

*Architectural Fix:* We discarded the Cursor entirely and rewrote it using a CTE and a Window Function.
```sql
WITH RankedSpenders AS (
    SELECT 
        UserID, 
        SUM(Price) as TotalSpent,
        DENSE_RANK() OVER(ORDER BY SUM(Price) DESC) as LeaderboardRank
    FROM core.tickets
    WHERE YEAR(PurchaseDate) = 2026
    GROUP BY UserID
)
SELECT * FROM RankedSpenders WHERE LeaderboardRank <= 10;
```
*Result:* The job execution time dropped from **4.5 hours** to **1.2 seconds**. Set-based operations leverage the Optimizer and modern multi-core CPU parallel execution; cursors force single-threaded, row-by-row agony.

## 28. ASCII diagrams wherever helpful

**Understanding the Window Frame (`OVER`)**

```text
Dataset: [10, 20, 30, 40, 50]

Window Function: SUM(Value) OVER (ORDER BY Date ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)

Execution Engine View:
Row 1 (10): Frame = [NULL, 10] -> Result: 10
Row 2 (20): Frame = [10, 20]   -> Result: 30
Row 3 (30): Frame = [20, 30]   -> Result: 50
Row 4 (40): Frame = [30, 40]   -> Result: 70
Row 5 (50): Frame = [40, 50]   -> Result: 90

Output Dataset: 
Values:  [10, 20, 30, 40, 50]
Running: [10, 30, 50, 70, 90] 
(Notice how the frame "slides" down the dataset without collapsing the rows).
```

## 29. Enterprise design discussion

**The "Top N per Group" Problem**
This is the most common interview and architectural pattern.
"Get the most recent ticket purchased by each user."

*   *The Novice approach:* `GROUP BY UserID, MAX(PurchaseDate)`. (Fails because you can't easily retrieve the *TicketID* associated with that max date without a horrible self-join).
*   *The Architect approach:* 
    ```sql
    WITH RankedTickets AS (
        SELECT TicketID, UserID, PurchaseDate,
        ROW_NUMBER() OVER(PARTITION BY UserID ORDER BY PurchaseDate DESC) as rn
        FROM Tickets
    )
    SELECT * FROM RankedTickets WHERE rn = 1;
    ```
This elegantly partitions the data by user, sorts it newest-first, assigns a '1' to the newest ticket, and filters out the rest. It requires a single pass over the data.

## 30. Hands-on exercises

1. Write a standard `GROUP BY` query to get the total revenue per Event. Note the number of rows returned.
2. Write a query using `SUM(Price) OVER(PARTITION BY EventID)` to get the total revenue per Event. Note the number of rows returned. Observe how the Window Function retains the base Ticket rows.

## 31. Coding exercises

1. Write a query using `LAG()` to find the time difference (in minutes) between ticket purchases for a specific event. This shows how fast the event is selling out.
2. Write a Recursive CTE that generates numbers from 1 to 100. (In SQL Server, rely on the implicit `MAXRECURSION`; in Postgres, compare this to `SELECT * FROM generate_series(1, 100)`).
3. Use a CTE to delete duplicate rows from a table. Assume you have a table where `Email` was accidentally inserted twice, but they have different auto-incrementing IDs. Keep the row with the lowest ID.

## 32. Mini project

**Objective:** The Monthly Revenue Report.
The Finance team needs a view in the NextEvent system that displays:
1. The Month (e.g., '2026-01').
2. The total revenue for that month.
3. The total revenue of the *previous* month.
4. The percentage growth/decline compared to the previous month.
5. The Year-to-Date (YTD) running total revenue.

*Hint: Use a CTE to aggregate the data by month first. Then use `LAG()` for the previous month, and `SUM() OVER(ORDER BY)` for the YTD running total.*

## 33. Quiz

1. What is the fundamental difference between `GROUP BY` and an `OVER()` Window Function?
2. Why must you be careful when referencing a single CTE multiple times in a SQL Server query?
3. What is the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` when processing a tie?

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is a CTE and why use it?
    *   **A:** A Common Table Expression is a temporary, named result set defined via the `WITH` clause. We use it to break down complex queries into readable chunks, to prevent nested subqueries (spaghetti code), and to enable recursive queries.
*   **Q:** How do you find the second highest ticket price using a Window Function?
    *   **A:** Wrap the query in a CTE, apply `DENSE_RANK() OVER(ORDER BY Price DESC) as rnk`, and then filter the outer query `WHERE rnk = 2`.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** Explain what `PARTITION BY` does inside a Window Function.
    *   **A:** It divides the result set into boundaries (buckets). When the window function calculates its value (like a running total or a row number), the calculation resets to zero/start whenever the `PARTITION BY` column value changes.
*   **Q:** You wrote a window function using `ORDER BY`. The query is incredibly slow. What execution plan operator is likely causing it, and how do you fix it?
    *   **A:** The `Sort` operator is causing it. Window functions require ordered data. If no index exists, the engine must sort the entire dataset in memory or on disk. To fix it, create a covering index on the `PARTITION BY` and `ORDER BY` columns.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** In SQL Server, a developer writes `SUM(Amount) OVER (PARTITION BY DepartmentID ORDER BY SaleDate)`. The query runs 10x slower than expected. You change nothing but add `ROWS UNBOUNDED PRECEDING` to the `OVER` clause, and it finishes instantly. Why?
    *   **A:** When `ORDER BY` is specified in an `OVER` clause without an explicit framing clause, SQL Server defaults to `RANGE UNBOUNDED PRECEDING`. The `RANGE` logic handles duplicate order values by analyzing the entire logical frame, which SQL Server implements physically by dumping the rows into an on-disk Spool in TempDB. By explicitly specifying `ROWS UNBOUNDED PRECEDING`, you instruct the engine to use a physical, row-by-row, in-memory counter, bypassing the disk spool entirely and resulting in massive performance gains.

## 35. Chapter summary

### Learning Summary
We bridged the gap between basic data retrieval and advanced analytical processing. We utilized CTEs to modularize complex SQL and navigate hierarchical data via recursion. We unlocked the immense power of Window Functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, `SUM OVER`) to perform cross-row calculations without collapsing datasets. We established that set-based SQL processing is inherently superior to application-layer procedural loops.

### Key Takeaways
*   Window functions calculate aggregates across rows related to the current row without losing the base row context.
*   CTEs are logical constructs (inline views) in SQL Server, meaning multiple references execute multiple times. Postgres 12+ allows manual `MATERIALIZED` control.
*   Never use a Cursor or `WHILE` loop when a Window Function can achieve the same result.
*   Always provide explicit `ROWS` framing in SQL Server to avoid the `RANGE` spool penalty.
*   The "Top N per Group" problem is elegantly solved using `ROW_NUMBER() OVER(PARTITION BY...)`.

### Glossary
*   **CTE:** Common Table Expression. Defined with `WITH`.
*   **Window Function:** A function operating over a frame of rows, defined by `OVER()`.
*   **Partition:** The logical grouping/bucket within a window function.
*   **Recursive CTE:** A CTE that contains a `UNION ALL` referencing itself.
*   **Cursor:** A procedural method to iterate through a dataset row-by-row (Anti-pattern).

### Common Mistakes
*   Using `GROUP BY` when you actually needed to retain the base rows.
*   Using Cursors for running totals.
*   Forgetting the performance difference between `RANGE` and `ROWS` in SQL Server.

### Best Practices
*   Use CTEs to make massive queries readable. 
*   Replace complex self-joins used for chronological comparisons with `LAG()` and `LEAD()`.
*   Index your Window Functions (`PARTITION BY` columns first, then `ORDER BY` columns).

### Further Reading
*   *T-SQL Window Functions* by Itzik Ben-Gan (The definitive guide).
*   PostgreSQL Documentation: *Window Functions Tutorial*.
*   *SQL for Smarties* by Joe Celko.

### Preparation for Next Chapter
In Chapter 7, we will explore **Advanced Storage: Partitioning, Sharding, and Distributed Databases**. We will learn how to architect systems that have grown beyond a single disk or a single server, exploring horizontal scaling, table partitioning by date, and how massive platforms like Uber and Netflix shard their databases to achieve infinite scale.
