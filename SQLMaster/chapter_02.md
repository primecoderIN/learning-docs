# Chapter 2: SQL Fundamentals & Data Types

## Learning Objectives
By the end of this chapter, you will be able to:
*   Categorize SQL statements into DDL, DML, DCL, and TCL.
*   Select the exact, most optimal data types for a global SaaS application, particularly regarding EV charging metrics and timezones.
*   Analyze the severe performance implications of standard `UNIQUEIDENTIFIER` (GUID) columns on Clustered Indexes.
*   Implement distributed ID generation strategies (e.g., Sequential GUIDs) to scale beyond the limitations of `IDENTITY(1,1)`.
*   Calculate the hidden financial and performance costs of oversized data types in a billion-row database.

---

## 2.1 The Core SQL Languages

While ORMs like Entity Framework (EF) Core abstract much of the raw SQL, a database architect must speak the native language of the engine. SQL is divided into four distinct sub-languages:

1.  **DDL (Data Definition Language):** Defines the structure (schema).
    *   `CREATE`, `ALTER`, `DROP`.
    *   *Example:* `CREATE TABLE core.Stations (...)`
2.  **DML (Data Manipulation Language):** Manipulates the data within the structures.
    *   `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`.
    *   *Note:* In SQL Server, `SELECT` is strictly considered DML.
3.  **DCL (Data Control Language):** Manages security and permissions.
    *   `GRANT`, `REVOKE`, `DENY`.
4.  **TCL (Transaction Control Language):** Manages transactions to ensure ACID compliance.
    *   `BEGIN TRAN`, `COMMIT`, `ROLLBACK`.

EF Core handles DML transparently during normal operations and uses DDL during Migrations. However, EF Core rarely touches DCL or TCL natively without custom extensions.

---

## 2.2 Data Types: The Foundation of Performance

Remember from Chapter 1 that SQL Server stores data in 8KB Pages. Every byte counts. If you choose `BIGINT` (8 bytes) when `INT` (4 bytes) would suffice, you have immediately cut your page density in half, doubling the disk I/O required for large scans.

Let's design the optimal data types for our EV Charging SaaS, **VoltCore**.

### Strings: VARCHAR vs. NVARCHAR
*   `VARCHAR(N)`: Uses 1 byte per character. Suitable for ASCII text.
*   `NVARCHAR(N)`: Uses 2 bytes per character. Supports Unicode (Emojis, international characters).

*Architect Rule:* For user-facing strings (Names, Descriptions), use `NVARCHAR`. For system-generated strings (MAC Addresses, RFID tags, API Keys), strictly use `VARCHAR(X)` to save 50% space. 

```sql
-- DDL for our Station table
CREATE TABLE core.Stations (
    StationId UNIQUEIDENTIFIER NOT NULL,
    -- MAC Addresses are always 17 chars (e.g., 00:1A:2B:3C:4D:5E)
    -- Using VARCHAR(17) instead of NVARCHAR(MAX) saves immense space.
    MacAddress VARCHAR(17) NOT NULL 
);
```

### Numerics: DECIMAL vs. FLOAT (The EV Power Metrics)
In EV charging, we track Kilowatt-hours (kWh) and billing currency.
*   `FLOAT`: Approximate-number data type. Fast, but prone to rounding errors (e.g., `0.1 + 0.2 = 0.30000000000000004`). **Never** use `FLOAT` for money or exact telemetry.
*   `DECIMAL(p, s)`: Exact-number data type. `p` (precision) is total digits, `s` (scale) is digits after the decimal.

For charging sessions, standard OCPP hardware reports energy to the 4th decimal place.
```sql
-- TotalKwh can store up to 999999.9999 (9 bytes of storage)
TotalKwh DECIMAL(10, 4) NOT NULL,
-- Currency calculation
TotalCost DECIMAL(19, 4) NOT NULL 
```

### Dates: DATETIME2 vs. DATETIMEOFFSET
In a global SaaS, timezones will destroy your sanity.
*   `DATETIME`: The legacy type. 3.33ms accuracy. Do not use.
*   `DATETIME2(7)`: 100ns accuracy. Stores only the date and time.
*   `DATETIMEOFFSET`: Stores the date, time, and the UTC offset (e.g., `2024-10-15 14:00:00 -08:00`).

*Enterprise Best Practice:* Always store timestamps in absolute **UTC** using `DATETIME2(3)` (millisecond accuracy is enough for IoT, saving 2 bytes over precision 7). Handle timezone display strictly in the UI layer (React/Next.js) using the user's browser timezone.

---

