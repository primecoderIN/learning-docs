# Module 3: Essential CRUD Operations

In this module, we will explore how to Create, Read, Update, and Delete records in our database using Entity Framework Core. We will also look at how EF Core handles queries and tracks changes in memory before saving them to the database.

---

## 1. Fetching Data (Read)

When fetching a single record from the database, EF Core provides three primary methods. 

### Ways to Fetch a Single Record
1.  **`FirstOrDefaultAsync`**: Queries the database and returns the first match it finds. It returns `null` if no match is found.
    ```csharp
    var tenant = await context.Tenants.FirstOrDefaultAsync(t => t.Id == id);
    ```
2.  **`SingleOrDefaultAsync`**: Similar to `FirstOrDefault`, but it **throws an error** if more than one match is found in the database. Use this when you expect exactly one record or none.
    ```csharp
    var tenant = await context.Tenants.SingleOrDefaultAsync(t => t.Id == id);
    ```
3.  **`FindAsync`**: This is a highly optimized method that checks memory first! It serves the match from the local in-memory tracker if it has already been fetched during this request; otherwise, it queries the database.
    ```csharp
    var tenant = await context.Tenants.FindAsync(id);
    ```

### Comparison: `FirstOrDefault` vs `SingleOrDefault` vs `Find`

| Method | DB Hit? | Throws if >1 result? | Checks cache first? | Use When |
|---|---|---|---|---|
| `FirstOrDefaultAsync` | Always | No | No | You expect 0 or 1 result, order matters |
| `SingleOrDefaultAsync` | Always | Yes | No | You expect exactly 0 or 1 (enforces business rule) |
| `FindAsync` | Only if not cached | No | **Yes** | Fetching by primary key repeatedly in the same request |

### IQueryable vs IEnumerable (Deferred Execution)
When you write LINQ against a `DbSet`, you are working with an `IQueryable<T>`. 
*   **IQueryable** does not immediately fetch data. It represents a query that can be built progressively and executed by the query provider. This is called **Deferred Execution**. *Note: For large data sets, consider pagination with deferred execution.*
*   **IEnumerable** makes sense when data is already loaded into memory. We use it to filter on an in-memory collection, iterate over a sequence, and process objects using LINQ to Objects.

```csharp
// 1. Create a queryable reference (No database hit yet!)
IQueryable<Tenant> allTenants = context.Tenants;

// 2. Apply a filter (Still no database hit!)
IQueryable<Tenant> filteredTenants = allTenants.Where(t => t.IsActive);

// 3. Execute the query (Hits the database!)
var tenantsList = await filteredTenants.ToListAsync(); 
```
*Best Practice:* Filtering happens at the database level rather than loading everything into memory. If there are 1,000 tenants and only 100 are active, only 100 rows will be fetched over the network!

### Understanding Projection
Projection helps us prevent fetching redundant data. If you only need the `Id` and `Name` of a tenant, don't fetch the entire row!

```csharp
var year = 2024;
var filteredTitles = await context.Tenants
    .Where(t => t.CreatedYear == year)
    .Select(t => new { Id = t.Id, Title = t.Name })
    .ToListAsync();
```
This generates an optimized SQL query fetching only what was mentioned in the DTO: 
`SELECT [t].[Id], [t].[Name] FROM [SystemTenants] As [t] WHERE DATEPART(year, [t].[CreatedDate]) = @__year_0`

*(Note: We will cover the massive performance implications of Projections and exactly how they work under the hood in **Module 6: Querying and Projections**).*

---

## 2. Creating Data (Create)

To insert a new record, we use the `AddAsync` method followed by `SaveChangesAsync`.

```csharp
public async Task<IActionResult> CreateTenant(Tenant newTenant)
{
    // 1. Add to the DbContext (tracked in memory, no ID assigned yet)
    await context.Tenants.AddAsync(newTenant);
    
    // 2. Commit to the database (INSERT statement executed)
    await context.SaveChangesAsync();

    // After SaveChanges, the 'Id' property is automatically populated by EF Core!
    return CreatedAction(nameof(GetTenant), new { id = newTenant.Id }, newTenant);
}
```

> **Note on `Add` vs `AddAsync`:** For SQL Server, `Add()` (synchronous) is actually fine for adding entities because no I/O happens — the entity is just registered in memory. The actual I/O happens during `SaveChangesAsync()`. `AddAsync()` exists for database providers with special async ID generation (like sequences in PostgreSQL). For most apps using SQL Server with GUID or identity keys, `Add()` is perfectly acceptable.

---

## 3. Updating Data (Update)

EF Core makes updating data incredibly intuitive through its **Change Tracker**. You don't need to write `UPDATE` commands; you just modify the C# object.

```csharp
// 1. Fetch the existing record
var existingTenant = await context.Tenants.FindAsync(id);

if (existingTenant == null) return NotFound();

// 2. Modify the properties (Changes are only tracked in memory!)
existingTenant.Name = request.NewName;
existingTenant.Domain = request.NewDomain;

// 3. Save Changes (Generates and executes the UPDATE SQL)
await context.SaveChangesAsync();
```

