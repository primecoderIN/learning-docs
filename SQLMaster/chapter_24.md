# Chapter 24: EF Core Bulk Updates & Raw SQL

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the performance flaws of traditional Entity Framework Core updates (Querying before Updating).
*   Implement EF Core 7+ `ExecuteUpdateAsync` and `ExecuteDeleteAsync` to modify data without touching the Change Tracker.
*   Understand the architectural trade-offs of bypassing the Change Tracker (lost auditing/events).
*   Execute raw SQL commands securely using `ExecuteSqlInterpolatedAsync` to prevent SQL Injection.

---

## 24.1 The Traditional EF Core Update Problem

In Chapter 23, we saw how the Change Tracker generates `UPDATE` statements. 
Suppose the Tenant Admin clicks a button to disable all 50 of their charging stations.

**The Traditional (Pre-EF7) Way:**
```csharp
// 1. SELECT all 50 stations into C# Memory (Slow)
var stations = await _context.Stations
    .Where(s => s.TenantId == tenantId)
    .ToListAsync();

// 2. Loop through them in C# (CPU Overhead)
foreach(var station in stations)
{
    station.Status = "Disabled";
}

// 3. Fire 50 individual UPDATE statements to the DB (Network Latency)
await _context.SaveChangesAsync();
```

This is terribly inefficient. To update data, you shouldn't have to download it first. In raw SQL, you would just write: `UPDATE Stations SET Status = 'Disabled' WHERE TenantId = X`.

---

## 24.2 EF Core 7+ Bulk Updates (`ExecuteUpdateAsync`)

Entity Framework Core 7 completely solved this problem by introducing `ExecuteUpdateAsync`. 
This method translates your LINQ expression directly into a SQL `UPDATE` statement and executes it instantly, completely bypassing the Change Tracker.

```csharp
// The Modern Way
int rowsAffected = await _context.Stations
    .Where(s => s.TenantId == tenantId) // The WHERE clause
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.Status, "Disabled")); // The SET clause
```

**Why this is revolutionary:**
1.  **Zero Memory:** No data is downloaded to the C# application.
2.  **One Query:** EF Core generates exactly one `UPDATE` statement, regardless of whether it updates 5 rows or 50,000 rows.
3.  **Speed:** The execution time drops from hundreds of milliseconds to 2 milliseconds.

### Dynamic Updates
You can even use existing column values in the update calculation:
```csharp
// UPDATE Wallets SET Balance = Balance - 15.00 WHERE UserId = @p0
await _context.Wallets
    .Where(w => w.UserId == userId)
    .ExecuteUpdateAsync(w => w.SetProperty(x => x.Balance, x => x.Balance - 15.00m));
```

---

## 24.3 EF Core 7+ Bulk Deletes (`ExecuteDeleteAsync`)

The exact same concept applies to deleting records. You no longer need to query them just to call `_context.Remove()`.

```csharp
// Instantly generates: DELETE FROM core.Telemetry WHERE CreatedAt < @p0
int rowsDeleted = await _context.Telemetry
    .Where(t => t.CreatedAt < DateTime.UtcNow.AddMonths(-3))
    .ExecuteDeleteAsync();
```
*(Architect Reminder: If the table has 500 million rows, you should use Table Partitioning for deletes as covered in Chapter 22, not `ExecuteDelete`!)*

---

## 24.4 Architect Perspective: The Trade-off

Bypassing the Change Tracker using `ExecuteUpdate` is incredibly fast, but it comes with severe architectural trade-offs that you must consider.

1.  **Lost SaveChanges Interceptors:** If you configured EF Core Interceptors to automatically update a `LastModifiedUtc` column every time `SaveChanges` is called, `ExecuteUpdate` **will not trigger it**. You must manually update that property in the `.SetProperty()` call.
2.  **Lost Domain Events:** If your Domain Entities dispatch events when properties change (e.g., publishing a `StationDisabledEvent`), bypassing the Change Tracker means those events will never fire.
3.  **In-Memory Inconsistency:** If you have a `Station` loaded in your C# memory, and you run `ExecuteUpdate` on the database, the C# object in memory will still have the old values.

**Architect Rule:** Use `SaveChanges()` for complex domain logic involving single aggregates (where validation and events are critical). Use `ExecuteUpdate()` for bulk operational tasks, data patching, or performance-critical micro-updates (like deducting a wallet balance).

---

## 24.5 Raw SQL Execution

Sometimes, LINQ is simply not expressive enough. If you need to execute a complex `MERGE` statement (Chapter 11) or a query utilizing Window Functions (Chapter 10), you must drop down to Raw SQL.

### Queries (Returning Data)
To map a raw SQL query directly to an Entity or a DTO, use `FromSqlInterpolated`.

