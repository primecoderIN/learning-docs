# Chapter 5: The SELECT Statement & Filtering

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand the **Logical Query Processing** order (why SQL executes `WHERE` before `SELECT`).
*   Identify the performance dangers of `SELECT *` in high-throughput APIs.
*   Master the concept of **SARGability** (Search Argument-able) and why applying functions to columns in a `WHERE` clause destroys index seeks.
*   Ensure EF Core generates SARGable queries for optimal execution.

---

## Part 2: Querying the SaaS Domain

Now that we understand how data is physically stored and constrained, we must learn how to retrieve it efficiently. In a global SaaS, retrieving data quickly is often harder than storing it.

## 5.1 Logical Query Processing (The Execution Order)

When you write a C# LINQ query, the code executes top-to-bottom. SQL is fundamentally different; it is a **declarative** language. You tell the engine *what* you want, and the Query Optimizer decides *how* to get it. 

Even though you type `SELECT` first, SQL Server processes the query in a specific logical order:
1.  `FROM` (Identify the tables)
2.  `WHERE` (Filter the rows)
3.  `GROUP BY` (Aggregate)
4.  `HAVING` (Filter the aggregated groups)
5.  `SELECT` (Return the specific columns)
6.  `ORDER BY` (Sort the result)
7.  `OFFSET / FETCH` (Paginate)

**Architect Perspective:** Because `SELECT` is processed step 5, you cannot use a column alias defined in the `SELECT` clause within your `WHERE` clause (step 2). Understanding this order is crucial for debugging complex analytical queries.

---

## 5.2 The SELECT Clause and Memory Grants

The `SELECT` clause projects the columns you want returned to the application.

```sql
-- BAD (The Anti-Pattern)
SELECT * 
FROM core.Sessions 
WHERE TenantId = 'T1-UUID';

-- GOOD (Explicit Projection)
SELECT SessionId, StartTime, TotalKwh 
FROM core.Sessions 
WHERE TenantId = 'T1-UUID';
```

### Why `SELECT *` is a Production Killer
1.  **Network Payload:** `SELECT *` sends every column over the wire. If someone adds a `VARCHAR(MAX)` `DebugLog` column to the `Sessions` table, your API payload instantly balloons, causing network latency and high GC (Garbage Collection) pressure in your .NET backend.
2.  **Memory Grants:** Before SQL Server executes a query, it estimates how much RAM it needs to hold the results in memory for sorting or hashing. If you use `SELECT *`, the engine requests a massive memory grant. Under heavy load, other queries will be forced to wait in a queue for memory to free up, causing application-wide timeouts.
3.  **Covering Indexes:** Explicit `SELECT` lists allow the Query Optimizer to use "Covering Indexes" (indexes that contain all the columns you requested), preventing expensive trips to the physical table (Key Lookups).

---

## 5.3 The WHERE Clause & Null Handling

The `WHERE` clause filters the rows returned by the `FROM` clause.

### Boolean Logic & NULLs
SQL uses **Three-Valued Logic**: `TRUE`, `FALSE`, and `UNKNOWN` (NULL).
If you evaluate `WHERE EndTime = NULL`, the result is `UNKNOWN`. The `WHERE` clause only returns rows that evaluate to `TRUE`. 

```sql
-- Will always return zero rows.
SELECT SessionId FROM core.Sessions WHERE EndTime = NULL; 

-- Correct way to query NULLs
SELECT SessionId FROM core.Sessions WHERE EndTime IS NULL;
```

---

## 5.4 SARGability: The Most Important Concept in SQL Filtering

**SARGable** stands for **S**earch **ARG**ument **ABLE**. 
A query is SARGable if the SQL Server Query Optimizer can use an Index to quickly find the rows (an **Index Seek**). If a query is non-SARGable, the engine must read every single row in the table (an **Index Scan**).

In a SaaS with 100 million charging sessions, an Index Scan will take 30 seconds. An Index Seek will take 2 milliseconds.

### The Golden Rule of SARGability
**Never apply a function or mathematical operation to the column side of your `WHERE` clause.**

#### Example 1: Date Filtering
We want all sessions that started in the year 2024.

```sql
-- NON-SARGABLE (Terrible Performance)
-- SQL must evaluate YEAR() for all 100 million rows to check the result. (Index Scan)
SELECT SessionId FROM core.Sessions
WHERE YEAR(StartTime) = 2024;

-- SARGABLE (Millisecond Performance)
-- SQL can traverse the B-Tree index on StartTime to find the exact range. (Index Seek)
SELECT SessionId FROM core.Sessions
WHERE StartTime >= '2024-01-01' AND StartTime < '2025-01-01';
```

#### Example 2: String Searching
We want to find all Stations whose names start with 'Lobby'.

```sql
-- NON-SARGABLE (Index Scan)
SELECT StationId FROM core.Stations
WHERE SUBSTRING(Name, 1, 5) = 'Lobby';

-- NON-SARGABLE (Index Scan - Leading Wildcard)
-- The engine cannot use an alphabetical index if the first letter is unknown.
SELECT StationId FROM core.Stations
WHERE Name LIKE '%Lobby%';

-- SARGABLE (Index Seek)
-- The engine can instantly jump to the 'L' section of the index.
SELECT StationId FROM core.Stations
WHERE Name LIKE 'Lobby%';
```

