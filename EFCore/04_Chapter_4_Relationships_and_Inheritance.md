# Chapter 4: Mastering Relationships and Inheritance

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Configure One-to-Many and Many-to-Many relationships explicitly using the Fluent API to avoid convention-based mapping errors.
*   Understand the mechanics of Relationship Fixup within the Change Tracker.
*   Utilize Owned Types to map complex Value Objects (like Addresses) to a single table, adhering to DDD principles.
*   Implement advanced mapping techniques like Table Splitting to segregate heavy payloads from frequently queried entities.
*   Evaluate and implement the three Entity Framework inheritance strategies: Table-Per-Hierarchy (TPH), Table-Per-Type (TPT), and Table-Per-Concrete-Type (TPC), analyzing their impact on SQL execution plans.

## 2. Introduction

Relational databases are defined by relationships: Primary Keys and Foreign Keys linking distinct tables together to enforce data integrity. In Object-Oriented Programming (OOP), relationships are represented by object graphs—collections of references pointing to other objects in memory.

Mapping a relational Foreign Key to an object graph is where many Object-Relational Mappers fail. EF Core, however, provides a remarkably sophisticated engine for relationship mapping. It understands Principal entities (the "One" side), Dependent entities (the "Many" side), and how to cascade changes across the graph.

Furthermore, OOP relies heavily on Inheritance (e.g., `FastCharger` inherits from `Charger`). Relational databases do not natively understand inheritance. A Solution Architect must explicitly instruct EF Core on how to flatten or split these inheritance hierarchies across SQL tables. Choosing the wrong inheritance mapping strategy can lead to catastrophic performance issues involving massive, unoptimized `SQL JOIN` operations.

This chapter provides the blueprint for mapping complex object graphs and inheritance hierarchies accurately and performantly.

## 3. Core Concepts

### Principal vs. Dependent Entities
*   **Principal Entity:** The entity that contains the Primary Key. In a Tenant-to-Site relationship, `Tenant` is the Principal.
*   **Dependent Entity:** The entity that contains the Foreign Key. `Site` is the Dependent because it holds `TenantId`.

### Navigation Properties
Properties on a C# class that hold references to related entities.
*   **Reference Navigation Property:** Points to a single entity (e.g., `Site.Tenant`).
*   **Collection Navigation Property:** Points to a collection of entities (e.g., `Tenant.Sites`).

### Relationship Fixup
If you query a `Tenant`, and then in a separate query, you fetch a `Site` that belongs to that `Tenant`, EF Core's Change Tracker automatically detects that they are related and automatically populates the `Site.Tenant` and `Tenant.Sites` C# navigation properties in memory. You don't have to link them manually.

### Owned Types (Value Objects)
In DDD, an `Address` is not an Entity; it has no identity of its own. It is a Value Object. If the address changes, the whole object is replaced. In EF Core, Owned Types allow you to model `Address` as a separate C# class, but store its properties (Street, City, Zip) as columns *in the same table* as the parent entity (e.g., `Sites` table).

### Inheritance Mapping Strategies
*   **TPH (Table-Per-Hierarchy):** All classes in an inheritance hierarchy are mapped to a single SQL table. A `Discriminator` column indicates the specific type.
*   **TPT (Table-Per-Type):** Every class in the hierarchy maps to its own table. Querying a derived type requires a `SQL JOIN` to the base table.
*   **TPC (Table-Per-Concrete-Type):** Only non-abstract derived classes map to tables. Each table has all columns for both the base and derived properties. No `JOIN` is required, but there is no central table for the base type.

## 4. Visual Diagrams

