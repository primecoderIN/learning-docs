# Chapter 6: Sorting & Aggregation (GROUP BY, HAVING)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand the physical cost of sorting (`ORDER BY`) and pagination (`OFFSET FETCH`).
*   Master the `GROUP BY` clause to aggregate billing and telemetry data across our EV charging SaaS.
*   Differentiate between the `WHERE` clause and the `HAVING` clause in logical query processing.
*   Analyze the performance impact of TempDB spills during massive aggregations.
*   Translate complex grouping and pagination requirements into optimal EF Core LINQ queries.

---

## 6.1 Sorting with ORDER BY and Pagination

Retrieving data is only half the battle; presenting it to the user in a consumable format requires sorting and pagination.

### The ORDER BY Clause
`ORDER BY` is processed at the very end of the logical query lifecycle (Step 6). 
```sql
SELECT SessionId, StartTime, TotalKwh
FROM core.Sessions
WHERE TenantId = 'T1-UUID'
ORDER BY StartTime DESC;
```

**Architect Perspective: The Cost of Sorting**
Sorting is mathematically expensive ($O(N \log N)$ complexity). If you ask SQL Server to sort 5 million charging sessions, it will attempt to allocate a massive memory grant. If the server does not have enough RAM, the sort operation "spills" over into `TempDB` (the physical hard drive). A query that takes 10ms in RAM might take 5 seconds if it spills to disk.

To prevent this, you must build **Covering Indexes** that pre-sort the data on disk (e.g., an index on `TenantId, StartTime DESC`), allowing SQL Server to simply read the B-Tree in order without performing an explicit Sort operation in memory.

### Pagination (OFFSET FETCH)
No enterprise API returns 10,000 rows at once. We paginate.
```sql
SELECT SessionId, StartTime, TotalKwh
FROM core.Sessions
WHERE TenantId = 'T1-UUID'
ORDER BY StartTime DESC
OFFSET 50 ROWS        -- Skip the first 50 (e.g., Page 3)
FETCH NEXT 25 ROWS ONLY; -- Take the next 25
```
*Rule:* `OFFSET FETCH` strictly requires an `ORDER BY` clause. You cannot paginate an unordered set.

---

## 6.2 Introduction to Aggregation

SaaS dashboards rarely display raw rows; they display aggregates (KPIs, charts, totals).
SQL provides standard aggregate functions:
*   `SUM(TotalCost)`
*   `AVG(TotalKwh)`
*   `MIN(StartTime)` / `MAX(EndTime)`
*   `COUNT(*)` or `COUNT(DISTINCT UserId)`

```sql
-- Returns a single row with the totals for a specific tenant
SELECT 
    COUNT(*) AS TotalSessions,
    SUM(TotalCost) AS TotalRevenue,
    AVG(TotalKwh) AS AverageEnergyPerSession
FROM core.Sessions
WHERE TenantId = 'T1-UUID'
  AND StartTime >= '2024-01-01' AND StartTime < '2024-02-01';
```

---

## 6.3 The GROUP BY Clause

To break those aggregates down into categories (e.g., Revenue *per Station*), we use `GROUP BY`.

```sql
SELECT 
    StationId,
    COUNT(*) AS TotalSessions,
    SUM(TotalCost) AS TotalRevenue
FROM core.Sessions
WHERE TenantId = 'T1-UUID'
  AND StartTime >= '2024-01-01' AND StartTime < '2024-02-01'
GROUP BY StationId
ORDER BY TotalRevenue DESC;
```

### The Golden Rule of GROUP BY
**Any column in your `SELECT` list that is NOT inside an aggregate function MUST be included in the `GROUP BY` clause.**
If you attempt to `SELECT StationId, PortId, SUM(TotalCost) ... GROUP BY StationId`, SQL Server will throw an error. The engine does not know which `PortId` to display for the aggregated Station.

---

## 6.4 The HAVING Clause (Filtering Aggregates)

What if the SaaS Admin wants to see: *"Show me all Stations that generated more than $1,000 in Revenue this month."*

You cannot use the `WHERE` clause for this. Remember Logical Query Processing (Chapter 5): `WHERE` runs *before* `GROUP BY`. The `WHERE` clause filters raw rows, not aggregated totals.

