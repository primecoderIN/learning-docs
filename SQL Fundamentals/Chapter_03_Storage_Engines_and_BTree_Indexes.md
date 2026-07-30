# Chapter 3 – Storage Engines, Pages, and the B-Tree Index Architecture

## 1. Concept Overview

A database is not a magic black box; it is fundamentally a sophisticated file manager. The **Storage Engine** is the component responsible for taking logical data (rows and columns) and physically writing it to spinning disks (HDD) or solid-state drives (SSD). 

Because reading from disk (I/O) is astronomically slower than reading from RAM, storage engines organize data into fixed-size chunks called **Pages** (or **Blocks**). When a database needs a single row, it never reads just that row; it reads the entire Page containing that row from the disk into memory.

To avoid scanning every single Page to find a specific row (a **Table Scan**), databases use **Indexes**. The most ubiquitous index structure in relational databases is the **B-Tree (Balanced Tree)**. A B-Tree allows the database engine to find a specific row among billions of rows in a matter of milliseconds by traversing a hierarchical tree structure with logarithmic time complexity: **O(log N)**.

## 2. History

In the 1960s, data was stored sequentially on magnetic tape. Finding a record meant reading the tape from start to finish. In 1970, Rudolf Bayer and Edward M. McCreight introduced the **B-Tree** at Boeing Scientific Research Labs. The B-Tree was revolutionary because it was self-balancing; it guaranteed that every path from the root of the tree to a leaf node was the exact same length, ensuring consistent, highly predictable read performance regardless of how large the dataset grew. This mathematical structure became the backbone of modern RDBMS storage.

## 3. Real-world analogy

Imagine a massive **Telephone Directory** with 10 million names.

*   **Heap / Table Scan:** If the names were printed in completely random order, and you needed to find "John Smith," you would have to read every single page from cover to cover.
*   **Clustered Index:** The phone book itself is naturally sorted alphabetically by Last Name, then First Name. You don't read page by page. You jump to 'S', then 'Sm', then 'Smi', finding John Smith in seconds. The data *itself* is sorted.
*   **Non-Clustered (Secondary) Index:** Now suppose you only know John Smith's phone number, but not his name. The alphabetical sorting doesn't help. You need a secondary index at the back of the book sorted purely by Phone Number. You look up the number, and it gives you a pointer (a page number) telling you exactly where John Smith's full record is located in the main book.

## 4. Business problem solved

As enterprise tables grow to millions or billions of rows (e.g., a `Tickets` table), querying data without indexes causes CPU and I/O exhaustion. A query that takes 15 minutes via a Table Scan can be reduced to 3 milliseconds with a proper B-Tree index. Indexes solve the problem of physical data retrieval latency.

---

## 5. Microsoft SQL Server explanation

SQL Server stores data in an MDF (Primary Data File) or NDF (Secondary Data File). The fundamental unit of storage is an **8KB Page**. Eight contiguous pages form a **64KB Extent**.

SQL Server uses a strictly **Clustered Index Architecture**:
*   **Clustered Index:** This dictates the physical sorting of the data pages on disk. The leaf nodes of the B-Tree *are* the actual data pages. A table can only have ONE Clustered Index. If a table does not have a Clustered Index, it is called a **Heap** (highly discouraged in SQL Server).
*   **Non-Clustered Index:** A separate B-Tree structure. The leaf nodes do not contain the full row data; instead, they contain the index key columns and a pointer. If the table has a Clustered Index, this pointer is the **Clustering Key** (usually the Primary Key).

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Create a table (Primary Key automatically creates a Clustered Index)
CREATE TABLE Core.Tickets (
    TicketID UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID() PRIMARY KEY, -- Clustered Index
    EventID UNIQUEIDENTIFIER NOT NULL,
    UserID INT NOT NULL,
    PurchaseDate DATETIME2 NOT NULL,
    Price DECIMAL(10,2) NOT NULL
);
GO

-- 2. Create a Non-Clustered Index for fast lookups by Event
CREATE NONCLUSTERED INDEX IX_Tickets_EventID 
ON Core.Tickets (EventID);
GO

