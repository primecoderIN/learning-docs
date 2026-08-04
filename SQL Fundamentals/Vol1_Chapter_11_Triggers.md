# Volume 1, Chapter 11 – Triggers

## 1. Concept Overview

A **Trigger** is a specialized type of Stored Procedure that executes *automatically* when a specific event occurs in the database. You do not `EXECUTE` a trigger; the database engine fires it invisibly in the background.

Triggers are "event listeners" attached to a specific table (or view). They listen for three specific Data Manipulation Language (DML) events:
1.  `INSERT`
2.  `UPDATE`
3.  `DELETE`

The most common enterprise use case for a trigger is an **Audit Log**. When a user updates their email address, you want the database to automatically save the *old* email address and the *new* email address into an `AuditLogs` table, without relying on the application developer to remember to write a second `INSERT` statement.

## 2. The "Magic Tables" (State Transition)

How does a trigger know what data was just changed? 
When an `UPDATE` happens, the database engine temporarily creates two virtual, in-memory tables that exist *only* for the split-second the trigger is running.

### SQL Server (The `INSERTED` and `DELETED` tables)
*   **`INSERTED` table:** Contains the *new* row(s) coming into the database.
*   **`DELETED` table:** Contains the *old* row(s) that were just overwritten or removed.
*(Note: An `UPDATE` statement is essentially a DELETE followed by an INSERT. Therefore, during an UPDATE, both magic tables are populated).*

### PostgreSQL (The `NEW` and `OLD` records)
Postgres uses a slightly different paradigm. Instead of virtual tables containing multiple rows, Postgres triggers typically execute `FOR EACH ROW`, granting access to the `NEW` and `OLD` records directly.

## 3. AFTER Triggers (SQL Server Syntax)

An `AFTER` trigger fires immediately *after* the modifying statement completes, but *before* the transaction is finalized. 

```sql
-- SQL SERVER SYNTAX: Create an Audit Trigger
CREATE TRIGGER trg_AuditUserEmailUpdate
ON Users
AFTER UPDATE
AS
BEGIN
    -- Check if the Email column was actually modified
    IF UPDATE(Email) 
    BEGIN
        -- Insert the old and new values into the Audit table
        INSERT INTO AuditLogs (UserID, OldEmail, NewEmail, ChangedAt)
        SELECT 
            i.UserID, 
            d.Email AS OldEmail, 
            i.Email AS NewEmail, 
            GETDATE()
        FROM inserted i
        INNER JOIN deleted d ON i.UserID = d.UserID;
    END
END;
GO
```

## 4. BEFORE Triggers (PostgreSQL Syntax)

PostgreSQL natively supports `BEFORE` triggers. These fire *before* the data actually hits the physical table. This is incredibly useful for **Data Validation** or auto-formatting data before it is saved.

```sql
-- POSTGRESQL SYNTAX: Create a formatting trigger
-- Step 1: In Postgres, you must create a function first.
CREATE OR REPLACE FUNCTION format_user_email()
RETURNS TRIGGER AS $$
BEGIN
    -- Force the incoming email to be strictly lowercase
    NEW.email = LOWER(NEW.email);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Step 2: Attach the function to the table as a BEFORE trigger
CREATE TRIGGER trg_FormatEmail
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION format_user_email();
```
*(Notice how we intercepted `NEW.email`, modified it, and then returned it to the database engine to finish the save operation).*

## 5. INSTEAD OF Triggers

An `INSTEAD OF` trigger intercepts the SQL statement and completely stops it from executing. It then runs its own custom logic *instead*.

The most common use case is for **Updatable Views**.
If you have a complex View that joins `Users` and `Tickets`, the database engine will not let you run an `UPDATE` on it. However, you can place an `INSTEAD OF UPDATE` trigger on the View. When the user tries to update the View, the trigger intercepts the data, figures out exactly which underlying table needs to be updated, and executes the physical update manually.

## 6. The Danger of Triggers (Architect's Warning)

Triggers are powerful, but enterprise architects use them sparingly. Why?
1.  **Hidden Logic (Spooky Action at a Distance):** A junior developer writes an `INSERT` statement. It takes 10 seconds to execute. They spend hours trying to figure out why a simple insert is so slow, unaware that a hidden trigger is silently firing in the background and executing a massive cascading update across 5 other tables.
2.  **The RBAR Problem (Row-By-Agonizing-Row):** In Postgres, `FOR EACH ROW` triggers execute 1,000 separate times if you update 1,000 rows. In SQL Server, triggers execute *once per statement*, meaning the `INSERTED` table contains 1,000 rows, forcing developers to write complex `JOIN` logic inside the trigger.
3.  **Debugging Nightmares:** Triggers can trigger other triggers (Nested Triggers). If Table A triggers Table B, which triggers Table C, debugging a failure is incredibly painful.

## 7. Hands-on Exercises

1. Create a table called `PriceHistory` with columns for `TicketID`, `OldPrice`, `NewPrice`, and `ChangeDate`.
2. In SQL Server (or Postgres, adapting the syntax), write an `AFTER UPDATE` trigger on the `Tickets` table.
3. If a ticket's `Price` changes, the trigger should insert a log into the `PriceHistory` table using the `INSERTED`/`DELETED` (or `NEW`/`OLD`) data.
4. Run an `UPDATE` on the `Tickets` table and verify your trigger worked.

## 8. Interview Questions

**Entry Level**
*   **Q:** What is a database trigger?
    *   **A:** A trigger is a specialized stored procedure that executes automatically in response to specific DML events (`INSERT`, `UPDATE`, `DELETE`) on a table or view.
*   **Q:** What are the magic tables used in SQL Server triggers, and what do they contain?
    *   **A:** The `INSERTED` table contains the new data being written. The `DELETED` table contains the old data being removed or overwritten.

**Intermediate Level**
*   **Q:** You want to prevent a user from deleting an Event if the date of the event is in the past. Would you use an AFTER trigger or an INSTEAD OF trigger (in SQL Server)?
    *   **A:** You would use an `INSTEAD OF DELETE` trigger (or a `BEFORE DELETE` trigger in Postgres). By intercepting the action *before* the deletion happens, you can evaluate the date. If the date is in the past, you simply issue a `ROLLBACK` and raise an error, aborting the delete entirely.
*   **Q:** Why are triggers considered a "code smell" in large software architectures?
    *   **A:** They create hidden, side-effect logic that isn't visible in the application's source code. When business rules are split between the application backend and hidden database triggers, it creates a fragmented, difficult-to-maintain system. Modern architectures often prefer using the application layer (or CDC tools like Debezium) for event-handling.

## 9. Preparation for Next Chapter
In Chapter 12, we will cover **Transactions**. We have discussed modifying data with `INSERT` and `UPDATE`, but what happens if a script crashes halfway through? You will learn how to wrap your operations in `BEGIN TRAN` and `COMMIT` to ensure your database never ends up in a corrupted, partially updated state.
