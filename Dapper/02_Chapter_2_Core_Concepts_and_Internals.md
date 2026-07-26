# Chapter 2: Core Concepts and Internal Architecture

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Explain the internal architecture of Dapper and how it extends the ADO.NET foundation.
*   Understand the role of Reflection, IL Generation (Emit), and Caching in Dapper's ultra-high performance.
*   Master the concept of Parameter Binding using both anonymous objects and `DynamicParameters` to prevent SQL injection.
*   Evaluate the performance and memory implications of Buffered vs. Unbuffered query execution.
*   Design thread-safe database connection management strategies within an ASP.NET Core environment.

## 2. Introduction

To truly master Dapper, one must look beneath the simple `connection.Query<T>()` API and understand the mechanical sympathy it has with the .NET Runtime and ADO.NET. Dapper is not magic; it is essentially a highly optimized, intelligent wrapper around `SqlDataReader`. 

When working on enterprise systems, a Solution Architect cannot treat an ORM as a black box. You must understand *why* it is fast, how it allocates memory, and how it translates your C# intent into Tabular Data Stream (TDS) protocol commands sent to SQL Server. This chapter dissects Dapper's internal engine, explaining how it achieves object mapping without the severe reflection penalties that plague other serialization frameworks.

## 3. Core Concepts: The Anatomy of Dapper

### What is Dapper Internally?
At its core, Dapper is simply a collection of extension methods on the `System.Data.IDbConnection` interface. When you reference the Dapper NuGet package, you are not inheriting from a base class, registering heavy background services, or configuring a complex DbContext. You are simply gaining access to methods like `Query`, `Execute`, and `QueryMultiple` on your existing database connections.

The entire framework originally started as a single file: `SqlMapper.cs`. 

### The Mapping Pipeline
When you execute `connection.Query<User>("SELECT Id, Name FROM Users");`, Dapper does not blindly execute the query and use Reflection to populate the object row-by-row. That would be disastrously slow. Instead, it performs the following pipeline:

1. **Hash Generation (Identity):** Dapper creates a structural hash based on the SQL string, the connection string, the target type (`User`), and the type of parameters passed.
2. **Cache Lookup:** It checks an internal static `ConcurrentDictionary` (the Cache) using this hash to see if a mapping delegate for this exact query already exists.
3. **Execution:** Dapper passes the SQL and parameters down to ADO.NET (`SqlCommand`), calls `ExecuteReader()`, and obtains an `IDataReader`.
4. **Delegate Generation (Cache Miss):** If the delegate is not in the cache, Dapper inspects the `User` class using Reflection. It looks at the properties and matches them to the column names returned by the `IDataReader`. 
5. **IL Emit:** Instead of using slow runtime Reflection to populate the object on every row, Dapper uses `System.Reflection.Emit.DynamicMethod` to write raw Intermediate Language (IL) instructions dynamically. It writes a custom, highly optimized method specifically for mapping *this exact* `IDataReader` shape to *this exact* `User` class.
6. **Caching & Execution:** The generated IL delegate is compiled, stored in the Cache for future use, and then executed. The delegate rapidly reads the rows, instantiates `User` objects, and populates a list.

### Reflection vs IL Emit
Reflection in .NET is slow because it requires metadata lookups at runtime (e.g., "Find the property named 'Id' on this object, check if I have write access, then set its value"). 

Dapper pays the "Reflection Tax" **exactly once** per query/type combination. By dynamically emitting IL, Dapper writes machine code at runtime that is identical in performance to a developer manually writing:
```csharp
var user = new User();
user.Id = reader.GetInt32(0);
user.Name = reader.GetString(1);
```
This is the secret to Dapper's speed. It writes the boilerplate for you at runtime, caches the compiled delegate, and reuses it for millions of subsequent executions.

## 4. Visual Diagrams

