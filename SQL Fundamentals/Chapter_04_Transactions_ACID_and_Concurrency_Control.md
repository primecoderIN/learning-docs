# Chapter 4 – Transactions, ACID Properties, and Concurrency Control

## 1. Concept Overview

A database is useless if it cannot guarantee the integrity of data when thousands of users access and modify it simultaneously. A **Transaction** is a logical, atomic unit of work that contains one or more SQL statements. The database engine must guarantee that a transaction strictly adheres to the **ACID** properties:

*   **A - Atomicity:** "All or Nothing." If a transaction involves 5 updates, and the server crashes on the 5th, the first 4 are perfectly rolled back.
*   **C - Consistency:** The transaction must transition the database from one valid state to another. Constraints (like Foreign Keys and Check Constraints) must never be violated.
*   **I - Isolation:** Concurrent transactions should not interfere with each other. The degree of isolation (Isolation Levels) dictates what anomalies (Dirty Reads, Non-Repeatable Reads, Phantoms) a transaction might see.
*   **D - Durability:** Once a transaction commits, it is permanently written to non-volatile storage (via the Write-Ahead Log) and will survive an immediate power loss.

**Concurrency Control** is the mechanism the database uses to enforce Isolation. There are two primary schools of thought:
1.  **Pessimistic Concurrency:** Assumes conflicts will happen. Uses physical **Locks** to block other users until a transaction finishes.
2.  **Optimistic Concurrency:** Assumes conflicts are rare. Uses **MVCC (Multi-Version Concurrency Control)**, keeping older versions of data available for readers, so readers never block writers, and writers never block readers.

## 2. History

The formalization of transaction processing was pioneered by Jim Gray (who later won the Turing Award) in the 1970s during the IBM System R project. Gray defined the ACID properties and developed the mathematics for Two-Phase Locking (2PL), which became the industry standard for preventing concurrency anomalies. Decades later, as web traffic demanded higher read scalability without locking, MVCC (originally theorized in 1979) became the dominant model for modern systems like PostgreSQL and Oracle.

## 3. Real-world analogy

Imagine a **Bank Transfer** of $500 from Account A to Account B.
1. The teller checks the balance of Account A.
2. The teller deducts $500 from Account A.
3. The teller adds $500 to Account B.

*   **Atomicity:** If the bank catches fire after step 2, Account A must have the $500 restored. You cannot lose the money in transit.
*   **Isolation (Pessimistic):** While the teller is performing this transfer, the teller places a physical padlock on Account A's file. If an ATM tries to withdraw money simultaneously, the ATM must wait until the teller finishes.
*   **Isolation (Optimistic/MVCC):** The teller makes a photocopy of Account A's file. The ATM can read the photocopy to check the balance, while the teller updates the original. Readers don't wait for writers.

## 4. Business problem solved

