# Chapter 3: Advanced Entity Modeling and the Fluent API

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Defend the architectural mandate of utilizing the Fluent API over Data Annotations to maintain a pristine Domain Layer.
*   Master the `OnModelCreating` pipeline to configure complex constraints, default values, and column mappings.
*   Implement Value Converters to seamlessly map complex C# value objects and JSON configurations to standard SQL columns.
*   Utilize Shadow Properties to inject infrastructure-level data (e.g., Auditing timestamps) without polluting domain entities.
*   Configure Backing Fields to strictly enforce encapsulation and Domain-Driven Design (DDD) invariants.
*   Optimize application startup time using Compiled Models.

## 2. Introduction

Entity Framework Core operates on convention by default. If you create a class named `Charger` with a property named `Id`, EF Core assumes it maps to a table named `Chargers` with a Primary Key column named `Id`.

However, enterprise databases rarely align perfectly with default conventions. Legacy databases have esoteric naming schemes. Modern SaaS platforms require mapping complex JSON payloads into single columns, or applying strict composite uniqueness constraints across Tenant IDs.

To mold EF Core to fit the exact shape of your database, you must explicitly model it. Historically, developers used Data Annotations (`[Table]`, `[Column]`, `[Required]`). While convenient for simple CRUD apps, Data Annotations violate the core tenets of Clean Architecture by tightly coupling your Domain entities to the `System.ComponentModel.DataAnnotations` and EF Core namespaces.

This chapter explores the Architect's weapon of choice: **The Fluent API**. We will dissect how EF Core builds its internal model, and how you can override that model to map complex value objects, enforce strict encapsulation, and bend the database schema to your exact will.

## 3. Core Concepts

### Data Annotations vs. The Fluent API
*   **Data Annotations:** C# attributes placed directly on entity classes. They are intrusive, limited in scope (cannot express complex index configurations or shadow properties), and mix data-access concerns with business logic.
*   **The Fluent API:** A method-chaining API used exclusively inside the `DbContext.OnModelCreating` method (or via `IEntityTypeConfiguration<T>`). It is completely non-intrusive, keeps Domain entities pure (POCOs), and offers 100% of EF Core's configuration surface area.

### Value Converters
A mechanism that instructs EF Core how to convert a property value before sending it to the database, and how to convert it back when reading from the database. Essential for mapping Enums to strings, Domain Value Objects to primitives, or objects to JSON strings.

### Shadow Properties
Properties that do not exist in your C# entity class but do exist in the EF Core model and the underlying database table. They are managed entirely via the `DbContext` API. Ideal for tracking infrastructure data like `CreatedBy`, `LastModifiedAt`, or `TenantId` without bloating the Domain logic.

### Backing Fields
In Domain-Driven Design (DDD), properties often have `private` setters or expose `IReadOnlyCollection<T>` instead of `List<T>`. EF Core must bypass these encapsulation boundaries to materialize data from the database. Configuring Backing Fields instructs EF Core to write data directly to the private C# field (e.g., `_chargers`) rather than using the public property setter.

### Keyless Entities
Not everything in a database is a table with a primary key. Views, Table-Valued Functions, or raw SQL queries might return data without a unique identifier. EF Core can map to these using Keyless Entities, which bypass the Change Tracker entirely.

## 4. Visual Diagrams

```text
=============================================================================
             THE EF CORE MODEL BUILDING PIPELINE
=============================================================================

[ Application Startup (First DbContext Instantiation) ]
       │
       ▼
[ 1. Discovery ] ──────▶ Scans DbSets and Navigation Properties
       │
       ▼
[ 2. Conventions ] ────▶ Applies default rules (Id = PK, string = nvarchar(max))
       │
       ▼
[ 3. Annotations ] ────▶ Overrides conventions based on C# Attributes
       │
       ▼
[ 4. Fluent API ] ─────▶ (OnModelCreating) Highest precedence. Overrides everything.
       │                 Executes IEntityTypeConfiguration<T> classes.
       ▼
[ 5. Model Validation] ─▶ Checks for mapping errors (e.g., missing keys).
       │
       ▼
[ IModel ] ────────────▶ The finalized, immutable Metadata Model (Cached for lifetime).
```

## 5. API Deep Dive: The Fluent API

The Fluent API is accessed via the `ModelBuilder` inside the `OnModelCreating` method of your `DbContext`.