To filter based on an aggregate, you must use the `HAVING` clause, which runs *after* `GROUP BY`.

```sql
SELECT 
    StationId,
    SUM(TotalCost) AS TotalRevenue
FROM core.Sessions
WHERE TenantId = 'T1-UUID'                     -- 1. Filter raw rows (Sargable)
GROUP BY StationId                             -- 2. Aggregate into buckets
HAVING SUM(TotalCost) > 1000.00                -- 3. Filter the buckets
ORDER BY TotalRevenue DESC;                    -- 4. Sort the result
```

---

## 6.5 The Code: EF Core Translations

How do we write these complex aggregations in C# using EF Core?

### Pagination (`Skip` and `Take`)
```csharp
// Generates ORDER BY ... OFFSET 50 ROWS FETCH NEXT 25 ROWS ONLY
var page = await context.Sessions
    .Where(s => s.TenantId == tenantId)
    .OrderByDescending(s => s.StartTime)
    .Skip(50)
    .Take(25)
    .ToListAsync();
```

### Grouping and Aggregation (`GroupBy`)
```csharp
// Generates SELECT StationId, SUM(TotalCost) ... GROUP BY StationId HAVING SUM(TotalCost) > 1000
var topStations = await context.Sessions
    .Where(s => s.TenantId == tenantId)
    .GroupBy(s => s.Port.StationId)
    .Select(group => new 
    {
        StationId = group.Key,
        TotalRevenue = group.Sum(s => s.TotalCost)
    })
    .Where(result => result.TotalRevenue > 1000.00m) // EF translates this to HAVING
    .OrderByDescending(result => result.TotalRevenue)
    .ToListAsync();
```
*Architect Note:* EF Core is exceptionally smart here. The `.Where()` placed *after* the `.Select()` containing the aggregation is correctly translated into a SQL `HAVING` clause.

---

## 6.6 The Code: Offset vs Keyset (Cursor) Pagination (EF Core & Dapper)

While `OFFSET/FETCH` is easy to write, as we discussed, it degrades massively on deep pages. The enterprise standard is **Keyset (Cursor) Pagination**, where you use the last viewed item's unique, sorted value as an anchor.

### EF Core: Offset vs Keyset

**Bad (Offset):**
```csharp
// Fetches page 10,000. SQL Server reads 500,000 rows, discards 499,950.
var offsetPage = await context.Sessions
    .OrderByDescending(s => s.StartTime)
    .Skip(499950)
    .Take(50)
    .ToListAsync();
```

**Good (Keyset/Cursor):**
```csharp
// Fetches the exact 50 rows instantly using a B-Tree index seek.
// Assuming the client passed `cursorTime` from the last record of the previous page.
var keysetPage = await context.Sessions
    .Where(s => s.StartTime < cursorTime)
    .OrderByDescending(s => s.StartTime)
    .Take(50)
    .ToListAsync();
```

### Dapper: Offset vs Keyset

Dapper requires you to write the raw SQL, which gives you complete control over the execution plan.

**Bad (Offset):**
```csharp
string sql = @"
    SELECT SessionId, StartTime, TotalCost 
    FROM core.Sessions 
    ORDER BY StartTime DESC 
    OFFSET @Offset ROWS FETCH NEXT @Take ROWS ONLY;";

var sessions = await db.QueryAsync<SessionDto>(sql, new { Offset = 499950, Take = 50 });
```

**Good (Keyset/Cursor):**
```csharp
string sql = @"
    SELECT TOP (@Take) SessionId, StartTime, TotalCost 
    FROM core.Sessions 
    WHERE StartTime < @CursorTime 
    ORDER BY StartTime DESC;";

var sessions = await db.QueryAsync<SessionDto>(sql, new { CursorTime = lastSeenTime, Take = 50 });
```
*(Architect Note: Always ensure the column used for the Cursor—in this case `StartTime`—is indexed and highly unique. If you have thousands of duplicate timestamps, you must use a tie-breaker column like `SessionId` in both the `WHERE` and `ORDER BY` clauses to ensure deterministic sorting).*

