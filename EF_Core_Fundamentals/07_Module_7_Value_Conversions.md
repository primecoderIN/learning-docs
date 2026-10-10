# Module 7: The Change Tracker, Concurrency, and Value Conversions

In this final module, we will explore advanced EF Core concepts that are essential for building robust enterprise applications. We will dive deep into the **Change Tracker**, learn how to boost read performance with `AsNoTracking`, handle data conflicts with **Concurrency Tokens**, and bridge the gap between C# types and SQL types using **Value Conversions**.

---

## 1. The Change Tracker and Entity States

The **Change Tracker** is the heart of Entity Framework Core. When you query entities from the database, EF Core doesn't just return objects; it actively monitors them.

Every entity tracked by the `DbContext` has an `EntityState`. There are five possible states:

1.  **`Unchanged`:** The entity was queried from the database and has not been modified. (`SaveChanges` ignores it).
2.  **`Modified`:** The entity was queried, and one or more of its properties have been changed in C#. (`SaveChanges` generates an `UPDATE`).
3.  **`Added`:** The entity is new and was added to the context (e.g., via `context.Add()`). It does not exist in the database yet. (`SaveChanges` generates an `INSERT`).
4.  **`Deleted`:** The entity was marked for deletion (e.g., via `context.Remove()`). (`SaveChanges` generates a `DELETE`).
5.  **`Detached`:** The entity is not being tracked by the `DbContext`. This happens if you instantiate it manually without adding it, or if you use `AsNoTracking()`.

### How SaveChanges Works
When you call `await context.SaveChangesAsync()`, EF Core loops through the Change Tracker. It ignores `Unchanged` and `Detached` entities. For the rest, it generates the appropriate SQL statements, wraps them all in a **single database transaction**, and executes them. 

---

## 2. Boosting Read Performance: `AsNoTracking`

By default, every query you write tracks the returned entities. Tracking requires memory and CPU. The Change Tracker has to store a snapshot of the original values so it can compare them later to see what changed.

If you are querying data **only to display it** (e.g., returning it from a Web API, generating a report) and you have no intention of updating it, tracking is a massive waste of resources.

### The Fix: `.AsNoTracking()`
You can explicitly tell EF Core to skip the Change Tracker entirely using `.AsNoTracking()`.

```csharp
var activeTenants = await dbContext.Tenants
    .AsNoTracking() // EF Core will NOT track these objects!
    .Where(t => t.IsActive)
    .ToListAsync();
```

**Benefits:**
*   Uses significantly less RAM.
*   Executes much faster.
*   Prevents accidental database updates if a developer modifies the object later in the code.

**Important Rule for Projections:** If you use a `.Select(dto => new Dto { ... })` projection, EF Core **automatically** behaves as no-tracking because it knows it's creating a custom shape, not a tracked entity. You do not need to explicitly call `.AsNoTracking()` when using `.Select()`.

---

## 3. Handling Data Conflicts: Optimistic Concurrency

In a web application, two users might try to edit the same record at the same time. 

1. Admin A fetches `Tenant 1`.
2. Admin B fetches `Tenant 1`.
3. Admin A renames it to "Acme Corp" and saves.
4. Admin B renames it to "Acme Inc" and saves.

By default, "last one wins." Admin B's save completely overwrites Admin A's changes without warning. Admin A's work is silently lost.

### The Fix: Concurrency Tokens (Optimistic Concurrency)
We can tell EF Core to reject the second save if the data was modified behind the scenes. We do this by adding a **Concurrency Token**.

The best way to do this in SQL Server is to use a `RowVersion` (a byte array that the database automatically changes every time the row is updated).

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    
    // The Concurrency Token
    public byte[] RowVersion { get; set; }
}
```

**Fluent API Configuration:**
```csharp
builder.Property(t => t.RowVersion)
       .IsRowVersion(); // Tells EF Core: This is a concurrency token managed by SQL Server
