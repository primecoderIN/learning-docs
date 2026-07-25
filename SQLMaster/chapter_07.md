# Chapter 7: Relational Joins & Subqueries

## Learning Objectives
By the end of this chapter, you will be able to:
*   Master the 5 types of Relational Joins (`INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`).
*   Reconstruct the complex domain hierarchy (Tenant -> Station -> Port -> Session) in a single highly-optimized query.
*   Differentiate between Uncorrelated and Correlated subqueries, and understand why the latter causes massive performance degradation.
*   Configure EF Core to avoid the "Cartesian Explosion" problem using Split Queries.

---

## 7.1 Reconstructing the Hierarchy

Because we normalized our database (storing Stations, Ports, and Sessions in separate tables), we eliminated data duplication. However, to display a dashboard showing a Station's name alongside its recent Sessions, we must stitch those tables back together at query time. We do this using **Joins**.

---

## 7.2 The INNER JOIN

The `INNER JOIN` returns only the rows that have matching values in *both* tables. It is the logical intersection.

**Requirement:** Display all Charging Sessions, but include the `PortNumber` they occurred on.

```sql
SELECT 
    s.SessionId, 
    s.TotalCost, 
    p.PortNumber
FROM core.Sessions s
INNER JOIN core.Ports p 
    ON s.PortId = p.PortId
WHERE s.TenantId = 'T1-UUID';
```
*Note:* If a Port exists but has *never* had a Session, that Port will **not** appear in these results.

---

## 7.3 Outer Joins (LEFT, RIGHT, FULL)

Outer joins do not require a match in both tables. 

### The LEFT JOIN
A `LEFT JOIN` returns *all* rows from the left table, and the matched rows from the right table. If there is no match, the right side returns `NULL`.

**Requirement:** The Admin wants a list of *all* Stations, and if they have any Ports, show the Ports. If they don't have Ports installed yet, still show the Station.

```sql
SELECT 
    st.Name AS StationName,
    p.PortNumber
FROM core.Stations st
LEFT JOIN core.Ports p 
    ON st.StationId = p.StationId
WHERE st.TenantId = 'T1-UUID';
```
*Result:* A new Station with zero ports will return `StationName = 'New Lobby', PortNumber = NULL`.

### RIGHT JOIN and FULL OUTER JOIN
*   `RIGHT JOIN` is exactly the same as `LEFT JOIN`, just reading from right to left. (Architects rarely use it; just swap the table order in a `LEFT JOIN` for readability).
*   `FULL OUTER JOIN` returns all rows from *both* tables, padding with `NULL`s where there is no match.

---

## 7.4 The CROSS JOIN

A `CROSS JOIN` produces the **Cartesian Product** of two tables. It pairs every row in Table A with every row in Table B.
If Table A has 1,000 rows and Table B has 1,000 rows, a Cross Join produces 1,000,000 rows.

**Architect Perspective:** Cross Joins are rarely used in UI queries, but they are incredibly powerful for generating test data or building Calendar/Date tables for financial reporting.

---

## 7.5 Subqueries (Uncorrelated)

A subquery is a query nested inside another query. 

An **Uncorrelated Subquery** is independent. It can be executed completely on its own, returning a result set that the outer query uses.

**Requirement:** Find all Sessions that consumed more energy than the overall SaaS average.

```sql
-- Step 1: SQL executes the inner query ONCE. (e.g., returns 25.5 kWh)
-- Step 2: SQL substitutes 25.5 into the outer query.
SELECT SessionId, TotalKwh
FROM core.Sessions
WHERE TotalKwh > (
    SELECT AVG(TotalKwh) FROM core.Sessions
);
```
Uncorrelated subqueries are generally highly optimized by the SQL engine.

---

## 7.6 Correlated Subqueries (The Anti-Pattern)

A **Correlated Subquery** references a column from the outer query. Therefore, it *cannot* be evaluated independently.

**Requirement:** For every Station, find its *most recent* Session date.

```sql
-- THE ANTI-PATTERN
SELECT 
    st.StationId,
    (
        SELECT MAX(StartTime) 
        FROM core.Sessions s 
        WHERE s.PortId IN (SELECT PortId FROM core.Ports p WHERE p.StationId = st.StationId)
    ) AS LastSessionDate
FROM core.Stations st;
```

### The RBAR Problem (Row-By-Agonizing-Row)
Because the inner query relies on `st.StationId`, SQL Server cannot evaluate it once. If you have 10,000 Stations, SQL Server must execute the inner query **10,000 individual times**. This is called RBAR processing, and it will crush your CPU.

**The Fix:** Always rewrite correlated subqueries as `JOIN`s or use Window Functions (covered in Chapter 10).

