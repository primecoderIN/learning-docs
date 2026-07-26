# Chapter 3: The Dapper API Deep Dive

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Master the complete surface area of Dapper's `IDbConnection` extension methods.
*   Decide precisely when to use `Query`, `QueryFirst`, `QuerySingle`, `Execute`, or `ExecuteScalar` based on architectural and performance requirements.
*   Understand the performance implications and exception-handling behavior of every API.
*   Utilize the `CommandDefinition` struct to achieve enterprise-grade control over timeouts, cancellation tokens, and execution flags.
*   Implement production-ready repositories using Dapper's asynchronous APIs for the EV Charging Platform.

## 2. Introduction

While Dapper's internal engine is complex, its public API surface is elegantly simple. Dapper operates on a design philosophy of "pit of success"—the default methods are highly optimized, but explicit control is always available when you need it.

In enterprise software, using the "wrong" method (such as using `Query().FirstOrDefault()` instead of `QueryFirstOrDefault()`) can lead to subtle memory leaks, unnecessary database round-trips, or swallowed exceptions that mask data corruption. This chapter provides a forensic examination of every Dapper API, explaining the parameters, internal behaviors, and explicit rules on when *not* to use them.

## 3. Visual Diagrams

```text
=============================================================================
                  DAPPER API SELECTION DECISION TREE
=============================================================================

Are you modifying data (INSERT/UPDATE/DELETE)?
 ├── YES ──▶ Do you need a generated ID/value back?
 │            ├── YES ──▶ ExecuteScalarAsync<T>() 
 │            └── NO ───▶ ExecuteAsync()
 │
 └── NO (It's a SELECT query)
      │
      ├── Does the query return multiple result sets (e.g. 2 SELECTs)?
      │    └── YES ──▶ QueryMultipleAsync() -> GridReader
      │
      └── NO (Single result set)
           │
           ├── Do you need just one primitive value (e.g. SELECT COUNT(*))?
           │    └── YES ──▶ ExecuteScalarAsync<T>()
           │
           └── NO (Mapping to objects)
                │
                ├── Do you expect exactly ONE row, and > 1 is a fatal data error?
                │    ├── Might be 0 rows? ──▶ QuerySingleOrDefaultAsync<T>()
                │    └── Must be 1 row? ───▶ QuerySingleAsync<T>()
                │
                ├── Do you just want the FIRST row (ignoring others)?
                │    ├── Might be 0 rows? ──▶ QueryFirstOrDefaultAsync<T>()
                │    └── Must be 1 row? ───▶ QueryFirstAsync<T>()
                │
                └── You want multiple rows?
                     └── QueryAsync<T>() (Use buffered:false if dataset is huge)
```

## 4. API Deep Dive

### 4.1. The Base Parameters
Almost all Dapper methods share the following base parameters:
*   `string sql`: The T-SQL command to execute.
*   `object param`: (Optional) The parameters to bind. Can be an anonymous object, `DynamicParameters`, or a list.
*   `IDbTransaction transaction`: (Optional) The active transaction. Required if the connection has an open transaction.
*   `int? commandTimeout`: (Optional) Time in seconds to wait before throwing a `SqlException` for a timeout. Defaults to ADO.NET's 30 seconds.
*   `CommandType? commandType`: (Optional) Usually `CommandType.Text` (default) or `CommandType.StoredProcedure`.

### 4.2. Query and QueryAsync
**Signature:** `IEnumerable<T> Query<T>(...)` / `Task<IEnumerable<T>> QueryAsync<T>(...)`
**Purpose:** Executes a SQL query and maps the result set to an `IEnumerable` of type `T`.

**Internal Behavior:**
It calls `ExecuteReader`, loops through the `SqlDataReader`, uses the emitted IL delegate to instantiate `T`, and (if `buffered: true`) adds them to a `List<T>`.

**When NOT to use:**
*   Do not use if you only expect a single row. It will unnecessarily allocate a `List<T>`.
*   Do not use if executing an `INSERT/UPDATE/DELETE`. It expects a result set.

### 4.3. The "First" Family
**APIs:** `QueryFirst<T>`, `QueryFirstOrDefault<T>` (and `*Async` variants)
**Purpose:** Returns the first row of a result set.

