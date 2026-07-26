# Chapter 9: Advanced Architecture (CQRS and Multi-Tenancy)

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Implement a strict CQRS (Command Query Responsibility Segregation) architecture, utilizing EF Core for Commands and highly optimized projections (or Dapper) for Queries.
*   Evaluate the three primary Multi-Tenancy architectures (Database-per-Tenant, Schema-per-Tenant, Shared-Database) and implement the Shared-Database model using EF Core.
*   Utilize Global Query Filters to guarantee Tenant Data Isolation, preventing cross-tenant data leaks at the compiler level.
*   Architect the ASP.NET Core Dependency Injection container to securely pass contextual state (like `TenantId`) into the `DbContext` without violating Clean Architecture.
*   Understand the performance implications of Global Query Filters on SQL Server execution plans.

## 2. Introduction

Building a simple CRUD application is easy. Building a multi-tenant Enterprise SaaS platform that serves hundreds of different companies from a single codebase is one of the hardest challenges in software engineering.

When a user from "Acme Corp" logs in, the system must guarantee—with absolute mathematical certainty—that they can never see data belonging to "Globex Corp". A single developer forgetting to add a `WHERE TenantId = X` clause in a LINQ query can cause a data breach that destroys a company.

Furthermore, as the SaaS scales, the traditional "Repository Pattern" collapses. Reading complex data grids requires massively different SQL optimization than validating complex Domain Aggregates during a write operation.

This chapter introduces two enterprise architectural patterns: **CQRS** and **Multi-Tenancy**. We will configure EF Core not just as a data access tool, but as a rigid structural boundary that enforces security and segregates read/write workloads at the deepest levels of the application.

## 3. Core Concepts

### CQRS (Command Query Responsibility Segregation)
CQRS dictates that the code used to read data (Queries) should be entirely separate from the code used to mutate data (Commands). 
*   **Commands:** Use EF Core, the Change Tracker, and Domain-Driven Design (Aggregates) to enforce strict business rules. 
*   **Queries:** Bypass the Domain Layer entirely. Use EF Core `.AsNoTracking().Select()` or raw Micro-ORMs (Dapper) to read flattened data directly into DTOs for maximum speed.

### Multi-Tenancy Architectures
1.  **Database-per-Tenant:** Highest isolation, highest cost. Each tenant has their own physical database.
2.  **Schema-per-Tenant:** Medium isolation. One database, but each tenant has their own schema (e.g., `acme.Users`, `globex.Users`). Rarely used with EF Core due to migration complexity.
3.  **Shared-Database (Row-Level):** Lowest cost, highest risk. All tenants share the same tables. A `TenantId` column distinguishes data. This is the most common SaaS pattern and requires rigorous software enforcement.

### Global Query Filters
An EF Core feature that automatically appends a LINQ `Where` clause to *every single query* executed for a specific entity type. It is the primary mechanism for enforcing Shared-Database multi-tenancy and Soft Deletes (`IsDeleted == false`).

## 4. Visual Diagrams

```text
=============================================================================
             STRICT CQRS ARCHITECTURE
=============================================================================
                          [ API Controller ]
                                 │
           ┌─────────────────────┴──────────────────────┐
           ▼                                            ▼
[ Command (Write) Stack ]                    [ Query (Read) Stack ]
       │                                            │
[ CommandHandler (MediatR) ]                 [ QueryHandler (MediatR) ]
       │                                            │
[ Domain Entities (Aggregates) ]             [ Flattened DTOs / ReadModels ]
       │                                            │
[ EF Core DbContext (Tracked) ]              [ EF Core .AsNoTracking() OR Dapper ]
       │                                            │
       └─────────────────────┬──────────────────────┘
                             ▼
                    [ SQL Server Database ]
```

```text
=============================================================================
             SHARED-DATABASE MULTI-TENANCY FLOW
=============================================================================

1. HTTP Request arrives:  `GET /api/chargers` (Header: `X-Tenant-Id: Acme-123`)
2. ASP.NET Core Middleware extracts `Acme-123` and sets it in `ITenantProvider`.
3. Controller calls `context.Chargers.ToList()`.
4. EF Core Query Compiler kicks in.
5. EF Core intercepts the query, injects the Global Query Filter.
6. Generated SQL: `SELECT * FROM Chargers WHERE TenantId = 'Acme-123'`
(The developer never explicitly wrote the WHERE clause).
```

