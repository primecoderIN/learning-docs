# Chapter 8: Set Operations (UNION)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Combine multiple independent result sets using Set Operations.
*   Understand the massive performance difference between `UNION` and `UNION ALL`.
*   Utilize `EXCEPT` and `INTERSECT` for data reconciliation.
*   Implement an architectural pattern for querying active data and archived cold storage data simultaneously.
*   Translate Set Operations cleanly into EF Core LINQ queries.

---

## 8.1 Introduction to Set Operations

While `JOIN`s combine tables horizontally (adding columns from one table to the rows of another), **Set Operations** combine result sets vertically (stacking the rows of one query on top of the rows from another).

To use a Set Operation, the two queries must follow strict rules:
1.  They must return the **same number of columns**.
2.  The data types of the corresponding columns must be **compatible** (e.g., you cannot stack a `UNIQUEIDENTIFIER` on top of a `DATETIME2`).
3.  The column names in the final result set are determined by the *first* query.

---

## 8.2 UNION vs. UNION ALL (The Hidden Cost)

### The `UNION ALL` Operator
`UNION ALL` takes the result of Query A and blindly slaps the result of Query B directly underneath it. It is incredibly fast.

```sql
SELECT Email FROM core.Users WHERE TenantId = 'T1-UUID'
UNION ALL
SELECT Email FROM core.Users WHERE TenantId = 'T2-UUID';
```
If Query A returns 1,000 rows and Query B returns 1,000 rows, the result is exactly 2,000 rows.

### The `UNION` Operator
`UNION` (without the `ALL`) does the same thing, but **it removes duplicate rows**. 

```sql
SELECT Email FROM core.Users WHERE TenantId = 'T1-UUID'
UNION 
SELECT Email FROM core.Users WHERE TenantId = 'T2-UUID';
```

**Architect Perspective:** How does SQL Server remove duplicates? It must **Sort** the entire combined result set or build a **Hash Table** to find matching rows. As discussed in Chapter 6, sorting massive datasets requires huge memory grants and often spills to TempDB.
If you know for a mathematical fact that Query A and Query B cannot possibly contain duplicates (e.g., they are filtering on mutually exclusive `TenantId`s), **never use `UNION`**. Always use `UNION ALL` to bypass the sorting penalty.

---

## 8.3 The EXCEPT and INTERSECT Operators

### EXCEPT (Difference)
Returns distinct rows from the first query that are *not* found in the second query.
This is perfect for data reconciliation. For example, finding Stations that exist in our database but have *not* sent a heartbeat telemetry ping today.

```sql
-- All Stations
SELECT StationId FROM core.Stations
EXCEPT
-- Stations that sent a heartbeat today
SELECT StationId FROM core.Heartbeats WHERE Timestamp >= CAST(SYSUTCDATETIME() AS DATE);
```

### INTERSECT
Returns distinct rows that are output by *both* queries.
```sql
SELECT StationId FROM core.Stations WHERE Status = 'Faulted'
INTERSECT
SELECT StationId FROM core.Stations WHERE FirmwareVersion = '1.0.4';
```
*(Note: While `INTERSECT` is mathematically elegant, you would typically write the above using a simple `AND` in the `WHERE` clause for better readability).*

---

## 8.4 Architect Perspective: Cold Storage and Partition Views

In our EV SaaS, the `Sessions` table grows by millions of rows per month. Querying a 500-million row table is slow.
A common enterprise pattern is **Cold Storage Archiving**:
1.  Keep the active `core.Sessions` table small (e.g., only the last 6 months of data).
2.  Move older sessions to a separate table `archive.Sessions` (often located on a cheaper, slower Filegroup/Disk).

But what if an Admin needs to generate a 3-year historical billing report? We use a `UNION ALL` View.

```sql
CREATE VIEW core.vw_AllSessions AS
SELECT SessionId, TenantId, StartTime, TotalCost FROM core.Sessions
UNION ALL
SELECT SessionId, TenantId, StartTime, TotalCost FROM archive.Sessions;
```

When an EF Core query hits `vw_AllSessions` looking for data from 2024, SQL Server's Query Optimizer is smart enough to look at the `CHECK` constraints on the underlying tables. If `archive.Sessions` has a check constraint limiting it to `< 2023`, SQL Server will completely skip scanning the archive table. This is called **Partition Elimination**.

---