**Internal Behavior:**
Dapper executes the query, reads the *very first row* from the `SqlDataReader`, maps it, and immediately calls `Dispose()` on the reader, safely ignoring any subsequent rows that the database might have returned.
*   `QueryFirst`: Throws `InvalidOperationException` if the result set is completely empty.
*   `QueryFirstOrDefault`: Returns `default(T)` (usually `null`) if the result set is empty.

**When NOT to use:**
*   **Crucial Rule:** Do not use this as a lazy way to ignore duplicates. If your database schema dictates that `Email` should be unique, but you have corrupt data with two identical emails, `QueryFirstOrDefault` will hide this corruption. Use `Single` instead.

### 4.4. The "Single" Family
**APIs:** `QuerySingle<T>`, `QuerySingleOrDefault<T>` (and `*Async` variants)
**Purpose:** Returns exactly one row, guaranteeing uniqueness.

**Internal Behavior:**
Dapper reads the first row. Then, it attempts to read a *second* row from the `SqlDataReader`.
*   If a second row exists, it throws an `InvalidOperationException` ("Sequence contains more than one element").
*   `QuerySingle`: Throws if 0 rows or > 1 row.
*   `QuerySingleOrDefault`: Returns `default(T)` if 0 rows. Throws if > 1 row.

**When NOT to use:**
*   Do not use if the SQL query naturally returns multiple rows (e.g., `SELECT * FROM Users`) and you just want the first one. It will throw an exception.

### 4.5. Execute and ExecuteAsync
**APIs:** `int Execute(...)` / `Task<int> ExecuteAsync(...)`
**Purpose:** Executes a command that modifies data (INSERT, UPDATE, DELETE).
**Returns:** An `int` representing the "Rows Affected" as reported by SQL Server.

**Internal Behavior:**
Dapper bypasses `ExecuteReader` and instead calls `SqlCommand.ExecuteNonQuery()`.

**When NOT to use:**
*   Do not use if you need data back (e.g., an `INSERT` statement with an `OUTPUT Inserted.Id` clause). It will only return the rows affected, not the inserted ID.

### 4.6. ExecuteScalar and ExecuteScalarAsync
**APIs:** `T ExecuteScalar<T>(...)` / `Task<T> ExecuteScalarAsync<T>(...)`
**Purpose:** Executes a query and returns the value of the first column of the first row in the result set.

**Internal Behavior:**
Calls `SqlCommand.ExecuteScalar()`. Any additional columns or rows are completely ignored by the database driver. Extremely fast for aggregate functions.

**When NOT to use:**
*   Do not use for mapping whole objects. Use it strictly for single primitive values (e.g., `int`, `decimal`, `bool`, `string`).

### 4.7. QueryMultiple (GridReader)
**APIs:** `SqlMapper.GridReader QueryMultiple(...)` (and `*Async`)
**Purpose:** Executes multiple SQL statements in a single batch and allows reading multiple result sets sequentially.

**Internal Behavior:**
Sends the entire batched SQL string to SQL Server in one network trip. SQL Server executes them and streams back multiple result sets. The returned `GridReader` provides `.Read<T>()` methods. Calling `Read<T>()` advances the internal `SqlDataReader` to the next result set via `NextResult()`.

**When NOT to use:**
*   Do not use if the second query depends on application-layer processing of the first query.

### 4.8. CommandDefinition struct
When you use the simple API (`connection.QueryAsync(sql, param)`), Dapper internally creates a `CommandDefinition` struct. For enterprise applications, you should often use `CommandDefinition` explicitly.

```csharp
var command = new CommandDefinition(
    commandText: "SELECT * FROM Users WHERE Id = @Id",
    parameters: new { Id = 1 },
    transaction: _currentTransaction,
    commandTimeout: 120, // Override to 2 minutes
    commandType: CommandType.Text,
    flags: CommandFlags.None,
    cancellationToken: cancellationToken // Critical for ASP.NET Core!
);

var user = await connection.QuerySingleOrDefaultAsync<User>(command);
```

**Why it's critical:** This is the *only* way to pass a `CancellationToken` to Dapper. In an ASP.NET Core API, if a client disconnects (closes their browser) during a 10-second database query, passing the `CancellationToken` allows Dapper to call `SqlCommand.Cancel()`, immediately terminating the query on the SQL Server and freeing up database CPU and worker threads.