---

## 7.7 The Code: EF Core Translations

How do we query hierarchies in EF Core?

### Navigation Properties and `.Include()`
```csharp
// Generates a massive LEFT JOIN query
var stationsWithPorts = await context.Stations
    .Include(s => s.Ports)
    .Where(s => s.TenantId == tenantId)
    .ToListAsync();
```

### The Cartesian Explosion Problem
If you include multiple collections (e.g., a Station has many Ports, and a Station has many MaintenanceLogs), EF Core generates a massive SQL query joining everything. The database returns a flat table. If a Station has 2 Ports and 10 Logs, SQL Server returns 20 rows of duplicated Station data. If it has 10 Ports and 100 Logs, it returns 1,000 rows. This network payload will crash your API.

### The Solution: `.AsSplitQuery()`
In EF Core, you must instruct the engine to split the query into multiple smaller SQL statements when loading multiple collections.

```csharp
var stations = await context.Stations
    .Include(s => s.Ports)
    .Include(s => s.MaintenanceLogs)
    .Where(s => s.TenantId == tenantId)
    .AsSplitQuery() // CRITICAL for enterprise performance
    .ToListAsync();
```
EF Core will execute 3 separate, highly efficient queries (one for Stations, one for Ports, one for Logs) and stitch them together in C# memory.

---

## 7.8 Performance & Security Analysis

### Performance Analysis: Join Algorithms
When SQL Server joins two tables, it chooses a physical algorithm:
1.  **Nested Loops:** Fast for small datasets. For every row in Table A, it scans Table B.
2.  **Merge Join:** Extremely fast, but requires both inputs to be sorted on the join key.
3.  **Hash Match:** Used for massive, unsorted datasets. It builds a hash table in RAM. If RAM is full, it spills to TempDB.
*Architect Rule:* If you see a Hash Match in your Execution Plan where you expect a Nested Loop, it means your Foreign Keys are missing Non-Clustered Indexes!

---

## 7.9 Common Mistakes & Production Pitfalls

1.  **Implicit Inner Joins in EF Core:** If you use `.Select(s => new { s.Port.PortNumber })` on a nullable Foreign Key relationship, EF Core might translate it as a `LEFT JOIN`. But if the FK is non-nullable, it translates as an `INNER JOIN`. Developers often don't realize that changing C# nullability changes the SQL join type, potentially dropping rows from the UI.
2.  **Over-using `.Include()`:** Developers often chain 10 `.Include()` statements to build a massive object graph for a simple API endpoint that only needed 3 properties. Use `.Select()` to project exactly what you need; EF Core will automatically generate the most efficient joins.

---

## 7.10 Production Checklist

*   [ ] Every Foreign Key constraint has a corresponding Non-Clustered Index to ensure optimal Nested Loop/Merge joins.
*   [ ] EF Core queries retrieving multiple `1:Many` collections utilize `.AsSplitQuery()` to prevent Cartesian Explosions.
*   [ ] Correlated Subqueries in legacy SQL views/stored procedures are identified and rewritten as `JOIN`s.

---

## 7.11 Exercises

1.  **Join Translation:** The Admin wants a report showing every `Tenant Name`, and the total number of `Stations` they have. Even if a Tenant has 0 Stations, their name must appear with a count of 0. Write this query using `LEFT JOIN` and `GROUP BY`.
2.  **EF Core Projection:** Write the C# EF Core LINQ query for the scenario in Exercise 1 using `.Select()` instead of `.Include()`.

---

## 7.12 Interview Questions

**Q1: What is the "Cartesian Explosion" problem in ORMs, and how do you resolve it in EF Core?**
*Answer:* A Cartesian Explosion occurs when an ORM translates a query requesting multiple `1:Many` relationships (e.g., `Include(Ports)` and `Include(Logs)`) into a single SQL query with multiple `LEFT JOIN`s. The database returns a flat table multiplying the rows together, causing massive network payloads and C# memory consumption. In EF Core, we resolve this by appending `.AsSplitQuery()`, which breaks it into separate, efficient SQL queries.

**Q2: In SQL Server Execution Plans, what physical join operation indicates that you are likely missing an index on a Foreign Key?**
*Answer:* The Hash Match join. When SQL Server joins tables via a Foreign Key that lacks an index, it cannot use a highly efficient Nested Loop or Merge Join. Instead, it must scan both tables, build a hash table in memory, and probe for matches. This requires significant CPU and memory grants, often spilling to TempDB under load.

---
**Next up in Chapter 8:** We will explore Set Operations, specifically focusing on `UNION` and `UNION ALL`, and how to combine historical archiving tables with active transactional tables.
