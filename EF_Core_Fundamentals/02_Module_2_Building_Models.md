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

## 4. Registering the DbContext (Dependency Injection Lifetimes)

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

## Summary
In this module, we learned how to keep our domain models clean by using the Fluent API inside `OnModelCreating`, why `= null!` is used for `DbSet`, and why `DbContext` must always be registered as a Scoped service. In the next module, we will dive into creating, reading, updating, and deleting data (CRUD)!