### Disconnected Updates (ASP.NET Core APIs)
In a web API, you typically receive a DTO from the client, not a tracked entity. You must re-fetch the entity first:

```csharp
// CORRECT: Fetch-then-update pattern
public async Task<IActionResult> UpdateTenant(Guid id, UpdateTenantRequest request)
{
    var tenant = await context.Tenants.FindAsync(id);
    if (tenant == null) return NotFound();

    // Map DTO onto the tracked entity
    tenant.Name = request.Name;
    tenant.Domain = request.Domain;

    await context.SaveChangesAsync();
    return NoContent();
}
```

> **WRONG approach (avoid):** Never call `context.Update(dto)` on a DTO that was not fetched from the database. This marks ALL properties as modified — even ones the user didn't change — generating a full `UPDATE` statement that overwrites every column, including ones you didn't intend to change (like `CreatedAt`, `CreatedByUserId`).

---

## 4. Deleting Data: Hard Delete vs. Soft Delete

When removing data from an enterprise application, you must choose between a **Hard Delete** (permanently erasing the row from SQL) or a **Soft Delete** (flagging the row as deleted but keeping the historical data).

### Approach A: The Hard Delete (`.Remove()`)
A hard delete permanently destroys the record in the database.

```csharp
var existingTenant = await context.Tenants.FindAsync(id);
if (existingTenant == null) return NotFound();

// Deleted only in memory
context.Tenants.Remove(existingTenant); 

// Physically deleted from the database (DELETE FROM SystemTenants WHERE Id = ...)
await context.SaveChangesAsync(); 
```

#### 💡 Pro-Tip: The One-Round-Trip Hard Delete Trick
In the example above, we made a round-trip to the database just to fetch the record so we could delete it. If we don't care about throwing a 404 Not Found error (e.g., if it's already gone, we don't care), we can do this in **one round trip**:

```csharp
// Create a "stub" object with just the Primary Key
var stubTenant = new Tenant { Id = id };

// Tell EF Core to delete it
context.Tenants.Remove(stubTenant);

// Executes the DELETE statement based purely on the ID!
await context.SaveChangesAsync();
```

### Approach B: The Soft Delete (The Enterprise Standard)
In systems like Normora, you rarely want to Hard Delete a Tenant or User. Doing so would orphan audit logs, break historical invoices, and destroy data you might need for compliance.

Instead, we use a **Soft Delete**. We add a boolean flag to our entity and *Update* the record instead of deleting it.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    
    // The Soft Delete Flag
    public bool IsDeleted { get; set; } 
}
```

When an admin clicks "Delete", we do not call `.Remove()`. We update the flag:

```csharp
var existingTenant = await context.Tenants.FindAsync(id);
if (existingTenant == null) return NotFound();

// SOFT DELETE: Just flip the flag!
existingTenant.IsDeleted = true;

// Executes an UPDATE statement, NOT a DELETE statement!
await context.SaveChangesAsync();
```

*(Note: To ensure these soft-deleted records don't show up in normal queries, you combine this with the **Global Query Filter** pattern discussed in Module 6).*

### Advanced: Intercepting `.Remove()` to force a Soft Delete
Sometimes developers accidentally call `context.Remove()` even when they are supposed to Soft Delete. You can protect your database by overriding the `SaveChanges` method in your `DbContext` to automatically intercept deletions and convert them into updates!

```csharp
public override int SaveChanges()
{
    // Find all entities that are currently marked as "Deleted" in memory
    var entries = ChangeTracker.Entries()
        .Where(e => e.State == EntityState.Deleted);

    foreach (var entry in entries)
    {
        // If the entity supports soft delete (e.g. has an IsDeleted property)
        if (entry.Entity is Tenant tenant) // Or an ISoftDeletable interface
        {
            // 1. Change the state from "Deleted" back to "Modified"
            entry.State = EntityState.Modified;
            
            // 2. Flip the soft delete flag
            tenant.IsDeleted = true;
        }
    }

    return base.SaveChanges();
}
```
Now, even if a junior developer calls `.Remove()`, EF Core will secretly swap it to an `UPDATE IsDeleted = 1` right before it hits the database!

### 💡 Pro-Tip: Using an Interface for Generic Soft Delete
Instead of checking `if (entry.Entity is Tenant)`, use an interface to apply soft delete to all eligible entities:

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

// In SaveChanges override:
foreach (var entry in ChangeTracker.Entries<ISoftDeletable>()
                                   .Where(e => e.State == EntityState.Deleted))
{
    entry.State = EntityState.Modified;
    entry.Entity.IsDeleted = true;
    entry.Entity.DeletedAt = DateTime.UtcNow;
}
```

---

## 5. Debugging EF Core (Logging)
If you want to see exactly what SQL statements EF Core is generating, you can enable console logging in your options builder:

```csharp
optionsBuilder.LogTo(Console.WriteLine);
```
*Note:* By default, EF Core does not log sensitive data (like parameter values) until explicitly enabled via `EnableSensitiveDataLogging()`.