### Basic Configuration (`EntityTypeBuilder`)
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Access the configuration builder for the 'Tenant' entity
    modelBuilder.Entity<Tenant>(builder =>
    {
        // Map to a specific schema and table name
        builder.ToTable("Tenants", "saas");

        // Override the Primary Key convention
        builder.HasKey(t => t.TenantCode);

        // Configure Property Constraints
        builder.Property(t => t.Name)
            .IsRequired()
            .HasMaxLength(255)
            .HasColumnName("TenantName")
            .HasColumnType("varchar(255)"); // Force non-unicode

        // Configure a Composite Index
        builder.HasIndex(t => new { t.Region, t.IsActive })
            .HasDatabaseName("IX_Tenants_Region_Active");
    });
}
```

### Implementing `IEntityTypeConfiguration<T>`
Putting all configuration in `OnModelCreating` creates a massive, unmaintainable method. The architectural best practice is to separate configuration into distinct classes.

```csharp
// Infrastructure/Data/Configurations/TenantConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        builder.ToTable("Tenants");
        builder.HasKey(t => t.Id);
        // ... specific tenant configuration
    }
}

// In DbContext:
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Dynamically applies ALL configurations found in the executing assembly
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(EvDbContext).Assembly);
}
```

## 6. Complete Examples: EV Platform Case Study

Let's model the `Charger` entity using advanced DDD principles and map it flawlessly to SQL Server.

### The Domain Entity (Pure C#)
```csharp
// Domain/Entities/Charger.cs
public class Charger
{
    // Encapsulation: Private setter
    public Guid Id { get; private set; }
    
    public Guid SiteId { get; private set; }
    
    // Encapsulation: Value Object
    public SerialNumber Serial { get; private set; } 
    
    // Encapsulation: Complex object we want stored as JSON in a single column
    public HardwareConfig Configuration { get; private set; } 

    // Constructor required by EF Core (can be private)
    private Charger() { } 

    public Charger(Guid siteId, SerialNumber serial, HardwareConfig config)
    {
        Id = Guid.NewGuid();
        SiteId = siteId;
        Serial = serial;
        Configuration = config;
    }
}

// A C# Record acting as a Value Object
public record SerialNumber(string Value);

public class HardwareConfig 
{
    public int MaxKw { get; set; }
    public string FirmwareVersion { get; set; } = string.Empty;
}
```

### The EF Core Configuration
We must map this complex domain model to a flat SQL Server table (`Chargers`).

```csharp
// Infrastructure/Data/Configurations/ChargerConfiguration.cs
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class ChargerConfiguration : IEntityTypeConfiguration<Charger>
{
    public void Configure(EntityTypeBuilder<Charger> builder)
    {
        builder.ToTable("Chargers");
        builder.HasKey(c => c.Id);

        // 1. VALUE CONVERTER: Map 'SerialNumber' Record to a standard SQL string
        builder.Property(c => c.Serial)
            .HasConversion(
                serial => serial.Value, // Write to DB
                dbString => new SerialNumber(dbString) // Read from DB
            )
            .HasColumnName("SerialNumber")
            .HasMaxLength(50)
            .IsRequired();

        // 2. JSON CONVERTER: Map the complex HardwareConfig to a JSON string column
        builder.Property(c => c.Configuration)
            .HasConversion(
                config => JsonSerializer.Serialize(config, (JsonSerializerOptions)null),
                json => JsonSerializer.Deserialize<HardwareConfig>(json, (JsonSerializerOptions)null)!
            )
            .HasColumnType("nvarchar(max)")
            .HasColumnName("HardwareConfigJson");

        // 3. SHADOW PROPERTY: Add tracking data without modifying the Domain class
        builder.Property<DateTime>("LastModifiedAt")
            .HasColumnType("datetime2");
            
        builder.Property<string>("ModifiedBy")
            .HasMaxLength(100);
    }
}
```

### Interacting with Shadow Properties
Because `LastModifiedAt` doesn't exist on the `Charger` class, how do we query or update it? We use the `EF.Property` static method in LINQ, and the `ChangeTracker` API for updates.

```csharp
// Querying a shadow property
var staleChargers = await context.Chargers
    .Where(c => EF.Property<DateTime>(c, "LastModifiedAt") < DateTime.UtcNow.AddDays(-30))
    .ToListAsync();

