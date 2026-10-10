# Module 9: Value Generators in EF Core

In enterprise applications, we often need certain database columns to be populated automatically. The most common examples are generating a sequential `Id`, or setting a `CreatedAt` and `ModifiedAt` timestamp when a row is inserted or updated.

Entity Framework Core provides a feature called **Value Generators** to handle this requirement strictly on the C# application side, *before* the data ever hits the database.

---

## 1. What is a Value Generator?

A Value Generator is a C# class that inherits from `ValueGenerator<T>`. It allows you to inject custom logic to automatically generate a value for a property when an entity is added to the `DbContext`.

Instead of the database generating the value (like a SQL `IDENTITY` or a SQL `DEFAULT (GETUTCDATE())`), **EF Core generates the value in memory** and explicitly includes it in the `INSERT` or `UPDATE` statement.

---

## 2. The Architectural Alternatives

Before building a Value Generator, you must understand the three ways to generate default values and why a Value Generator might be the best choice.

### Option A: Constructor Initialization (The "Quick & Dirty" Way)
```csharp
public class User 
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime ModifiedAt { get; set; } = DateTime.UtcNow;
}
```
**Why it's bad (Wasted CPU & State Failures):** 
1. **Wasted CPU:** If you fetch 1,000 existing users (`db.Users.ToList()`), EF Core must instantiate 1,000 `User` objects in memory before mapping the database columns to them. The constructor fires 1,000 times, forcing the CPU to calculate the exact time (`DateTime.UtcNow`) 1,000 times, only for EF Core to instantly throw those values in the trash and overwrite them with the real dates from the database row.
2. **The `ModifiedAt` Failure:** Constructors only run when the object is *first created* in memory. If an Admin queries a user at 10:00 AM, changes their name at 10:05 AM, and hits `SaveChanges()`, the constructor does NOT run again. `ModifiedAt` stays stuck at 10:00 AM and never updates.

### Option B: Database Defaults (`HasDefaultValueSql`)
```csharp
builder.Property(u => u.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
```
**Why it's sometimes bad (The Round-Trip Problem):** 
*   A "round trip" is data traveling over the network from C# to SQL and back. When you use database defaults, C# sends an `INSERT` statement without a timestamp. SQL Server inserts the row and uses its own clock to assign the date.
*   Because C# didn't generate the time, your C# `user` object doesn't know its own creation time! To fix this, EF Core must instantly send a **second query** (`SELECT CreatedAt FROM Users WHERE Id = ...`) over the network to ask SQL Server what time it just used. This second network round-trip slows down high-throughput applications.
*   It is also database-provider specific (`GETUTCDATE()` breaks if you switch from SQL Server to PostgreSQL).

### Option C: The EF Core Value Generator (The Clean Architecture Way)
By using a Value Generator, you ensure the logic lives entirely in C#. EF Core generates the value *before* sending the `INSERT`, meaning:
1. No wasted CPU cycles in constructors.
2. No second database round-trip required to read the generated value.
3. 100% database agnostic (works on SQL Server, Postgres, SQLite identically).

---

## 3. Creating a Custom `DateTimeUtcGenerator`

Let's fulfill a real-world requirement: We want a `CreatedAt` property to automatically receive the exact UTC time when a new entity is inserted into the database.

We do this by creating a custom class that inherits from `ValueGenerator<DateTime>`.

```csharp
using Microsoft.EntityFrameworkCore.ChangeTracking;
using Microsoft.EntityFrameworkCore.ValueGeneration;
using System;

public class DateTimeUtcGenerator : ValueGenerator<DateTime>
{
    /// <summary>
    /// This method is called by EF Core when the entity is Added.
    /// It returns the generated value.
    /// </summary>
    public override DateTime Next(EntityEntry entry)
    {
        // Always generate a pure UTC DateTime
        return DateTime.UtcNow;
    }

    /// <summary>
    /// If true, EF Core treats this as a temporary value that the database will replace.
    /// If false, EF Core knows THIS is the final value and saves it to the database.
    /// Because WE are generating the final timestamp in C#, this must be false.
    /// </summary>
    public override bool GeneratesTemporaryValues => false;
}
```

---

## 4. Applying the Value Generator

