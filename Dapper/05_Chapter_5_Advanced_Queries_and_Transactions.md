# Chapter 5: Advanced Queries, Bulk Operations, and Transactions

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Execute complex Stored Procedures, capturing Output Parameters and Return Values using `DynamicParameters`.
*   Architect ultra-high-performance bulk data ingestion using Table-Valued Parameters (TVPs) and `SqlBulkCopy`, bypassing Dapper's sequential execution limits.
*   Implement highly concurrent `UPSERT` and `MERGE` statements to handle massive telemetry ingestion without race conditions.
*   Master database concurrency by explicitly managing `SqlTransaction` boundaries, Isolation Levels, and `TransactionScope` within a Clean Architecture setup.
*   Design resilient systems capable of automatically recovering from SQL Server deadlocks and transient connectivity failures.

## 2. Introduction

Simple `SELECT` and `INSERT` statements are sufficient for building a minimum viable product. However, as a SaaS platform scales to handle thousands of concurrent users or IoT devices, simple CRUD operations become the primary bottleneck. 

When your platform must ingest 50,000 telemetry readings per second, attempting to execute 50,000 separate `INSERT` statements via Dapper will instantly exhaust your connection pool and cripple SQL Server's CPU. Similarly, when a financial transaction requires updating an Invoice, deducting a Wallet Balance, and writing an Audit Log, executing these outside of an explicitly controlled transactional boundary guarantees eventual data corruption.

This chapter transitions from basic data mapping into true enterprise data engineering. We will explore how to bend Dapper to support Table-Valued Parameters for bulk operations, how to execute complex stored procedures, and how to rigorously enforce ACID (Atomicity, Consistency, Isolation, Durability) properties using ADO.NET transactions.

## 3. Core Concepts

### Table-Valued Parameters (TVPs)
A TVP allows you to send multiple rows of data to a Transact-SQL routine (like a stored procedure) or directly to a parameterized SQL batch as a single parameter. This is the absolute fastest way to insert or update medium-to-large datasets (100 to 50,000 rows) from a .NET application, drastically reducing network round-trips from $O(n)$ to $O(1)$. 

Dapper supports TVPs by allowing you to pass a `DataTable` object as a parameter, provided the database has a matching User-Defined Table Type (UDTT).

### The MERGE Statement (UPSERT)
When IoT devices (like EV Chargers) send state updates, you often don't know if the record already exists in the database. You need an "Insert if not exists, Update if exists" operation (UPSERT). In SQL Server, this is achieved using the `MERGE` statement. Executing this via Dapper requires careful parameterization to avoid lock escalation and deadlocks.

### Transaction Management
Dapper is completely agnostic to transactions. It relies entirely on ADO.NET. If you start a `SqlTransaction` on a `SqlConnection`, ADO.NET dictates that *every single command* executed on that connection must explicitly be assigned that transaction object. Dapper requires you to pass this transaction via the `transaction:` parameter in its extension methods.

## 4. Visual Diagrams

```text
=============================================================================
             BULK INGESTION ARCHITECTURE: TVP vs SEQUENTIAL EXECUTE
=============================================================================

[ ANTI-PATTERN: Dapper connection.Execute(sql, ListOf1000Items) ]
Application ──(TDS)──▶ SQL Server (INSERT Item 1)
Application ◀───────── ACK
Application ──(TDS)──▶ SQL Server (INSERT Item 2)
... repeated 1,000 times (Massive Network Latency & CPU overhead)

[ OPTIMAL: Table-Valued Parameters (TVP) ]
Application Layer:
 1. Convert List<Session> to DataTable
 2. Pass DataTable to Dapper as Parameter
 
Application ──(TDS)──▶ SQL Server
                       │
                       ├── 1 Network RPC Call
                       ├── 1 Execution Plan Compilation
                       └── Set-Based INSERT of 1,000 rows instantly
Application ◀───────── ACK
```