```

**How it works during `SaveChanges`:**
When EF Core generates the `UPDATE` statement, it adds a `WHERE` clause checking the RowVersion!

```sql
UPDATE [Tenants] SET [Name] = 'Acme Inc' 
WHERE [Id] = 1 AND [RowVersion] = 0x00000000000007D1;
```

Because Admin A already updated the row, the database changed the `RowVersion` to `0x...7D2`. Admin B's `UPDATE` statement will affect **0 rows**. 

EF Core detects this and throws a `DbUpdateConcurrencyException`. You can catch this exception in your API and tell Admin B: *"This record was modified by someone else while you were editing it. Please refresh and try again."*

---

## 4. WHAT is Value Conversion?

Value Conversion is a feature in EF Core that allows you to translate (convert) data from one format into another *as it travels between your C# application and your database*. 

*   **When saving:** It converts your rich C# data type into a simpler database type.
*   **When reading:** It converts that simple database type back into your rich C# data type.

Think of it as a translator sitting directly between your `DbContext` and the SQL Server.

---

## 5. WHY do we need it? (The Use Case)

You need Value Conversions because **C# is much richer than SQL**. C# has complex types like `List<T>`, custom Enums, `TimeSpan`, `IPAddress`, and custom Value Objects. Relational databases only understand basic primitives like `INT`, `VARCHAR`, and `DATETIME`.

Without Value Conversions, if you try to save a `List<string>` to a SQL Server table, EF Core will throw an exception because it has no idea how to map a C# List to a SQL column.

**Use cases include:**
*   Storing a list or dictionary as a single JSON string.
*   Storing an `Enum` as a readable string (e.g., `"Active"`) instead of a meaningless integer (e.g., `1`).
*   Encrypting a string before it hits the database, and decrypting it when it comes out.
*   Using Domain-Driven Design (DDD) "Value Objects" (e.g., wrapping an email address in a strict `EmailAddress` C# class, but saving it just as a `varchar` in the DB).

---

## 6. WHEN should you use it?

*   **Use it when:** You want to keep your C# Domain Models completely pure and type-safe, but the database doesn't natively support that data type.
*   **Avoid it when:** You are just dealing with standard strings, integers, or dates. EF Core handles those perfectly by default.
*   **Warning:** You cannot easily use LINQ `.Where()` queries against properties that have complex custom conversions (like JSON serialization), because EF Core doesn't know how to translate C# JSON parsing into a SQL `WHERE` clause.

---

## 7. WHERE is it configured?

Value conversions are configured in the **Fluent API** (inside your `IEntityTypeConfiguration<T>` classes), using the `.HasConversion()` method.

---

## 8. HOW to use Built-In Converters

EF Core comes with dozens of built-in converters for the most common scenarios. You don't have to write any conversion logic yourself; you just tell EF Core which built-in converter to use.

### Normora Example 1: The Deep Dive on Enums as Strings

In C#, Enums are incredibly useful for strong typing. By default, EF Core saves Enums to the database as their underlying integer value (e.g., `0`, `1`, `2`). 

Let's say Normora has a `TenantStatus` enum:
```csharp
public enum TenantStatus 
{
    Pending = 0,
    Active = 1,
    Suspended = 2
}

public class Tenant
{
    public Guid Id { get; set; }
    public TenantStatus Status { get; set; }
}
```

**The WHY (The Use Case):**
If you let EF Core save this as an `INT`, your database administrators, data analysts, or reporting tools (like PowerBI) will just see `1` in the database. They will have absolutely no idea what `1` means without digging through the C# source code. Storing it as a string makes the database **human-readable** and self-documenting.

**The HOW:**
You use the built-in generic `<string>` converter in the Fluent API.

```csharp
public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        // Converts the Enum to a string when saving, and back to an Enum when reading
        builder.Property(t => t.Status)
               .HasConversion<string>()
               .HasMaxLength(20); // BEST PRACTICE: Always set a max length for string enums!
    }
}
```

**How Querying Works (The Magic):**
You might wonder: *If I query this in LINQ using `t.Status == TenantStatus.Active` (which is technically the number 1 in C#), will it break because the database is holding strings?* 

No! EF Core is incredibly smart. When it translates your LINQ into SQL, it passes your parameter through the value converter first. 
```csharp
// Your C# Code:
var activeTenants = db.Tenants.Where(t => t.Status == TenantStatus.Active).ToList();