## 5. API Deep Dive: Implementing Shared-Database Multi-Tenancy

To build a secure SaaS, we must inject the current user's Tenant ID into the `DbContext` and apply it globally.

### Step 1: The Tenant Provider
We need a stateless way for the DbContext to know who is executing the query.

```csharp
// Application Layer
public interface ITenantProvider
{
    Guid GetTenantId();
}

// API Layer (Implementation)
public class HttpContextTenantProvider : ITenantProvider
{
    private readonly IHttpContextAccessor _accessor;
    public HttpContextTenantProvider(IHttpContextAccessor accessor) => _accessor = accessor;

    public Guid GetTenantId()
    {
        // Extract from JWT Claims or Custom Header
        var claim = _accessor.HttpContext?.User.FindFirst("TenantId");
        return claim != null ? Guid.Parse(claim.Value) : throw new UnauthorizedAccessException();
    }
}
```

### Step 2: Injecting into the DbContext
We inject the provider into the DbContext constructor and store the current Tenant ID.

```csharp
public class EvDbContext : DbContext
{
    private readonly Guid _tenantId;

    // Inject the provider
    public EvDbContext(DbContextOptions<EvDbContext> options, ITenantProvider tenantProvider) 
        : base(options)
    {
        _tenantId = tenantProvider.GetTenantId();
    }
}
```

### Step 3: Configuring the Global Query Filter
In `OnModelCreating`, we apply the filter to all entities that implement an `IMustHaveTenant` interface.

```csharp
public interface IMustHaveTenant { Guid TenantId { get; set; } }

protected override void OnModelCreating(ModelBuilder builder)
{
    // Apply filter strictly relying on the private _tenantId field
    builder.Entity<Site>().HasQueryFilter(s => s.TenantId == _tenantId);
    builder.Entity<Charger>().HasQueryFilter(c => c.TenantId == _tenantId);
    
    // (In a real system, you would use reflection to apply this automatically 
    // to all entities implementing IMustHaveTenant).
}
```

## 6. EF Core Internals: Dynamic Query Compilation

When you use a field (like `_tenantId`) inside a `HasQueryFilter`, EF Core recognizes it as a parameter, not a constant.

If User A (TenantId: 1) executes `context.Sites.ToList()`, EF Core caches the query plan:
`SELECT * FROM Sites WHERE TenantId = @p0` (where `@p0` evaluates to `1`).

If User B (TenantId: 2) executes the exact same code, EF Core hits the cache, reuses the SQL plan, but passes `2` as `@p0`. This is highly optimized and prevents cache bloat.

**The Internal Warning:**
If you try to inject `ITenantProvider` directly into `OnModelCreating` and call `.GetTenantId()` inside the lambda (`HasQueryFilter(s => s.TenantId == provider.GetTenantId())`), EF Core will execute that method *every single time a query is evaluated*. This introduces massive overhead. You must resolve the ID once per DbContext instance and store it in a field.

## 7. Complete Examples: EV Platform Case Study

We are building a CQRS system for the EV Platform. 

### The Command Stack (Writes - EF Core)
When a user wants to create a new Site, they send a Command. We use EF Core to validate domain rules and save.

```csharp
// MediatR Command Handler
public async Task Handle(CreateSiteCommand command, CancellationToken ct)
{
    // 1. Domain Validation
    var site = new Site(command.Name, command.Address);
    
    // 2. Multi-Tenancy Enforcement (Implicit)
    // The TenantId is automatically set by overriding SaveChangesAsync
    
    // 3. Save via tracked DbContext
    _context.Sites.Add(site);
    await _context.SaveChangesAsync(ct);
}

// Inside EvDbContext.cs - Securing the Write Path
public override Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    foreach (var entry in ChangeTracker.Entries<IMustHaveTenant>().Where(e => e.State == EntityState.Added))
    {
        // Force every new entity to belong to the current user's tenant
        entry.Entity.TenantId = _tenantId; 
    }
    return base.SaveChangesAsync(ct);
}
```

### The Query Stack (Reads - Projections)
When a user views their dashboard, we bypass the Domain completely.

