# Module 6: Querying, Deferred Execution, and Projections

When querying data in EF Core, the key to high performance is understanding **where the execution happens**: does filtering and projection happen in the **database**, or does it happen after the data has already been loaded into your application's memory?

Let's explore the critical difference using our Normora `Tenant` and `User` entities.

---

## 1. Deferred Execution: `IQueryable` vs `IEnumerable` vs `List`

You can achieve deferred execution without using `IQueryable<T>`. However, it depends heavily on *what* you are executing and *when* you want the execution to happen.

Let's understand this using the `Tenant` entity in Normora.

### A. Using `IQueryable<T>` — Deferred execution in SQL

```csharp
IQueryable<Tenant> tenants = db.Tenants;

var filteredTenants = tenants
    .Where(t => t.CreatedYear == 2024);
```

At this point:
* EF Core has an expression describing the query.
* The SQL has not necessarily been generated yet.
* **The database has not been queried yet.**

Now execute it:

```csharp
var result = await filteredTenants.ToListAsync();
```

At this point:
1. EF Core translates the expression into SQL.
2. EF Core sends the SQL to the database.
3. The database executes the query.
4. The matching Tenants are returned and materialized into a `List<Tenant>`.

### Remember the sequence
`IQueryable` → Build query → `ToListAsync()` → Translate to SQL → Execute against database → Return results.

*Important:* `ToListAsync()` is one way to execute a query, but it's not the only one. Methods such as `FirstOrDefaultAsync()`, `SingleOrDefaultAsync()`, and `CountAsync()` also trigger the exact same execution sequence against the database!

### B. Using `IEnumerable<T>` — Deferred execution in Memory

You can also use `IEnumerable<T>`:

```csharp
IEnumerable<Tenant> tenants = db.Tenants
    .AsEnumerable()
    .Where(t => t.CreatedYear == 2024);

// The filtering operation is deferred.

var result = tenants.ToList();
```

**Here is what happens:**
1. `AsEnumerable()` switches subsequent LINQ operations to LINQ-to-Objects.
2. The database query (`SELECT * FROM Tenants`) executes when enumeration begins (at `.ToList()`).
3. **The `Where()` filtering happens in memory, not in SQL.**

*Important:* `AsEnumerable()` itself does not immediately execute the query. However, when enumeration starts, EF Core retrieves the database results, and the subsequent filtering happens in memory. For a large table, this retrieves massive amounts of unnecessary records!

### C. Using `List<T>` — Immediate Database Execution

```csharp
List<Tenant> tenants = await db.Tenants.ToListAsync();

var result = tenants
    .Where(t => t.CreatedYear == 2024)
    .ToList();
```

**Here:**
* `ToListAsync()` immediately executes the database query.
* All retrieved tenants are loaded into memory.
* `Where()` executes when the in-memory collection is enumerated.

So, **the filtering is deferred, but the database query is not.**

### D. Comparison Matrix

| Feature | `IQueryable<T>` | `IEnumerable<T>` | `List<T>` |
| :--- | :--- | :--- | :--- |
| **Supports deferred execution?** | Yes | Yes, for deferred LINQ operators | No, for the initial database retrieval |
| **Builds expression trees?** | Yes | No, subsequent LINQ operations use delegates | No |
| **Filtering location** | Usually database (via SQL) | In memory after switching to LINQ-to-Objects | In memory |
| **SQL execution happens...** | When enumerated (e.g. `ToListAsync`) | When enumerated, if backed by an unexecuted EF query | Already executed when populated using `ToListAsync()` |
| **Can translate LINQ to SQL?** | Yes | Not for LINQ operations performed *after* switching to LINQ-to-Objects | No |

### The Key Takeaway
**Deferred execution and `IQueryable<T>` are not the same thing.**
* `IQueryable<T>` allows EF Core to build a query expression and translate it into SQL when executed.
* `IEnumerable<T>` supports deferred in-memory iteration and can also wrap an unexecuted EF Core query.
* `List<T>` represents an already-materialized collection, although further LINQ operations on it can still be deferred.

> **Rule:** For database filtering, always prefer `IQueryable<T>` so EF Core can translate the filtering into SQL and avoid retrieving unnecessary records.

---

## 2. DB Projection (The Right Way) — `IQueryable` + `Select`

When you chain a `.Select()` onto an `IQueryable` **before** executing the query, EF Core translates that projection directly into SQL.

```csharp
var users = await db.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name,
        Email = u.Email
    })
    .ToListAsync();
```

EF Core translates this projection into highly optimized SQL:

```sql
SELECT [u].[Id], [u].[Name], [u].[Email]
FROM [SystemUsers] AS [u];
```

Notice that only the required columns travel from the Database → Application!

**The Flow:**
```text
Database
   ↓
SELECT Id, Name, Email
   ↓
UserDto
   ↓
Application memory
```