Without ACID transactions, enterprise systems face catastrophic logical corruption:
*   **Dirty Reads:** User 1 updates a ticket price to $100 (but hasn't committed). User 2 reads the $100 price and buys the ticket. User 1 rolls back their transaction (price is actually $50). User 2 was just robbed based on ghost data.
*   **Lost Updates / Race Conditions:** Two users see 1 ticket left. Both click "Buy" at the exact same millisecond. Without concurrency control, both transactions succeed, the ticket count becomes -1, and two angry customers show up to the same physical seat.

---

## 5. Microsoft SQL Server explanation

Historically, Microsoft SQL Server was a strictly **Pessimistic** database. It relies heavily on a Lock Manager to govern concurrency.

When a query touches data, the engine acquires locks at varying granularities (Row, Page, Object/Table).
*   **Shared Lock (S):** Acquired when reading data. Other transactions can also read, but no one can write.
*   **Exclusive Lock (X):** Acquired when modifying data. No other transaction can read or write to this data.

If User A holds an Exclusive lock on Row 1, and User B attempts to read Row 1 (requiring a Shared lock), User B will be **blocked**. Their query will sit and spin indefinitely (unless a lock timeout is set).

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Explicit Transaction Control
BEGIN TRANSACTION; -- or BEGIN TRAN
    
    BEGIN TRY
        -- Deduct from user wallet
        UPDATE Core.Wallets SET Balance = Balance - 100 WHERE UserID = 1;
        
        -- Create the ticket
        INSERT INTO Core.Tickets (EventID, UserID, Price) VALUES ('E1', 1, 100);
        
        COMMIT TRANSACTION; -- Persist the changes
    END TRY
    BEGIN CATCH
        -- If anything fails (e.g., Check Constraint violation if Balance < 0)
        ROLLBACK TRANSACTION; 
        -- Log the error...
    END CATCH
GO

-- 2. Changing the Isolation Level
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRAN;
    -- Query logic here...
COMMIT;
```

## 7. SQL Server internals

The heart of SQL Server's ACID guarantee is the **Write-Ahead Log (WAL)** (the `.ldf` file). 
When a transaction commits, the changes to the 8KB data pages are *not* immediately written to the disk (.mdf). That would be too slow. 

Instead:
1. The transaction is synchronously written to the Log File (.ldf) sequentially.
2. The data pages remain modified in RAM (marked as "Dirty Pages").
3. SQL Server tells the application "Commit Successful."
4. Later, a background process called the **Checkpoint** flushes the dirty data pages from RAM to the physical disk.

If the server loses power at step 3, upon reboot, SQL Server runs "Crash Recovery." It reads the Log File, sees a committed transaction that never made it to the data file, and "Rolls Forward" (Redo) the changes into memory. 

## 8. SQL Server execution

**The Deadlock**
1. Transaction A acquires an Exclusive Lock on the `Users` table, and needs an Exclusive Lock on the `Tickets` table to finish.
2. Transaction B acquires an Exclusive Lock on the `Tickets` table, and needs an Exclusive Lock on the `Users` table to finish.
3. Transaction A waits for B. Transaction B waits for A. This is a **Deadlock** (a circular blocking chain).
4. SQL Server has a background "Deadlock Monitor" thread that runs every 5 seconds. It detects the circle, chooses the transaction that is cheapest to rollback (the "Victim"), kills it, and returns Error 1205 to the application. The other transaction succeeds.

## 9. SQL Server enterprise examples

*   **Banking:** Heavy reliance on the `SERIALIZABLE` isolation level for financial transactions to guarantee zero concurrency anomalies, accepting the penalty of severe locking and blocking.
*   **Modern Web Apps:** To compete with Postgres/Oracle, Microsoft introduced **Read Committed Snapshot Isolation (RCSI)**. Enabling this turns SQL Server into an optimistic (MVCC) engine. It stores previous row versions in `TempDB`. Now, writers no longer block readers!

## 10. SQL Server performance considerations

*   **Lock Escalation:** If a transaction updates 5,000 rows in a table, the Lock Manager requires memory to track 5,000 Row Locks. To save memory, SQL Server will dynamically "escalate" the lock into a single, massive **Table Lock**. This instantly freezes out every other user trying to access *any* row in that table. Keeping transactions short and modifying data in batches prevents lock escalation.

## 11. SQL Server security considerations

*   Transactions inherently secure data integrity, preventing malicious actors from manipulating race conditions to generate unauthorized funds or tickets. 

## 12. SQL Server common mistakes

*   **Leaving a Transaction Open:** 
    ```sql
    BEGIN TRAN;
    UPDATE Users SET Email = 'new@email.com' WHERE UserID = 1;
    -- Developer goes to lunch without running COMMIT or ROLLBACK.
    ```
    This leaves an Exclusive lock on the row, and potentially escalates to a Table lock. The entire application hangs for all users until the DBA identifies the Sleeping connection and kills it.

## 13. SQL Server best practices

*   Keep transactions as short as possible. Do not put network calls, API requests, or user prompts inside a `BEGIN TRAN ... COMMIT` block.
*   Always use `BEGIN TRY ... BEGIN CATCH` to ensure errors properly route to a `ROLLBACK`.
*   For web applications, heavily consider turning on RCSI (`ALTER DATABASE DBName SET READ_COMMITTED_SNAPSHOT ON`) to eliminate reader/writer blocking at the cost of `TempDB` overhead.

---

## 14. PostgreSQL explanation

PostgreSQL was designed from the ground up to use **Optimistic Concurrency via MVCC (Multi-Version Concurrency Control)**. 

In Postgres, **Readers never block writers, and writers never block readers.** 
When a transaction updates a row, Postgres does not overwrite the data in place (like SQL Server does). Instead, it inserts a completely new copy of the row into the Heap. Both the old row and the new row exist simultaneously. 

Every transaction is assigned an **XID (Transaction ID)**. Postgres uses these XIDs to determine which version of a row a specific query is allowed to see (Tuple Visibility).

## 15. PostgreSQL syntax

Postgres automatically wraps every single statement in an implicit transaction. If you want to group them, you use explicit blocks:

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. Explicit Transaction Control
BEGIN; -- (or START TRANSACTION)

    -- Deduct from user wallet
    UPDATE core.wallets SET balance = balance - 100 WHERE user_id = 1;
    
    -- Create the ticket
    INSERT INTO core.tickets (event_id, user_id, price) VALUES ('E1', 1, 100);

COMMIT; -- (or ROLLBACK)

-- 2. Changing the Isolation Level
BEGIN ISOLATION LEVEL REPEATABLE READ;
    -- Query logic here...
COMMIT;
```
*(Note: Postgres does not have `BEGIN TRY...CATCH`. Error handling is done via PL/pgSQL `EXCEPTION` blocks, or at the application ORM level).*

## 16. PostgreSQL internals

Every row (Tuple) in Postgres has hidden system columns: `xmin` and `xmax`.
*   **xmin:** The XID of the transaction that inserted the row.
*   **xmax:** The XID of the transaction that deleted/updated the row.

If Transaction 100 (`XID 100`) updates a row, the old row gets an `xmax` of 100. A brand new row is inserted with an `xmin` of 100.
If Transaction 99 runs a `SELECT`, it compares its XID (99) to the rows. It sees the new row (`xmin 100`) and knows it was created in the future, so it ignores it. It sees the old row (`xmax 100`) and knows it is valid for its timeline. It reads the old data without requiring any physical locks!

## 17. PostgreSQL execution

**The Double Booking Race Condition (Select... For Update)**
Because Postgres is optimistic, two users can read the exact same data simultaneously. 
1. User A checks if Ticket 1 is available. (Yes)
2. User B checks if Ticket 1 is available. (Yes)
3. User A updates Ticket 1 to "Sold".
4. User B updates Ticket 1 to "Sold".

To prevent this in Postgres, you must explicitly request a row-level lock during the read phase using `FOR UPDATE`.

```sql
BEGIN;
-- This physically locks the specific row against other writers
SELECT * FROM core.tickets WHERE ticket_id = 1 AND status = 'AVAILABLE' FOR UPDATE;
-- If the row was returned, we own it. Update it.
UPDATE core.tickets SET status = 'SOLD' WHERE ticket_id = 1;
COMMIT;
```
If User B runs that exact `SELECT ... FOR UPDATE` while User A holds the lock, User B's query will pause and wait for User A to commit/rollback.

## 18. PostgreSQL enterprise examples

*   **High-Volume E-Commerce:** Shopify utilizes Postgres's MVCC to allow millions of users to browse catalogs (Read) without ever being blocked by warehouse systems updating inventory counts (Write).
*   **Strict Financial Systems:** Postgres offers **SSI (Serializable Snapshot Isolation)**. It achieves the strictness of `SERIALIZABLE` isolation without physical locking; instead, it monitors for read/write dependencies and artificially aborts transactions if it detects a logical anomaly would occur.

## 19. PostgreSQL performance considerations

*   **Transaction ID Wraparound:** XIDs in Postgres are 32-bit integers (~4 billion). At scale, you will run out of XIDs. Autovacuum must run constantly to "Freeze" old XIDs, replacing them with a special system ID, allowing the 32-bit counter to wrap back to 0. If Autovacuum fails, Postgres will shut down completely to prevent data corruption.
*   **MVCC Bloat:** Every update creates a new row. A table updated 100 times has 100 copies of the row in the Heap. Autovacuum must clean up the 99 "dead tuples".

## 20. PostgreSQL security considerations

*   Explicit transaction blocks prevent incomplete data from being written. In highly secure environments, `FOR UPDATE` locks must be used rigorously to prevent Time-Of-Check to Time-Of-Use (TOCTOU) race condition exploits, a common vulnerability in financial APIs.

## 21. PostgreSQL common mistakes

*   **"Idle in Transaction":** An application opens a connection, executes `BEGIN`, runs a query, and then sits there doing nothing for 5 hours. Because Postgres must keep older versions of rows available for this old transaction, Autovacuum cannot clean up *any* table in the database. Bloat skyrockets, and disk space vanishes. *Fix: Set `idle_in_transaction_session_timeout` in postgresql.conf.*
*   **Assuming Read Committed prevents Race Conditions:** The default isolation level in Postgres is `Read Committed`. It **does not** prevent Lost Updates. You must use `FOR UPDATE` or escalate the isolation level.

## 22. PostgreSQL best practices

*   Always handle deadlocks gracefully in application code. When Postgres detects a deadlock, it kills one transaction. The application should catch this specific exception and transparently retry the transaction.
*   Ensure Autovacuum is tuned aggressively so MVCC dead tuples are reclaimed before they impact disk I/O.

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Default Concurrency** | Pessimistic (Locks) | Optimistic (MVCC) | SQL Server blocks by default. Postgres creates row versions by default. |
| **Update Mechanism** | In-place update | Out-of-place (Insert new row) | SQL Server is cleaner for storage. Postgres generates massive bloat if updated heavily. |
| **Read without blocking**| Requires RCSI enabled | Native (MVCC) | Enabling RCSI in SQL Server makes it behave very similarly to Postgres. |
| **Deadlock handling** | Deadlock Monitor thread kills victim | Deadlock Detector kills victim | Both engines detect deadlocks automatically. Apps *must* be coded to retry. |
| **Transaction IDs (XID)** | Internal LSNs, unbounded | 32-bit, requires wraparound | Postgres requires careful Autovacuum monitoring to prevent wraparound outages. |

## 24. Architect recommendations

**Choosing the Right Isolation Level**
*   **Read Uncommitted (Dirty Reads):** Never use this unless you are doing a massive analytical query on a live OLTP system and you *do not care* if the numbers are off by a few percent.
*   **Read Committed:** The default for both engines. Good for 95% of queries. Protects against reading uncommitted data, but vulnerable to Race Conditions.
*   **Repeatable Read / Serializable:** Use for financial transfers, inventory reservation, and ticket booking. It guarantees strict mathematical correctness at the cost of high concurrency performance and increased deadlock likelihood.

## 25. DBA recommendations

*   **SQL Server DBAs:** Monitor `sys.dm_tran_locks`. If you see `LCK_M_U` or `LCK_M_X` wait types dominating your server, you have severe blocking. Look into index optimization, shorter transactions, or enabling RCSI.
*   **Postgres DBAs:** Monitor `pg_stat_activity` for connections in `state = 'idle in transaction'`. Kill them aggressively using a monitoring script to protect your database from MVCC bloat.

## 26. Developer recommendations

*   **Idempotency:** Design your API endpoints to be idempotent (calling it twice has the same effect as calling it once). If the database kills your transaction due to a deadlock, your backend code should automatically retry the exact same transaction logic without duplicating business side-effects.
*   **Pessimistic Locking in ORMs:** Learn how your ORM translates concurrency. In Entity Framework or Hibernate, learn how to issue a explicit "Lock" (which translates to `WITH (UPDLOCK)` in SQL Server or `FOR UPDATE` in Postgres).

## 27. Production case study

**The NextEvent Phantom Ticket Bug**

*Scenario:* A popular artist's tickets went on sale. The venue had exactly 10,000 seats. Our database recorded exactly 10,000 successful `INSERT` statements into the `Tickets` table. However, the venue reported that 10,050 people showed up with valid tickets.

*RCA:* The application code looked like this:
```javascript
// Node.js backend
let count = await db.query("SELECT COUNT(*) FROM Tickets WHERE EventID = 1");
if (count < 10000) {
    await db.query("INSERT INTO Tickets...");
}
```
At peak load, 200 Node.js threads executed the `SELECT` statement at the exact same millisecond. They all saw `count = 9999`. They all proceeded to the `INSERT` statement. The database gleefully executed all 200 inserts because there were no constraints preventing it. This is a classic Time-Of-Check to Time-Of-Use (TOCTOU) race condition.

*Architectural Fix:* We moved the concurrency control to the database. We created a `VenueCapacity` table. The application now uses an atomic update with a constraint:
```sql
BEGIN;
-- This physically locks the capacity row. 
UPDATE VenueCapacity SET TicketsSold = TicketsSold + 1 
WHERE EventID = 1 AND TicketsSold < MaxCapacity;

-- If the update affected 0 rows, the venue is full. Rollback.
-- If it affected 1 row, insert the ticket.
COMMIT;
```
Because the `UPDATE` inherently takes an Exclusive lock, the 200 concurrent requests are lined up single-file by the database engine. Exactly one succeeds, and 199 fail instantly.

## 28. ASCII diagrams wherever helpful

**PostgreSQL MVCC (Multi-Version Concurrency Control)**

```text
Time T1: Transaction 100 Inserts "Alice".
Time T2: Transaction 101 Updates "Alice" to "Bob".

PHYSICAL HEAP BLOCK (Data on Disk)
+-------------------------------------------------------+
| CTID | xmin | xmax | Data                             |
+-------------------------------------------------------+
| 0,1  | 100  | 101  | "Alice" (Dead to new queries)    |
| 0,2  | 101  | 0    | "Bob"   (Live data)              |
+-------------------------------------------------------+

* Transaction 99 (Started before the update): 
  - Looks at row 0,1. Sees xmax=101. Knows 101 is in the future. Reads "Alice".
* Transaction 102 (Started after the update): 
  - Looks at row 0,1. Sees xmax=101. Knows 101 is in the past. Ignores "Alice".
  - Looks at row 0,2. Sees xmin=101. Knows 101 is valid. Reads "Bob".

Readers (Tx 99) and Writers (Tx 101) operate completely independently without locking!
```

## 29. Enterprise design discussion

**Distributed Transactions (Two-Phase Commit vs. Sagas)**

In microservices architectures, data is often split. Our NextEvent platform might have a `TicketingDB` and a separate `PaymentDB`.
How do we guarantee ACID across two different database servers?

*   **Two-Phase Commit (2PC):** A Distributed Transaction Coordinator (DTC) asks both databases to "Prepare" to commit (Phase 1). They lock the rows. If both say yes, it sends "Commit" (Phase 2). 
    *   *Problem:* Extremely slow. If the network drops between Phase 1 and 2, rows are locked forever. Modern cloud architectures (AWS/Azure) actively discourage or outright ban 2PC.
*   **The Saga Pattern:** The modern enterprise standard. Do not use 2PC. Instead, execute a local transaction on the `PaymentDB`. Send an asynchronous message via Kafka to the `TicketingDB`. If the ticketing fails, the `TicketingDB` sends a message back to the `PaymentDB` to run a **Compensating Transaction** (e.g., refund the money). ACID is relaxed to **BASE** (Basically Available, Soft state, Eventual consistency).

## 30. Hands-on exercises

1. Open two separate query windows in SSMS (or two terminals in psql). This simulates two concurrent users.
2. In Window 1: `BEGIN TRAN; UPDATE Users SET Name = 'Test' WHERE UserID = 1;` (Do not commit yet).
3. In Window 2: `SELECT * FROM Users WHERE UserID = 1;`
4. Observe the behavior. In default SQL Server, Window 2 will hang (blocked). In Postgres, Window 2 will instantly return the old value (MVCC).
5. In Window 1, type `COMMIT;`. Watch Window 2 instantly resolve.

## 31. Coding exercises

1. Write a T-SQL stored procedure that transfers money between two users. It must use explicit transactions, a `TRY/CATCH` block, and verify that the sender's balance does not drop below zero before committing.
2. Write the exact equivalent procedure using Postgres PL/pgSQL. 
3. Modify your Postgres script to use `SELECT ... FOR UPDATE` to lock the sender's row before checking the balance to prevent race conditions.

## 32. Mini project

**Objective:** Implement secure Ticket Purchasing for NextEvent.
We need an API endpoint query that reserves a ticket.
1. Create a `TicketReservations` table.
2. Write a transactional SQL script that:
    a) Finds the first available ticket in `Tickets` for a specific Event.
    b) Locks that row so no one else can grab it.
    c) Updates the ticket status to 'PENDING'.
    d) Inserts a record into `TicketReservations` with a 15-minute expiration timestamp.
    e) Commits.
