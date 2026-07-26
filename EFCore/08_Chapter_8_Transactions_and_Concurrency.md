# Chapter 8: Transactions, Concurrency, and Resilience

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Control database transactions explicitly in EF Core to guarantee atomicity across complex, multi-entity operations.
*   Implement Optimistic Concurrency Control using `RowVersion` (Concurrency Tokens) to prevent data loss in highly concurrent web applications.
*   Handle `DbUpdateConcurrencyException` gracefully, presenting users with actionable resolution strategies.
*   Configure EF Core Execution Strategies to automatically recover from transient cloud network failures and SQL Server deadlocks.
*   Architect the Outbox Pattern to ensure transactional consistency between EF Core database updates and external Message Broker events.

## 2. Introduction

In a single-user prototype, database operations succeed sequentially. In a globally distributed SaaS platform, a thousand users might attempt to modify the same data at the exact same millisecond, while the cloud network connection to the database randomly drops for 50 milliseconds.

If your code is not designed for this reality, two things will happen:
1.  **Lost Updates:** User A overwrites User B's changes silently, causing irreversible data corruption.
2.  **Cascading Failures:** A momentary network blip causes your API to throw unhandled SQL exceptions, failing HTTP requests, and resulting in angry customers.

Entity Framework Core provides robust mechanisms to handle these enterprise realities. It manages implicit transactions automatically, offers explicit transaction boundaries, implements Optimistic Concurrency via tokens, and provides Execution Strategies for connection resilience. 

This chapter transitions your EF Core knowledge from "it works on my machine" to "it survives Black Friday traffic."

## 3. Core Concepts

### Transactions (ACID)
A transaction guarantees that a series of database operations either all succeed (Commit) or all fail (Rollback). EF Core's `SaveChanges()` is inherently transactional. If you add a `Tenant`, a `Site`, and a `Charger`, and the `Charger` fails validation on the database, the `Tenant` and `Site` are automatically rolled back.

### Optimistic vs. Pessimistic Concurrency
*   **Pessimistic Locking:** You lock the database row the moment you read it (`SELECT ... FOR UPDATE`). No one else can read or write it until you finish. This prevents conflicts but severely degrades scalability and causes deadlocks. (EF Core does not support this natively without raw SQL).
*   **Optimistic Concurrency:** You read the row, let the lock go, modify the object in C#, and attempt to save. If someone else modified the row in the meantime, the database rejects your save. This is highly scalable and is the standard for web applications.

### Concurrency Tokens (`RowVersion`)
To implement Optimistic Concurrency, you designate a property (usually a SQL Server `rowversion` / `timestamp`) as a Concurrency Token. When EF Core updates a record, it automatically adds a `WHERE` clause checking if the token still matches what it was when you originally read it.

### Transient Faults and Execution Strategies
Cloud databases (like Azure SQL) will periodically drop connections for micro-seconds due to load balancing, failovers, or throttling. These are "transient" (temporary) faults. An Execution Strategy automatically catches these specific SQL error codes and retries the EF Core operation using an exponential backoff algorithm.

## 4. Visual Diagrams

```text
=============================================================================
             OPTIMISTIC CONCURRENCY CONTROL FLOW
=============================================================================

[ Database: Charger (Id: 1, MaxKw: 50, Version: 0x1) ]

TIME T1: User A requests Charger 1. 
         (C# gets Id=1, MaxKw=50, Version=0x1)
TIME T2: User B requests Charger 1. 
         (C# gets Id=1, MaxKw=50, Version=0x1)

TIME T3: User A changes MaxKw to 100, calls SaveChanges().
         EF SQL: UPDATE Chargers SET MaxKw=100 WHERE Id=1 AND Version=0x1
         (Success. Database Version increments to 0x2).

TIME T4: User B changes MaxKw to 75, calls SaveChanges().
         EF SQL: UPDATE Chargers SET MaxKw=75 WHERE Id=1 AND Version=0x1
         (FAILURE! Zero rows updated. EF Core throws DbUpdateConcurrencyException).
```

