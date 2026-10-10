# Module 2: Building Your First Data Model

In this module, we will explore how to configure our C# domain classes (like `Tenant` and `User` in Normora) so that EF Core understands exactly how to map them to the database. We'll also cover the best practices for defining these mappings.

---

## 1. How DbContext Works (The Mental Model)
**`DbContext` is the base class that EF Core offers for implementing the mapping between the code side and the database.** It acts as the mediator orchestrating how your objects correspond to table rows.

The `DbContext` works based on **mappings** between our C# classes and our database tables. 

When the server starts and the `DbContext` is spun up, it builds a **mental model** of the relationship between your classes and the database. Based on your models and EF Core conventions, it knows the exact shape of the data in the database. 

Because it builds this mental model once, it does not need to rediscover the mappings every time you make a request. Any changes you make to your classes/models are tracked inside the `DbContext` and then propagated to the database on the `SaveChangesAsync()` method call.

### The DbContext is Database Agnostic
When you write C# code and LINQ queries against the `DbContext`, **you do not have to know which database you are using.** The LINQ statements remain exactly the same whether you are using SQL Server, PostgreSQL, or SQLite.

**Why?** EF Core uses a **Provider Model**. When you write a query in C#, EF Core translates it into an Abstract Syntax Tree. The specific database provider (e.g., `Npgsql` for Postgres or `Microsoft.EntityFrameworkCore.SqlServer` for SQL Server) is responsible for taking that tree and translating it into the exact SQL dialect required. This abstraction protects your application code from being tightly coupled to a specific database engine.

---

## 2. The DbSet<T> and Null-Forgiving Operator

As we saw in Module 1, `DbSet<T>` represents the collection of entities in our context. But when defining it, you might run into C# compiler warnings about uninitialized non-nullable properties.

```csharp
public class TenantsDbContext : DbContext
{
    // Approach 1: The null-forgiving operator (= null!)
    public DbSet<Tenant> Tenants { get; set; } = null!;

    // Approach 2: Using Set<T>() directly
    public DbSet<User> Users => Set<User>();
}
```

### Understanding DbSet Initialization

In modern C#, declaring a non-nullable property (like `DbSet<Tenant>`) without initializing it in the constructor throws a compiler warning (CS8618). Since EF Core handles the initialization of these properties behind the scenes, we need a way to satisfy the compiler. Here are the two standard approaches:

#### 1. Approach 1: The Null-Forgiving Operator (`= null!`)
*   **How:** You define a standard auto-implemented property but assign `= null!` at the end.
*   **Why:** The `= null!` syntax is a directive to the C# compiler that essentially says: *"I know this looks like it will be null, but trust me, a framework (EF Core) will initialize it via reflection before I ever use it."* It suppresses the warning without actually initializing the object in your code.
*   **Benefits:** It keeps the standard `{ get; set; }` syntax that C# developers are deeply familiar with, while correctly delegating the actual object creation to EF Core's internal lifecycle.

#### 2. Approach 2: Using `Set<T>()` Directly
*   **How:** You define an expression-bodied property (`=>`) that delegates directly to the base `DbContext`'s `Set<T>()` method.
*   **Why:** `Set<T>()` is an EF Core method that returns the `DbSet` associated with the current context. Because you are using an expression body (`=>`), there is no backing field stored in memory for this property. Since there is no backing field, the compiler doesn't expect you to initialize it, naturally bypassing the warning.
*   **Benefits:** You don't have to use a "hacky" null-forgiving operator. It is arguably cleaner and safer because it actively calls a base class method instead of just suppressing a compiler warning.

*Note:* `DbSet<Tenant>` represents the entity collection in our code, **not** the actual database records. EF Core only retrieves records when you explicitly execute a query.

---

## 2. Defining Constraints: Data Annotations vs. Fluent API

Once we have our `Tenant` class, we need to define constraints (e.g., maximum length of a string, or specifying the table name). There are two ways to do this: Data Annotations (Attributes) and the Fluent API.

### The Problem with Data Annotations
Data Annotations involve decorating your domain classes with attributes:

```csharp
[Table("SystemTenants")]
public class Tenant
{
    [Key]
    public Guid Id { get; set; }

    [Required]
    [MaxLength(128)]
    [Column(TypeName = "varchar")]
    public string Name { get; set; }
}
```

While this works for simple apps, **it is not recommended for enterprise applications like Normora.** 
1. **Polluted Domain Models:** We add too many attributes to our pure C# domain classes, cluttering them with database-specific concerns.
2. **Database Coupling:** If we switch database providers, some attributes might break as not all providers support the exact same attribute behaviors. 

### The Solution: The Fluent API (`OnModelCreating`)
The Fluent API allows us to define our data mappings and constraints inside the `DbContext` itself, keeping our domain classes clean. This is the **best place** to define mapping definitions.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; } // Clean, pure C# class!
}

public class TenantsDbContext : DbContext
{
    public DbSet<Tenant> Tenants { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Fluent API Configuration
        modelBuilder.Entity<Tenant>()
            .ToTable("SystemTenants")           
            .HasKey(t => t.Id);                 

        modelBuilder.Entity<Tenant>()
            .Property(t => t.Name)
            .HasColumnName("TenantName")        
            .HasColumnType("varchar")           
            .HasMaxLength(128)                  
            .IsRequired();                      
    }
}
```

### Why EF Core Defaults to the "Worst-Case" (e.g., `nvarchar(MAX)`)
When you don't explicitly configure data constraints (like max length), EF Core defaults to the "safest-case" scenario to prevent your application from crashing. 

1. **The "Safety First" Principle:** In C#, a `string` has virtually no length limit. However, relational databases require an exact maximum length. If EF Core blindly guessed `nvarchar(255)` for a `Title` column, the database would throw a **Data Truncation Exception** and crash the moment a user typed a 256-character title. To guarantee that *any* valid C# string can be saved without crashing, EF Core defaults to the absolute maximum size: `nvarchar(MAX)`.
2. **C# Types Lack Database Context:** To EF Core, `public string Title` and `public string Synopsis` look exactly the same. It has no idea that a title shouldn't exceed 100 characters.

While `nvarchar(MAX)` prevents crashes, it is the "worst-case" for database performance (slower indexing, higher storage overhead). This is precisely why we must use the Fluent API to override these safe defaults and tell EF Core our actual business constraints!

#### Other Types Affected by "Worst-Case" Defaults
This "safest/widest" behavior isn't just for strings. Here are other C# types where EF Core takes a heavily unoptimized approach by default:

1. **`byte[]` (Byte Arrays):** Has no fixed length in C#. 
   * **Default:** `varbinary(MAX)`. 
   * **Problem:** If you are storing a 32-byte hash, `MAX` is a massive waste of architecture. Use `.HasMaxLength(32)`.
2. **`DateTime`:** Highly precise in C#.
   * **Default:** `datetime2(7)` (Takes 8 bytes). 
   * **Problem:** If you only need a Birthday (Date), use `.HasColumnType("date")` (Takes 3 bytes).
3. **`decimal`:** 128-bit high-precision number.
   * **Default:** Usually `decimal(18, 2)` (18 total digits, 2 decimal places). 
   * **Problem:** If you are storing cryptocurrency, 2 decimal places will heavily truncate your data. If storing simple prices, `18` digits is overkill.
4. **`enum` (Enumerations):** 
   * **Default:** `int` (4 bytes).
   * **Problem:** If your enum only has 3 states, it easily fits in a `tinyint` (1 byte). Over millions of rows, the storage waste is significant.

### Common Fluent API Methods & Their Use Cases

When building enterprise applications like Normora, you will frequently use these Fluent API methods to define exact database constraints:

1. **`ToTable(string tableName)`**
   *   **Use Case:** By default, EF Core maps your entity to a table with the same name as the `DbSet` property (e.g., `Tenants`). However, in Normora, we might want our table to be named `SystemTenants` for administrative clarity. `ToTable()` overrides the default naming convention.
2. **`HasKey(Expression<Func<T, object>> keyExpression)`**
   *   **Use Case:** Defines the primary key of the table. While EF Core automatically assumes a property named `Id` or `TenantId` is the primary key, it is considered a best practice to explicitly define it using `HasKey(t => t.Id)` to prevent unexpected behavior during refactoring.
3. **`HasColumnName(string columnName)`**
   *   **Use Case:** Similar to `ToTable()`, this maps a C# property to a specific column name in the database. In our C# code, the property might just be `Name`, but in the database schema, it might need to be `TenantName`.
4. **`HasColumnType(string columnType)`**
   *   **Use Case:** Overrides the default SQL data type. For strings, EF Core might default to `nvarchar(max)`. By using `.HasColumnType("varchar")`, we enforce a non-unicode string type, which saves database storage if unicode characters (like emojis or special languages) are not needed for a Tenant's name.
5. **`HasMaxLength(int maxLength)`**
   *   **Use Case:** Restricts the maximum length of the data. Without this, a string property might map to an unbounded `varchar(max)` column, which is terrible for database indexing and performance. Setting `.HasMaxLength(128)` ensures the column is created as `varchar(128)`.
6. **`IsRequired()`**
   *   **Use Case:** Adds a `NOT NULL` constraint to the database column. In Normora, a Tenant absolutely must have a Name. If someone attempts to save a Tenant with a null name, EF Core will throw an exception before even hitting the database, and the database schema will enforce it as well.
7. **`HasData(params TEntity[] data)`**
   *   **Use Case:** Used for **Data Seeding**. It hardcodes `INSERT` (or `UPDATE`) statements directly into your database migrations to guarantee that critical initial data exists when the app boots.
   *   **Normora Example:** Seeding the default `SystemRoles` (Owner, Admin, Member) so the authorization system functions correctly on a brand new deployment.
   *   **Crucial Rule:** You **must** manually provide the Primary Key (e.g., `Id = 1`) inside the `new` object you pass to `HasData()`. EF Core relies on this hardcoded Primary Key to track future changes to the seeded data and generate correct migration updates.

---

## 4. Shadow Properties (Hidden Columns)

A **Shadow Property** is a column that exists in your database table, but **does not exist** as a property in your C# entity class. The value and state of a shadow property are maintained purely by the EF Core Change Tracker. Note that if we mention some property in configuration and that property does not exist in the model, EF Core will create that property as a shadow property.

### How to Configure a Shadow Property
Because the property doesn't exist in your C# class, you must configure it using the Fluent API by passing the data type and a string name:

```csharp
public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        // Tell EF Core to create a DateTime column named 'CreatedAt'
        // even though the 'Tenant' C# class has no such property!
        builder.Property<DateTime>("CreatedAt")
               .HasValueGenerator<CreatedDateGenerator>(); // (Optional) Auto-generates the value on insert
               
        // We can also create a hidden column for auditing who created this record!
        builder.Property<Guid>("CreatedByUserId");
    }
}
```

### The WHY (The Use Case)
The primary use case for Shadow Properties is **Auditing**. 
In enterprise systems like Normora, DBAs usually require columns like `CreatedAt`, `LastModifiedAt`, and `CreatedByUserId` on every single table for compliance and security. However, as a C# developer, you absolutely do not want these infrastructural database concerns polluting your pure Domain Models. By using shadow properties, you keep your C# classes perfectly clean while keeping the database schema fully compliant!

### How to Read and Write Shadow Properties
Since `tenant.CreatedAt` doesn't exist in C#, how do you interact with it? You have to ask the Change Tracker!

**Setting the value manually:**
```csharp
var newTenant = new Tenant { Name = "Acme Corp" };
context.Tenants.Add(newTenant);