// What EF Core sends to SQL Server:
// SELECT * FROM Tenants WHERE Status = 'Active'
```
It seamlessly handles comparing the string in the database instead of the number!

**The WHEN:**
*   **Use it when:** You have a small set of enum values, and humans or external reporting tools frequently query the database directly.
*   **Don't use it when:** You are building an ultra-high-performance system (like a stock trading engine) where saving 4 bytes per row vs 20 bytes per row actually matters for billions of records.

**The PITFALLS (Danger!):**
1.  **Refactoring Breaks Data:** If a C# developer renames the enum value from `Active` to `Live` in the code, the application will suddenly crash when trying to read existing `"Active"` records from the database, because `"Active"` can no longer be mapped back to a valid C# enum value.
2.  **Storage Space:** A SQL `INT` takes exactly 4 bytes. The string `"Suspended"` takes 18 bytes (as `NVARCHAR`). Over 10 million rows, this drastically increases database size and memory usage.
3.  **Indexing Performance:** Searching and sorting integer columns is significantly faster for the database engine than searching and sorting string columns.

### Normora Example 2: Storing DateTime as Ticks (`long`)
Sometimes you need absolute nanosecond precision for auditing (like `CreatedAt`), but your older database engine (like older SQL Server) rounds `DATETIME` columns to the nearest 3 milliseconds.

**The Fix:** Use the built-in `long` converter to store the `DateTime` as "Ticks" (a massive integer).

```csharp
builder.Property(t => t.CreatedAt)
       .HasConversion<long>(); // Built-in converter: C# DateTime <-> DB BIGINT (Ticks)
```

---

## 9. HOW to use Custom Converters

When EF Core's built-in converters aren't enough, you must write your own logic. You do this by passing two lambda expressions to `.HasConversion()`:
1.  **Expression 1:** How to convert C# ➔ Database
2.  **Expression 2:** How to convert Database ➔ C#

### Normora Example 1: Lists to JSON Strings
Imagine we want to store a list of allowed IP addresses for a `Tenant` to restrict login access.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    
    // SQL Server doesn't support List<string>!
    public List<string> AllowedIpAddresses { get; set; } = new List<string>();
}
```

We can create a custom converter that serializes the list into a JSON string before saving it to a `NVARCHAR(MAX)` column, and deserializes it back into a C# list when reading it!

```csharp
public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        builder.Property(t => t.AllowedIpAddresses)
            .HasConversion(
                // 1. C# to Database (Serialize to JSON String)
                csharpList => JsonSerializer.Serialize(csharpList, (JsonSerializerOptions)null),
                
                // 2. Database to C# (Deserialize back to List)
                dbString => JsonSerializer.Deserialize<List<string>>(dbString, (JsonSerializerOptions)null) ?? new List<string>()
            );
    }
}
```

### Normora Example 2: Encrypting Data
Imagine Normora stores a third-party API Key for a tenant (like a Stripe Secret Key). We want this to be a normal string in C#, but encrypted in the database.

```csharp
builder.Property(t => t.StripeSecretKey)
    .HasConversion(
        // 1. C# to DB (Encrypt)
        csharpString => EncryptionService.Encrypt(csharpString),
        
        // 2. DB to C# (Decrypt)
        dbString => EncryptionService.Decrypt(dbString)
    );
```

### Normora Example 3: The "Force UTC" Date Converter
In many relational databases, standard `datetime` columns do not store the "Timezone Kind" (UTC vs Local). When you save `DateTime.UtcNow`, it correctly goes into the database. However, when EF Core reads it back, the `Kind` property is lost and defaults to `Unspecified`. If your app then tries to call `.ToLocalTime()`, it breaks because C# doesn't know the base time was UTC.

We can fix this universally across Normora with a custom converter that forces the `Kind` back to UTC upon reading!

```csharp
builder.Property(t => t.CreatedAt)
    .HasConversion(
        // 1. C# to DB: Ensure it is strictly UTC before saving
        csharpDate => csharpDate.ToUniversalTime(),
        
        // 2. DB to C#: The DB gives us a raw DateTime. We explicitly tag it as UTC!
        dbDate => DateTime.SpecifyKind(dbDate, DateTimeKind.Utc)
    );
```