```text
=============================================================================
             CONNECTION RESILIENCY (EXECUTION STRATEGY)
=============================================================================

[ Application ]                     [ Azure SQL Server ]
      │
      ├── SaveChangesAsync() ─────────▶ (Network Drops - Error 40613)
      │
[ Execution Strategy pauses 1s ]
      │
      ├── SaveChangesAsync() (Retry 1) ─▶ (Database undergoing failover - Error 40613)
      │
[ Execution Strategy pauses 3s ]
      │
      ├── SaveChangesAsync() (Retry 2) ─▶ SUCCESS! (Rows updated)
      │
      ▼
(Application continues normally, user never sees an error).
```

## 5. API Deep Dive

### 5.1 Explicit Transactions
While `SaveChanges()` creates an implicit transaction, you sometimes need an explicit boundary (e.g., calling `SaveChanges` multiple times, or mixing EF Core with Dapper).

```csharp
using var transaction = await context.Database.BeginTransactionAsync();

try
{
    context.Sites.Add(newSite);
    await context.SaveChangesAsync(); // Does not commit to DB yet!

    // Raw SQL executed within the SAME transaction
    await context.Database.ExecuteSqlRawAsync(
        "UPDATE Inventory SET Count = Count - 1 WHERE PartId = 1");

    await transaction.CommitAsync(); // Actually commits to SQL Server
}
catch (Exception)
{
    await transaction.RollbackAsync(); // Reverts everything
    throw;
}
```

### 5.2 Configuring Concurrency Tokens
In SQL Server, the best concurrency token is a `rowversion` column. It is an 8-byte binary number that SQL Server automatically increments every time the row is updated.

```csharp
public class Charger
{
    public int Id { get; set; }
    public string Status { get; set; }
    
    // The Token
    public byte[] Version { get; set; } 
}

public class ChargerConfiguration : IEntityTypeConfiguration<Charger>
{
    public void Configure(EntityTypeBuilder<Charger> builder)
    {
        builder.Property(c => c.Version)
               .IsRowVersion(); // Configures Optimistic Concurrency!
    }
}
```

### 5.3 Configuring Execution Strategies
Configured during DbContext registration in `Program.cs`.

```csharp
builder.Services.AddDbContext<EvDbContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        // Enables automatic retries for transient SQL errors
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null); // You can add custom SQL Error Codes here
    });
});
```

## 6. EF Core Internals: The Concurrency Check

When you call `SaveChanges` on an entity configured with `IsRowVersion()`, EF Core modifies the T-SQL generation.

If `Charger` was read with `Version = 0x00000000000007D1`:

```sql
-- Normal Update
UPDATE [Chargers] SET [Status] = 'Faulted' WHERE [Id] = 1;

-- Concurrency Update (Generated by EF Core)
UPDATE [Chargers] SET [Status] = 'Faulted' 
WHERE [Id] = 1 AND [Version] = 0x00000000000007D1; -- <--- The crucial check
```

EF Core then inspects the `RowsAffected` returned by ADO.NET.
*   If `RowsAffected == 1`: Success.
*   If `RowsAffected == 0`: EF Core instantly throws a `DbUpdateConcurrencyException`. The entity existed, but the Version changed, meaning someone else modified it before you could.

## 7. Complete Examples: EV Platform Case Study

We are building the "Book a Charging Slot" feature. Two drivers (Driver A and Driver B) see an open slot at 12:00 PM and click "Book" at the exact same time.

Without Concurrency Control, Driver B overwrites Driver A's booking. Driver A shows up and someone is in their spot. Disaster.

### Step 1: Handling the Exception
When Driver B's request fails, we must catch the exception and handle it.

```csharp
public async Task<bool> BookSlotAsync(int slotId, Guid driverId)
{
    var slot = await _context.ChargingSlots.FindAsync(slotId);
    
    if (slot.IsBooked) return false;

    slot.IsBooked = true;
    slot.DriverId = driverId;

    try
    {
        await _context.SaveChangesAsync();
        return true; // Success for Driver A
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // This hits for Driver B.
        // We know the row was modified by someone else.
        
        // Strategy 1: Tell the user it failed (Client Wins/User Resolution)
        _logger.LogWarning("Concurrency conflict for slot {SlotId}", slotId);
        return false; 

        // (Other strategies involve inspecting ex.Entries and merging changes)
    }
}
```

## 8. ASP.NET Core Integration: Execution Strategies and Explicit Transactions