```text
=============================================================================
                  DAPPER QUERY EXECUTION PIPELINE
=============================================================================

[ Application Layer ]
        │  connection.Query<User>("SELECT * FROM Users WHERE Id = @Id", new { Id = 1 })
        ▼
[ Dapper: SqlMapper Extension ]
        │  1. Create Identity Hash (SQL + Expected Type + Param Type)
        ▼
[ Identity Cache (Static ConcurrentDictionary) ]
        │───▶ CACHE HIT: Retrieve compiled IL Delegate ────┐
        │                                                  │
        │───▶ CACHE MISS:                                  │
        │      1. Call SqlCommand.ExecuteReader()          │
        │      2. Map DB Columns to C# Properties          │
        │      3. Emit IL via DynamicMethod                │
        │      4. Compile to Action/Func Delegate          │
        │      5. Store in Cache                           │
        ▼                                                  ▼
[ ADO.NET Execution ]                              [ IL Delegate Execution ]
  SqlCommand.ExecuteReader()                         Invoke() on SqlDataReader
        │                                                  │
        ▼                                                  ▼
[ SQL Server ] ◀─────── (TDS Protocol) ────────▶ [ Object Materialization ]
                                                           │
                                                           ▼
                                              [ Return IEnumerable<User> ]
```

## 5. Internal Implementation Deep Dive

### Parameter Binding
To prevent SQL injection, you must never concatenate strings. Dapper abstracts away ADO.NET's `SqlParameterCollection`.

**Anonymous Objects:**
The most common approach is passing an anonymous object. Dapper parses the object's properties (again, caching the mapping) and creates the underlying `SqlParameter` objects.
```csharp
// Dapper automatically translates this into @TenantId and @Status SqlParameters
var users = await connection.QueryAsync<User>(
    "SELECT * FROM Users WHERE TenantId = @TenantId AND Status = @Status",
    new { TenantId = currentTenantId, Status = "Active" });
```

**DynamicParameters:**
For dynamic SQL building, output parameters, or stored procedures, Dapper provides `DynamicParameters`. This class allows you to imperatively add parameters, specify their exact DB type, size, and direction.
```csharp
var parameters = new DynamicParameters();
parameters.Add("@TenantId", currentTenantId, DbType.Guid);

if (includeInactive) {
    parameters.Add("@Status", "Inactive", DbType.String, size: 20);
}

// Used for Stored Procedure Output
parameters.Add("@TotalCount", dbType: DbType.Int32, direction: ParameterDirection.Output);

var users = await connection.QueryAsync<User>(sql, parameters);
int count = parameters.Get<int>("@TotalCount");
```

### Buffered vs Unbuffered Queries
Understanding this distinction is critical for memory management in enterprise applications.

**Buffered Queries (Default):**
By default, Dapper queries are **Buffered**. When you call `Query<T>`, Dapper reads *all* rows from the `SqlDataReader`, materializes them into `T` objects, adds them to a `List<T>`, and returns the fully populated list as an `IEnumerable<T>`.
*   **Pros:** The database connection can be released to the pool immediately. The result is safe to iterate multiple times. Predictable memory usage for small datasets.
*   **Cons:** High memory allocation if you select millions of rows.

**Unbuffered Queries:**
If you pass `buffered: false` to the `Query` method, Dapper uses the C# `yield return` keyword. It returns an `IEnumerable<T>` that streams objects one-by-one as you iterate over them, without loading the entire result set into memory at once.
```csharp
// The SqlConnection remains OPEN and locked while you iterate!
var users = connection.Query<User>("SELECT * FROM Users", buffered: false);

foreach (var user in users) 
{
    // Process one user at a time. The memory footprint stays flat.
    ProcessUser(user); 
}
```
*   **Warning:** The underlying `SqlDataReader` and `SqlConnection` are held open while iterating. You cannot execute other commands on that connection until iteration completes or is disposed. If you send this unbuffered `IEnumerable` to a slow consumer (like an HTTP response stream), you tie up a database connection for a long time.

### Thread Safety and Connection Management
*   **Dapper is Thread-Safe:** The internal caches (`ConcurrentDictionary`) and extension methods are entirely thread-safe.
*   **SqlConnection is NOT Thread-Safe:** You must never share a single `SqlConnection` instance across multiple threads or concurrent requests. 

**Best Practice:** Always instantiate a new `SqlConnection` inside a `using` block (or rely on Dependency Injection with a `Scoped` lifetime). Let the underlying ADO.NET Connection Pool manage the physical network connections. Creating a `new SqlConnection()` is virtually free; it just takes a connection from the pool.

## 6. Real Production Case Study: EV Charging Platform

In our Multi-Tenant EV Charging Management Platform, we need to handle incoming "Heartbeat" messages from chargers. This endpoint will be hit thousands of times per minute. We need absolute maximum performance.

