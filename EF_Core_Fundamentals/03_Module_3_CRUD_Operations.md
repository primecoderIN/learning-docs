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

---

## 4. Deleting Data (Delete)

Deleting follows a similar pattern to updating. We fetch the record, pass it to the `Remove` method, and save changes.

```csharp
var existingTenant = await context.Tenants.FindAsync(id);
if (existingTenant == null) return NotFound();

// Deleted only in memory
context.Tenants.Remove(existingTenant); 
// Note: We can also do it directly on the context: context.Remove(existingTenant); 
// This works the exact same way because DbContext knows where Tenants are saved.

// Deleted from the database
await context.SaveChangesAsync(); 
```

### 💡 Pro-Tip: The One-Round-Trip Deletion Trick
In the example above, we made a round-trip to the database just to fetch the record so we could delete it. If we don't care about throwing a 404 Not Found error (e.g., if it's already gone, we don't care), we can do this in **one round trip**:

```csharp
// Create a "stub" object with just the Primary Key
var stubTenant = new Tenant { Id = id };

// Tell EF Core to delete it
context.Tenants.Remove(stubTenant);

// Executes the DELETE statement based purely on the ID!
await context.SaveChangesAsync();
```

## 5. Debugging EF Core (Logging)
If you want to see exactly what SQL statements EF Core is generating, you can enable console logging in your options builder:

```csharp
optionsBuilder.LogTo(Console.WriteLine);
```
*Note:* By default, EF Core does not log sensitive data (like parameter values) until explicitly enabled via `EnableSensitiveDataLogging()`.
