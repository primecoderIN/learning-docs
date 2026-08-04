# Volume 1, Chapter 8 – Functions (String, Date, Numeric)

## 1. Concept Overview

While aggregate functions (`SUM`, `AVG`) operate on multiple rows to return a single value, **Scalar Functions** operate on a *single* row and return a single value for that row. 
They allow developers to manipulate, format, and calculate data on the fly inside the `SELECT`, `WHERE`, or `ORDER BY` clauses without having to write complex data-transformation code in the application layer (like Python or C#).

Functions are generally categorized into three domains: Strings (Text), Dates, and Numerics (Math).

## 2. String Functions

String functions are heavily used for data cleaning and formatting output for the UI.

### Concatenation (Joining strings)
```sql
-- SQL Server uses CONCAT() or the + operator
SELECT CONCAT(FirstName, ' ', LastName) AS FullName FROM Users;
SELECT FirstName + ' ' + LastName AS FullName FROM Users;

-- PostgreSQL uses CONCAT() or the || operator
SELECT FirstName || ' ' || LastName AS FullName FROM users;
```

### Changing Case & Trimming Whitespace
When users type data into forms, they often leave trailing spaces or mess up capitalization.
```sql
-- Convert to upper/lower case
SELECT UPPER(LastName), LOWER(Email) FROM Users;

-- Remove leading and trailing spaces (crucial for WHERE clauses!)
SELECT TRIM(Email) FROM Users; 
-- (Note: SQL Server historically used LTRIM(RTRIM(Email)), but TRIM() is standard now).
```

### Extracting and Replacing text
```sql
-- Extract the first 3 letters of a phone number (Area Code)
-- SUBSTRING(column, start_position, length) -- 1-indexed!
SELECT SUBSTRING(PhoneNumber, 1, 3) AS AreaCode FROM Users;

-- Replace 'Ave' with 'Avenue' in an address
SELECT REPLACE(Address, 'Ave', 'Avenue') FROM Venues;

-- Get the length of a string (SQL Server = LEN, Postgres = LENGTH)
SELECT LEN(FirstName) FROM Users;      -- SQL Server
SELECT LENGTH(first_name) FROM users;  -- Postgres
```

## 3. Date & Time Functions

Handling timezones and date math is notoriously difficult in programming. Pushing this to the database engine ensures perfect accuracy.

### Current Date and Time
```sql
SELECT GETDATE();           -- SQL Server
SELECT CURRENT_TIMESTAMP;   -- Standard ANSI / Postgres
SELECT NOW();               -- Postgres specific
```

### Date Arithmetic (Adding / Subtracting)
**Scenario:** A ticket expires 30 days after purchase.
```sql
-- SQL Server uses DATEADD(datepart, number, date)
SELECT DATEADD(DAY, 30, PurchaseDate) AS ExpirationDate FROM Tickets;

-- Postgres uses Interval math
SELECT purchase_date + INTERVAL '30 days' AS expiration_date FROM tickets;
```

### Date Difference
**Scenario:** How many days ago did the user register?
```sql
-- SQL Server uses DATEDIFF(datepart, start, end)
SELECT DATEDIFF(DAY, CreatedAt, GETDATE()) AS DaysActive FROM Users;

-- Postgres allows direct subtraction for Dates, or DATE_PART for timestamps
SELECT CURRENT_DATE - CAST(created_at AS DATE) AS days_active FROM users;
```

## 4. Numeric Functions

Numeric functions are essential for financial calculations and reporting.

```sql
-- 1. ROUND (Round to 2 decimal places)
SELECT ROUND(Price, 2) FROM Tickets;

-- 2. CEILING (Always round UP to the nearest integer)
-- 10.1 becomes 11
SELECT CEILING(Price) FROM Tickets;

-- 3. FLOOR (Always round DOWN to the nearest integer)
-- 10.9 becomes 10
SELECT FLOOR(Price) FROM Tickets;

-- 4. ABS (Absolute Value - converts negative to positive)
SELECT ABS(-500); -- Returns 500

-- 5. MOD / Modulo (Returns the remainder of a division)
-- Is the ticket ID an even or odd number?
SELECT TicketID % 2 FROM Tickets; -- SQL Server
SELECT MOD(ticket_id, 2) FROM tickets; -- Postgres
```

## 5. The Performance Danger of Functions

**Architect's Warning:** You must be incredibly careful when using functions in the `WHERE` clause.

```sql
-- THIS IS A CATASTROPHIC ANTI-PATTERN:
SELECT * FROM Users WHERE UPPER(Email) = 'JOHN@TEST.COM';
```
By wrapping the `Email` column in the `UPPER()` function, you have completely blinded the Query Optimizer. It can no longer use the B-Tree index on the Email column (an Index Seek). It must perform a **Full Table Scan**, running the `UPPER()` function on every single row in the database before comparing it. 
*Fix:* Ensure your database collation is case-insensitive, or format the input in the application *before* querying the database.

## 6. Hands-on Exercises

1. Write a query that returns the `Title` of all Events in all uppercase letters.
2. In SQL Server, write a query using `DATEDIFF` to find the number of *years* between '2000-01-01' and today. (If using Postgres, use `EXTRACT(YEAR FROM age(NOW(), '2000-01-01'))`).
3. Write a query that calculates a 15% tax on all `Tickets`, and uses `ROUND()` to ensure the tax only has 2 decimal places.

## 7. Interview Questions

**Entry Level**
*   **Q:** What does the `SUBSTRING()` function do?
    *   **A:** It extracts a specific portion of a string, taking three arguments: the column/string, the starting position (which is 1-indexed in SQL), and the length of the extraction.
*   **Q:** If a ticket price is `$14.01`, what will `CEILING(Price)` and `FLOOR(Price)` return?
    *   **A:** `CEILING` rounds up to the next integer, returning `15`. `FLOOR` rounds down to the previous integer, returning `14`.

**Intermediate Level**
*   **Q:** You want to find all users whose registration anniversary is exactly today. A junior developer writes: `WHERE MONTH(CreatedAt) = MONTH(GETDATE()) AND DAY(CreatedAt) = DAY(GETDATE())`. Why is this bad for performance?
    *   **A:** Wrapping the `CreatedAt` column in functions (`MONTH` and `DAY`) makes the query "Non-SARGable" (Search Argument Able). The database cannot use an index on `CreatedAt` and is forced to do a Full Table Scan. (A better approach is generating the start and end of the current day in the application and using `WHERE CreatedAt BETWEEN @StartOfDay AND @EndOfDay`).

## 8. Preparation for Next Chapter
In Chapter 9, we will explore **CTEs and Window Functions**. We will move beyond simple single-row functions and learn how to generate running totals, rank rows (`ROW_NUMBER()`), and compare a row to the row immediately preceding it (`LAG()`).
