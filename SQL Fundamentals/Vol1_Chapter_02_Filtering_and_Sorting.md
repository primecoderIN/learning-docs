# Volume 1, Chapter 2 – Filtering and Sorting

## 1. Concept Overview

A database is useless if you can only retrieve the entire table at once. In enterprise applications, you rarely want to select *all* 10 million rows. You want to retrieve a very specific subset of data (Filtering) and you usually want that data presented in a specific sequence (Sorting).

*   **Filtering (`WHERE`):** Allows you to specify the exact conditions a row must meet to be included in the result set. If the row evaluates to `TRUE`, it is returned. If `FALSE` or `NULL`, it is excluded.
*   **Sorting (`ORDER BY`):** Tells the database engine to sort the final result set alphabetically, chronologically, or numerically before returning it to the application.
*   **Pagination (`LIMIT / TOP / OFFSET`):** Restricts the total number of rows returned, which is crucial for web pages that display data in chunks (e.g., "Page 1 of 50").

## 2. The WHERE Clause and Comparison Operators

The `WHERE` clause is placed immediately after the `FROM` clause. It uses standard mathematical comparison operators to filter data.

```sql
-- Connect to the NextEventDB
-- 1. Exact Match (Equals)
SELECT FirstName, Email FROM Users WHERE UserID = 5;

-- 2. Not Equal To
-- Note: Both != and <> are valid in standard SQL. <> is technically the ANSI standard.
SELECT * FROM Tickets WHERE Status <> 'Cancelled';

-- 3. Greater Than / Less Than
SELECT EventID, Title FROM Events WHERE Capacity >= 5000;
```

## 3. Logical Operators (AND, OR, NOT)

When you need to combine multiple conditions, you use logical operators.
*Architect's Note:* Always use parentheses `()` when mixing `AND` and `OR` to explicitly define the order of operations, just like in algebra.

```sql
-- 1. AND (Both conditions must be TRUE)
SELECT * FROM Tickets 
WHERE Price > 100 AND Status = 'Purchased';

-- 2. OR (Only one condition needs to be TRUE)
SELECT * FROM Events 
WHERE City = 'New York' OR City = 'Los Angeles';

-- 3. Mixing AND/OR with Parentheses (Crucial!)
-- We want VIP tickets in NY, OR ANY ticket in LA.
SELECT * FROM Tickets 
WHERE (City = 'New York' AND Status = 'VIP') OR (City = 'Los Angeles');

-- 4. NOT (Reverses the condition)
SELECT * FROM Users WHERE NOT Status = 'Banned';
```

## 4. Advanced Filters (BETWEEN, IN, LIKE, IS NULL)

SQL provides elegant shorthand operators to replace long, messy `AND/OR` chains.

### The `BETWEEN` Operator
Used for ranges (numbers or dates). It is **inclusive** of both endpoints.
```sql
-- Instead of: WHERE Price >= 50 AND Price <= 150
SELECT * FROM Tickets WHERE Price BETWEEN 50 AND 150;

-- Crucial Note on Dates: Dates usually contain times! 
-- '2026-01-01' technically means '2026-01-01 00:00:00'. 
-- A ticket bought at 10:00 AM on Jan 1st will NOT be included if you say BETWEEN '2025-01-01' AND '2026-01-01'.
```

### The `IN` Operator
Used to match a value against a list of options. Vastly cleaner than multiple `OR` statements.
```sql
-- Instead of: WHERE City = 'NY' OR City = 'LA' OR City = 'CHI'
SELECT * FROM Events WHERE City IN ('NY', 'LA', 'CHI');
```

### The `LIKE` Operator (Pattern Matching)
Used to search for partial text matches using wildcards.
*   `%` (Percent sign): Represents zero, one, or multiple characters.
*   `_` (Underscore): Represents exactly one single character.

