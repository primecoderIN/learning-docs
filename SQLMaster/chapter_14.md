# Chapter 14: Dynamic SQL in Enterprise Apps

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the severe performance issues caused by the "Catch-All" query anti-pattern in reporting dashboards.
*   Construct high-performance Dynamic SQL strings inside Stored Procedures.
*   Execute Dynamic SQL safely using `sp_executesql` to prevent SQL Injection attacks.
*   Replicate dynamic query generation cleanly in the application layer using EF Core `IQueryable` chaining.

---

## 14.1 The Need for Dynamic Filters

In our EV SaaS, the Tenant Admin dashboard has a complex search grid. The user can filter sessions by `StationId`, `PortId`, `Status`, `StartDate`, `EndDate`, or any combination of these five parameters. 

If they leave a filter blank, the API passes `NULL` to the database.

How do you write a single Stored Procedure to handle 32 different combinations of parameters?

---

## 14.2 The "Catch-All" Query Anti-Pattern

Historically, developers solved this using a technique called the "Catch-All" (or "Kitchen Sink") query.

```sql
CREATE PROCEDURE reporting.usp_SearchSessions 
    @TenantId UNIQUEIDENTIFIER,
    @StationId UNIQUEIDENTIFIER = NULL,
    @Status VARCHAR(20) = NULL
AS
BEGIN
    SELECT SessionId, StartTime, TotalCost 
    FROM core.Sessions
    WHERE TenantId = @TenantId
      AND (@StationId IS NULL OR StationId = @StationId)
      AND (@Status IS NULL OR Status = @Status);
END
```

### The Performance Disaster
This looks elegant, but it destroys performance. 
When SQL Server compiles the Execution Plan for this procedure, it has to create a plan that works for *any* parameter combination. Because it doesn't know if `@StationId` will be provided, it cannot safely use the Index on `StationId`. 

The result? The Query Optimizer gives up and performs a **Clustered Index Scan** (scanning the entire table) every single time, regardless of what parameters you pass.

---

## 14.3 Implementing Dynamic SQL via `sp_executesql`

To get perfect index seeks for every combination of filters, we must build the SQL string dynamically at runtime and execute it. 
This ensures the Query Optimizer generates an Execution Plan specifically tailored to the exact parameters provided.

### Building the String
We construct the base query, and then append `WHERE` clauses only if the parameter is not null.

```sql
CREATE PROCEDURE reporting.usp_SearchSessionsDynamic
    @TenantId UNIQUEIDENTIFIER,
    @StationId UNIQUEIDENTIFIER = NULL,
    @Status VARCHAR(20) = NULL
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    
    -- 1. Base Query
    SET @SQL = N'
        SELECT SessionId, StartTime, TotalCost 
        FROM core.Sessions 
        WHERE TenantId = @TenantId ';

    -- 2. Dynamically append filters
    IF @StationId IS NOT NULL
        SET @SQL = @SQL + N' AND StationId = @StationId ';
        
    IF @Status IS NOT NULL
        SET @SQL = @SQL + N' AND Status = @Status ';

    -- 3. Execute safely via sp_executesql
    EXEC sp_executesql 
        @stmt = @SQL, 
        @params = N'@TenantId UNIQUEIDENTIFIER, @StationId UNIQUEIDENTIFIER, @Status VARCHAR(20)',
        @TenantId = @TenantId,
        @StationId = @StationId,
        @Status = @Status;
END
```

**Why this is fast:** If the user only provides `@TenantId`, the executed SQL is literally `WHERE TenantId = @TenantId`. SQL Server will compile an Index Seek plan perfectly optimized for that exact string.

---

## 14.4 Mitigating SQL Injection in Dynamic SQL

**CRITICAL RULE:** Never concatenate parameter *values* directly into the SQL string.

### The Lethal Mistake
```sql
-- DO NOT DO THIS!
SET @SQL = N'SELECT * FROM Users WHERE Email = ''' + @Email + '''';
EXEC(@SQL);
```
If `@Email` is `'admin@test.com' OR 1=1; DROP TABLE core.Sessions; --`, the attacker just deleted your production database.

### The Solution: Parameterized Execution
As shown in section 14.3, we use `sp_executesql`. We concatenate the parameter *names* (e.g., `@Status`), not the values. We then pass the strongly-typed variables into `sp_executesql`. This guarantees the database engine treats the user input as literal data, completely neutralizing SQL Injection.

---

## 14.5 The Code: EF Core Dynamic LINQ

One of the greatest advantages of EF Core is that you rarely need to write Dynamic SQL in Stored Procedures. The `IQueryable<T>` interface in C# allows you to conditionally chain `WHERE` clauses before the query is ever sent to the database.

