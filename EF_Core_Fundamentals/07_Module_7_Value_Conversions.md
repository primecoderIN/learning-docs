# Module 7: Master Value Conversions

In this module, we will deeply explore **Value Conversions** in EF Core. We will break down exactly what they are, why they are used, when to use them, where they live, and how to implement them using real-world scenarios from Normora.

---

## 1. WHAT is Value Conversion?

Value Conversion is a feature in EF Core that allows you to translate (convert) data from one format into another *as it travels between your C# application and your database*. 

*   **When saving:** It converts your rich C# data type into a simpler database type.
*   **When reading:** It converts that simple database type back into your rich C# data type.

Think of it as a translator sitting directly between your `DbContext` and the SQL Server.

---

## 2. WHY do we need it? (The Use Case)

You need Value Conversions because **C# is much richer than SQL**. C# has complex types like `List<T>`, custom Enums, `TimeSpan`, `IPAddress`, and custom Value Objects. Relational databases only understand basic primitives like `INT`, `VARCHAR`, and `DATETIME`.

Without Value Conversions, if you try to save a `List<string>` to a SQL Server table, EF Core will throw an exception because it has no idea how to map a C# List to a SQL column.

**Use cases include:**
*   Storing a list or dictionary as a single JSON string.
*   Storing an `Enum` as a readable string (e.g., `"Active"`) instead of a meaningless integer (e.g., `1`).
*   Encrypting a string before it hits the database, and decrypting it when it comes out.
*   Using Domain-Driven Design (DDD) "Value Objects" (e.g., wrapping an email address in a strict `EmailAddress` C# class, but saving it just as a `varchar` in the DB).

---

## 3. WHEN should you use it?

*   **Use it when:** You want to keep your C# Domain Models completely pure and type-safe, but the database doesn't natively support that data type.
*   **Avoid it when:** You are just dealing with standard strings, integers, or dates. EF Core handles those perfectly by default.
*   **Warning:** You cannot easily use LINQ `.Where()` queries against properties that have complex custom conversions (like JSON serialization), because EF Core doesn't know how to translate C# JSON parsing into a SQL `WHERE` clause.

---

## 4. WHERE is it configured?

Value conversions are configured in the **Fluent API** (inside your `IEntityTypeConfiguration<T>` classes), using the `.HasConversion()` method.

---

## 5. HOW to use Built-In Converters

EF Core comes with dozens of built-in converters for the most common scenarios. You don't have to write any conversion logic yourself; you just tell EF Core which built-in converter to use.

### Normora Example: Storing Enums as Strings
Let's say Normora has a `Tenant` with a `Status`.

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

By default, if a tenant is Active, EF Core saves the number `1` in the database. But database administrators hate this! When they run a SQL query, they see `1` and have no idea what it means. They want to see the word `"Active"`.

**The Fix:** We use the built-in string converter!

```csharp
public class TenantConfiguration : IEntityTypeConfiguration<Tenant>
{
    public void Configure(EntityTypeBuilder<Tenant> builder)
    {
        builder.HasKey(t => t.Id);

        // Tell EF Core to convert the Enum to a string when saving, 
        // and back to an Enum when reading!
        builder.Property(t => t.Status)
               .HasConversion<string>(); 
    }
}
```
Now, your C# code still safely uses `TenantStatus.Active`, but the database stores the `VARCHAR` string `"Active"`.

### Normora Example 2: Storing DateTime as Ticks (`long`)
Sometimes you need absolute nanosecond precision for auditing (like `CreatedAt`), but your older database engine (like older SQL Server) rounds `DATETIME` columns to the nearest 3 milliseconds.

**The Fix:** Use the built-in `long` converter to store the `DateTime` as "Ticks" (a massive integer).

```csharp
builder.Property(t => t.CreatedAt)
       .HasConversion<long>(); // Built-in converter: C# DateTime <-> DB BIGINT (Ticks)
```

---

## 6. HOW to use Custom Converters

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

---

## 7. Global Value Conversions (Pre-Convention Configuration)

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

## 8. Architectural Bonus: Handling Multiple Time Zones

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

## Summary
Value Conversions are the ultimate tool for keeping your C# Domain Models clean and rich, without being artificially limited by what your database engine can natively store. By mastering `.HasConversion()`, you can seamlessly bridge the gap between complex Object-Oriented code and flat Relational tables.