There is a massive architectural gotcha when you combine Explicit Transactions (`BeginTransactionAsync`) with Connection Resiliency (`EnableRetryOnFailure`).

If an execution strategy retries a block of code, and that block contains a manual `BeginTransaction()`, EF Core throws an `InvalidOperationException`. Why? Because the strategy cannot guarantee that the code *before* the transaction block is idempotent and safe to retry.

**The Solution: Manually Invoking the Execution Strategy**

If you have explicit transactions in an application with Resiliency enabled, you *must* wrap the transaction logic in the strategy's `ExecuteAsync` delegate.

```csharp
public async Task ProcessComplexOrder()
{
    // 1. Get the execution strategy configured in Program.cs
    var strategy = _context.Database.CreateExecutionStrategy();

    // 2. Wrap your transaction inside the strategy
    await strategy.ExecuteAsync(async () =>
    {
        // This entire block will be retried if a transient error occurs!
        using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            _context.Orders.Add(new Order());
            await _context.SaveChangesAsync();

            await _context.Database.ExecuteSqlRawAsync("UPDATE...");

            await transaction.CommitAsync();
        }
        catch (Exception)
        {
            await transaction.RollbackAsync();
            throw; // Rethrowing allows the strategy to evaluate if it should retry
        }
    });
}
```

## 9. Clean Architecture Perspective

### Where do transactions belong?
In Clean Architecture, transactions belong strictly in the **Application Layer** (Use Cases / Commands). The Domain Layer should know nothing about databases. The Infrastructure layer implements the repositories, but it should not `Commit` the transaction, because a single Use Case might orchestrate changes across multiple aggregates/repositories.

*Architectural Pattern:* The Unit of Work (`IUnitOfWork`). In EF Core, the `DbContext` *is* the Unit of Work. The Application layer calls repository methods to mutate entities in memory, and finally calls `_unitOfWork.SaveChangesAsync()` once at the very end of the Command Handler.

### The Outbox Pattern (Eventual Consistency)
If your Use Case saves data to EF Core AND publishes an event to a Message Broker (e.g., Azure Service Bus), you have a distributed transaction problem. If `SaveChanges` succeeds, but the network to Azure Service Bus fails, your system is inconsistent.

The Outbox Pattern solves this using a single EF Core transaction:
1.  Mutate Domain Entity (e.g., `Charger.Status = "Faulted"`).
2.  Create an `OutboxMessage` entity containing the serialized event payload.
3.  Add both to the `DbContext`.
4.  Call `SaveChanges()`. (Atomic commit to SQL Server).
5.  A separate background worker polls the `OutboxMessages` table and publishes the messages to the broker, ensuring At-Least-Once delivery.

## 10. Enterprise SaaS Perspective: Cross-Database Transactions

In a Microservices architecture, you might need to update the `BillingDb` and the `EvOperationsDb` simultaneously. 

Historically, .NET relied on `TransactionScope` and the Distributed Transaction Coordinator (MSDTC) to achieve this via Two-Phase Commit (2PC). 
**Architectural Reality:** Cloud databases (like Azure SQL) explicitly *do not support* MSDTC. Distributed transactions across different cloud databases are fundamentally impossible via standard database protocols.

To achieve consistency in a modern Enterprise SaaS, the Architect must abandon ACID distributed transactions and implement the **Saga Pattern** using asynchronous messaging, compensating transactions (Undo commands), and eventual consistency.

## 11. Real Production Case Study

In our EV Platform, we experienced transient failures during peak load. The Azure SQL Database would periodically throttle our connections (Error 10928) for 2 seconds. Because we did not have an Execution Strategy configured, our API instantly returned 500 Internal Server Error to the mobile app.

We enabled `EnableRetryOnFailure()` in `Program.cs`. 
Immediately, 99% of our 500 Errors disappeared. The API simply paused for 1 second and retried successfully. 

However, we introduced a new bug: **Duplicate Invoices**.
The sequence was:
1. API calls `SaveChanges()` to create an Invoice.
2. SQL Server creates the invoice.
3. The network drops exactly as SQL Server sends the "Success" response back to the API.
4. The API assumes failure and throws an exception.
5. The Execution Strategy catches it and retries.
6. SQL Server creates a *second* duplicate invoice.