*(Ensure you use `FOR UPDATE` in Postgres, or `WITH (UPDLOCK, READPAST)` in SQL Server to skip already-locked rows and find the next available one).*

## 33. Quiz

1. What does the "I" in ACID stand for, and what problem does it solve?
2. In SQL Server, what is the role of the Write-Ahead Log (WAL) when a transaction commits?
3. Explain why PostgreSQL requires an "Autovacuum" process as a direct consequence of MVCC.

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is a database transaction?
    *   **A:** A transaction is a single logical unit of work consisting of one or more SQL statements. It must be atomic; either all statements succeed (Commit) or all fail (Rollback), ensuring the database is never left in a partially updated state.
*   **Q:** What is a Deadlock?
    *   **A:** A deadlock occurs when two transactions block each other by holding locks that the other transaction needs to proceed. The database engine must intervene, kill one transaction (the victim), and allow the other to complete.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** How does PostgreSQL handle a reader attempting to query a row that is currently being updated by another transaction?
    *   **A:** By default, Postgres uses MVCC (Multi-Version Concurrency Control). It does not block the reader. Instead, the reader retrieves the older, pre-update version of the row based on its transaction snapshot (XID visibility), ensuring consistent reads without locking.
*   **Q:** What is the difference between a Dirty Read and a Non-Repeatable Read?
    *   **A:** A Dirty Read is reading uncommitted data from another transaction (which might roll back). A Non-Repeatable Read is reading a committed row, then reading it again later in the same transaction, and finding that another committed transaction has changed the values in the meantime.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** In SQL Server, you have a high-concurrency ticketing app. Users frequently hit deadlocks when attempting to book the same seats. You change the isolation level to `SERIALIZABLE` to enforce strictness, but the deadlocks actually *increase*. Why?
    *   **A:** `SERIALIZABLE` isolation requires the database to place Shared (Range) locks on data just to read it, and holds those locks until the end of the transaction. If two users read the same seat (both getting Shared locks), and then both try to update it (requiring Exclusive locks), they instantly deadlock because neither can escalate their Shared lock while the other holds one. The correct fix is pessimistic locking using `SELECT ... WITH (UPDLOCK)` to grab the exclusive lock during the read phase, preventing the second user from reading it entirely.
