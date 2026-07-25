# Chapter 36: In-Memory OLTP (Hekaton)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the physical limits of Disk-Based tables (Buffer Pool page latch contention).
*   Architect **Memory-Optimized Tables** to completely eliminate locking and blocking.
*   Understand the performance benefits of **Natively Compiled Stored Procedures** (Hekaton).
*   Configure Entity Framework Core to seamlessly map to In-Memory tables.
*   Evaluate the architectural risks of In-Memory OLTP (OOM Exceptions).

---

## 36.1 The Physical Limits of Disk-Based Tables

Throughout this book, we have optimized Disk-Based tables. We minimized reads, added Covering Indexes, and used `NOLOCK`/RCSI.

However, even if a table is fully cached in RAM (the Buffer Pool), SQL Server still enforces mechanical structures. Data is stored on 8KB pages. When 100 API threads try to `INSERT` a row into the same table simultaneously, they all target the exact same 8KB page at the end of the Clustered Index.
SQL Server uses **Page Latches** (micro-locks) to ensure two threads don't corrupt the page memory. The threads queue up, waiting for the latch.
If you need to process 50,000 inserts per second (e.g., massive IoT telemetry storms), Page Latch contention becomes a physical bottleneck that no amount of tuning can solve.

You must bypass the Buffer Pool entirely.

---

## 36.2 Introduction to In-Memory OLTP (Hekaton)

In SQL Server 2014, Microsoft introduced **In-Memory OLTP** (Project Hekaton).

It is a completely separate database engine living *inside* SQL Server. 
*   It does not use 8KB pages.
*   It does not use Buffer Pools.
*   It does not use B-Tree indexes (it uses Hash Indexes and Bw-Trees).
*   **It uses zero locks and zero latches.**

When you create a **Memory-Optimized Table**, the data lives permanently in raw RAM as C-style structs. Multiple threads can insert data simultaneously with absolutely zero contention.

---

## 36.3 Memory-Optimized Tables

Creating an In-Memory table requires a specific Filegroup, but the syntax is straightforward.

```sql
-- 1. Must add a specialized MEMORY_OPTIMIZED_DATA filegroup to the DB
ALTER DATABASE VoltCore ADD FILEGROUP HekatonFG CONTAINS MEMORY_OPTIMIZED_DATA;

-- 2. Create the Table
CREATE TABLE core.Telemetry_InMemory (
    TelemetryId UNIQUEIDENTIFIER NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000000) PRIMARY KEY,
    StationId UNIQUEIDENTIFIER,
    Metric VARCHAR(100),
    CreatedAt DATETIME2
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
```

### Durability Modes
*   **SCHEMA_AND_DATA:** The default. While the data lives in RAM, SQL Server asynchronously writes a continuous stream of changes to the Transaction Log. If the server reboots, it rebuilds the RAM state from the log. No data is lost.
*   **SCHEMA_ONLY:** Data is never written to disk. If the server reboots, the table is instantly truncated. This is incredibly fast and is perfect for massive, transient ASP.NET Session State or temporary staging tables.

---

## 36.4 Natively Compiled Stored Procedures

If the table lives in raw RAM, the traditional SQL Server Query Optimizer (which translates T-SQL into execution plans) becomes the slowest part of the process.

Hekaton allows you to write **Natively Compiled Stored Procedures**.
When you execute the `CREATE PROCEDURE` script, SQL Server literally translates your T-SQL into C code, compiles it into a `.dll` file using a built-in C compiler, and loads it into the SQL Server memory space.

Executing the procedure is indistinguishable from executing a native C++ function. Latency drops from milliseconds to *microseconds*.

```sql
CREATE PROCEDURE core.usp_InsertTelemetry 
    @TelemetryId UNIQUEIDENTIFIER, @StationId UNIQUEIDENTIFIER, @Metric VARCHAR(100)
WITH NATIVE_COMPILATION, SCHEMABINDING, EXECUTE AS OWNER
AS
BEGIN ATOMIC WITH (TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N'us_english')
    INSERT INTO core.Telemetry_InMemory (TelemetryId, StationId, Metric, CreatedAt)
    VALUES (@TelemetryId, @StationId, @Metric, GETUTCDATE());
END
```

---

## 36.5 Architect Perspective: The Memory Trap

In-Memory OLTP sounds like magic. Why don't we use it for every table?

**The Trap:** RAM is finite.
If your standard Disk-Based table grows to 500GB, SQL Server simply streams data on and off the hard drive. If a Memory-Optimized table hits 500GB, and your server only has 256GB of RAM, the database throws an **Out Of Memory (OOM) Exception**.
Worse, in Azure SQL, hitting your memory tier limit causes the database engine to forcefully reject all new `INSERT` and `UPDATE` statements, crashing your API.