### Normora Example 4: Reusable Custom Converter Classes (Legacy Database Integration)
While passing inline lambdas (like we did above) is great for one-off scenarios, if you want to reuse a converter across many properties, it is best practice to create a dedicated class that inherits from `ValueConverter<TModel, TProvider>`.

**The Use Case:** Imagine Normora needs to integrate with a legacy mainframe database. This 40-year-old database doesn't have a `DATETIME` column; it just stores dates as an 8-character string like `"20231004"` (`yyyyMMdd`). 

We want our modern C# code to use standard `DateTime` objects, but we need EF Core to translate it to that specific 8-character string for the legacy database.

```csharp
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;
using System.Globalization;

// TModel = DateTime (What C# uses)
// TProvider = string (What the Database uses)
public class DateTimeToChar8Converter : ValueConverter<DateTime, string>
{
    public DateTimeToChar8Converter() : base(
        // 1. C# to DB: Convert DateTime to "20231004"
        dateTime => dateTime.ToString("yyyyMMdd", CultureInfo.InvariantCulture),
        
        // 2. DB to C#: Parse "20231004" back into a DateTime object
        stringValue => DateTime.ParseExact(stringValue, "yyyyMMdd", CultureInfo.InvariantCulture)
    )
    {
    }
}
```

**How to use it:**
You simply pass the class type into `.HasConversion()`:
```csharp
builder.Property(t => t.LegacyCreatedAt)
       .HasConversion<DateTimeToChar8Converter>();
```
*(Note: This is also the exact class type you would pass into the Global Bulk Configuration we discuss in the next section!)*

---

## 10. Global Value Conversions (Pre-Convention Configuration)

What if you want to apply a converter (like the "Force UTC" date converter) to **every single `DateTime` property** across your entire database? Writing `.HasConversion()` 50 times in 50 different configuration classes is a maintenance nightmare.

Starting in EF Core 6.0, Microsoft introduced **Bulk Configuration** (also known as pre-convention configuration). You can override `ConfigureConventions` in your `DbContext` to apply a converter globally to a specific C# type!

```csharp
public class TenantsDbContext : DbContext
{
    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        base.ConfigureConventions(configurationBuilder);

        // Any property in the entire app of type `DateTime` will use this built-in converter!
        configurationBuilder.Properties<DateTime>()
                            .HaveConversion<long>(); // Converts ALL dates to Ticks globally!

        // Or, you can pass a custom ValueConverter class for your UTC logic!
        configurationBuilder.Properties<DateTime>()
                            .HaveConversion<MyCustomUtcConverter>(); 
    }
}
```

**Why this is powerful:** It guarantees absolute consistency across your entire application. If a developer adds a new `TenantActivityLog` table with a `CreatedAt` property, they don't even have to remember to configure the timezone or tick conversion. EF Core applies it automatically during the initial model building phase!

---

## 11. Architectural Bonus: Handling Multiple Time Zones

If you are building an enterprise application (like Normora) that supports users across different time zones, you don't just need a converter—you need a strict architectural strategy. There are two enterprise-standard ways to handle this:

### Strategy 1: The "Strict UTC + User Settings" Approach (Most Common)
This is the standard approach used by almost every major SaaS application. 

**The Rule:** The database and backend **only ever speak UTC**. 