```csharp
public async Task<List<SessionDto>> SearchSessionsAsync(
    Guid tenantId, Guid? stationId, string? status)
{
    // 1. Base Query (Not executed yet!)
    IQueryable<Session> query = _context.Sessions
        .Where(s => s.TenantId == tenantId);

    // 2. Conditionally append filters
    if (stationId.HasValue)
    {
        query = query.Where(s => s.StationId == stationId.Value);
    }

    if (!string.IsNullOrEmpty(status))
    {
        query = query.Where(s => s.Status == status);
    }

    // 3. Execute
    // EF Core translates the chained expression tree into a perfectly optimized,
    // parameterized SQL string matching the EXACT provided filters.
    return await query
        .Select(s => new SessionDto { /* ... */ })
        .ToListAsync();
}
```
**Architect Perspective:** This `IQueryable` chaining pattern is the preferred, enterprise-standard way to handle dynamic search grids. It pushes the string-building logic into the application layer, keeps the database layer clean, and is 100% immune to SQL Injection (because EF Core parameterizes everything natively).

---

## 14.6 Performance & Security Analysis

### Performance Analysis: Plan Cache Bloat
If you use Dynamic SQL (or EF Core), SQL Server caches an Execution Plan for every unique SQL string generated. 
*   `WHERE TenantId = @p1` generates Plan A.
*   `WHERE TenantId = @p1 AND Status = @p2` generates Plan B.
This is generally good for performance, but in highly complex grids with 50+ optional filters, it can lead to **Plan Cache Bloat**, where thousands of slightly different execution plans consume SQL Server's RAM. To mitigate this in massive systems, enable the `Optimize for Ad hoc Workloads` setting at the server level.

### Security Implications
*   **EXEC vs sp_executesql:** `EXEC(@SQL)` executes a raw string and is highly vulnerable to injection. `sp_executesql` enforces parameterization. Always ban `EXEC(@SQL)` in your code reviews.

---

## 14.7 Common Mistakes & Production Pitfalls

1.  **Quotation Mark Hell:** Writing Dynamic SQL requires handling single quotes inside strings. 
    `SET @SQL = 'SELECT * FROM Table WHERE Name = ''O''Connor'''`. 
    This is extremely prone to typos and syntax errors. Avoid hardcoding strings inside Dynamic SQL; use parameters via `sp_executesql`.
2.  **Schema and Object Injection:** While `sp_executesql` protects column *values*, it cannot parameterize column *names* or table *names*. If you dynamically build a query where the user selects which column to sort by (`ORDER BY @ColumnName`), you must manually validate `@ColumnName` against a hardcoded whitelist in your C# or T-SQL code to prevent injection.

---

## 14.8 Production Checklist

*   [ ] "Catch-All" queries (`WHERE @Param IS NULL OR Col = @Param`) have been removed from high-traffic Stored Procedures.
*   [ ] Dynamic SQL generation in T-SQL is strictly executed using `sp_executesql` with strong parameterization.
*   [ ] Application-side dynamic queries utilize EF Core `IQueryable` chaining rather than raw SQL string concatenation.
*   [ ] Dynamic `ORDER BY` columns are validated against a strict whitelist to prevent object-injection attacks.

---

## 14.9 Exercises

1.  **Anti-Pattern Identification:** A reporting procedure uses `WHERE (st.Name LIKE '%' + @SearchTerm + '%' OR @SearchTerm IS NULL)`. Explain the two distinct performance anti-patterns present in this single line of code.
2.  **IQueryable Implementation:** Write a C# method that takes a `startDate` and an optional `endDate`. Using `IQueryable`, query the `Sessions` table. If `endDate` is provided, filter for `StartTime < endDate`. If it is not provided, do not apply the filter.

---

## 14.10 Interview Questions

**Q1: Why is the "Catch-All" query pattern (`WHERE @P1 IS NULL OR Column = @P1`) detrimental to database performance?**
*Answer:* The SQL Server Query Optimizer must generate a single Execution Plan that works for all possible parameter combinations. Because the parameter might be NULL, the optimizer cannot guarantee that an Index Seek is safe or exhaustive. Consequently, it usually defaults to a Clustered Index Scan, meaning the engine will read every single row in the table, destroying performance on large datasets.

**Q2: What is the primary security difference between using `EXEC(@SQLString)` and `EXEC sp_executesql @SQLString`?**
*Answer:* `EXEC(@SQLString)` executes the raw string exactly as provided. If user input is concatenated into that string, it allows trivial SQL Injection. `sp_executesql` requires you to define and pass strongly-typed parameters. The database engine treats these parameters strictly as literal values, never as executable code, completely mitigating standard SQL Injection attacks.

---
**Next up in Chapter 15:** We will explore how SQL Server handles Semi-Structured data, focusing on storing, querying, and indexing JSON payloads directly from our IoT hardware.
