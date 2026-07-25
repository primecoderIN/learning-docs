# Chapter 4: Core CRUD Operations

## Learning Objectives
By the end of this chapter, you will be able to:
*   Write advanced `INSERT`, `UPDATE`, and `DELETE` statements, leveraging the `OUTPUT` clause for audit trails.
*   Understand the fundamental performance differences between `DELETE` and `TRUNCATE`.
*   Analyze how Entity Framework (EF) Core translates C# objects into SQL DML, specifically focusing on EF Core 7+ bulk update features.
*   Avoid the catastrophic failure of accidentally overwriting or deleting another tenant's data in a multi-tenant environment.

---

## 4.1 The INSERT Statement

The `INSERT` statement adds new rows to a table. In high-throughput IoT systems like our EV Charging SaaS, inserts are the most frequent operation.

### Basic Insert
```sql
INSERT INTO core.Stations (StationId, TenantId, MacAddress, Name)
VALUES (NEWID(), 'T1-UUID', '00:1A:2B:3C:4D:5E', 'Lobby Charger 1');
```

### Multi-Row Insert (Row Constructors)
SQL Server allows inserting multiple rows in a single statement. This is significantly faster than executing multiple individual `INSERT` statements because it reduces network round-trips and transaction log overhead.
```sql
INSERT INTO core.Ports (PortId, StationId, PortNumber, MaxKw)
VALUES 
    (NEWID(), 'S1-UUID', 1, 50.00),
    (NEWID(), 'S1-UUID', 2, 50.00);
```

### The `OUTPUT` Clause
In a distributed system, when you insert a row, you often need to know exactly what the database engine generated (e.g., default values, computed columns, or `IDENTITY`/`NEWSEQUENTIALID` keys). The `OUTPUT` clause returns data from the inserted (or updated/deleted) rows.

```sql
INSERT INTO core.Sessions (TenantId, StartTime, TotalKwh)
OUTPUT INSERTED.SessionId, INSERTED.StartTime
VALUES ('T1-UUID', SYSUTCDATETIME(), 0.00);
```
EF Core relies heavily on the `OUTPUT` clause behind the scenes to populate your C# entity's ID property after calling `SaveChanges()`.

---

## 4.2 The UPDATE Statement

The `UPDATE` statement modifies existing data. 

```sql
UPDATE core.Sessions
SET EndTime = SYSUTCDATETIME(),
    TotalKwh = 45.50,
    TotalCost = 12.75
WHERE SessionId = 'Session-UUID'
  AND TenantId = 'T1-UUID'; -- CRITICAL: Multi-Tenant Boundary
```

### Architect Perspective: The Multi-Tenant Filter
Notice the `AND TenantId = 'T1-UUID'` in the `WHERE` clause. 
If an API endpoint accepts a `PUT /api/sessions/{sessionId}` request, a malicious user could guess another tenant's `SessionId` and attempt to update it. If your SQL `UPDATE` statement only filters by `SessionId`, you have a massive security vulnerability (Insecure Direct Object Reference - IDOR). **Every UPDATE in a shared database must include the TenantId in the WHERE clause.**

---

## 4.3 DELETE vs. TRUNCATE

When you need to remove data, you have two options. Understanding the difference is vital for a Database Architect.

### The `DELETE` Statement
```sql
DELETE FROM core.Sessions 
WHERE StartTime < '2023-01-01';
```
*   **How it works:** `DELETE` removes rows one at a time. It records every single deleted row in the Transaction Log (LDF). 
*   **Performance:** Very slow for large datasets. If you delete 10 million rows, the transaction log will swell massively, potentially filling the disk and taking hours.
*   **Triggers & FKs:** `DELETE` fires triggers and enforces Foreign Key constraints.

### The `TRUNCATE` Statement
```sql
TRUNCATE TABLE core.AuditLogs;
```
*   **How it works:** `TRUNCATE` does not log individual row deletions. Instead, it deallocates the 8KB data pages that house the table's data. 
*   **Performance:** Instantaneous, regardless of whether the table has 10 rows or 10 billion rows.
*   **Limitations:** You **cannot** use `TRUNCATE` on a table that is referenced by a Foreign Key (even if the child table is empty). It also does not fire `DELETE` triggers. It is generally used for staging tables or clearing out massive log tables during maintenance.

---

## 4.4 The Code: EF Core Translation

How does EF Core handle these commands in our SaaS?

### The N+1 Insert Problem
If you loop through a list of 1,000 new IoT telemetry events and call `context.Add(event)` followed by `context.SaveChanges()` inside the loop, EF Core will execute 1,000 separate `INSERT` statements over the network. This will crush performance.
**Solution:** Call `context.AddRange(events)` and call `SaveChanges()` *once* outside the loop. EF Core will automatically batch the `INSERT` statements into optimal chunks (usually 42 statements per network trip).