**The Fix:** Idempotency Keys. 
When creating an entity inside an Execution Strategy, the Application Layer must generate a unique `IdempotencyKey` (Guid) and pass it to the command. The database must have a `UNIQUE CONSTRAINT` on this key. When the strategy retries, the second `SaveChanges` attempts to insert the same key, hits the unique constraint, and we write logic to catch the constraint violation and treat it as a success, acknowledging the first attempt actually worked.

## 12. Common Mistakes

### Beginner
*   **Mistake:** Calling `SaveChanges()` inside a `foreach` loop.
*   **Correction:** `SaveChanges()` initiates a database transaction and network round trip. Calling it 100 times in a loop is agonizingly slow. Modify all entities in the loop, and call `SaveChanges()` *once* outside the loop to commit them all atomically in a single transaction.

### Intermediate
*   **Mistake:** Catching `DbUpdateConcurrencyException`, logging it, and pretending it succeeded.
*   **Correction:** If this exception is thrown, the data in the database *was not updated*. Your in-memory entity is now out of sync with the database. You must either return a failure to the user, or reload the entity from the database, re-apply the changes, and try again.

### Senior
*   **Mistake:** Relying on `[ConcurrencyCheck]` (which checks a specific column, like `LastName`) instead of a dedicated `[Timestamp]/RowVersion` column.
*   **Correction:** If you use `[ConcurrencyCheck]` on `LastName`, and two users update the *same* record, but one updates `Email` and the other updates `Phone`, EF Core will *not* throw a concurrency exception because `LastName` didn't change. A `RowVersion` token guarantees any modification to the row increments the token, providing absolute optimistic concurrency.

### Architect
*   **Mistake:** Utilizing `TransactionScope` to span transactions across multiple `DbContext` instances pointing to different databases.
*   **Correction:** This works locally because Windows initiates an MSDTC transaction. When deployed to Azure App Service and Azure SQL, it will instantly crash. The Architect must design the system to use eventual consistency (Sagas/Outbox) instead of distributed ACID transactions.

## 13. Interview Questions

### Beginner (10)
1.  **What does ACID stand for?**
    *Answer:* Atomicity, Consistency, Isolation, Durability.
2.  **Does `SaveChanges()` automatically use a transaction?**
    *Answer:* Yes. All changes tracked by the DbContext are saved in a single, atomic transaction.
3.  **What happens if one entity fails to save during `SaveChanges()`?**
    *Answer:* The entire transaction is rolled back. No data is saved to the database.
4.  **What is Optimistic Concurrency?**
    *Answer:* A strategy that assumes conflicts are rare. It reads data, allows modifications, and checks for conflicts only at the exact moment of saving, rejecting the save if the underlying data changed.
5.  **What exception does EF Core throw when an optimistic concurrency conflict occurs?**
    *Answer:* `DbUpdateConcurrencyException`.
6.  **What is a Concurrency Token?**
    *Answer:* A property (like a RowVersion) configured so that EF Core checks its value during an `UPDATE` or `DELETE` to ensure the row hasn't been modified by another process.
7.  **What is an Execution Strategy?**
    *Answer:* A mechanism in EF Core that automatically retries database operations when transient errors (like temporary network drops) occur.
8.  **How do you explicitly start a transaction in EF Core?**
    *Answer:* `await context.Database.BeginTransactionAsync()`.
9.  **Why would you use an explicit transaction?**
    *Answer:* To group multiple `SaveChanges` calls, or to group a `SaveChanges` call with a raw SQL execution (`ExecuteSqlRaw`) into a single atomic unit.
10. **What must you call to finalize an explicit transaction?**
    *Answer:* `await transaction.CommitAsync()`.

### Intermediate (10)
11. **Explain the difference between `[ConcurrencyCheck]` and `[Timestamp] / IsRowVersion()`.**
    *Answer:* `[ConcurrencyCheck]` adds a specific property's original value to the `WHERE` clause. `IsRowVersion` maps to a database-generated value (like SQL Server `rowversion`) that automatically increments on *any* row change, providing foolproof concurrency tracking.
12. **How does SQL Server internally handle a `rowversion` column?**
    *Answer:* It is an 8-byte binary number that is guaranteed to be unique within the database and automatically increments every time a row is inserted or updated.