```csharp
// MediatR Query Handler
public async Task<List<SiteDashboardDto>> Handle(GetDashboardQuery query, CancellationToken ct)
{
    // 1. CQRS Read: Use AsNoTracking and Select
    // 2. Security: The Global Query Filter automatically appends WHERE TenantId = _tenantId
    return await _context.Sites
        .AsNoTracking()
        .Select(s => new SiteDashboardDto 
        {
            SiteName = s.Name,
            TotalChargers = s.Chargers.Count
        })
        .ToListAsync(ct);
}
```
*Note: Even though the developer didn't write a `WHERE` clause, the SaaS is secure. Data isolation is mathematically guaranteed by the architecture.*

## 8. Performance Implications of Query Filters

Adding `HasQueryFilter(x => x.TenantId == _tenantId)` changes *every* SQL query.

If you execute `context.Sites.Where(s => s.Region == "US").ToList()`, the SQL becomes:
`SELECT * FROM Sites WHERE Region = 'US' AND TenantId = @p0`.

**The Missing Index Disaster:**
If a DBA created an index on `(Region)`, that index is now virtually useless. SQL Server has to filter by `TenantId` first. 

**Architectural Rule:** In a Shared-Database multi-tenant system, `TenantId` **must** be the leading column in almost every single composite index in the database. 
`CREATE INDEX IX_Sites_Tenant_Region ON Sites(TenantId, Region);`

If you fail to do this, your Global Query Filters will cause catastrophic Index Scans across the entire database.

## 9. ASP.NET Core Integration: Bypassing Filters

Occasionally, a background worker or a "Super Admin" needs to query data across all tenants (e.g., generating a global billing report).

You can bypass the Global Query Filter using `.IgnoreQueryFilters()`.

```csharp
public async Task<decimal> CalculateGlobalRevenueAsync()
{
    // Bypasses the TenantId filter! Extremely dangerous, use with caution.
    return await _context.Invoices
        .IgnoreQueryFilters() 
        .SumAsync(i => i.TotalAmount);
}
```
*Warning:* `IgnoreQueryFilters()` removes *all* filters on the entity, including Soft Delete (`IsDeleted == false`). If you need to ignore the Tenant filter but keep the Soft Delete filter, you cannot use EF Core's built-in feature. You must manually construct the query or use raw SQL.

## 10. Clean Architecture Perspective

CQRS naturally aligns with Clean Architecture:
*   **Commands** interact with the core Domain Layer. They load Aggregates, invoke business methods (`site.Deactivate()`), and utilize Repositories to save.
*   **Queries** belong strictly in the Application Layer (or even a specialized Read Layer). They query the database directly and return Data Transfer Objects (DTOs). Queries do not use Domain Entities, and they do not use Repositories.

This segregation prevents the `Site` Domain Entity from becoming bloated with 50 different properties needed only for various UI screens.

## 11. Enterprise SaaS Perspective: Cross-Database Multi-Tenancy

If an enterprise client (like a government agency) demands strict physical isolation, you must abandon the Shared-Database model and use Database-per-Tenant.

EF Core handles this beautifully through dynamic Connection Strings.

```csharp
// The Tenant Provider now returns a Connection String, not an ID
public class EvDbContext : DbContext
{
    private readonly string _tenantConnectionString;

    public EvDbContext(DbContextOptions<EvDbContext> options, ITenantProvider provider) : base(options)
    {
        _tenantConnectionString = provider.GetConnectionString();
    }

    protected override void OnConfiguring(DbContextOptionsBuilder builder)
    {
        // Override the default DI configuration dynamically!
        if (!builder.IsConfigured || !string.IsNullOrEmpty(_tenantConnectionString))
        {
            builder.UseSqlServer(_tenantConnectionString);
        }
    }
}
```
*Architecture Note:* In this model, you do *not* use Global Query Filters for `TenantId` because the data is physically isolated. However, schema migrations become exceptionally difficult, as you must orchestrate the deployment of `update.sql` against hundreds of separate databases.

## 12. Real Production Case Study

In the EV Platform, we used Global Query Filters to enforce multi-tenancy. It worked flawlessly for 6 months.

Then, we introduced a feature where a `Driver` (a global entity, not tied to a specific tenant) could be assigned an `RfidTag` (which *is* tied to a specific tenant).

A developer wrote this query in a background job running outside of a web request (meaning `ITenantProvider` returned `null` or `Empty Guid`):

`var tags = await context.Drivers.Include(d => d.RfidTags).ToListAsync();`

