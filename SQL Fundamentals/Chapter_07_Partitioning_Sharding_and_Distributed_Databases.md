# Chapter 7 – Advanced Storage: Partitioning, Sharding, and Distributed Databases

## 1. Concept Overview

As an enterprise application grows, a single table might swell to contain billions of rows and terabytes of data. At this scale, standard B-Tree indexes and traditional maintenance operations (like backups or index rebuilds) begin to fail or take days to complete. To break through these physical limitations, Database Architects employ advanced storage strategies:

1.  **Partitioning (Local):** Splitting a single logical table into multiple physical tables (partitions) *on the same database server*. The application still queries `SELECT * FROM Tickets`, but the storage engine physically routes the query to a specific underlying partition based on a **Partition Key** (e.g., `PurchaseDate`).
2.  **Sharding (Distributed):** Splitting data across *multiple independent database servers* (nodes) in a "Shared-Nothing" architecture. If you shard by `TenantID`, Customer A's data physically lives on Server 1, and Customer B's data lives on Server 2.
3.  **Distributed Databases:** Systems explicitly designed to span multiple nodes seamlessly, handling data replication, distributed consensus (e.g., Paxos/Raft), and distributed transactions across shards automatically.

## 2. History

In the 1990s, the standard approach to database growth was **Vertical Scaling** (Scaling Up) — simply buying a bigger, more expensive server from IBM, Sun, or HP. However, physical physics dictates a hard limit on CPU cores and RAM on a single motherboard.
In the 2000s, web giants like Google, Amazon, and Facebook proved that **Horizontal Scaling** (Scaling Out) — stringing together thousands of cheap, commodity servers — was the only way to handle infinite internet scale. This gave birth to the NoSQL movement, Sharding, and eventually modern "NewSQL" distributed relational databases (like CockroachDB or Google Spanner).

## 3. Real-world analogy

Imagine a massive **Corporate Archive Warehouse** holding paper records for the last 50 years.

*   **The Monolith (Unpartitioned):** All records are thrown into one giant room. Searching for a 2026 record requires walking through millions of irrelevant 1980s records.
*   **Partitioning:** You build 50 separate rooms, one for each year (The Partition Key). If someone asks for a 2026 record, the archivist instantly ignores 49 rooms and only searches the 2026 room.
*   **Sharding:** You realize one warehouse building is running out of physical land. So, you build a warehouse in New York for East Coast records, and a warehouse in LA for West Coast records. You have *sharded* your physical storage.

## 4. Business problem solved

*   **Maintenance Windows:** You cannot rebuild a 5-terabyte B-Tree index during a 2-hour Saturday maintenance window. Partitioning allows a DBA to rebuild only the "Current Month" partition in 5 minutes, leaving historical partitions untouched.
*   **Instant Archiving (Data Lifecycle):** Deleting 100 million rows from last year takes hours and bloats the transaction log. With partitioning, you can "Drop" or "Switch Out" an entire partition in 1 millisecond as a pure metadata operation.
*   **Hardware Limits:** Sharding solves the problem of a database outgrowing the largest available AWS EC2 instance.

---

## 5. Microsoft SQL Server explanation

SQL Server implements partitioning through a highly structured, three-step metadata architecture:
1.  **Partition Function:** Defines *how* the data is sliced mathematically (e.g., split by year).
2.  **Partition Scheme:** Maps the slices defined in the Partition Function to physical **Filegroups** (allowing you to put 2026 data on fast NVMe SSDs, and 2020 data on cheap, slow HDDs).
3.  **Partitioned Table:** The actual table creation, which is bound to the Partition Scheme instead of a single Filegroup.

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Create a Partition Function (Range Right means the boundary value goes to the right/newer partition)
CREATE PARTITION FUNCTION pf_TicketsByYear (DATETIME2)
AS RANGE RIGHT FOR VALUES ('2025-01-01', '2026-01-01', '2027-01-01');
GO

-- 2. Create a Partition Scheme (Map to the PRIMARY filegroup for simplicity, though normally mapped to different disks)
CREATE PARTITION SCHEME ps_TicketsByYear
AS PARTITION pf_TicketsByYear ALL TO ([PRIMARY]);
GO