13. **You have enabled `EnableRetryOnFailure`. Your code calls `BeginTransaction()`. What happens at runtime?**
    *Answer:* EF Core throws an `InvalidOperationException` stating that explicit transactions must be wrapped in `IExecutionStrategy.ExecuteAsync`.
14. **How do you resolve a concurrency conflict (Store Wins)?**
    *Answer:* Catch the `DbUpdateConcurrencyException`, call `ex.Entries.Single().Reload()`, and tell the user they must retry because the data changed.
15. **How do you resolve a concurrency conflict (Client Wins)?**
    *Answer:* Catch the exception. Access `entry.OriginalValues`. Set the original values to match the database's current values (tricking EF Core into thinking it has the latest version), and call `SaveChanges` again to overwrite the database.
16. **What is a "Transient Fault"?**
    *Answer:* A temporary error, usually network-related, that is highly likely to succeed if the exact same operation is retried a few seconds later.
17. **Can you specify custom SQL Error numbers for the Execution Strategy to retry?**
    *Answer:* Yes, you can pass an array of error numbers to `errorNumbersToAdd` in `EnableRetryOnFailure`.
18. **What isolation level does an EF Core explicit transaction use by default?**
    *Answer:* It uses the default isolation level of the underlying database provider (usually `READ COMMITTED` for SQL Server).
19. **How do you specify a different isolation level (e.g., Serializable)?**
    *Answer:* `await context.Database.BeginTransactionAsync(IsolationLevel.Serializable)`.
20. **Why should you avoid `IsolationLevel.Serializable` in high-throughput web apps?**
    *Answer:* It places severe range locks on tables, preventing other users from reading or writing, drastically reducing concurrency and leading to massive deadlocks.

### Senior (10)
21. **Analyze the performance overhead of Optimistic Concurrency vs. Pessimistic Concurrency.**
    *Answer:* Optimistic concurrency has almost zero overhead during the read phase and a tiny overhead during the update phase (checking the token). Pessimistic concurrency requires issuing lock commands (`UPDLOCK`, `ROWLOCK`) on the database server during the read phase, blocking other threads, consuming server resources, and risking deadlocks. Optimistic is vastly superior for web-scale applications.
22. **Architect the Outbox Pattern using EF Core.**
    *Answer:* Define an `OutboxMessage` entity. In the application command handler, modify the business entities and create an `OutboxMessage` containing the serialized event. Call `_context.SaveChangesAsync()`. Both the state change and the message are committed atomically. A background worker (e.g., Hangfire) queries the `OutboxMessage` table, publishes to the message broker, and marks the message as processed.
23. **You implement Connection Resiliency. During a retry, a duplicate record is created. Explain how this happens and how to prevent it architecturally.**
    *Answer:* It happens when the initial `INSERT` succeeds on SQL Server, but the network drops before the acknowledgement reaches the API. The API assumes failure and retries, creating a duplicate. To prevent this, the Architect must implement Idempotency Keys (a unique constraint in SQL Server). The retry will attempt the same `INSERT`, hit the constraint, and the application logic must interpret the constraint violation as a successful operation.
24. **How do you implement Pessimistic Locking in EF Core, given there is no native API for it?**
    *Answer:* You must drop down to raw SQL. For SQL Server: `await context.Chargers.FromSqlRaw("SELECT * FROM Chargers WITH (UPDLOCK, ROWLOCK) WHERE Id = 1").FirstOrDefaultAsync()`. This locks the row until the explicit EF Core transaction commits or rolls back.
25. **Explain "Read Skew" and how explicit transactions handle it.**
    *Answer:* Read Skew occurs when you query Table A, then query Table B, but another transaction modified Table B in the middle. Your memory now holds an inconsistent state. To prevent this, wrap both reads in an explicit transaction using `IsolationLevel.Snapshot` (if enabled in SQL Server), ensuring both reads represent the database at the exact same point in time.
26. **What is a Deadlock in SQL Server, and how does the EF Core Execution Strategy handle it?**
    *Answer:* A deadlock occurs when Transaction A locks Row 1 and needs Row 2, while Transaction B locks Row 2 and needs Row 1. SQL Server detects this and kills one transaction (Error 1205). The EF Core Execution Strategy recognizes Error 1205 as a transient fault and will automatically back off and retry the killed transaction.