## 2.3 The Primary Key Dilemma: UUIDs vs. INTs

This is the most critical architectural decision in a multi-tenant SaaS. How do we generate Primary Keys?

### The Problem with `IDENTITY(1, 1)` (Auto-Incrementing Integers)
Historically, developers used auto-incrementing `INT` or `BIGINT`. 
*   **Pros:** It is sequential (1, 2, 3), which is perfect for SQL Server's B-Tree Clustered Indexes (no page splits!). It is small (4 or 8 bytes).
*   **Cons:** In a distributed, Microservices, or offline-capable SaaS, the client cannot know the ID of an entity until it `INSERT`s it into the central DB. This prevents clients from building nested object graphs (e.g., a Station with Ports) before sending them to the server. Furthermore, integers are predictable, making API enumeration attacks easy (e.g., `GET /api/tenants/5`).

### The Problem with Standard `UNIQUEIDENTIFIER` (GUID)
To solve the distributed generation problem, architects switch to GUIDs (UUID v4).
*   *Example:* `F9168C5E-CEB2-4faa-B6BF-329BF39FA1E4`
*   **Pros:** Can be generated anywhere (C#, React) instantly. Globally unique.
*   **Cons (The Silent Killer):** GUIDs are completely random.

SQL Server physical tables are ordered on disk by the **Clustered Index** (which is usually the Primary Key). 
If you insert rows with random GUIDs, SQL Server must physically insert the new row *between* existing rows on disk. 
Because the 8KB page is likely full, this triggers a **Page Split**:
1.  SQL Server pauses the transaction.
2.  It allocates a brand new 8KB page.
3.  It moves 50% of the rows from the old page to the new page.
4.  It updates the B-Tree pointers.

In a system processing 10,000 EV sessions per minute, random GUIDs will cause catastrophic Page Splits, driving disk I/O to 100%, fragmenting the index to 99%, and bringing the database to its knees.

### The Architect's Solution: Sequential GUIDs
We need the best of both worlds: Distributed generation *and* sequential ordering.
In SQL Server, we achieve this using **Sequential GUIDs**.

*   **Database Level:** SQL Server provides `NEWSEQUENTIALID()` as a default value constraint. It generates a GUID that is greater than any previously generated GUID on that server.
*   **Application Level (UUID v7 or HiLo):** Using modern .NET, we can generate UUIDv7 (Time-ordered UUIDs).

```sql
-- The perfect SaaS Table Definition
CREATE TABLE core.Sessions (
    -- 16 bytes, but inserts sequentially at the end of the B-Tree!
    SessionId UNIQUEIDENTIFIER CONSTRAINT DF_SessionId DEFAULT NEWSEQUENTIALID() NOT NULL,
    TenantId UNIQUEIDENTIFIER NOT NULL,
    StartTime DATETIME2(3) NOT NULL,
    TotalKwh DECIMAL(10, 4) NOT NULL,
    
    CONSTRAINT PK_Sessions PRIMARY KEY CLUSTERED (SessionId ASC)
);
```

---

## 2.4 Architect Perspective: The Billion-Row Cost

Let's calculate the "Hidden Cost of Oversized Data Types".
Imagine VoltCore scales to 1,000,000,000 (1 Billion) charging sessions.

**Scenario A (Poor Design):**
*   `MacAddress NVARCHAR(50)` -> 100 bytes
*   `TotalCost FLOAT` -> 8 bytes
*   `StartTime DATETIME` -> 8 bytes
*   Total Row Size (excluding headers): ~116 bytes.
*   Total Storage: 116 GB.

**Scenario B (Architect Design):**
*   `MacAddress VARCHAR(17)` -> 17 bytes
*   `TotalCost DECIMAL(9,4)` -> 5 bytes
*   `StartTime DATETIME2(3)` -> 7 bytes
*   Total Row Size: ~29 bytes.
*   Total Storage: 29 GB.

By simply picking the correct data types, you saved **87 GB of RAM** requirement (for the buffer pool) and reduced disk I/O by 75%. *That* is database engineering.

---

## 2.5 The Code: EF Core Configurations

To enforce these exact data types in ASP.NET Core, we do not rely on Data Annotations. We use the **Fluent API** in `IEntityTypeConfiguration` to maintain clean POCOs.

```csharp
// Persistence/Configurations/SessionConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class SessionConfiguration : IEntityTypeConfiguration<Session>
{
    public void Configure(EntityTypeBuilder<Session> builder)
    {
        builder.ToTable("Sessions", "core");

        // Use sequential GUIDs in SQL Server to prevent index fragmentation
        builder.HasKey(x => x.SessionId);
        builder.Property(x => x.SessionId)
               .HasDefaultValueSql("NEWSEQUENTIALID()");

        // Exact precision for money and energy
        builder.Property(x => x.TotalKwh)
               .HasColumnType("DECIMAL(10,4)")
               .IsRequired();

        builder.Property(x => x.TotalCost)
               .HasColumnType("DECIMAL(19,4)")
               .IsRequired();

        // Optimize DateTime2 precision
        builder.Property(x => x.StartTime)
               .HasColumnType("DATETIME2(3)");
    }
}
```

---

## 2.6 Performance & Security Analysis

### Performance Analysis
*   **Index Fragmentation:** A clustered index built on a random `UUID v4` will quickly hit 95%+ fragmentation. This causes the database to read partially empty pages, wasting memory. Rebuilding indexes locks tables. Sequential GUIDs prevent this fragmentation entirely.
*   **Network Payload:** Sending `DECIMAL(19,4)` instead of `FLOAT` ensures that billing APIs do not suffer from JSON deserialization rounding bugs when transferring data from the .NET backend to a Next.js frontend.

### Security Implications
*   **Predictability of NEWSEQUENTIALID:** The GUIDs generated by `NEWSEQUENTIALID()` are sequential and therefore predictable. If a User ID is sequential, an attacker might guess the next ID. 
*   **Mitigation:** In SaaS, never rely on ID obscurity for security. You must authorize *every* request (e.g., "Does User A have permission to read Session X?"). We handle this via `TenantId` partitioning and ASP.NET Authorization Policies.

---

## 2.7 Common Mistakes & Production Pitfalls

1.  **Using `NVARCHAR(MAX)` for everything:** Developers often use `string` in C# without defining a `MaxLength` in EF Core. EF Core defaults to `NVARCHAR(MAX)`, which forces the database engine to store data off-row in LOB (Large Object) storage, drastically slowing down sorting and indexing.
2.  **Using `DATETIME.NOW`:** Inserting local server time into the database. When your SaaS moves to an Azure region in Europe, all your timestamps will shift. Always use `DateTime.UtcNow`.
3.  **Client-Side GUID Generation:** Using `Guid.NewGuid()` in C# and passing it to EF Core bypasses `NEWSEQUENTIALID()`, resulting in random inserts and page splits.

---

## 2.8 Production Checklist

*   [ ] Primary Keys on heavy-insert tables use Sequential GUIDs (`NEWSEQUENTIALID` or UUIDv7), not random GUIDs.
*   [ ] Currency/Billing columns use `DECIMAL(19,4)`, never `FLOAT`.
*   [ ] ASCII-only data (MAC addresses, Hashes, API keys) explicitly use `VARCHAR` to cut storage in half.
*   [ ] All timestamps are stored in UTC using `DATETIME2(3)`.

---

## 2.9 Exercises

1.  **Schema Refactoring:** A junior developer created a table for `RfidTags` (used by drivers to swipe at stations). The schema is: 
    `CREATE TABLE RfidTags (TagId INT IDENTITY(1,1), TagHex NVARCHAR(MAX), CreatedAt DATETIME)`
    Rewrite this `CREATE TABLE` statement using enterprise best practices for a multi-tenant SaaS.
2.  **Storage Calculation:** How many bytes does `VARCHAR(50)` consume if it stores the string "HELLO" (5 characters)? How many bytes does `NVARCHAR(50)` consume for the same string?

---

## 2.10 Interview Questions

**Q1: Why is using a standard `Guid.NewGuid()` in C# as a Primary Key in SQL Server considered an anti-pattern for performance?**
*Answer:* Standard GUIDs (UUID v4) are cryptographically random. SQL Server orders the physical table based on the Clustered Index (the Primary Key). Inserting random values forces SQL Server to constantly insert rows in the middle of existing 8KB pages. This causes Page Splits, severe index fragmentation, increased transaction log generation, and massive disk I/O. 

**Q2: In a financial billing system for EV charging, why must we use `DECIMAL` instead of `FLOAT`?**
*Answer:* `FLOAT` is an approximate data type based on floating-point math. It cannot accurately represent all base-10 decimals, leading to rounding errors in aggregation (e.g., 0.1 + 0.2 = 0.30000000000000004). `DECIMAL` is an exact-number data type that guarantees precision and scale, which is legally required for financial and billing calculations.

---
**Next up in Chapter 3:** We will dive into Constraints, Data Integrity, and enforcing the domain rules of our EV SaaS directly at the database level to protect against bad code deployments.
