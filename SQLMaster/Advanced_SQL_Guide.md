# Advanced SQL & Referential Integrity Guide

Welcome to your masterclass on SQL! We will cover everything from how relational databases handle connected data (Referential Actions) to advanced querying techniques that senior engineers use daily.

---

## Part 1: Referential Actions (Cascade, Restrict, Set Null)

When two tables are connected via a **Foreign Key** (e.g., an `Event` belongs to an `Organization`), the database needs rules on what to do if the parent record is deleted or updated. These rules are called **Referential Actions**.

### 1. CASCADE
- **What it is:** If the parent record is deleted, automatically delete all related child records.
- **Scenario:** You have an `Organization` and `OrganizationRoles`. If the organization goes out of business and gets deleted from the database, the roles associated with it are completely useless.
- **Example:**
```sql
CREATE TABLE OrganizationRoles (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    OrganizationId UNIQUEIDENTIFIER,
    Name VARCHAR(80),
    FOREIGN KEY (OrganizationId) REFERENCES Organizations(Id) ON DELETE CASCADE
);
-- If you run: DELETE FROM Organizations WHERE Id = '123';
-- SQL Server will automatically delete all OrganizationRoles where OrganizationId = '123'.
```

### 2. RESTRICT (or NO ACTION)
- **What it is:** If the parent record has child records, **block** the deletion of the parent. An error is thrown. (Note: `RESTRICT` and `NO ACTION` behave almost identically in SQL Server).
- **Scenario:** A `User` creates an `Organization`. You do not want to allow an Admin to accidentally delete a `User` from the database if they still own organizations.
- **Example:**
```sql
CREATE TABLE Organizations (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    OwnerUserId NVARCHAR(450),
    Name VARCHAR(160),
    FOREIGN KEY (OwnerUserId) REFERENCES AspNetUsers(Id) ON DELETE NO ACTION
);
-- If you run: DELETE FROM AspNetUsers WHERE Id = 'abc';
-- SQL Server will throw an error: "The DELETE statement conflicted with the REFERENCE constraint..."
```

- **Another Scenario (Multiple Cascade Paths Error):** In SQL Server, if you have multiple foreign keys that can cascade to the same table, it will throw a "multiple cascade paths" error. To fix this, you must change one of the relationships from `CASCADE` to `RESTRICT` (or `NO ACTION`).
- **Example (Fixing the Multiple Cascade Paths Error):**
```sql
CREATE TABLE OrganizationMemberRoles (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    MemberId UNIQUEIDENTIFIER,
    OrganizationRoleId UNIQUEIDENTIFIER,
    -- If we used CASCADE here, it might conflict with other delete paths.
    -- We use RESTRICT to break the cycle and avoid the error.
    FOREIGN KEY (OrganizationRoleId) REFERENCES OrganizationRoles(Id) ON DELETE RESTRICT 
);
```

### 3. SET NULL
- **What it is:** If the parent record is deleted, keep the child record, but set its Foreign Key column to `NULL`.
- **Scenario:** An `Event` belongs to a `Category` (e.g., "Music"). If an admin deletes the "Music" category, you don't want to delete the actual Event! You just want the Event to have *no* category until it's reassigned.
- **Example:**
```sql
CREATE TABLE Events (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    Title VARCHAR(200),
    CategoryId UNIQUEIDENTIFIER NULL,
    FOREIGN KEY (CategoryId) REFERENCES Categories(Id) ON DELETE SET NULL
);
-- If you run: DELETE FROM Categories WHERE Id = 'music-id';
-- The Event's CategoryId becomes NULL.
```

---

## Part 2: Hard SQL Topics Masterclass

Here are advanced techniques every senior database developer should know, complete with examples.

### A. Window Functions (OVER, PARTITION BY, ROW_NUMBER)