27. **Evaluate the use of `TransactionScope` in .NET 8 with EF Core.**
    *Answer:* `TransactionScope` implicitly wraps operations in a transaction. It is useful for coordinating multiple disparate connections (e.g., EF Core and ADO.NET) against the *same* database. However, if it spans multiple physical databases, it requires MSDTC. Because Azure SQL and modern cloud databases do not support MSDTC, `TransactionScope` is a dangerous architectural choice that will fail in cloud deployments.
28. **How does `ExecuteUpdate` (EF7+) interact with Concurrency Tokens?**
    *Answer:* `ExecuteUpdate` bypasses the Change Tracker. Therefore, it *does not* automatically check Concurrency Tokens. If you want optimistic concurrency with `ExecuteUpdate`, you must explicitly add the token check to the `Where` clause: `context.Chargers.Where(c => c.Id == 1 && c.Version == expectedVersion).ExecuteUpdate(...)` and manually check if `RowsAffected == 0`.
29. **Design a solution to update a massive aggregate root with a complex graph (e.g., an Order with 100 OrderLines) securely using Optimistic Concurrency.**
    *Answer:* Configure the `Order` root with a `RowVersion`. When modifying an `OrderLine`, you must manually increment or touch the `Order` root's version. Otherwise, two users could modify different `OrderLines` concurrently without conflict, violating the aggregate's business rules. EF Core provides this capability by modifying the state of the parent entity to `Modified` even if only children changed.
30. **Explain how EF Core handles nested transactions using `BeginTransaction`.**
    *Answer:* EF Core does not support true nested database transactions. If you call `BeginTransaction` while another transaction is already active on the context, EF Core will throw an `InvalidOperationException`. You must architect your code to pass the existing `IDbContextTransaction` down the call stack or rely entirely on ambient implicit `SaveChanges` behavior.

### Staff Engineer (5)
31. **Architect a globally distributed Saga across three Microservices (Orders, Billing, Shipping) where each service has its own dedicated database and DbContext. ACID distributed transactions are prohibited. Detail the failure compensation mechanism.**
    *Answer:* The Architect implements Choreography or Orchestration (via MassTransit/NServiceBus). 
    1. OrderService saves Order (Pending) via EF Core and publishes `OrderCreatedEvent` via Outbox.
    2. BillingService receives event, saves Invoice via EF Core, publishes `BilledEvent` via Outbox.
    3. ShippingService receives event, attempts to reserve inventory. Inventory is empty. ShippingService saves Failure state and publishes `ShippingFailedEvent`.
    4. OrderService and BillingService receive `ShippingFailedEvent`. They execute *Compensating Transactions* (Undo Logic): Billing issues a refund (saving to DbContext). OrderService marks Order as Cancelled (saving to DbContext). Eventual consistency is maintained without a single distributed database lock.
32. **A legacy, high-volume trading system requires absolute monotonic incrementing of a `Sequence` number across thousands of concurrent transactions, but using an `IDENTITY` column causes unacceptable insert-page latch contention (PAGELATCH_EX). Architect a solution using EF Core and SQL Server primitives.**
    *Answer:* The Architect bypasses EF Core identity mechanisms entirely. They implement a SQL Server `SEQUENCE` object with a large `CACHE` size (e.g., `CACHE 1000`). In EF Core, they configure the property mapping: `builder.Property(e => e.TradeId).HasDefaultValueSql("NEXT VALUE FOR TradeSequence")`. This pushes the high-performance, lock-free monotonic allocation entirely into the SQL Server memory cache, eliminating the physical page latch contention while maintaining sequential integrity.
33. **Analyze the systemic risk of using optimistic concurrency (RowVersion) on a highly contented row (e.g., a central `GlobalCounters` table) in a system doing 5,000 requests per second.**
    *Answer:* Optimistic concurrency on a single hot row guarantees high latency and massive failure rates. Every request reads the row, but only one can win the update. The other 4,999 requests throw `DbUpdateConcurrencyException`, retry, and compete again, creating a retry-storm that consumes the entire application Thread Pool and CPU. The Architect must fundamentally redesign the system to avoid central state bottlenecks (e.g., using Redis Atomic increments, or SQL Server `UPDATE SET Counter = Counter + 1` without concurrency checks).