## 5. Complete Examples: EV Charging Platform

### Simple: Reading a Single Entity
```csharp
public async Task<SiteDto?> GetSiteAsync(Guid siteId)
{
    const string sql = @"
        SELECT Id, TenantId, Name, Address, Status 
        FROM Sites 
        WHERE Id = @SiteId";

    // QuerySingleOrDefaultAsync ensures we don't have duplicate Site IDs
    return await _connection.QuerySingleOrDefaultAsync<SiteDto>(
        sql, 
        new { SiteId = siteId });
}
```

### Intermediate: Generating an ID and Returning It
```csharp
public async Task<Guid> CreateTenantAsync(string name, string region)
{
    const string sql = @"
        INSERT INTO Tenants (Id, Name, Region, CreatedAt)
        OUTPUT Inserted.Id -- SQL Server specific output clause
        VALUES (NEWID(), @Name, @Region, GETUTCDATE());";

    // We use ExecuteScalar because we want the first column of the first row (the new ID)
    return await _connection.ExecuteScalarAsync<Guid>(
        sql, 
        new { Name = name, Region = region });
}
```

### Enterprise Production: Cancellable, Transacted, Timeout-Controlled Command
In the EV Platform, generating a monthly invoice for a Tenant requires aggregating millions of charging sessions. This query might take 45 seconds. We must use `CommandDefinition`.

```csharp
public async Task<int> GenerateMonthlyInvoicesAsync(
    Guid tenantId, 
    int month, 
    int year, 
    CancellationToken ct)
{
    const string sql = "sp_GenerateMonthlyInvoices"; // Stored Procedure

    var command = new CommandDefinition(
        commandText: sql,
        parameters: new { TenantId = tenantId, Month = month, Year = year },
        transaction: _activeSqlTransaction, // Participate in an existing UoW
        commandTimeout: 180, // 3 minutes timeout for heavy aggregation
        commandType: CommandType.StoredProcedure,
        cancellationToken: ct // Kill the SP if the API request is aborted
    );

    // Returns the number of invoices generated
    return await _connection.ExecuteAsync(command); 
}
```

## 6. Common Mistakes

### Beginner
*   **Mistake:** Using `.FirstOrDefault()` from LINQ on the result of `Query<T>`: `connection.Query<User>(sql).FirstOrDefault()`.
*   **Correction:** This executes the query, pulls *all* rows into memory into a `List<T>`, and then LINQ throws away everything except the first item. Always use Dapper's native `connection.QueryFirstOrDefault<T>(sql)`.

### Intermediate
*   **Mistake:** Confusing `QueryFirst` and `QuerySingle`.
*   **Correction:** `QueryFirst` is essentially `SELECT TOP 1`. It ignores duplicates. `QuerySingle` is a data integrity check—it explicitly checks if the database erroneously returned multiple rows for what should be a unique query, throwing an exception if it did. Use `Single` when querying by Primary Keys or Unique Constraints.

### Senior
*   **Mistake:** Executing heavy reporting queries via Dapper in an ASP.NET Core controller without passing the `HttpContext.RequestAborted` cancellation token via `CommandDefinition`.
*   **Correction:** If a user clicks "Generate Report" 10 times and refreshes the page, you now have 10 orphaned, heavy queries hammering SQL Server. Always pass the `CancellationToken` to `CommandDefinition`. Dapper will propagate this to ADO.NET, which will send an Attention signal (ATTN) to SQL Server to abort the execution plan.

### Architect
*   **Mistake:** Allowing developers to use `Execute` with `IEnumerable` (List Expansion) for bulk inserts (e.g., passing a `List<User>` to `connection.Execute("INSERT...", listOfUsers)`).
*   **Correction:** While Dapper supports this, it executes a separate `INSERT` statement for *every single item* in the list. For 10,000 items, that is 10,000 network round trips. This will cripple database performance. Enforce the use of `SqlBulkCopy` or Table-Valued Parameters (TVPs) for bulk operations.

## 7. Interview Questions

### Beginner (10)
1.  **What happens if you use `QuerySingle` and the database returns no rows?**
    *Answer:* Dapper throws an `InvalidOperationException` stating the sequence contains no elements.
2.  **If I want to insert a record and don't care about the result, which API should I use?**
    *Answer:* `Execute` or `ExecuteAsync`.