```text
=============================================================================
             INHERITANCE MAPPING STRATEGIES IN SQL SERVER
=============================================================================
C# Hierarchy: 
abstract class Charger { Id, MaxKw }
class FastCharger : Charger { HasLiquidCooling }
class SlowCharger : Charger { RequiresRfid }

1. TPH (Table-Per-Hierarchy) - Default & Fastest for Reading
Table: Chargers
| Id | MaxKw | Discriminator | HasLiquidCooling | RequiresRfid |
|----|-------|---------------|------------------|--------------|
| 1  | 150   | FastCharger   | True             | NULL         |
| 2  | 7     | SlowCharger   | NULL             | True         |
(Sparse columns, but zero JOINs required).

2. TPT (Table-Per-Type) - Cleanest Schema, Slowest for Reading
Table: Chargers (Base)    Table: FastChargers       Table: SlowChargers
| Id | MaxKw |            | ChargerId | Cooling |   | ChargerId | Rfid |
|----|-------|            |-----------|---------|   |-----------|------|
| 1  | 150   |            | 1         | True    |   | 2         | True |
| 2  | 7     |            
(Requires a LEFT JOIN to query all chargers).

3. TPC (Table-Per-Concrete-Type) - Best for Write-Heavy & Partitioning
Table: FastChargers                Table: SlowChargers
| Id | MaxKw | Cooling |           | Id | MaxKw | Rfid |
|----|-------|---------|           |----|-------|------|
| 1  | 150   | True    |           | 2  | 7     | True |
(No Base table. Queries for 'Charger' require a UNION ALL).
```

## 5. API Deep Dive: Relationships

While EF Core can infer relationships if you name your properties `TenantId`, relying on conventions in an enterprise application is dangerous. A simple refactoring can silently break the database schema. **Always use the Fluent API to explicitly configure relationships.**

### 5.1 One-to-Many
The most common relationship. A `Tenant` has many `Site`s.

```csharp
public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        // HasMany (from Principal) -> WithOne (from Dependent)
        builder.HasMany(tenant => tenant.Sites)
               .WithOne(site => site.Tenant)
               .HasForeignKey(site => site.TenantId)
               .IsRequired() // Enforces an INNER JOIN, makes TenantId NOT NULL
               .OnDelete(DeleteBehavior.Restrict); // Critical: Prevent accidental cascade deletes
    }
}
```

### 5.2 One-to-One
A `Site` has exactly one `SiteManager`.

```csharp
public class SiteConfiguration : IEntityTypeConfiguration<Site>
{
    public void Configure(EntityTypeBuilder<Site> builder)
    {
        // HasOne -> WithOne
        builder.HasOne(site => site.Manager)
               .WithOne(manager => manager.Site)
               // You MUST specify which entity is the Dependent (holds the FK)
               .HasForeignKey<SiteManager>(manager => manager.SiteId); 
    }
}
```

### 5.3 Many-to-Many
In EF Core 5+, you do not need to explicitly create the C# class for the join table unless that join table has a payload (extra columns like `AssignedDate`). EF Core manages the join table automatically.

A `Driver` has many `RfidTag`s. An `RfidTag` can belong to many `Driver`s (e.g., shared fleet cards).

```csharp
public class DriverConfiguration : IEntityTypeConfiguration<Driver>
{
    public void Configure(EntityTypeBuilder<Driver> builder)
    {
        builder.HasMany(d => d.RfidTags)
               .WithMany(t => t.Drivers)
               .UsingEntity("DriverRfidTags"); // explicitly name the hidden join table
    }
}
```

## 6. Complete Examples: EV Platform Case Study

### 6.1 Owned Types (Value Objects)
Our `Site` entity requires an `Address`. We want `Address` to be a strictly validated C# object, but we don't want a separate `Addresses` table requiring a `JOIN` every time we query a Site.

```csharp
// Domain: Value Object
public record Address(string Street, string City, string PostalCode);

// Domain: Entity
public class Site
{
    public Guid Id { get; private set; }
    public Address LocationAddress { get; private set; } // Complex object
    
    // ...
}

// Configuration
public class SiteConfiguration : IEntityTypeConfiguration<Site>
{
    public void Configure(EntityTypeBuilder<Site> builder)
    {
        // Map the complex object to columns in the Sites table
        builder.OwnsOne(s => s.LocationAddress, addressBuilder =>
        {
            addressBuilder.Property(a => a.Street).HasColumnName("Street").IsRequired();
            addressBuilder.Property(a => a.City).HasColumnName("City").IsRequired();
            addressBuilder.Property(a => a.PostalCode).HasColumnName("PostalCode").IsRequired();
        });
    }
}
```
*Resulting SQL Table `Sites`: `[Id], [Street], [City], [PostalCode]`.*

### 6.2 Table Splitting
Our `Charger` entity has heavy diagnostic logs (`DiagnosticDump`). We rarely need this data unless actively troubleshooting. If we put it in the `Chargers` table, every `SELECT *` pulls massive amounts of data into memory.