Now that we have our `DateTimeUtcGenerator`, we need to tell EF Core to use it for our `CreatedAt` property. We do this in the Fluent API.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        builder.HasKey(t => t.Id);

        // Tell EF Core to use our custom generator when adding this entity!
        builder.Property(t => t.CreatedAt)
               .HasValueGenerator<DateTimeUtcGenerator>()
               .ValueGeneratedOnAdd(); // IMPORTANT: Tell EF Core this only happens on INSERT
    }
}
```

### What happens under the hood?
When you write this C# code:
```csharp
var tenant = new Tenant { Name = "Acme Corp" }; 
// Note: We did NOT set CreatedAt

dbContext.Tenants.Add(tenant); 
// EF Core intercepts the Add(). 
// It sees the ValueGenerator, calls Next(), and assigns DateTime.UtcNow to tenant.CreatedAt!

await dbContext.SaveChangesAsync();
```

EF Core generates this exact SQL:
```sql
-- EF Core explicitly inserts the value it generated! No database defaults are used.
INSERT INTO [Tenants] ([Id], [Name], [CreatedAt])
VALUES ('...', 'Acme Corp', '2023-10-27T14:32:00.0000000Z');
```

---

## 5. How Value Generators Interact with Seed Data (`HasData`)

A very common mistake is assuming that your `DateTimeUtcGenerator` will automatically fire when you seed data using `.HasData()`. **It will not.**

### Why they don't work during seeding
Value Generators only run **at runtime**, inside the EF Core Change Tracker, when you explicitly call `dbContext.Add(entity)`. 

When you seed data using `.HasData()`, you are bypassing the Change Tracker entirely. EF Core's model builder reads the seed data and hardcodes the `INSERT` statements directly into your Migration files (C# code). Because the model builder is just creating migrations—not running a live application—it never triggers your custom `ValueGenerator<T>`.

### The Fix: Be Explicit
If you use `.HasData()` and do not provide a `CreatedAt` value, EF Core will fall back to C#'s default for `DateTime` (`0001-01-01`), and your database will get seeded with data from the year 1 AD!

When seeding data, you **must explicitly provide the value** manually inside the seed block:

```csharp
builder.HasData(
    new Tenant 
    { 
        Id = Guid.Parse("..."), 
        Name = "Acme", 
        
        // You MUST explicitly provide the value during seeding!
        // Best Practice: Hardcode a specific date so migrations are deterministic
        CreatedAt = DateTime.Parse("2023-10-27T12:00:00Z") 
    }
);
```

---

## 6. Real-World Use Case: A Complete Auditing Setup

While `HasValueGenerator` is fantastic for `CreatedAt` (`ValueGeneratedOnAdd`), handling `ModifiedAt` (`ValueGeneratedOnUpdate`) via Value Generators is historically tricky because generators are primarily designed for identity generation.

For a complete enterprise auditing system (where you want `CreatedAt`, `ModifiedAt`, and `CreatedBy`), the industry standard is to use the **Change Tracker `SaveChanges` Interception** approach instead of a Value Generator, but knowing how Value Generators work is crucial for isolated properties.

### Bonus: The `GuidValueGenerator`
Did you know EF Core automatically applies a Value Generator to all `Guid` primary keys? 
Behind the scenes, EF Core automatically applies its internal `SequentialGuidValueGenerator` to your `Guid Id` properties. This generates sequential GUIDs in memory (which prevents database index fragmentation) before executing the `INSERT`!

---

## 7. Bonus: Value Converters vs Value Generators (Strongly-Typed IDs)

It is very common to confuse Value **Generators** with Value **Converters** (which we covered in Module 7). 

*   **Value Generator:** *Creates* a brand new value out of thin air (like `DateTime.UtcNow` or a new `Guid`).
*   **Value Converter:** *Translates* an existing C# object into a database primitive (e.g., converting a C# Enum into a SQL string).

### The Enterprise Pattern: Strongly-Typed IDs
In large multi-tenant applications like Normora, you pass `Guid`s everywhere. You have `TenantId`, `UserId`, `DocumentId`. Because they are all just `Guid`s, the C# compiler cannot protect you from catastrophic mistakes like this:

```csharp
public async Task<User> GetUser(Guid tenantId, Guid userId)
{
    // FATAL SECURITY FLAW! The developer accidentally swapped the parameters!
    // The compiler allows it because they are both just Guids.
    return await db.Users.SingleAsync(u => u.TenantId == userId && u.Id == tenantId); 
}
```

### The Fix: Domain-Driven Design
We fix this by creating custom strict types (usually `readonly record struct`) for our IDs, so the compiler catches the swap:

```csharp
public readonly record struct TenantId(Guid Value);
public readonly record struct UserId(Guid Value);

