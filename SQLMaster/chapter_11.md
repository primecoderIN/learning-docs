# Chapter 11: Advanced DML: MERGE & UPSERT

## Learning Objectives
By the end of this chapter, you will be able to:
*   Define the "UPSERT" pattern (Update if exists, Insert if not).
*   Write complex `MERGE` statements in T-SQL to synchronize data from external IoT endpoints (OCPP).
*   Compare SQL Server's `MERGE` with PostgreSQL's `ON CONFLICT DO UPDATE` syntax.
*   Identify and fix severe concurrency bugs (race conditions) inherent to the `MERGE` statement under high load using the `HOLDLOCK` hint.
*   Implement UPSERT logic safely via Entity Framework Core.

---

## 11.1 The UPSERT Problem

In our EV SaaS, charging stations communicate with our backend using OCPP (Open Charge Point Protocol). A station might send a "Boot Notification" or a "Heartbeat". 
When we receive this payload, the business requirement is:
*   If we already have a record for this Station, **Update** its `LastSeen` timestamp and `FirmwareVersion`.
*   If we do not have a record for this Station (e.g., a technician just installed it), **Insert** a new row.

Historically, developers solved this by executing an `IF EXISTS` check, followed by an `UPDATE` or `INSERT`.

```sql
-- The old, slow way (Multiple network trips, prone to race conditions)
IF EXISTS (SELECT 1 FROM core.Stations WHERE MacAddress = 'AA:BB:CC')
BEGIN
    UPDATE core.Stations SET LastSeen = GETUTCDATE() WHERE MacAddress = 'AA:BB:CC';
END
ELSE
BEGIN
    INSERT INTO core.Stations (StationId, MacAddress, LastSeen) 
    VALUES (NEWID(), 'AA:BB:CC', GETUTCDATE());
END
```
This approach requires two separate physical reads/writes and is highly susceptible to race conditions.

---

## 11.2 The T-SQL MERGE Statement

SQL Server introduced the `MERGE` statement to perform INSERT, UPDATE, and DELETE operations in a single, unified command.

### The MERGE Syntax
`MERGE` requires a **Target** (the table you are modifying) and a **Source** (the incoming data).

```sql
MERGE core.Stations AS Target
USING (
    -- The incoming data from our API (The Source)
    SELECT 
        'T1-UUID' AS TenantId,
        'AA:BB:CC' AS MacAddress,
        '1.0.5' AS FirmwareVersion,
        SYSUTCDATETIME() AS LastSeen
) AS Source
ON (Target.MacAddress = Source.MacAddress AND Target.TenantId = Source.TenantId)

-- Condition 1: The row exists
WHEN MATCHED THEN
    UPDATE SET 
        Target.FirmwareVersion = Source.FirmwareVersion,
        Target.LastSeen = Source.LastSeen

-- Condition 2: The row does NOT exist
WHEN NOT MATCHED BY TARGET THEN
    INSERT (StationId, TenantId, MacAddress, FirmwareVersion, LastSeen)
    VALUES (NEWID(), Source.TenantId, Source.MacAddress, Source.FirmwareVersion, Source.LastSeen);
```
*(Note: A `MERGE` statement in SQL Server MUST be terminated with a semicolon `;`)*

---

## 11.3 PostgreSQL vs. SQL Server UPSERT

As an Architect, you should understand how different engines handle this pattern, as you may eventually port the SaaS to AWS Aurora (PostgreSQL).

PostgreSQL does not use `MERGE`. It uses an incredibly elegant extension to the `INSERT` statement called `ON CONFLICT`.

```sql
-- PostgreSQL Syntax (For comparison)
INSERT INTO core.stations (station_id, tenant_id, mac_address, firmware_version, last_seen)
VALUES (gen_random_uuid(), 'T1-UUID', 'AA:BB:CC', '1.0.5', NOW())
ON CONFLICT (tenant_id, mac_address) 
DO UPDATE SET 
    firmware_version = EXCLUDED.firmware_version,
    last_seen = EXCLUDED.last_seen;
```
This is generally considered safer and easier to read than T-SQL's `MERGE`.

---

## 11.4 Architect Perspective: Concurrency and Race Conditions

Here is the most critical lesson in this chapter: **The T-SQL `MERGE` statement is NOT inherently thread-safe.**

Imagine two identical OCPP Boot Notifications arrive at your API at the exact same millisecond.
1.  Thread A executes `MERGE`. It checks `ON (Target = Source)`. It finds no match.
2.  Thread B executes `MERGE`. It checks `ON (Target = Source)`. It finds no match.
3.  Thread A executes the `INSERT`.
4.  Thread B executes the `INSERT`.

Result: **Primary Key Violation Error (or duplicated data if no Unique Constraint exists).**

### The Fix: `HOLDLOCK` (Serializable Isolation)
To make `MERGE` safe for high-concurrency IoT applications, you must force SQL Server to take a lock on the key range during the initial read, and hold that lock until the transaction completes.

You do this by adding the `WITH (HOLDLOCK)` hint to the Target table.

```sql
MERGE core.Stations WITH (HOLDLOCK) AS Target
USING (...)
```
This briefly escalates the transaction isolation level to Serializable just for this statement, guaranteeing thread safety at the cost of slight concurrency reduction.

---

## 11.5 The Code: EF Core Upsert Implementations

Entity Framework Core (as of .NET 8) does not have a native, built-in `.Upsert()` method.