-- 3. Create the Table on the Partition Scheme
CREATE TABLE Core.TicketsPartitioned (
    TicketID UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID(),
    EventID UNIQUEIDENTIFIER NOT NULL,
    PurchaseDate DATETIME2 NOT NULL,
    Price DECIMAL(10,2) NOT NULL,
    
    -- The Partition Key MUST be part of the Clustered Index
    CONSTRAINT PK_TicketsPart PRIMARY KEY CLUSTERED (PurchaseDate, TicketID)
) ON ps_TicketsByYear(PurchaseDate);
GO
```

## 7. SQL Server internals

When SQL Server executes a query against a partitioned table, the Relational Engine performs **Partition Elimination** (or Partition Discovery). 
If you query `WHERE PurchaseDate = '2026-05-01'`, the Optimizer mathematically evaluates the Partition Function at compile time. It realizes this date falls exactly into Partition 3. The Execution Plan will explicitly show that it is only scanning 1 out of the 4 partitions, completely ignoring the terabytes of data in the other partitions.

## 8. SQL Server execution

If you write a query *without* the partition key:
`SELECT * FROM Core.TicketsPartitioned WHERE TicketID = 'A1...';`
Because `PurchaseDate` is missing from the `WHERE` clause, the Optimizer cannot perform Partition Elimination. It must search Partition 1, then Partition 2, then Partition 3... This is called a **Partition Scan**. If you partition a table, developers **must** include the partition key in their queries, or performance will be worse than if the table was unpartitioned.

## 9. SQL Server enterprise examples

*   **Log/Audit Tables:** Enterprise applications generate millions of audit logs per day. SQL Server DBAs partition the `AuditLogs` table by Day. At midnight, a SQL Agent Job uses `ALTER TABLE ... SWITCH PARTITION` to instantly move the oldest day's data into a cold-storage archive table in 1 millisecond.
*   **Scale-Out (Sharding):** SQL Server does not have native, transparent sharding built into the core engine. Microsoft provides elastic database tools for Azure SQL, or DBAs build custom Application-Level Routing (the app decides which SQL Server connection string to use based on the Tenant ID).

## 10. SQL Server performance considerations

*   **Aligned Indexes:** When creating Non-Clustered Indexes on a partitioned table, you must ensure the index is "Partition Aligned" (built on the same partition scheme). If an index is not aligned, you cannot perform partition switching (metadata operations) without breaking the index.

## 11. SQL Server security considerations

*   Partitioning does not change security. However, mapping historical partitions to Read-Only Filegroups provides an immutable physical security layer. Even a sysadmin cannot run an `UPDATE` on a row sitting in a Read-Only filegroup.

## 12. SQL Server common mistakes

*   **Partitioning for Query Performance:** Junior DBAs often think partitioning makes queries faster. It usually doesn't. A B-Tree index on an unpartitioned table is incredibly fast (O(log N)). Partitioning is primarily a **Manageability** and **Data Maintenance** feature.
*   **Too Many Partitions:** Creating a partition for every single day over 10 years (3,650 partitions) exhausts SQL Server's internal memory structures and slows down query compilation.

## 13. SQL Server best practices

*   Partition keys should almost always be temporal (Date/Time based).
*   Always include the Partition Key in the Clustered Index.
*   Mandate that developers include the Partition Key in the `WHERE` clause of all queries targeting the partitioned table to guarantee Partition Elimination.

---

## 14. PostgreSQL explanation

Historically, Postgres achieved partitioning through a messy hack involving table inheritance and complex trigger functions. 
Starting in PostgreSQL 10 (and vastly improved in 11 and 12), Postgres introduced **Declarative Partitioning**. 

In Postgres, you create a parent "Logical" table specifying the partition strategy (`RANGE`, `LIST`, or `HASH`). Then, you explicitly create physical "Child" tables and attach them to the parent. The application queries the parent, and Postgres routes the query to the correct physical child table.

## 15. PostgreSQL syntax

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. Create the Logical Parent Table
CREATE TABLE core.audit_logs (
    log_id UUID DEFAULT gen_random_uuid(),
    event_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);
-- Note: A partitioned parent table cannot have a primary key directly unless it includes the partition key.

-- 2. Create the Physical Child Partitions
CREATE TABLE core.audit_logs_2025 PARTITION OF core.audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE core.audit_logs_2026 PARTITION OF core.audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- 3. Create a Default Partition (Catches anything that doesn't fit the rules)
CREATE TABLE core.audit_logs_default PARTITION OF core.audit_logs DEFAULT;

-- 4. Create Indexes on the Parent (Postgres 11+ automatically propagates them to children)
CREATE INDEX idx_audit_logs_event ON core.audit_logs(event_name);
```

## 16. PostgreSQL internals