public class User
{
    // Now these are strongly typed!
    public UserId Id { get; set; }
    public TenantId TenantId { get; set; }
}
```

If a developer attempts to write `u.TenantId == userId` now, Visual Studio immediately throws a red compile error: *"Operator '==' cannot be applied to operands of type 'TenantId' and 'UserId'"*. **You have achieved 100% compile-time safety.**

### How EF Core Handles This (Value Converters)
EF Core doesn't know how to save a custom `TenantId` record into SQL Server. To fix this, we build a **Value Converter** (not a generator, because we aren't creating a new ID, we are just *translating* it).

```csharp
public class TenantIdConverter : ValueConverter<TenantId, Guid>
{
    public TenantIdConverter() : base(
        tenantIdObj => tenantIdObj.Value, // C# to DB: Unwrap the Guid
        dbGuid => new TenantId(dbGuid)    // DB to C#: Wrap it back up
    ) { }
}
```

We apply this globally using Bulk Configuration in the DbContext:
```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder builder)
{
    // Automatically use this converter for EVERY TenantId property in the app!
    builder.Properties<TenantId>().HaveConversion<TenantIdConverter>();
    builder.Properties<UserId>().HaveConversion<UserIdConverter>();
}
```

### The Final Result
Your method signature now demands the strict types, and your LINQ query can use them natively. When EF Core translates the LINQ into SQL, the Value Converter kicks in, unwraps the objects, and sends the raw GUIDs to SQL Server!

```csharp
public async Task<User> GetUser(TenantId tenantId, UserId userId)
{
    // 100% type-safe. The Value Converter translates this to standard SQL Guids behind the scenes!
    return await db.Users.SingleAsync(u => u.TenantId == tenantId && u.Id == userId);
}
```

---

## 8. Interview Questions

**Q1: What is the difference between a Value Generator and `HasDefaultValueSql`?**
> `HasDefaultValueSql` delegates the value generation to the database engine (e.g., using SQL Server's `GETUTCDATE()`). This means EF Core must perform a second query to read the generated value back into memory after insertion. A Value Generator executes C# code to generate the value in memory *before* the `INSERT` statement is sent, preventing the extra database round-trip and keeping the application database-agnostic.

**Q2: What is the purpose of the `GeneratesTemporaryValues` property in a custom Value Generator?**
> It tells EF Core whether the generated value is final or just a placeholder. If `true` (like a temporary negative ID used before inserting into a database with `IDENTITY` columns), EF Core will expect the database to replace it. If `false` (like our `DateTimeUtcGenerator`), EF Core knows the generated C# value is the final value to be saved to the database.

**Q3: Why shouldn't you just set `public DateTime CreatedAt { get; set; } = DateTime.UtcNow;` in the entity's constructor?**
> Setting default values in a constructor causes performance overhead and logic issues. Every time EF Core queries a row from the database, it must instantiate the object, which triggers the constructor to generate a brand new `UtcNow` time, only for EF Core to immediately overwrite it a millisecond later with the actual value from the database. Value generators avoid this because they only fire when an entity is explicitly added to the `DbContext`.

**Q4: Can a Value Generator depend on external services, like fetching the logged-in user's ID?**
> Technically yes, but it is highly discouraged. Value Generators are singletons created once per DbContext model. Injecting scoped services (like an `IHttpContextAccessor` to get the current user) into a singleton generator causes Dependency Injection scope issues. For complex contextual generation (like setting `ModifiedBy`), it is much better to override `SaveChangesAsync()` and inspect the Change Tracker.

---

## Summary
Value Generators provide a clean, database-agnostic way to automatically populate properties in memory before they are saved to the database. By building a custom `DateTimeUtcGenerator`, we can efficiently stamp `CreatedAt` properties without suffering the performance penalties of constructor initialization or the database round-trips caused by SQL defaults.