// Updating a shadow property (Usually done inside a SaveChangesInterceptor)
var entry = context.Entry(charger);
entry.Property("LastModifiedAt").CurrentValue = DateTime.UtcNow;
entry.Property("ModifiedBy").CurrentValue = "System";
```

## 7. Backing Fields for DDD Collections

In DDD, you should never expose a `List<T>` that someone can arbitrarily `.Add()` to. You expose a read-only collection and a specific method to manipulate the data.

```csharp
// Domain
public class Site
{
    private readonly List<Charger> _chargers = new(); // Backing Field
    
    // Public exposure is read-only
    public IReadOnlyCollection<Charger> Chargers => _chargers.AsReadOnly();

    public void AddCharger(Charger charger)
    {
        // Business logic and validation goes here
        if (_chargers.Count >= 10) throw new Exception("Max capacity reached.");
        _chargers.Add(charger);
    }
}
```

**The Mapping Issue:** When EF Core queries the database, it tries to populate the `Chargers` property. But it is read-only. It will fail. We must tell EF Core to write directly to the `_chargers` private field.

```csharp
// Configuration
builder.Navigation(s => s.Chargers)
    .HasField("_chargers") // Tell EF Core to use the private field
    .UsePropertyAccessMode(PropertyAccessMode.Field); // Bypass the public property getter/setter entirely during materialization
```
*Note: EF Core 9 is remarkably smart and can often auto-detect backing fields if they follow standard naming conventions (e.g., `_chargers`), but explicit configuration is safer.*

## 8. Performance Implications

### Value Converters and LINQ Translation
Value Converters are extremely powerful, but they have a severe performance caveat: **EF Core cannot always translate them inside LINQ `Where` clauses.**

If you have a Value Converter that encrypts a string (e.g., `Name` -> Encrypted `DB_String`), and you write `context.Users.Where(u => u.Name == "John")`, SQL Server does not know how to encrypt "John". EF Core will attempt to encrypt the parameter locally and pass it, which works for strict equality. However, if you write `context.Users.Where(u => u.Name.Contains("John"))`, EF Core cannot translate this. It will either throw an exception or evaluate it client-side (pulling the whole table into memory). 

*Architectural Rule:* Be highly cautious when applying Value Converters to columns that you need to search or filter against using complex string operations.

### Compiled Models (Startup Performance)
Building the `IModel` (discovering DbSets, parsing configurations, validating the graph) takes time. In a massive enterprise context with 500 entities, `OnModelCreating` might take 2-5 seconds. In a serverless environment (Azure Functions) where Cold Start time is critical, this is unacceptable.

EF Core provides **Compiled Models**. You execute a CLI command (`dotnet ef dbcontext optimize`) during your CI/CD build process. EF Core generates physical C# classes that represent the compiled `IModel`.

```csharp
// In Program.cs, instruct EF Core to bypass OnModelCreating and use the pre-compiled C# model
builder.Services.AddDbContext<EvDbContext>(options =>
    options.UseSqlServer("...")
           .UseModel(EvDbContextModel.Instance)); // Drastically reduces startup time