Table Splitting maps *two* C# entities to the *exact same SQL table*.

```csharp
// Domain
public class Charger { public Guid Id { get; set; } /* lightweight props */ }
public class ChargerDiagnostics { public Guid Id { get; set; } public string Dump { get; set; } }

// Configuration
public class ChargerConfiguration : IEntityTypeConfiguration<Charger>
{
    public void Configure(EntityTypeBuilder<Charger> builder)
    {
        builder.ToTable("Chargers");
        
        // One-to-One relationship...
        builder.HasOne<ChargerDiagnostics>()
               .WithOne()
               .HasForeignKey<ChargerDiagnostics>(d => d.Id);
    }
}

public class ChargerDiagnosticsConfiguration : IEntityTypeConfiguration<ChargerDiagnostics>
{
    public void Configure(EntityTypeBuilder<ChargerDiagnostics> builder)
    {
        // Map to the EXACT SAME TABLE NAME
        builder.ToTable("Chargers"); 
    }
}
```
*Querying `context.Chargers` now only selects the lightweight columns. Querying `context.ChargerDiagnostics` selects the heavy column. Both write to the `Chargers` physical table.*

## 7. EF Core Internals: Inheritance Configuration

Let's configure the TPC (Table-Per-Concrete-Type) strategy for our Chargers, as recommended for high-performance SaaS applications where we partition data.

```csharp
public class ChargerConfiguration : IEntityTypeConfiguration<Charger>
{
    public void Configure(EntityTypeBuilder<Charger> builder)
    {
        // 1. Tell EF Core to use TPC
        builder.UseTpcMappingStrategy();
    }
}

public class FastChargerConfiguration : IEntityTypeConfiguration<FastCharger>
{
    public void Configure(EntityTypeBuilder<FastCharger> builder)
    {
        // Maps strictly to FastChargers table. Contains ID, MaxKw, and HasLiquidCooling
        builder.ToTable("FastChargers"); 
    }
}
```

## 8. Performance Implications

### Relationship Fixup Overhead
When executing a query that returns 10,000 entities, if the Change Tracker is enabled, it must perform "Fixup". It iterates through every entity, checks all Foreign Keys against its internal dictionary, and wires up the C# object graph. This is incredibly CPU intensive. 

**This is the secondary reason why `.AsNoTracking()` is mandatory for reads.** Without tracking, EF Core skips Relationship Fixup entirely, blindly instantiating the objects exactly as requested by the query projection, resulting in massive performance gains.

### The Inheritance Trap (TPT vs TPH)
*   **The TPT Disaster:** If you use Table-Per-Type, and you execute `context.Chargers.ToList()`, EF Core must execute a `SELECT` statement that performs a `LEFT JOIN` on *every single derived table*. If you have 10 derived charger types, that is a 10-table join. This will decimate database performance. **Never use TPT in EF Core unless mandated by legacy DBA schemas.**
*   **The TPH Default:** Table-Per-Hierarchy is the default because it is the fastest for querying. It requires zero JOINs. The trade-off is that columns belonging to specific derived types must be nullable, which prevents strict database-level `NOT NULL` constraints.
*   **The TPC Compromise:** Introduced fully in EF7, Table-Per-Concrete-Type is exceptional. Queries for a specific type (`context.FastChargers`) are incredibly fast. Queries for the base type (`context.Chargers`) generate a `UNION ALL`, which is generally faster than TPT's `LEFT JOINs`, but still slower than TPH.

## 9. ASP.NET Core Integration

When an ASP.NET Core API receives a JSON payload to create a `Site`, that JSON might include the `Address` properties. Because `Address` is an Owned Type, it is instantiated automatically when EF Core reads from the database. But when receiving from an API (a Detached entity), you must ensure the Owned Type property is not null before attaching it to the DbContext, otherwise EF Core will throw an exception during `SaveChanges`.

## 10. Clean Architecture Perspective

In DDD, relationships are tightly controlled via **Aggregates**. An Aggregate Root (e.g., `Tenant`) controls all access to its children (`Site`s). 

You should configure EF Core to respect these boundaries. Use `DeleteBehavior.Restrict` on almost all Foreign Keys to prevent accidental database-level cascading deletes. If a `Tenant` is deleted, the Domain layer should explicitly coordinate the deletion of its `Site`s, raising Domain Events for each deletion, rather than letting SQL Server blindly wipe the data via `CASCADE DELETE`.