- **What it is:** Window functions allow you to perform calculations across a set of rows related to the current row, *without collapsing the rows* like `GROUP BY` does.
- **Scenario:** You want to find the **top 2 highest-selling events for EVERY organization**.
- **The Wrong Way:** Using `GROUP BY` only gives you aggregates (like MAX), it doesn't give you the actual row details.
- **The Right Way:**
```sql
SELECT OrganizationId, EventTitle, TicketsSold
FROM (
    SELECT 
        OrganizationId, 
        Title as EventTitle, 
        TicketsSold,
        -- Assign a rank 1, 2, 3... to each event INSIDE each organization
        ROW_NUMBER() OVER (PARTITION BY OrganizationId ORDER BY TicketsSold DESC) as Rank
    FROM Events
) as RankedEvents
WHERE Rank <= 2; -- Only keep the top 2 per organization!
```

### B. Common Table Expressions (CTEs) & Recursion

- **What it is:** A CTE (`WITH` clause) is like a temporary named result set. A **Recursive CTE** can reference itself, which is perfect for hierarchical data.
- **Scenario:** You have a `Categories` table where categories can have a `ParentCategoryId` (e.g., Electronics -> Computers -> Laptops). You want to get the full breadcrumb path for "Laptops".
- **The Table:** `Categories (Id, Name, ParentCategoryId)`
- **The Solution:**
```sql
WITH CategoryHierarchy AS (
    -- 1. Base case: Start with the target category
    SELECT Id, Name, ParentCategoryId, 1 as Level, Name as Path
    FROM Categories 
    WHERE Name = 'Laptops'
    
    UNION ALL
    
    -- 2. Recursive step: Join the CTE to the Categories table to find parents
    SELECT c.Id, c.Name, c.ParentCategoryId, ch.Level + 1, c.Name + ' > ' + ch.Path
    FROM Categories c
    INNER JOIN CategoryHierarchy ch ON ch.ParentCategoryId = c.Id
)
-- 3. Get the top-level parent's constructed path
SELECT Path FROM CategoryHierarchy ORDER BY Level DESC;
-- Result: "Electronics > Computers > Laptops"
```

### C. CROSS APPLY (The SQL Server Superweapon)

- **What it is:** `CROSS APPLY` acts like an `INNER JOIN`, but it allows you to evaluate a subquery or table-valued function *for each row* of the outer table. 
- **Scenario:** You want a list of all Organizations, alongside the exact `Title` and `Date` of their *most recent* event. (Trying to do this with standard `JOIN` and `GROUP BY` is notoriously difficult).
- **The Solution:**
```sql
SELECT o.Name AS OrgName, LatestEvent.Title, LatestEvent.Date
FROM Organizations o
CROSS APPLY (
    -- This subquery runs once for EVERY organization row
    SELECT TOP 1 Title, Date 
    FROM Events e 
    WHERE e.OrganizationId = o.Id 
    ORDER BY Date DESC
) AS LatestEvent;
```

### D. PIVOT (Rows to Columns)

- **What it is:** `PIVOT` rotates your data, turning unique values from one column into multiple columns.
- **Scenario:** You want a sales report where the rows are Organizations, and the columns are the Months (Jan, Feb, Mar), showing total events created.
- **The Solution:**
```sql
SELECT OrgName, [1] AS Jan, [2] AS Feb, [3] AS Mar
FROM (
    SELECT o.Name AS OrgName, MONTH(e.Date) as EventMonth, e.Id as EventId
    FROM Organizations o
    JOIN Events e ON e.OrganizationId = o.Id
) AS SourceTable
PIVOT (
    COUNT(EventId) -- What to aggregate
    FOR EventMonth IN ([1], [2], [3]) -- The column headers to create
) AS PivotTable;
```

### E. SARGability (Search-ARGument-ABLE)

- **What it is:** A query is "SARGable" if the database engine can actually use indexes to find the data. If you wrap a column in a function, the database has to scan the *entire* table, ignoring indexes.
- **Scenario:** You want to find all events happening in the year 2026.
- **The BAD Way (Not SARGable - Ignores Indexes):**
```sql
-- The DB must run the YEAR() function on EVERY row before comparing to 2026. (Table Scan)
SELECT * FROM Events WHERE YEAR(Date) = 2026; 
```
- **The GOOD Way (SARGable - Uses Indexes):**
```sql
-- The DB can immediately jump to the index where dates start at 2026-01-01. (Index Seek)
SELECT * FROM Events WHERE Date >= '2026-01-01' AND Date < '2027-01-01';
```

---

# SQL Foreign Keys & REFERENCES – Complete Summary

