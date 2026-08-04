# Volume 1, Chapter 13 – Normalization and Database Design

## 1. Concept Overview

If you sit down at a keyboard and start creating tables without a plan, you will inevitably build a bad database. A poorly designed database suffers from three deadly "Anomalies":
1.  **Update Anomaly:** A user's email address is stored in 5 different rows. If they change their email, you must update it in 5 places. If you miss one, your data is corrupted.
2.  **Delete Anomaly:** You delete a cancelled event from the database, but because the Venue's address was stored in the same row, you accidentally deleted the Venue information permanently.
3.  **Insert Anomaly:** You cannot add a new Venue into the database until an Event is scheduled there, because they share the same table and the Event columns cannot be null.

**Normalization** is the academic process of dividing data into multiple, smaller tables and defining relationships between them to completely eliminate these anomalies.

## 2. First Normal Form (1NF)

To reach 1NF, a table must follow two strict rules:
1.  **Atomic Values:** Every column must hold a single, indivisible value. You cannot store a comma-separated list of tags in a single column (e.g., `Tags: 'Music, Outdoor, VIP'`).
2.  **No Repeating Groups:** You cannot create columns like `Tag1`, `Tag2`, `Tag3` to get around the first rule.

**The Fix:** Create a separate `Tags` table and an `EventTags` mapping table to handle the one-to-many or many-to-many relationships.

## 3. Second Normal Form (2NF)

To reach 2NF, a table must:
1.  Already be in 1NF.
2.  **Have no Partial Dependencies.** (This only applies to tables with Composite Primary Keys).

**Scenario:** You have a mapping table called `TicketSales`. The Composite Primary Key is `(EventID, UserID)`. The table also has a column called `EventCity`.
*   **The Violation:** `EventCity` depends entirely on the `EventID`. It has absolutely nothing to do with the `UserID`. Therefore, it is only "partially dependent" on the Primary Key.
*   **The Fix:** Move `EventCity` out of the mapping table and into the `Events` table.

## 4. Third Normal Form (3NF)

To reach 3NF, a table must:
1.  Already be in 2NF.
2.  **Have no Transitive Dependencies.** (A non-key column cannot depend on another non-key column).

**Scenario:** Your `Events` table has `EventID` (PK), `LocationID`, and `LocationZipCode`.
*   **The Violation:** `LocationZipCode` depends on `LocationID`, not on the `EventID`. `LocationID` is just a foreign key, not the primary key. This is a transitive dependency (Event -> Location -> ZipCode).
*   **The Fix:** Move `LocationZipCode` into the `Locations` table. 

*(A common memory trick for 3NF is the DBA oath: "Every column must depend on the key, the whole key, and nothing but the key, so help me Codd.")*

## 5. Boyce-Codd Normal Form (BCNF)

BCNF is essentially a slightly stricter version of 3NF. It handles rare edge cases where a table has overlapping candidate keys. For most working developers, if a database is in 3NF, it is generally considered fully normalized for OLTP (Online Transaction Processing) workloads.

## 6. Denormalization (Breaking the Rules)

If Normalization is so perfect, why do Data Architects sometimes intentionally violate it?
**Performance.**

When a database is perfectly normalized into 3NF, retrieving a simple user receipt might require a 6-table `INNER JOIN`. If you have a billion rows, 6-table joins take significant CPU time. 

**Denormalization** is the deliberate process of adding redundant data back into a table to speed up read operations. 
*   **Example:** Storing `TotalTicketsSold` directly on the `Events` table. Technically, this violates normalization (you should just run a `SUM()` query on the `Tickets` table). But storing the hardcoded number makes the web page load instantly. You must then use Triggers or Stored Procedures to keep that redundant number perfectly synced.

Denormalization is the foundation of OLAP (Online Analytical Processing) and Data Warehouses (like Snowflake or Redshift), where data is heavily duplicated into "Star Schemas" so analysts can query it lightning-fast without complex joins.

## 7. Hands-on Exercises

Look at this badly designed table:
`Table: EmployeeProjects (EmpID, ProjectID, EmpName, ProjectName, HourlyRate)`
*(Assume EmpID + ProjectID is the Composite Primary Key).*

1. Does this table violate 2NF? Why? (Hint: Does `EmpName` depend on the `ProjectID`?)
2. How would you redesign this single table into three separate tables to achieve 3NF? Write down the table names and their columns.

## 8. Interview Questions

**Entry Level**
*   **Q:** What is Normalization?
    *   **A:** Normalization is the process of organizing data in a database to reduce redundancy and eliminate insertion, update, and deletion anomalies. It involves breaking large tables into smaller ones and linking them with foreign keys.
*   **Q:** What is the rule for First Normal Form (1NF)?
    *   **A:** All columns must contain atomic (indivisible) values, and there can be no repeating groups or arrays stored in a single column.

**Intermediate Level**
*   **Q:** Explain the difference between 2NF and 3NF.
    *   **A:** 2NF deals with Composite Primary Keys; it states that no non-key column can depend on only *part* of a composite key. 3NF deals with all keys; it states that no non-key column can depend on *another* non-key column (a transitive dependency).
*   **Q:** Why would you intentionally Denormalize a database?
    *   **A:** To improve read performance. In highly normalized databases, complex queries require many expensive `JOINs`. By denormalizing (duplicating) data into a single table, you can eliminate joins and drastically speed up `SELECT` queries, usually at the cost of slower `INSERT/UPDATE` operations and requiring extra logic to keep the duplicated data in sync.

## 9. Preparation for Next Chapter
In Chapter 14, we will cover **Advanced SQL and Performance**. We will look at how to pivot data, write Dynamic SQL safely, use Temporary Tables, and analyze Query Execution Plans to understand why your query is running slowly and how to fix it.
