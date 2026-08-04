# Volume 1, Chapter 12 – Transactions (Basics)

## 1. Concept Overview

A database is useless if it cannot guarantee that its data is accurate. 

Imagine a user buys a $100 ticket. Your application must do two things:
1.  Deduct $100 from the user's `Wallet` table.
2.  Insert a new row into the `Tickets` table.

What happens if step 1 succeeds, but the database server crashes exactly one millisecond before step 2 executes? The user lost $100, but didn't get a ticket. Your data is now corrupted.

A **Transaction** solves this. It groups multiple SQL statements together into a single, indivisible unit of work. It operates on an "All or Nothing" principle: either every single statement succeeds, or every single statement fails and is wiped away as if nothing ever happened.

## 2. The ACID Properties

Every modern relational database guarantees four fundamental properties (ACID):
*   **Atomicity:** The "all or nothing" rule. A transaction cannot be partially completed.
*   **Consistency:** The transaction must leave the database in a valid state (e.g., it cannot violate Foreign Key or CHECK constraints).
*   **Isolation:** If two transactions are happening at the exact same time, they should not interfere with each other.
*   **Durability:** Once a transaction is saved (committed), it is permanent. Even if you pull the power cord out of the server 1 second later, the data will still be there when it reboots.

## 3. Basic Syntax (BEGIN, COMMIT, ROLLBACK)

To explicitly control a transaction, you use three commands:

1.  **`BEGIN TRANSACTION` (or `BEGIN`):** Tells the database to stop auto-saving every statement and wait for your command.
2.  **`COMMIT`:** Tells the database everything went perfectly, permanently save the changes to the hard drive.
3.  **`ROLLBACK`:** Tells the database an error occurred, instantly undo all changes made since the `BEGIN` statement.

```sql
-- SQL SERVER SYNTAX
BEGIN TRANSACTION;

-- Step 1: Deduct money
UPDATE Wallets SET Balance = Balance - 100 WHERE UserID = 5;

-- Step 2: Issue ticket
INSERT INTO Tickets (UserID, EventID, Price) VALUES (5, 99, 100.00);

-- If both succeeded without errors:
COMMIT;
-- If an error happened:
-- ROLLBACK;
```

## 4. Error Handling (`TRY...CATCH`)

In the real world, you don't manually highlight `COMMIT` or `ROLLBACK` in your IDE. You write error-handling logic (usually inside a Stored Procedure) to handle it automatically.

### SQL Server Example
```sql
BEGIN TRY
    BEGIN TRANSACTION;
    
    UPDATE Wallets SET Balance = Balance - 100 WHERE UserID = 5;
    INSERT INTO Tickets (UserID, EventID, Price) VALUES (5, 99, 100.00);
    
    -- If no errors occurred, execution reaches here
    COMMIT;
    PRINT 'Ticket purchased successfully.';
END TRY
BEGIN CATCH
    -- If ANY error occurred in the TRY block, execution instantly jumps here
    IF @@TRANCOUNT > 0
    BEGIN
        ROLLBACK;
        PRINT 'An error occurred. The transaction was rolled back.';
    END
END CATCH;
```

### PostgreSQL Example
PostgreSQL handles errors slightly differently inside its PL/pgSQL functions. If an error occurs inside a block, Postgres automatically rolls back that specific block.

## 5. Savepoints

Sometimes, you don't want to roll back the *entire* transaction. You just want to roll back a specific part of it. You can do this by setting a **Savepoint** (a bookmark) halfway through.

```sql
BEGIN TRANSACTION;

INSERT INTO Users (FirstName, Email) VALUES ('Alice', 'alice@test.com');

-- Bookmark this spot
SAVE TRANSACTION FirstUserSaved; 
-- (In Postgres, the syntax is simply: SAVEPOINT FirstUserSaved;)

INSERT INTO Users (FirstName, Email) VALUES ('Bob', 'bob@test.com');

-- Uh oh, Bob's insert failed for some reason! Let's undo Bob, but keep Alice.
ROLLBACK TRANSACTION FirstUserSaved;

-- Permanently save Alice
COMMIT;
```

## 6. Implicit vs Explicit Transactions

*   **Implicit (Auto-Commit):** If you just highlight `UPDATE Users SET Status = 'Active'` and click Execute, the database secretly wraps a `BEGIN` and `COMMIT` around it. It auto-commits immediately.
*   **Explicit:** When you manually type `BEGIN TRANSACTION`, you have taken the steering wheel. The database will hold locks on those rows, blocking all other users from updating them, until you explicitly type `COMMIT` or `ROLLBACK`.
*(Warning: If you type `BEGIN` and go to lunch without committing, you will lock up the entire application. Other users will be frozen waiting for your transaction to finish!)*

## 7. Hands-on Exercises

1. Open a new query window. Write `BEGIN TRANSACTION;` and run an `UPDATE` statement on the `Events` table changing all cities to 'London'. Do NOT commit yet.
2. Open a *second* query window. Try to run a `SELECT * FROM Events` query. Notice how the query just spins and hangs? This is because Window 1 is holding an exclusive lock on the table.
3. Go back to Window 1 and type `ROLLBACK;`. Watch Window 2 instantly unfreeze and return the original, unchanged data.

## 8. Interview Questions

**Entry Level**
*   **Q:** What does ACID stand for?
    *   **A:** Atomicity, Consistency, Isolation, and Durability.
*   **Q:** What happens if you run a `ROLLBACK`?
    *   **A:** The database engine instantly undoes every `INSERT`, `UPDATE`, or `DELETE` that occurred since the `BEGIN TRANSACTION` statement was issued, restoring the database to its exact previous state.

**Intermediate Level**
*   **Q:** What is the danger of leaving a transaction uncommitted?
    *   **A:** While an explicit transaction is open, the database places exclusive locks on the modified rows (and sometimes the entire page or table) to ensure Isolation. If you forget to `COMMIT` or `ROLLBACK`, those locks remain indefinitely, blocking all other queries from accessing that data and eventually causing the application to grind to a halt (blocking/deadlocking).
*   **Q:** Can you roll back a `TRUNCATE TABLE` statement?
    *   **A:** Yes, in both SQL Server and PostgreSQL, `TRUNCATE` is fully transactional. If it is wrapped in a `BEGIN TRANSACTION`, you can roll it back. (Note: This is NOT true in Oracle or MySQL/InnoDB, where DDL statements implicitly commit).

## 9. Preparation for Next Chapter
In Chapter 13, we will step back from syntax and look at **Database Design and Normalization**. You will learn the academic rules (1NF, 2NF, 3NF) for designing tables properly from scratch, and when it is acceptable to break those rules (Denormalization) for the sake of performance.
