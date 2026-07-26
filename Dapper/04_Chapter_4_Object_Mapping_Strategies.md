# Chapter 4: Object Mapping Strategies

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Understand the limitations of flat DTO mapping and why hierarchical object graphs require explicit mapping strategies.
*   Master Dapper's Multi-Mapping API (`Query<TFirst, TSecond, TReturn>`) to resolve One-to-One, One-to-Many, and Many-to-Many relationships.
*   Effectively utilize the `splitOn` parameter and internal dictionary state to eliminate the "N+1 query" problem.
*   Implement custom `SqlMapper.ITypeHandler` classes to seamlessly map advanced SQL Server types (like `JSON` and `Geography`) to domain-specific C# objects.
*   Evaluate the memory and performance trade-offs of application-level grouping versus database-level joining.

## 2. Introduction

Relational databases store data in flat, two-dimensional tables. Object-oriented languages represent data as complex, nested, hierarchical graphs. The friction between these two paradigms is famously known as the "Object-Relational Impedance Mismatch."

Full ORMs like Entity Framework Core attempt to completely hide this mismatch. They allow developers to navigate navigation properties (e.g., `Site.Chargers.First()`), and the ORM translates these traversals into SQL queries on the fly. While highly productive, this abstraction frequently leads to disastrous performance bottlenecks, most notably the "N+1 query" problem, where iterating over a parent collection executes a separate SQL query for every single child collection.

Dapper takes the opposite approach. It does not hide the database; it forces you to embrace it. When you need to retrieve a parent entity and its children, Dapper requires you to write a single, optimized SQL `JOIN` statement. It then provides a powerful, high-performance API called **Multi-Mapping** to stitch that flat SQL result set back into a rich C# object graph in memory. This chapter explores how to master these mapping strategies for enterprise applications.

## 3. Core Concepts

### Multi-Mapping
Dapper allows you to map a single row from a `SqlDataReader` into multiple distinct C# objects. It does this by scanning the columns left-to-right and splitting the row into distinct segments based on a designated column name, usually the primary key of the joined table.

### The `splitOn` Parameter
The core mechanism that powers multi-mapping is the `splitOn` parameter. When Dapper encounters the column specified in `splitOn`, it assumes that this column marks the beginning of the next object in the generic type list. By default, `splitOn` is assumed to be `"Id"`.

### Application-Level Deduplication
When you execute a SQL `JOIN` (e.g., `SELECT * FROM Sites INNER JOIN Chargers`), the parent data (`Site`) is duplicated for every child row (`Charger`). Dapper does *not* automatically deduplicate this for you. If a Site has 5 Chargers, Dapper will return 5 distinct `Site` objects. It is the developer's responsibility to use a tracking mechanism (usually a C# `Dictionary`) within the mapping delegate to deduplicate the parent object and append the children to its collection.

## 4. Visual Diagrams

```text
=============================================================================
             DAPPER MULTI-MAPPING INTERNAL EXECUTION (One-To-Many)
=============================================================================

[ SQL Result Set (INNER JOIN Sites & Chargers) ]
Row 1: [SiteId:1] [Name:"HQ"] |splitOn| [ChargerId:10] [Model:"Fast"]
Row 2: [SiteId:1] [Name:"HQ"] |splitOn| [ChargerId:11] [Model:"Slow"]

[ Dapper Multi-Mapping Parser ]
        │
        ├── Reads Row 1
        │    ├── Maps columns to `Site` object (Id:1, Name:HQ)
        │    ├── Encounters 'ChargerId' (splitOn)
        │    └── Maps remaining to `Charger` object (Id:10, Model:Fast)
        │         └── Invokes Mapping Delegate: Func<Site, Charger, Site>
        │
        └── Reads Row 2
             ├── Maps columns to NEW `Site` object (Id:1, Name:HQ)
             ├── Encounters 'ChargerId' (splitOn)
             └── Maps remaining to `Charger` object (Id:11, Model:Slow)
                  └── Invokes Mapping Delegate: Func<Site, Charger, Site>

[ Application Delegate (Using a Dictionary for Deduplication) ]
        │
        ├── Invocation 1:
        │    Is Site Id(1) in Dictionary? NO.
        │    Add Site(1) to Dictionary.
        │    Add Charger(10) to Site(1).Chargers.
        │    Return Site(1).
        │
        └── Invocation 2:
             Is Site Id(1) in Dictionary? YES. (Discard the duplicate Site obj)
             Get existing Site(1) from Dictionary.
             Add Charger(11) to existing Site(1).Chargers.
             Return existing Site(1).
```