When parsing a query against `core.audit_logs`, the Postgres Planner checks `enable_partition_pruning` (default is ON). 
If the `WHERE` clause dictates `created_at = '2026-05-01'`, the Planner dynamically removes the 2025 and Default child tables from the Execution Plan before it even executes.

Unlike SQL Server's Filegroup architecture, Postgres child partitions are entirely standard Postgres tables. You can query `core.audit_logs_2026` directly, dump it, or detach it independently.

## 17. PostgreSQL execution

If you run `EXPLAIN` on a partitioned query:
```text
Append  (cost=0.00..35.50 rows=10 width=40)
  ->  Seq Scan on audit_logs_2026  (cost=0.00..35.50 rows=10 width=40)
        Filter: (created_at >= '2026-05-01'::timestamp with time zone)
```
Notice the `Append` node. Postgres views partitioning as a `UNION ALL` of child tables. Because of Partition Pruning, the `Append` node only contains the `audit_logs_2026` table, rather than appending all 3 child tables.

## 18. PostgreSQL enterprise examples

*   **Sharding via Citus:** While native Postgres doesn't shard automatically across servers, Microsoft acquired **Citus Data**, an extension that transforms Postgres into a distributed database. Citus intercepts Postgres queries, determines the Shard Key (e.g., `tenant_id`), and routes the query across a cluster of 50 different Postgres nodes seamlessly, aggregating the results back to the user. This powers multi-tenant SaaS platforms like Cloudflare and Notion.
*   **Federation via postgres_fdw:** You can create a partition in Server A, but define it as a Foreign Table pointing to Server B using the Foreign Data Wrapper. This is a manual, primitive way to shard data geographically in native Postgres.

## 19. PostgreSQL performance considerations

*   **Hash Partitioning:** If you want to distribute I/O evenly across physical disks for a massive table (e.g., `Tickets`), but you don't query by Date, you can use `PARTITION BY HASH (ticket_id)`. Postgres will run a hash algorithm on the ID and distribute the rows evenly across 10 child tables.
*   **Partition Pruning Overhead:** Having 5,000 partitions in Postgres causes severe planning time overhead. The Planner must evaluate 5,000 tables during the `EXPLAIN` phase. Keep partition counts reasonable (under 1000).

## 20. PostgreSQL security considerations

*   Because partitions are physical child tables, you must ensure that Role privileges (`GRANT SELECT ON core.audit_logs TO app_user`) propagate correctly. In modern Postgres, granting permissions on the parent automatically applies to the children.

## 21. PostgreSQL common mistakes

*   **Missing a Default Partition:** If you use Range Partitioning and forget to create the 2027 partition, on January 1st, 2027, every single `INSERT` into the application will crash with a "no partition found" error. Always have a `DEFAULT` partition, and set up a monitoring script to alert you if data lands in it.

## 22. PostgreSQL best practices

*   Automate partition creation. Use an extension like `pg_partman` to automatically create next month's partition and detach partitions older than 1 year.
*   Unlike SQL Server, Postgres allows a partition to be a foreign table. Use this for geographical data localization (GDPR compliance).

---

## 23. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Architecture** | Metadata (Functions/Schemes/Filegroups) | Physical (Parent/Child Tables) | Postgres's child tables are easier to manage individually. SQL Server's filegroups offer deeper disk-level optimization. |
| **Partition Key in PK**| Mandatory | Mandatory | Both engines require the partition key to be part of any Unique/Primary constraint to guarantee global uniqueness without scanning all partitions. |
| **Distributed Sharding**| No native engine support (App-level) | Via Citus extension | Citus makes Postgres a world-class distributed system. SQL Server requires significant custom architecture. |
| **Partition Maintenance**| `ALTER TABLE ... SWITCH` | `ALTER TABLE ... ATTACH/DETACH` | Both allow instant, lock-free metadata operations for archiving. |

## 24. Architect recommendations

**The Sharding Fallacy**
Developers love the idea of Sharding because it sounds web-scale. As an Architect, **avoid sharding until it is the absolute last resort.** 
When you shard a database across multiple physical servers:
1. You lose cross-shard `JOIN` capabilities. (You cannot join User data on Server A with Ticket data on Server B efficiently).
2. You lose strict ACID guarantees. (Foreign Keys cannot span servers).
3. Operational complexity increases 100x (Backups, migrations, monitoring).

Scale vertically first (buy a bigger server). Then optimize indexes. Then implement read-replicas. Then partition. Only shard if your write-throughput literally exceeds the physical NVMe bus limits of the largest available server.

## 25. DBA recommendations