For production applications, route EF Core logs through the standard .NET logging system:
```csharp
builder.Services.AddDbContext<TenantsDbContext>(options =>
{
    options.UseSqlServer(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information)
           .EnableSensitiveDataLogging(); // Only in Development!
});
```

---

## 6. Real-World Scenarios

### Scenario A: Paginated User List (Normora Dashboard)
The admin dashboard in Normora shows a paginated list of users. Using deferred execution, the filter, sort, and page are all pushed to SQL:

```csharp
public async Task<List<UserDto>> GetUsersAsync(Guid tenantId, int page, int pageSize, string? search)
{
    var query = context.Users
        .Where(u => u.TenantId == tenantId && !u.IsDeleted);

    if (!string.IsNullOrEmpty(search))
        query = query.Where(u => u.Name.Contains(search) || u.Email.Contains(search));

    return await query
        .OrderBy(u => u.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(u => new UserDto { Id = u.Id, Name = u.Name, Email = u.Email })
        .ToListAsync();
}
```
**Generated SQL:** Only the requested page of filtered, sorted users is fetched — never the full table.

### Scenario B: Bulk Insert with `AddRange`
When a new tenant signs up, Normora seeds default departments:

```csharp
var defaultDepartments = new[]
{
    new Department { Name = "Engineering", TenantId = newTenant.Id },
    new Department { Name = "HR", TenantId = newTenant.Id },
    new Department { Name = "Finance", TenantId = newTenant.Id },
};

// Add all at once — EF Core batches these into fewer round-trips
context.Departments.AddRange(defaultDepartments);
await context.SaveChangesAsync();
```

### Scenario C: Implementing ISoftDeletable System-Wide
```csharp
// Base class for all soft-deletable entities
public abstract class SoftDeletableEntity : ISoftDeletable
{
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// All entities that should be soft-deleted inherit from it
public class Tenant : SoftDeletableEntity { ... }
public class User : SoftDeletableEntity { ... }
```
Combined with the Global Query Filter (`.HasQueryFilter(e => !e.IsDeleted)`) and the `SaveChanges` interceptor, this gives a bulletproof, zero-effort soft delete system.

---

## 7. Interview Questions

**Q1: What is the difference between `FirstOrDefaultAsync` and `SingleOrDefaultAsync`?**
> Both return one record or `null`. The difference is `SingleOrDefaultAsync` **throws an `InvalidOperationException`** if more than one record matches. Use `FirstOrDefault` when you only care about getting *one* result (and don't mind if multiple exist). Use `SingleOrDefault` when your business logic dictates exactly one result should ever match — it enforces this as a database-level assertion.

**Q2: What is the difference between a Hard Delete and a Soft Delete? When would you use each?**
> A Hard Delete uses `context.Remove()` and executes a physical `DELETE` SQL statement — the data is permanently gone. A Soft Delete adds an `IsDeleted` boolean flag and updates it to `true` instead, keeping the row in the database. Use Soft Delete in any enterprise/compliance-heavy system where you need audit trails, historical data integrity, or the ability to recover deleted records. Use Hard Delete for truly ephemeral data (like temporary tokens or sessions) where historical records have zero value.

**Q3: What is deferred execution in EF Core? Why does it matter?**
> Deferred execution means that writing a LINQ query (`.Where()`, `.OrderBy()`, etc.) on an `IQueryable` does NOT hit the database immediately. The query is only executed when you call a terminating method like `.ToListAsync()`, `.FirstOrDefaultAsync()`, or `.CountAsync()`. This matters because you can build up complex, conditional queries (adding filters based on input parameters) without hitting the database multiple times — only one final, optimized SQL query is sent.

**Q4: How does EF Core know which properties were changed when you call `SaveChanges`?**
> The **Change Tracker** takes a snapshot of every tracked entity's property values when they are first loaded from the database. When `SaveChanges` is called, it compares the current property values against the original snapshot. Only properties that differ are included in the `UPDATE` SQL statement. This means EF Core generates minimal, precise SQL — updating only what actually changed.

**Q5: What is the "stub entity" trick for deletion, and when is it useful?**
> Instead of fetching an entity from the database just to delete it (which costs a round-trip), you create a new instance of the entity class with only the primary key set, and then call `.Remove()` on it. EF Core treats it as if it were a real tracked entity and generates a `DELETE WHERE Id = ...` statement. This is useful when you don't need to validate 404 (or handle it at the database level) and want to save a database round-trip.

**Q6: Why is it dangerous to call `context.Update(dto)` on a DTO received from an API request?**
> When you call `context.Update()`, EF Core marks ALL properties of the entity as `Modified`, which generates an `UPDATE` statement that overwrites every column in the row. If the DTO doesn't contain all fields (e.g., it omits `CreatedAt`), those fields will be set to their default values, destroying existing data. The safe pattern is to always fetch the entity first, then map only the changed fields from the DTO onto the tracked entity.

---

## Summary
In this module, we mastered the four CRUD operations in EF Core. Key takeaways: use deferred execution with `IQueryable` to push filtering to the database; use soft deletes for compliance-critical data; always use the fetch-then-update pattern in disconnected APIs; and intercept `SaveChanges` to automate cross-cutting concerns like soft delete and auditing.