34. **Design a resilient architecture where an Azure Function uses EF Core to process messages from a Service Bus Queue, ensuring exactly-once processing despite transient database failures and message peek-lock timeouts.**
    *Answer:* The Function must use the Azure Service Bus Trigger, which handles peek-locks. Inside the function, initialize the EF Core Execution Strategy. Wrap the database processing in `strategy.ExecuteAsync`. Crucially, the operation must be Idempotent using the `MessageId` as the unique key in a `ProcessedMessages` SQL table. If the database fails, the strategy retries. If the strategy exhausts retries, the Function throws, and Service Bus abandons the message (eventually moving to a Dead Letter Queue). If the database succeeds but the Function crashes before completing the Service Bus message, the message reappears. The next execution sees the `MessageId` in the `ProcessedMessages` table and safely ignores it, guaranteeing exactly-once processing.
35. **Evaluate the interaction between EF Core's Connection Resiliency, ADO.NET Connection Pooling, and Azure SQL's dynamic port assignments during a backend replica failover.**
    *Answer:* When Azure SQL fails over, the primary node drops. The ADO.NET connection pool is now poisoned with dead connections. When EF Core requests a connection, ADO.NET throws a transport-level error (e.g., 40613). The EF Core Execution Strategy catches this. Simultaneously, ADO.NET realizes the connection is dead and clears the pool. During the backoff period (e.g., 3 seconds), Azure SQL DNS updates to the new replica node. The strategy's subsequent retry forces ADO.NET to establish a fresh physical connection to the new node, successfully resuming operations without app downtime.

## 15. Exercises

### Easy
1.  **Implicit Transactions:** Write a method that adds two entities to the `DbContext` and calls `SaveChanges`. Throw an exception immediately after `SaveChanges`. Verify in the database that both entities were saved (because they were part of the same transaction before the exception was thrown).

### Medium
1.  **Explicit Transactions:** Rewrite the previous exercise, but wrap the `Add` and `SaveChanges` calls in an explicit transaction using `await context.Database.BeginTransactionAsync()`. Throw an exception *before* calling `CommitAsync()`. Verify in the database that neither entity was saved.
2.  **Concurrency Setup:** Add a `byte[] Version` property to a `Site` entity. Use the Fluent API to configure it as `IsRowVersion()`. Generate and apply a migration. Verify the column is created in SQL Server as `rowversion`.

### Hard
1.  **Simulating Concurrency:** Write a test method that fetches a `Site` (Entity A) into memory. Then, open a *new* separate `DbContext` instance, fetch the same `Site` (Entity B), modify Entity B, and save it. Finally, modify Entity A and try to save it. Catch the resulting `DbUpdateConcurrencyException`.

### Enterprise
1.  **The Outbox Pattern:** Create a `DomainEvent` entity. Write a complex command handler that creates a `Tenant`, creates a `Site`, and creates a `DomainEvent` (representing "TenantCreated") all within a single `SaveChangesAsync()` call. This guarantees that your business data and your outbound message queue are strictly consistent.

## 16. Production Checklist

- [ ] Are all entities that can be modified concurrently by multiple users protected with a `[Timestamp]` or `.IsRowVersion()` property?
- [ ] Is `EnableRetryOnFailure()` configured in the DbContext options to survive transient cloud network drops?
- [ ] Are explicit transactions (`BeginTransaction`) wrapped inside `ExecutionStrategy.ExecuteAsync()` blocks to prevent runtime exceptions?
- [ ] Is the Outbox Pattern utilized for publishing messages to external brokers to ensure distributed consistency?
- [ ] Are bulk operations that circumvent the Change Tracker (`ExecuteUpdate`) manually designed to be idempotent?

## 17. Summary

An Enterprise application is defined by how it handles failure. By implementing Execution Strategies, our EF Core pipeline becomes resilient to the chaotic reality of cloud networking. By mastering explicit transactions and Optimistic Concurrency, we guarantee absolute data integrity, ensuring that a thousand concurrent users cannot corrupt our database state.

We have mastered modeling, querying, performance, and resilience. In the final phase of this textbook, we look toward the future. In the next chapter, we will explore Advanced Architectures, mastering multi-tenancy, CQRS, and the revolutionary integration of EF Core with JSON documents and NoSQL patterns inside SQL Server.