```text
=============================================================================
             CQRS TRANSACTIONAL BOUNDARY (Clean Architecture)
=============================================================================

[ HTTP Request: StartChargingSessionCommand ]
        │
[ MediatR Command Handler ]
        │
        ├── 1. connection.OpenAsync()
        ├── 2. transaction = connection.BeginTransaction(IsolationLevel.ReadCommitted)
        │
        ├── 3. _sessionRepo.CreateAsync(session, transaction)   [DAPPER]
        ├── 4. _chargerRepo.UpdateStatusAsync(id, transaction)  [DAPPER]
        ├── 5. _auditRepo.LogEventAsync(event, transaction)     [DAPPER]
        │
        ├── 6. transaction.CommitAsync()
        │
[ HTTP 200 OK ]
```

## 5. API Deep Dive

### DynamicParameters for Stored Procedures
To execute a stored procedure that uses `OUTPUT` parameters or returns a status code, an anonymous object is insufficient. You must construct a `DynamicParameters` object.

```csharp
var parameters = new DynamicParameters();
// Input
parameters.Add("@TenantId", tenantId, DbType.Guid);
// Output
parameters.Add("@TotalCost", dbType: DbType.Decimal, direction: ParameterDirection.Output, precision: 18, scale: 2);
// Return Value (e.g. RETURN 0 for success, -1 for error)
parameters.Add("@ReturnValue", dbType: DbType.Int32, direction: ParameterDirection.ReturnValue);

await connection.ExecuteAsync("sp_CalculateInvoice", parameters, commandType: CommandType.StoredProcedure);

decimal cost = parameters.Get<decimal>("@TotalCost");
int status = parameters.Get<int>("@ReturnValue");
```

### ICustomQueryParameter for TVPs
To pass a TVP, Dapper provides an extension method on `DataTable` called `AsTableValuedParameter(typeName)`. Internally, this wraps the `DataTable` in a class implementing `SqlMapper.ICustomQueryParameter`, which instructs Dapper to bind the parameter as `SqlDbType.Structured`.

## 6. Complete Examples: EV Charging Platform

### Scenario 1: Bulk Ingestion of Telemetry via TVP
Our platform receives a batched array of 5,000 meter readings from a charger.

**1. SQL Server Schema Setup:**
```sql
CREATE TYPE dbo.udt_MeterReadings AS TABLE (
    ChargerId UNIQUEIDENTIFIER,
    Timestamp DATETIME2,
    EnergyKwh DECIMAL(18,4)
);

CREATE PROCEDURE dbo.sp_InsertMeterReadings
    @Readings dbo.udt_MeterReadings READONLY
AS
BEGIN
    INSERT INTO MeterReadings (ChargerId, Timestamp, EnergyKwh)
    SELECT ChargerId, Timestamp, EnergyKwh FROM @Readings;
END
```

**2. C# Implementation:**
```csharp
public async Task InsertReadingsBulkAsync(IEnumerable<MeterReadingDto> readings)
{
    // 1. Manually construct the DataTable to match the SQL UDT EXACTLY
    var dataTable = new DataTable();
    dataTable.Columns.Add("ChargerId", typeof(Guid));
    dataTable.Columns.Add("Timestamp", typeof(DateTime));
    dataTable.Columns.Add("EnergyKwh", typeof(decimal));

    foreach (var reading in readings)
    {
        dataTable.Rows.Add(reading.ChargerId, reading.Timestamp, reading.EnergyKwh);
    }

    // 2. Pass to Dapper using AsTableValuedParameter
    var parameters = new DynamicParameters();
    parameters.Add("@Readings", dataTable.AsTableValuedParameter("dbo.udt_MeterReadings"));

    // 3. Execute in a single network trip
    await _connection.ExecuteAsync(
        "dbo.sp_InsertMeterReadings", 
        parameters, 
        commandType: CommandType.StoredProcedure);
}
```

### Scenario 2: High-Performance UPSERT (MERGE)
When a charger connects, we want to insert it if it's new, or update its IP address if it exists.

```csharp
public async Task UpsertChargerAsync(Guid chargerId, string ipAddress, Guid siteId)
{
    const string sql = @"
        MERGE INTO Chargers WITH (HOLDLOCK) AS target
        USING (SELECT @Id AS Id, @IpAddress AS IpAddress, @SiteId AS SiteId) AS source
        ON target.Id = source.Id
        WHEN MATCHED THEN
            UPDATE SET IpAddress = source.IpAddress, LastSeen = GETUTCDATE()
        WHEN NOT MATCHED THEN
            INSERT (Id, SiteId, IpAddress, LastSeen)
            VALUES (source.Id, source.SiteId, source.IpAddress, GETUTCDATE());";

    // HOLDLOCK is critical in MERGE to prevent concurrency race conditions
    await _connection.ExecuteAsync(sql, new 
    { 
        Id = chargerId, 
        IpAddress = ipAddress, 
        SiteId = siteId 
    });
}
```