1. **The Database:** You store all dates as standard `DateTime` (using SQL Server's `datetime2`).
2. **The Converter:** You use the exact **"Force UTC" Custom Converter** globally (as shown in Section 7). This guarantees that if a developer accidentally tries to save a local time, it is forced into UTC before hitting the database, and is explicitly tagged as UTC when read back.
3. **The User Table:** You add a `TimezoneId` string property to your `User` table (e.g., `"Asia/Kolkata"` or `"America/New_York"`).
4. **The UI/API:** Your API sends the strict UTC date to the frontend (Angular/React). The frontend then reads the logged-in user's `TimezoneId` and converts the UTC date into their local time for display.

### Strategy 2: Using `DateTimeOffset` (The "Zero Converter" Approach)
Instead of using `DateTime` in your C# models, you change all your properties to `DateTimeOffset`.

Unlike `DateTime`, `DateTimeOffset` stores **two** pieces of information:
1. The exact UTC point in time.
2. The specific timezone offset (e.g., `+05:30`) of the user who performed the action.

```csharp
public class TenantActivityLog
{
    public Guid Id { get; set; }
    public string Action { get; set; }
    
    // Stores exactly when it happened AND what timezone the user was in!
    public DateTimeOffset OccurredAt { get; set; } 
}
```

**Do you need a converter for this?**
*   **Modern Databases (SQL Server / PostgreSQL):** No converter is needed! Modern SQL Server has a native `datetimeoffset` column type that perfectly maps to C#'s `DateTimeOffset`.
*   **Legacy Databases (SQLite):** Yes. SQLite doesn't natively support `DateTimeOffset`. You would use a built-in converter to translate the `DateTimeOffset` into a `long` (Ticks) or a `string` before saving it to the DB.

**Rule of Thumb:** If you want to know exactly what time it was *on the user's local clock* when they did something, use **Strategy 2 (`DateTimeOffset`)**. If you just want to record exactly when something happened globally and convert it later for display based on user preferences, use **Strategy 1 (Strict UTC Converter)**.

---

## 12. Interview Questions

**Q1: What are the states an entity can be in the EF Core Change Tracker?**
> The five states are `Added`, `Unchanged`, `Modified`, `Deleted`, and `Detached`. EF Core uses these states to determine whether to generate an `INSERT`, ignore the entity, generate an `UPDATE`, or generate a `DELETE` when `SaveChangesAsync()` is called.

**Q2: When and why should you use `.AsNoTracking()`?**
> You should use `.AsNoTracking()` on read-only queries where you have no intention of updating the returned entities (e.g., returning data for a GET request). It bypasses the Change Tracker, meaning EF Core doesn't allocate memory to store original snapshots or spend CPU cycles setting up tracking. This significantly improves performance and reduces memory overhead.

**Q3: How do you handle concurrent updates in EF Core to prevent data loss?**
> You implement Optimistic Concurrency by adding a Concurrency Token (like a `[Timestamp]` or `RowVersion` column) to the entity. When EF Core updates the record, it checks if the token in the database matches the token from when the entity was originally queried. If another user modified the row, the tokens won't match, the update will fail, and EF Core will throw a `DbUpdateConcurrencyException`, allowing you to handle the conflict gracefully.

**Q4: What is a Value Converter in EF Core? Give an example.**
> A Value Converter acts as a translator between complex C# types and simple database types. It tells EF Core how to serialize data when saving and deserialize it when reading. A common example is storing a C# `Enum` as its string representation (`"Active"`) in a `VARCHAR` column instead of its default integer value (`1`), or storing a `List<string>` as a JSON-serialized string in the database.

**Q5: Can you query against properties that use complex Value Conversions using LINQ?**
> Usually, no. If you use a custom Value Converter (like serializing a List to JSON), EF Core cannot translate a C# LINQ `.Where(x => x.AllowedIps.Contains("1.1.1.1"))` into SQL because it doesn't understand your custom serialization logic. However, for simple, built-in converters (like Enum to String), EF Core handles the translation perfectly, allowing full LINQ support.

**Q6: What is the difference between `Add` and `Attach`?**
> `context.Add(entity)` sets the entity's state to `Added`, meaning EF Core will generate an `INSERT` statement for it. `context.Attach(entity)` tells EF Core to start tracking the entity but sets its state to `Unchanged`, meaning it assumes the entity already exists in the database and hasn't been modified. `Attach` is useful in disconnected scenarios when you want to track an object without immediately marking it as new or modified.

---

## Summary
In this final module, we mastered the EF Core Change Tracker, learned how to optimize reads with `AsNoTracking`, protected data integrity with concurrency tokens, and bridged the gap between C# models and relational schemas using Value Conversions. Value Conversions are the ultimate tool for keeping your C# Domain Models clean and rich, without being artificially limited by what your database engine can natively store. You now have the foundational knowledge required to build high-performance, enterprise-grade data access layers with Entity Framework Core!