## 5. API Deep Dive

### Query<TFirst, TSecond, TReturn>
**Signature:**
```csharp
public static IEnumerable<TReturn> Query<TFirst, TSecond, TReturn>(
    this IDbConnection cnn, 
    string sql, 
    Func<TFirst, TSecond, TReturn> map, 
    object param = null, 
    IDbTransaction transaction = null, 
    bool buffered = true, 
    string splitOn = "Id", 
    int? commandTimeout = null, 
    CommandType? commandType = null)
```

**How it works:**
1. Dapper executes the `sql`.
2. For each row, it generates `TFirst` and `TSecond` based on the `splitOn` string.
3. It calls your `map` function, passing in the two instantiated objects.
4. Your `map` function contains the logic to link them together and returns `TReturn`.
5. Dapper supports up to 7 input types (`TFirst` through `TSeventh`).

### Custom Type Handlers (`SqlMapper.ITypeHandler`)
Sometimes, a database column doesn't map cleanly to a primitive C# type. For example, a `NVARCHAR(MAX)` containing a JSON payload, or a `UNIQUEIDENTIFIER` that you want to map to a strongly-typed domain ID struct (e.g., `TenantId`).

You implement `ITypeHandler`:
```csharp
public interface ITypeHandler
{
    void SetValue(IDbDataParameter parameter, object value);
    object Parse(Type destinationType, object value);
}
```
You then register it globally exactly once at application startup:
`SqlMapper.AddTypeHandler(new MyCustomHandler());`

## 6. Complete Examples: EV Charging Platform

### Scenario 1: One-to-One Mapping
A `Charger` has exactly one `ChargerModel` (manufacturer specs).

```csharp
public async Task<IEnumerable<ChargerDto>> GetChargersWithModelsAsync(Guid siteId)
{
    const string sql = @"
        SELECT 
            c.Id, c.SerialNumber, c.Status,
            -- Split happens here
            m.Id, m.Manufacturer, m.MaxKw
        FROM Chargers c
        INNER JOIN ChargerModels m ON c.ModelId = m.Id
        WHERE c.SiteId = @SiteId";

    // map: (charger, model) => { ... }
    var result = await _connection.QueryAsync<ChargerDto, ChargerModelDto, ChargerDto>(
        sql,
        (charger, model) => 
        {
            // Stitch the graph
            charger.Model = model;
            return charger;
        },
        new { SiteId = siteId },
        splitOn: "Id" // Tells Dapper to split when it hits the second 'Id' column
    );

    return result;
}
```

### Scenario 2: One-to-Many Mapping (The Dictionary Deduplication Pattern)
A `Site` has many `Chargers`. We must execute a `JOIN` and deduplicate the `Site` objects.

```csharp
public async Task<IEnumerable<SiteDto>> GetSitesWithChargersAsync(Guid tenantId)
{
    const string sql = @"
        SELECT 
            s.Id, s.Name, s.Address,
            -- Split happens here (ChargerId)
            c.Id AS ChargerId, c.SerialNumber, c.Status
        FROM Sites s
        LEFT JOIN Chargers c ON s.Id = c.SiteId
        WHERE s.TenantId = @TenantId";

    // We use a dictionary to track which Sites we've already seen
    var siteDictionary = new Dictionary<Guid, SiteDto>();

    var result = await _connection.QueryAsync<SiteDto, ChargerDto, SiteDto>(
        sql,
        (site, charger) =>
        {
            // 1. Try to get the existing site from the dictionary
            if (!siteDictionary.TryGetValue(site.Id, out var existingSite))
            {
                // 2. If it's new, add it to the dictionary and initialize the list
                existingSite = site;
                existingSite.Chargers = new List<ChargerDto>();
                siteDictionary.Add(existingSite.Id, existingSite);
            }

            // 3. Add the child object to the parent's collection (handling LEFT JOIN nulls)
            if (charger != null && charger.ChargerId != Guid.Empty)
            {
                existingSite.Chargers.Add(charger);
            }

            // 4. Return the tracking reference
            return existingSite;
        },
        new { TenantId = tenantId },
        splitOn: "ChargerId" // Must match the column alias in the SQL exactly!
    );

    // Dapper's 'result' variable actually contains a list of duplicate Site references.
    // The dictionary contains the true, deduplicated list.
    return siteDictionary.Values;
}
```

