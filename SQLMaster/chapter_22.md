# Chapter 22: Managing Massive Data (Table Partitioning)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand why deleting millions of rows from massive IoT tables using the `DELETE` statement causes production outages.
*   Implement **Table Partitioning** to physically divide massive tables across disk files while keeping them logically unified to the application.
*   Design and execute the **Sliding Window Archival Pattern** using `SWITCH PARTITION` to instantly archive millions of rows with zero blocking.
*   Leverage **Partition Elimination** to optimize reporting queries.
*   Differentiate Table Partitioning (vertical) from Database Sharding (horizontal).

---

## 22.1 The Massive Data Problem

In our EV SaaS, charging stations send "Heartbeat" metrics every 60 seconds. With 50,000 stations, that is 1.5 billion rows per month inserted into `core.Telemetry`.

After 6 months, the table is 500GB. The business says, "We only need to keep 3 months of telemetry online. Delete the old data."
A junior developer writes:
`DELETE FROM core.Telemetry WHERE CreatedAt < '2024-04-01';`

**The Catastrophe:** 
This statement attempts to delete 4.5 billion rows. 
1. It escalates to a Table Lock, taking the telemetry API offline.
2. It writes 4.5 billion records to the Transaction Log (LDF), blowing out the server's hard drive and crashing the database entirely.

You cannot manage Terabyte-scale tables using standard DML (`INSERT/UPDATE/DELETE`). You must use **DDL** (Data Definition Language) via Table Partitioning.

---

## 22.2 Introduction to Table Partitioning

**Table Partitioning** allows you to divide a single physical table into multiple, separate partitions on disk, usually separated by Date. 
To EF Core and your C# application, it still looks and acts like one single `core.Telemetry` table. But under the hood, SQL Server stores January's data in Partition 1, February's data in Partition 2, and so on.

### Step 1: The Partition Function
The Partition Function defines the boundary values (e.g., the cutoff dates).
```sql
CREATE PARTITION FUNCTION pf_MonthlyTelemetry (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', 
    '2024-02-01', 
    '2024-03-01'
);
```

### Step 2: The Partition Scheme
The Partition Scheme maps the partitions created by the Function to physical Filegroups on disk. (You can put current data on expensive NVMe drives, and older partitions on cheap HDD storage).
```sql
CREATE PARTITION SCHEME ps_MonthlyTelemetry
AS PARTITION pf_MonthlyTelemetry
ALL TO ([PRIMARY]); -- For simplicity, we put them all in the primary filegroup
```

### Step 3: Create the Table on the Scheme
When creating the table (or its Clustered Index), you apply it to the Scheme instead of a standard Filegroup.
```sql
CREATE TABLE core.Telemetry (
    TelemetryId UNIQUEIDENTIFIER,
    StationId UNIQUEIDENTIFIER,
    CreatedAt DATETIME2,
    Metric VARCHAR(100)
) ON ps_MonthlyTelemetry(CreatedAt); -- Partitioned by CreatedAt!
```

---

## 22.3 The "Sliding Window" Archival Pattern

Now we return to our original problem: How do we delete January's data without crashing the server?
We use the **Sliding Window** pattern via the `ALTER TABLE ... SWITCH` command.

Because January's data is isolated in Partition 1, we can instantly "switch" that partition into an empty staging table. This is a metadata-only operation. It moves the pointers, not the data.

```sql
-- 1. Create an empty table with the exact same schema
CREATE TABLE core.Telemetry_Archive (
    TelemetryId UNIQUEIDENTIFIER,
    StationId UNIQUEIDENTIFIER,
    CreatedAt DATETIME2,
    Metric VARCHAR(100)
);

-- 2. Switch Partition 1 out of the live table into the archive table
ALTER TABLE core.Telemetry SWITCH PARTITION 1 TO core.Telemetry_Archive;

-- 3. Truncate the archive table (Instantaneous, doesn't bloat the Transaction Log)
TRUNCATE TABLE core.Telemetry_Archive;
```
**Result:** We just deleted 1.5 billion rows in **0.01 seconds**, generated zero transaction log bloat, and caused zero blocking on the live table.

---

## 22.4 Partition Elimination

Table partitioning also drastically speeds up read queries. 
If a dashboard runs:
`SELECT * FROM core.Telemetry WHERE CreatedAt >= '2024-03-05'`

SQL Server looks at the Partition Function. It knows that all data after March 1st is in Partition 3 and Partition 4. 
It completely ignores Partition 1 and Partition 2. This is called **Partition Elimination**. Instead of scanning 500GB of data, it only scans 200GB, immediately doubling reporting performance.

---

## 22.5 Architect Perspective: Partitioning vs. Sharding

Do not confuse Partitioning with Sharding.
*   **Table Partitioning:** Splitting a table across different disks *within the same database server*. It solves local storage management and archival. It does *not* give you more CPU or RAM.
*   **Database Sharding:** Splitting data across entirely different physical database servers (e.g., Tenant A lives on Server 1, Tenant B lives on Server 2). Sharding gives you infinite horizontal scale (CPU/RAM/Disk), but introduces massive application complexity (EF Core must route queries dynamically based on the Tenant). 