### Scenario 3: Explicit Transaction Management in CQRS
A Command Handler must orchestrate multiple repository calls atomically.

```csharp
public async Task<bool> Handle(StopSessionCommand request, CancellationToken ct)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync(ct);

    // 1. Begin explicit transaction
    await using var transaction = (SqlTransaction)await connection.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

    try
    {
        // 2. Pass transaction to Repositories
        await _sessionRepo.StopSessionAsync(request.SessionId, transaction);
        await _billingRepo.CreateInvoiceAsync(request.SessionId, transaction);

        // 3. Commit if both succeed
        await transaction.CommitAsync(ct);
        return true;
    }
    catch (Exception)
    {
        // 4. Rollback on failure
        await transaction.RollbackAsync(ct);
        throw;
    }
}
```
*Note: In the Repository implementation, you MUST pass the transaction to Dapper:*
`await connection.ExecuteAsync(sql, param, transaction: _transaction);`

## 7. Performance Implications

### TVP vs Sequential Execution
If you pass an `IEnumerable<T>` to `connection.Execute(sql, list)`, Dapper generates a parameterized SQL string and executes it inside a `foreach` loop. 
*   **10,000 rows via Sequential Execute:** ~10,000 network round trips. Execution time: ~15 seconds.
*   **10,000 rows via TVP:** 1 network round trip. Execution time: ~40 milliseconds.
*   **Architect's Rule:** Never use Dapper's native list execution for more than ~50 rows. Always build a TVP.

### Transaction Isolation Levels
Choosing the correct isolation level is crucial for performance.
*   **ReadCommitted (Default):** Good balance. Readers block writers, writers block readers (unless Read Committed Snapshot Isolation is enabled on the DB).
*   **Serializable:** Highest integrity, lowest performance. Use strictly for financial operations where phantom reads are disastrous. It locks entire ranges of indexes, causing massive concurrency bottlenecks.
*   **Snapshot:** Ideal for SaaS. Readers do not block writers. Relies on `tempdb` row versioning. Highly recommended to enable at the database level for Dapper applications to maximize read throughput.

## 8. Common Mistakes

### Beginner
*   **Mistake:** Forgetting to specify `commandType: CommandType.StoredProcedure` when passing a stored procedure name.
*   **Correction:** Without this, SQL Server attempts to execute the procedure name as raw text (e.g., trying to parse the string `"sp_MyProc"` as a SQL command), resulting in a syntax error.
*   **Mistake:** Catching a SQL exception but forgetting to call `transaction.Rollback()`.
*   **Correction:** If an exception occurs and you don't roll back, the connection returns to the pool with a "pending" transaction. The next query that uses this connection will fail abruptly. Always use `try/catch/finally` or `await using` for transactions.

### Intermediate
*   **Mistake:** Passing an opened `SqlTransaction` to Dapper, but trying to execute a query on a *different* `SqlConnection` object.
*   **Correction:** A `SqlTransaction` is irrevocably bound to the specific `SqlConnection` that created it. You must pass both the specific connection and the specific transaction to the repository.
*   **Mistake:** Using `TransactionScope` with `async/await` without specifying `TransactionScopeAsyncFlowOption.Enabled`.
*   **Correction:** If omitted, when the thread resumes after an `await`, the ambient transaction context is lost, and the next Dapper call will execute outside the transaction.