### Scenario 3: Custom Type Handler for JSON
Our EV Chargers send a raw JSON payload of telemetry data. We store this in a `NVARCHAR(MAX)` column named `RawTelemetry`, but we want it strongly typed as `TelemetryConfig` in C#.

**1. Create the Handler:**
```csharp
using System.Text.Json;
using System.Data;
using Dapper;

public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T>
{
    public override void SetValue(IDbDataParameter parameter, T value)
    {
        parameter.Value = value == null 
            ? DBNull.Value 
            : JsonSerializer.Serialize(value);
        parameter.DbType = DbType.String;
    }

    public override T Parse(object value)
    {
        if (value == null || value is DBNull) return default;
        return JsonSerializer.Deserialize<T>(value.ToString());
    }
}
```

**2. Register at Startup (Program.cs):**
```csharp
SqlMapper.AddTypeHandler(new JsonTypeHandler<TelemetryConfig>());
```

**3. Use in Repository (Completely transparent):**
```csharp
// Dapper automatically invokes the TypeHandler during mapping
var session = await _connection.QuerySingleAsync<ChargingSession>(
    "SELECT Id, RawTelemetry FROM Sessions WHERE Id = @Id", 
    new { Id = 1 });

// session.RawTelemetry is already a fully populated TelemetryConfig object!
Console.WriteLine(session.RawTelemetry.BatteryTemperature);
```

## 7. Performance Implications

When dealing with large One-to-Many relationships, the architectural choice between joining vs multiple queries is critical.

**Approach A: Single JOIN + Dictionary Deduplication**
*   **Pros:** Only one network round-trip to the database.
*   **Cons:** Network bandwidth waste. If a Site has 50 columns and 100 Chargers, the 50 Site columns are duplicated and transmitted over the network 100 times. Dapper instantiates 100 duplicate `Site` C# objects before the dictionary deduplicates and garbage collects 99 of them.

**Approach B: GridReader (Multiple Active Queries)**
*   **Pros:** Only one network round trip. Zero data duplication over the wire. Zero duplicate object instantiation in C#.
*   **Cons:** Requires manual LINQ stitching in memory.

**GridReader Alternative to Dictionary Pattern:**
```csharp
public async Task<IEnumerable<SiteDto>> GetSitesGridReaderAsync(Guid tenantId)
{
    const string sql = @"
        SELECT * FROM Sites WHERE TenantId = @TenantId;
        SELECT c.* FROM Chargers c 
        INNER JOIN Sites s ON c.SiteId = s.Id 
        WHERE s.TenantId = @TenantId;";

    using var multi = await _connection.QueryMultipleAsync(sql, new { TenantId = tenantId });
    
    var sites = (await multi.ReadAsync<SiteDto>()).ToList();
    var chargers = (await multi.ReadAsync<ChargerDto>()).ToList();

    // Stitching in application memory (Fast and allocation efficient)
    var chargersBySite = chargers.GroupBy(c => c.SiteId).ToDictionary(g => g.Key, g => g.ToList());

    foreach (var site in sites)
    {
        if (chargersBySite.TryGetValue(site.Id, out var siteChargers))
        {
            site.Chargers = siteChargers;
        }
    }

    return sites;
}
```
*Architect's Rule:* Use the Dictionary pattern for small relationships (e.g., an Order with 5 LineItems). Use GridReader for large relationships (e.g., a Site with 500 Chargers) to avoid massive network payload bloat.

