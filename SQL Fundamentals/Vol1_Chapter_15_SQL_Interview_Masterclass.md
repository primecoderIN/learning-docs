# Volume 1, Chapter 15 – SQL Interview Masterclass

## 1. Concept Overview

Congratulations on reaching the end of **Volume 1: SQL Developer Foundations**. You now possess the syntax and structural knowledge required to build and query relational databases professionally. 

This final chapter serves as an Interview Masterclass. It directly addresses the most frequently asked "Vs" (Comparison) questions that trip up junior and mid-level developers during technical interviews.

---

## 2. The Classic "Vs" Questions

### 1. `DELETE` vs `TRUNCATE` vs `DROP`
*   **`DELETE` (DML):** Removes rows one-by-one. It writes every deletion to the transaction log, making it slow for large tables, but allowing it to be easily rolled back. It fires `DELETE` triggers and can use a `WHERE` clause.
*   **`TRUNCATE` (DDL):** Instantly empties the table by deallocating the physical data pages. It is blazingly fast and uses minimal logging. It *cannot* use a `WHERE` clause (it's all-or-nothing), and it does *not* fire triggers. (It can still be rolled back if wrapped in an explicit transaction).
*   **`DROP` (DDL):** Destroys the table structure entirely. Both the data and the table metadata are wiped from the database.

### 2. `WHERE` vs `HAVING`
*   **`WHERE`:** Filters individual rows *before* any grouping or aggregations occur. It cannot contain aggregate functions (e.g., `WHERE SUM(Price) > 10` is an error).
*   **`HAVING`:** Filters groups *after* aggregations have been calculated. It is almost always used in conjunction with a `GROUP BY` clause. 

### 3. `JOINs` vs `Subqueries`
*   **Performance:** Historically, `JOINs` were always faster than subqueries. In modern databases, the Query Optimizer will often rewrite an uncorrelated subquery into a `JOIN` behind the scenes, making performance identical. However, *Correlated Subqueries* (which execute row-by-row) are generally terrible for performance and should be rewritten as `JOINs`.
*   **Readability:** Subqueries are often easier for beginners to read mathematically. `JOINs` are the industry standard for combining relational data.

### 4. `RANK()` vs `DENSE_RANK()` vs `ROW_NUMBER()`
Imagine three users who scored: 100, 100, and 90.
*   **`ROW_NUMBER()`:** Ignores ties. Assigns exactly: **1, 2, 3**.
*   **`RANK()`:** Acknowledges ties, but skips the next rank. Assigns: **1, 1, 3**. (Nobody gets 2nd place).
*   **`DENSE_RANK()`:** Acknowledges ties and does *not* skip the next rank. Assigns: **1, 1, 2**.

### 5. `CTE` (Common Table Expression) vs `Temporary Table`
*   **CTE:** A logical, temporary result set defined in memory for the duration of a *single* `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement. It cannot be indexed and disappears the moment the query finishes. It is primarily used to make code readable.
*   **Temporary Table (`#Table`):** A physical table stored in `tempdb`. It lasts for the duration of the entire session or stored procedure. You can write multiple independent queries against it, add Primary Keys, and build Indexes on it. It is used for heavy data lifting.

### 6. Clustered vs Non-Clustered Index
*   **Clustered Index:** Dictates the physical order of the data on the hard drive. A table can only have one. (Usually the Primary Key).
*   **Non-Clustered Index:** A separate lookup table (like a book index) containing pointers to the actual data rows. A table can have many.

---

## 3. Query Optimization Scenarios

**Scenario A: The developer writes `SELECT * FROM Users WHERE YEAR(CreatedAt) = 2025`. Why is it slow?**
**Answer:** The query is non-SARGable. By wrapping the `CreatedAt` column in the `YEAR()` function, the Query Optimizer cannot use an index on that column. It must perform a Full Table Scan. 
*Fix:* Rewrite it as `WHERE CreatedAt >= '2025-01-01' AND CreatedAt < '2026-01-01'`.

**Scenario B: A query uses a `LEFT JOIN` to connect `Users` to `Tickets`, but then adds `WHERE Tickets.Price > 50`. What happens?**
**Answer:** The `WHERE` clause accidentally converts the `LEFT JOIN` into an `INNER JOIN`. If a user has no tickets, the `LEFT JOIN` returns a `NULL` price. The `WHERE` clause then checks if `NULL > 50`, evaluates to false, and drops the user entirely. 
*Fix:* Move the condition into the join itself: `LEFT JOIN Tickets ON Users.UserID = Tickets.UserID AND Tickets.Price > 50`.

**Scenario C: You need to insert 10,000 rows into a table that has 15 indexes. It is taking 5 minutes. Why?**
**Answer:** Every single `INSERT` must also update all 15 B-Tree indexes synchronously. The table is over-indexed. 
*Fix:* If it's a nightly batch load, drop or disable the indexes, insert the 10,000 rows, and then rebuild the indexes.

---

## 4. Architectural Concept Questions

*   **Q: What is a View, and can you pass parameters to it?**
    *   **A:** A View is a saved, virtual table representing a query. You *cannot* pass parameters to a standard View. If you need parameterized logic (e.g., `GetTicketsByEvent(5)`), you must use an Inline Table-Valued Function (iTVF) or a Stored Procedure.
*   **Q: What is Normalization?**
    *   **A:** The process of organizing data to reduce redundancy and eliminate insert/update/delete anomalies. (Usually aiming for 3rd Normal Form).
*   **Q: What is an ACID transaction?**
    *   **A:** A transaction that guarantees Atomicity (all-or-nothing), Consistency (rules are followed), Isolation (concurrency control), and Durability (saved permanently).

---

## 5. Next Steps: The 500 Scenario Workbook

You have mastered the developer syntax. To truly test your skills, you must apply them to real-world business problems. 

In **Volume 3**, we will abandon theory and dive entirely into practical application. You will face 500 grueling scenario-based questions spanning five major industry domains (E-Commerce, Finance, Healthcare, Social Media, and Logistics). Get ready to write code.
