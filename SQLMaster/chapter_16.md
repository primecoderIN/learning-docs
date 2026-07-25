# Part 5: Concurrency & Transaction Management

# Chapter 16: Transactions & ACID Properties

## Learning Objectives
By the end of this chapter, you will be able to:
*   Define the ACID properties and understand how SQL Server guarantees them.
*   Write robust explicit transactions in T-SQL using `TRY/CATCH` and `XACT_ABORT`.
*   Control transaction scopes in EF Core using `IDbContextTransaction`.
*   Analyze the architectural shift from Two-Phase Commit (2PC) to the Saga Pattern for distributed transactions in microservices.

---

## 16.1 Introduction to ACID

Up until now, we have treated database operations as single, isolated commands. However, in our EV SaaS, processing a completed charging session requires multiple steps:
1.  Update `core.Sessions` (Set `EndTime` and `TotalKwh`).
2.  Insert into `billing.Invoices` (Generate the bill).
3.  Update `billing.Wallets` (Deduct the customer's balance).

If the server crashes exactly between Step 2 and Step 3, the customer gets an invoice, but their wallet balance is never deducted. This data corruption destroys trust.
To prevent this, relational databases guarantee the **ACID** properties.

### The 4 Pillars of ACID
1.  **A**tomicity: *All or Nothing.* Either all three steps succeed, or none of them do. There is no partial success.
2.  **C**onsistency: *Rule Enforcement.* A transaction must take the database from one valid state to another valid state, enforcing all constraints (Foreign Keys, CHECK constraints) along the way.
3.  **I**solation: *Invisible Work.* If Transaction A is deducting the wallet balance, Transaction B (running concurrently) should not see the half-finished work.
4.  **D**urability: *Permanence.* Once the database says "Committed", the data is safe. Even if someone unplugs the server 1 millisecond later, the data will be there upon reboot (Thanks to Write-Ahead Logging to the LDF file, as discussed in Chapter 1).

---

## 16.2 Explicit vs. Implicit Transactions

By default, every single `INSERT`, `UPDATE`, or `DELETE` statement you run in SQL Server is an **Implicit Transaction**. SQL Server automatically wraps it in a transaction, commits it if it succeeds, and rolls it back if it fails.

To bind our three billing steps into a single unit of work, we must use an **Explicit Transaction**.

```sql
BEGIN TRAN;
-- Do Work
COMMIT TRAN; -- Or ROLLBACK TRAN
```

---

## 16.3 Handling Transactions in T-SQL

A junior developer might write an explicit transaction like this:
```sql
-- DANGEROUS CODE
BEGIN TRAN;
UPDATE core.Sessions SET EndTime = GETUTCDATE() WHERE SessionId = 'S1';
INSERT INTO billing.Invoices (Amount) VALUES (15.00); -- What if this fails?
UPDATE billing.Wallets SET Balance = Balance - 15.00;
COMMIT TRAN;
```
If the `INSERT` fails due to a constraint violation, SQL Server defaults to rolling back *only the statement that failed*, but it **continues executing the rest of the batch**. It will commit the Session update and the Wallet deduction, leaving your data corrupted!

### The Fix: `XACT_ABORT` and `TRY/CATCH`
To write enterprise-grade transactions in T-SQL, you must force SQL Server to abort the entire transaction if *any* error occurs. We do this by turning on `XACT_ABORT`.

```sql
CREATE PROCEDURE billing.usp_ProcessSessionCompletion
    @SessionId UNIQUEIDENTIFIER
AS
BEGIN
    SET NOCOUNT ON;
    -- CRITICAL: Forces full rollback on any error
    SET XACT_ABORT ON; 

    BEGIN TRY
        BEGIN TRAN;
        
        -- Step 1
        UPDATE core.Sessions SET EndTime = GETUTCDATE() WHERE SessionId = @SessionId;
        
        -- Step 2
        INSERT INTO billing.Invoices (SessionId, Amount) VALUES (@SessionId, 15.00);
        
        -- Step 3
        UPDATE billing.Wallets SET Balance = Balance - 15.00 WHERE UserId = 'U1';

        COMMIT TRAN;
    END TRY
    BEGIN CATCH
        -- Only rollback if the transaction is still active
        IF XACT_STATE() <> 0 
        BEGIN
            ROLLBACK TRAN;
        END
        
        -- Bubble the error up to C#
        THROW; 
    END CATCH
END
```

---

## 16.4 The Code: EF Core Transactions

Entity Framework Core simplifies transactions immensely.

### Implicit SaveChanges
When you call `_context.SaveChanges()`, EF Core automatically wraps all the `INSERT/UPDATE/DELETE` statements currently tracked in the context into a single database transaction. If one fails, they all roll back. You get Atomicity for free.

### Explicit Transaction Scopes
What if you need to coordinate multiple `SaveChanges()` calls, or mix EF Core operations with raw SQL via Dapper? You must manually control the transaction scope.

```csharp
public async Task ProcessBillingAsync(Guid sessionId)
{
    // 1. Begin the explicit transaction
    using var transaction = await _context.Database.BeginTransactionAsync();
    
    try
    {
        var session = await _context.Sessions.FindAsync(sessionId);
        session.EndTime = DateTime.UtcNow;
        
        // This save does NOT commit to disk yet
        await _context.SaveChangesAsync(); 

        var invoice = new Invoice { SessionId = sessionId, Amount = 15m };
        _context.Invoices.Add(invoice);
        
        // This save also does NOT commit to disk yet
        await _context.SaveChangesAsync(); 

        // 2. Commit the transaction (Atomicity achieved)
        await transaction.CommitAsync();
    }
    catch (Exception ex)
    {
        // 3. Rollback on failure
        await transaction.RollbackAsync();
        _logger.LogError(ex, "Billing transaction failed.");
        throw;
    }
}
```

---

## 16.5 Architect Perspective: Distributed Transactions

The ACID properties apply to a **single** database.
But what if our EV SaaS is built using Microservices?
*   Service A uses Azure SQL (`core.Sessions`)
*   Service B uses PostgreSQL (`billing.Wallets`)

If you need to update both databases Atomically, you cannot use a simple `BEGIN TRAN`. 
Historically, architectures used **Two-Phase Commit (2PC)** (via MS DTC - Distributed Transaction Coordinator) to lock both databases over the network. 

**The Modern Architect's Rule:** 2PC is dead in the cloud era. It causes massive blocking, network timeouts, and single points of failure.
To achieve atomicity across multiple databases, we abandon strict ACID and embrace **Eventual Consistency** using the **Saga Pattern**.
If the wallet deduction fails in Service B, Service B publishes a "Compensation Event" to a message broker (RabbitMQ/Service Bus), which tells Service A to manually undo the Session completion. We will cover this heavily in Chapter 35.

---

## 16.6 Performance & Security Analysis

### Performance Analysis: Long-Running Transactions
The longer a transaction is open, the longer SQL Server holds locks on the modified rows (and sometimes the entire table). 
*   **Rule:** Transactions must be blazingly fast. Never put an API call, an email send, or a file I/O operation inside a database transaction. Prepare all data first, open the transaction, execute the SQL, and commit immediately.

### Security Implications
*   **Deadlocks (Denial of Service):** Poorly ordered transactions can cause Deadlocks (where Thread A waits for Thread B, and Thread B waits for Thread A). Malicious users can exploit poorly designed transaction scopes to intentionally trigger deadlocks, creating an application-layer DoS attack. We will learn how to resolve deadlocks in Chapter 17.

---

## 16.7 Common Mistakes & Production Pitfalls

1.  **Swallowing the ROLLBACK Exception:** In the EF Core code (Section 16.4), notice the `throw;` statement in the `catch` block. If you omit this, the method will return successfully, and the upstream caller (e.g., the API controller) will return a HTTP 200 OK to the client, even though the data was rolled back. Always rethrow transaction exceptions.
2.  **Forgetting `SET XACT_ABORT ON`:** Without this, legacy stored procedures will partially commit data when encountering constraint errors, leading to "impossible" data states that ruin financial reporting.

---

## 16.8 Production Checklist

*   [ ] Multi-step data modifications in T-SQL are wrapped in `BEGIN TRY / CATCH` blocks.
*   [ ] `SET XACT_ABORT ON` is the first line of every stored procedure performing DML.
*   [ ] EF Core transactions (`IDbContextTransaction`) are disposed via a `using` statement to guarantee rollback if the thread crashes.
*   [ ] No third-party API calls (e.g., Stripe, SendGrid) are placed *inside* an open database transaction scope.

---

## 16.9 Exercises

1.  **Refactoring for Safety:** You find a legacy SQL script:
    ```sql
    BEGIN TRAN
    DELETE FROM core.Ports WHERE StationId = 'S1';
    DELETE FROM core.Stations WHERE StationId = 'S1';
    COMMIT TRAN
    ```
    Rewrite this batch using enterprise-grade safety constructs (`XACT_ABORT` and `TRY/CATCH`).
2.  **Transaction Scoping:** Why is the default behavior of EF Core's `SaveChanges()` sufficient for a standard HTTP request updating a single graph of objects, but insufficient for a background job processing a batch of 10 independent charging sessions?

---

## 16.10 Interview Questions

**Q1: Explain the "D" in ACID and how SQL Server physically guarantees it, even in the event of a sudden power failure.**
*Answer:* The "D" stands for Durability. Once a transaction is committed, it is permanent. SQL Server guarantees this using Write-Ahead Logging (WAL). When you issue a `COMMIT`, the engine does not wait to write the data to the MDF (data) file. Instead, it synchronously writes the transaction record sequentially to the LDF (Transaction Log) file on disk. As soon as the LDF write is confirmed, the client receives success. If the server loses power 1 millisecond later, upon reboot, SQL Server will read the LDF file and "roll forward" the committed transaction into the MDF file.

**Q2: Why is it an anti-pattern to call a third-party REST API (like Stripe to charge a credit card) while a database transaction is currently open?**
*Answer:* A database transaction holds locks on tables and rows. If you call an external REST API, you introduce unpredictable network latency (e.g., the API takes 5 seconds to respond). During those 5 seconds, your database transaction remains open, holding its locks. This blocks all other users trying to read or write to those tables, causing a cascading failure and bringing down the entire application. All external calls must be completed *outside* of the database transaction scope.

---
**Next up in Chapter 17:** We will dive deep into the specific mechanisms SQL Server uses to isolate transactions: Locking, Blocking, and how to read the dreaded Deadlock Graph.
