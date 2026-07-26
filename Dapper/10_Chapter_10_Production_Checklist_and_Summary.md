# Chapter 10: The Production Checklist and Summary

## 1. Learning Objectives

By the end of this final chapter, you will be able to:
*   Evaluate an entire ASP.NET Core Dapper solution against a rigorous Enterprise Production Checklist.
*   Identify critical security, performance, and architectural misconfigurations before deploying to a live environment.
*   Synthesize the core tenets of Enterprise Data Access learned throughout this book.

## 2. Introduction

You have traversed the entire landscape of Dapper. You started with the fundamental mechanics of IL Emit and `IDataReader`, moved through complex object mapping and Table-Valued Parameters, and finally architected secure, multi-tenant CQRS pipelines with Azure deployment and OpenTelemetry observability.

However, knowledge without disciplined execution leads to production incidents. Before any Enterprise SaaS platform goes live, it must pass a strict readiness review. This chapter provides the ultimate "Dapper Architect's Production Checklist." It is a distilled aggregation of every best practice, common mistake, and architectural mandate discussed in this book.

Print this checklist. Pin it to your wall. Enforce it in your Pull Requests.

---

## 3. The Dapper Architect's Production Checklist

### Layer 1: Security & Identity
- [ ] **No Hardcoded Passwords:** Connection strings do not contain `User ID` or `Password`. The application utilizes Azure Managed Identity (or AWS IAM) via `DefaultAzureCredential` to authenticate to the database.
- [ ] **Strict Parameterization:** There is absolutely zero dynamic string concatenation of user input into Dapper SQL strings. All variables are passed via anonymous objects or `DynamicParameters`.
- [ ] **Whitelist Dynamic Ordering:** If the application requires dynamic `ORDER BY` clauses, the user input is strictly validated against a hardcoded C# dictionary/whitelist of allowed column names before being injected into the SQL string.
- [ ] **Tenant Isolation (RLS):** For shared-database multi-tenancy, Row-Level Security (RLS) is enabled in SQL Server. The `ISqlConnectionFactory` explicitly sets `SESSION_CONTEXT(N'TenantId')` for every HTTP request before returning the connection.
- [ ] **Principle of Least Privilege:** The database user account used by the Dapper application has `SELECT/INSERT/UPDATE/DELETE` permissions only. It cannot execute DDL (`CREATE/DROP/ALTER`) or access `sysadmin` functions.
- [ ] **Data Masking / Encryption:** Highly sensitive PII (like SSNs or Credit Cards) utilizes SQL Server Dynamic Data Masking or Always Encrypted. Dapper handles this transparently via the driver.

### Layer 2: Performance & Connectivity
- [ ] **Factory Pattern Enforced:** Connections are never registered as `Scoped` or `Singleton` in the DI container. An `ISqlConnectionFactory` is injected, and connections are created locally within `using` blocks in the Repositories.
- [ ] **Explicit Pagination:** All list-based API endpoints enforce `OFFSET/FETCH`. Unbounded `SELECT * FROM Table` queries are prohibited to prevent Out-Of-Memory (OOM) exceptions under load.
- [ ] **No N+1 Queries:** `QueryMultiple` (GridReader) is used to retrieve parent and child collections in a single network round trip. Foreach loops containing Dapper `Query` calls are strictly banned.
- [ ] **Buffered Execution by Default:** Unless streaming massive datasets directly to a file via a pipeline, Dapper queries use the default `buffered: true` behavior to instantly free the ADO.NET connection for the pool.
- [ ] **TVPs for Bulk Operations:** Inserting or updating more than 50 rows is executed via a Table-Valued Parameter (TVP) and a User-Defined Table Type, never via sequential looping or Dapper's native list execution.

### Layer 3: Architecture & Clean Code
- [ ] **Dependency Inversion:** The Application layer (MediatR Handlers, Domain Logic) has zero references to `Dapper` or `Microsoft.Data.SqlClient`. It relies entirely on Repository interfaces.
- [ ] **CQRS Separation:** Complex database reads (Queries) bypass the Domain Entities and use Dapper to map directly to flat, UI-optimized DTOs.
- [ ] **Centralized Type Handlers:** `SqlMapper.AddTypeHandler` for Enums, JSON, or custom structs is executed exactly once during application startup in `Program.cs`.
- [ ] **Transaction Boundaries:** Transactions are managed explicitly via `SqlTransaction`. The transaction is injected into repositories when multiple mutations must be atomic. Ambient `TransactionScope` is avoided in HTTP pipelines unless strictly necessary.

### Layer 4: Resilience & Observability
- [ ] **Polly Integration:** The `ISqlConnectionFactory` is wrapped in a Polly `AsyncRetryPolicy` configured with exponential backoff and jitter to silently handle transient cloud network faults (e.g., Azure SQL Error 40613).
- [ ] **OpenTelemetry Active:** `OpenTelemetry.Instrumentation.SqlClient` is configured in `Program.cs` with `SetDbStatementForText = true`.
- [ ] **Timeout Management:** Long-running analytics queries utilize `CommandDefinition` to explicitly increase the `CommandTimeout` beyond the default 30 seconds, preventing false failures.
- [ ] **Deadlock Retries:** Application logic explicitly catches `SqlException` Error 1205 (Deadlock) and implements a specific, short-delay retry mechanism for the entire transaction block.

### Layer 5: Testing & CI/CD
- [ ] **Zero Mocking of Dapper:** `IDbConnection` extension methods are never mocked. Business logic is unit-tested by mocking the Repository interface.
- [ ] **Testcontainers Integration:** 100% of Dapper Repositories are covered by Integration Tests that spin up ephemeral SQL Server Docker containers via the `Testcontainers` library.
- [ ] **Transaction Rollback Pattern:** Integration tests wrap Dapper executions in a `SqlTransaction` and call `RollbackAsync()` during test teardown to guarantee pristine database state in milliseconds.
- [ ] **Automated Migrations:** Database schema migrations (e.g., DbUp, FluentMigrator) execute as a strict prerequisite step in the CI/CD deployment pipeline *before* the new Dapper code is deployed.
- [ ] **Expand and Contract:** Destructive schema changes (like `DROP COLUMN`) are executed across multiple staggered deployments to guarantee zero downtime for the Dapper application.

---

## 4. Summary and Conclusion

Dapper is not an Object-Relational Mapper in the traditional sense; it is a surgical data access conduit. It does not try to hide the database from you. It assumes that you, the Architect, know exactly what you are doing.

When you choose Dapper, you choose to take full responsibility for the T-SQL execution plan, the concurrency isolation level, the connection lifecycle, and the object materialization graph. 

Entity Framework Core is exceptional for managing complex domain invariants, tracking changes, and accelerating basic CRUD operations. However, when building high-performance SaaS platforms, enterprise reporting engines, or systems requiring millions of transactions per second, the abstractions of heavy ORMs become severe bottlenecks. 

By utilizing Dapper—and wrapping it in the rigorous Clean Architecture, Security, and Testing paradigms detailed in this book—you unlock the absolute maximum performance of your database engine. You eliminate memory allocations. You collapse network latency. You gain deterministic observability.

You have now mastered the Micro-ORM.

Go build something incredibly fast.