```sql
-- Find any user whose email ends with '@gmail.com'
SELECT * FROM Users WHERE Email LIKE '%@gmail.com';

-- Find any user whose first name starts with 'J'
SELECT * FROM Users WHERE FirstName LIKE 'J%';

-- Find a 3-letter name starting with 'S' and ending with 'm' (e.g., Sam)
SELECT * FROM Users WHERE FirstName LIKE 'S_m';
```
*(Warning: Using a leading wildcard like `%gmail.com` prevents the database from using an index, forcing a massive full table scan. Avoid leading wildcards in production when possible).*

### The `IS NULL` Operator
In SQL, `NULL` means "Unknown" or "Missing Data." You **cannot** use an equals sign to check for NULL. `WHERE Email = NULL` will always return false, because you cannot equal an unknown.
```sql
-- Correct way to find missing data
SELECT * FROM Users WHERE PhoneNumber IS NULL;

-- Correct way to find populated data
SELECT * FROM Users WHERE PhoneNumber IS NOT NULL;
```

## 5. Sorting Data (ORDER BY)

By default, SQL does not guarantee any specific sort order. If you don't specify an `ORDER BY`, the database returns rows in whatever physical order they happen to be on the hard drive (which can change randomly).

```sql
-- Sort ascending (A-Z, Lowest to Highest). ASC is the default.
SELECT * FROM Events ORDER BY Date ASC;

-- Sort descending (Z-A, Highest to Lowest)
SELECT * FROM Tickets ORDER BY Price DESC;

-- Multi-column sorting (Sort by City A-Z, then by Date newest-first within that city)
SELECT * FROM Events ORDER BY City ASC, Date DESC;
```

## 6. Limiting and Pagination (SQL Server vs PostgreSQL)

When building a website, you don't want to load 10,000 events on one page. You want to load 50 at a time. This syntax varies heavily between database engines.

### SQL Server Syntax (`TOP` and `OFFSET...FETCH`)
```sql
-- Return only the first 10 rows
SELECT TOP 10 * FROM Tickets ORDER BY Price DESC;

-- Pagination (SQL Server 2012+). Skip the first 50 rows, return the next 10.
SELECT * FROM Tickets 
ORDER BY PurchaseDate DESC
OFFSET 50 ROWS 
FETCH NEXT 10 ROWS ONLY;
```

### PostgreSQL Syntax (`LIMIT` and `OFFSET`)
```sql
-- Return only the first 10 rows
SELECT * FROM tickets ORDER BY price DESC LIMIT 10;

-- Pagination. Skip the first 50 rows, return the next 10.
SELECT * FROM tickets 
ORDER BY purchase_date DESC 
LIMIT 10 OFFSET 50;
```

## 7. Hands-on Exercises

1. Write a query to find all `Events` where the `Capacity` is greater than 1,000 AND the `City` is 'London'.
2. Write a query to find all `Users` who registered in the year 2026. (Hint: Use `BETWEEN '2026-01-01' AND '2026-12-31'`).
3. Write a query to find the top 5 most expensive `Tickets` ever sold. (Remember to use `ORDER BY` and `LIMIT`/`TOP`).
4. Write a query to find all `Users` whose `LastName` is missing (NULL).

## 8. Interview Questions

**Entry Level**
*   **Q:** What is the difference between `=` and `LIKE`?
    *   **A:** `=` looks for an exact, identical string match. `LIKE` is used with wildcard characters (`%` or `_`) to find partial pattern matches within a string.
*   **Q:** How do you filter records where a specific column is empty/null?
    *   **A:** You must use `IS NULL`. Using `= NULL` is syntactically invalid because NULL is an unknown state, not a value.

**Intermediate Level**
*   **Q:** Why might a DBA ask you to avoid using `WHERE FirstName LIKE '%John%'`?
    *   **A:** Placing a wildcard `%` at the very beginning of a string prevents the Query Optimizer from navigating a B-Tree index (Index Seek). The database is forced to read every single row in the entire table (Index Scan / Table Scan), which will destroy performance on large tables.

## 9. Preparation for Next Chapter
In Chapter 3, we will cover **Aggregations and Grouping**. We will learn how to take millions of rows and compress them into meaningful statistics—calculating the total revenue for an event (`SUM`), finding the average ticket price (`AVG`), and counting the number of active users (`COUNT`) using `GROUP BY`.