*Architect Rule:* Only use Memory-Optimized tables for highly transient, extremely high-throughput tables. You must pair them with a strict archiving job (or use Temporal Tables) to move old data off to Disk-Based tables constantly.

---

## 36.6 The Code: EF Core Integration

Entity Framework Core maps to Memory-Optimized tables effortlessly. Because EF Core relies on standard T-SQL for inserts/updates, it is completely unaware that the table lives in Hekaton.

The only difference is how you configure the Migration. EF Core has a fluent API extension to mark a table as Memory-Optimized.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Telemetry>()
        .ToTable("Telemetry", "core")
        .IsMemoryOptimized(); // Tells the Migration builder to append the WITH clause
}
```

*Note: You cannot use traditional `IDENTITY` columns effectively in Memory-Optimized tables without severe performance penalties. Always use `UNIQUEIDENTIFIER` (Guid) generated in the C# application to ensure zero contention.*

---

## 36.7 Performance & Security Analysis

### Performance Analysis: Hash Indexes
Memory-Optimized tables do not support standard B-Trees. They use **Hash Indexes**. 
A Hash Index is an array of "buckets." You must specify a `BUCKET_COUNT` when creating the index. 
*   If you set the bucket count too low (e.g., 1,000 buckets for 1,000,000 rows), you get massive **Hash Collisions**. The engine has to chain rows together, destroying performance.
*   *Rule of Thumb:* Set the `BUCKET_COUNT` to 1.5x to 2x the maximum number of unique values you ever expect the column to hold.

### Security Implications
*   **Cross-Database Transactions:** In-Memory OLTP strictly forbids cross-database transactions. If your security architecture relies on an API initiating a Distributed Transaction (DTC) that updates a Disk-Based table in Database A and an In-Memory table in Database B, it will instantly fail.

---

## 36.8 Common Mistakes & Production Pitfalls

1.  **Unsupported Features:** Memory-Optimized tables do not support Foreign Keys linking to Disk-Based tables. You cannot have a `Telemetry_InMemory` row enforce an FK constraint against the standard `Stations` table on disk. You must enforce relational integrity in the application layer.
2.  **Schema Modifications:** In older versions of SQL Server, altering an In-Memory table required dropping and recreating the entire table. Modern versions support `ALTER TABLE`, but it executes entirely offline, pausing all operations against that table. Schema migrations on Hekaton are dangerous in production.

---

## 36.9 Production Checklist

*   [ ] Memory-Optimized tables are exclusively used for "shock absorber" tables (high-velocity inserts) or session state.
*   [ ] A strict archival mechanism (or System-Versioned Temporal Table) is implemented to continuously flush In-Memory data to disk, preventing OOM exceptions.
*   [ ] Hash Indexes are configured with a `BUCKET_COUNT` appropriately sized for the expected cardinality of the column.
*   [ ] All `INSERT` logic utilizes C# client-side generated GUIDs rather than database `IDENTITY` columns to maximize thread concurrency.

---

## 36.10 Exercises

1.  **Bottleneck Identification:** You have a Disk-Based table receiving 20,000 inserts per second. The CPU is at 20%, and the SSD IOPS are at 10%. However, the inserts are timing out. You check Wait Statistics (Chapter 31) and see massive `PAGELATCH_EX` waits. Explain why a larger SSD will not fix this, and why In-Memory OLTP is the correct architectural choice.
2.  **Schema Design:** You need to store ASP.NET Core API Session Tokens. The tokens expire every 20 minutes, and it is acceptable if users have to log in again if the server reboots. Write the T-SQL to create this Memory-Optimized table using the correct `DURABILITY` mode.

---

## 36.11 Interview Questions

**Q1: What physical bottleneck of traditional relational tables does In-Memory OLTP (Hekaton) solve?**
*Answer:* Traditional tables are stored on 8KB pages within the Buffer Pool. When multiple threads attempt to insert or update rows on the same physical page, SQL Server enforces Page Latches (micro-locks) to prevent memory corruption. Under massive concurrency (e.g., IoT telemetry), this causes severe latch contention and blocking, regardless of SSD speed. Hekaton solves this by abandoning 8KB pages and locks entirely, storing data as C-style structs in raw RAM, allowing infinite concurrency.

**Q2: What is a Natively Compiled Stored Procedure, and why is it faster than standard T-SQL?**
*Answer:* A Natively Compiled Stored Procedure is a T-SQL script that SQL Server translates into C code, compiles into a physical Windows `.dll` file, and loads directly into the database engine's memory space. It is drastically faster because it bypasses the entire SQL Server Query Execution engine (compilation, parsing, interpretation). It executes with the raw speed of a native C++ application against data already residing in RAM.

---
**Next up in Chapter 37:** We will explore the final data type optimization: Geospatial Data. We will learn how to calculate distances between EV Charging Stations and users on the surface of the Earth without relying on expensive external APIs.