Let's look at the connection lifecycle and Dapper execution within our Clean Architecture setup.

```csharp
using Microsoft.Data.SqlClient;
using Dapper;

namespace EVPlatform.Infrastructure.Data
{
    public class ChargerRepository : IChargerRepository
    {
        private readonly string _connectionString;

        // The connection string is injected, NOT the connection itself.
        // We control the connection lifecycle locally to ensure thread safety.
        public ChargerRepository(IConfiguration configuration)
        {
            _connectionString = configuration.GetConnectionString("EVDatabase");
        }

        public async Task UpdateHeartbeatAsync(Guid chargerId, DateTime timestamp, decimal currentKwh)
        {
            // 1. Connection instantiation. 
            // Gets a connection from the ADO.NET pool.
            await using var connection = new SqlConnection(_connectionString);
            
            // 2. Optimized T-SQL using UPSERT (MERGE) logic if needed, or simple UPDATE
            const string sql = @"
                UPDATE Chargers 
                SET LastHeartbeat = @Timestamp, 
                    TotalKwh = @CurrentKwh 
                WHERE Id = @ChargerId";
            
            // 3. Dapper execution with anonymous parameters
            // Because this query does not return data, we use ExecuteAsync.
            await connection.ExecuteAsync(sql, new 
            { 
                ChargerId = chargerId, 
                Timestamp = timestamp, 
                CurrentKwh = currentKwh 
            });

            // 4. Connection is automatically disposed and returned to the pool 
            // at the end of the async method due to 'await using'.
        }
    }
}
```

## 7. Common Mistakes

### Beginner
*   **Mistake:** Reusing a single `SqlConnection` instance across the entire application (e.g., registering it as a `Singleton` in ASP.NET Core DI).
*   **Correction:** `SqlConnection` is not thread-safe. Register it as `Transient` or `Scoped`, or better yet, inject the connection string and create `using var conn = new SqlConnection()` locally. The connection pool handles the efficiency.
*   **Mistake:** Putting a `try/catch` around Dapper but forgetting to close the connection in a `finally` block (when not using `using` statements).
*   **Correction:** Always use `using` or `await using` to ensure connections are returned to the pool even if an exception occurs.

### Intermediate
*   **Mistake:** Passing `DynamicParameters` to every query just out of habit, even when an anonymous object would suffice.
*   **Correction:** Anonymous objects are slightly faster and result in cleaner code. Only use `DynamicParameters` when you need conditional parameters, output parameters, or explicit `DbType` mapping.
*   **Mistake:** Returning an Unbuffered query (`buffered: false`) directly from an ASP.NET Core controller.
*   **Correction:** The JSON serializer will slowly iterate the results, keeping the SQL connection open for the duration of the HTTP response. If the client is on a slow network, you will exhaust your SQL connection pool. Buffer the results first, or use a dedicated streaming architecture.

### Senior
*   **Mistake:** Building dynamic SQL strings (e.g., for complex search filters) and appending values directly into the string, thinking Dapper will parameterize them if they are wrapped in a Dapper call.
*   **Correction:** Dapper only parameterizes what you pass in the `param` argument. If you concatenate `$"WHERE Name = '{name}'"`, Dapper hashes that literal string. This causes Cache Bloat. Use a library like `SqlBuilder` (part of Dapper) to dynamically build queries while maintaining parameterization.
*   **Mistake:** Ignoring the `CommandTimeout` parameter on long-running reports.
*   **Correction:** ADO.NET defaults to a 30-second timeout. Dapper respects this. For heavy analytics queries, you must explicitly set the timeout in the Dapper call (e.g., `commandTimeout: 120`).

### Architect
*   **Mistake:** Not monitoring the SQL Server Plan Cache and Dapper's internal Identity Cache memory usage. If a developer dynamically generates `IN` clauses (e.g., `WHERE Id IN (1,2)` then `WHERE Id IN (1,2,3)`), both caches will bloat exponentially until the application crashes with an OOM exception.
*   **Correction:** Architect a standard querying pattern. Enforce the use of Dapper's list expansion feature (`WHERE Id IN @Ids`) which properly parametrizes arrays and stabilizes the cache hashes.

## 8. Interview Questions