## 11. Enterprise SaaS Perspective: Cascading Deletes

Cascading deletes are the silent killer of Enterprise SaaS applications.
If `Tenant -> Sites` is configured with `DeleteBehavior.Cascade` (the EF Core default for required relationships), and a junior developer accidentally calls `context.Tenants.Remove(tenant)`, SQL Server will execute a cascading delete that wipes out the Tenant, all their Sites, all their Chargers, all their Charging Sessions, and all their Invoices. 

**Architectural Rule:** Override the EF Core default conventions globally in `OnModelCreating` to set all Foreign Keys to `DeleteBehavior.Restrict`.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Apply configurations...

    // Globally disable Cascade Deletes
    var cascadeFKs = modelBuilder.Model.GetEntityTypes()
        .SelectMany(t => t.GetForeignKeys())
        .Where(fk => !fk.IsOwnership && fk.DeleteBehavior == DeleteBehavior.Cascade);

    foreach (var fk in cascadeFKs)
        fk.DeleteBehavior = DeleteBehavior.Restrict;
}
```

## 12. Real Production Case Study

In our EV Platform, we need to query `ChargingSession`s. A Session belongs to a `Charger`, which belongs to a `Site`, which belongs to a `Tenant`. 

If we don't configure relationships correctly, calculating the total energy consumed by a Tenant requires complex manual LINQ joins. By explicitly configuring the `HasOne().WithMany()` graph, EF Core allows us to write elegant queries:

```csharp
var totalEnergy = await context.Tenants
    .Where(t => t.Id == tenantId)
    .SelectMany(t => t.Sites)
    .SelectMany(s => s.Chargers)
    .SelectMany(c => c.Sessions)
    .SumAsync(sess => sess.KwhDelivered);