3.  **What is the difference between `QueryFirstOrDefault` and `QuerySingleOrDefault`?**
    *Answer:* `QueryFirstOrDefault` returns the first row and ignores any subsequent rows. `QuerySingleOrDefault` returns the first row, but throws an exception if a second row exists in the result set.
4.  **How do you specify that a command is a Stored Procedure instead of raw SQL?**
    *Answer:* By passing `commandType: CommandType.StoredProcedure` to the Dapper method call or via a `CommandDefinition`.
5.  **What does `ExecuteScalar` return?**
    *Answer:* It returns the value of the first column of the first row of the result set, ignoring everything else.
6.  **Can I use Dapper's `Execute` method to run a `CREATE TABLE` script?**
    *Answer:* Yes. `Execute` works for any Data Definition Language (DDL) or Data Manipulation Language (DML) statement.
7.  **If I want to map a `SELECT COUNT(*)` query, which method is most efficient?**
    *Answer:* `ExecuteScalarAsync<int>()`.
8.  **What happens if a SQL string is syntactically invalid?**
    *Answer:* Dapper passes it to SQL Server, and SQL Server will return an error. Dapper will then surface this as a `SqlException` in your application.
9.  **Why should you prefer the `*Async` methods (like `QueryAsync`) in a web application?**
    *Answer:* They utilize non-blocking I/O. While waiting for the database to respond, the .NET thread is returned to the thread pool to serve other HTTP requests, vastly improving application scalability.
10. **How do you pass a timeout to a simple `Query` call without using `CommandDefinition`?**
    *Answer:* Most Dapper methods have an optional `commandTimeout` parameter (e.g., `connection.Query(sql, param, commandTimeout: 60)`).

### Intermediate (10)
11. **Explain the mechanical difference between `connection.Query(sql).FirstOrDefault()` and `connection.QueryFirstOrDefault(sql)`.**
    *Answer:* The first uses LINQ on the returned `IEnumerable`. Dapper fetches all rows, allocates a `List<T>`, populates it, and then LINQ takes the first. The second uses Dapper natively. Dapper reads exactly one row from the `SqlDataReader`, maps it, and immediately closes the reader, avoiding mass allocations.
12. **In a transaction, if I call `connection.Execute(sql)` but forget to pass the `transaction` parameter, what happens?**
    *Answer:* ADO.NET will throw an `InvalidOperationException` stating that "Execute requires the command to have a transaction when the connection assigned to the command is in a pending local transaction."
13. **What is `SqlMapper.GridReader` and which method returns it?**
    *Answer:* It is returned by `QueryMultiple` (or `QueryMultipleAsync`). It is an object that allows you to sequentially read multiple result sets from a single batched database query.
14. **How do you cancel a long-running Dapper query?**
    *Answer:* You must create a `CommandDefinition` struct, pass your `CancellationToken` into it, and pass the `CommandDefinition` to the Dapper execution method.
15. **If I have a query that returns 5 columns, but I use `ExecuteScalar`, what happens to the other 4 columns?**
    *Answer:* The underlying `SqlCommand.ExecuteScalar` method only reads the very first value. The rest of the columns (and any subsequent rows) are completely ignored at the driver level, making it highly efficient.
16. **Why does Dapper's `Execute` return an integer, and what does it represent?**
    *Answer:* It represents the number of rows affected by an INSERT, UPDATE, or DELETE statement, as reported by SQL Server's `@@ROWCOUNT`.
17. **Can I use `QueryMultiple` to execute a SELECT, an UPDATE, and another SELECT in one round trip?**
    *Answer:* Yes. You can batch them in one SQL string (separated by `;`). The GridReader will skip over the UPDATE (since it produces no result set) and read the two SELECT result sets when you call `.Read<T>()`.
18. **If `QuerySingle` encounters multiple rows, it throws an exception. Does it read the entire table before throwing?**
    *Answer:* No. It reads the first row, then calls `reader.Read()` a second time. If the second read returns true, it immediately throws the exception without reading the rest of the rows.
19. **What is the `CommandFlags.NoCache` flag used for?**
    *Answer:* It instructs Dapper *not* to store the generated IL mapping delegate in its internal `ConcurrentDictionary`. This is useful for dynamically generated queries that will truly only ever be run once, preventing cache bloat.