// Tell the Change Tracker to set the shadow properties
context.Entry(newTenant).Property("CreatedAt").CurrentValue = DateTime.UtcNow;
context.Entry(newTenant).Property("CreatedByUserId").CurrentValue = currentUserId;

await context.SaveChangesAsync();
```
*(Note: You can also assign a `HasValueGenerator` in the Fluent API or use an intercepted `SaveChanges` (like we did for Soft Deletes) so EF Core sets these automatically!)*

**Querying with LINQ:**
To use it in a `Where` clause, you must use the special `EF.Property` static method:
```csharp
// Querying a shadow property
var olderTenants = await context.Tenants
    .Where(t => EF.Property<DateTime>(t, "CreatedAt") < new DateTime(2023, 1, 1))
    .ToListAsync();

// Updating a shadow property (Usually done inside a SaveChangesInterceptor)
context.Entry(existingTenant).Property("LastModifiedAt").CurrentValue = DateTime.UtcNow;
```

### Advanced: Applying Shadow Properties Globally
If you want *every single table* in your entire database to have auditing columns, you do not need to configure them one-by-one in 50 different `IEntityTypeConfiguration` classes! 

Instead, you can loop through your entities dynamically in `OnModelCreating`:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    // Loop through ALL entities registered in the DbContext
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        // Automatically inject these shadow properties into EVERY table!
        modelBuilder.Entity(entityType.ClrType).Property<DateTime>("CreatedAt");
        modelBuilder.Entity(entityType.ClrType).Property<Guid>("CreatedByUserId");
    }
}
```
If you combine this global configuration with the `SaveChanges()` interception trick (to auto-fill the dates when saving), you achieve a **100% automated enterprise auditing system**. Your C# domain models stay perfectly clean, and developers literally don't have to write a single line of auditing code ever again!