*   **Q:** Explain "Transaction ID Wraparound" in PostgreSQL and the catastrophic failure it causes.
    *   **A:** Postgres XIDs are 32-bit (max ~4.2 billion). As transactions occur, the XID counter increments. When it hits the limit, it wraps around to 0. Because MVCC uses XID math to determine visibility (e.g., "XID 100 is older than XID 200"), a wraparound means new transactions (XID 1) appear "older" than existing data (XID 4 billion). Suddenly, all data in the database becomes invisible. Autovacuum prevents this by periodically running to "freeze" old tuples (marking them permanently visible), resetting their XID dependency.

## 35. Chapter summary

### Learning Summary
We explored the critical importance of ACID properties in maintaining logical data integrity. We contrasted SQL Server's historical pessimistic locking model (which leads to blocking) with PostgreSQL's modern optimistic MVCC model (which prevents blocking but generates bloat). We examined how race conditions cause massive enterprise failures (like double-booking tickets) and how to explicitly wield `FOR UPDATE` locks and Isolation Levels to force serialized execution when required.

### Key Takeaways
*   ACID guarantees that a database will never save partial, corrupted data in the event of a system crash.
*   SQL Server uses the Write-Ahead Log (WAL) to guarantee Durability while maximizing memory performance.
*   PostgreSQL's MVCC ensures readers do not block writers, using `xmin` and `xmax` to determine row visibility.
*   Application code must *expect* deadlocks and implement automatic retry logic (Idempotency).
*   For distributed microservices architectures, discard Two-Phase Commit in favor of the Saga pattern (Eventual Consistency).

