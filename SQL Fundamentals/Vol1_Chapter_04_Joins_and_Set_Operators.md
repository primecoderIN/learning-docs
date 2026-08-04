# Volume 1, Chapter 4 – Joins and Set Operators

## 1. Concept Overview

Relational databases are built on the principle of **Normalization**—breaking large datasets into smaller, specialized tables to prevent data duplication. For example, instead of storing a user's name and email on every single ticket they buy, we store the user once in the `Users` table, and only store their `UserID` in the `Tickets` table.

But when a web application needs to display a ticket alongside the buyer's email address, we must stitch those tables back together. We do this using **JOINs**. 
A JOIN acts as a bridge, linking two tables based on a common column (usually a Primary Key to Foreign Key relationship).

If Joins link data *horizontally* (adding columns from Table B to Table A), **Set Operators** (like `UNION`) link data *vertically* (stacking rows from Query B underneath the rows from Query A).

## 2. INNER JOIN

The `INNER JOIN` is the most common join. It returns *only* the rows where there is a match in **both** tables. If a user exists but hasn't bought any tickets, they will be excluded from the results.

```sql
-- Connect to the NextEventDB

-- Show the Ticket ID and the exact Email of the person who bought it.
SELECT 
    t.TicketID, 
    t.Price, 
    u.Email 
FROM Tickets t
INNER JOIN Users u ON t.UserID = u.UserID;
```
*(Notice the use of `t` and `u`. These are **Table Aliases**. They save you from having to type `Tickets.TicketID` repeatedly, making queries much cleaner).*

## 3. OUTER JOINS (LEFT, RIGHT, FULL)

Unlike an INNER JOIN, an OUTER JOIN guarantees that all rows from one side of the join are returned, even if there is no match on the other side. Missing matches are filled with `NULL`.

### `LEFT JOIN` (or `LEFT OUTER JOIN`)
Returns **ALL** rows from the "left" table (the one written first), and the matched rows from the "right" table.
```sql
-- Show ALL users, and any tickets they might have bought.
-- If a user has zero tickets, the TicketID and Price columns will just show NULL.
SELECT 
    u.FirstName, 
    u.Email, 
    t.TicketID 
FROM Users u
LEFT JOIN Tickets t ON u.UserID = t.UserID;
```

### `RIGHT JOIN`
The exact opposite of a LEFT JOIN. It returns all rows from the right table. In professional environments, `RIGHT JOIN` is rarely used because you can achieve the exact same result by simply swapping the order of the tables and using a `LEFT JOIN` (which reads more logically left-to-right).

### `FULL OUTER JOIN`
Returns **ALL** rows from **BOTH** tables. If there's a match, they are linked. If there is a User with no tickets, they show up with NULL tickets. If there is a Ticket with a deleted/missing User, it shows up with a NULL user.
```sql
SELECT 
    u.Email, 
    t.TicketID 
FROM Users u
FULL OUTER JOIN Tickets t ON u.UserID = t.UserID;
```

## 4. CROSS JOIN (Cartesian Product)

A `CROSS JOIN` matches *every single row* in Table A with *every single row* in Table B. It does not use an `ON` clause. 
If Table A has 10 rows and Table B has 100 rows, a `CROSS JOIN` returns 1,000 rows.

```sql
-- Generate every possible combination of Event and Ticket Tier (VIP, General, etc.)
SELECT 
    e.Title, 
    tt.TierName 
FROM Events e
CROSS JOIN TicketTiers tt;
```
*(Warning: Accidental Cross Joins—often caused by omitting the `ON` clause in old ANSI-89 syntax—are the #1 cause of database memory exhaustion).*

## 5. SELF JOIN

A `SELF JOIN` is not a special keyword. It is simply a standard join (Inner or Left) where a table is joined to *itself*. This is crucial for querying hierarchical data, like a company org chart or a referral system.

```sql
-- We have a Users table where some users were referred by other users (ReferredBy_UserID).
SELECT 
    u.Email AS NewUser, 
    ref.Email AS ReferredBy
FROM Users u
LEFT JOIN Users ref ON u.ReferredBy_UserID = ref.UserID;
```

## 6. Set Operators (UNION, INTERSECT, EXCEPT)

Set Operators combine the results of two completely separate `SELECT` queries into a single vertical column stack.
**Rule:** Both queries must have the exact same number of columns, and the data types must be compatible.

### `UNION` vs `UNION ALL`
Combines the result sets.
*   `UNION` automatically scans the combined list and removes any duplicate rows (which requires a hidden, expensive Sort operation).
*   `UNION ALL` simply stacks them together, keeping duplicates. It is significantly faster. Always use `UNION ALL` unless you specifically need deduplication.

```sql
-- Create a master mailing list from both Users and GuestPurchasers
SELECT Email, FirstName FROM Users
UNION ALL
SELECT Email, FirstName FROM GuestPurchasers;
```

### `INTERSECT`
Returns *only* the distinct rows that appear in **BOTH** queries. (e.g., Show me emails that are in both the 2025 and 2026 attendee lists).

### `EXCEPT` (SQL Server) / `MINUS` (Oracle)
Returns distinct rows that appear in the first query, but **NOT** in the second query. 
```sql
-- Find users who bought a ticket in 2025, but DID NOT buy one in 2026
SELECT UserID FROM Tickets WHERE YEAR(PurchaseDate) = 2025
EXCEPT
SELECT UserID FROM Tickets WHERE YEAR(PurchaseDate) = 2026;
```
*(Note: Postgres supports both `EXCEPT` and `INTERSECT` natively).*

## 7. Hands-on Exercises

1. Write an `INNER JOIN` linking the `Events` table to the `Locations` table on `LocationID`, returning the Event Title and the Location Name.
2. Write a `LEFT JOIN` linking `Events` to `Tickets`. Can you find any events that have sold zero tickets? (Hint: add `WHERE t.TicketID IS NULL`).
3. Write a `UNION ALL` query that stacks a list of `Event` Titles on top of a list of `Location` Names.

## 8. Interview Questions

**Entry Level**
*   **Q:** What is the difference between an `INNER JOIN` and a `LEFT JOIN`?
    *   **A:** An `INNER JOIN` requires a match in both tables. If a row in the left table doesn't have a match in the right table, it is dropped from the results. A `LEFT JOIN` guarantees that every row from the left table is returned; if there is no match on the right, the right-side columns are filled with NULLs.
*   **Q:** What is the difference between `UNION` and `UNION ALL`?
    *   **A:** `UNION` removes duplicate rows, which makes it slower because the database must perform a sort/distinct operation. `UNION ALL` includes all duplicates and is much faster.

**Intermediate Level**
*   **Q:** How would you find rows in Table A that do *not* exist in Table B?
    *   **A:** There are two main ways. You can use an `EXCEPT` set operator. Or, you can use a `LEFT JOIN` from A to B, and add a `WHERE B.PrimaryKey IS NULL` clause to filter out the matches.
*   **Q:** Why is a `CROSS JOIN` dangerous in a production environment?
    *   **A:** It creates a Cartesian product. If you accidentally CROSS JOIN two tables with 1 million rows each, the database engine will attempt to generate a 1-trillion-row result set in memory, which will instantly crash the server or trigger the Out-Of-Memory killer.

## 9. Preparation for Next Chapter
In Chapter 5, we will explore **Constraints and Keys**. Now that you know how to query multiple tables, you need to understand how the database enforces the rules that link them together. We will cover Primary Keys, Foreign Keys, Unique constraints, and how they prevent garbage data from entering your system.
