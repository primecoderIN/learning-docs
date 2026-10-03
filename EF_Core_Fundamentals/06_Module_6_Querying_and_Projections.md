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