### Glossary
*   **WAL (Write-Ahead Log):** The transaction log where changes are synchronously written before being applied to data pages.
*   **MVCC:** Multi-Version Concurrency Control.
*   **Deadlock:** A circular blocking chain between two or more transactions.
*   **RCSI:** Read Committed Snapshot Isolation (SQL Server's implementation of MVCC).
*   **XID:** Transaction ID (PostgreSQL).

### Common Mistakes
*   Leaving transactions "Idle in Transaction" (kills Postgres via bloat; kills SQL Server via locking).
*   Putting long-running API calls inside a database transaction block.
*   Assuming the database magically handles race conditions without explicit locking strategies (`FOR UPDATE`).

### Best Practices
*   Keep transaction blocks as small and fast as humanly possible.
*   Use `SELECT ... FOR UPDATE` (Postgres) or `WITH (UPDLOCK)` (SQL Server) when reading a row you intend to update milliseconds later, preventing TOCTOU race conditions.
*   Monitor Autovacuum health aggressively in Postgres.

### Further Reading
*   *Transaction Processing: Concepts and Techniques* by Jim Gray.
*   SQL Server documentation on Lock Escalation and RCSI.
*   PostgreSQL documentation on Transaction ID Wraparound and SSI.

### Preparation for Next Chapter
In Chapter 5, we will demystify the **Query Optimizer and Execution Plans**. We will learn exactly how the database engine translates your declarative SQL into a physical roadmap of algorithms (Nested Loops, Hash Matches, Merge Joins). We will learn how to read these Execution Plans, identify missing statistics, and force the optimizer's hand when it makes a terrible decision.