*   When designing a partition scheme for archiving, always keep your active partition empty of "old" data. If you partition by month, do not constantly update rows from 3 months ago, or you will trigger row-movement across partitions, which is incredibly slow and causes heavy locking.

## 26. Developer recommendations

*   When interacting with partitioned tables, or distributed systems like Citus, the **Distribution Key (Partition Key)** must become a first-class citizen in your application code. Every API call, every ORM lookup, must include the Tenant ID or the Date, so the database knows exactly which physical node/partition to hit.

## 27. Production case study

**The NextEvent Audit Log Meltdown**

*Scenario:* For compliance, NextEvent logged every click, login, and purchase into an `AuditLogs` table. It reached 2 Billion rows (1.5 TB). The query performance was fine (B-Tree indexes worked), but the DBA team could no longer rebuild the indexes or delete data older than 1 year. A simple `DELETE FROM AuditLogs WHERE CreatedAt < '2025-01-01'` took 14 hours and blew up the Transaction Log to 2 TB, crashing the production server.

*Architectural Fix:* We rebuilt the `AuditLogs` table using Monthly Range Partitioning. 
Now, when the 1-year data retention policy kicks in, the DBA executes a metadata command: `ALTER TABLE core.audit_logs DETACH PARTITION audit_logs_2025_01`. 
This operation takes **1 millisecond**. The child table is immediately decoupled from the parent. The DBA then drops the child table instantly. Maintenance time was reduced from 14 hours to 1 second, with zero transaction log bloat.

## 28. ASCII diagrams wherever helpful

**Partition Elimination vs. Sharding**

```text
======================================================
 LOCAL PARTITIONING (Single Server, Partition Elimination)
======================================================
Query: SELECT * FROM Logs WHERE Date = 'Feb'

      [ QUERY OPTIMIZER ] -> Determines 'Feb' is Partition 2
              |
      +-------+-------+
      |       |       | (Pruned / Ignored)
    [JAN]   [FEB]   [MAR]   <-- Physical Partitions on Disk

======================================================
 DISTRIBUTED SHARDING (Multiple Servers, e.g., Citus)
======================================================
Query: SELECT * FROM Users WHERE TenantID = 99

   [ ROUTER NODE (Coordinator) ] -> Has mapping: Tenant 99 is on Node B
          /             \
         /               \
[ DATA NODE A ]     [ DATA NODE B ]
(Tenants 1-50)      (Tenants 51-100)
                       |
                   Executes query and returns data
```

## 29. Enterprise design discussion

**The CAP Theorem (Why Relational Databases Struggle with Sharding)**

In 2000, Eric Brewer formulated the CAP Theorem for distributed data stores. It states you can only pick two of three guarantees:
*   **C - Consistency:** Every read receives the most recent write. (ACID).
*   **A - Availability:** Every request receives a non-error response.
*   **P - Partition Tolerance:** The system continues to operate despite network failures between nodes.

Relational databases (SQL Server, Postgres) strongly prefer **CA** (Consistency and Availability). They assume a reliable single-node network.
When you shard across a network, you introduce **P** (Network Partitions are inevitable). Therefore, you must sacrifice either C or A.
NoSQL databases (Cassandra, MongoDB) often chose **AP** (Sacrificing Consistency for Eventual Consistency). Modern NewSQL databases (Spanner, CockroachDB) use atomic clocks and consensus algorithms (Raft) to achieve **CP** at a global scale, providing relational SQL over distributed nodes, though with higher write latency.

## 30. Hands-on exercises

1. Open your database engine. Create a Partition Function/Scheme (SQL Server) or a logical Parent Table (Postgres).
2. Insert 10 rows with dates from 2025, and 10 rows from 2026.
3. Query the data explicitly by the 2026 date. Look at the Execution Plan and verify that Partition Elimination/Pruning occurred.

## 31. Coding exercises

1. Write the SQL script to create a daily partitioned table for `WebTraffic` for the current month.
2. Write a script to simulate an archival process: Create an empty `WebTraffic_Archive` table. In SQL Server, write the `SWITCH` command. In Postgres, write the `DETACH` command.

## 32. Mini project

**Objective:** Multi-Tenant Design for NextEvent.
NextEvent is transitioning to a SaaS model. We will host thousands of event companies (Tenants) on a single Postgres database. We must prepare for future Citus sharding.
1. Alter the `Core.Events` and `Core.Tickets` tables to include a `TenantID`.
2. Drop and recreate the Primary Keys on both tables. *Crucial:* You must include the `TenantID` as part of a composite Primary Key, or else a distributed sharded database cannot guarantee global uniqueness without querying every single shard.
3. Write a sample query fetching a ticket, ensuring the `TenantID` is in the `WHERE` clause to enable future Shard Routing.