```csharp
var tenantId = new Guid("...");
// The string interpolation '{tenantId}' looks like SQL Injection, but it is NOT.
// EF Core intercepts the interpolation and generates a secure DbParameter!
var stations = await _context.Stations
    .FromSqlInterpolated($"SELECT * FROM core.Stations WHERE TenantId = {tenantId}")
    .ToListAsync();
```

### Commands (Modifying Data)
To execute `INSERT`, `UPDATE`, `DELETE`, or Stored Procedures that do not return Entities, use `ExecuteSqlInterpolatedAsync`.

```csharp
var tenantId = new Guid("...");
var month = 1;
// Executes: EXEC billing.sp_ProcessInvoice @p0, @p1
await _context.Database.ExecuteSqlInterpolatedAsync(
    $"EXEC billing.sp_ProcessInvoice {tenantId}, {month}");
```

---

## 24.6 Performance & Security Analysis

### Security Implications: SQL Injection
If you use `FromSqlRaw` instead of `FromSqlInterpolated`, and you use standard C# string concatenation, you have introduced a critical SQL Injection vulnerability.

```csharp
// DANGEROUS: SQL Injection Vulnerability!
var badSql = "SELECT * FROM Users WHERE Email = '" + userInput + "'";
var users = await _context.Users.FromSqlRaw(badSql).ToListAsync();
```
*   **The Fix:** Always use `FromSqlInterpolated` (which safely parameterizes interpolated variables) OR explicitly create `SqlParameter` objects if using `FromSqlRaw`. Never concatenate strings.

---

## 24.7 Common Mistakes & Production Pitfalls

1.  **Calling SaveChanges after ExecuteUpdate:** 
    `ExecuteUpdateAsync` sends the command to the database *immediately*. You do not need to call `_context.SaveChangesAsync()` afterward. Doing so is useless and adds a round-trip to the DB.
2.  **ExecuteUpdate on JOINs:** While you can use `ExecuteUpdate` on a filtered `IQueryable`, if your `IQueryable` involves an `Include()` or a complex `.Join()`, EF Core may throw an exception or generate a highly inefficient subquery. Keep bulk updates simple and targeted to a single table.

---

## 24.8 Production Checklist

*   [ ] Operations that merely flip a flag on multiple rows (e.g., `IsActive = 0`) utilize `ExecuteUpdateAsync` instead of the `SELECT-Foreach-SaveChanges` anti-pattern.
*   [ ] Raw SQL execution strictly uses `FromSqlInterpolated` or `ExecuteSqlInterpolatedAsync` to guarantee parameterization.
*   [ ] String concatenation (`+`) is explicitly banned in all EF Core raw SQL methods during code reviews.

---

## 24.9 Exercises

1.  **Refactoring for Performance:** A developer wrote the following code to reset all active alarms for a specific station:
    ```csharp
    var alarms = await _context.Alarms.Where(a => a.StationId == id && a.IsActive).ToListAsync();
    foreach(var alarm in alarms) { alarm.IsActive = false; }
    await _context.SaveChangesAsync();
    ```
    Refactor this code to use exactly one line of code via EF Core 7+ `ExecuteUpdateAsync`.
2.  **SQL Injection Audit:** Explain why `_context.Database.ExecuteSqlInterpolatedAsync($"EXEC sp_Delete {userId}")` is safe, but `_context.Database.ExecuteSqlRawAsync($"EXEC sp_Delete {userId}")` is a critical vulnerability.

---

## 24.10 Interview Questions

**Q1: What is the primary performance benefit of using `ExecuteUpdateAsync` in EF Core 7+ compared to the traditional `SaveChanges` approach?**
*Answer:* The traditional approach requires a network round-trip to `SELECT` the data into C# memory, consumes RAM to attach those objects to the Change Tracker, and then generates individual `UPDATE` statements for each modified row. `ExecuteUpdateAsync` bypasses the Change Tracker entirely, generating and executing a single SQL `UPDATE ... WHERE ...` statement directly on the database. This eliminates memory allocation and reduces network latency to a single round-trip.

**Q2: What is the architectural trade-off of bypassing the Change Tracker with Bulk Updates?**
*Answer:* Bypassing the Change Tracker means that EF Core's lifecycle hooks are skipped. If your architecture relies on overriding `SaveChanges` to automatically set auditing columns (like `ModifiedDate` or `ModifiedBy`), or if you use Domain-Driven Design (DDD) to dispatch Domain Events when entity states change, none of those will execute. You trade business-logic encapsulation for raw database performance.

---
**Next up in Chapter 25:** We will tackle the complexities of Concurrency Control. What happens when two users try to edit the same Station configuration at the exact same time? We will explore Optimistic Concurrency and Concurrency Tokens.