## 1. Primary Key (PK)

* Uniquely identifies each row in a table.
* Cannot contain duplicate or NULL values.
* Used as the parent key for relationships.

**Example:**
```sql
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    Username NVARCHAR(50) NOT NULL
);
```

## 2. Foreign Key (FK)

* A column (or columns) in a child table that references a Primary Key or Unique Key in a parent table.
* Maintains **referential integrity** by ensuring referenced parent records exist.

**Example:**
```sql
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    UserId UNIQUEIDENTIFIER,
    FOREIGN KEY (UserId) REFERENCES Users(Id)
);
```

## 3. REFERENCES

* Used to define a foreign key relationship.
* Can be declared inline or separately using the `FOREIGN KEY` constraint.

**Example (Inline):**
```sql
CREATE TABLE UserProfiles (
    ProfileId INT PRIMARY KEY,
    UserId UNIQUEIDENTIFIER REFERENCES Users(Id) -- Inline declaration
);
```

## 4. Relationship Types

### One-to-One (1:1)

* One parent record relates to one child record.
* Implemented using a Foreign Key with a `UNIQUE` constraint.

**Example:** User ↔ Passport
```sql
CREATE TABLE Passports (
    PassportNumber VARCHAR(20) PRIMARY KEY,
    UserId UNIQUEIDENTIFIER UNIQUE, -- UNIQUE ensures One-to-One
    FOREIGN KEY (UserId) REFERENCES Users(Id)
);
```

---

### One-to-Many (1:N)

* One parent can have many child records.
* Most common relationship.

**Example:** Customer → Orders
```sql
CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY,
    Name NVARCHAR(100)
);

CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);
```

---

### Many-to-Many (M:N)

* Many parent records relate to many child records.
* Implemented using a **junction (bridge) table** containing two foreign keys.
* The junction table usually has a **composite primary key**.

**Example:** Students ↔ Courses
```sql
CREATE TABLE Students (StudentId INT PRIMARY KEY);
CREATE TABLE Courses (CourseId INT PRIMARY KEY);

CREATE TABLE StudentCourses (
    StudentId INT,
    CourseId INT,
    PRIMARY KEY (StudentId, CourseId),
    FOREIGN KEY (StudentId) REFERENCES Students(StudentId),
    FOREIGN KEY (CourseId) REFERENCES Courses(CourseId)
);
```

---

### Self-Referencing Relationship

* A table references itself.
* Used for hierarchical data.

**Examples:**

* Employee → Manager
* Category → Parent Category

**Example:**
```sql
CREATE TABLE Employees (
    EmployeeId INT PRIMARY KEY,
    Name NVARCHAR(100),
    ManagerId INT NULL,
    FOREIGN KEY (ManagerId) REFERENCES Employees(EmployeeId)
);
```

---

## 5. Composite Foreign Key

* A foreign key made up of multiple columns.
* References a composite primary key in another table.
* Common in junction tables.

**Example:**
```sql
CREATE TABLE EmployeeRoles (
    DepartmentId INT,
    RoleId INT,
    PRIMARY KEY (DepartmentId, RoleId)
);

CREATE TABLE ProjectAssignments (
    ProjectId INT,
    DepartmentId INT,
    RoleId INT,
    FOREIGN KEY (DepartmentId, RoleId) REFERENCES EmployeeRoles(DepartmentId, RoleId)
);
```

---

# Foreign Key Actions

## ON DELETE CASCADE

* Deleting a parent automatically deletes all related child records.
* Best when child data cannot exist without the parent.

**Example**

* Delete Customer
* All Orders for that customer are deleted automatically.
```sql
FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId) ON DELETE CASCADE
```

---

## ON DELETE SET NULL

* Deletes the parent.
* Child records remain.
* Foreign key is set to `NULL`.
* Foreign key column must allow `NULL`.

**Use when**

* Child data is still valuable without the parent.

**Example:**
```sql
FOREIGN KEY (CategoryId) REFERENCES Categories(CategoryId) ON DELETE SET NULL
```

---

## ON DELETE NO ACTION (Default in SQL Server)

* Prevents deleting a parent if child records exist.
* Parent can only be deleted after child records are removed or updated.