This is generally what you want for large datasets, as it dramatically reduces network traffic and memory usage.

---

## 3. In-Memory Projection (The Dangerous Way) — `ToList()` first

If you call an execution method (like `.ToListAsync()`) **before** your projection, you pull the entire table into memory first.

```csharp
// 🚨 DANGER: This pulls ALL columns into memory!
var users = await db.Users
    .ToListAsync();

// This projection is now happening in .NET using LINQ to Objects
var result = users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name,
        Email = u.Email
    })
    .ToList();
```

Here, EF Core executes a massive query:

```sql
SELECT *
FROM [SystemUsers];
```

**The Flow:**
```text
Database
   ↓
SELECT *
   ↓
Entities loaded into memory
   ↓
LINQ Select()
   ↓
UserDto
```

For a table with 1 million rows, or a table containing massive columns (like `byte[]` documents), this can cause severe performance issues or Out-Of-Memory exceptions.

---

## 4. The Important Distinction

| Feature | DB Projection | In-memory Projection |
| :--- | :--- | :--- |
| **Code Pattern** | `.Select(...).ToListAsync()` | `.ToListAsync().Select(...)` |
| **Where it happens** | Database | Application (.NET) |
| **Data fetched** | Only selected columns | Usually all entity columns (`SELECT *`) |
| **Memory usage** | Lower | Higher |
| **Network transfer** | Lower | Higher |
| **Can leverage SQL** | Yes | No |
| **Good for large datasets** | ✅ Yes | ❌ No |
| **Good for complex C# logic** | Sometimes impossible | ✅ Yes |

---

## 5. A Common Mistake

Developers often mistakenly execute a query and then try to return a projection immediately after.

```csharp
// MISTAKE!
var result = await db.Users.ToListAsync();

return result.Select(x => new UserDto 
{ 
    Id = x.Id, 
    Name = x.Name 
});
```

You have already lost the main benefit of projection because the database query has already executed (`SELECT *`). 

Instead, always swap the order:

```csharp
// CORRECT!
return await db.Users
    .Select(x => new UserDto 
    { 
        Id = x.Id, 
        Name = x.Name 
    })
    .ToListAsync();
```

---

## 6. One More Important Point: Translatable Expressions

EF Core can translate **only expressions it understands** into SQL.

For example, simple string concatenation is understood by EF Core and translated to SQL `+` or `CONCAT`:

```csharp
var result = await db.Users
    .Select(x => new UserDto
    {
        Name = x.FirstName + " " + x.LastName
    })
    .ToListAsync();
```

But arbitrary, complex C# logic is **not** translatable:

```csharp
// EF Core will throw an exception here! It cannot translate 'MyComplexCSharpMethod' to SQL.
var result = await db.Users
    .Select(x => new UserDto
    {
        Name = MyComplexCSharpMethod(x.FirstName)
    })
    .ToListAsync();
```

If you absolutely must use complex C# logic, you have to fetch the required data first (perhaps an intermediate projection of just the necessary columns), and then perform that complex part in-memory.

---

## 7. The Golden Rule of Thumb

Always follow this chain of events:

> **Filter → Project → Execute (on the database whenever possible)**

```csharp
var activeUsers = await db.Users
    .Where(x => x.IsActive)        // 1. Filter
    .Select(x => new UserDto       // 2. Project
    {
        Id = x.Id,
        Name = x.Name
    })
    .ToListAsync();                // 3. Execute
```

This is one of the most fundamentally important performance patterns in Entity Framework Core.

---

## 8. Loading Related Data: Eager Loading (`.Include()`)

By default, when you query an entity in EF Core, **it does not load related data**. 

If you query a `User` from the database, its navigation properties (like `user.Tenant`) will be `null`. If you try to access `user.Tenant.Name`, your application will crash with a `NullReferenceException`.

To fix this, you use **Eager Loading** via the `.Include()` method.

### The Syntax (Normora Example)

```csharp
var user = await _context.Users
    .Include(u => u.Tenant) // Eagerly load the related Tenant data!
    .SingleOrDefaultAsync(u => u.Id == id);

Console.WriteLine(user.Tenant.Name); // This now works perfectly!
```

### What happens in the Database?
When you use `.Include()`, EF Core translates it into a SQL `JOIN`. It fetches both the User and the Tenant in a **single round-trip** to the database:

```sql
SELECT [u].[Id], [u].[Name], [u].[TenantId], [t].[Id], [t].[Name]
FROM [SystemUsers] AS [u]
LEFT JOIN [SystemTenants] AS [t] ON [u].[TenantId] = [t].[Id]
WHERE [u].[Id] = @id
```

### Loading Multiple Levels (`ThenInclude`)
If you need to go deeper—for example, loading a `User`, their `Tenant`, and then the `TenantBranding` profile for that tenant—you use `.ThenInclude()`:

```csharp
var user = await _context.Users
    .Include(u => u.Tenant)           // 1. Load the Tenant
        .ThenInclude(t => t.Branding) // 2. Load the Branding FOR that Tenant
    .SingleOrDefaultAsync(u => u.Id == id);
```

### Why is this important?
Eager loading is crucial for performance. If you forget to use `.Include()` but still need the related data, you might be tempted to loop through records and query the database one-by-one (known as the dreaded **N+1 Query Problem**). Eager loading solves this by bringing everything back in one highly-optimized query.

### The Danger of Eager Loading: Object Cycle Errors (JSON)
If you heavily use `.Include()` on bidirectional relationships and try to return that entity directly from a Web API controller, your application will crash with a `JsonException: A possible object cycle was detected`.

**Why?**
1. EF Core successfully loads the `User` and their `Tenant` using the JOIN.
2. The Web API's JSON Serializer starts building the HTTP response.
3. It serializes the `User`, sees the `Tenant` property, and dives into it.
4. Inside the `Tenant`, it sees the `Users` collection, dives into it, finds the original `User`, dives into the `Tenant`... and loops infinitely until the server crashes.

**How to fix it:**
1. **(Best Practice):** Use DTOs and Projections (`.Select()`) so you never return raw EF Core entities directly to the client. This gives you ultimate control over exactly what JSON is created without permanently locking your database models.
2. **(Model Fix - `[JsonIgnore]`):** Add `[JsonIgnore]` to the "child" side of the navigation property (as covered in Module 5). 
   * **How it breaks the loop:** When the serializer dives into the `Tenant` and reads the `User` again, it sees `[JsonIgnore]` on `user.Tenant`, stops serializing that path immediately, and safely cuts the infinite cycle.
   * **The Catch:** If you *ever* actually wanted to return a `User` from a specific API endpoint and include their `Tenant` data in the JSON response, you can't. `[JsonIgnore]` permanently blocks that property from ever being serialized anywhere in your entire app.
3. **(Global API Fix):** Tell your API's JSON Serializer to gracefully ignore cyclical references in `Program.cs`:
   ```csharp
   builder.Services.AddControllers().AddJsonOptions(options =>
   {
       options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
   });
   ```

---

## 9. Global Query Filters (`.HasQueryFilter`)

A **Global Query Filter** is a LINQ query predicate that is applied automatically to **every single query** executed against a specific entity type. You configure it once in the Fluent API, and EF Core silently injects it into every `SELECT`, `UPDATE`, and `DELETE` operation.

### The Two Ultimate Use Cases

#### 1. Soft Delete
Instead of hard-deleting records (which destroys historical data and breaks foreign keys), many enterprise applications use "Soft Delete" by adding an `IsDeleted` boolean column.

However, you don't want developers to have to remember to write `.Where(x => !x.IsDeleted)` on every single query they ever write.

```csharp
builder.Entity<User>()
       .HasQueryFilter(u => !u.IsDeleted); 
```
Now, `dbContext.Users.ToList()` will *magically* only return active users. EF Core handles it invisibly!

#### 2. Multi-Tenancy (The Normora Example)
Normora is a multi-tenant application. When a user from "Tenant A" logs in, they must **never** be able to see data from "Tenant B". Relying on developers to remember to write `.Where(u => u.TenantId == currentTenantId)` on every query is a massive security risk. If they forget even once, you have a critical data leak.

**The Fix:** Inject the current `TenantId` into the `DbContext`, and apply a Global Query Filter!

```csharp
public class TenantsDbContext : DbContext
{
    private readonly Guid _currentTenantId;

    // Inject a service that knows who the current logged-in user is
    public TenantsDbContext(DbContextOptions options, ITenantService tenantService) : base(options)
    {
        _currentTenantId = tenantService.GetCurrentTenantId();
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Now, EVERY query against the Users table will automatically append:
        // "WHERE TenantId = @currentTenantId"
        modelBuilder.Entity<User>()
            .HasQueryFilter(u => u.TenantId == _currentTenantId);
            
        modelBuilder.Entity<SupportTicket>()
            .HasQueryFilter(t => t.TenantId == _currentTenantId);
    }
}
```
This guarantees absolute data isolation at the database layer. A developer literally *cannot* accidentally query another tenant's data.

### Bypassing the Filter (`IgnoreQueryFilters`)
Sometimes, a master system administrator needs to see *everything* (e.g., calculating total active users across all tenants, or physically purging soft-deleted records in a background job).

You can explicitly bypass the global filter on a per-query basis using `.IgnoreQueryFilters()`:

```csharp
// Returns ALL users, bypassing the TenantId and IsDeleted filters!
var allSystemUsers = await dbContext.Users
    .IgnoreQueryFilters() 
    .ToListAsync();
```