## 33. Quiz

1. What is the primary difference between Partitioning and Sharding?
2. Why is it a terrible idea to partition a table to speed up a slow query that doesn't use the partition key?
3. What is the CAP theorem, and why does it make distributed SQL transactions difficult?

## 34. Interview questions

**Entry Level (Developer)**
*   **Q:** What is database partitioning?
    *   **A:** Partitioning is the process of splitting a single large logical table into multiple smaller physical pieces based on a key (like a date). The application queries the logical table normally, but the database engine routes the query to the specific physical piece, improving manageability.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** What is Partition Elimination (or Pruning) and why is it important?
    *   **A:** It is an optimizer optimization. When the engine parses a query, if the `WHERE` clause filters on the partition key, the engine mathematically eliminates all irrelevant partitions from the execution plan. It prevents scanning terabytes of unnecessary data.
*   **Q:** Why do DBAs prefer partitioning for data archival instead of running `DELETE` statements?
    *   **A:** A `DELETE` statement is a fully logged, row-by-row transaction. Deleting 50 million rows takes hours, blows up the transaction log, and causes heavy locking. Dropping or detaching a partition is a metadata-only operation that takes milliseconds and uses virtually zero transaction log space.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** You have a multi-tenant SaaS application on Postgres. You decide to shard the database using the `tenant_id` as the shard key across 10 physical nodes. A developer writes a query: `SELECT * FROM Users WHERE email = 'test@test.com'`. What happens architecturally, and why is this a catastrophe at scale?
    *   **A:** Because the query does not include the Shard Key (`tenant_id`), the Coordinator node cannot route the query. It must perform a "Scatter-Gather" operation—broadcasting the query to all 10 physical nodes, waiting for all 10 to execute an index scan, and aggregating the results over the network. At scale (e.g., thousands of queries a second across 50 nodes), this network chatter will instantly bring the entire distributed cluster down. In a sharded system, the shard key must be propagated to every single query.

## 35. Chapter summary

### Learning Summary
We explored how to scale databases beyond the physical limits of a single table or server. We mastered local Horizontal Partitioning in both SQL Server and PostgreSQL, learning how it transforms impossible data archiving tasks into millisecond metadata operations. We defined the boundaries between Partitioning (manageability) and Sharding (distributed hardware scaling), and touched upon the CAP theorem's constraints on distributed relational integrity.

### Key Takeaways
*   Partitioning splits tables physically on one server; Sharding distributes them across multiple servers.
*   Partitioning is primarily a DBA tool for data lifecycle management and index maintenance, not a magic bullet for query performance.
*   Queries against partitioned/sharded tables must include the Partition/Shard Key, or they will cause devastating Partition Scans or Scatter-Gather network broadcasts.
*   Never shard an enterprise database unless vertical scaling and read-replicas have been completely exhausted.

### Glossary
*   **Partition Key:** The column used to mathematically divide data into physical partitions.
*   **Partition Elimination:** The Optimizer's ability to ignore irrelevant partitions during query execution.
*   **Sharding:** A Shared-Nothing distributed architecture splitting data across multiple servers.
*   **Scatter-Gather:** A distributed query that must fan out to all shards and combine results over the network.
*   **Citus:** A Postgres extension that enables transparent distributed sharding.

### Common Mistakes
*   Failing to include the Partition Key in the Primary Key constraint.
*   Using Sharding to solve a problem that could be fixed with better indexes.
*   Forgetting to create a `DEFAULT` partition in Postgres.

### Best Practices
*   Use Date/Time columns for range partitioning (e.g., Monthly logs).
*   Use Hash partitioning if the goal is purely to distribute I/O across disks for a massive, evenly-accessed table.
*   Adopt the Saga pattern for microservices rather than attempting cross-shard distributed transactions.

### Further Reading
*   SQL Server Partitioned Tables and Indexes (Microsoft Learn).
*   PostgreSQL Declarative Partitioning Documentation.
*   *Designing Data-Intensive Applications* by Martin Kleppmann (Essential reading for the CAP theorem and distributed systems).

### Preparation for Next Chapter
In Chapter 8, we will explore **High Availability, Disaster Recovery, and Replication**. We will learn how to design systems that survive server crashes, data center fires, and ransomware attacks. We will configure SQL Server Always On Availability Groups, PostgreSQL Streaming Replication, and design architectures with zero data loss (RPO = 0) and minimal downtime (RTO).