### Beginner (10)
1.  **What namespace contains the Dapper extension methods?**
    *Answer:* `Dapper`.
2.  **Does Dapper open the database connection automatically?**
    *Answer:* Yes, if you call a Dapper method on a closed connection, Dapper will implicitly open it, execute the query, and close it. However, it is a best practice to manage the connection state explicitly if you are doing multiple operations.
3.  **What is the default timeout for a Dapper query?**
    *Answer:* Dapper defaults to the underlying ADO.NET timeout, which is 30 seconds.
4.  **How do you pass parameters to a Dapper query?**
    *Answer:* Usually by passing an anonymous object as the second argument (e.g., `new { Id = 1 }`).
5.  **What does the `Execute` method return?**
    *Answer:* It returns an `int` representing the number of rows affected by the query (useful for INSERT, UPDATE, DELETE).
6.  **Can I map a query result to a primitive type like `int` or `string`?**
    *Answer:* Yes, you can do `connection.Query<int>("SELECT COUNT(*) FROM Users")`.
7.  **What is the purpose of the `using` statement when working with `SqlConnection`?**
    *Answer:* It ensures that `Dispose()` is called on the connection, which safely returns it to the ADO.NET connection pool, even if an exception is thrown.
8.  **Will Dapper throw an error if my SQL query returns more columns than my C# class has properties?**
    *Answer:* No. Dapper will safely ignore any extra columns in the result set that do not have a matching property in the C# class.
9.  **Will Dapper throw an error if my C# class has properties that are not returned by the SQL query?**
    *Answer:* No. Those properties will simply remain at their default values (e.g., `null` or `0`).
10. **What is an anonymous object in C# and why does Dapper use them?**
    *Answer:* An anonymous object is a read-only type generated by the compiler without an explicit class name (`new { Name = "John" }`). Dapper uses them to provide a concise, strongly-typed way to pass parameters without creating temporary classes.

### Intermediate (10)
11. **Explain what happens when you set `buffered: false` in a Dapper `Query`.**
    *Answer:* Dapper skips creating a `List<T>` to hold the results. Instead, it uses `yield return` to stream the objects directly from the `SqlDataReader`. This reduces memory usage but keeps the database connection open while iterating.
12. **How does Dapper avoid the performance penalty of .NET Reflection?**
    *Answer:* It only uses Reflection once per query/type combination. It uses the reflected metadata to emit highly optimized Intermediate Language (IL) via `DynamicMethod`, compiles it into a delegate, caches it, and executes that delegate for all future calls.
13. **What is the `SqlMapper.Identity` struct used for?**
    *Answer:* It represents the unique hash key used in Dapper's internal cache. It is comprised of the SQL string, the connection string, the command type, the type of the parameters, and the expected return type.
14. **Why is it a bad idea to inject an `IDbConnection` as a Singleton in ASP.NET Core?**
    *Answer:* `IDbConnection` implementations (like `SqlConnection`) are not thread-safe. A Singleton would be shared across all concurrent HTTP requests, leading to thread contention, mixed query results, and exceptions regarding the connection state.
15. **How do you handle output parameters from a Stored Procedure in Dapper?**
    *Answer:* You must use `DynamicParameters`. You add the parameter and set `direction: ParameterDirection.Output`. After calling `Execute`, you retrieve the value using `parameters.Get<T>("@ParamName")`.
16. **How does Dapper handle C# `enum` types when mapping from SQL?**
    *Answer:* Dapper automatically maps SQL integers to C# enums based on the underlying integer value. It can also map SQL strings to enums if the string matches the enum name (case-insensitive).
17. **What is connection pooling and how does it relate to Dapper?**
    *Answer:* Connection pooling is a feature of ADO.NET (not Dapper) that keeps physical database connections open in a pool to avoid the high cost of TCP handshakes on every request. Dapper utilizes this automatically when you instantiate and dispose of `SqlConnection` objects.
18. **If I execute `connection.Query("SELECT * FROM Users")` without specifying a generic `<T>`, what does Dapper return?**
    *Answer:* It returns an `IEnumerable<dynamic>`. Each row is represented as a `DapperRow` object (which implements `IDictionary<string, object>`).