```
*EF Core translates this seamlessly into a series of highly optimized `INNER JOIN`s, executing entirely on SQL Server, returning a single integer to C#.*

## 13. Common Mistakes

### Beginner
*   **Mistake:** Relying on naming conventions (`SiteId`) to create relationships, but misspelling the property, resulting in EF Core creating a hidden Shadow Property (e.g., `SiteId1`) for the Foreign Key.
*   **Correction:** Always use the Fluent API `HasForeignKey(x => x.SiteId)` to explicitly map the property. If it's misspelled, the C# compiler throws an error.

### Intermediate
*   **Mistake:** Using a collection navigation property (`public List<Session> Sessions { get; set; }`) but forgetting to initialize it, leading to `NullReferenceException`s when adding items.
*   **Correction:** Always initialize collection navigation properties: `public ICollection<Session> Sessions { get; set; } = new List<Session>();`. EF Core will happily use the pre-initialized list.

### Senior
*   **Mistake:** Using TPT (Table-Per-Type) for a deep inheritance hierarchy to satisfy a DBA's desire for a "normalized schema," resulting in queries that take 5 seconds due to massive `LEFT JOIN`s.
*   **Correction:** The Architect must intervene and mandate TPH or TPC. Database normalization rules must yield to performance realities in high-throughput ORM environments.

### Architect
*   **Mistake:** Allowing EF Core to generate the Join Table for Many-to-Many relationships silently.
*   **Correction:** In enterprise systems, Many-to-Many relationships *always* eventually require a payload (e.g., `UserRoles` eventually needs an `AssignedDate` or `AssignedBy` column). The Architect should mandate explicit C# entities for all Join Tables (using `HasOne().WithMany()` twice) to future-proof the schema, avoiding painful migrations later when the payload is inevitably requested by the business.

## 14. Interview Questions

### Beginner (10)
1.  **What is a Navigation Property in EF Core?**
    *Answer:* A property on an entity that references a related entity (or collection of entities), allowing you to traverse the object graph.
2.  **What is the difference between a Principal and a Dependent entity?**
    *Answer:* The Principal entity holds the Primary Key. The Dependent entity holds the Foreign Key pointing to the Principal.
3.  **How do you configure a One-to-Many relationship using the Fluent API?**
    *Answer:* `builder.HasMany(p => p.Children).WithOne(c => c.Parent).HasForeignKey(c => c.ParentId);`
4.  **What is a cascading delete?**
    *Answer:* When a Principal entity is deleted, the database automatically deletes all related Dependent entities to maintain referential integrity.
5.  **Why should you avoid cascading deletes in an enterprise system?**
    *Answer:* Because an accidental deletion of a root entity (like a Tenant) can silently wipe out millions of rows of related data, causing a catastrophic data loss event.
6.  **What is an Owned Type?**
    *Answer:* A complex C# object (like an Address) that has no identity of its own and is mapped to columns within the same table as its parent entity.
7.  **What is the default inheritance mapping strategy in EF Core?**
    *Answer:* Table-Per-Hierarchy (TPH).
8.  **What does the `Discriminator` column do in TPH?**
    *Answer:* It stores a string (or int) indicating which specific derived C# class that particular database row represents.
9.  **Do you need to create a C# class for the join table in a Many-to-Many relationship?**
    *Answer:* Not in EF Core 5+. It can manage the join table automatically if there is no extra data (payload) required on the join table.
10. **How do you require a relationship to be non-nullable?**
    *Answer:* By adding `.IsRequired()` to the relationship configuration, or by making the Foreign Key property a non-nullable type (e.g., `Guid` instead of `Guid?`).

### Intermediate (10)
11. **Explain the concept of Relationship Fixup.**
    *Answer:* The process where the Change Tracker automatically populates navigation properties of tracked entities when it realizes they are related based on their Primary and Foreign Key values.
12. **What happens if you query with `AsNoTracking()`—does Relationship Fixup occur?**
    *Answer:* No. Identity Resolution and Relationship Fixup are completely bypassed to save memory and CPU. If you want navigation properties populated in a no-tracking query, you must explicitly use `.Include()`.
13. **How do you configure a One-to-One relationship?**
    *Answer:* `builder.HasOne(a => a.B).WithOne(b => b.A).HasForeignKey<EntityB>(b => b.A_Id);`. You must explicitly specify which generic type holds the foreign key.
14. **What is Table Splitting?**
    *Answer:* Mapping two distinct C# entities to the exact same physical SQL Server table. It is used to separate heavy columns (like large text or binary data) into a separate entity to improve read performance for the primary entity.
15. **Explain the performance drawback of Table-Per-Type (TPT) inheritance.**
    *Answer:* Querying the base type requires SQL Server to perform a `LEFT JOIN` against every single table representing a derived type. This causes severe execution plan degradation and slow performance.
16. **How do you rename the Discriminator column in TPH?**
    *Answer:* `builder.HasDiscriminator<string>("TypeColumnName");`
17. **What is a "payload" in a Many-to-Many relationship?**
    *Answer:* Additional columns on the join table beyond the two foreign keys (e.g., `CreatedAt`, `IsActive`). If a payload exists, you must explicitly model the join table as a C# entity.
18. **If you have a `Site` that owns an `Address`, how does EF Core name the columns in the database by default?**
    *Answer:* It prefixes them: `Address_Street`, `Address_City`.
19. **How do you prevent an Owned Type from being null?**
    *Answer:* In EF Core 6+, Owned Types are inherently required if the parent entity is instantiated. You must explicitly configure `.IsRequired(false)` on the navigation if you want it to be nullable, though this complicates table mapping.
20. **What is the difference between `DeleteBehavior.ClientSetNull` and `DeleteBehavior.Restrict`?**
    *Answer:* `ClientSetNull` tells the EF Core Change Tracker to set the FK to null in memory, but creates a `NO ACTION` constraint in SQL. `Restrict` enforces `NO ACTION` in SQL and throws an exception if you attempt to delete a Principal that still has Dependents.

### Senior (10)
21. **Architecturally, why would you choose TPC (Table-Per-Concrete-Type) over TPH for a high-volume SaaS platform?**
    *Answer:* TPH stores everything in one table, which can lead to massive, sparse tables that are difficult to partition. TPC creates distinct, tightly packed tables for each concrete type. This allows for dedicated indexing strategies per type and makes horizontal partitioning (sharding) vastly easier at the DBA level.
22. **You execute `context.Sites.Remove(site)`. The `Site` has a collection of `Chargers`. The relationship is configured as `DeleteBehavior.Cascade`. Explain exactly what SQL EF Core generates.**
    *Answer:* It depends on tracking. If the `Chargers` are NOT loaded in memory, EF Core generates a single `DELETE FROM Sites WHERE Id = 1` and relies entirely on the SQL Server `ON DELETE CASCADE` constraint to delete the chargers. If the `Chargers` ARE tracked in memory, EF Core's Change Tracker takes over, marks all Chargers as Deleted, and generates explicit `DELETE FROM Chargers...` statements for every charger, *then* deletes the Site.
23. **How do you dynamically disable all Cascade Deletes across an entire DbContext with 100 tables?**
    *Answer:* Inside `OnModelCreating`, iterate through `modelBuilder.Model.GetEntityTypes().SelectMany(t => t.GetForeignKeys())`. If `DeleteBehavior == Cascade` and it is not an Ownership relationship, mutate the `DeleteBehavior` to `Restrict`.
24. **Explain how EF Core 7+ handles mapping JSON columns using Owned Types.**
    *Answer:* You map an Owned Type, and then call `.ToJson()` on the builder. EF Core stores the entire Owned Type as a JSON string in a single `NVARCHAR(MAX)` column. Crucially, it translates LINQ queries against the Owned Type properties into SQL Server `JSON_VALUE` scalar functions, allowing database-side filtering.
25. **What is the "Cartesian Explosion" problem, and how does it relate to One-to-Many relationships?**
    *Answer:* If you query a `Tenant` and `.Include(t => t.Sites).Include(t => t.Users).Include(t => t.Invoices)`, EF Core (historically) generated a single massive SQL `JOIN` query. The result set is the Cartesian product of all those tables, resulting in millions of duplicated rows being transferred over the network.
26. **How do you solve the Cartesian Explosion problem in EF Core?**
    *Answer:* By appending `.AsSplitQuery()` to the LINQ statement. EF Core will generate multiple distinct, smaller SQL queries (one for Tenants, one for Sites, etc.) and stitch the object graph together in memory via Relationship Fixup, trading multiple database round-trips for massively reduced network payload size.
27. **You have an `Order` and an `OrderLine`. You want to enforce that an `OrderLine` can NEVER exist without an `Order`. How do you map this strictly in EF Core?**
    *Answer:* You map `OrderLine` as an Owned Type collection (`builder.OwnsMany(o => o.OrderLines)`). This strictly enforces the lifecycle; the `OrderLine` has no independent identity and is treated as a structural part of the `Order` aggregate.
28. **In a Many-to-Many relationship using an explicit Join Entity (e.g., `UserRoles`), how do you configure the Primary Key?**
    *Answer:* You must configure a composite primary key consisting of both foreign keys: `builder.HasKey(ur => new { ur.UserId, ur.RoleId });`
29. **What happens if you use `Include()` on a query, but the target relationship is an Owned Type?**
    *Answer:* Owned Types (stored in the same table) are retrieved automatically. You do not need to use `Include()`. Calling `Include()` on an Owned Type will often result in a compiler or runtime error depending on the EF Core version.
30. **Evaluate the use of Bi-directional vs. Uni-directional navigation properties.**
    *Answer:* Bi-directional (e.g., `Tenant.Sites` AND `Site.Tenant`) makes LINQ querying easier but couples the Domain entities tightly and increases memory overhead during fixup. Uni-directional (e.g., only `Site.TenantId`) enforces strict Aggregate Root traversal rules in DDD, reducing memory footprint and preventing developers from accidentally bypassing the `Tenant` root when accessing `Sites`.

### Staff Engineer (5)
31. **Architect a mechanism using EF Core Interceptors to implement a "Soft Delete" cascade. When a `Tenant` is soft-deleted (`IsDeleted = true`), all related `Sites` and `Chargers` must also be soft-deleted in the same transaction, without relying on SQL Server triggers.**
    *Answer:* In a `SaveChangesInterceptor`, during `SavingChanges`, find all entities marked as `Modified` where `IsDeleted` changed to `true`. You cannot simply iterate navigation properties, as they might not be loaded in memory. The Architect must use the `DbContext` to dynamically issue an `ExecuteUpdateAsync` command for the dependent tables. E.g., `await context.Sites.Where(s => s.TenantId == tenant.Id).ExecuteUpdateAsync(s => s.SetProperty(x => x.IsDeleted, true))`. This guarantees the cascade occurs entirely within the EF Core transaction boundary without requiring all child entities to be hydrated into RAM.
32. **A legacy database schema utilizes a polymorphic relationship: A `Comments` table has a `ParentId` and a `ParentType` (string) column. It can point to a `User`, a `Site`, or an `Invoice`. Analyze how to map this accurately in EF Core.**
    *Answer:* EF Core does not natively support true polymorphic associations (where a single FK points to multiple unknown tables based on a discriminator). The Architect must use a workaround: Map the `Comment` entity with three separate, nullable Foreign Keys (`UserId`, `SiteId`, `InvoiceId`) and configure a database-level `CHECK CONSTRAINT` ensuring exactly one is non-null. The `ParentType` column becomes obsolete or acts strictly as a computed column.
33. **Explain the internal execution plan differences between querying a TPH base class versus a TPC base class, and dictate when an Architect should choose one over the other.**
    *Answer:* TPH queries the base class using `SELECT * FROM Table`. Very fast. TPC queries the base class by generating `SELECT * FROM (SELECT Cols FROM DerivedA UNION ALL SELECT Cols FROM DerivedB)`. This prevents the use of many advanced indexing strategies on the base query. An Architect chooses TPH for 90% of workloads due to query speed. TPC is chosen ONLY when the derived types are massive, mutually exclusive, and the system requires horizontal partitioning or strict Non-Nullable column constraints at the database schema level.
34. **You are modeling a deeply nested Aggregate: `Invoice -> InvoiceLine -> TaxDetail`. Using `.AsSplitQuery()` causes 3 separate database queries. Explain the isolation level risks associated with Split Queries in a highly concurrent system.**
    *Answer:* Split Queries are not atomic by default. Query 1 reads `Invoices`. Context switch. Query 2 reads `InvoiceLines`. If another process deletes an `InvoiceLine` during that millisecond context switch, the resulting C# object graph is inconsistent (Read Skew). If data consistency is paramount, the Architect must wrap the Split Query in a strict `TransactionScope` using `IsolationLevel.Serializable` or `Snapshot`, or revert to a single, massive `JOIN` query.
35. **Evaluate the use of EF Core `ModelBuilder` conventions to automatically configure all properties named `*Json` as Value Converters that serialize to/from JSON, overriding the need for explicit Fluent API configuration per entity.**
    *Answer:* EF Core 6+ allows overriding pre-convention model building via `ConfigureConventions`. An Architect can write: `configurationBuilder.Properties<JsonDocument>().HaveConversion<JsonDocumentConverter>()`. This is highly recommended for enterprise systems as it enforces architectural consistency globally, reduces Fluent API boilerplate, and guarantees that developers cannot accidentally map a complex object as a standard string.

### Architect (5)
36. **Design a CQRS Read-Model architecture where the Domain heavily utilizes Owned Types and TPC inheritance, but the Read API requires microsecond response times returning flattened DTOs. Justify the interaction between EF Core and the database.**
    *Answer:* EF Core's instantiation of Owned Types and resolution of TPC hierarchies via `UNION ALL` adds unacceptable overhead for microsecond reads. The Architect must design the system to use EF Core *only* for the Command stack. For the Read stack, the Architect creates a highly indexed SQL Server View (e.g., `vw_FlatChargerDashboard`) that pre-joins and flattens the TPC tables and Owned Type columns. Dapper is then used to query this View directly into a flat C# DTO, bypassing EF Core entirely for the critical read path.
37. **In a globally distributed SaaS, a Tenant's data is sharded across multiple SQL Server instances based on `TenantId`. Explain how to configure EF Core relationships when a `User` (stored in a central global database) needs to reference a `Site` (stored in a tenant-specific shard).**
    *Answer:* EF Core cannot manage relationships or execute `JOIN`s across disparate physical databases or DbContexts. The Architect must sever the EF Core navigation property. The `User` entity holds a `SiteId` as a standard `Guid` property, not a Foreign Key. The Application Layer orchestrates the interaction: Query the central DbContext for the User, extract the `SiteId`, resolve the correct tenant DbContext shard via a factory, and issue a second query. Referential integrity must be managed by the application via Saga patterns, not by EF Core or SQL Server.
38. **Defend the decision to map a Many-to-Many relationship explicitly using two One-to-Many relationships and a physical Join Entity class, rather than relying on EF Core's implicit Many-to-Many mapping.**
    *Answer:* Implicit Many-to-Many is a trap for enterprise systems. It works for prototypes, but the moment the business requests "When was this user added to this role?" or "Who approved this assignment?", the implicit join table lacks the payload columns (`CreatedAt`, `ApprovedBy`) to store it. Migrating an implicit join table to an explicit one later requires complex data migration scripts. The Architect future-proofs the schema by always defining the explicit `UserRole` join entity from day one.
39. **Evaluate the performance and architectural impact of using `Include()` inside a loop within an Application Service.**
    *Answer:* Using `Include()` inside a loop (e.g., `foreach(var id in ids) { context.Sites.Include(s => s.Chargers).Single(s => s.Id == id); }`) is an N+1 query disaster. It executes a separate database query for every iteration. The Architect must refactor this to a set-based query before the loop: `context.Sites.Include(s => s.Chargers).Where(s => ids.Contains(s.Id)).ToList()`. This hits the database exactly once, utilizing EF Core's Relationship Fixup to prepare the graph for subsequent in-memory processing.
40. **How do you architect a system to handle EF Core's `DbUpdateConcurrencyException` specifically when it is triggered by a cascading delete conflict?**
    *Answer:* A cascading delete conflict occurs when Process A deletes a `Tenant`, and simultaneously Process B adds a new `Site` to that `Tenant`. Process A attempts to delete the Tenant and cascade to Sites. Process B attempts to insert a Site. SQL Server detects the FK violation or deadlock. The Architect must wrap the `SaveChanges` call in an explicit Execution Strategy (Polly). If the specific SQL Error Number indicates an FK constraint failure during a concurrent delete, the strategy must abort the `Site` creation and return a clean Domain error ("Parent entity no longer exists") rather than crashing the API.

## 15. Exercises

### Easy
1.  **One-to-Many:** Create an `Author` class and a `Book` class. Use the Fluent API to explicitly map a One-to-Many relationship where an Author has many Books. Ensure the Foreign Key is named `AuthorId` and is required.

### Medium
1.  **Owned Types:** Create a `Customer` entity. Create a `ContactInfo` record containing `Email` and `PhoneNumber`. Configure `ContactInfo` as an Owned Type of `Customer`. Generate a migration and verify that `Email` and `PhoneNumber` are created as columns in the `Customers` table.

### Hard
1.  **Many-to-Many Payload:** Create `Student` and `Course` entities. Create an explicit join entity `Enrollment` containing `StudentId`, `CourseId`, and an `EnrollmentDate` payload. Use the Fluent API to configure the composite primary key and the two One-to-Many relationships that form the Many-to-Many graph.

### Enterprise
1.  **Inheritance and Fixup:** Implement a TPC (Table-Per-Concrete-Type) hierarchy with an abstract `Vehicle` base class and concrete `Car` and `Truck` classes. Create a `Fleet` entity that has a collection of `Vehicle`s. Write a query to fetch a `Fleet` and `.Include(f => f.Vehicles)`. Analyze the generated SQL to understand how EF Core constructs the `UNION ALL` query to satisfy the relationship across distinct concrete tables.

## 16. Production Checklist

- [ ] Are all relationships explicitly defined using the Fluent API (`HasOne().WithMany()`) rather than relying on fragile naming conventions?
- [ ] Have Cascade Deletes been globally disabled (`DeleteBehavior.Restrict`) to prevent catastrophic data loss?
- [ ] Are complex Value Objects (like Addresses or Coordinates) mapped using Owned Types to adhere to DDD?
- [ ] Has TPT (Table-Per-Type) inheritance been strictly avoided in favor of TPH or TPC to ensure query performance?
- [ ] Are explicit Join Entities used for Many-to-Many relationships to allow for future payload column expansion?

## 17. Summary

Mastering relationships is the difference between a system that scales and a system that collapses under the weight of Cartesian explosions and N+1 queries. By utilizing the Fluent API, we can explicitly define Principal and Dependent entities, enforce strict DDD boundaries with Owned Types, and optimize inheritance hierarchies using TPH or TPC.

With our domain modeled and our relationships secured, we are ready to query this data. In the next chapter, we will dissect the EF Core LINQ Provider, exploring how Expression Trees are compiled into T-SQL, and how to write high-performance queries that avoid the deadly pitfalls of client-side evaluation.
