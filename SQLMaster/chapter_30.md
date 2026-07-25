# Chapter 30: Message Queues & Change Data Capture (CDC)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the catastrophic performance impacts of the "Database as a Queue" anti-pattern (polling).
*   Architect the **Transactional Outbox Pattern** to solve the Dual-Write problem in microservices.
*   Implement Entity Framework Core Interceptors to automatically write to an Outbox table.
*   Understand how **Change Data Capture (CDC)** reads the Transaction Log to asynchronously publish database events to Kafka/Service Bus without blocking the application.

---

## 30.1 The "Database as a Queue" Anti-Pattern

In our EV SaaS, when a charging session completes, we need to send an email receipt to the user.
A junior developer creates a table called `core.EmailQueue` and writes a background worker (C# `BackgroundService`) to process it.

```sql
-- The Polling Anti-Pattern
WHILE (1=1)
BEGIN
    -- Wait 5 seconds
    WAITFOR DELAY '00:00:05'; 

    -- Find the next email to send
    UPDATE TOP (1) core.EmailQueue 
    SET Status = 'Processing'
    OUTPUT INSERTED.*
    WHERE Status = 'Pending';
END
```

**Why this is a disaster:**
1.  **CPU Thrashing:** If the queue is empty, the background worker wakes up every 5 seconds, queries the database, finds nothing, and goes back to sleep. If you scale out to 20 background workers, your database is hit 4 times a second just to return 0 rows.
2.  **Concurrency Hotspots:** All 20 workers are hammering the exact same Clustered Index page, fighting for Exclusive (X) locks on the exact same rows, causing massive blocking and deadlocks.

*Architect Rule:* Relational databases are meant for storage, not messaging. Never poll a SQL table. Use a dedicated message broker (RabbitMQ, Azure Service Bus, Kafka).

---

## 30.2 The Dual-Write Problem

So, we abandon the database queue and switch to RabbitMQ.
When a session completes, the API must:
1. Save the Session to SQL Server.
2. Publish a `SessionCompletedEvent` to RabbitMQ.

```csharp
// DANGEROUS CODE
await _context.SaveChangesAsync(); // Step 1: Write to SQL
await _messageBus.PublishAsync(new SessionCompletedEvent(session.Id)); // Step 2: Write to Broker
```

What if Step 1 succeeds, but the network drops before Step 2 executes? The database is updated, but the email is never sent. 
What if you reverse the order? You send the email, but `SaveChangesAsync` fails due to a foreign key constraint. The user gets an email for a session that doesn't exist.

This is the **Dual-Write Problem**. You cannot guarantee atomicity across two fundamentally different storage systems without Two-Phase Commit (which we banned in Chapter 16).

---

## 30.3 The Transactional Outbox Pattern

To achieve perfect atomicity, we use the **Outbox Pattern**.
Instead of publishing directly to RabbitMQ, the API writes the event to a standard SQL table (`core.OutboxMessages`) *inside the exact same EF Core transaction* as the business data.

Because they are in the same database, they are guaranteed by ACID properties to either both succeed or both fail.

```csharp
public async Task CompleteSessionAsync(Guid sessionId)
{
    var session = await _context.Sessions.FindAsync(sessionId);
    session.Status = "Completed";

    var outboxMessage = new OutboxMessage 
    {
        Id = Guid.NewGuid(),
        Type = "SessionCompleted",
        Payload = JsonSerializer.Serialize(new { SessionId = session.Id })
    };
    _context.OutboxMessages.Add(outboxMessage);

    // Guaranteed Atomicity! Both tables save, or neither saves.
    await _context.SaveChangesAsync(); 
}
```

Now the data is safely on disk. How do we get it from `core.OutboxMessages` into RabbitMQ without polling?

---

## 30.4 Change Data Capture (CDC)

**Change Data Capture (CDC)** is a feature built into SQL Server (and PostgreSQL) that monitors the Transaction Log (LDF).

When EF Core executes the `INSERT INTO core.OutboxMessages`, SQL Server writes that change to the Transaction Log. 
A background CDC process (completely invisible to your C# application) reads that log sequentially and writes a clean history of the changes to a hidden system table.

### The Modern Pipeline: Debezium & Kafka
In enterprise SaaS, we don't even read the CDC tables manually. We deploy a tool like **Debezium**.
1. Debezium connects to SQL Server and listens to the CDC transaction stream in real-time.
2. The instant an Outbox message hits the transaction log, Debezium streams it directly into **Apache Kafka** or **Azure Service Bus**.
3. A separate microservice listens to Kafka, reads the message, and sends the email.

*Result:* Zero polling, zero locks, zero application performance overhead, and mathematically perfect data consistency.

---

## 30.5 The Code: EF Core Outbox Interceptors

Writing `_context.OutboxMessages.Add(...)` in every single controller is tedious and prone to human error (developers will forget to do it).
Architects automate this by hooking into EF Core's `SaveChanges` pipeline using an Interceptor or overriding the `SaveChanges` method.

If you use Domain-Driven Design (DDD), your Entities hold a list of Domain Events in memory. We intercept the save, pop the events off the entities, and serialize them into the Outbox automatically.

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    // 1. Find all entities being tracked that have Domain Events
    var entities = ChangeTracker.Entries<Entity>()
        .Where(e => e.Entity.DomainEvents.Any())
        .Select(e => e.Entity);

    // 2. Extract and serialize the events
    var outboxMessages = entities
        .SelectMany(e => e.DomainEvents)
        .Select(domainEvent => new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = domainEvent.GetType().Name,
            Payload = JsonSerializer.Serialize(domainEvent)
        }).ToList();

    // 3. Add to the DbContext (joins the current transaction!)
    this.Set<OutboxMessage>().AddRange(outboxMessages);

    // 4. Clear the in-memory events so they aren't processed twice
    foreach (var entity in entities) { entity.ClearDomainEvents(); }

    // 5. Commit to the database
    return await base.SaveChangesAsync(cancellationToken);
}
```

---

## 30.6 Performance & Security Analysis

### Performance Analysis: CDC Overhead
While CDC is vastly superior to polling, it is not entirely free. Enabling CDC forces SQL Server to run a background SQL Agent capture job. If you enable CDC on a table handling 10,000 IoT inserts per second, the capture job will consume significant CPU and inflate the Transaction Log (because the log cannot be truncated until the CDC reader processes it). **Only enable CDC on the Outbox table**, never on the high-throughput raw telemetry tables.

### Security Implications
*   **PII in the Outbox:** The Outbox payload is usually JSON. If you serialize an entire User object into the Outbox table, you have just duplicated PII (Personally Identifiable Information) into a plain-text column. This violates GDPR / CCPA if the original User is deleted but the Outbox retains the payload. Always serialize *References* (e.g., `UserId = X`), not sensitive data.

---

## 30.7 Common Mistakes & Production Pitfalls

1.  **Infinite Outbox Growth:** The Outbox table grows endlessly. Because Debezium/Kafka reads the transaction log, it doesn't automatically delete the row from the `OutboxMessages` table. You must implement a background SQL Agent job to `DELETE FROM core.OutboxMessages WHERE CreatedAt < GETUTCDATE() - 7` to keep the table small.
2.  **Order of Operations:** In asynchronous message processing, you cannot guarantee that Message B will arrive after Message A. Your receiving microservice (the email sender) must be **Idempotent**, meaning it can handle receiving the same message twice, or out of order, without crashing or duplicating the email.

---

## 30.8 Production Checklist

*   [ ] "Database as a Queue" polling loops (`WHILE(1=1) SELECT ... WAITFOR DELAY`) are aggressively refactored out of the architecture.
*   [ ] The Dual-Write problem is mitigated by utilizing the Transactional Outbox Pattern to guarantee ACID compliance.
*   [ ] EF Core `SaveChangesAsync` is overridden to automatically serialize in-memory Domain Events into the Outbox table.
*   [ ] CDC (Change Data Capture) or a transactional log tailer (Debezium) is used to asynchronously push Outbox messages to a message broker.

---

## 30.9 Exercises

1.  **Dual-Write Disaster:** A microservice charges a customer's credit card via Stripe (REST API) and then calls `_context.SaveChangesAsync()` to record the payment in SQL Server. Why is this a Dual-Write problem? Which pattern from Chapter 16 and Chapter 30 combined provides the correct architectural solution?
2.  **Outbox Automation:** Look at the code in Section 30.5. If an entity triggers a `StationOfflineEvent`, explain exactly how and when that event is guaranteed to be saved to the database.

---

## 30.10 Interview Questions

**Q1: What is the "Dual-Write" problem in microservice architectures, and how does the Transactional Outbox Pattern solve it?**
*Answer:* The Dual-Write problem occurs when an application must update two disparate systems (e.g., saving to SQL Server and publishing to RabbitMQ) without a distributed transaction. If one succeeds and the network fails before the other executes, the systems become permanently inconsistent. The Outbox Pattern solves this by serializing the message intended for RabbitMQ into a local SQL table (`OutboxMessages`) inside the exact same database transaction as the business data update. Because they share a single SQL transaction, Atomicity guarantees both succeed or both fail.

**Q2: Why is polling a SQL Server table to process queue messages considered an architectural anti-pattern, and what is the modern alternative?**
*Answer:* Polling requires executing `SELECT` queries on a timer (e.g., every 5 seconds). If multiple workers are polling the same table, they cause intense CPU overhead, constant Page Latch contention, and Deadlocks as they fight for Exclusive locks on the exact same index pages. The modern alternative is to use Change Data Capture (CDC) combined with a log-tailing tool like Debezium. This reads the physical Transaction Log sequentially in the background, incurring near-zero locks and pushing events instantly to a true message broker like Kafka.

---
**Next up in Chapter 31:** As our system reaches peak complexity, we must learn how to monitor it. We will explore the most powerful diagnostic tool in SQL Server: **Query Store**, and how to identify CPU hogs without running heavy traces.