19. **How do you pass a list of IDs to an `IN` clause using Dapper?**
    *Answer:* You pass an `IEnumerable` (like a list or array) in your parameter object. Dapper's list expansion feature automatically detects this and transforms the SQL from `WHERE Id IN @Ids` to `WHERE Id IN (@Ids1, @Ids2, @Ids3)` and generates the corresponding parameters.
20. **Can Dapper map private properties or fields?**
    *Answer:* Out of the box, Dapper only maps to public properties. However, you can configure custom mappings or use the constructor-mapping feature if the column names match the constructor parameter names.

### Senior (10)
21. **Explain the potential impact of "Cache Bloat" in Dapper and how to prevent it.**
    *Answer:* Dapper caches IL delegates based on the exact SQL string. If you generate dynamic SQL by appending literals (e.g., `WHERE Name = 'John'` vs `'Jane'`), Dapper creates a new delegate for every distinct string. This causes unbounded memory growth in Dapper's static cache and SQL Server's plan cache. Prevention requires strictly using parameterized inputs (`WHERE Name = @Name`) so the SQL string remains static.
22. **When implementing a Repository pattern with Dapper, how do you handle Unit of Work and Transactions across multiple repositories?**
    *Answer:* You must share the connection and transaction. You can create a `UnitOfWork` class that manages a single `SqlConnection` and `SqlTransaction`. The UoW injects this transaction into the repositories, and the repositories pass the `transaction: _dbTransaction` parameter to all Dapper extension methods.
23. **What is `SqlMapper.PurgeQueryCache()` and when might you need to use it?**
    *Answer:* It clears Dapper's internal `ConcurrentDictionary` cache of IL delegates. You rarely need it in production, but it might be necessary in a long-running application if you are heavily generating dynamic SQL (which is an anti-pattern) and need to manually prevent an OutOfMemory exception.
24. **How does Dapper's Multi-Mapping feature work internally compared to standard mapping?**
    *Answer:* For Multi-Mapping (e.g., `Query<T1, T2, TReturn>`), Dapper emits a slightly more complex IL delegate. It scans the `SqlDataReader` columns left-to-right, looks for the `splitOn` column name (usually "Id"), and splits the row into segments. It then applies the standard IL Emit mapping logic to each segment separately, instantiates the multiple objects, and passes them to the custom `Func<T1, T2, TReturn>` you provided to stitch them together.
25. **If a SQL query takes 5 minutes to execute, how do you prevent the Dapper method from blocking a thread-pool thread in a high-throughput API?**
    *Answer:* You must use the `Async` methods (e.g., `QueryAsync`) which utilize `SqlDataReader.ReadAsync`. This relies on I/O Completion Ports (IOCP) at the OS level, freeing the .NET thread-pool thread while waiting for the network response from SQL Server.
26. **You have an unbuffered query that processes 10 million rows. You need to write the results to a CSV file. What network/DB risks does this pose?**
    *Answer:* Because the unbuffered query keeps the `SqlDataReader` open, it holds a lock on the SQL Server session. If writing to the CSV involves slow disk I/O, the network buffers between SQL Server and the application will fill up. SQL Server might block (ASYNC_NETWORK_IO wait type) waiting for the application to consume the TDS packets, degrading database performance for other queries.
27. **How do you execute multiple completely independent SQL statements in a single network round trip using Dapper?**
    *Answer:* You use `connection.QueryMultiple()`, which returns a `GridReader`. You write a single SQL string with multiple statements separated by semicolons. You then call `Read<T>()` on the `GridReader` sequentially to process each result set.
28. **Explain the behavior of Dapper when encountering a database `DBNull.Value`.**
    *Answer:* Dapper checks for `DBNull` in the emitted IL. If the DB value is null, and the C# property is a nullable type (e.g., `string` or `int?`), it sets it to `null`. If the C# property is a non-nullable value type (e.g., `int`), Dapper will throw a `DataException` during deserialization.
29. **What is the `CommandFlags` enumeration in Dapper?**
    *Answer:* `CommandFlags` allows you to control specific behaviors per query, such as `CommandFlags.Buffered` (to disable buffering manually on the `CommandDefinition`), `CommandFlags.NoCache` (to prevent Dapper from caching the IL delegate for this specific query), and `CommandFlags.Pipelined` (for async pipelining).
