# Database Engineering for Global SaaS Applications
### From SQL Fundamentals to Enterprise Multi-Tenant Architecture

**Case Study**: A production-grade Multi-Tenant EV Charging Management SaaS Platform. Throughout the book, we will build the schema, queries, API integrations, and scaling strategies for managing charging stations, sessions, billing, and tenants (organizations/customers).

---

## Part 1: Relational Foundations & Schema Design
**Chapter 1: Database Fundamentals**
*   Introduction to Relational Databases and SQL Server/Azure SQL.
*   The EV Charging SaaS: Understanding the domain (Tenants, Stations, Ports, Users, Sessions).
*   Storage internals: Pages, extents, and the transaction log.

**Chapter 2: SQL Fundamentals & Data Types**
*   SQL basics: DDL, DML, DCL, TCL.
*   Choosing the right Data Types (UUIDs vs. INTs for distributed SaaS, precision for EV power metrics).
*   *Architect Perspective*: The hidden cost of oversized data types in a billion-row SaaS.

**Chapter 3: Constraints & Data Integrity**
*   Primary Keys, Foreign Keys, CHECK constraints, and UNIQUE constraints.
*   Enforcing domain rules at the DB level (e.g., preventing negative charging kW).

## Part 2: Querying the SaaS Domain
**Chapter 4: Core CRUD Operations**
*   INSERT, UPDATE, DELETE basics.
*   *EF Core Context*: How Entity Framework translates basic commands.

**Chapter 5: The SELECT Statement & Filtering**
*   SELECT, WHERE, and Boolean logic.
*   Sargability: Why certain WHERE clauses kill index performance.

**Chapter 6: Sorting & Aggregation (GROUP BY, HAVING)**
*   ORDER BY and pagination (OFFSET FETCH).
*   Aggregating billing data and session metrics (SUM, AVG, MIN, MAX).

**Chapter 7: Relational Joins & Subqueries**
*   INNER, LEFT, RIGHT, FULL, CROSS joins.
*   Reconstructing the Tenant -> Station -> Session hierarchy.
*   Correlated vs. Uncorrelated Subqueries (and when to avoid them).

**Chapter 8: Set Operations (UNION)**
*   UNION vs. UNION ALL.
*   Combining historical and active EV charging tables.

## Part 3: Advanced Analytical Patterns
**Chapter 9: CTEs & Recursive CTEs**
*   Organizing complex queries with Common Table Expressions.
*   Recursive CTEs for hierarchical data (e.g., Station Group/Fleet hierarchies).

**Chapter 10: Window Functions**
*   OVER(), PARTITION BY, ROW_NUMBER(), RANK(), DENSE_RANK().
*   Calculating running totals for tenant billing and finding consecutive charging anomalies.

**Chapter 11: Advanced DML: MERGE & UPSERT**
*   The MERGE statement for syncing station data from OCPP IoT endpoints.
*   UPSERT patterns in SQL Server vs. PostgreSQL.
*   *Production Pitfalls*: Concurrency issues with MERGE.

## Part 4: Programmability & Semi-Structured Data
**Chapter 12: Views & Materialization**
*   Abstracting schema complexity with Views.
*   Indexed Views (Materialized Views) for real-time dashboard performance.

**Chapter 13: Stored Procedures & Functions**
*   Encapsulating complex billing logic in Stored Procs.
*   Scalar vs. Table-Valued Functions (and the dreaded UDF performance penalty).

**Chapter 14: Dynamic SQL in Enterprise Apps**
*   `sp_executesql` for dynamic filtering in the SaaS Admin dashboard.
*   Mitigating SQL Injection risks in Dynamic SQL.

**Chapter 15: Handling JSON & XML**
*   Storing raw OCPP (Open Charge Point Protocol) payloads as JSON in SQL Server.
*   Querying and indexing JSON data. XML legacy support.

## Part 5: Concurrency & Transaction Management
**Chapter 16: Transactions & ACID**
*   BEGIN TRAN, COMMIT, ROLLBACK.
*   Ensuring Atomicity during a billing run (Session + Invoice + Payment).

**Chapter 17: Locking, Blocking, & Deadlocks**
*   Lock hierarchy (Row, Page, Table).
*   Analyzing and resolving Deadlocks in high-concurrency IoT environments.

