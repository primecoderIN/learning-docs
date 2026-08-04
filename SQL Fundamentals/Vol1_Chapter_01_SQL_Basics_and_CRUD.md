# Volume 1, Chapter 1 – SQL Basics and CRUD Operations

## 1. Concept Overview

Before diving into advanced architecture, every developer must master the foundational vocabulary of Relational Database Management Systems (RDBMS).

*   **SQL (Structured Query Language):** The standard language used to communicate with a relational database. It is declarative, meaning you describe *what* you want, not *how* to get it.
*   **Database:** A structured, electronic container that houses data securely.
*   **Table:** A grid within a database, similar to a spreadsheet tab, that holds related data.
*   **Row (Record/Tuple):** A single, horizontal entry in a table (e.g., one specific User).
*   **Column (Field/Attribute):** A vertical category in a table (e.g., First Name, Email, Date of Birth). Every column must have a defined **Data Type**.

SQL commands are traditionally categorized into three primary subsets:
1.  **DDL (Data Definition Language):** Defines the structure (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`).
2.  **DML (Data Manipulation Language):** Modifies the data (`INSERT`, `UPDATE`, `DELETE`).
3.  **DQL (Data Query Language):** Retrieves the data (`SELECT`).

## 2. Common Data Types

When creating a table, you must strictly define what type of data each column will hold.

| Category | SQL Server | PostgreSQL | Description |
| :--- | :--- | :--- | :--- |
| **Integers** | `INT`, `BIGINT` | `INTEGER`, `BIGINT` | Whole numbers. Use `BIGINT` for massive IDs. |
| **Decimals** | `DECIMAL(10,2)` | `NUMERIC(10,2)` | Exact precision numbers, crucial for financial/money data. |
| **Text** | `VARCHAR(255)`, `NVARCHAR` | `VARCHAR(255)`, `TEXT` | Variable-length strings. (`NVARCHAR` in SQL Server supports Unicode/Emojis). |
| **Boolean** | `BIT` (0 or 1) | `BOOLEAN` (True/False) | Logical true or false. |
| **Date & Time**| `DATETIME2` | `TIMESTAMP`, `TIMESTAMPTZ` | Stores date and time. `TIMESTAMPTZ` (Postgres) includes time zone offsets. |
| **Unique IDs** | `UNIQUEIDENTIFIER` | `UUID` | Universally Unique Identifiers (e.g., `123e4567-e89b-12d3...`). |

## 3. Creating Databases and Tables (DDL)

To build the foundation of our **NextEvent** enterprise platform, we must first create the database and its tables.

```sql
-- 1. Create the Database
CREATE DATABASE NextEventDB;

-- (In SQL Server, you must explicitly switch to the new database context)
USE NextEventDB;

-- 2. Create a Table
CREATE TABLE Users (
    UserID INT,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    Email VARCHAR(100),
    CreatedAt DATETIME2
);
```

## 4. Altering and Dropping Tables

Requirements change. If the marketing team suddenly wants to track a user's phone number, we use the `ALTER` command rather than deleting the table.

```sql
-- Add a new column to an existing table
ALTER TABLE Users ADD PhoneNumber VARCHAR(20);

-- Change the data type of an existing column (SQL Server syntax)
ALTER TABLE Users ALTER COLUMN PhoneNumber VARCHAR(50);

-- Change the data type of an existing column (Postgres syntax)
ALTER TABLE Users ALTER COLUMN PhoneNumber TYPE VARCHAR(50);

-- Delete a column completely
ALTER TABLE Users DROP COLUMN PhoneNumber;

-- Completely destroy a table and all its data permanently
DROP TABLE Users;
```

## 5. CRUD Operations (DML & DQL)

CRUD stands for **Create, Read, Update, Delete**. These four operations encompass 95% of all interactions a web application will have with a database.

### CREATE (`INSERT`)
Adds new rows into a table.

```sql
-- Standard Insert (Specifying columns is best practice)
INSERT INTO Users (UserID, FirstName, LastName, Email, CreatedAt)
VALUES (1, 'Alice', 'Smith', 'alice@test.com', '2026-01-01 10:00:00');

-- Multi-row Insert (Faster than multiple single INSERT statements)
INSERT INTO Users (UserID, FirstName, LastName, Email)
VALUES 
    (2, 'Bob', 'Jones', 'bob@test.com'),
    (3, 'Charlie', 'Brown', 'charlie@test.com');
```

### READ (`SELECT`)
Retrieves data from a table.

```sql
-- Retrieve all columns and all rows (Avoid doing this in production!)
SELECT * FROM Users;

-- Retrieve specific columns (Best practice for network performance)
SELECT FirstName, Email FROM Users;

-- Use ALIASES to rename output columns for easier readability
SELECT FirstName AS [First Name], Email AS ContactEmail FROM Users;
```

### UPDATE (`UPDATE`)
Modifies existing data. 
**DANGER:** If you forget the `WHERE` clause, you will update *every single row* in the table.

```sql
-- Update a specific user's email
UPDATE Users 
SET Email = 'alice.new@test.com' 
WHERE UserID = 1;

-- Update multiple columns at once
UPDATE Users 
SET FirstName = 'Robert', Email = 'bob.new@test.com'
WHERE UserID = 2;
```

### DELETE (`DELETE`)
Removes specific rows from a table.
**DANGER:** If you forget the `WHERE` clause, you will delete *every single row* in the table.

```sql
-- Delete a specific user
DELETE FROM Users WHERE UserID = 3;
```

## 6. The Big Three: DELETE vs TRUNCATE vs DROP

A classic interview question is knowing the distinct differences between these three destructive commands.

*   **`DELETE FROM TableName` (DML):**
    *   Removes rows one-by-one.
    *   Fires triggers.
    *   Is fully logged in the Transaction Log (can be rolled back easily).
    *   Extremely slow for large tables.
*   **`TRUNCATE TABLE TableName` (DDL):**
    *   Instantly empties the entire table by deallocating the physical data pages.
    *   Does *not* fire triggers.
    *   Uses minimal transaction logging.
    *   Cannot have a `WHERE` clause (it's all or nothing).
*   **`DROP TABLE TableName` (DDL):**
    *   Destroys the data AND the table structure itself. The table ceases to exist.

## 7. SQL Server vs PostgreSQL Syntax Nuances

While basic CRUD is nearly identical across all SQL dialects, data types often trip developers up.

| Concept | SQL Server | PostgreSQL |
| :--- | :--- | :--- |
| **String Concatenation** | `SELECT FirstName + ' ' + LastName` | `SELECT FirstName || ' ' || LastName` |
| **Date Time** | `GETDATE()` | `CURRENT_TIMESTAMP` or `NOW()` |
| **Auto-Incrementing IDs**| `IDENTITY(1,1)` inside CREATE TABLE | `SERIAL` or `GENERATED ALWAYS AS IDENTITY` |

## 8. Hands-on Exercises

1. Create a table named `Venues` with columns for `VenueID` (Integer), `Name` (String), and `Capacity` (Integer).
2. Insert 3 rows of dummy data into the `Venues` table.
3. Update the `Capacity` of one specific venue to `50000`.
4. Add a new column to the table called `City` (String) using `ALTER TABLE`.

## 9. Interview Questions

**Entry Level**
*   **Q:** What is the difference between DDL and DML?
    *   **A:** DDL (Data Definition Language) commands like `CREATE`, `ALTER`, and `DROP` change the *structure* of the database. DML (Data Manipulation Language) commands like `INSERT`, `UPDATE`, and `DELETE` modify the *data* inside those structures.
*   **Q:** Why is using `SELECT *` considered a bad practice in production?
    *   **A:** It retrieves all columns, wasting network bandwidth and memory for data the application might not need. Furthermore, if a DBA adds a massive new column (like a 5MB image blob) to the table later, the `SELECT *` query will suddenly become incredibly slow.

**Intermediate Level**
*   **Q:** You accidentally executed `DELETE FROM Users;` (without a WHERE clause). What happens, and is it faster or slower than `TRUNCATE TABLE Users;`?
    *   **A:** `DELETE` without a WHERE clause removes all rows, but it does so row-by-row, writing every single deletion into the Transaction Log. It is significantly slower than `TRUNCATE`, which is a DDL command that simply deallocates the physical data pages instantly.

## 10. Preparation for Next Chapter
In Chapter 2, we will master **Filtering and Sorting**. You will learn how to write complex `WHERE` clauses using logical operators (`AND`, `OR`, `IN`, `LIKE`) to pinpoint exact records, and how to format the output using `ORDER BY` and `LIMIT`.
