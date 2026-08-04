# Volume 1, Chapter 14 – Advanced SQL and Performance

## 1. Concept Overview

This chapter bridges the gap between a Junior Developer who writes queries that *work*, and a Senior Developer who writes queries that *scale*. We will cover advanced techniques for restructuring data (Pivot), breaking down complex logic (Temp Tables), generating SQL on the fly (Dynamic SQL), and the absolute basics of Query Tuning.

## 2. Temporary Tables & Table Variables

When writing a 2,000-line Stored Procedure, you often need a place to temporarily store intermediate calculations. 

### Temporary Tables (`#TempTables`)
A Temporary Table behaves exactly like a real physical table, but it is stored in the database's `tempdb` memory space. It is automatically deleted the moment your session (or stored procedure) closes. You can add indexes to it, making it incredible for handling millions of temporary rows.

```sql
-- Create a Temp Table (SQL Server syntax using the # prefix)
CREATE TABLE #TopSpenders (
    UserID INT,
    TotalSpent DECIMAL(10,2)
);

-- Insert data into it
INSERT INTO #TopSpenders
SELECT UserID, SUM(Price) FROM Tickets GROUP BY UserID HAVING SUM(Price) > 5000;

-- Query it, join it to other tables, then it deletes itself when you disconnect.
SELECT u.FirstName, t.TotalSpent 
FROM Users u 
INNER JOIN #TopSpenders t ON u.UserID = t.UserID;
```

### Table Variables (`@TableVars`)
Table Variables are declared like standard variables. They are stored entirely in RAM (memory) rather than on disk. Because they lack advanced statistics, they are only fast for *very small* datasets (under 1,000 rows).
```sql
DECLARE @RecentEvents TABLE (EventID INT, Title VARCHAR(100));

INSERT INTO @RecentEvents
SELECT EventID, Title FROM Events WHERE Date > GETDATE();
```

## 3. Dynamic SQL

Sometimes you don't know the exact query you need to run until the user clicks a button. **Dynamic SQL** allows you to build a SQL query as a text string, and then command the database engine to execute that string.

```sql
-- SQL Server Example
DECLARE @TableName VARCHAR(50) = 'Users';
DECLARE @SQLQuery NVARCHAR(MAX);

-- Build the string
SET @SQLQuery = 'SELECT COUNT(*) FROM ' + @TableName;

-- Execute the string
EXEC sp_executesql @SQLQuery;
```

**The Grave Danger of Dynamic SQL:**
If you ever concatenate user input directly into a Dynamic SQL string, you open your database to **SQL Injection**. 
*Example:* A user types `Users; DROP TABLE Tickets;` into a search box. Your dynamic SQL string becomes:
`SELECT COUNT(*) FROM Users; DROP TABLE Tickets;`
Always use parameterized inputs (`sp_executesql` in SQL Server, or `USING` in Postgres) to sanitize dynamic queries!

## 4. Pivot and Unpivot

Reporting dashboards often require data to be rotated.
*   **PIVOT:** Turns rows into columns. (e.g., Turning a `Month` column with 12 rows into 12 separate columns: `Jan | Feb | Mar`).
*   **UNPIVOT:** Turns columns into rows.

### SQL Server PIVOT Syntax
```sql
-- Base Data: (Year, City, TotalTickets)
-- We want a table where Cities are the Rows, and Years (2025, 2026) are the Columns.

SELECT City, [2025], [2026]
FROM (
    -- The raw data query
    SELECT City, YEAR(PurchaseDate) AS TicketYear, Price 
    FROM Tickets t INNER JOIN Events e ON t.EventID = e.EventID
) AS SourceTable
PIVOT (
    -- The aggregation we want to perform
    SUM(Price) 
    -- The column we are rotating horizontally
    FOR TicketYear IN ([2025], [2026])
) AS PivotTable;
```
*(Note: Postgres does not have a native `PIVOT` keyword. You achieve the same result using the `crosstab()` function from the `tablefunc` extension, or by using aggregate `FILTER` clauses).*

## 5. Performance Optimization: Avoiding Full Table Scans

The most important skill a developer can learn is how to write **SARGable** (Search ARGument ABLE) queries. A query is SARGable if the Query Optimizer can use an Index Seek to find the data.

### Rule 1: Never wrap indexed columns in functions
**Bad (Full Table Scan):** `WHERE YEAR(PurchaseDate) = 2026`
**Good (Index Seek):** `WHERE PurchaseDate >= '2026-01-01' AND PurchaseDate < '2027-01-01'`

### Rule 2: Never use leading wildcards
**Bad (Full Table Scan):** `WHERE Email LIKE '%@gmail.com'`
**Good (Index Seek):** `WHERE Email LIKE 'john@%'` (Only works if you know the start of the string). If you must search suffixes, consider Full-Text Search engines.

### Rule 3: Avoid Math on Columns
**Bad (Full Table Scan):** `WHERE Price * 1.10 > 100` (Database must calculate tax on every row before checking).
**Good (Index Seek):** `WHERE Price > 100 / 1.10` (Math is done once on the static number, index is preserved).

## 6. Hands-on Exercises

1. Write a query to create a temporary table named `#HighCapacityVenues` containing the `VenueID` and `Name` of all venues with a capacity > 10,000.
2. Rewrite the following non-SARGable query to be SARGable so it can use an index: 
`SELECT * FROM Users WHERE LEFT(FirstName, 1) = 'A';`

## 7. Interview Questions

**Entry Level**
*   **Q:** What is the difference between a Temp Table (`#Table`) and a normal table?
    *   **A:** A Temp table only exists for the duration of the session or connection that created it. Once the script finishes or the connection drops, it is automatically deleted.
*   **Q:** What does PIVOT do?
    *   **A:** It rotates data from a row-based format into a column-based format, usually applying an aggregate function (like SUM or COUNT) in the process. It's heavily used for BI reporting.

**Intermediate Level**
*   **Q:** You have a stored procedure that inserts 5 million rows into a temporary storage space before processing them. Do you use a Table Variable (`@Table`) or a Temp Table (`#Table`)?
    *   **A:** You must use a Temp Table. Table Variables do not maintain distribution statistics and are highly inefficient for massive datasets. Temp Tables allow you to build indexes and are processed by the query optimizer just like physical tables.
*   **Q:** What is SQL Injection, and how does Dynamic SQL cause it?
    *   **A:** SQL Injection is a cyberattack where a malicious user inserts executable SQL commands into an input field. If a developer uses basic string concatenation (`SET @Query = 'SELECT * FROM Users WHERE Name = ' + @UserInput`), the dynamic string executes the hacker's payload.

## 8. Preparation for Next Chapter
In Chapter 15, we will conclude Volume 1 with the **SQL Interview Masterclass**. We will consolidate everything you have learned into the Top 50 most frequently asked SQL interview questions, covering everything from `TRUNCATE` vs `DELETE`, to `HAVING` vs `WHERE`, to ranking Window Functions.