### Senior
*   **Mistake:** Using a SQL `MERGE` statement in a highly concurrent environment without the `WITH (HOLDLOCK)` table hint.
*   **Correction:** `MERGE` is not inherently atomic internally. Under heavy load, two threads might both evaluate `NOT MATCHED` simultaneously and both try to `INSERT`, causing a Primary Key violation constraint error. `HOLDLOCK` (equivalent to Serializable isolation for that specific statement) prevents this race condition.
*   **Mistake:** Defining a TVP `DataTable` with columns in a different order than the SQL Server User-Defined Table Type (UDT).
*   **Correction:** TVP column mapping is strictly positional by default in ADO.NET, not by name. If your C# `DataTable` adds `[Energy, Timestamp]` but SQL expects `[Timestamp, Energy]`, you will get silent data corruption or cast exceptions. The `DataTable` column order must match the SQL UDT perfectly.

### Architect
*   **Mistake:** Wrapping entire API HTTP requests in an ambient `TransactionScope` (e.g., via Middleware) to achieve Unit of Work.
*   **Correction:** This is a devastating anti-pattern in high-throughput SaaS. It opens a transaction immediately, locking database rows while the API might be doing slow non-DB work (like calling a third-party payment gateway). Transactions must be kept as short as possible, strictly wrapping the database calls, utilizing explicit `SqlTransaction` objects injected via MediatR pipeline behaviors or Unit of Work classes.
*   **Mistake:** Failing to implement retry logic for Deadlocks (Error 1205).
*   **Correction:** Deadlocks are not application bugs; they are a natural feature of a concurrent relational database. Dapper does not retry deadlocks. The Architect must implement a Polly `AsyncRetryPolicy` that specifically catches `SqlException` where `Number == 1205`, waits a random few milliseconds, and re-executes the transaction block.

## 9. Interview Questions

### Beginner (10)
1.  **How do you tell Dapper to execute a Stored Procedure?**
    *Answer:* Pass the stored procedure name as the SQL string, and set the `commandType` parameter to `CommandType.StoredProcedure`.
2.  **What class do you use to capture an Output parameter from a Stored Procedure?**
    *Answer:* `DynamicParameters`. You add a parameter and set its direction to `ParameterDirection.Output`.
3.  **If you start a `SqlTransaction`, what must you add to your Dapper `Query` calls?**
    *Answer:* You must explicitly pass the transaction object to the `transaction:` parameter of the Dapper method.
4.  **What does ACID stand for?**
    *Answer:* Atomicity, Consistency, Isolation, Durability.
5.  **What happens if you don't commit a transaction before closing the connection?**
    *Answer:* The transaction is automatically rolled back by ADO.NET/SQL Server to maintain consistency.
6.  **Can I pass a `List<string>` directly as a SQL Server Table-Valued Parameter (TVP)?**
    *Answer:* No. You must project it into a `DataTable` or use a custom `ICustomQueryParameter` wrapper to pass it as `SqlDbType.Structured`.
7.  **What is the purpose of the `MERGE` statement in SQL?**
    *Answer:* It performs an "UPSERT" operation—inserting a row if it doesn't exist, or updating it if it does, all in a single SQL command.
8.  **Will Dapper automatically retry a query if it fails?**
    *Answer:* No. Dapper is a thin wrapper. It immediately bubbles up the `SqlException`. You must implement retry logic yourself.
9.  **What does `SqlMapper.ICustomQueryParameter` do?**
    *Answer:* It is an interface that allows you to define exactly how a complex C# object should be bound to an ADO.NET `IDbDataParameter` (e.g., binding a DataTable as a TVP).
10. **How do you retrieve the "Return Value" (e.g., `RETURN 0`) of a stored procedure using Dapper?**
    *Answer:* Add a parameter using `DynamicParameters` and set `direction: ParameterDirection.ReturnValue`.

### Intermediate (10)
11. **Explain the performance difference between calling Dapper's `Execute` with a `List<T>` of 1,000 items versus using a TVP.**
    *Answer:* `Execute` with a list executes 1,000 distinct `INSERT` statements sequentially over the network (high latency). A TVP marshals all 1,000 items into a single binary stream, requiring only 1 network call and 1 execution plan, resulting in massive performance gains.
12. **What is `TransactionScope` and how does it differ from `SqlTransaction`?**
    *Answer:* `SqlTransaction` is specific to a single database connection. `TransactionScope` is an ambient transaction manager provided by .NET. It can automatically enlist multiple `SqlConnection` objects, and if they point to different databases, it automatically elevates to a Distributed Transaction via the MSDTC (Microsoft Distributed Transaction Coordinator).