The background job crashed. Why?
Because `RfidTag` has a Global Query Filter (`WHERE TenantId = _tenantId`). In the background job, `_tenantId` was Empty. EF Core diligently appended `WHERE TenantId = '00000000...'` to the `LEFT JOIN` for `RfidTags`. The query executed, but it filtered out all the tags!

**The Lesson:** Global Query Filters apply to `Include()` statements as well. If an entity crosses tenant boundaries, or if jobs run outside tenant context, you must explicitly use `IgnoreQueryFilters()` and manually construct the security constraints.

## 13. Common Mistakes

### Beginner
*   **Mistake:** Forgetting to set the `TenantId` when creating a new entity.
*   **Correction:** Do not rely on developers to set it manually (`var s = new Site { TenantId = currentId }`). Override `SaveChangesAsync` and use the Change Tracker to automatically inject the `TenantId` into all entities in the `Added` state.

### Intermediate
*   **Mistake:** Storing the `ITenantProvider` interface directly in a field in the `DbContext` and calling `.GetTenantId()` inside the `HasQueryFilter` lambda.
*   **Correction:** This evaluates the dependency injection chain on every single SQL query, destroying performance. Resolve the ID once in the DbContext constructor and store it in a `private readonly Guid _tenantId` field. Use that field in the lambda.

### Senior
*   **Mistake:** Using DbContext Pooling (`AddDbContextPool`) with a dynamically injected `TenantId`.
*   **Correction:** If you resolve the `TenantId` in the constructor and store it in a private field, that context is now permanently bound to that tenant. If you return it to the pool, the next HTTP request might be for a different tenant, but it gets the old context! **You cannot use DbContext Pooling if your DbContext holds tenant-specific constructor state.** You must use a standard `AddDbContext`, or resolve the TenantId dynamically per method call.

### Architect
*   **Mistake:** Designing a Shared-Database multi-tenant system without explicitly mandating the composite indexing strategy upfront.
*   **Correction:** The Architect must write an EF Core convention or custom analyzer that enforces that the first column of every new index created in a migration is the `TenantId`. Failure to do so will result in a SaaS that grinds to a halt once it hits a million rows due to un-sargable query filters.

## 14. Interview Questions

### Beginner (10)
1.  **What does CQRS stand for?**
    *Answer:* Command Query Responsibility Segregation.
2.  **In CQRS, what is the responsibility of a Command?**
    *Answer:* To mutate state (Create, Update, Delete) and enforce strict business rules.
3.  **In CQRS, what is the responsibility of a Query?**
    *Answer:* To read data as fast as possible without mutating state or validating business rules.
4.  **What is a Global Query Filter in EF Core?**
    *Answer:* A LINQ expression configured in `OnModelCreating` that is automatically appended to every query executed against a specific entity.
5.  **What is the most common use case for a Global Query Filter?**
    *Answer:* Enforcing Soft Deletes (`IsDeleted == false`) and Row-Level Multi-Tenancy (`TenantId == currentTenant`).
6.  **How do you bypass a Global Query Filter?**
    *Answer:* By appending `.IgnoreQueryFilters()` to the LINQ query.
7.  **What is Shared-Database Multi-Tenancy?**
    *Answer:* All tenants share the same database and tables. Data is separated logically using a discriminator column (like `TenantId`).
8.  **What is Database-per-Tenant Multi-Tenancy?**
    *Answer:* Each tenant has their own separate physical database, providing maximum isolation.
9.  **Should Queries in CQRS use the Change Tracker?**
    *Answer:* No. They should always use `.AsNoTracking()` or bypass EF Core entirely using a Micro-ORM like Dapper.
10. **Where do you configure Global Query Filters?**
    *Answer:* Inside the `OnModelCreating` method of the `DbContext`.

### Intermediate (10)
11. **How do you ensure a developer never forgets to set the `TenantId` on a newly inserted record?**
    *Answer:* Override `SaveChangesAsync`, iterate through `ChangeTracker.Entries()`, find all entities in the `Added` state that implement `IMustHaveTenant`, and force their `TenantId` property to the current user's TenantId.
12. **Why is it dangerous to use `IgnoreQueryFilters()`?**
    *Answer:* It removes *all* filters. If an entity has both a Tenant filter and a Soft Delete filter, bypassing the Tenant filter to generate a global report will accidentally include all Soft Deleted records in the calculation.