-- 3. Create a Covering Index (Includes additional columns to prevent Key Lookups)
CREATE NONCLUSTERED INDEX IX_Tickets_UserID_Covering 
ON Core.Tickets (UserID)
INCLUDE (PurchaseDate, Price);
GO
```

## 7. SQL Server internals

An 8KB page in SQL Server has a 96-byte header containing metadata (Page ID, Object ID, LSN). Following the header are the actual data rows. At the very end of the page is the **Row Offset Array**, which points backwards to where each row starts on the page.

When an Index Seek occurs:
1. SQL Server reads the **Root Page** of the B-Tree.
2. It evaluates the key and follows a pointer to an **Intermediate Page**.
3. It follows the pointer down to the **Leaf Page**.
4. If it's a Clustered Index, the Leaf Page contains the actual row.
5. If it's a Non-Clustered Index, and the query requires columns *not* in the index, SQL Server must take the Clustering Key from the leaf node and perform a **Key Lookup** against the Clustered Index to retrieve the missing columns. Key Lookups are extremely expensive for large result sets.

## 8. SQL Server execution

Executing `SELECT Price FROM Core.Tickets WHERE EventID = 'A1...';`
1. The Optimizer chooses `IX_Tickets_EventID`.
2. The Storage Engine navigates the Non-Clustered B-Tree to find `EventID = 'A1...'`.
3. The leaf node contains the `TicketID` (Clustering Key).
4. The Relational Engine realizes it needs the `Price` column, which is not in the index.
5. It performs a **Key Lookup**, traversing the Clustered Index using the `TicketID`.
6. It retrieves the `Price` from the data page.

*(Note: If we used the Covering Index syntax with `INCLUDE (Price)`, step 5 is eliminated entirely, achieving a massive performance boost).*

## 9. SQL Server enterprise examples

*   **Financial Trading Platforms:** High-frequency trading applications rely heavily on `INCLUDE` columns in SQL Server to create completely "Covering Indexes." This guarantees that the read operations only ever hit the Non-Clustered Index pages and never hit the base table, minimizing I/O and locking contention.
*   **Data Warehouses:** Enterprise data warehouses utilize SQL Server's **Columnstore Indexes**, where data is grouped and compressed by column rather than by row. This is outside the standard B-Tree paradigm but critical for aggregations (e.g., `SUM(Price)` over a billion rows).

## 10. SQL Server performance considerations

*   **Page Splits:** If you insert a row with a value that belongs in the middle of a full 8KB page, SQL Server must allocate a new page and move half the rows to the new page. This is a **Page Split**. It is extremely CPU and I/O intensive and causes logical fragmentation. This is why random UUIDs (`UNIQUEIDENTIFIER` without `NEWSEQUENTIALID`) are terrible for Clustered Indexes.
*   **Fill Factor:** You can instruct SQL Server to leave a percentage of every page empty (e.g., `FILLFACTOR = 80`) to accommodate future inserts and prevent Page Splits.

## 11. SQL Server security considerations

*   **Transparent Data Encryption (TDE):** When TDE is enabled, SQL Server encrypts the 8KB pages as they are written to disk and decrypts them as they are read into memory. The B-Tree structure in memory remains unencrypted, meaning indexing performance is largely unaffected, but the physical disk files (.mdf/.ndf) are secure against theft.

## 12. SQL Server common mistakes

*   **Over-Indexing:** Creating an index on every single column. Every time a row is inserted, updated, or deleted, SQL Server must synchronously update every single B-Tree. Too many indexes will cripple `INSERT` performance.
*   **Ignoring SARGability:** Writing queries like `WHERE YEAR(PurchaseDate) = 2026`. Because a function is applied to the column, SQL Server cannot use the B-Tree index (it cannot navigate the tree). It must perform an Index Scan. This is called a non-SARGable (Search Argument) query.

## 13. SQL Server best practices

*   Always define a highly selective, sequential, narrow Clustered Index (like `INT IDENTITY`).
*   Use `INCLUDE` columns to cover frequent queries and eliminate Key Lookups.
*   Regularly monitor and rebuild/reorganize heavily fragmented indexes using `sys.dm_db_index_physical_stats`.

---

## 14. PostgreSQL explanation

PostgreSQL's storage architecture differs significantly from SQL Server. Postgres uses a **Heap Architecture**.
In Postgres, the table data is *always* stored in a Heap (an unordered collection of 8KB blocks). 

Postgres does **not** have the concept of a Clustered Index where the base table is physically ordered as a B-Tree. When you create a `PRIMARY KEY` in Postgres, it creates a unique B-Tree index, but the leaf nodes of this index do not contain the row data. Instead, they contain a **CTID (Tuple Identifier)**. 
A CTID looks like `(BlockNumber, TupleIndex)`. It is a direct physical pointer to the exact block and offset in the Heap where the row lives.

## 15. PostgreSQL syntax

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. Create a table (Primary Key creates a unique B-Tree index pointing to the Heap)
CREATE TABLE core.tickets (
    ticket_id UUID DEFAULT gen_random_uuid() PRIMARY KEY, 
    event_id UUID NOT NULL,
    user_id INT NOT NULL,
    purchase_date TIMESTAMPTZ NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

-- 2. Create a Secondary B-Tree Index
CREATE INDEX idx_tickets_event_id 
ON core.tickets (event_id);

-- 3. Create a Covering Index (Index-Only Scan)
CREATE INDEX idx_tickets_user_id_covering 
ON core.tickets (user_id) 
INCLUDE (purchase_date, price);
```