If a single SQL Server is running out of CPU, Table Partitioning will not save you. You must scale up, or Shard.

---

## 22.6 Columnstore Indexes

If you partition your tables but reporting is still too slow, you are likely hitting the limits of the standard B-Tree (Rowstore).
For multi-terabyte analytical queries (OLAP) that sum or average millions of rows, Architects use **Columnstore Indexes**.

Instead of storing data row-by-row on the 8KB page, Columnstore flips the storage engine and stores data *column-by-column*, applying massive compression. A 500GB table can compress down to 50GB, allowing SQL Server to load the entire table into RAM for instant analytics.
*(Note: Columnstore is heavily optimized for Data Warehouses, not transactional IoT inserts. Mixing them requires careful architecture).*

---

## 22.7 The Code: Partitioning in EF Core

Entity Framework Core is blissfully unaware of Table Partitioning. 
Because partitioning is completely transparent to `SELECT`, `INSERT`, `UPDATE`, and `DELETE` commands, your C# LINQ queries do not change at all.

The only caveat is Migrations. EF Core's Migration Builder does not have native fluent API methods for Partition Schemes. You must write raw SQL in your migration file.

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // You must write raw SQL to set up the partitions
    migrationBuilder.Sql(@"
        CREATE PARTITION FUNCTION ... ;
        CREATE PARTITION SCHEME ... ;
    ");
}
```

---

## 22.8 Performance & Security Analysis

### Performance Analysis: The Partitioning Key
To partition a table, the Partition Key (e.g., `CreatedAt`) **must** be part of the Clustered Index and *every* Unique Index on that table. If your Primary Key is `TelemetryId`, you must change it to a Composite Primary Key: `(TelemetryId, CreatedAt)`. This ensures SQL Server knows exactly which partition a row belongs in without scanning the entire table.

### Security Implications
*   **Data Leakage via Partition Swapping:** If you switch a partition to an archive table, and a junior DBA accidentally grants `SELECT` access to the archive table to the wrong Active Directory group, you have leaked 1.5 billion rows. Archive tables must have the exact same rigid Role-Based Access Control (RBAC) as the live tables.

---

## 22.9 Common Mistakes & Production Pitfalls

1.  **Over-Partitioning:** SQL Server supports up to 15,000 partitions per table. However, if you partition by the *Day* instead of by the *Month*, a query searching a 6-month date range has to query 180 separate partitions. The overhead of stitching those 180 streams back together destroys performance. Partition by Month or Year unless you have a petabyte-scale data warehouse.
2.  **Forgetting to Split:** A Partition Function has a hardcoded list of boundary dates. If you forget to `ALTER PARTITION FUNCTION ... SPLIT` to add a boundary for next year, all of next year's data will get dumped into the final partition, ruining your sliding window archival. You must have a SQL Agent job that runs monthly to create future partitions automatically.

---

## 22.10 Production Checklist

*   [ ] Highly volatile telemetry tables (IoT, Audit Logs) are partitioned by `CreatedAt` (Month).
*   [ ] Archival of old data is performed using `SWITCH PARTITION` to an archive table, followed by `TRUNCATE`, entirely avoiding the `DELETE` statement.
*   [ ] Reporting queries explicitly include the Partition Key (`CreatedAt`) in the `WHERE` clause to trigger Partition Elimination.
*   [ ] An automated script (SQL Agent Job) exists to proactively create new partition boundaries for upcoming months.

---

## 22.11 Exercises

1.  **Partition Elimination:** Your table is partitioned by `CreatedAt` (Monthly). A user runs a query: `SELECT SUM(TotalCost) FROM core.Sessions WHERE TenantId = 'T1'`. Will this query benefit from Partition Elimination? Why or why not?
2.  **Sliding Window Logic:** You have an empty table `Archive` and a partitioned table `Live`. You want to delete the oldest partition (Partition 1). Write the sequence of T-SQL commands to switch the partition and destroy the data safely.

---

## 22.12 Interview Questions

**Q1: Why is it catastrophic to run a `DELETE` statement to remove 500 million old records from a live production table, and how does Table Partitioning solve this?**
*Answer:* A massive `DELETE` statement operates row-by-row. It will escalate to a Table Lock (freezing the application) and it must write every single deleted row to the Transaction Log (LDF), which will quickly fill up the hard drive, causing the database engine to halt. Table Partitioning solves this via the "Sliding Window" pattern. You isolate the old data into its own partition, use `ALTER TABLE SWITCH` to instantly move the metadata pointers to an archive table, and then issue a `TRUNCATE` command. This happens in milliseconds with near-zero transaction log growth.

**Q2: What is "Partition Elimination" and how does it improve reporting performance?**
*Answer:* Partition Elimination occurs when the Query Optimizer evaluates a query's `WHERE` clause against the table's Partition Function boundaries. If the query asks for data from March, and the engine knows March data only lives in Partition 3, it will completely ignore the physical files for all other partitions. This drastically reduces the physical disk I/O and RAM required to satisfy the query.

---
**Next up in Chapter 23:** We are moving into Part 7: The Entity Framework Core Deep Dive. We will dissect how EF Core actually tracks changes, the dangers of `N+1` queries, and how to write high-performance LINQ.