## 8. Common Mistakes

### Beginner
*   **Mistake:** Forgetting to specify the `splitOn` parameter when it's not `"Id"`.
*   **Correction:** If your joined table's primary key is aliased as `ChargerId`, you MUST pass `splitOn: "ChargerId"`. Otherwise, Dapper throws a multi-mapping exception indicating it couldn't find the split column.
*   **Mistake:** Misunderstanding what Dapper returns from a One-to-Many multi-map.
*   **Correction:** Dapper returns the exact number of rows returned by SQL. If the SQL returned 10 rows (1 Site, 10 Chargers), Dapper returns an `IEnumerable<Site>` with 10 elements. You must return the `.Values` of your deduplication dictionary, not the direct output of Dapper's `Query` method.

### Intermediate
*   **Mistake:** Using a LEFT JOIN but failing to handle null children in the mapping delegate.
*   **Correction:** If a Site has no Chargers, the LEFT JOIN returns `NULL` for the Charger columns. Dapper instantiates a `Charger` object, but it will be empty (or null depending on type). You must add a null check (`if (charger != null && charger.Id != 0)`) before adding it to the parent's collection.
*   **Mistake:** Putting heavy logic inside the `TypeHandler.Parse` method.
*   **Correction:** The `Parse` method is executed synchronously on the thread processing the `SqlDataReader`. If you do something slow (like a blocking HTTP call or complex crypto), you will cripple database throughput.

### Senior
*   **Mistake:** Mapping deeply nested hierarchies (e.g., Tenant -> Site -> Charger -> Session) using a massive 5-table JOIN and a 5-generic-type Dapper Multi-Map.
*   **Correction:** While Dapper supports up to 7 types, debugging a 5-type mapping delegate is a nightmare. Furthermore, the Cartesian product of a 5-table join causes astronomical network payload bloat. For deeply nested graphs, strictly use `QueryMultiple` (GridReader) to fetch the flat tables separately and assemble them in C# memory.
*   **Mistake:** Registering the same `TypeHandler` multiple times.
*   **Correction:** `SqlMapper.AddTypeHandler` modifies a static global configuration. It should only be called once, typically in `Program.cs` or Startup. Calling it repeatedly per request throws an exception or causes threading issues.

### Architect
*   **Mistake:** Relying on Dapper Multi-Mapping for the Write/Command stack in CQRS.
*   **Correction:** Multi-mapping is a tool for projecting flat data into hierarchical DTOs for the Read API. Attempting to use Dapper to manage the state of a complex hierarchical domain aggregate (e.g., updating a Site, removing two Chargers, and adding one Charger in a single transaction) is reinventing a Full ORM. Use Entity Framework Core for complex Write aggregations where Change Tracking manages the graph state automatically.

## 9. Interview Questions

### Beginner (10)
1.  **What is the purpose of the `splitOn` parameter in Dapper?**
    *Answer:* It tells Dapper which column name marks the boundary between the first object and the second object when mapping a single row to multiple C# classes.
2.  **What is the default value of the `splitOn` parameter?**
    *Answer:* `"Id"`.
3.  **If I join three tables, how many generic arguments must I pass to `Query`?**
    *Answer:* Four. Three for the input types mapped from the columns, and a fourth representing the final return type (e.g., `Query<TFirst, TSecond, TThird, TReturn>`).
4.  **Does Dapper automatically remove duplicate parent objects when mapping a One-to-Many relationship?**
    *Answer:* No. The mapping delegate will be invoked for every row returned by the SQL query. You must manually deduplicate using a tracking mechanism like a Dictionary.
5.  **What interface must a class implement to tell Dapper how to parse a custom database column?**
    *Answer:* `SqlMapper.ITypeHandler`.
6.  **Can Dapper map multiple result sets from a single query execution?**
    *Answer:* Yes, using the `QueryMultiple` method which returns a `GridReader`.
