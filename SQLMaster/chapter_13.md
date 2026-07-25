# Chapter 13: Stored Procedures & Functions

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand when to encapsulate business logic in Stored Procedures versus the application layer.
*   Differentiate between Stored Procedures and User-Defined Functions (UDFs).
*   Analyze the devastating performance penalty of Scalar UDFs (Row-By-Agonizing-Row execution).
*   Implement Inline Table-Valued Functions (iTVFs) as a high-performance alternative to Views with parameters.
*   Execute Stored Procedures efficiently using Entity Framework Core.

---

## 13.1 Stored Procedures

A **Stored Procedure (SP)** is a batch of SQL statements saved in the database. Unlike Views (which only execute `SELECT` statements), Stored Procedures can perform complex logic including `IF/ELSE` branching, `WHILE` loops, `TRY/CATCH` error handling, and data modifications (`INSERT/UPDATE/DELETE`).

**Architect Perspective: DB vs. App Layer**
In the 2000s, architects put *all* business logic in Stored Procedures. In the modern cloud era, we generally prefer putting business logic in the Application Layer (C# / Next.js) so it can scale out elastically across stateless web servers. 
However, you **must** use Stored Procedures when a transaction requires multiple round-trips to the database to evaluate data before writing. A Stored Procedure executes entirely on the database server, eliminating network latency.

```sql
CREATE PROCEDURE billing.sp_ProcessInvoice 
    @TenantId UNIQUEIDENTIFIER,
    @Month INT,
    @Year INT
AS
BEGIN
    SET NOCOUNT ON; -- Prevents network spam of "1 row affected" messages
    
    BEGIN TRY
        BEGIN TRAN;
        
        -- 1. Calculate totals
        DECLARE @Total DECIMAL(19,4);
        SELECT @Total = SUM(TotalCost) 
        FROM core.Sessions 
        WHERE TenantId = @TenantId AND MONTH(StartTime) = @Month AND YEAR(StartTime) = @Year;
        
        -- 2. Insert Invoice
        INSERT INTO billing.Invoices (TenantId, Amount) VALUES (@TenantId, @Total);
        
        COMMIT TRAN;
    END TRY
    BEGIN CATCH
        ROLLBACK TRAN;
        THROW; -- Rethrow error to the C# application
    END CATCH
END;
```

---

## 13.2 User-Defined Functions (UDFs)

While Stored Procedures execute actions, **Functions** are designed to calculate and return values. Crucially, a Function can be embedded directly inside a `SELECT` or `WHERE` clause, whereas a Stored Procedure cannot.

There are two main types of UDFs:
1.  **Scalar Functions:** Returns a single value (e.g., a string or integer).
2.  **Table-Valued Functions (TVFs):** Returns a table.

---

## 13.3 The Scalar Function Penalty

Suppose the business requires a complex tax calculation on EV charging sessions based on state regulations. A junior developer creates a Scalar Function to encapsulate the math.

```sql
CREATE FUNCTION billing.fn_CalculateTax (@Cost DECIMAL(19,4), @StateCode VARCHAR(2))
RETURNS DECIMAL(19,4)
AS
BEGIN
    DECLARE @Tax DECIMAL(19,4);
    -- (Complex IF/ELSE logic omitted)
    SET @Tax = @Cost * 0.08;
    RETURN @Tax;
END;
```
They use it in a query:
```sql
SELECT SessionId, billing.fn_CalculateTax(TotalCost, 'CA') AS TaxAmount
FROM core.Sessions;
```

### The Disaster (RBAR)
Scalar functions are the single biggest performance killer in SQL Server. 
If the `core.Sessions` table has 1,000,000 rows, SQL Server does *not* process this as a set. It executes the query, and then it invokes the `fn_CalculateTax` function **one million individual times** in a loop (Row-By-Agonizing-Row). A query that should take 50ms will take 2 minutes.

*(Note: SQL Server 2019+ introduced Scalar UDF Inlining, which attempts to fix this automatically, but it often fails on complex logic. Always avoid Scalar UDFs for heavy queries).*

---

## 13.4 Inline Table-Valued Functions (iTVFs)

If Scalar Functions are bad, and Views don't accept parameters, how do we encapsulate parameterized query logic?

We use an **Inline Table-Valued Function (iTVF)**.
An iTVF returns a Table, but it contains no `BEGIN/END` block. It is simply a single `RETURN (SELECT ...)` statement. Because of this, the Query Optimizer treats it exactly like a View, expanding it into the main query, resulting in blazing fast performance.

```sql
CREATE FUNCTION reporting.fn_GetSessionsByDateRange (
    @TenantId UNIQUEIDENTIFIER,
    @StartDate DATETIME2,
    @EndDate DATETIME2
)
RETURNS TABLE
AS
RETURN (
    SELECT SessionId, StartTime, TotalCost
    FROM core.Sessions
    WHERE TenantId = @TenantId 
      AND StartTime >= @StartDate 
      AND StartTime < @EndDate
);
```
**Usage:**
```sql
SELECT * FROM reporting.fn_GetSessionsByDateRange('T1-UUID', '2024-01-01', '2024-02-01');
```

---

## 13.5 The Code: EF Core Integration

### Executing Stored Procedures
To execute a Stored Procedure that returns data (e.g., a complex report), use `.FromSqlRaw()`. To execute a Stored Procedure that modifies data (like `sp_ProcessInvoice`), use `.ExecuteSqlRawAsync()`.

```csharp
// Modifying Data
var tenantIdParam = new SqlParameter("TenantId", dto.TenantId);
var monthParam = new SqlParameter("Month", dto.Month);
var yearParam = new SqlParameter("Year", dto.Year);

await context.Database.ExecuteSqlRawAsync(
    "EXEC billing.sp_ProcessInvoice @TenantId, @Month, @Year", 
    tenantIdParam, monthParam, yearParam);
```

### Executing iTVFs
EF Core allows you to map a C# method directly to a SQL Function using `[DbFunction]`.

```csharp
// In VoltCoreDbContext.cs
[DbFunction("fn_GetSessionsByDateRange", "reporting")]
public IQueryable<SessionDto> GetSessions(Guid tenantId, DateTime start, DateTime end)
    => FromExpression(() => GetSessions(tenantId, start, end));
```
Now you can call this method directly in your LINQ queries, and EF Core will translate it perfectly into the iTVF.

---

## 13.6 Performance & Security Analysis

### Performance Analysis
*   **Parameter Sniffing:** Stored procedures are compiled and their Execution Plans are cached the first time they run. If the first run uses parameters that return 1 row, SQL caches an "Index Seek" plan. If the next run uses parameters that return 1,000,000 rows, it will reuse the "Seek" plan instead of switching to a "Scan", resulting in terrible performance. This is called Parameter Sniffing (Detailed deeply in Chapter 21).

### Security Implications
*   **Principle of Least Privilege:** Stored Procedures are excellent for security. You can grant an application service account `EXECUTE` permission on `sp_ProcessInvoice`, while completely denying it `UPDATE` or `DELETE` permissions on the underlying `core.Sessions` and `billing.Invoices` tables. The application can *only* perform the exact business workflows you defined.

---

## 13.7 Common Mistakes & Production Pitfalls

1.  **Multi-Statement TVFs:** If you create a TVF that uses a `BEGIN/END` block and populates a temporary table variable (`@Table TABLE (...)`), it is called a Multi-Statement TVF. These suffer from terrible cardinality estimates (SQL Server often assumes they will always return exactly 1 or 100 rows, regardless of reality), causing massive performance issues in large joins. Always use **Inline** TVFs.
2.  **Using `sp_` prefix:** Never name your Stored Procedures starting with `sp_` (e.g., `sp_GetUsers`). SQL Server reserves this prefix for System Procedures. It will first check the `master` database before checking your user database, causing a slight CPU penalty. Use prefixes like `usp_` or schema names (e.g., `billing.ProcessInvoice`).

---

## 13.8 Production Checklist

*   [ ] Procedural logic requiring high database iteration (e.g., looping through cursors, complex multi-step transactions) is encapsulated in Stored Procedures to reduce network latency.
*   [ ] Scalar UDFs are strictly avoided in queries that process large datasets.
*   [ ] Parameterized views are implemented using Inline Table-Valued Functions (iTVFs).
*   [ ] Stored procedures start with `SET NOCOUNT ON` to optimize network traffic.

---

## 13.9 Exercises

1.  **Refactoring to iTVF:** A developer wrote a Scalar Function `fn_GetStationName(@StationId)` and embedded it in a `SELECT` statement returning 500,000 rows, causing a massive RBAR performance issue. Refactor this logic into an Inline Table-Valued Function (iTVF) that takes a `@TenantId` and returns a table of Station Names and IDs.
2.  **Stored Procedure Best Practices:** Write the skeleton for a stored procedure named `usp_DeactivateTenant` in the `core` schema. It takes a `@TenantId`. It must include `SET NOCOUNT ON`, a `BEGIN TRY/CATCH` block, and a transaction wrapper.

---

## 13.10 Interview Questions

**Q1: Explain why a Scalar User-Defined Function (UDF) placed in a `SELECT` clause often causes severe performance issues on large tables.**
*Answer:* Scalar UDFs force SQL Server into RBAR (Row-By-Agonizing-Row) execution. Instead of processing the data set holistically, the engine must context-switch and invoke the function's logic individually for every single row returned by the query. This prevents parallel execution and massively inflates CPU and elapsed time.

**Q2: If a View cannot accept parameters, what SQL object should you use to create a reusable, parameterized, high-performance query?**
*Answer:* An Inline Table-Valued Function (iTVF). Because an iTVF consists of a single `RETURN (SELECT...)` statement without a `BEGIN/END` block, the Query Optimizer treats it exactly like a parameterized view. It expands the logic into the main query and generates an optimal execution plan, avoiding the RBAR penalty of scalar functions and the cardinality estimation issues of multi-statement TVFs.

---
**Next up in Chapter 14:** We will explore Dynamic SQL in enterprise applications, learning how to safely build queries at runtime for complex dashboard filters without opening the door to SQL Injection attacks.