```

## 9. ASP.NET Core Integration

When organizing an ASP.NET Core solution, place your `IEntityTypeConfiguration<T>` classes in the same assembly as your `DbContext` (the Infrastructure Layer). 

Do NOT load configurations manually in `OnModelCreating` like this:
`modelBuilder.ApplyConfiguration(new ChargerConfig());` // Bad for 100 entities.

Use Assembly scanning. It is fast and maintenance-free:
`modelBuilder.ApplyConfigurationsFromAssembly(typeof(EvDbContext).Assembly);`

## 10. Clean Architecture Perspective

Clean Architecture dictates that the Domain Layer is at the center and depends on nothing. 
If you use Data Annotations (`[Table("Users")]`), your Domain entities must reference the `System.ComponentModel.DataAnnotations` namespace. If you use EF Core specific annotations (`[Owned]`, `[Keyless]`), your Domain must reference the `Microsoft.EntityFrameworkCore` NuGet package.

This completely violates Clean Architecture. By strictly enforcing the use of the Fluent API via `IEntityTypeConfiguration<T>` housed in the Infrastructure Layer, your Domain entities remain 100% pure, ignorant of whether they are being saved to SQL Server via EF Core, an Oracle database via NHibernate, or a NoSQL document database.

## 11. Enterprise SaaS Perspective: Encrypted Data

In a SaaS environment, compliance (e.g., GDPR, HIPAA) may require certain columns (like a user's SSN or an EV Driver's payment token) to be encrypted at rest natively within the application (client-side encryption), rather than relying on Transparent Data Encryption (TDE).

Value Converters are the perfect mechanism for this.

```csharp
public class DriverConfiguration : IEntityTypeConfiguration<Driver>
{
    public void Configure(EntityTypeBuilder<Driver> builder)
    {
        var encryptionService = new AesEncryptionService("SECRET_KEY_FROM_VAULT");

        builder.Property(d => d.PaymentToken)
            .HasConversion(
                plainText => encryptionService.Encrypt(plainText),
                cipherText => encryptionService.Decrypt(cipherText)
            )
            .HasMaxLength(500); 
    }
}
```
*Every time you save, it encrypts. Every time you read, it decrypts. The Domain entity only ever sees the plain-text token.*

## 12. Real Production Case Study

In our EV Platform, different hardware vendors send vastly different configuration payloads. We cannot normalize this into strict SQL tables. We must store the `HardwareConfig` as a JSON string in SQL Server, but interact with it as a strongly typed C# object.

By using the Value Converter (as demonstrated in Section 6), we achieve exactly this. However, in EF Core 7+, Microsoft introduced native JSON column mapping using `.ToJson()`.

```csharp
// EF Core 7+ Native JSON Mapping (Superior to Value Converters for JSON)
builder.OwnsOne(c => c.Configuration, configBuilder => 
{
    configBuilder.ToJson(); // Native SQL Server JSON support!
    configBuilder.Property(c => c.MaxKw).HasColumnName("MaxKw");
});
```
This native `.ToJson()` mapping is vastly superior to the Value Converter approach for JSON because EF Core actually translates LINQ queries *into* JSON path queries in SQL Server. (e.g., `WHERE c.Configuration.MaxKw > 50` translates to `WHERE JSON_VALUE(Configuration, '$.MaxKw') > 50`).

## 13. Common Mistakes

### Beginner
*   **Mistake:** Putting logic inside the empty, parameterless constructor of an entity.
*   **Correction:** EF Core calls the parameterless constructor when it materializes data from the database. If you have `CreatedAt = DateTime.UtcNow` in that constructor, EF Core will overwrite the database value with the current time during materialization before it maps the database properties. Leave the parameterless constructor completely empty.

### Intermediate
*   **Mistake:** Using a Value Converter to map a list of primitives (e.g., `List<int> Tags`) to a comma-separated string.
*   **Correction:** While this works, it violates the First Normal Form (1NF) of relational databases. You can never query `WHERE Tags CONTAINS 5` efficiently using SQL indexes. It forces full table scans. Use a proper related table, or native JSON mapping if the database supports indexing JSON arrays.

### Senior
*   **Mistake:** Forgetting to configure `.IsRequired()` and `.HasMaxLength()` on strings in the Fluent API.
*   **Correction:** By default, EF Core maps C# `string` to `NVARCHAR(MAX) NULL`. This creates massive, unoptimized columns in SQL Server that cannot be included in standard B-Tree indexes. Every string property should have a max length and nullability explicitly configured.

### Architect
*   **Mistake:** Allowing developers to use `modelBuilder.Entity<T>()` repeatedly inside a monolithic `OnModelCreating` method in a context with 200 tables.
*   **Correction:** This creates merge conflict hell in Git and makes the code unreadable. The Architect must enforce the `IEntityTypeConfiguration<T>` pattern and Assembly scanning to segregate infrastructure configuration cleanly.

## 14. Interview Questions

### Beginner (10)
1.  **What is the difference between Data Annotations and the Fluent API?**
    *Answer:* Data Annotations are attributes placed directly on C# classes. The Fluent API is a method-chaining configuration done in `OnModelCreating` that keeps the C# classes clean of DB concerns.
2.  **How do you tell EF Core that a property should be the Primary Key using the Fluent API?**
    *Answer:* `builder.HasKey(x => x.Id);`
3.  **What does `HasMaxLength(50)` do to a string property?**
    *Answer:* It configures the generated SQL Server column to be `VARCHAR(50)` or `NVARCHAR(50)` instead of `NVARCHAR(MAX)`.
4.  **What is a Value Converter?**
    *Answer:* A mechanism to translate a property from one type in C# to a different type in the database (e.g., converting a boolean to "Y" or "N").
5.  **What is a Shadow Property?**
    *Answer:* A property defined in the EF Core model that exists in the database table but does not exist in the C# entity class.
6.  **Why does EF Core require a parameterless constructor on entities?**
    *Answer:* Because when it reads data from the database, it uses reflection to instantiate the object first, before setting the properties.
7.  **How do you configure EF Core to map an entity to a specific database schema (e.g., "auth")?**
    *Answer:* `builder.ToTable("Users", "auth");`
8.  **What is a Backing Field?**
    *Answer:* A private field in a C# class (e.g., `_name`) that stores the data for a public property (e.g., `Name`). EF Core can be configured to read/write directly to the field.
9.  **What does `ApplyConfigurationsFromAssembly` do?**
    *Answer:* It scans the specified assembly for all classes that implement `IEntityTypeConfiguration<T>` and automatically applies their configurations to the `ModelBuilder`.
10. **Can you map an Enum to a string column in the database using EF Core?**
    *Answer:* Yes, using a Value Converter (e.g., `HasConversion<string>()`).

### Intermediate (10)
11. **Explain why Data Annotations violate Clean Architecture.**
    *Answer:* Clean Architecture dictates that Domain entities should have no dependencies on infrastructure. Using `[Table]` or `[Column]` forces the Domain project to reference `System.ComponentModel.DataAnnotations`, coupling it to a specific persistence paradigm.
12. **How do you query a Shadow Property in a LINQ statement if it doesn't exist on the C# class?**
    *Answer:* By using the static method: `EF.Property<T>(entity, "PropertyName") == value`.
13. **What is the purpose of a Keyless Entity (`HasNoKey()`)?**
    *Answer:* It allows EF Core to map query results (like SQL Views or raw SQL queries) to a C# object, but because it has no primary key, it cannot be tracked by the Change Tracker for updates or deletes.
14. **How do you configure a Composite Primary Key using the Fluent API?**
    *Answer:* `builder.HasKey(x => new { x.TenantId, x.UserId });`
15. **If you have a private setter on an `Id` property, can EF Core still populate it from the database?**
    *Answer:* Yes. EF Core uses reflection and can bypass private setters to hydrate the object.
16. **What is a Sequence in SQL Server, and how do you configure it in EF Core?**
    *Answer:* A database object that generates numeric sequences. You configure it via `modelBuilder.HasSequence<int>("MySequence")` and then use `HasDefaultValueSql("NEXT VALUE FOR MySequence")` on a property.
17. **How do you configure a unique constraint (Unique Index) in the Fluent API?**
    *Answer:* `builder.HasIndex(x => x.Email).IsUnique();`
18. **Explain the difference between `HasColumnType("datetime2")` and standard EF Core DateTime mapping.**
    *Answer:* Older versions of EF Core mapped `DateTime` to SQL Server `datetime`. `datetime2` has a larger date range and higher fractional second precision. Modern EF Core defaults to `datetime2`, but explicit configuration is often used for clarity or overriding precision.
19. **How do you configure EF Core to ignore a specific property on your C# class so it isn't mapped to the database?**
    *Answer:* `builder.Ignore(x => x.CalculatedTotal);`
20. **What is the `UsePropertyAccessMode` configuration used for?**
    *Answer:* It dictates whether EF Core should read/write data using the public Property getter/setter, or bypass it and read/write directly to the Backing Field during materialization and change tracking.

### Senior (10)
21. **Analyze the performance implications of using a Value Converter to map a complex object to a JSON string versus using the native `.ToJson()` feature in EF Core 7+.**
    *Answer:* A Value Converter maps the object opaquely; SQL Server treats it as a standard string. If you write a LINQ query filtering on a property inside that JSON, EF Core cannot translate it to SQL; it evaluates it client-side. The native `.ToJson()` mapping allows EF Core to translate LINQ queries directly into SQL Server `JSON_VALUE` or `OPENJSON` syntax, allowing the database engine to perform the filtering efficiently.
22. **You need to implement a Multi-Tenant architecture where Tenant A and Tenant B use the exact same tables, but their data must be strictly isolated. How do you implement this at the EF Core Model configuration level?**
    *Answer:* You define a Shadow Property for `TenantId` on all entities. Then, you use `builder.HasQueryFilter(e => EF.Property<Guid>(e, "TenantId") == _tenantContext.CurrentTenantId)` to apply a Global Query Filter. EF Core will invisibly append this `WHERE` clause to every single query.
23. **What is an `IModelCacheKeyFactory` and when would you need to implement a custom one?**
    *Answer:* EF Core caches the metadata model after it builds it the first time. If your schema needs to change dynamically at runtime (e.g., dynamically assigning different table names or schemas based on the current HTTP request/Tenant), you must implement a custom factory that includes the TenantId in the cache key. This forces EF Core to build and cache a separate execution model for each tenant.
24. **Explain how EF Core instantiates objects that do not have parameterless constructors.**
    *Answer:* EF Core analyzes parameterized constructors. If the constructor parameters perfectly match the names (case-insensitive) of the mapped properties in the database, EF Core will invoke that specific constructor, passing in the database values directly.
25. **How do you map a database View to an EF Core entity without EF Core trying to create a table for it during Migrations?**
    *Answer:* You use `builder.ToView("ViewName")`. This maps the entity to the view for querying, and explicitly excludes it from migration table-creation scripts.
26. **What is the purpose of Compiled Models (`dotnet ef dbcontext optimize`)?**
    *Answer:* In applications with hundreds of entities, building the `IModel` using reflection during startup can take seconds, causing slow cold starts (critical in serverless). Compiled models generate pre-compiled C# classes representing the model metadata, bypassing the runtime discovery phase and drastically reducing startup time.
27. **How do you configure a default value for a column that is only evaluated on the database server (e.g., `GETUTCDATE()`)?**
    *Answer:* `builder.Property(x => x.CreatedAt).HasDefaultValueSql("GETUTCDATE()");`. This instructs EF Core not to send a value on INSERT, allowing SQL Server to generate the default.
28. **If you use a Backing Field (`_items`) and expose an `IReadOnlyCollection<T> Items`, how does the Change Tracker detect additions to the collection if it can't monitor the `.Add()` method of the private field?**
    *Answer:* EF Core wraps the collection in a specialized tracking collection internally (if tracking proxies are enabled) or uses Snapshot comparison during `DetectChanges()`. It compares the contents of the `_items` field during `SaveChanges` against the original snapshot to determine what was added or removed.
29. **Evaluate the use of `ValueGeneratedOnAddOrUpdate()` in the Fluent API.**
    *Answer:* This tells EF Core that the database generates a value for this column upon both `INSERT` and `UPDATE` (e.g., a RowVersion timestamp or a computed column). Consequently, EF Core will automatically append a `RETURNING` or `OUTPUT` clause to its T-SQL statements to read the newly generated value back into the C# entity after the mutation.
30. **How do you prevent a specific property from being updated once it is inserted?**
    *Answer:* `builder.Property(x => x.CreatedBy).Metadata.SetAfterSaveBehavior(PropertySaveBehavior.Ignore);`. This ensures that even if the property is modified in C#, EF Core will not include it in the generated `UPDATE` statement.

### Staff Engineer (5)
31. **Architect a domain-driven EF Core mapping strategy for a `Money` Value Object (containing `Amount` and `Currency`) that is used across 50 different entities, ensuring it maps to two distinct columns (`Price_Amount`, `Price_Currency`) without duplicating configuration code.**
    *Answer:* You cannot use a simple Value Converter because it maps to a single column. You must use EF Core **Owned Types** (`OwnsOne`). To avoid duplicating `builder.OwnsOne(x => x.Price)` 50 times, the Architect should create a custom extension method on `EntityTypeBuilder` or implement a convention-based model building hook (using the metadata API in `OnModelCreating`) that scans all entities for properties of type `Money` and automatically configures the `OwnsOne` mapping and column name prefixes dynamically.
32. **A legacy SQL Server database uses a massively complex composite primary key consisting of 5 columns. Analyze the impact of mapping this directly in EF Core on the performance of Identity Resolution and the Change Tracker.**
    *Answer:* EF Core handles composite keys well syntactically (`HasKey(e => new { e.K1, e.K2...})`). However, Identity Resolution relies on looking up entities in a dictionary by their Key. A 5-part composite key requires EF Core to instantiate an anonymous object or a `Tuple` to represent the key, and compute a complex hash code for every dictionary lookup. Under high-throughput materialization (e.g., querying 10,000 rows), the CPU overhead of this hashing and equality comparison will cause severe performance degradation compared to a single integer or GUID key.
33. **Design a solution using EF Core Shadow Properties and Interceptors to implement a row-level auditing system that tracks not just who modified the row, but also stores a JSON payload of exactly which columns changed.**
    *Answer:* Do not use shadow properties for the JSON payload, as it shouldn't be stored on the business table. Instead, define Shadow Properties for `ModifiedBy` and `ModifiedAt` on all entities via a base configuration interface. Create an `SaveChangesInterceptor`. During `SavingChanges`, iterate over the `ChangeTracker.Entries`. For `Modified` entries, use the `PropertyEntry.IsModified` flag to detect changes. Construct a JSON object comparing `OriginalValue` and `CurrentValue`. Write the `ModifiedBy` to the Shadow Property (which updates the main table), and simultaneously inject a new `AuditLog` entity into the `DbContext` containing the JSON payload, committing both atomically.
34. **You are using Compiled Models to speed up Azure Function cold starts. A developer adds a new Global Query Filter dynamically based on the HTTP Request headers. The filter fails to apply in production. Why?**
    *Answer:* Compiled Models freeze the EF Core metadata model at compile time. Dynamic model configuration (like evaluating a DI service during `OnModelCreating` to build a dynamic query filter) is explicitly incompatible with Compiled Models, because `OnModelCreating` is bypassed entirely at runtime. To fix this, you must rely on standard static Global Query Filters that evaluate state via an injected service (e.g., `_tenantService.Id`), which is evaluated at query execution time, not model building time.
35. **Evaluate the architectural boundary violation of using EF Core's `ILazyLoader` service injected directly into a Domain Entity's constructor to support Lazy Loading without proxy generation.**
    *Answer:* Injecting `ILazyLoader` (an EF Core specific interface) directly into a Domain Entity's constructor is a catastrophic violation of Clean Architecture and Domain-Driven Design. It tightly couples the Domain core to the specific ORM technology. The Domain entity can no longer be unit tested or moved to another project without referencing Entity Framework. The Architect must categorically reject this and mandate Eager Loading (`Include`), Explicit Loading, or true lazy-loading proxies that do not pollute the domain code.

## 15. Exercises

### Easy
1.  **Fluent API Basics:** Remove all `[Table]`, `[Key]`, and `[Column]` Data Annotations from a C# entity. Create an `IEntityTypeConfiguration<T>` class and re-implement the exact same mappings using the Fluent API.

### Medium
1.  **Value Converters:** Create an enum `ChargerStatus { Available, Charging, Faulted }`. In your entity, use this Enum. In your Fluent API configuration, use `HasConversion<string>()` to ensure it saves to the database as "Available" rather than the integer `0`. Verify the generated schema using Migrations.

### Hard
1.  **Shadow Properties:** Use the Fluent API to add a `DateTime` shadow property named `CreatedTimestamp` to an entity. Write a query that filters the database returning only entities where `CreatedTimestamp` is older than 7 days, using the `EF.Property` method.

### Enterprise
1.  **Strict Encapsulation:** Design a `Site` entity that contains a private `_name` field, and a public `Name` property with only a `get` accessor. Provide a `Rename(string newName)` method that enforces business validation (e.g., Name cannot be empty) before updating the private field. Configure EF Core to write directly to the `_name` backing field when materializing from the database, completely bypassing the business validation during database reads.

## 16. Production Checklist

- [ ] Are all EF Core configurations contained within `IEntityTypeConfiguration<T>` classes instead of monolithic `OnModelCreating` methods?
- [ ] Are Domain Entities 100% free of `[Table]`, `[Column]`, and other EF Core-specific attributes?
- [ ] Have all `string` properties been explicitly configured with `HasMaxLength()` to avoid `NVARCHAR(MAX)` performance penalties?
- [ ] Are JSON configurations mapped using the native `.ToJson()` (EF7+) rather than Value Converters to ensure LINQ translatability?
- [ ] If the application has > 100 entities, have Compiled Models been generated to optimize cloud cold-start times?

## 17. Summary

The Fluent API is the bridge between the pristine purity of Domain-Driven Design and the harsh physical reality of relational databases. By mastering `IEntityTypeConfiguration<T>`, Value Converters, and Backing Fields, you can design your C# domain exactly as it should be, without compromising on how the data is efficiently stored in SQL Server.

We have now modeled single entities. However, enterprise data is inherently relational. In the next chapter, we will tackle the most complex and error-prone aspect of Entity Framework Core: configuring and querying relationships, navigation properties, and inheritance hierarchies.