7.  **If I use a `LEFT JOIN` and the right side is null, what does Dapper pass into my mapping delegate?**
    *Answer:* Dapper passes a `null` reference for that object (if it's a reference type) or a default value (if a struct) into the mapping delegate.
8.  **Where should you configure and add Custom Type Handlers?**
    *Answer:* Globally, exactly once, during application startup (e.g., in `Program.cs`).
9.  **What happens if the `splitOn` column name does not exist in the SQL result set?**
    *Answer:* Dapper throws an `ArgumentException` stating "When using the multi-mapping APIs ensure you set the splitOn param if you have keys other than Id".
10. **Can I use `splitOn` with multiple columns?**
    *Answer:* Yes. If mapping three objects, you provide a comma-separated list of split points (e.g., `splitOn: "SiteId,ChargerId"`).

### Intermediate (10)
11. **Explain the Cartesian product problem when using Dapper multi-mapping for One-to-Many relationships.**
    *Answer:* A SQL JOIN duplicates the parent columns for every child row. If a Site row is 1KB and has 1,000 chargers, SQL Server transmits the same 1KB Site data 1,000 times over the network (1MB of wasted bandwidth). Dapper will also temporarily allocate 1,000 duplicate `Site` objects in memory before your dictionary deduplicates them.
12. **How do you solve the Cartesian product problem in Dapper for large collections?**
    *Answer:* By abandoning `JOINs` and using `QueryMultiple` (GridReader). You execute two separate flat queries (`SELECT * FROM Sites; SELECT * FROM Chargers;`) and manually link the collections in C# memory using LINQ `GroupBy` or Dictionaries.
13. **Write the C# signature of the mapping delegate required to join a `User` and their `Role` into a `UserDto`.**
    *Answer:* `Func<User, Role, UserDto>`
14. **How do you map a database `VARCHAR(10)` column storing "Active" or "Inactive" directly to a C# boolean?**
    *Answer:* Create a custom `TypeHandler<bool>`. In `Parse`, return `value.ToString() == "Active"`. In `SetValue`, set the parameter to `"Active"` or `"Inactive"`.
15. **If I map three objects using `Query<A, B, C, A>`, and my SQL is `SELECT A.*, B.*, C.*`, what should the `splitOn` value be?**
    *Answer:* It should be a comma-separated string containing the first column name of table B, and the first column name of table C (e.g., `splitOn: "Id,Id"` assuming both have an `Id` column).
16. **Why is it dangerous to return the direct result of a Dapper One-to-Many multi-map query?**
    *Answer:* Because the direct result is an `IEnumerable` containing duplicate parent object references (one for each row in the SQL result). You must return the `.Values` collection of your deduplication dictionary instead.
17. **Can I use Dapper multi-mapping to populate a nested Dictionary inside my object?**
    *Answer:* Yes. Inside the mapping delegate, instead of adding to a `List<T>`, you can use the child object's properties to `Add(key, value)` into the parent's Dictionary property.
18. **What is the maximum number of types Dapper's multi-mapping generic methods support?**
    *Answer:* Up to 7 input types.
19. **How does Dapper handle mapping SQL `JSON` columns in modern SQL Server?**
    *Answer:* SQL Server returns JSON as standard strings (`NVARCHAR`). Dapper will map it to a C# `string`. To map it directly to a POCO, you must write a `TypeHandler` that wraps `System.Text.Json`.
20. **Is the mapping delegate passed to Dapper compiled into the IL emit?**
    *Answer:* No. Dapper emits IL to instantiate the individual objects (e.g., the Site and the Charger) efficiently. It then invokes your provided C# `Func` delegate synchronously, passing those instantiated objects to it.

### Senior (10)
21. **Analyze the GC (Garbage Collection) pressure difference between Dictionary deduplication mapping and GridReader mapping for 100,000 rows.**
    *Answer:* Dictionary mapping (via JOIN) instantiates 100,000 parent objects, deduplicates them, and leaves 99,000 orphaned parent objects eligible for Gen 0 collection, causing rapid GC churn. GridReader executes two flat queries, instantiating exactly the required number of parents and children with zero duplicate allocations, resulting in significantly lower GC pressure.
22. **When implementing a `TypeHandler` for SQL Server's `hierarchyid` type, why must you reference `Microsoft.SqlServer.Types`?**
    *Answer:* Because `hierarchyid` is a CLR User-Defined Type (UDT) inside SQL Server. It is returned as a binary stream. To parse it correctly in C# without manually decoding the complex binary format, you need the official SQL Server types library to convert it back to a readable string or object.
23. **You are using Dapper to query a legacy database where the primary key column names change per table (e.g., `CustId`, `OrdId`). How do you handle multi-mapping?**
    *Answer:* You must explicitly define the `splitOn` parameter with a comma-separated list of the exact column aliases used in the query. For example: `splitOn: "OrdId,LineId"`.
24. **In a high-throughput API, your custom JSON `TypeHandler` is causing thread pool starvation. Profiling shows high CPU in `JsonSerializer.Deserialize`. How do you fix this without changing the DB schema?**
    *Answer:* JSON deserialization is synchronous within Dapper's mapping pipeline. To fix this, change Dapper to map the column to a `string` (or a `Lazy<T>`), and perform the `JsonSerializer.DeserializeAsync` *outside* of Dapper, closer to the API layer, allowing the thread pool to scale asynchronously.
25. **Explain how Dapper determines which properties to map when columns have identical names across joined tables (e.g., `CreatedAt`).**
    *Answer:* Dapper relies entirely on the `splitOn` definition. It maps columns sequentially left-to-right from the `SqlDataReader`. Once it hits a column matching the `splitOn` name, it stops populating the first object and starts populating the second object. The order of columns in the SQL `SELECT` clause absolutely dictates which object receives which `CreatedAt` value.
26. **What is the `SqlMapper.AddTypeMap` method and how does it differ from a `TypeHandler`?**
    *Answer:* `TypeHandler` converts a specific database value to a specific C# property type. `AddTypeMap` (or `SqlMapper.SetTypeMap`) allows you to override Dapper's entire strategy for mapping an object, such as implementing custom rules to map snake_case columns to PascalCase properties globally, or manually specifying which column maps to which property.
27. **Why is it an anti-pattern to execute Dapper `QueryFirst` inside a `foreach` loop to retrieve children for a list of parents?**
    *Answer:* This is the classic "N+1 query" problem. It executes 1 query for the parents, and N queries for the children. This causes N network round-trips, crippling performance. It should be replaced with a single SQL JOIN + Multi-Mapping, or a batched `QueryMultiple`.
28. **How would you map a Many-to-Many relationship (e.g., `Users`, `Roles`, and a `UserRoles` join table) using Dapper?**
    *Answer:* You execute a query joining all three tables. In your C# code, you need two dictionaries for deduplication: one for `Users` and one for `Roles`. In the mapping delegate, you ensure the User exists in the User dictionary, ensure the Role exists in the Role dictionary, and then add the Role reference to the User's `Roles` collection.
29. **You receive an exception: `Multi-mapping APIs cannot be used with types that do not have a default constructor.` Why?**
    *Answer:* While Dapper supports mapping to parameterized constructors for single-object mapping, Multi-Mapping relies on instantiating the objects before passing them to your custom `Func` delegate. If the types lack a parameterless constructor, Dapper cannot automatically instantiate them during the split phase.
30. **How do you pass a strongly-typed `List<int>` to a Stored Procedure as a Table-Valued Parameter (TVP) in Dapper?**
    *Answer:* You cannot pass a `List<int>` directly. You must convert it to a `DataTable` matching the SQL Server User-Defined Table Type, or use a custom extension method (like Dapper's `AsTableValuedParameter(typeName)` method) to marshal the data correctly.

### Staff Engineer (5)
31. **Architect a high-performance Read Model projector for a CQRS system that denormalizes an Order with 100 LineItems into a MongoDB document using Dapper as the source.**
    *Answer:* To avoid Cartesian product bloat in SQL Server and memory churn in C#, I would use Dapper's `QueryMultiple`. I would query the `Orders` table, and then query the `LineItems` table using `WHERE OrderId IN @OrderIds`. In C#, I would use a `Dictionary<OrderId, List<LineItem>>` to stitch them in memory, serialize the complete Order graph to JSON, and push it to MongoDB. This maximizes SQL Server's set-based retrieval and minimizes network payload.
32. **A developer wrote a `TypeHandler` for AES-256 encryption. It decrypts PII data during mapping. What are the security, performance, and architectural flaws here?**
    *Answer:* **Performance:** Decryption is CPU intensive. Doing it synchronously inside Dapper's materialization loop will block the thread pool and increase query latency exponentially. **Architecture:** Data access layers should be ignorant of business-level crypto. **Security:** The keys must be injected globally into a static `TypeHandler`, which is a severe security risk. Decryption should happen asynchronously at the Application/Domain layer, not during ORM mapping.
33. **Explain the impact of `splitOn` string parsing on Dapper's `Identity` Cache and how a poorly designed dynamic SQL generator can cause cache misses here.**
    *Answer:* The `splitOn` string is part of Dapper's internal hash key for caching IL delegates. If a dynamic SQL generator accidentally changes the casing of the `splitOn` parameter, or dynamically changes the column alias being used to split (e.g., `splitOn: "Id"` vs `splitOn: "ID"`), Dapper will generate a completely new IL delegate, leading to cache bloat and CPU overhead from repeated `Reflection.Emit` calls.
34. **How would you implement pagination (OFFSET/FETCH) on a One-to-Many query using JOINs without corrupting the page size?**
    *Answer:* You cannot simply append `OFFSET 0 FETCH NEXT 10 ROWS ONLY` to a `JOIN` query. Because of row duplication, this limits the *total rows*, not the *parent rows*. You might get 1 Site with 9 Chargers, instead of 10 Sites. You must use a CTE (Common Table Expression). The CTE performs the pagination on the Parent table (`Sites`) first. The outer query then `JOINs` the Chargers exclusively to the paginated Sites within the CTE.
35. **Evaluate the use of SQL Server `FOR JSON PATH` combined with Dapper instead of Dapper Multi-Mapping.**
    *Answer:* This is a highly advanced, highly efficient pattern. Instead of using Dapper Multi-Mapping or GridReader, you instruct SQL Server to construct the JSON graph natively (`SELECT * FROM Sites JOIN Chargers FOR JSON PATH`). Dapper simply executes `QuerySingle<string>` to get the raw JSON, and you use `System.Text.Json` to deserialize it directly into the full object graph. This offloads the hierarchical construction to the database engine (written in C++) and minimizes network traffic to a single string, often outperforming Dapper's C# mapping for complex graphs.

### Architect (5)
36. **In an Event-Driven Microservices architecture, Domain Events contain flat IDs, but the UI needs enriched hierarchical data. Contrast using Dapper Multi-Mapping on the fly versus an asynchronous Read Model projection.**
    *Answer:* Multi-mapping on the fly means the API executes complex JOINs in real-time. This is acceptable for low-to-medium scale, but couples the UI to the normalized schema and risks database locking under load. An asynchronous projection (e.g., listening to events and updating a flattened JSON document in SQL Server or a NoSQL store) is architecturally superior for high scale. The API then uses Dapper to simply `QuerySingle<string>` the pre-computed document, eliminating JOINs and mapping overhead entirely at read-time.
37. **Your multi-tenant platform uses a single SQL database. A requirement emerges to support custom fields (EAV pattern) per tenant on the `Charger` entity. Architect a Dapper read strategy that maps these custom fields efficiently.**
    *Answer:* The Entity-Attribute-Value (EAV) pattern via JOINs is notoriously slow. Instead, I would architect a `CustomData` `NVARCHAR(MAX)` column storing JSON on the `Chargers` table. I would implement a Dapper `TypeHandler` mapping this JSON to an `IDictionary<string, object>`. Dapper reads the row, the handler parses the JSON to a dictionary, and the Application Layer can seamlessly serve tenant-specific custom fields without complex multi-table EAV JOINs.
38. **Justify the strict architectural rule: "CQRS Command Handlers must never use Dapper Multi-Mapping."**
    *Answer:* Command Handlers are responsible for enforcing business invariants and changing state. To do this, they require a cohesive Domain Aggregate. Reconstructing a complex aggregate using manual Dapper Multi-Mapping and Dictionaries is reinventing Change Tracking and Unit of Work. It leads to fragile, boilerplate-heavy code. Command Handlers should use an ORM like EF Core to hydrate the aggregate, mutate it, and save it. Dapper Multi-Mapping is strictly a tool for projecting flat data into DTOs for the Query Handlers.
39. **Design a resilience strategy for a Dapper GridReader `QueryMultiple` execution that hits a transient Azure SQL connection drop between reading the first and second result set.**
    *Answer:* Once `QueryMultiple` starts executing, the network stream is active. If the connection drops between `.Read()` calls, the entire batch is compromised. You cannot retry just the second read. The resilience strategy must wrap the *entire* `QueryMultiple` invocation (including connection opening) inside a Polly `AsyncRetryPolicy`. The operation must be inherently idempotent, and the policy will re-execute the entire batched SQL statement upon detecting a transient `SqlException`.
40. **How do you enforce schema/DTO synchronization in a Dapper-heavy architecture? Full ORMs provide migrations; how do you prevent developers from renaming a DTO property and breaking Dapper mapping in production?**
    *Answer:* This is a CI/CD architectural challenge. We must implement Integration Testing. We spin up a Testcontainers SQL Server instance in the pipeline. We use reflection to find every Dapper Query defined in the codebase. We execute them against the schema to ensure the SQL is syntactically valid. Furthermore, we write tests that assert the DTO properties are successfully mapped (not null/default) by comparing the Dapper output to expected dataset fixtures. Without rigorous integration testing, Dapper projects degrade into runtime failures.

## 10. Exercises

### Easy
1.  **Multi-Mapping Setup:** Create a `Tenants` table and a `Sites` table with a foreign key to `TenantId`. Write a Dapper query that joins them. Use the `Query<Tenant, Site, Tenant>` method to map the result, placing a breakpoint in the mapping delegate to observe how Dapper splits the row.

### Medium
1.  **Dictionary Deduplication:** Expand the previous exercise. Insert 1 Tenant and 5 Sites. Use the Dictionary pattern inside the mapping delegate to deduplicate the Tenant object and populate a `List<Site>` on the Tenant. Return the correctly mapped, single Tenant object.

### Hard
1.  **Type Handlers:** Create a C# struct `UserId(Guid Value)`. Create a `Users` table in SQL Server with an `Id UNIQUEIDENTIFIER` column. Write a `SqlMapper.TypeHandler<UserId>` that seamlessly parses the SQL Guid into your strongly-typed `UserId` struct during a Dapper `Query`.

### Enterprise
1.  **GridReader vs JSON PATH:** Implement two CQRS Query Handlers to retrieve a massive `Tenant` -> `Sites` -> `Chargers` hierarchy. 
    *   Handler A uses `QueryMultiple` (GridReader), executing three flat SELECTs and stitching the collections together using C# LINQ `ToDictionary`.
    *   Handler B executes a single SQL Server query using `FOR JSON PATH` and uses `System.Text.Json` to deserialize the resulting string.
    *   Benchmark both endpoints using BenchmarkDotNet to compare CPU allocation and network round-trip time.

## 11. Summary

Dapper does not protect you from the complexities of relational database architectures; it empowers you to leverage them fully. By mastering Multi-Mapping, GridReader, and Custom Type Handlers, you transition from simply querying flat rows to architecting high-performance data pipelines capable of projecting complex object hierarchies.

The critical lesson of this chapter is understanding the network and memory implications of your mapping choices. While Multi-Mapping with JOINs is convenient, GridReader and JSON-parsing often provide superior scalability for deep, data-heavy relationships. In the next chapter, we will explore Advanced Queries, tackling Stored Procedures, Bulk Operations, and Transaction management.