20. **Is the `CancellationToken` supported on the synchronous Dapper methods?**
    *Answer:* Yes, you can pass a `CancellationToken` to a synchronous `CommandDefinition`. If canceled, ADO.NET will attempt to send a cancellation signal to SQL Server, though asynchronous methods are heavily preferred for cancellation scenarios.

### Senior (10)
21. **How does Dapper handle MARS (Multiple Active Result Sets) differently from `QueryMultiple`?**
    *Answer:* MARS allows multiple separate `SqlDataReader` instances to be interleaved on a single connection. Dapper does not require MARS for `QueryMultiple`. `QueryMultiple` sends a single batched command and processes a single, sequential `SqlDataReader` that moves to the next result set via `NextResult()`. MARS is generally discouraged due to complex locking overhead in SQL Server.
22. **You are implementing a Repository that returns a Domain Entity which requires a complex constructor. How do you instruct Dapper's `Query` method to use this specific constructor?**
    *Answer:* Dapper automatically supports constructor mapping. If the SQL query returns columns that exactly match the names of the constructor parameters (case-insensitive), Dapper will emit IL to invoke that specific parameterized constructor. If multiple constructors exist, it tries to match the most parameters.
23. **Why is it dangerous to use `Query` with `buffered: false` inside a `using` block for the `IDbConnection`?**
    *Answer:* If you return the `IEnumerable` out of the method, the `using` block will close and dispose the `SqlConnection` before the caller actually iterates over the `IEnumerable`. When the caller tries to `foreach` the result, it will throw an exception because the underlying `SqlDataReader` and Connection are closed.
24. **In high-concurrency systems, what is the impact of not setting `CommandTimeout` on a potentially bad Dapper query?**
    *Answer:* A bad query (e.g., missing an index) might take hours. The default timeout is 30 seconds. If connection pool max pool size is 100, and 100 concurrent requests hit this bad query, the entire application will exhaust the connection pool. Setting strict, shorter timeouts via `CommandDefinition` acts as a bulkhead pattern, failing the request fast and preserving connections for healthy operations.
