# Part 4: Database Programmability & Semi-Structured Data

# Chapter 12: Views & Materialization

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand how Standard Views abstract schema complexity and provide security boundaries.
*   Implement Indexed Views (Materialized Views) to pre-aggregate millions of rows for instant dashboard rendering.
*   Apply the `SCHEMABINDING` and `NOEXPAND` requirements to guarantee Indexed View performance.
*   Evaluate the architectural trade-offs (the "Write Penalty") of materializing data.
*   Map SQL Views to Read-Only entities in EF Core using `.ToView()`.

---

## 12.1 Introduction to Views

As an enterprise application evolves, the physical database schema often becomes highly normalized (spread across dozens of tables) to ensure data integrity. However, reporting dashboards and APIs usually prefer flat, denormalized data.

A **View** acts as a virtual table. It is a saved SQL query that you can `SELECT` from as if it were a real table.

```sql
CREATE VIEW reporting.vw_SessionDetails AS
SELECT 
    s.SessionId,
    s.TenantId,
    t.Name AS TenantName,
    st.Name AS StationName,
    p.PortNumber,
    s.StartTime,
    s.TotalKwh
FROM core.Sessions s
INNER JOIN core.Tenants t ON s.TenantId = t.TenantId
INNER JOIN core.Ports p ON s.PortId = p.PortId
INNER JOIN core.Stations st ON p.StationId = st.StationId;
```
Now, instead of writing a 4-table join in C#, a developer can simply query:
`SELECT * FROM reporting.vw_SessionDetails WHERE TenantId = 'T1-UUID';`

### Security Boundaries
Views are excellent for security. You can grant a reporting user `SELECT` permission on the View, while explicitly denying them access to the underlying `core.Sessions` table (which might contain sensitive credit card tokens).

---

## 12.2 Standard Views vs. Tables

A standard View does **not** store data on disk. 
When you query `vw_SessionDetails`, the SQL Server Query Optimizer expands the View definition and executes the underlying joins against the physical tables in real-time. 
If the underlying `core.Sessions` table has 100 million rows, querying the View will still have to process 100 million rows. Standard Views improve *readability*, not *performance*.

---

## 12.3 Indexed Views (Materialized Views)

If your SaaS Tenant Admin opens their dashboard, they expect to see their "Total Revenue by Station" instantly. If the database has to sum 5 million historical sessions in real-time, the dashboard will take 10 seconds to load.

To solve this, we use an **Indexed View** (known as a Materialized View in PostgreSQL/Oracle).

An Indexed View actually **writes the result of the query to disk**. It behaves exactly like a physical table. When data in the base table changes, SQL Server automatically updates the Indexed View in the background.

### Creating an Indexed View
Creating an Indexed View has strict requirements:
1.  You must use `WITH SCHEMABINDING`. This prevents anyone from altering or dropping the base tables without first dropping the view.
2.  If you are aggregating data (e.g., `SUM`), you must also include `COUNT_BIG(*)`.

```sql
-- Step 1: Create the View with SCHEMABINDING
CREATE VIEW reporting.vw_StationRevenue
WITH SCHEMABINDING 
AS
SELECT 
    StationId,
    TenantId,
    SUM(TotalCost) AS TotalRevenue,
    COUNT_BIG(*) AS SessionCount -- Mandatory for aggregation views
FROM core.Sessions
GROUP BY StationId, TenantId;
GO

-- Step 2: Materialize it by creating a Unique Clustered Index
CREATE UNIQUE CLUSTERED INDEX CIX_vw_StationRevenue_StationId 
ON reporting.vw_StationRevenue (StationId);
```
Once that index is created, the aggregation is calculated and saved to the MDF file. Querying `SELECT TotalRevenue FROM reporting.vw_StationRevenue` now takes **0 milliseconds**, because the math is already done.

### The `NOEXPAND` Hint
In SQL Server Standard Edition, the Query Optimizer might ignore your Indexed View and recalculate the math against the base tables anyway. To force the engine to use the materialized disk structure, always use the `WITH (NOEXPAND)` hint.
`SELECT * FROM reporting.vw_StationRevenue WITH (NOEXPAND);`

---

## 12.4 Architect Perspective: The Write Penalty

Materialization is not a silver bullet; it is a strict architectural trade-off.

*   **The Read Benefit:** Reads drop from seconds to milliseconds. CPU usage for dashboards drops to zero.
*   **The Write Penalty:** Every time an IoT device `INSERT`s a new charging session, SQL Server must also implicitly `UPDATE` the Indexed View on disk to adjust the `SUM(TotalCost)`. This slows down your `INSERT` performance.