---

## 6.7 Performance & Security Analysis

### Performance Analysis: The Sort/Hash Warning
When SQL Server executes a `GROUP BY`, it typically uses a **Hash Match Aggregate** or a **Stream Aggregate**.
*   **Stream Aggregate:** Extremely fast, but requires the data to already be sorted by the Grouping column.
*   **Hash Match Aggregate:** Builds a hash table in memory. If memory is insufficient, it spills to TempDB.
If you open an Execution Plan and see a yellow warning triangle over a "Sort" or "Hash Match" operator, it indicates a TempDB spill. You fix this by creating an index that covers the grouping and sorting columns.

### Security Implications
*   **Denial of Service (DoS) via Uncapped Pagination:** If you expose an API `GET /sessions?skip=0&take=1000000`, a malicious user can force your database to allocate massive memory grants to sort 1 million rows, bringing down the DB for other tenants. **Always enforce a maximum `take` (e.g., `take = Math.Min(request.Take, 100)`) at the API layer.**

---

## 6.8 Common Mistakes & Production Pitfalls

1.  **Deep Pagination (The OFFSET Trap):** As `OFFSET` grows, performance degrades. `OFFSET 100000 FETCH NEXT 50` requires the database to internally read and sort 100,050 rows, discard the first 100,000, and return 50. For deep pagination, implement **Keyset Pagination** (e.g., `WHERE StartTime < LastSeenStartTime ORDER BY StartTime DESC FETCH NEXT 50`).
2.  **Counting Rows Inefficiently:** Running `SELECT COUNT(*)` on a 50-million row table is slow because it physically counts the rows. If you only need an *approximate* count for a dashboard, query the Dynamic Management Views (DMVs) which store row counts as metadata.

---

## 6.9 Production Checklist

*   [ ] API endpoints enforcing pagination have a hardcoded upper limit on page size.
*   [ ] Deep pagination use cases (like endless scrolling feeds) use Keyset Pagination instead of `OFFSET/FETCH`.
*   [ ] Covering indexes are created for frequent `GROUP BY` and `ORDER BY` operations to prevent TempDB spills.
*   [ ] `WHERE` is used to filter raw rows *before* aggregation, and `HAVING` is used strictly for filtering the aggregated totals.

---

## 6.10 Exercises

1.  **Query Translation:** A business analyst asks: *"For Tenant 'T1', how many total charging sessions occurred on each Port, but only show Ports that have had more than 50 sessions?"*
    Write the T-SQL query using `WHERE`, `GROUP BY`, and `HAVING`.
2.  **Keyset Pagination:** Write a T-SQL query that fetches the next 25 Sessions (ordered by `StartTime DESC`) where the `StartTime` is older than `2024-10-15 14:00:00.000` (the timestamp of the last record on the previous page).

---

## 6.11 Interview Questions

**Q1: Explain the difference between `WHERE` and `HAVING`. Can you use them both in the same query?**
*Answer:* Yes, they are frequently used together. The `WHERE` clause filters individual rows *before* they are grouped and aggregated (Step 2 of Logical Query Processing). The `HAVING` clause filters the aggregated results *after* they are grouped (Step 4). For example, you use `WHERE` to limit data to the current year, and `HAVING` to return only groups that generated more than $1000.

**Q2: Why does pagination using `OFFSET 500000 FETCH NEXT 50` become extremely slow, and how do you fix it architecturally?**
*Answer:* `OFFSET` still requires the database engine to locate and sort all 500,050 rows before discarding the first 500,000. It is a linear time degradation. To fix this, architects use Keyset Pagination (or Seek Pagination), where the client passes the last value of the sorted column (e.g., `LastSessionId` or `LastTimestamp`), and the query is modified to `WHERE Timestamp < LastTimestamp ORDER BY Timestamp DESC FETCH NEXT 50`. This allows the engine to jump directly to the exact B-Tree node, providing $O(1)$ pagination performance.

---
**Next up in Chapter 7:** We will unlock the true power of Relational Databases: Joins and Subqueries. We will reconstruct our Tenant -> Station -> Port hierarchy and discuss the dangers of Correlated Subqueries.