13. **Why must you use `TransactionScopeAsyncFlowOption.Enabled` with `TransactionScope`?**
    *Answer:* Because `async/await` causes the execution to jump across different thread-pool threads. Without this option, the ambient transaction context is lost when the thread jumps, causing subsequent queries to execute outside the transaction.
14. **When executing a stored procedure, does Dapper automatically map the result set to a C# object like a standard `Query`?**
    *Answer:* Yes. `connection.Query<T>("sp_Name", commandType: CommandType.StoredProcedure)` works exactly the same as querying a view or table; it maps the output columns to `T`.
15. **What is a SQL Server Deadlock (Error 1205)?**
    *Answer:* A deadlock occurs when Process A holds a lock on Resource 1 and waits for Resource 2, while Process B holds a lock on Resource 2 and waits for Resource 1. SQL Server detects this circular dependency and forcibly kills one of the processes (the victim) to resolve the block.
16. **How should an application handle a Deadlock exception thrown by Dapper?**
    *Answer:* The application should catch the `SqlException`, check if `Number == 1205`, apply a small random delay, and retry the entire transaction block from the beginning.
17. **Why is the column order of a `DataTable` critical when passing it as a TVP?**
    *Answer:* ADO.NET passes TVP data to SQL Server strictly by positional index, not by column name. If the C# `DataTable` column order does not perfectly match the SQL Server `TYPE` column order, data will be inserted into the wrong columns or cause casting errors.
18. **Can you execute multiple TVPs in a single Dapper command?**
    *Answer:* Yes, a stored procedure can accept multiple `READONLY` table parameters, and you can pass multiple `DataTable` objects mapped via `AsTableValuedParameter` in your `DynamicParameters`.
19. **What is the `Snapshot` isolation level?**
    *Answer:* It uses row versioning in `tempdb`. When a transaction reads data, it gets a point-in-time snapshot of the data, meaning readers do not block writers, and writers do not block readers, greatly increasing concurrency.
20. **Why should you keep transactions as short as possible?**
    *Answer:* To minimize the duration of database locks. Long transactions block other concurrent queries, leading to timeouts, connection pool exhaustion, and degraded system throughput.

### Senior (10)
21. **Analyze the architectural flaw of performing an HTTP call to a 3rd party API *inside* a `SqlTransaction` block.**
    *Answer:* Network calls are slow and unpredictable. If the HTTP call takes 5 seconds, the `SqlTransaction` holds database locks open for 5 seconds. In a highly concurrent system, this will cause cascading blocks and deadlocks. Transactions must only wrap the immediate database I/O operations. The HTTP call must happen before the transaction begins, or be managed via a distributed Saga pattern.
22. **You need to UPSERT 50,000 rows. A single `MERGE` statement using a TVP is causing massive lock escalation and blocking the entire table. How do you re-architect this using Dapper?**
    *Answer:* Lock escalation occurs when SQL Server replaces thousands of row locks with a single table lock to save memory. To fix this, I would batch the 50,000 rows into chunks of 2,000. I would execute the TVP `MERGE` via Dapper in a loop, chunk by chunk. This prevents lock escalation, allowing other concurrent processes to access the table while the bulk operation completes.
23. **Explain how to use Dapper to implement the "Transactional Outbox" pattern.**
    *Answer:* In a single transaction via Dapper, you execute an `INSERT/UPDATE` to your domain tables, and an `INSERT` into an `OutboxMessages` table containing a serialized JSON event. You `Commit` the transaction. A separate background worker periodically polls the `OutboxMessages` table via Dapper (using `SELECT ... WITH (UPDLOCK, READPAST)`), publishes the events to a message broker (e.g., RabbitMQ), and marks them as processed. This guarantees reliable event publishing without distributed transactions.
