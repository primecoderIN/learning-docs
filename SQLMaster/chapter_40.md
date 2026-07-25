# Chapter 40: The Architectural Review Board (ARB) Checklist

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand the purpose of an Architectural Review Board (ARB) in enterprise software development.
*   Utilize a comprehensive, 40-point checklist to evaluate the production-readiness of any SQL Server and EF Core application.
*   Adopt the "Architect's Mindset" when designing distributed systems.

---

## 40.1 What is an ARB?

Before a new microservice or major architectural change is deployed to production in a large enterprise, it must pass an **Architectural Review Board (ARB)**. 
The ARB is a panel of Principal and Staff Engineers who grill the development team to ensure the system will not crash, leak data, or bankrupt the company when it hits production.

This chapter serves as your personal ARB Checklist. Use this rubric to grade your applications before they go live. If you can confidently check every box, your database architecture is world-class.

---

## 40.2 Category 1: Schema & Data Types

*   [ ] **Guids vs Ints:** Have you explicitly chosen `UNIQUEIDENTIFIER` or `INT`/`BIGINT` for Primary Keys based on security (IDOR) and scaling requirements, rather than defaulting? (Chapter 5)
*   [ ] **Sequential Guids:** If using Guids for Clustered Indexes, are they generated sequentially (e.g., `NEWSEQUENTIALID` or UUIDv7) to prevent massive page fragmentation? (Chapter 8)
*   [ ] **String Sizing:** Are `VARCHAR` and `NVARCHAR` lengths strictly defined (e.g., `VARCHAR(50)`)? Have you banned `VARCHAR(MAX)` unless storing multi-megabyte documents? (Chapter 3)
*   [ ] **Soft Deletes:** Are you using `IsDeleted = 1` bit flags instead of destructive `DELETE` statements to preserve audit history and prevent cascade locking? (Chapter 11)
*   [ ] **Constraint Enforcement:** Are Foreign Keys and Unique Constraints enforced at the database level, rather than relying solely on C# application logic? (Chapter 6)

---

## 40.3 Category 2: Indexing & Query Tuning

*   [ ] **Clustered Indexes:** Does every table have exactly one carefully chosen Clustered Index, prioritizing narrow, static, and ever-increasing values? (Chapter 8)
*   [ ] **Covering Indexes:** Are your most frequent, high-value queries supported by Nonclustered Indexes that `INCLUDE` the necessary columns to prevent Key Lookups? (Chapter 19)
*   [ ] **SARGability:** Do all `WHERE` clauses avoid wrapping columns in functions (e.g., avoiding `YEAR(CreatedAt) = 2026`) to ensure the Query Optimizer can seek the index? (Chapter 19)
*   [ ] **Parameter Sniffing:** Are highly skewed multi-tenant reporting queries protected against Parameter Sniffing using `OPTION (RECOMPILE)` or dynamic SQL? (Chapter 21)
*   [ ] **Pagination:** Are API endpoints returning lists using `OFFSET/FETCH` or Keyset Pagination, strictly capping the maximum `Take()` to prevent memory exhaustion? (Chapter 10)

---

## 40.4 Category 3: Transactions & Concurrency

*   [ ] **Transaction Scope:** Are database transactions kept as short as possible? Have third-party network calls (like Stripe or SendGrid) been explicitly removed from inside SQL transactions? (Chapter 16)
*   [ ] **Isolation Levels:** Is Read Committed Snapshot Isolation (RCSI) enabled on the database to prevent readers from blocking writers? (Chapter 18)
*   [ ] **Optimistic Concurrency:** Do highly collaborative tables include a `RowVersion` column to prevent the "Lost Update" problem? (Chapter 25)
*   [ ] **Deadlock Mitigation:** Do all transactions access tables in the exact same alphabetical order to mathematically prevent circular deadlocks? (Chapter 17)
*   [ ] **The Outbox Pattern:** Are events destined for RabbitMQ/Kafka saved to an Outbox table in the same transaction as the business data to prevent Dual-Write inconsistencies? (Chapter 30)

---

## 40.5 Category 4: Entity Framework Core

*   [ ] **Change Tracker:** Do all read-only API endpoints explicitly use `.AsNoTracking()` to reduce memory consumption? (Chapter 23)
*   [ ] **N+1 Queries:** Are all child collections loaded safely using `.Include()` or Explicit Loading to prevent N+1 query storms? (Chapter 23)
*   [ ] **Cartesian Explosion:** Do queries including multiple 1-to-Many collections utilize `.AsSplitQuery()` to prevent memory exhaustion from massive row multiplication? (Chapter 23)
*   [ ] **Bulk Updates:** Are bulk status changes executing via EF Core 7+ `ExecuteUpdateAsync()` to bypass the Change Tracker? (Chapter 24)
*   [ ] **Raw SQL Safety:** Are all raw SQL commands utilizing `FromSqlInterpolated` instead of standard string concatenation to prevent SQL Injection? (Chapter 24)

---

## 40.6 Category 5: Security & Isolation

*   [ ] **Multi-Tenant Isolation:** Is the multi-tenant architecture explicitly defined (Shared vs Isolated), and if shared, is Row-Level Security (RLS) enabled at the storage engine level? (Chapter 27 & 39)
*   [ ] **Principle of Least Privilege:** Does the API connect to SQL Server using a Login that only has `db_datareader` and `db_datawriter` permissions, explicitly lacking schema alteration rights? (Chapter 6)
*   [ ] **PII Encryption:** Is Highly Sensitive PII (like SSNs or Bank Accounts) encrypted at rest using Always Encrypted or application-level cryptography? (Chapter 34)
*   [ ] **Side-Channel Prevention:** Are external users prevented from executing ad-hoc T-SQL to prevent timing attacks against RLS predicates? (Chapter 27)

---

## 40.7 Category 6: Maintenance, HA & DR

*   [ ] **High Availability:** Are production databases configured in an Always On Availability Group (or Azure equivalent) for automatic, sub-minute failovers? (Chapter 28)
*   [ ] **Read-Scale Out:** Are heavy reporting queries routed to the secondary replica using `ApplicationIntent=ReadOnly`? (Chapter 28)
*   [ ] **Transaction Log Management:** Are Transaction Log backups running at least every 15 minutes to control LDF growth and ensure a tight RPO? (Chapter 32)
*   [ ] **Index Maintenance:** Is Ola Hallengren's script (or Azure automated tuning) scheduled weekly to rebuild fragmented indexes and update statistics? (Chapter 32)
*   [ ] **Zero-Downtime Deployments:** Are breaking schema changes (like column renames) executed using the Expand-and-Contract pattern alongside Blue/Green deployments? (Chapter 38)

---

## 40.8 Conclusion: The Architect's Mindset

You have completed the journey. You now possess the knowledge to architect databases that handle billions of rows, thousands of concurrent users, and the strictest security requirements in the world.

But knowledge of syntax is not what makes an Architect.

**The Architect's Mindset is recognizing that every single technical decision is a trade-off.**
*   You trade Disk Space for Read Speed (Indexes).
*   You trade Write Speed for Data Safety (Synchronous Commits).
*   You trade Consistency for Scalability (CQRS / Eventual Consistency).
*   You trade Simplicity for Zero-Downtime (Expand-and-Contract).

Your job is no longer just writing code. Your job is to understand the business requirements—RTO, RPO, Budget, and Risk Tolerance—and pull the exact right levers to construct a system that perfectly balances those opposing forces.

Good luck, and build beautifully.

---
**End of Book.**