30. **How would you map a custom JSON column in SQL Server to a strongly-typed nested C# object using Dapper?**
    *Answer:* You create a custom class implementing `SqlMapper.ITypeHandler`. In the `Parse` method, you take the string value from SQL and use `System.Text.Json.JsonSerializer.Deserialize<T>()`. In the `SetValue` method, you serialize the object to a string. You register this handler globally using `SqlMapper.AddTypeHandler(new MyJsonTypeHandler())` at application startup.

### Staff Engineer (5)
31. **Analyze the garbage collection (GC) profile of an unbuffered Dapper query vs a buffered query.**
    *Answer:* A buffered query allocates a large `List<T>`, backing arrays, and all `T` instances immediately. This often pushes the allocations into Gen 2 of the GC if the list is large, causing expensive Gen 2 sweeps later. An unbuffered query allocates `T` instances one at a time. Once the `foreach` loop processes the instance, it becomes eligible for Gen 0 collection, keeping memory pressure low and GC pauses extremely short.
32. **We are migrating from EF Core to Dapper. EF Core's `AsNoTracking()` queries are allocating 100MB/sec, but Dapper is allocating 80MB/sec. Why isn't Dapper's allocation zero?**
    *Answer:* Dapper still has to allocate strings. Every `string` in C# is a reference type allocated on the heap. If your query returns 10,000 rows with 5 string columns, Dapper must allocate 50,000 string objects. To optimize further, you would need to look at intern pools or refactoring the domain to use smaller value types (structs) where possible, but the string allocation is a .NET fundamental, not a Dapper flaw.
33. **Explain how ADO.NET Connection Pooling determines if a connection is "clean" when returned by Dapper, and what happens if a Transaction was left open?**
    *Answer:* When `Dispose()` is called on a `SqlConnection`, ADO.NET intercepts it. It calls `sp_reset_connection` on the SQL Server to reset session state (temporary tables, SET options). If an explicit `SqlTransaction` was started but neither committed nor rolled back, the `Dispose` process will automatically roll back the pending transaction before returning the connection to the pool to prevent state leakage to the next consumer.
34. **Design a mechanism to intercept and log all slow Dapper queries (execution time > 1 second) across a massive microservices architecture without modifying every repository method.**
    *Answer:* I would implement a custom `DbConnection` wrapper (Decorator pattern) or use a tool like MiniProfiler. The decorator implements `IDbConnection` and wraps the actual `SqlConnection`. It intercepts `CreateCommand()`, returning a custom `DbCommand` wrapper. This wrapper starts a `Stopwatch` in `ExecuteReaderAsync`, executes the underlying command, and if the stopwatch exceeds 1 second, logs the SQL, parameters, and execution plan to Application Insights or OpenTelemetry before returning the reader to Dapper.
35. **Why does Dapper's list expansion feature (`WHERE Id IN @Ids`) cap at 2,000 parameters when executing against SQL Server?**
    *Answer:* This is a hard limitation of SQL Server's TDS protocol and the Database Engine, which supports a maximum of 2,100 parameters per batch (RPC call). If Dapper expands an array of 3,000 integers into `@Id1, @Id2... @Id3000`, the `SqlCommand` will throw an exception before executing. For lists larger than ~2,000 items, you must use Table-Valued Parameters (TVPs) or `SqlBulkCopy`.

### Architect (5)
36. **Architect an idempotent background worker that processes 1 million delayed events using Dapper, guaranteeing exactly-once processing even during server crashes.**
    *Answer:* I would use a Transactional Outbox pattern combined with a SQL Server `UPDLOCK`. The worker executes a Dapper query: `BEGIN TRAN; SELECT TOP (100) * FROM Events WITH (UPDLOCK, READPAST) WHERE Status = 'Pending';`. `READPAST` allows concurrent workers to skip locked rows. The worker processes the events, uses Dapper to update their status to 'Processed' within the same transaction, and then `COMMIT`. If the worker crashes, the transaction rolls back, releasing the locks, and another worker picks them up.