24. **How does Dapper handle parameter sniffing issues in Stored Procedures, and how do you resolve them?**
    *Answer:* Parameter sniffing is a SQL Server engine behavior, not a Dapper issue. SQL Server compiles the execution plan based on the first parameter values passed by Dapper. If those values are atypical, subsequent executions with normal values will use the bad plan and perform terribly. You resolve it in the T-SQL by using `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, or masking parameters inside the stored procedure to force generic plan generation.
25. **When using `TransactionScope`, what happens if the connection to the database drops right after SQL Server commits, but before the ACK reaches the C# application?**
    *Answer:* SQL Server has committed the data durably. However, the .NET application receives a `SqlException` and thinks it failed. If the application has automated retries, it will retry the operation. Therefore, the database operation (and the Dapper queries) *must* be strictly idempotent (e.g., using `UPSERT` or checking for existing unique constraints) to avoid corrupting data upon retry.
26. **You have a complex Domain Aggregate root in C# with several nested collections. You must save it atomically. Justify using EF Core over Dapper for this specific Command Handler.**
    *Answer:* Saving a complex graph with Dapper requires writing manual `INSERT/UPDATE/DELETE` statements for the parent and every child collection, manually tracking which children were removed or added, and orchestrating it all inside a `SqlTransaction`. This is highly error-prone boilerplate. EF Core's Change Tracker automatically detects graph modifications, generates the optimal ordered SQL, and wraps it in a Unit of Work. Use EF Core for complex graph mutations, and Dapper for the flat reads.
27. **What is `SqlBulkCopy` and when should you use it instead of a Dapper TVP?**
    *Answer:* `SqlBulkCopy` is a native ADO.NET class designed specifically for ultra-fast, bulk streaming of data into a single table. It bypasses the standard SQL engine parsing and writes directly to the data pages. Use a TVP (via Dapper) when inserting/updating < 10,000 rows or when you need to trigger complex stored procedure logic on the data. Use `SqlBulkCopy` (bypassing Dapper) when strictly inserting massive datasets (millions of rows) into a single table as fast as possible (e.g., initial data migration or massive telemetry dumps).
28. **Explain the impact of `CommandType.Text` vs `CommandType.StoredProcedure` on SQL Server's Plan Cache.**
    *Answer:* They both utilize the plan cache. However, `CommandType.Text` with parameterized Dapper queries creates an execution plan tied to that exact SQL string. Stored Procedures create a plan tied to the object ID of the procedure. Stored Procedures are generally easier for DBAs to monitor in tools like Query Store, and allow DBAs to update query logic and force plan recompilations without redeploying the C# application.
29. **How do you pass a C# `Enum` to a Stored Procedure integer parameter using Dapper?**
    *Answer:* Dapper automatically casts C# enums to their underlying integer values when mapping to database parameters. You do not need to manually cast it to `(int)MyEnum` unless explicitly required by a `DynamicParameters` type declaration.
30. **In a Multi-Tenant database, how can you ensure a Dapper transaction cannot accidentally modify another tenant's data?**
    *Answer:* At the start of the `SqlTransaction`, execute a Dapper command to set the session context: `EXEC sp_set_session_context 'TenantId', @TenantId`. Configure SQL Server Row-Level Security (RLS) policies on all tables to enforce that `UPDATE/DELETE` operations implicitly filter by `SESSION_CONTEXT(N'TenantId')`. This pushes the security boundary down to the DB engine, preventing application-level Dapper bugs from causing cross-tenant data corruption.

### Staff Engineer (5)
31. **Architect a resilient Distributed Transaction spanning SQL Server (via Dapper) and Azure Service Bus without using MSDTC (which is not supported in cloud environments).**
    *Answer:* MSDTC is legacy. In cloud architectures, we must use the **Saga Pattern** or **Transactional Outbox**. For Outbox: We use Dapper and a local `SqlTransaction` to save the business entity AND an `OutboxEvent` record to SQL Server atomically. A background service polls the Outbox table, publishes the event to Service Bus, and then updates the Outbox record. This guarantees Eventual Consistency and At-Least-Once delivery across heterogeneous systems without requiring distributed locking.
32. **A production system using Dapper TVPs for bulk updates is experiencing severe `PAGELATCH_EX` wait types on SQL Server. What is the root cause and how do you re-architect the C# TVP mapping to fix it?**
    *Answer:* `PAGELATCH_EX` usually indicates a "Last Page Insert Contention" issue, common when thousands of concurrent threads use TVPs to insert rows into a table clustered by a monotonically increasing `IDENTITY` or `DATETIME2` column. All threads fight to lock the exact same physical page at the end of the index. To resolve this, change the clustered index to a `NEWSEQUENTIALID()` or a composite key leading with a Hash/TenantId to distribute the inserts across the B-Tree, alleviating the physical page contention. Dapper's code remains identical, but the DB architecture changes.
33. **Evaluate the performance differences between mapping a 50-column TVP using a manual `DataTable` vs using the reflection-based `SqlMapper.GetCustomQueryParameter` generic extensions found in third-party libraries.**
    *Answer:* Manual `DataTable` population involves creating rows and boxing/unboxing values into an object array. Reflection-based libraries dynamically emit IL to map an `IEnumerable<T>` directly to a `SqlDataRecord` streaming interface. The streaming `SqlDataRecord` approach is vastly superior for memory allocation, avoiding the heavy `DataTable` footprint, reducing Gen 2 GC pressure during massive bulk uploads.
34. **Your CQRS application executes a Dapper query. The DBA kills the SPID (Session ID) in SQL Server mid-execution. Trace the exact sequence of exceptions and connection pool behaviors that follow in the .NET application.**
    *Answer:* 1. SQL Server forcefully closes the TCP socket. 2. The ADO.NET driver detects the broken socket and throws a `SqlException` (Severity 20, Class 11, "A severe error occurred on the current command"). 3. Dapper does not catch this; it bubbles it up to the application's `await` point. 4. The `using` block attempts to call `Dispose()` on the connection. 5. ADO.NET recognizes the connection is broken and removes it from the connection pool entirely (it is not returned for reuse). 6. The application's global exception handler logs the error and returns a 500 response.
35. **Design a strict Dependency Injection architecture for managing `SqlTransaction` boundaries across multiple generic Repositories without leaking `Microsoft.Data.SqlClient` namespaces into the Domain Layer.**
    *Answer:* Define an `IUnitOfWork` interface in the Domain Layer (e.g., `Task CommitAsync()`). In the Infrastructure layer, implement `DapperUnitOfWork` which encapsulates the `SqlConnection` and `SqlTransaction`. The Repositories receive an `IDbSession` interface containing the active connection and transaction. The MediatR Command Handler (Application Layer) requests `IUnitOfWork`, orchestrates the Domain logic via Repositories, and calls `_uow.CommitAsync()`. The Domain remains entirely oblivious to ADO.NET and Dapper.

### Architect (5)
36. **Architect a zero-downtime database migration strategy for a high-traffic SaaS platform where you are renaming a heavily used column queried by dozens of Dapper multi-mapping endpoints.**
    *Answer:* Renaming columns in-place causes instant downtime. We must use the Expand and Contract pattern. 
    1. **Expand DB:** Add the new column (nullable).
    2. **Expand App:** Update Dapper `INSERT/UPDATE` queries to write to *both* the old and new columns. Deploy.
    3. **Backfill:** Run a background script to copy old data to the new column.
    4. **Migrate Reads:** Update Dapper `SELECT` queries and mapping delegates to read exclusively from the new column. Deploy.
    5. **Contract App:** Remove writes to the old column from Dapper `Execute` commands. Deploy.
    6. **Contract DB:** Drop the old column from SQL Server. Zero downtime achieved.
37. **A junior architect suggests replacing all Dapper `Query` calls with Stored Procedures "for security and performance." Defend the architectural decision to retain inline parameterized T-SQL via Dapper.**
    *Answer:* Security: Parameterized inline SQL via Dapper is 100% secure against SQL injection. Performance: SQL Server caches execution plans for parameterized text exactly identically to Stored Procedures; there is no performance difference. Maintainability: Inline SQL keeps the query logic adjacent to the C# mapping logic (Cohesion) and allows the application code to be version-controlled in Git as a single unit. Moving to Stored Procedures introduces a split deployment pipeline, logic fragmentation, and "sprawl" where thousands of dead procedures accumulate in the DB. SPs should be reserved exclusively for complex, multi-step batch processing or DB-level security requirements.
38. **Evaluate the resilience of Dapper in a serverless environment (e.g., Azure Functions consumption plan) connecting to Azure SQL Serverless. What specific configuration and API patterns are mandatory?**
    *Answer:* Serverless environments suffer from compute cold starts, and Azure SQL Serverless can pause, requiring ~30 seconds to resume. Native Dapper calls will fail immediately with timeouts. Mandatory architecture:
    1. **Connection Resiliency:** Implement Polly `AsyncRetryPolicy` wrapping all Dapper calls to handle transient dropped connections and DB resume timeouts.
    2. **Connection Pooling:** In serverless, connection pools are destroyed when instances spin down. Ensure connection strings use `Max Pool Size=100` but rely on infrastructure like Azure SQL Database Proxy to multiplex connections.
    3. **Timeouts:** Use `CommandDefinition` to aggressively extend timeouts specifically for the initial connection attempt.
39. **Design a CQRS pipeline that enforces Idempotency for financial transactions executed via Dapper, ensuring a network retry from the client does not charge their credit card twice.**
    *Answer:* The client generates a unique `IdempotencyKey` (GUID) and sends it in the header. The Command Handler receives it. We create an `IdempotencyRecords` table in SQL Server. We start a `SqlTransaction`. The *very first* Dapper command is: `INSERT INTO IdempotencyRecords (Key, Status) VALUES (@Key, 'Started')`. If this throws a Primary Key violation, we immediately return HTTP 409 Conflict (or the cached original response). If it succeeds, we execute the financial Dapper commands, update the record to 'Completed', and commit the transaction.
40. **How do you architect automated integration testing for complex Dapper TVPs and Stored Procedures without creating flakey, state-dependent tests?**
    *Answer:* I architect a Test Fixture utilizing **Testcontainers** (spinning up a Dockerized SQL Server instance per test run). For state isolation, I use the **Transaction Rollback Pattern**. Before each test, the fixture opens a `SqlConnection` and starts a `SqlTransaction`. The Dapper TVP repository under test is injected with this transaction. The test executes the TVP, uses another Dapper query to assert the data was inserted into the containerized DB, and in the `Dispose` method of the test, it calls `transaction.Rollback()`. The database state is completely reset for the next test instantly, guaranteeing zero flakiness.

## 10. Exercises

### Easy
1.  **Stored Procedure Execution:** Create a Stored Procedure in your LocalDB that accepts `@FirstName` and `@LastName`, and returns the `Id` of a newly inserted record. Call it using Dapper and `CommandType.StoredProcedure`.

### Medium
1.  **Output Parameters:** Modify the previous Stored Procedure to include an `OUTPUT` parameter `@FullName`. Use `DynamicParameters` in Dapper to execute the procedure, capture the output parameter, and print it to the console.

### Hard
1.  **Transaction Rollback:** Create a C# method that starts a `SqlTransaction`. Inside the transaction, use Dapper to execute a valid `INSERT` statement, followed by an intentionally invalid SQL statement (e.g., inserting a string into an INT column). Catch the exception, execute `Rollback()`, and verify in the database that the first valid `INSERT` was successfully reverted.

### Enterprise
1.  **TVP Bulk Ingestion:** Architect the ultimate bulk ingestion pipeline.
    *   Create a SQL User-Defined Table Type matching a `ChargingSession` (Id, ChargerId, StartTime, EndTime, Kwh).
    *   Create a stored procedure that accepts this type.
    *   Write a C# program that generates 10,000 random `ChargingSession` objects.
    *   Use an extension library (like `FastMember`'s `ObjectReader`) to project the `IEnumerable` directly to an `IDataReader` to avoid `DataTable` memory allocation.
    *   Pass the `IDataReader` to Dapper as a TVP. Benchmark the execution time.

## 11. Summary

Dapper is a data access tool, but operating it at an enterprise level requires a deep understanding of database engine mechanics. By utilizing Table-Valued Parameters, you can shatter the network latency bottlenecks associated with sequential looping. By rigorously applying explicit `SqlTransaction` boundaries, you guarantee data integrity in highly concurrent CQRS command handlers. 

The most capable Solution Architects do not treat Dapper as an abstraction; they use it as a high-speed conduit to execute carefully crafted, set-based T-SQL operations. In the final chapters of this book, we will explore how to integrate these Dapper concepts into a pristine ASP.NET Core Clean Architecture, complete with Dependency Injection, Unit Testing, and robust observability.