---

## 5.5 EF Core Translations

How do we ensure our C# code generates SARGable SQL?

### The `.Contains()` vs `.StartsWith()` Trap

```csharp
// Generates: WHERE Name LIKE N'%Lobby%' (NON-SARGABLE)
var badQuery = await context.Stations
    .Where(s => s.Name.Contains("Lobby"))
    .ToListAsync();

// Generates: WHERE Name LIKE N'Lobby%' (SARGABLE)
var goodQuery = await context.Stations
    .Where(s => s.Name.StartsWith("Lobby"))
    .ToListAsync();
```

### Date SARGability in EF Core

```csharp
// BAD: Generates WHERE DATEPART(year, StartTime) = 2024 (NON-SARGABLE)
var badDates = await context.Sessions
    .Where(s => s.StartTime.Year == 2024)
    .ToListAsync();

// GOOD: Generates WHERE StartTime >= '2024-01-01' AND ... (SARGABLE)
var start = new DateTime(2024, 1, 1);
var end = start.AddYears(1);
var goodDates = await context.Sessions
    .Where(s => s.StartTime >= start && s.StartTime < end)
    .ToListAsync();
```

---

## 5.6 Performance & Security Analysis

### Performance Analysis
*   **Implicit Conversions:** Another hidden cause of non-SARGable queries is data type mismatch. If your database column is `VARCHAR` but your C# code passes a `string` (which translates to `NVARCHAR`), SQL Server must implicitly convert every row's `VARCHAR` to `NVARCHAR` before comparing. This destroys the index seek. Always configure EF Core strings precisely using `.HasColumnType("VARCHAR(X)")`.

### Security Implications
*   **LIKE Operator SQL Injection:** If you allow users to pass wildcard characters (`%` or `_`) into a search box, they can intentionally trigger massive Index Scans (e.g., searching for `%a%`), causing a Denial of Service (DoS) attack on your database. You must sanitize or escape search inputs before passing them into `.StartsWith()` or `.Contains()`.

---

## 5.7 Common Mistakes & Production Pitfalls

1.  **Trusting the Local Dev Database:** A non-SARGable query against a local developer database with 1,000 rows will return in 5ms. The developer assumes it is fast. When deployed to production against 50 million rows, it times out immediately.
2.  **Using `ISNULL` in the WHERE clause:** `WHERE ISNULL(Status, 'Pending') = 'Pending'` is non-SARGable. Rewrite it as `WHERE Status IS NULL OR Status = 'Pending'`.

---

## 5.8 Production Checklist

*   [ ] No `SELECT *` used in production code; all EF Core queries use `.Select(x => new Dto { ... })` for explicit projection.
*   [ ] String searches default to `.StartsWith()` (Sargable) instead of `.Contains()` (Non-Sargable) unless full-text search is strictly required.
*   [ ] Date filtering uses explicit range boundaries (`>=` and `<`) instead of `.Year`, `.Month`, or `.Day` properties.

---

## 5.9 Exercises

1.  **Refactoring for SARGability:** A legacy stored procedure contains this filter: 
    `WHERE CONVERT(DATE, StartTime) = '2024-10-15'`. 
    Rewrite this `WHERE` clause so it is fully SARGable.
2.  **EF Core Projection:** You have a `Session` entity with 15 columns, including a `Guid SessionId`, `Guid TenantId`, and `Decimal TotalCost`. Write the EF Core LINQ query to return only a list of `TotalCost` values for a specific Tenant, ensuring no other columns are queried from the database.

---

## 5.10 Interview Questions

**Q1: Explain what SARGable means. Give an example of a SARGable query and a non-SARGable query targeting a Date column.**
*Answer:* SARGable means "Search Argument-able". It refers to a query's `WHERE` clause being written in a way that allows the Query Optimizer to use an Index Seek rather than an Index Scan. 
Non-SARGable: `WHERE YEAR(CreatedAt) = 2024` (The engine must apply the function to every row).
SARGable: `WHERE CreatedAt >= '2024-01-01' AND CreatedAt < '2025-01-01'` (The engine can traverse the B-Tree index using the exact values).

**Q2: Why does SQL Server evaluate the `WHERE` clause before the `SELECT` clause, and how does this impact query writing?**
*Answer:* SQL evaluates `FROM` -> `WHERE` -> `SELECT`. It must filter the rows (`WHERE`) before it knows which columns to project or format (`SELECT`). This means you cannot define an alias in the `SELECT` clause (e.g., `SELECT TotalKwh * 0.15 AS Tax`) and then attempt to filter on it in the `WHERE` clause (`WHERE Tax > 5.00`). You must repeat the calculation in the `WHERE` clause or use a Common Table Expression (CTE).

---
**Next up in Chapter 6:** We will explore Sorting and Aggregation, diving into the mechanics of `GROUP BY`, `HAVING`, and paginating massive datasets safely.