### EF Core 7+ Bulk Updates and Deletes
Historically, to update 1,000 rows in EF Core, you had to load all 1,000 rows into memory (RAM), modify their properties in C#, and call `SaveChanges()`. 
Since EF Core 7, we can execute bulk `UPDATE` and `DELETE` directly against the database without loading entities into memory using `ExecuteUpdateAsync` and `ExecuteDeleteAsync`.

```csharp
// Scenario: A Station goes offline. We must force-complete all active sessions.
public async Task ForceCompleteSessionsAsync(Guid tenantId, Guid stationId)
{
    // Executes a single, highly optimized SQL UPDATE statement directly.
    // No data is loaded into application memory.
    await _context.Sessions
        .Where(s => s.TenantId == tenantId && 
                    s.Port.StationId == stationId && 
                    s.EndTime == null)
        .ExecuteUpdateAsync(s => s
            .SetProperty(p => p.EndTime, DateTime.UtcNow)
            .SetProperty(p => p.Status, "ForceCompleted"));
}
```
*Architect Note:* `ExecuteUpdate` and `ExecuteDelete` bypass the EF Core Change Tracker. They are pure SQL translations.

---

## 4.5 Performance & Security Analysis

### Performance Analysis
*   **The UPDATE Lock Escalation:** If you run an `UPDATE` statement that affects hundreds of thousands of rows (e.g., updating a new `DiscountRate` column for an entire tenant), SQL Server will quickly run out of memory for individual Row Locks. It will perform a **Lock Escalation**, locking the *entire table*. During this time, no other tenant can insert charging sessions. For massive updates, architects must batch the updates in chunks of ~4,000 rows.

### Security Implications
*   **The Missing WHERE Clause:** Executing `UPDATE core.Users SET IsActive = 0;` (without a WHERE clause) will deactivate every user in the system across all tenants. This is a resume-generating event (RGE). In production environments, tooling (like SSMS or Azure Data Studio) should have execution warnings enabled, and application code must have strict integration tests verifying tenant isolation.

---

## 4.6 Common Mistakes & Production Pitfalls

1.  **Using `DELETE` for Archiving:** When archiving 5 years of old SaaS data, developers often write a script to `DELETE` millions of rows. This causes the Transaction Log to explode, filling the `C:\` drive and crashing the server. Large deletions must be done in batches (e.g., `DELETE TOP (5000)`) in a loop, or by using Table Partition switching.
2.  **Updating Primary Keys:** Never update a Primary Key. If a business requirement changes an entity's identity, delete the old entity and insert a new one. Updating a PK requires cascading updates to all foreign keys and shuffles the physical clustered index on disk.

---

## 4.7 Production Checklist

*   [ ] Multi-tenant isolation (`TenantId`) is strictly enforced in the `WHERE` clause of every `UPDATE` and `DELETE` statement.
*   [ ] Batching is used in EF Core (`AddRange`, `ExecuteUpdate`) for processing large volumes of data.
*   [ ] Mass deletions (archiving) are batched to prevent Transaction Log bloat.
*   [ ] Database backups (Transaction Log backups) run frequently enough to clear the LDF file during heavy insert/delete operations.

---

## 4.8 Exercises

1.  **Bulk Deletion:** Write a T-SQL script using a `WHILE` loop to delete 1,000,000 rows from `core.AuditLogs` where `CreatedAt < '2023-01-01'`, deleting them in chunks of 5,000 rows to prevent locking the table.
2.  **EF Core Translation:** Write the C# EF Core 7+ code using `ExecuteDeleteAsync` to delete all `Sessions` for a specific `TenantId` that have `TotalKwh = 0` (failed starts).

---

## 4.9 Interview Questions

**Q1: Explain the difference between `DELETE` and `TRUNCATE`. If you accidentally run both without a WHERE clause (TRUNCATE doesn't support WHERE), which one is easier to recover from, assuming an active transaction?**
*Answer:* `DELETE` is a DML operation that logs individual row deletions and fires triggers. `TRUNCATE` is a DDL operation that deallocates data pages and is minimally logged. However, *both* can be rolled back if they are wrapped in an explicit `BEGIN TRAN`. If there is no transaction, recovering from a `DELETE` might be possible via transaction log scraping (very difficult), whereas `TRUNCATE` page deallocations are much harder to reconstruct.

**Q2: In EF Core 7+, what is the difference between updating an entity by loading it, changing a property, and calling `SaveChanges()`, versus using `ExecuteUpdateAsync()`?**
*Answer:* Loading the entity tracks it in the EF Core Change Tracker. This requires a `SELECT` query (network round trip), memory allocation for the object, and then an `UPDATE` query when `SaveChanges()` is called. `ExecuteUpdateAsync()` bypasses the change tracker and memory entirely, translating the LINQ expression directly into a single SQL `UPDATE` statement, making it exponentially faster for bulk operations.

---
**Next up in Chapter 5:** We move into Part 2: Advanced Querying. We will cover the `SELECT` statement, filtering, and the critical concept of "Sargability"—understanding why certain `WHERE` clauses silently destroy database performance.