**Use when**

* Data integrity is more important than convenience.

**Example:**
```sql
FOREIGN KEY (OwnerId) REFERENCES Users(Id) ON DELETE NO ACTION
```

---

## ON DELETE RESTRICT

* Prevents deleting a parent when child records exist.
* Similar behavior to `NO ACTION`.
* **Not supported in SQL Server.**
* Supported in MySQL and PostgreSQL.

**Example:**
```sql
FOREIGN KEY (OrganizationRoleId) REFERENCES OrganizationRoles(Id) ON DELETE RESTRICT
```

---

## ON UPDATE CASCADE

* Updates child foreign keys automatically when the parent key changes.

**Example:**
```sql
FOREIGN KEY (UsernameId) REFERENCES Usernames(Id) ON UPDATE CASCADE
```

---

## ON UPDATE NO ACTION

* Prevents updating the parent key if child records reference it.

---

## ON UPDATE SET NULL

* Parent key changes.
* Child foreign key becomes `NULL`.

---

## ON UPDATE/DELETE SET DEFAULT

* Child foreign key is set to its default value.
* Requires a default constraint on the foreign key column.
* Supported in SQL Server.

**Example:**
```sql
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    StatusId INT DEFAULT 1,
    FOREIGN KEY (StatusId) REFERENCES Statuses(Id) ON DELETE SET DEFAULT
);
```

---

# SQL Server Supported Referential Actions

| Action      | Supported |
| ----------- | --------- |
| NO ACTION   | ✅         |
| CASCADE     | ✅         |
| SET NULL    | ✅         |
| SET DEFAULT | ✅         |
| RESTRICT    | ❌         |

---

# NO ACTION vs RESTRICT

### NO ACTION

* SQL Server default.
* Constraint is checked when the statement completes.
* In SQL Server, behaves like `RESTRICT`.

### RESTRICT

* Checks immediately before the operation.
* Not available in SQL Server.
* Available in MySQL and PostgreSQL.

**For most practical scenarios, both prevent deleting or updating a parent while child records exist.**

---

# Referential Integrity

Foreign keys ensure:

* No child record references a non-existent parent.
* Relationships remain valid.
* Accidental orphan records are prevented.
* Database consistency is maintained.

---

# Best Practices

* Use **CASCADE** only for dependent data (e.g., Order → Order Items).
* Use **SET NULL** when child records should be retained after the parent is removed.
* Use **NO ACTION** to prevent accidental deletion or updates of parent records.
* Avoid updating primary key values; prefer immutable keys such as Identity columns or GUIDs.
* Index foreign key columns for better JOIN and lookup performance.
* Use composite keys only when they naturally represent the business relationship.
* Use self-referencing foreign keys for hierarchical structures.
* Choose the foreign key action based on business rules rather than convenience.

---

# Common Real-World Examples

| Relationship         | Type                          |
| -------------------- | ----------------------------- |
| Customer → Orders    | One-to-Many                   |
| Order → Order Items  | One-to-Many (often `CASCADE`) |
| User → Passport      | One-to-One                    |
| Student ↔ Course     | Many-to-Many                  |
| Employee → Manager   | Self-Referencing              |
| Product ↔ Category   | Many-to-One                   |
| Invoice → Customer   | One-to-Many                   |
| Blog Post → Comments | One-to-Many                   |

---

# Key Interview Takeaways

* **Primary Key:** Uniquely identifies a row.
* **Foreign Key:** References a parent table to enforce relationships.
* **REFERENCES:** Defines the parent-child relationship.
* **CASCADE:** Automatically propagates deletes/updates to child records.
* **SET NULL:** Preserves child records by clearing the reference.
* **NO ACTION:** Prevents deleting or updating a parent with existing child records (default in SQL Server).
* **RESTRICT:** Similar to `NO ACTION` but not supported in SQL Server.
* **SET DEFAULT:** Sets the foreign key to its default value.
* **Composite Foreign Key:** References a composite primary key.
* **Self-Referencing Foreign Key:** Used for hierarchical data.
* **Referential Integrity:** Ensures relationships remain consistent and valid.

---
*Generated by Gemini for NextEvent Developer Onboarding.*