**Architect Rule:** Never create an Indexed View on a table that suffers from extremely high-frequency, high-concurrency inserts (like raw IoT telemetry heartbeats). Use them for tables with a High-Read / Low-Write ratio (like Daily Billing Rollups).

---

## 12.5 The Code: EF Core and Views

Entity Framework Core can easily map to Views, treating them as read-only entities.

```csharp
// 1. Create the C# DTO
public class StationRevenueView
{
    public Guid StationId { get; set; }
    public Guid TenantId { get; set; }
    public decimal TotalRevenue { get; set; }
}

// 2. Configure the mapping in DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<StationRevenueView>(entity =>
    {
        // Map to the View instead of a Table
        entity.ToView("vw_StationRevenue", "reporting");
        
        // Views don't have primary keys in EF Core, so we define no tracking
        entity.HasNoKey(); 
    });
}

// 3. Querying
var revenue = await context.Set<StationRevenueView>()
    .Where(v => v.TenantId == tenantId)
    .ToListAsync();
```
*(Note: To inject the `WITH (NOEXPAND)` hint in EF Core, you typically have to use EF Core Interceptors or raw SQL.)*

---

## 12.6 Performance & Security Analysis

### Performance Analysis
*   **View Nesting (The Anti-Pattern):** Never create a View that selects from another View, which selects from another View. This is called "View Nesting." It destroys the Query Optimizer's ability to accurately estimate cardinality, resulting in terrible execution plans and massive TempDB spills. Keep Views strictly tied to base tables.

### Security Implications
*   **Row-Level Security (RLS) Leakage:** If you implement RLS on your base tables (covered in Chapter 28), standard Views automatically inherit those security policies. However, managing security permissions on Indexed Views can become complex if they aggregate data across tenant boundaries. Ensure the `TenantId` is always part of the View's Clustered Index.

---

## 12.7 Common Mistakes & Production Pitfalls

1.  **Using `SELECT *` in a View:** If you write `CREATE VIEW v AS SELECT * FROM Table`, and later add a column to `Table`, the View will break or return incorrect metadata until you run `sp_refreshview`. Always explicitly name columns in a View definition.
2.  **Outer Joins in Indexed Views:** SQL Server does *not* allow `LEFT JOIN`, `RIGHT JOIN`, or `OUTER JOIN` in an Indexed View. You can only materialize data using `INNER JOIN`s.

---

## 12.8 Production Checklist

*   [ ] Reporting APIs query Views rather than manually rebuilding 6-table joins in C# LINQ.
*   [ ] Views are defined with explicit column lists (no `SELECT *`).
*   [ ] Dashboard aggregations use Indexed Views to pre-calculate data.
*   [ ] The Write-Penalty has been calculated before applying an Indexed View to a high-transaction IoT table.
*   [ ] EF Core configurations for Views use `.HasNoKey()` and `.ToView()`.

---

## 12.9 Exercises

1.  **View Creation:** Write a T-SQL statement to create a standard View named `reporting.vw_ActiveUsers` that returns the `UserId`, `Email`, and `TenantId` for all users who have `IsActive = 1`.
2.  **Schema Binding:** You attempt to create an Indexed View, but SQL Server throws an error: *"Cannot create index on view because the view is not schema bound."* Write the exact `ALTER VIEW` statement required to fix this issue on `reporting.vw_ActiveUsers`.

---

## 12.10 Interview Questions

**Q1: What is the physical difference between a Standard View and an Indexed (Materialized) View?**
*Answer:* A Standard View is merely a saved SQL script; it does not store data on disk. When queried, the engine expands the view and queries the underlying base tables. An Indexed View physically materializes the result of the query to the disk (MDF file) via a Unique Clustered Index. Querying it is instantaneous, but it incurs a write-penalty because the engine must keep the disk structure updated whenever the base tables change.

**Q2: Why does SQL Server force you to use `WITH SCHEMABINDING` when creating an Indexed View?**
*Answer:* Because the Indexed View materializes data to disk, the database engine must guarantee that the underlying table structure never changes out from under it. `SCHEMABINDING` locks the base tables. If a DBA tries to `DROP` a column from the base table that the Indexed View relies on, the engine will block the `DROP` command, ensuring the integrity of the materialized data.

---
**Next up in Chapter 13:** We will dive deeper into database programmability by exploring Stored Procedures and Functions. We will discuss when to encapsulate business logic in the DB vs. the Application Layer, and why Scalar Functions are notorious performance killers.