13. **How does EF Core handle Global Query Filters on navigation properties via `.Include()`?**
    *Answer:* The filter is automatically applied to the `JOIN` clause. If you `Include(t => t.Sites)`, it generates `LEFT JOIN Sites ON ... AND Sites.IsDeleted = 0`.
14. **How do you inject HTTP context (like a JWT claim) into a DbContext?**
    *Answer:* Create an interface (e.g., `ITenantProvider`), implement it using `IHttpContextAccessor`, register it as a Scoped service in DI, and inject it into the `DbContext` constructor.
15. **What is the performance implication of Global Query Filters on SQL Server Indexes?**
    *Answer:* Every query will now include the filter column (e.g., `TenantId`) in the `WHERE` clause. All non-clustered indexes must be updated to include `TenantId` as the leading key, otherwise SQL Server will perform index scans instead of index seeks.
16. **Why are Repositories often considered an anti-pattern when using strict CQRS with EF Core?**
    *Answer:* Repositories are designed for aggregate roots (Commands). Using a Repository to execute a complex read projection (Query) spanning multiple aggregates forces the Repository to leak domain boundaries or return awkward DTOs. CQRS suggests bypassing Repositories entirely for reads.
17. **Can you configure multiple Global Query Filters on a single entity?**
    *Answer:* No. You can only call `HasQueryFilter` once per entity. To combine them, you must use logical AND: `HasQueryFilter(e => !e.IsDeleted && e.TenantId == _tenantId)`.
18. **If you have an administrative background job that needs to update data for Tenant A, how do you scope the DbContext?**
    *Answer:* You must manually create an implementation of `ITenantProvider` specifically for the background job that returns Tenant A's ID, create a new DI Scope, and resolve the DbContext within that scope so it binds to the correct tenant.
19. **What happens if you use `ExecuteUpdate` on an entity with a Global Query Filter?**
    *Answer:* The filter IS applied. EF Core automatically appends the filter to the generated `UPDATE ... WHERE` clause.
20. **Is it possible to use DbContext Pooling with a dynamic connection string (Database-per-tenant)?**
    *Answer:* No. DbContext Pooling requires all contexts in the pool to share the exact same configuration and connection string.

### Senior (10)
21. **Architect a mechanism to apply Global Query Filters dynamically to all entities without explicitly mapping them one by one in `OnModelCreating`.**
    *Answer:* In `OnModelCreating`, use reflection: `foreach (var entityType in builder.Model.GetEntityTypes())`. Check if `entityType.ClrType` implements `IMustHaveTenant`. If so, use reflection to invoke a generic method that builds the expression tree `e => e.TenantId == _tenantId` and applies it via `builder.Entity(entityType.ClrType).HasQueryFilter()`.
22. **Explain the security vulnerability in Shared-Database multi-tenancy known as "Cross-Tenant ID Guessing" and how EF Core mitigates it.**
    *Answer:* If Tenant A sends a request `PUT /api/chargers/999`, but Charger 999 belongs to Tenant B, a naive `context.Chargers.Update(charger)` will overwrite Tenant B's data. With a Global Query Filter, the `UPDATE` statement is generated as `WHERE Id = 999 AND TenantId = 'TenantA'`. The database returns 0 rows updated, throwing a `DbUpdateConcurrencyException` or silently failing, but mathematically preventing the data breach.
23. **You are implementing a multi-tenant system using a single DbContext. Tenant A requires SQL Server. Tenant B requires PostgreSQL. Is this possible?**
    *Answer:* It is technically possible but architecturally disastrous. You would have to dynamically call `.UseSqlServer()` or `.UseNpgsql()` in `OnConfiguring` based on the tenant. However, EF Core caches its internal `IModel` (metadata) globally per DbContext type. SQL Server and PostgreSQL have vastly different metadata (e.g., Identity vs Sequences). You must implement `IModelCacheKeyFactory` to force EF Core to build and cache a separate internal model for each database provider.
24. **Evaluate the architectural choice of using MediatR to implement CQRS versus standard Controller injection.**
    *Answer:* MediatR forces strict decoupling. The Controller depends only on `IMediator`. It prevents developers from accidentally injecting Domain Services or DbContexts directly into the UI layer. It strictly separates the Request (Command/Query) from the Execution (Handler), making the code highly testable and forcing a rigid CQRS boundary.