## 16. PostgreSQL internals

Postgres blocks are also 8KB. A block contains a Page Header, Item Pointers (similar to SQL Server's Row Offset Array), and the Tuples (Rows).

Because Postgres uses MVCC, multiple versions of the same row can exist in the Heap simultaneously. The B-Tree index points to all versions. When an Index Scan finds a CTID, the Executor goes to the Heap block and evaluates the `xmin` and `xmax` transaction IDs on the tuple to determine if that specific version of the row is visible to the current transaction.

**TOAST (The Oversized-Attribute Storage Technique):** If a row exceeds 8KB (e.g., storing a massive JSON payload), Postgres transparently compresses it and moves the large columns to a hidden TOAST table, leaving a small pointer in the main Heap.

## 17. PostgreSQL execution

Executing `SELECT price FROM core.tickets WHERE event_id = 'A1...';`
1. The Planner chooses an **Index Scan** on `idx_tickets_event_id`.
2. The Executor traverses the B-Tree and finds the leaf node.
3. The leaf node contains a CTID (e.g., Block 500, Item 3).
4. The Executor performs a Heap fetch: It reads Block 500 from Shared Buffers.
5. It evaluates MVCC rules to ensure the row is visible.
6. It extracts the `price` column.

*(Note: If using the Covering Index, Postgres performs an **Index-Only Scan**, skipping the Heap fetch entirely, provided the Visibility Map indicates all tuples on the page are visible to everyone).*

## 18. PostgreSQL enterprise examples

*   **Geospatial (PostGIS):** Postgres allows completely different index types. Instead of B-Trees, geographic mapping systems use **GiST (Generalized Search Tree)** indexes to query "venues within 5 miles of coordinates X, Y". 
*   **Full-Text Search:** Platforms like Reddit use Postgres **GIN (Generalized Inverted Index)** to perform lightning-fast searches across arrays and massive text documents, operations where B-Trees are useless.

## 19. PostgreSQL performance considerations

*   **Bitmap Index Scans:** If a query will return many rows via an index, fetching from the Heap row-by-row is inefficient. Postgres will dynamically create an in-memory Bitmap representing the Heap blocks. It sorts the bitmap, and then reads the Heap blocks sequentially. This transforms random I/O into sequential I/O, providing a massive performance boost over SQL Server's traditional Key Lookups.
*   **Index Bloat:** Because of MVCC, when a row is updated, the new version gets a new CTID in the Heap. The B-Tree must be updated to point to the new CTID, while keeping the old pointer for concurrent readers. This causes B-Trees to bloat heavily over time, requiring `REINDEX CONCURRENTLY` maintenance.

## 20. PostgreSQL security considerations

*   Unlike SQL Server's TDE, native PostgreSQL does not currently support cluster-wide Transparent Data Encryption. Enterprise deployments rely on OS-level volume encryption (LUKS) or cloud-provider encryption (AWS EBS Encryption) to secure the underlying 8KB blocks on disk.

## 21. PostgreSQL common mistakes

*   **Using `CLUSTER` command carelessly:** Postgres has a `CLUSTER` command that physically rewrites the Heap to match the order of an index. However, unlike SQL Server's Clustered Index, this is a **one-time operation**. Future inserts will not maintain this order. Furthermore, it takes an exclusive lock on the table, blocking all reads and writes.
*   **Failing to tune `random_page_cost`:** By default, Postgres assumes random disk reads are 4x slower than sequential reads (optimized for old HDDs). On modern NVMe SSDs, a DBA must lower `random_page_cost` to 1.1; otherwise, the Planner will refuse to use B-Tree indexes and perform Table Scans instead.

## 22. PostgreSQL best practices

*   Utilize Partial Indexes: `CREATE INDEX idx_active_users ON users (email) WHERE is_active = true;`. This creates a tiny, incredibly fast B-Tree containing only active users, saving massive amounts of RAM and disk space.
*   Monitor Index Usage: Query `pg_stat_user_indexes`. If `idx_scan` is 0 over a month, drop the index. Unused indexes drain `INSERT`/`UPDATE` performance for no benefit.

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Base Table Storage** | Clustered Index (B-Tree) | Heap | SQL Server keeps data sorted automatically. Postgres relies on the Heap + CTID pointers. |
| **Index Lookups** | Key Lookup (uses Clustered Key) | Heap Fetch (uses CTID) | SQL Server traverses a 2nd B-Tree for lookups. Postgres jumps directly to the physical block (faster, but susceptible to bloat). |
| **Massive Lookups** | Table Scan or repeated Lookups | Bitmap Index Scan | Postgres dynamically optimizes random I/O into sequential I/O in memory. |
| **Partial Indexes** | Filtered Indexes | Partial Indexes | Both support indexing a subset of data (e.g., `WHERE DeletedAt IS NULL`). |

## 24. Architect recommendations

**The SARGability Rule:** 
As an Architect, you must rigorously review developer queries for SARGability (Search Argument-able).
*   **Bad:** `SELECT * FROM Events WHERE CONVERT(VARCHAR, StartDate, 112) = '20261012'` (Causes an Index Scan; the CPU must evaluate the function on every single row).
*   **Good:** `SELECT * FROM Events WHERE StartDate >= '2026-10-12' AND StartDate < '2026-10-13'` (Causes an Index Seek; navigates the B-Tree directly).
If a query is not SARGable, the B-Tree index is useless.

## 25. DBA recommendations

*   **SQL Server DBAs:** Beware of widespread Page Splits. If a table experiences heavy random inserts, rebuild the indexes with a lower Fill Factor (e.g., 80%) to leave breathing room on the 8KB pages.
*   **Postgres DBAs:** Beware of HOT (Heap-Only Tuples) update failures. If an update changes an indexed column, Postgres cannot use HOT updates, causing heavy index bloat. Keep indexes narrow and avoid indexing highly volatile columns (like `last_login_timestamp`).

## 26. Developer recommendations

*   Never use `SELECT *` in production code. Always specify explicit columns (`SELECT Title, StartDate`). If you `SELECT *`, you guarantee that the database cannot use a Covering Index, forcing an expensive Key Lookup / Heap Fetch for every row.
*   The order of columns in a Composite Index matters immensely. An index on `(LastName, FirstName)` can satisfy a search for `LastName = 'Smith'`, but it **cannot** satisfy a search for `FirstName = 'John'`. Lead with the most highly queried, selective column.

## 27. Production case study

**The NextEvent Ticket Drop Catastrophe**

*Scenario:* A highly anticipated concert went on sale. Within seconds, 100,000 users rushed the system to check ticket availability. The query executed was:
`SELECT COUNT(*) FROM core.tickets WHERE event_id = 'A1' AND status = 'AVAILABLE';`

*Failure:* The database CPU hit 100% instantly, and the application timed out. 
*RCA:* The developer had created an index on `(status)`. Because 99% of tickets in the database were 'AVAILABLE', the index was not selective. The Postgres Planner realized the index was useless and performed a Sequential Scan (Table Scan) of the 50-million row `tickets` table 100,000 times concurrently.

*Architectural Fix:* We killed the blocking queries and deployed a composite index: `CREATE INDEX idx_tickets_event_status ON core.tickets (event_id, status)`. The query time dropped from 15 seconds to 0.1 milliseconds. The B-Tree allowed the engine to jump directly to the specific event and count the statuses without reading the rest of the table.

## 28. ASCII diagrams wherever helpful

**B-Tree Index Architecture (SQL Server Non-Clustered / Postgres Secondary)**

```text
                             [ ROOT PAGE ]
                             (Values: M)
                            /           \
                           /             \
            [ INTERMEDIATE PAGE ]       [ INTERMEDIATE PAGE ]
            (Values: C, G)              (Values: R, W)
             /      |      \             /      |      \
            /       |       \           /       |       \
   [ LEAF ]     [ LEAF ]    [ LEAF ] [ LEAF ] [ LEAF ]   [ LEAF ]
   (A, B)       (D, E, F)   (H, J, L)(N, P, Q)(S, T, U)  (X, Y, Z)
      |             |           |        |        |          |
      v             v           v        v        v          v
  Points to     Points to    ...      ...      ...       Points to
  Base Row      Base Row                                 Base Row
 (Clustered Key  (CTID in                                (Clustered Key
  or CTID)        Postgres)                              or CTID)
```
*Notice how searching for 'E' takes exactly 3 reads (Root -> Intermediate -> Leaf), regardless of whether the table has 100 rows or 100 million rows.*

## 29. Enterprise design discussion

**UUIDs vs. Integers for Primary Keys**

This is the most debated topic in storage architecture.
*   **The Integer (IDENTITY/SERIAL):** Generates 1, 2, 3, 4. 
    *   *Pros:* Perfect for B-Trees. Always inserts at the end of the tree. Zero page splits. High performance. Narrow (4 bytes).
    *   *Cons:* Predictable (IDOR security risk). Hard to merge databases (ID collisions).
*   **The V4 UUID:** Completely random string of 32 hex characters.
    *   *Pros:* Globally unique. Safe from guessing.
    *   *Cons:* Devastating to B-Tree inserts. Because they are random, a new insert must be placed randomly in the middle of the B-Tree, forcing the database to split the 8KB page in half constantly. Very wide (16 bytes), bloating all foreign keys.

*Enterprise Compromise:*
1.  **SQL Server:** Use `UNIQUEIDENTIFIER` but set the default to `NEWSEQUENTIALID()`. This generates a UUID that is tied to the MAC address and time, ensuring it is sequential at the database level, preventing page splits.
2.  **Postgres:** Standard V4 UUIDs cause massive bloat. Use the new **UUIDv7** standard (which prefixes the UUID with a Unix timestamp, making it naturally sequential) or use a custom generator function.

## 30. Hands-on exercises

1. In SQL Server, create a table with a `UNIQUEIDENTIFIER` primary key defaulting to `NEWID()` (Random). Insert 10,000 rows.
2. Use the system view `sys.dm_db_index_physical_stats` to check the fragmentation percentage of the table. You will likely see >90% fragmentation.
3. Repeat the exercise using `NEWSEQUENTIALID()`. Check fragmentation again. It should be <5%.

## 31. Coding exercises

1. Write a query to find all events occurring in the year 2026. Write one version that is NOT SARGable, and one version that IS SARGable.
2. Create a Composite Index that perfectly supports your SARGable query.
3. Create a Covering Index for a query that selects `Title` and `StartDate` for a specific `LocationID`.

## 32. Mini project

**Objective:** Optimize the NextEvent `Users` table.
1. Our Marketing team frequently searches for users by `Email`. Create the appropriate B-Tree index.
2. Our Security team frequently queries `SELECT PasswordHash FROM Users WHERE Email = @email`. Modify your index to ensure this query performs an Index-Only Scan (Covering Index), preventing the engine from having to look up the base row.
3. In Postgres, create a Partial Index on the `Users` table to quickly find users who have `is_banned = true`.

## 33. Quiz

1. What is the fundamental unit of disk I/O in both SQL Server and PostgreSQL?
2. What is a Key Lookup in SQL Server, and why is it bad for performance?
3. Explain the difference in leaf node contents between a SQL Server Clustered Index and a PostgreSQL B-Tree Index.

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is a Table Scan and why do we want to avoid it?
    *   **A:** A Table Scan (or Sequential Scan) is when the database must read every single page of a table from disk into memory to find the requested rows. It is extremely slow and I/O intensive. We avoid it by using Indexes.
*   **Q:** What is a B-Tree?
    *   **A:** A Balanced Tree. It's a data structure that keeps data sorted and allows searches, sequential access, insertions, and deletions in logarithmic time (O(log n)).

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** Explain what a Covering Index is.
    *   **A:** A Covering Index is a non-clustered/secondary index that "covers" the query entirely. It contains all the columns in the `WHERE` clause (in the index key) and all the columns in the `SELECT` clause (in the `INCLUDE` portion). Because all data is present in the index leaf nodes, the database engine does not need to hit the base table.
*   **Q:** Why is `SELECT *` detrimental to database performance?
    *   **A:** Aside from increasing network payload, `SELECT *` makes it impossible to use Covering Indexes. The database will always be forced to do a Key Lookup or Heap Fetch to retrieve the unindexed columns, severely degrading I/O performance.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** You have a composite index on `(A, B, C)`. Will the index be used for the query: `SELECT * FROM table WHERE B = 5 AND C = 10`?
    *   **A:** No. A B-Tree index is structured hierarchically left-to-right. If the leading column (A) is not provided in the search argument, the database cannot navigate the tree. It must scan the index (or table). To fix this, you either provide A, or create a new index starting with B.
*   **Q:** In PostgreSQL, you update a row's non-indexed `description` column. Why might this still cause performance degradation in all of the table's B-Tree indexes?
    *   **A:** Because of Postgres MVCC. An update creates a physically new row in the Heap with a new CTID. Unless it qualifies as a HOT (Heap-Only Tuple) update (meaning the new row fits on the exact same 8KB block as the old row), Postgres must update *every single index* on the table to point to the new CTID, causing heavy write amplification and index bloat.

## 35. Chapter summary

### Learning Summary
We dismantled the "black box" of database storage, examining the physical 8KB Pages and 64KB Extents that reside on disk. We explored the mathematical brilliance of the B-Tree index and how it reduces search time to logarithmic complexity. We contrasted SQL Server's Clustered Index architecture (where the index *is* the data) with PostgreSQL's Heap architecture (where the index points to physical CTIDs). Finally, we learned how to architect composite, covering, and partial indexes to eliminate Key Lookups and optimize I/O.

### Key Takeaways
*   Indexes transform O(N) Table Scans into O(log N) Index Seeks.
*   A SQL Server Clustered Index determines the physical sort order of the base data.
*   A Postgres B-Tree Index stores CTID pointers mapping to a Heap.
*   `SELECT *` destroys the ability to use Covering Indexes.
*   Random UUID Primary Keys cause massive Page Splits and fragmentation.

### Glossary
*   **Page/Block:** The fundamental 8KB unit of disk storage and I/O.
*   **SARGable:** Search Argument-able. A query structured so the optimizer can use an index.
*   **Key Lookup / Heap Fetch:** The expensive operation of navigating from a secondary index back to the base table to retrieve missing columns.
*   **Covering Index:** An index containing all columns required by a query, via `INCLUDE`.

### Common Mistakes
*   Using functions on indexed columns in the `WHERE` clause (destroying SARGability).
*   Indexing every column in the table (killing write performance).
*   Using random strings or V4 UUIDs as clustered keys.

### Best Practices
*   Design indexes based on the specific `WHERE`, `JOIN`, and `ORDER BY` clauses generated by the application.
*   Order composite indexes starting with the most selective column.
*   Use `INCLUDE` to prevent key lookups for high-frequency read queries.

### Further Reading
*   *SQL Server Execution Plans* by Grant Fritchey.
*   PostgreSQL Documentation: *Index Types* and *MVCC*.
*   *Use The Index, Luke!* (A Guide to Database Performance for Developers).

### Preparation for Next Chapter
In Chapter 4, we will unravel the complexities of **Transactions, ACID Properties, and Concurrency Control**. We will simulate multiple users attempting to buy the exact same concert ticket at the exact same millisecond, and we will learn how to wield Isolation Levels, Pessimistic Locking, and Optimistic MVCC to prevent race conditions and data corruption.