**Option 1: The Raw SQL approach**
The safest and fastest way to execute a high-throughput UPSERT is to write the `MERGE` statement via EF Core's `ExecuteSqlRawAsync`.

```csharp
public async Task UpsertStationAsync(StationDto dto)
{
    var sql = @"
        MERGE core.Stations WITH (HOLDLOCK) AS Target
        USING (SELECT @TenantId AS TenantId, @MacAddress AS Mac) AS Source
        ON (Target.TenantId = Source.TenantId AND Target.MacAddress = Source.Mac)
        WHEN MATCHED THEN 
            UPDATE SET LastSeen = GETUTCDATE()
        WHEN NOT MATCHED THEN 
            INSERT (StationId, TenantId, MacAddress, LastSeen) 
            VALUES (NEWID(), Source.TenantId, Source.Mac, GETUTCDATE());";

    await _context.Database.ExecuteSqlRawAsync(sql, 
        new SqlParameter("TenantId", dto.TenantId),
        new SqlParameter("MacAddress", dto.MacAddress));
}
```

**Option 2: Third-Party Libraries**
For enterprise applications preferring LINQ, architects often install libraries like `linq2db.EntityFrameworkCore` or `EFCore.BulkExtensions`, which provide a `.BulkInsertOrUpdateAsync()` method that generates the `MERGE` statement under the hood.

---

## 11.6 Performance & Security Analysis

### Performance Analysis
*   **The Cost of HOLDLOCK:** While `HOLDLOCK` prevents Primary Key violations, it places a RangeS-S (Range Shared) lock on the index. If hundreds of IoT devices are hammering the exact same index range simultaneously, this can lead to locking contention or Deadlocks (which we will cover in Chapter 17). Ensure the `ON` clause utilizes a highly selective, perfectly covered index.

### Security Implications
*   **WHEN NOT MATCHED BY SOURCE:** The `MERGE` statement has a third, optional clause: `WHEN NOT MATCHED BY SOURCE THEN DELETE`. This means if a row exists in the Target (Database) but is *missing* from the Source (API Payload), the database will delete it. 
*   **Danger:** If an API endpoint accepts an array of Stations to sync, and a bug causes the client to send an empty array `[]`, the `MERGE` statement will assume all stations have been removed and will **delete the entire Tenant's fleet**. Architects rarely use the `DELETE` clause in a `MERGE` unless syncing against a strictly controlled, infallible golden source.

---

## 11.7 Common Mistakes & Production Pitfalls

1.  **Forgetting the Semicolon:** `MERGE` is the only DML statement in T-SQL that absolutely requires a terminating semicolon `;`. Omitting it causes a syntax error.
2.  **Updating the Join Key:** You cannot update a column that is referenced in the `ON` clause. If you need to change a `MacAddress`, it must be a `DELETE` followed by an `INSERT`, not an `UPDATE`.
3.  **Missing TenantId in the ON clause:** If you `MERGE ... ON Target.MacAddress = Source.MacAddress`, you might accidentally overwrite Station data belonging to Tenant B with data coming from Tenant A, breaching data isolation.

---

## 11.8 Production Checklist

*   [ ] `MERGE` statements operating under high concurrency explicitly include the `WITH (HOLDLOCK)` hint on the Target table.
*   [ ] The `ON` clause of the `MERGE` statement strictly includes the `TenantId` to guarantee tenant isolation.
*   [ ] `WHEN NOT MATCHED BY SOURCE THEN DELETE` is completely avoided unless implementing a full-state synchronization pattern with extensive safety guards.
*   [ ] `MERGE` statements are properly parameterized to prevent SQL Injection.

---

## 11.9 Exercises

1.  **Syntax Checking:** Identify the two critical errors in the following `MERGE` statement intended for a multi-tenant application handling 5,000 requests per second:
    ```sql
    MERGE core.Users AS Target
    USING (SELECT 'T1' as TenantId, 'user@test.com' as Email) AS Source
    ON Target.Email = Source.Email
    WHEN MATCHED THEN UPDATE SET Target.LastLogin = GETUTCDATE()
    ```
2.  **PostgreSQL Port:** Rewrite the fixed SQL Server `MERGE` statement from Exercise 1 using PostgreSQL's `ON CONFLICT DO UPDATE` syntax.

---

## 11.10 Interview Questions

**Q1: A Junior Developer wrote a `MERGE` statement to UPSERT user profiles. Under high load, the database is throwing "Violation of PRIMARY KEY constraint" errors. Why is this happening, and how do you fix it?**
*Answer:* The `MERGE` statement in SQL Server is not atomically thread-safe by default. Two concurrent threads can both evaluate the `ON` clause, find no matching row, and both attempt to execute the `INSERT` clause simultaneously, causing a PK violation. To fix this, you must apply the `WITH (HOLDLOCK)` table hint to the Target table to serialize the read and write operations.

**Q2: Compare the traditional `IF EXISTS (...) UPDATE ELSE INSERT` pattern with the `MERGE` statement. Why is `MERGE` preferred?**
*Answer:* The traditional `IF EXISTS` pattern requires multiple network round trips and multiple context switches within the database engine (a read, followed by a separate write). `MERGE` consolidates this into a single atomic statement, generating a single execution plan, reducing network latency, and providing a cleaner, declarative syntax.

---
**Next up in Chapter 12:** We transition to Part 4 (Programmability). We will explore how to abstract complex schema designs from the application layer using **Views** and how to vastly improve dashboard performance using **Indexed (Materialized) Views**.