25. **How do you handle a scenario where a user (e.g., a Support Tech) belongs to Multiple Tenants and needs to query data across them?**
    *Answer:* The `_tenantId` field must become a `List<Guid> _tenantIds`. The Global Query Filter must be rewritten using `.Contains()`: `HasQueryFilter(e => _tenantIds.Contains(e.TenantId))`. This introduces performance complexities in SQL Server execution plans (IN clauses).
26. **Design a solution to enforce Row-Level Security (RLS) inside SQL Server itself, rather than relying on EF Core Global Query Filters.**
    *Answer:* Create a SQL Server Security Policy and a Predicate Function that filters rows based on `SESSION_CONTEXT()`. In EF Core, before executing any query, you must open the connection and execute `await context.Database.ExecuteSqlRawAsync("EXEC sp_set_session_context 'TenantId', @id", p)`. This pushes security into the database engine, providing defense-in-depth even if a developer uses raw SQL or `IgnoreQueryFilters()`.
27. **What is the danger of using `.IgnoreQueryFilters()` inside a complex query with multiple `.Include()` statements?**
    *Answer:* It removes filters globally for the entire query tree. You cannot selectively ignore the filter for the Root entity but keep it for the Included child entities, or vice-versa. It's an all-or-nothing switch that often leads to unintended data exposure on child collections.