### When NOT to use Shadow Properties
**Never use shadow properties for core business data.** If your application logic or UI frequently needs to read, write, or display the data (e.g., `DateOfBirth`, `Price`, `IsActive`), it **must** be a real property in your C# class. 

Because shadow properties are hidden from C#:
1. They make LINQ queries clunky to write (`EF.Property()`).
2. You lose compiler safety (a typo in `"CreatedDate"` will crash at runtime).
3. They are completely invisible to the JSON serializer, meaning you can't easily return them from a Web API.

**Rule of Thumb:** Reserve shadow properties strictly for hidden, infrastructural data (like audit logs or internal foreign keys that C# never needs to touch).

---

## 5. Registering the DbContext (Dependency Injection Lifetimes)

When we register our `DbContext` in `Program.cs`, we must choose a DI lifetime. 

```csharp
// By default, AddDbContext registers the context as Scoped
builder.Services.AddDbContext<TenantsDbContext>(); 
```

### Why should DbContext be **Scoped**?
*   It provides a natural "Unit of Work" lifetime (typically one instance per HTTP Request).
*   It ensures the context is properly disposed of when the HTTP request/scope ends.
*   It allows EF Core to track changes to all entities loaded during that specific request safely.

### Why not **Singleton**?
*   **Not Thread-Safe:** `DbContext` is not thread-safe. Sharing one instance across concurrent HTTP requests will cause concurrency exceptions and problematic change-tracking behavior. It will also cause performance problems.

### Why not **Transient**?
*   If multiple services in a single HTTP request ask for the `DbContext`, Transient will create a brand new instance for each service. This means they won't share the same change tracker or database transaction, potentially complicating change tracking, unit of work, and transaction coordination.

### The `CreateScope` Hack & Database Creation Without Migrations
Sometimes, you need to use the `DbContext` outside of a normal HTTP request (like on startup). To do this safely, you must manually create a Scope to simulate an API call:

```csharp
// This simulates as if we made an API call
var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<TenantsDbContext>();

// This can be used to create DB without migration
context.Database.EnsureDeleted();
context.Database.EnsureCreated();
```

---

## 6. Real-World Scenarios

### Scenario A: Auditing Every Table with Shadow Properties (Normora)
In Normora, every table requires `CreatedAt`, `CreatedByUserId`, and `UpdatedAt` columns for SOC 2 compliance. Instead of adding these to every domain model (which pollutes them), we use shadow properties globally and auto-populate them by overriding `SaveChangesAsync`:

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var currentUserId = _httpContextAccessor.HttpContext?.User.GetUserId();

    foreach (var entry in ChangeTracker.Entries())
    {
        if (entry.State == EntityState.Added)
        {
            entry.Property("CreatedAt").CurrentValue = DateTime.UtcNow;
            entry.Property("CreatedByUserId").CurrentValue = currentUserId;
        }
        if (entry.State == EntityState.Modified)
        {
            entry.Property("UpdatedAt").CurrentValue = DateTime.UtcNow;
        }
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```
Zero developer effort — every row is automatically audited!

### Scenario B: Seeding Default Roles on First Deployment
```csharp
public class RoleConfiguration : IEntityTypeConfiguration<SystemRole>
{
    public void Configure(EntityTypeBuilder<SystemRole> builder)
    {
        builder.HasData(
            new SystemRole { Id = 1, Name = "Owner" },
            new SystemRole { Id = 2, Name = "Admin" },
            new SystemRole { Id = 3, Name = "Member" }
        );
    }
}
```
When `dotnet ef database update` is run on a fresh deployment, the migration automatically seeds these roles. The application's authorization system works correctly from day one without manual SQL scripts.

### Scenario C: Using `nvarchar` vs `varchar` Correctly
```csharp
// Tenant's name could contain international characters (Arabic, Chinese, etc.)
// Use nvarchar (unicode-safe) for user-facing strings
builder.Property(t => t.Name).HasColumnType("nvarchar").HasMaxLength(128);

// Tenant's subdomain is always ASCII (e.g., "acme-corp")
// Use varchar (non-unicode) for technical identifiers — saves 50% storage
builder.Property(t => t.Subdomain).HasColumnType("varchar").HasMaxLength(63);
```

---

## 7. Interview Questions

**Q1: What is the difference between Data Annotations and the Fluent API? Which should you prefer and why?**
> Data Annotations are C# attributes placed directly on entity classes (e.g., `[MaxLength(128)]`). The Fluent API is configured inside `OnModelCreating` or `IEntityTypeConfiguration` classes. The Fluent API is preferred in enterprise applications because: (1) it keeps domain models clean and free from database concerns, (2) it supports more advanced configurations that Data Annotations cannot express (complex indexes, shadow properties, alternate keys), and (3) it doesn't couple your domain layer to EF Core.

**Q2: What is a Shadow Property? Give a real-world use case.**
> A shadow property is a column that exists in the database but has no corresponding property in the C# entity class. It's managed purely by EF Core's Change Tracker. The most common use case is auditing — you can add `CreatedAt`, `UpdatedAt`, and `CreatedByUserId` columns to every table without polluting your domain models. You access shadow properties via `EF.Property<T>(entity, "PropertyName")` in queries.

**Q3: Why must `DbContext` be registered as Scoped and not Singleton?**
> `DbContext` is not thread-safe. If registered as Singleton, it would be shared across concurrent HTTP requests, causing race conditions, incorrect change tracking, and connection pool exhaustion. Scoped lifetime ensures one instance per HTTP request — matching the natural "unit of work" pattern — and it is properly disposed when the request ends.

**Q4: Why does EF Core default to `nvarchar(MAX)` for strings, and what are the performance implications?**
> EF Core defaults to `nvarchar(MAX)` because it has no way to know the maximum length of a C# `string`. Using `MAX` prevents data truncation exceptions but hurts performance: `MAX` columns cannot be indexed efficiently, consume more storage, and increase network transfer size. You should always use `.HasMaxLength()` in the Fluent API to set appropriate constraints.

**Q5: If you define `builder.Property<Guid>("TenantId")` in configuration but the entity class has no `TenantId` property, what happens?**
> EF Core creates it as a **shadow property**. The column will exist in the database and be managed by the Change Tracker, but it won't appear in the C# class. This is a valid pattern — for example, when using foreign keys that are purely infrastructural and don't belong in the domain model.

**Q6: What is `EnsureCreated()` vs running migrations? When would you use each?**
> `EnsureCreated()` creates the database schema based on the current model in a single shot — but it does not create a migrations history table and cannot handle schema evolution (updates). It's fine for unit tests or simple demos. Migrations (`dotnet ef migrations add`, `dotnet ef database update`) create a versioned, incremental history of schema changes and are the correct approach for any production application.

---

## Summary
In this module, we learned how to keep our domain models clean by using the Fluent API inside `OnModelCreating`, why `= null!` is used for `DbSet`, how shadow properties enable invisible auditing, and why `DbContext` must always be registered as a Scoped service. In the next module, we will dive into creating, reading, updating, and deleting data (CRUD)!