## 8.5 The Code: EF Core Translations

EF Core supports these operators using LINQ methods.

### Translating `UNION ALL` (`.Concat()`)
To generate a `UNION ALL` (no distinct sort), use `.Concat()`.
```csharp
var activeUsers = context.Users.Where(u => u.IsActive);
var pendingUsers = context.Users.Where(u => u.Status == "Pending");

// Generates UNION ALL
var combinedUsers = await activeUsers.Concat(pendingUsers).ToListAsync();
```

### Translating `UNION` (`.Union()`)
To generate a `UNION` (removes duplicates), use `.Union()`.
```csharp
// Generates UNION (Requires Sorting overhead in SQL)
var combinedDistinct = await activeUsers.Union(pendingUsers).ToListAsync();
```

---

## 8.6 Performance & Security Analysis

### Performance Analysis
*   **The Implicit Sort:** The most common performance killer regarding Set Operations is developers using `UNION` when they meant `UNION ALL`. The resulting Sort/Distinct operator in the Execution Plan will consume massive CPU. 
*   **Indexing the Views:** If you use the Partition View pattern (`vw_AllSessions`), ensure the base tables have identical schemas and perfectly aligned Non-Clustered indexes. If they don't, SQL Server cannot push the `WHERE` clause down into the individual tables (Predicate Pushdown), resulting in a full table scan of the archive.

### Security Implications
*   **Column Misalignment:** Because `UNION ALL` matches columns purely by positional order (not by name), a developer modifying a SQL view might accidentally swap `Email` and `PasswordHash` in the second query. The database engine won't care because both are `VARCHAR`, but the UI will now display Password Hashes in the Email field. Always use strict, ordered `SELECT` lists.

---

## 8.7 Common Mistakes & Production Pitfalls

1.  **Data Type Conversion:** If Query 1 returns a `VARCHAR(50)` and Query 2 returns an `NVARCHAR(50)`, SQL Server must implicitly convert the entire first result set to `NVARCHAR` to satisfy data type precedence rules. This hidden conversion consumes CPU. Ensure column data types align perfectly.
2.  **ORDER BY placement:** You cannot put an `ORDER BY` clause on the first query. The `ORDER BY` must go at the very end of the final query, applying to the entire combined result set.

---

## 8.8 Production Checklist

*   [ ] `UNION ALL` is used as the default. `UNION` is strictly reserved for scenarios where duplicate removal is an absolute business requirement.
*   [ ] The application uses `.Concat()` in EF Core to achieve `UNION ALL`.
*   [ ] If using Partition Views to query cold storage, both active and archive tables have rigid `CHECK` constraints to allow the optimizer to perform Partition Elimination.

---

## 8.9 Exercises

1.  **Reconciliation Script:** The billing department needs a list of `UserIds` who registered in the system (`core.Users`), but have NEVER initiated a charging session (`core.Sessions`). Write this query using the `EXCEPT` operator.
2.  **View Creation:** Write the DDL to create a View named `Reporting.vw_TenantActivity` that combines a query selecting `TenantId` and `Name` from `core.Tenants` with a query selecting `TenantId` and a hardcoded string `'Archived'` from `archive.Tenants`. Ensure duplicates are retained without a sorting penalty.

---

## 8.10 Interview Questions

**Q1: From a physical execution perspective, why is `UNION ALL` dramatically faster than `UNION`?**
*Answer:* `UNION ALL` simply concatenates two result sets together. `UNION` has an implied `DISTINCT` requirement. To guarantee uniqueness, the database engine must process the combined data through a Sort operator or a Hash Match operator to identify and remove duplicates. This requires a memory grant and potential TempDB I/O, which is exponentially slower than simple concatenation.

**Q2: What is "Predicate Pushdown" in the context of a `UNION ALL` View (Partition View)?**
*Answer:* When querying a view that unions multiple tables (like an Active and Archive table), Predicate Pushdown is the Query Optimizer's ability to take the `WHERE` clause from the outer query and apply it directly to the underlying base tables *before* the UNION occurs. Combined with Check Constraints, this allows the engine to completely skip scanning the Archive table if the query filters for current dates (Partition Elimination).

---
**Next up in Chapter 9:** We enter Part 3 (Advanced Analytical Patterns) by mastering Common Table Expressions (CTEs), exploring how to make complex queries readable and how to navigate hierarchical data using Recursive CTEs.