28. **Explain the implementation differences between intercepting `SaveChanges` via overriding the method versus using an `ISaveChangesInterceptor`.**
    *Answer:* Overriding `SaveChangesAsync` in the `DbContext` is tightly coupled but allows easy access to private DbContext fields (like `_tenantId`). `ISaveChangesInterceptor` is a globally registered singleton service. It is highly decoupled but makes accessing scoped state (like the current HTTP Request's TenantId) difficult, requiring complex DI resolution (`Lazy<ITenantProvider>`) within the interceptor.
29. **Why might CQRS Queries using Dapper fail to deserialize JSON columns configured via EF Core's Value Converters?**
    *Answer:* Dapper has no knowledge of EF Core's internal metadata or Value Converters. If EF Core serializes a C# `Address` object to a JSON string in SQL Server, Dapper will read it as a raw string. You must write custom Dapper Type Handlers to manually replicate the deserialization logic, highlighting the maintenance burden of mixing ORMs.
30. **In a Database-per-tenant architecture, how do you execute Entity Framework Migrations?**
    *Answer:* You cannot use `dotnet ef database update`. You must generate an Idempotent SQL script. Then, write a custom DevOps pipeline (or C# utility) that iterates through a centralized database containing a list of all Tenant connection strings, opens a connection to each one sequentially (or in parallel batches), and executes the SQL script using ADO.NET.

### Staff Engineer (5)
31. **Architect a Multi-Tenant system that utilizes a Shared-Database model for free-tier users, but automatically migrates a user to a Database-per-Tenant model when they upgrade to an Enterprise tier, seamlessly switching connection strategies at runtime.**
    *Answer:* The `ITenantProvider` must query a central highly-available "Tenant Directory" cache (e.g., Redis). The cache returns a `TenantConfiguration` object indicating `IsDedicated` and `ConnectionString`. The DbContext's `OnConfiguring` intercepts this. If `IsDedicated` is false, it uses the default connection string. If true, it uses the dedicated connection string. The Global Query Filter is dynamically disabled in `OnModelCreating` if `IsDedicated` is true. The actual data migration requires a distributed Saga to pause tenant traffic, BCP copy data to the new DB, delete from the shared DB, update the Directory cache, and route traffic.
32. **Analyze the execution plan degradation caused by `HasQueryFilter` when querying a heavily partitioned SQL Server table using EF Core.**
    *Answer:* If a massive table is partitioned by `CreatedAt` (Date), but the EF Core Global Query Filter appends `WHERE TenantId = X`, SQL Server might suffer from "Partition Elimination Failure". If the index is not aligned with the partition function, SQL Server must scan every single partition looking for that `TenantId`. The Architect must ensure that all heavily partitioned tables include the TenantId in the partitioning function, or explicitly design indexes that cover the query filter to allow index seeks within individual partitions.
33. **Design a CQRS Event Sourcing architecture where EF Core is strictly used as the Read Model projection engine, while a NoSQL database stores the Event Stream.**
    *Answer:* The Command stack completely bypasses EF Core, writing raw JSON events (e.g., `SiteCreated`, `SiteNameChanged`) to an Event Store (like EventStoreDB or CosmosDB). A background worker subscribes to the Event Store stream. When an event arrives, the worker uses EF Core to update a heavily denormalized SQL Server Read Model. EF Core is utilized purely for its Change Tracker and SQL generation capabilities to maintain the projected state (`context.Sites.Update(...)`). The Read API queries this SQL database using Dapper or EF Core `.AsNoTracking()`.
34. **Evaluate the security and performance implications of using SQL Server Row-Level Security (RLS) versus EF Core Global Query Filters in a multi-tenant application with 50,000 tenants.**
    *Answer:* EF Core Query Filters are evaluated at compile time; the SQL string is modified in RAM. It has zero database overhead beyond normal index requirements. SQL Server RLS requires executing a Predicate Function on *every single row evaluated by the execution plan*. For 50,000 tenants, RLS can introduce catastrophic CPU overhead on the database server during table scans. However, RLS provides absolute defense-in-depth, protecting against compromised developer credentials accessing the database via SSMS. The Architect must weigh application-layer performance vs database-layer paranoia.
35. **A development team wants to implement GraphQL on top of an EF Core DbContext to allow clients to define their own queries. Architecturally critique this decision in the context of CQRS and Enterprise Security.**
    *Answer:* It is an architectural disaster. GraphQL directly exposes the data model to the client. This violates CQRS because it bypasses optimized Read Models. It destroys performance because malicious clients can request infinitely deep object graphs (`tenant -> sites -> chargers -> sessions -> charger -> ...`), triggering massive Cartesian Explosions or N+1 queries that EF Core attempts to fulfill. Furthermore, enforcing complex aggregate-level security rules dynamically on arbitrary GraphQL ASTs is incredibly difficult. An Architect must place a strict DTO/Resolver boundary between GraphQL and EF Core, never allowing the client to query `IQueryable` directly.

## 15. Exercises

### Easy
1.  **Global Query Filter:** Add an `IsDeleted` boolean property to a `Charger` entity. In `OnModelCreating`, configure a Global Query Filter so that queries only return chargers where `IsDeleted == false`. Write a LINQ query and verify the generated SQL includes the filter.

### Medium
1.  **Bypassing Filters:** Write a query that fetches all `Chargers`, including the deleted ones, by using `.IgnoreQueryFilters()`.
2.  **Multi-Tenancy Injection:** Create a dummy `ITenantProvider` that always returns a hardcoded `Guid`. Inject it into your `DbContext`. Configure a Global Query Filter on the `Site` entity using this injected ID.

### Hard
1.  **Automatic Auditing:** Override `SaveChangesAsync` in your `DbContext`. Iterate through the Change Tracker. For every entity in the `Modified` state, automatically update a `LastModifiedAt` DateTime property to `DateTime.UtcNow`.

### Enterprise
1.  **Strict CQRS:** Create a MediatR `CommandHandler` that creates a new `Site` using EF Core. Create a separate MediatR `QueryHandler` that retrieves a list of Sites using a Micro-ORM approach (either Dapper, or EF Core using raw SQL `context.Database.SqlQuery<T>`). Ensure the Read models (DTOs) are completely separate from the Domain models.

## 16. Production Checklist

- [ ] Are Global Query Filters implemented to enforce row-level Multi-Tenancy security?
- [ ] Has the `ITenantProvider` (or similar contextual state) been injected securely without utilizing dangerous static variables?
- [ ] Are all database indexes optimized to include the `TenantId` (or query filter column) as the leading key to prevent full table scans?
- [ ] Is CQRS strictly enforced, ensuring complex read-queries bypass the Domain Layer and Change Tracker?
- [ ] Is DbContext Pooling disabled if the DbContext constructor retains tenant-specific state?

## 17. Summary

An Enterprise application cannot rely on developer discipline to enforce security; it must rely on architecture. By implementing Global Query Filters, we push multi-tenant data isolation down to the compiler level. By adopting CQRS, we decouple the heavy transactional logic of our write-paths from the ultra-fast, flattened data requirements of our read-paths.

We have mastered the relational aspects of EF Core. But modern applications are rarely purely relational. In the final chapter, we will cross the boundary into NoSQL, exploring how modern EF Core bridges the gap between relational integrity and document-database flexibility.
