# Volume 1, Chapter 6 – Subqueries and Views

## 1. Concept Overview

As your SQL skills grow, you will encounter business problems that cannot be solved with a single, simple query. 
*   **Subqueries** (or Nested Queries) allow you to embed a `SELECT` statement *inside* another SQL statement. The inner query executes first, and its result is passed to the outer query.
*   **Views** allow you to save a complex, multi-join query permanently inside the database. To the application, a View looks and acts exactly like a real physical table, but it is actually a "virtual" table that runs your saved query dynamically every time it is called.

## 2. Single-Row Subqueries

A single-row subquery returns exactly one column and one row (a scalar value). It is typically used with standard comparison operators (`=`, `>`, `<`).

**Scenario:** Find all tickets that cost more than the *average* ticket price.
You cannot write `WHERE Price > AVG(Price)` because aggregate functions aren't allowed in the `WHERE` clause. You must use a subquery.

```sql
-- Connect to NextEventDB

-- Step 1 (Inner Query): Calculates the average ($75.50)
-- Step 2 (Outer Query): Selects tickets where Price > 75.50
SELECT TicketID, Price 
FROM Tickets 
WHERE Price > (
    SELECT AVG(Price) FROM Tickets
);
```
*(Warning: If the inner query accidentally returns 2 rows, the database will throw an error because it cannot mathematically compute `Price > (List of multiple values)`).*

## 3. Multi-Row Subqueries

When a subquery returns a single column but *multiple* rows, you must use list operators like `IN`, `ANY`, or `ALL`.

**Scenario:** Find the names of all Users who have purchased a ticket to a 'VIP' event.

```sql
-- The inner query returns a list of EventIDs (e.g., 5, 12, 19)
-- The outer query checks if the user's ticket matches any of those EventIDs.
SELECT FirstName, LastName 
FROM Users 
WHERE UserID IN (
    SELECT UserID 
    FROM Tickets 
    WHERE EventID IN (
        SELECT EventID FROM Events WHERE Tier = 'VIP'
    )
);
```
*(Note: While nested subqueries work, this specific example is usually written as a 3-table `INNER JOIN` for better readability and performance).*

## 4. Correlated Subqueries

This is a crucial concept for technical interviews.
*   **Uncorrelated Subquery:** Runs exactly **once**. The result is cached and used by the outer query. (Like the average price example above).
*   **Correlated Subquery:** References a column from the *outer* query. Because of this dependency, the inner query must execute **once for every single row** evaluated by the outer query. 

**Scenario:** Find users who have spent more than the average amount *for their specific city*.

```sql
SELECT u1.FirstName, u1.City, u1.TotalSpent
FROM Users u1
WHERE u1.TotalSpent > (
    -- This inner query runs over and over, once for every user in u1.
    -- If u1 is currently evaluating a user in 'London', it calculates the London average.
    SELECT AVG(TotalSpent) 
    FROM Users u2 
    WHERE u2.City = u1.City
);
```
*(Architect's Warning: Because correlated subqueries execute row-by-row, they act like procedural `For-Loops`. On a 10-million row table, the inner query runs 10 million times. They are notoriously slow and should often be rewritten using `JOINs` or Window Functions).*

## 5. Subqueries in the FROM Clause (Derived Tables)

You can place a subquery in the `FROM` clause. The database treats the result of this subquery as a temporary table. You **must** give this derived table an alias.

```sql
-- Step 1: Create a derived table of total revenue per event
-- Step 2: Query the derived table to find the top earners
SELECT EventID, TotalRevenue 
FROM (
    SELECT EventID, SUM(Price) AS TotalRevenue 
    FROM Tickets 
    GROUP BY EventID
) AS EventRevenues
WHERE TotalRevenue > 1000000;
```

## 6. Views

If you write a complex 5-table join with multiple subqueries, you don't want the application developers to have to copy-paste that 50-line SQL monstrosity into their code every time they need a report. Instead, you create a **View**.

```sql
CREATE VIEW vw_VIP_User_Report AS
SELECT 
    u.FirstName, 
    u.LastName, 
    e.Title AS EventTitle, 
    t.Price
FROM Users u
INNER JOIN Tickets t ON u.UserID = t.UserID
INNER JOIN Events e ON t.EventID = e.EventID
WHERE t.Price > 500;
```

Now, the application developer simply runs:
```sql
SELECT * FROM vw_VIP_User_Report ORDER BY LastName;
```
The database engine intercepts this, expands the underlying SQL of the view, and executes it.

## 7. Updatable Views

Views are primarily used for `SELECT` (reading data). However, in both SQL Server and PostgreSQL, you can occasionally run `INSERT`, `UPDATE`, or `DELETE` against a view, and the database will pass the changes down to the underlying physical table.

**The Catch:** The view must be simple. If the view contains a `JOIN`, a `GROUP BY`, `DISTINCT`, or an Aggregate Function, it is strictly **Read-Only**. You cannot `UPDATE` an average.

## 8. Hands-on Exercises

1. Write a single-row subquery to find all `Events` whose capacity is less than the maximum capacity of any event in the database.
2. Write a multi-row subquery using `IN` to find all `Locations` that have hosted an event in the year 2026.
3. Save the query from Exercise 2 as a permanent View named `vw_Active_Locations_2026`.
4. Run a `SELECT *` from your new View.

## 9. Interview Questions

**Entry Level**
*   **Q:** What is a View?
    *   **A:** A View is a virtual table representing the result of a stored SQL query. It does not store data physically (unless it's a Materialized View); it simply saves the SQL logic to simplify complex queries and restrict data access.
*   **Q:** What happens if a single-row subquery returns two rows?
    *   **A:** The database will throw a runtime error, because operators like `=` or `>` mathematically require exactly one value on the right side of the equation.

**Intermediate Level**
*   **Q:** Explain the difference between a Correlated and Uncorrelated subquery.
    *   **A:** An uncorrelated subquery is completely independent of the outer query; it executes once, and its result is substituted into the outer query. A correlated subquery references a column from the outer query, meaning it must execute repeatedly—once for every row processed by the outer query.
*   **Q:** Can you `INSERT` data into a View?
    *   **A:** Yes, but only if it is an "Updatable View." This generally means the view must map directly to a single base table without any `JOINs`, `GROUP BY` clauses, or aggregate calculations.

## 10. Preparation for Next Chapter
In Chapter 7, we will cover **Indexes (Basics)**. You will learn the developer's perspective of how to speed up slow queries by creating Clustered and Non-Clustered indexes, ensuring your `WHERE` clauses and `JOINs` run in milliseconds instead of minutes.