**Chapter 18: Isolation Levels**
*   READ COMMITTED, REPEATABLE READ, SERIALIZABLE.
*   Optimistic Concurrency: SNAPSHOT isolation and RCSI.

## Part 6: Performance Optimization at Scale
**Chapter 19: Indexing Strategies**
*   Clustered vs. Non-Clustered Indexes.
*   Covering indexes, Filtered indexes (perfect for `IsDeleted = 0`).

**Chapter 20: Execution Plans & Query Optimization**
*   Reading graphical and XML execution plans.
*   Index Seeks vs. Scans, Key Lookups, Spooling.

**Chapter 21: Statistics & Parameter Sniffing**
*   How the query optimizer uses Statistics.
*   Diagnosing and fixing Parameter Sniffing in multi-tenant queries (where tenant sizes vary wildly).

**Chapter 22: Table Partitioning**
*   Partitioning multi-terabyte charging telemetry tables by Date/Month.
*   Sliding window partition management.

## Part 7: Enterprise Schema Tracking
**Chapter 23: Normalization & Denormalization**
*   1NF to 3NF.
*   *Architect Perspective*: When to intentionally denormalize for read-heavy SaaS dashboards.

**Chapter 24: Audit Logging & Soft Deletes**
*   Implementing `IsDeleted` flags.
*   Global query filters in EF Core.

**Chapter 25: Temporal Tables & CDC**
*   System-Versioned Temporal Tables for tracking historical configuration changes.
*   Change Data Capture (CDC) for streaming EV data to data warehouses.

## Part 8: Security & Compliance
**Chapter 26: Security Basics & SQL Injection**
*   Authentication vs. Authorization (Logins vs. Users).
*   Preventing SQL injection via parameterization and ORMs.

**Chapter 27: Encryption**
*   Always Encrypted (client-side) for PCI-compliant credit card data.
*   Transparent Data Encryption (TDE) at rest.

**Chapter 28: Dynamic Data Masking & RLS**
*   Masking PII (emails, phone numbers) from support staff.
*   Row-Level Security (RLS) for tenant isolation at the database engine level.

## Part 9: Cloud Database Architecture
**Chapter 29: Azure SQL & Elastic Pools**
*   DTUs vs. vCores.
*   Cost-optimizing a multi-tenant DB-per-tenant model using Elastic Pools.

**Chapter 30: Geo-Replication, Backup & DR**
*   Active Geo-Replication for global EV charging networks.
*   RPO and RTO strategies.

**Chapter 31: Monitoring & Query Store**
*   Using Query Store to catch performance regressions after application deployments.

## Part 10: Multi-Tenant Architecture
**Chapter 32: Multi-Tenant Database Models**
*   Shared Database / Shared Schema (Row-level separation).
*   Shared Database / Isolated Schemas.
*   Database per Tenant.
*   *Architect Perspective*: The ultimate trade-off matrix.

**Chapter 33: White Labeling & Feature Flags**
*   Schema design for storing white-label themes and feature toggles per tenant.

**Chapter 34: Idempotent Seeding & Distributed IDs**
*   Why IDENTITY fails in distributed systems.
*   Implementing Snowflake IDs, UUIDv7, or HiLo for distributed entity generation.

**Chapter 35: Scaling Patterns: CQRS, Event Sourcing & Caching**
*   Command Query Responsibility Segregation (CQRS) for the EV platform.
*   Event Sourcing the wallet/billing ledger.
*   Integrating Redis caching for station availability states.

## Part 11: The Application Layer & Production
**Chapter 36: EF Core in Enterprise**
*   Handling migrations in CI/CD.
*   No-tracking queries, split queries, and avoiding N+1 problems.

**Chapter 37: Application Performance Tuning**
*   Connection pooling, retry policies (Polly), and handling transient Azure faults.

**Chapter 38: Enterprise Best Practices & Deployments**
*   Zero-downtime database deployments (Expand/Contract pattern).
*   The deployment checklist.

**Chapter 39: Scaling to Millions of Users**
*   Sharding, Read Replicas, and asynchronous processing (Azure Service Bus/RabbitMQ).

**Chapter 40: Complete Production Case Study**
*   A holistic review combining all concepts into the final architecture diagram of the EV SaaS platform.
*   Final Interview Questions & Capstone Exercise.

---
*Appendices: Glossary, SQL Cheat Sheets, EF Core Cheat Sheets, Index.*
