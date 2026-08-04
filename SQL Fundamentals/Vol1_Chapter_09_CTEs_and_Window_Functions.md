# Volume 1, Chapter 9 – CTEs and Window Functions

## 1. Concept Overview

As a developer, you will eventually outgrow basic `JOINs` and `GROUP BY` statements. You will face two common challenges:
1.  **The "Spaghetti Code" Problem:** Nested subqueries inside of subqueries become impossible to read or debug.
2.  **The "Running Total" Problem:** You need to calculate a moving average or a running total, but standard aggregate functions (`SUM`, `AVG`) collapse all the rows into a single number, hiding the individual details.

Advanced SQL solves these problems with **CTEs** (Common Table Expressions) and **Window Functions**.

## 2. Common Table Expressions (CTEs)

A CTE is a temporary, named result set. Think of it as a variable that holds a table, which you define at the very top of your SQL script. It is vastly superior to nested subqueries for readability.

```sql
-- Connect to the NextEventDB

-- Standard Subquery (Hard to read)
SELECT * FROM (
    SELECT UserID, SUM(Price) AS TotalSpent FROM Tickets GROUP BY UserID
) AS sub WHERE TotalSpent > 1000;

-- CTE Equivalent (Clean and Top-Down)
WITH UserSpending AS (
    SELECT UserID, SUM(Price) AS TotalSpent
    FROM Tickets
    GROUP BY UserID
)
SELECT * 
FROM UserSpending 
WHERE TotalSpent > 1000;
```

You can even chain multiple CTEs together by separating them with a comma:
```sql
WITH UserSpending AS (
    SELECT UserID, SUM(Price) AS TotalSpent FROM Tickets GROUP BY UserID
),
VIP_Users AS (
    SELECT UserID FROM UserSpending WHERE TotalSpent > 1000
)
SELECT u.FirstName, u.Email 
FROM Users u
INNER JOIN VIP_Users v ON u.UserID = v.UserID;
```
*(Note: CTEs are generally NOT temporary tables in memory. They are just syntactical sugar that the query optimizer expands during execution).*

## 3. Window Functions

A Window Function performs a calculation across a set of table rows that are somehow related to the current row. 
*   A standard `SUM(Price) GROUP BY EventID` turns 100 ticket rows into 1 row. 
*   A Window Function `SUM(Price) OVER(PARTITION BY EventID)` calculates the total sum, but **keeps all 100 original rows**, simply appending the total sum as a new column on the end.

### The `OVER()` Clause
This is the magic keyword that defines a Window Function. It has two main sub-clauses:
1.  **`PARTITION BY`**: Divides the rows into buckets (just like `GROUP BY`).
2.  **`ORDER BY`**: Sorts the rows *within* that bucket.

## 4. Ranking Functions

Ranking functions are the most common use case for Window Functions.

*   **`ROW_NUMBER()`**: Assigns a unique, sequential integer (1, 2, 3) to each row within the partition.
*   **`RANK()`**: Assigns a rank, but if there is a tie, it gives the same number and *skips* the next number (1, 2, 2, 4).
*   **`DENSE_RANK()`**: Assigns a rank, but if there is a tie, it does *not* skip the next number (1, 2, 2, 3).
*   **`NTILE(N)`**: Divides the rows into *N* equal groups (e.g., `NTILE(4)` puts users into quartiles).

```sql
-- Rank tickets chronologically, grouped (partitioned) by each specific event.
SELECT 
    TicketID, 
    EventID, 
    Price,
    ROW_NUMBER() OVER (PARTITION BY EventID ORDER BY PurchaseDate ASC) AS TicketSequence
FROM Tickets;
```

## 5. Offset Functions (LEAD and LAG)

These functions allow a row to literally look backward or forward in time at its neighboring rows.

*   **`LAG()`**: Retrieves a value from a *previous* row.
*   **`LEAD()`**: Retrieves a value from a *subsequent* row.

**Scenario:** Calculate how many days elapsed between a user's first ticket purchase and their second ticket purchase.
```sql
SELECT 
    UserID,
    TicketID,
    PurchaseDate,
    -- Grab the purchase date of the PREVIOUS row for this specific user
    LAG(PurchaseDate, 1) OVER (PARTITION BY UserID ORDER BY PurchaseDate ASC) AS PrevPurchaseDate
FROM Tickets;
```
*(You can then wrap this in a CTE, and use `DATEDIFF` to subtract `PrevPurchaseDate` from `PurchaseDate`!)*

## 6. Running Totals (Window Frames)

If you combine an aggregate function (`SUM`) with an `ORDER BY` inside the `OVER` clause, it automatically creates a **Running Total**.

```sql
-- Calculate the running total of revenue for an event, ticket by ticket
SELECT 
    EventID, 
    TicketID, 
    Price,
    SUM(Price) OVER (PARTITION BY EventID ORDER BY PurchaseDate ASC) AS RunningRevenue
FROM Tickets;
```
*Output Example:*
*   Ticket 1 ($50) -> Running: $50
*   Ticket 2 ($50) -> Running: $100
*   Ticket 3 ($100) -> Running: $200

## 7. Hands-on Exercises

1. Write a CTE named `EventCounts` that counts the number of tickets sold for each `EventID`. Then write a `SELECT` statement outside the CTE to only show events that sold more than 500 tickets.
2. Write a query using `ROW_NUMBER()` to assign a sequence number to all `Users`, ordered by their `CreatedAt` date (Oldest first).
3. Write a query using `DENSE_RANK()` to rank `Events` based on their `Capacity` (Highest capacity is Rank 1).

## 8. Interview Questions

**Entry Level**
*   **Q:** Why would you use a CTE instead of a subquery?
    *   **A:** CTEs (Common Table Expressions) make complex SQL queries much easier to read and maintain because they are declared at the top of the script in a linear, top-down fashion, rather than nested deeply inside `FROM` clauses.
*   **Q:** What is the difference between `ROW_NUMBER()` and `RANK()`?
    *   **A:** `ROW_NUMBER()` always guarantees a unique, sequential number (1, 2, 3, 4) even if the values being ordered are tied. `RANK()` will assign the same number to ties, and then skip the next available number (1, 2, 2, 4).

**Intermediate Level**
*   **Q:** The CEO wants a report showing the 3 most recently registered users *per City*. How do you write this?
    *   **A:** This is the classic "Top N per Group" problem. You cannot do this with a simple `GROUP BY`. You must use a CTE and a Window Function.
        ```sql
        WITH RankedUsers AS (
            SELECT FirstName, City, CreatedAt,
            ROW_NUMBER() OVER(PARTITION BY City ORDER BY CreatedAt DESC) AS rn
            FROM Users
        )
        SELECT * FROM RankedUsers WHERE rn <= 3;
        ```

## 9. Preparation for Next Chapter
In Chapter 10, we will dive into **Stored Procedures and User-Defined Functions (UDFs)**. You will learn how to encapsulate your SQL queries into reusable, parameterized scripts that can be called directly from your backend application code.