37. **Your multi-tenant SaaS shares a single SQL database. A junior developer deployed a Dapper query lacking the `WHERE TenantId = @id` clause, exposing data across tenants. Architect a fail-safe mechanism at the Dapper/ADO level to make this impossible.**
    *Answer:* We must enforce Row-Level Security (RLS) at the SQL Server database tier, completely decoupling security from the application's Dapper queries. We configure an `IAsyncDbConnectionInterceptor` in our infrastructure (or a custom connection factory). When a connection is opened, we execute `EXEC sp_set_session_context @key=N'TenantId', @value=@CurrentTenantId` using Dapper. SQL Server's RLS policy then automatically applies a hidden filter to every table, ensuring that even if a developer writes `SELECT * FROM Users`, SQL Server only returns rows matching the session's TenantId.
38. **Evaluate the architectural differences between Dapper and Native AOT (Ahead-of-Time) compilation in .NET 8/9. Will Dapper work in Native AOT?**
    *Answer:* Dapper relies heavily on `System.Reflection.Emit` to generate IL at runtime. Native AOT removes the JIT compiler and runtime code generation capabilities entirely to produce self-contained, high-performance executables. Therefore, standard Dapper **does not work** in a strict Native AOT environment. To achieve Dapper-like performance in Native AOT, an architect must select a Source-Generated ORM (like Dapper.AOT or EF Core 8+ compiled models) which generates the mapping code as standard C# during the compilation phase, bypassing the need for runtime Reflection Emit.
39. **In a high-throughput API, your telemetry shows thread pool starvation. The profiles point to Dapper's `Query` method. Why does this happen if SQL Server is fast, and how do you resolve it?**
    *Answer:* The synchronous `Query` method blocks the calling .NET thread while waiting for network I/O from SQL Server. In high-throughput scenarios, if the API receives a burst of requests, all thread pool threads block, leading to starvation (503 Service Unavailable). This occurs even if the DB is fast, due to network latency. The architectural fix is to enforce a strict asynchronous-all-the-way pattern, migrating all `Query` calls to `QueryAsync`. This uses I/O Completion Ports, allowing the thread to return to the pool while waiting for the TDS response.
40. **How do you manage database schema migrations in a Dapper-centric architecture? Dapper provides no built-in migration tool like EF Core's `Add-Migration`.**
    *Answer:* Dapper is agnostic to schema management. As an Architect, I decouple schema management entirely from the ORM. I integrate a dedicated migration tool like **DbUp**, **FluentMigrator**, or **Flyway** into the CI/CD pipeline. These tools execute versioned SQL scripts (e.g., `V1__Create_Users.sql`). The application simply expects the database schema to match its DTO structures. This provides superior control over indexes, views, and stored procedures compared to relying on EF Core's automatic migration generation.

## 9. Exercises

### Easy
1.  **Cache Observation:** Write a Dapper query using a parameterized input. Run it in a loop 1,000 times. Use a `Stopwatch` to measure the time taken for the first execution vs the subsequent 999 executions. Observe the difference caused by the initial IL Emit and caching.

### Medium
1.  **Dynamic Parameters:** Create a stored procedure in your LocalDB that accepts an input parameter `@SiteId` and returns an output parameter `@TotalChargers`. Use `DynamicParameters` in Dapper to execute the procedure and print the output parameter to the console.

### Hard
1.  **Memory Profiling:** Create a table with 500,000 rows. Write a console application that executes a buffered `Query<T>`. Use Visual Studio's Diagnostic Tools to take a memory snapshot before and after the query. Then, change it to `buffered: false`, iterate using `foreach`, and take another snapshot. Compare the peak heap allocations.

### Enterprise
1.  **Decorator Connection Factory:** Implement an `IDbConnectionFactory` for ASP.NET Core DI. Implement a Decorator around it that injects the current HTTP Request context. When a connection is opened, use Dapper to execute `sp_set_session_context` to set the `TenantId` automatically, ensuring all subsequent Dapper queries on that connection are inherently tenant-scoped for Row-Level Security.

## 10. Summary

Dapper achieves its extreme performance by dynamically generating and compiling Intermediate Language (IL) delegates at runtime, paying the cost of Reflection exactly once per query. It securely parameters data, translates C# intent into highly efficient ADO.NET commands, and manages memory effectively through features like unbuffered streaming. 

However, "with great power comes great responsibility." Because Dapper operates so close to the metal, you must deeply understand ADO.NET connection lifecycles, memory allocation, and thread safety. In the next chapter, we will perform a comprehensive deep dive into the Dapper API surface, exploring every variation of the `Query` and `Execute` methods.