25. **Explain the execution plan implications if you use `connection.Execute("INSERT...", listOfUsers)` with a list of 100 objects.**
    *Answer:* Dapper will generate the same parameterized SQL statement and execute it 100 separate times in a loop over the same connection. While the SQL execution plan is cached and reused (because it's parameterized), the network latency of 100 separate RPC calls to SQL server is enormous.
26. **How do you map a database column named `has_active_subscription` to a C# property named `IsPremium` using Dapper?**
    *Answer:* Dapper maps by name. You must alias the column in your SQL query: `SELECT has_active_subscription AS IsPremium FROM Users`. (Alternatively, you could use a custom `ColumnAttribute` mapper if using an extension like Dapper.FluentMap, but raw SQL aliasing is the standard Dapper way).
27. **What happens at the ADO.NET level when Dapper executes `QueryFirstOrDefault`?**
    *Answer:* Dapper calls `SqlCommand.ExecuteReader()`. It calls `reader.Read()`. It maps the single row. It then calls `reader.Dispose()`. Closing the reader sends a signal to SQL Server that the rest of the TDS stream is no longer needed. SQL Server stops processing the query, but might still incur a minor overhead depending on the execution plan (e.g., if it had to sort the entire table before streaming).
28. **We have an API endpoint that executes `QueryAsync` returning 50,000 rows. Memory usage spikes to 2GB per request. How do you refactor this using Dapper and ASP.NET Core `IAsyncEnumerable`?**
    *Answer:* We cannot just use `buffered: false` because it returns a synchronous `IEnumerable` which blocks the thread during iteration. Currently, Dapper does not natively support `IAsyncEnumerable`. To truly stream asynchronously, we must drop down to raw ADO.NET `DbDataReader.ReadAsync()`, or use an extension library, or use an Unbuffered query strictly within a background worker that writes to a `Channel<T>`.
29. **How would you use Dapper to securely execute dynamic sorting (ORDER BY) requested by a client API (e.g., `?sort=Price&dir=desc`)?**
    *Answer:* You **cannot** use Dapper parameters (like `@SortColumn`) for `ORDER BY` columns in SQL Server (it will sort by the literal string value, not the column name). You must use a whitelist pattern in C#. Validate the `sort` string against an explicit list of allowed column names, and dynamically concatenate the verified string into the SQL query before passing it to Dapper.
30. **What is `ExecuteReader` and when would you use it instead of `Query`?**
    *Answer:* `ExecuteReader` returns the raw `IDataReader`. You use it when Dapper's IL Emit mapping is insufficient, such as when the result set schema is entirely dynamic and unknown at compile time, or when you need to manually process hierarchical data that changes shape per row.

### Staff Engineer (5)
31. **Analyze the performance difference between using `OUTPUT Inserted.Id` with `ExecuteScalar` versus using a Stored Procedure with an `OUTPUT` parameter for identity generation.**
    *Answer:* `OUTPUT Inserted.Id` via `ExecuteScalar` returns the result as a standard TDS result set containing one row and one column. A Stored Procedure `OUTPUT` parameter returns the value in the RPC return packet. The RPC return packet is slightly more efficient on the network wire as it avoids result-set metadata overhead. However, the difference is measured in nanoseconds; `OUTPUT Inserted` is vastly preferred for code readability and avoiding stored procedure sprawl.
32. **In a Microservices architecture, a Dapper `QueryMultiple` command executes 3 queries. The 2nd query fails with a SqlException. What is the state of the 1st query's data, and how do you handle it?**
    *Answer:* `QueryMultiple` (by default) does not wrap the batch in a transaction. The 1st query executed successfully. The 2nd failed, terminating the batch. To ensure atomicity of the reads (e.g., ensuring you read an Invoice and its LineItems at the exact same point in time), you must wrap the `QueryMultiple` call in a `SqlTransaction` using an isolation level like `Snapshot` or `Serializable`, depending on the consistency requirements.
33. **Explain how Dapper's `CommandFlags.Pipelined` works and its integration with SQL Server.**
    *Answer:* `CommandFlags.Pipelined` is an advanced flag used in highly specific scenarios where you are sending multiple commands asynchronously and want to multiplex them. However, standard SQL Server (via TDS) does not support true command pipelining on a single connection (unlike Redis). If used improperly with SQL Server, it can lead to deadlocks or connection state exceptions. It is generally reserved for custom ADO.NET providers that explicitly support it.
34. **You are tasked with writing a Dapper `TypeHandler` to serialize an entity to JSON for storage in a `NVARCHAR(MAX)` column. What are the severe performance pitfalls of this approach within a `Query<T>`?**
    *Answer:* Dapper's IL mapper is blazing fast (nanoseconds). If your `TypeHandler.Parse` method invokes `System.Text.Json.Deserialize()`, you introduce a massive performance bottleneck. JSON deserialization is orders of magnitude slower than Dapper's native mapping. If the query returns 10,000 rows, the JSON deserializer is invoked 10,000 times, causing massive CPU spikes and GC pressure, completely negating Dapper's performance benefits.
35. **Your Dapper application frequently hits `SqlException (0x80131904): Timeout expired`. The DBAs prove the query takes 50ms in SSMS. Explain the architectural reasons this happens.**
    *Answer:* This is a classic "Connection Pool Starvation" issue. The query takes 50ms, but the application cannot get a connection from the pool to execute it. This happens because connections are being leaked (missing `using` blocks), threads are blocked on synchronous Dapper calls (`Query` instead of `QueryAsync`), or an unbuffered query (`buffered: false`) is being streamed slowly over the network, holding the connection open for minutes.

### Architect (5)
36. **Architect a mechanism to inject Tenant Context (for Row Level Security) into every Dapper query seamlessly across a monolith with 500 repository methods, without altering the repository method signatures.**
    *Answer:* I would create a custom `IDbConnectionFactory`. In ASP.NET Core, this factory injects the `IHttpContextAccessor`. When `CreateConnectionAsync()` is called, the factory instantiates the `SqlConnection`, opens it, extracts the `TenantId` from the HTTP Context (via JWT claims), and uses Dapper to execute `EXEC sp_set_session_context 'TenantId', @Id`. It then returns this prepared connection to the Repository. The repository methods remain untouched, but every Dapper query executed on that connection is now sandboxed by SQL Server RLS.
37. **Evaluate the use of Dapper `Execute` for a CQRS Event Store where you must guarantee absolute ordering of 10,000 events inserted per second.**
    *Answer:* Dapper is merely a transport mechanism. Absolute ordering requires sequence generation (e.g., `SEQUENCE` object in SQL Server) and strict transactional boundaries. For 10k inserts/sec, individual Dapper `Execute` calls will fail due to network chatter. We must architect a bulk insert strategy (e.g., TVP via Dapper) where a batch of events is passed to a stored procedure. The SP uses `sp_getapplock` to serialize access, applies the Sequence numbers, and does a set-based `INSERT`. Dapper's role is strictly marshaling the TVP to the database.
38. **How do you design a Dapper Data Access Layer that supports both SQL Server and PostgreSQL, given that they use different parameter prefixes (`@` vs `:`)?**
    *Answer:* Dapper is technically DB-agnostic, but T-SQL is not. As an Architect, I would not try to write "lowest common denominator" SQL. I would define abstract Repositories (e.g., `IUserRepository`). I would implement a `SqlServerUserRepository` and a `PostgresUserRepository`, each writing highly optimized SQL for their respective engines. The DI container injects the correct implementation based on configuration. Attempting to build an AST (Abstract Syntax Tree) to translate SQL dialects dynamically defeats the purpose of using a Micro ORM.
39. **In a high-availability architecture, your Primary SQL Server fails over to a Secondary replica. How does Dapper handle the connection termination mid-query?**
    *Answer:* Dapper does not handle it; ADO.NET and the application must. When the failover occurs, the open connection is forcefully terminated. Dapper will throw a `SqlException` (e.g., transport-level error). The architecture must include an execution strategy (like Polly) wrapped around the Dapper calls. Polly catches the specific transient exception, applies exponential backoff, and attempts the Dapper call again. By the time it retries, the ADO.NET connection pool will have established a new connection to the promoted Secondary replica.
40. **Justify when you would deliberately choose to use `Query` (buffered) over `Query` (unbuffered: false) for a result set of 50,000 rows.**
    *Answer:* I would choose buffered when the processing of those 50,000 rows involves complex, slow CPU logic or calling external APIs for each row. If I use unbuffered, I hold a lock on the SQL Server connection and consume a session for the entire duration of that slow processing. By buffering, I accept a 50MB memory spike in the application server to instantly free up the SQL Server connection, ensuring database resources are available for other concurrent API requests. Memory in the App Server is generally cheaper and scales horizontally easier than SQL Server connections.

## 8. Exercises

### Easy
1.  **Top 1 Selection:** Create a `Users` table. Write a Dapper query using `QueryFirstOrDefault` to find a user by their Email Address. Ensure your code handles the scenario where the email does not exist gracefully.

### Medium
1.  **Multiple Result Sets:** Create two tables: `Organizations` and `Sites`. Write a single Dapper method that accepts an `OrganizationId`. Use `QueryMultiple` to execute a batch SQL statement that returns the Organization details and a list of all its Sites. Map both and return a combined DTO.

### Hard
1.  **Timeout Exception Handling:** Write a Dapper query that purposefully takes a long time (e.g., use `WAITFOR DELAY '00:00:10'` in SQL Server). Use `CommandDefinition` to set the timeout to 2 seconds. Catch the resulting exception and verify it is a timeout error.

### Enterprise
1.  **Cancellable API Endpoint:** Create an ASP.NET Core Minimal API endpoint that generates a large report. Inject a `CancellationToken` into the endpoint. Pass this token all the way down to a Dapper `CommandDefinition` and execute it. Test the cancellation by initiating the request via Postman and immediately clicking "Cancel Request". Verify in SQL Server Profiler (or Extended Events) that an `Attention` signal was received and the query was aborted.

## 9. Summary

The Dapper API provides surgical tools for data access. `Query` handles standard result sets, while `QuerySingle` and `QueryFirst` provide explicit control over row-count expectations, forcing developers to declare their assumptions about data integrity. `Execute` and `ExecuteScalar` provide lightning-fast data modification and aggregation capabilities.

For the Enterprise Architect, mastering the `CommandDefinition` struct is non-negotiable. It is the gateway to integrating Dapper safely into an asynchronous, highly concurrent web application, providing vital controls over timeouts and cancellation. 

In the next chapter, we will explore the most complex aspect of Dapper: Object Mapping Strategies, including how to handle One-to-Many relationships and complex object graphs without succumbing to the N+1 query problem.
