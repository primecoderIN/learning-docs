# Volume 1, Chapter 10 – Stored Procedures and Functions

## 1. Concept Overview

Up to this point, we have written ad-hoc SQL queries. In a real-world enterprise application, you rarely send raw SQL strings from your C#, Python, or Java backend directly to the database. Instead, you encapsulate your SQL logic inside the database itself using **Stored Procedures** and **User-Defined Functions (UDFs)**.

This provides massive architectural benefits:
*   **Security:** You prevent SQL Injection by forcing the application to pass parameters rather than raw strings. You can also grant a user permission to execute a procedure without giving them permission to view the underlying tables.
*   **Network Traffic:** Instead of sending a 50-line SQL query across the network, the application sends a 1-line command: `EXEC GetVIPUsers`.
*   **Reusability:** Multiple applications (e.g., the Web App and the Mobile App) can call the exact same centralized database logic.

## 2. Stored Procedures

A Stored Procedure (often called an "SP" or "Sproc") is a pre-compiled collection of SQL statements saved under a specific name. It can take input parameters, perform complex logic (including `IF/ELSE` statements and loops), modify data (`INSERT`, `UPDATE`), and return multiple result sets.

### Creating and Executing an SP (SQL Server Syntax)
```sql
-- Connect to NextEventDB

-- 1. Create the Stored Procedure
CREATE PROCEDURE GetTicketsByEvent
    @EventID INT,                -- Input Parameter
    @MinimumPrice DECIMAL(10,2)  -- Input Parameter
AS
BEGIN
    -- Best practice: stops the '1 row affected' network messages
    SET NOCOUNT ON; 

    SELECT t.TicketID, u.FirstName, t.Price
    FROM Tickets t
    INNER JOIN Users u ON t.UserID = u.UserID
    WHERE t.EventID = @EventID 
      AND t.Price >= @MinimumPrice;
END;
GO

-- 2. Execute the Stored Procedure
EXEC GetTicketsByEvent @EventID = 5, @MinimumPrice = 100.00;
```

## 3. User-Defined Functions (UDFs)

A User-Defined Function is similar to a stored procedure, but it has strict limitations. 
Functions are primarily designed to compute and return a value that can be used directly inside a `SELECT` statement. 

**The Golden Rule of Functions:** Functions CANNOT modify data. You cannot put an `INSERT`, `UPDATE`, or `DELETE` statement inside a function. They must be purely read-only.

### Scalar Functions (Returns a single value)
```sql
-- Create a function to format a name
CREATE FUNCTION fn_FormatName (@FirstName VARCHAR(50), @LastName VARCHAR(50))
RETURNS VARCHAR(100)
AS
BEGIN
    RETURN UPPER(@LastName) + ', ' + @FirstName;
END;
GO

-- Using the function inline within a query
SELECT UserID, dbo.fn_FormatName(FirstName, LastName) AS FormattedName 
FROM Users;
```

### Table-Valued Functions (Returns a virtual table)
```sql
-- Create a function that returns a table of active events
CREATE FUNCTION fn_GetActiveEvents (@MinCapacity INT)
RETURNS TABLE
AS
RETURN (
    SELECT EventID, Title, Capacity 
    FROM Events 
    WHERE Capacity >= @MinCapacity AND Date > GETDATE()
);
GO

-- Using the function as if it were a physical table
SELECT * FROM fn_GetActiveEvents(5000);
```

## 4. Stored Procedures vs. Functions

This is a guaranteed interview question. You must know the difference.

| Feature | Stored Procedure | User-Defined Function (UDF) |
| :--- | :--- | :--- |
| **Primary Use Case** | Business logic, Modifying data (CRUD) | Calculations, Formatting data |
| **Can Modify Data?** | Yes (`INSERT`/`UPDATE`/`DELETE`) | No (Read-Only) |
| **How to Call It** | `EXEC procedure_name` | Inside a query: `SELECT fn_name()` |
| **Return Values** | Can return multiple result sets, or just an INT status code | Must return exactly ONE value (Scalar) or ONE Table |
| **Can call the other?**| A Procedure CAN call a Function | A Function CANNOT call a Procedure |

## 5. PostgreSQL Nuances (PL/pgSQL)

In PostgreSQL, the lines between Functions and Procedures were historically blurred. Before PG 11, Postgres only had Functions (which *could* modify data, unlike SQL Server). Postgres 11 finally introduced true Stored Procedures that support transaction control (`COMMIT`/`ROLLBACK` inside the procedure).

PostgreSQL uses a powerful procedural language called **PL/pgSQL**.

```sql
-- PostgreSQL Syntax for a Function (Returns a value)
CREATE OR REPLACE FUNCTION get_total_revenue(p_event_id INT)
RETURNS DECIMAL AS $$
DECLARE
    v_total DECIMAL;
BEGIN
    SELECT SUM(price) INTO v_total 
    FROM tickets WHERE event_id = p_event_id;
    
    RETURN COALESCE(v_total, 0);
END;
$$ LANGUAGE plpgsql;

-- Executing the Postgres Function
SELECT get_total_revenue(5);
```

## 6. Hands-on Exercises

1. Write a Stored Procedure named `sp_CreateUser` that accepts `@FirstName`, `@LastName`, and `@Email`. It should perform an `INSERT` into the `Users` table.
2. Execute your new Stored Procedure with dummy data.
3. Write a Scalar Function named `fn_CalculateTax` that accepts a `@Price` (Decimal) and returns the price multiplied by `0.15`.
4. Run a `SELECT` on the `Tickets` table and use your `fn_CalculateTax` function in the output.

## 7. Interview Questions

**Entry Level**
*   **Q:** Can you use a Stored Procedure in a `SELECT` statement? (e.g., `SELECT * FROM EXEC my_proc`)
    *   **A:** No. Stored Procedures must be executed standalone using the `EXEC` (or `CALL`) command. If you need to use parameterized logic inside a `SELECT` statement or `JOIN`, you must use a Table-Valued Function.
*   **Q:** What is the main security benefit of using Stored Procedures over raw SQL strings?
    *   **A:** Stored Procedures naturally protect against SQL Injection because the inputs are treated strictly as parameters/variables, not as executable SQL code.

**Intermediate Level**
*   **Q:** Why might a DBA tell you to avoid using a Scalar Function in the `SELECT` clause of a query that returns 1 million rows?
    *   **A:** Scalar functions execute row-by-row. If your query returns 1 million rows, the database must execute the function's logic 1 million separate times (acting like a hidden cursor/loop). This can severely degrade performance compared to standard set-based SQL math.
*   **Q:** In PostgreSQL 11+, why would you choose to create a `PROCEDURE` instead of a `FUNCTION`?
    *   **A:** You choose a Procedure when you need to manage transactions. Functions in Postgres execute within the transaction of the outer query; you cannot run `COMMIT` or `ROLLBACK` inside a function. Procedures allow you to commit partial work independently.

## 8. Preparation for Next Chapter
In Chapter 11, we will learn about **Triggers**. What happens if you want a piece of code to run *automatically* every time a row is inserted, without the application having to explicitly call a Stored Procedure? Triggers are the invisible, automatic event-handlers of the database world.
