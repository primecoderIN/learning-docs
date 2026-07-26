# Chapter 7: Enterprise SaaS: Multi-Tenancy and Security

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Evaluate and implement different Multi-Tenancy architectures (Shared vs. Isolated Database) using Dapper.
*   Secure an enterprise SaaS application from cross-tenant data leakage by utilizing SQL Server Row-Level Security (RLS) integrated with ASP.NET Core DI.
*   Design a resilient Audit Logging mechanism for tracking state mutations natively through Dapper.
*   Implement high-performance Role-Based Access Control (RBAC) and Feature Flag resolution directly within database reads.
*   Understand and mitigate advanced SQL Injection vectors that bypass naive parameterization.

## 2. Introduction

Building an internal corporate application is fundamentally different from building a B2B Software-as-a-Service (SaaS) platform. In a SaaS platform, a single application instance (and often a single database) serves multiple, completely isolated organizations (Tenants). 

The most catastrophic failure a SaaS platform can suffer is a cross-tenant data leak—for example, Tenant A viewing Tenant B's invoices. When using a Micro ORM like Dapper, where you write raw T-SQL strings, the risk of a developer forgetting to add a `WHERE TenantId = @TenantId` clause to a query is exceptionally high. 

This chapter focuses on defensive data architecture. We will explore how to push security constraints down to the database engine level, ensuring that even if application-level Dapper queries are written poorly, the database itself prevents unauthorized access. Furthermore, we will tackle the enterprise requirements of Audit Logging, Soft Deletes, and Role-Based Access Control using high-performance Dapper strategies.

## 3. Core Concepts

### Multi-Tenancy Models
1.  **Database-per-Tenant (Isolated):** Every tenant gets their own SQL Database. Highest security, highest cost, most complex schema migrations. Dapper handles this easily by simply resolving a different Connection String per request.
2.  **Shared Database, Shared Schema:** All tenants live in one database and share tables. Isolation is handled via a `TenantId` column. Lowest cost, easiest schema management, highest risk of data leakage.

### Row-Level Security (RLS)
RLS is a SQL Server feature that allows you to define a security policy (a predicate function) that automatically filters rows based on the execution context. When RLS is enabled, you do not need to write `WHERE TenantId = @id` in your Dapper queries. SQL Server invisibly appends the filter to every `SELECT`, `UPDATE`, and `DELETE`.

### Session Context
To make RLS work with connection pooling, we use `sp_set_session_context`. When an HTTP request begins, we grab a connection from the pool, use Dapper to set a temporary key-value pair (`TenantId` = `123`) on the SQL Server session, and then execute our business queries. SQL Server reads this context to evaluate the RLS policy.

## 4. Visual Diagrams

```text
=============================================================================
             MULTI-TENANT REQUEST PIPELINE (RLS & DAPPER)
=============================================================================

[ HTTP Request ] (Authorization: Bearer <JWT>)
        │
[ ASP.NET Core Middleware ]
        ├── Extracts TenantId (e.g., 'Tenant-789') from JWT Claims
        └── Stores in scoped ITenantContext
        │
[ MediatR Query Handler ]
        │
[ ITenantedConnectionFactory.CreateConnectionAsync() ]
        │
        ├── 1. ADO.NET gets open connection from Pool
        ├── 2. DAPPER: Execute("EXEC sp_set_session_context 'TenantId', @id", { id: 'Tenant-789' })
        └── 3. Returns prepared SqlConnection
        │
[ ChargerRepository.GetSitesAsync() ]
        │
        ├── DAPPER: Query("SELECT * FROM Sites")   <-- Notice no WHERE clause!
        │
[ SQL SERVER ENGINE ]
        ├── Intercepts query via Row-Level Security Policy
        ├── Reads Session Context ('Tenant-789')
        ├── Invisibly rewrites to: SELECT * FROM Sites WHERE TenantId = 'Tenant-789'
        └── Returns filtered Tabular Data Stream
```

## 5. API Deep Dive

### Implementing the RLS Connection Factory
We must ensure that *every* connection handed to a repository has its session context set. We do this by decorating our Connection Factory.

```csharp
public class TenantedConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public TenantedConnectionFactory(string connectionString, IHttpContextAccessor httpContextAccessor)
    {
        _connectionString = connectionString;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task<IDbConnection> CreateConnectionAsync(CancellationToken ct = default)
    {
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync(ct);

        // Extract TenantId from JWT Claims
        var tenantIdClaim = _httpContextAccessor.HttpContext?.User.FindFirst("TenantId")?.Value;
        
        if (Guid.TryParse(tenantIdClaim, out var tenantId))
        {
            // Execute session context injection BEFORE returning connection
            await connection.ExecuteAsync(
                "EXEC sp_set_session_context @key=N'TenantId', @value=@TenantId",
                new { TenantId = tenantId });
        }
        else
        {
            throw new UnauthorizedAccessException("Tenant context is missing.");
        }

        return connection;
    }
}
```

### Database RLS Setup (T-SQL)
For the Dapper code above to work securely, the Database Architect must configure SQL Server:

```sql
-- 1. Create a predicate function
CREATE FUNCTION Security.fn_TenantAccessPredicate(@TenantId UNIQUEIDENTIFIER)
    RETURNS TABLE
    WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_accessResult 
    WHERE 
        @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS UNIQUEIDENTIFIER) 
        OR CAST(SESSION_CONTEXT(N'IsSystemAdmin') AS INT) = 1;

-- 2. Apply it to the Sites table
CREATE SECURITY POLICY Security.TenantPolicy_Sites
    ADD FILTER PREDICATE Security.fn_TenantAccessPredicate(TenantId) ON dbo.Sites,
    ADD BLOCK PREDICATE Security.fn_TenantAccessPredicate(TenantId) ON dbo.Sites
    WITH (STATE = ON);
```

## 6. Complete Examples: EV Charging Platform

### Scenario 1: Feature Flags via Dapper
In an Enterprise SaaS, features are often toggled on/off per tenant. 

```csharp
public async Task<bool> IsFeatureEnabledAsync(string featureCode)
{
    // The TenantId is implicitly handled by RLS.
    // We just query if a record exists for the feature.
    const string sql = @"
        SELECT 1 
        FROM TenantFeatureFlags 
        WHERE FeatureCode = @FeatureCode AND IsEnabled = 1";

    var result = await _connection.QuerySingleOrDefaultAsync<int?>(
        sql, 
        new { FeatureCode = featureCode });

    return result.HasValue;
}
```

### Scenario 2: Audit Logging with Dapper & OUTPUT Clause
When a Charger's configuration is modified, we must record the exact change. Instead of doing a `SELECT`, `UPDATE`, and `INSERT` (3 round trips), we use the SQL `OUTPUT` clause in a single Dapper command.

```csharp
public async Task UpdateChargerConfigAsync(Guid chargerId, decimal newMaxKw, string modifiedBy)
{
    const string sql = @"
        UPDATE Chargers
        SET MaxKw = @NewMaxKw
        OUTPUT 
            deleted.Id AS EntityId, 
            'Charger' AS EntityType,
            deleted.MaxKw AS OldValue, 
            inserted.MaxKw AS NewValue,
            @ModifiedBy AS ModifiedBy,
            GETUTCDATE() AS Timestamp
        INTO AuditLogs (EntityId, EntityType, OldValue, NewValue, ModifiedBy, Timestamp)
        WHERE Id = @ChargerId;";

    await _connection.ExecuteAsync(sql, new 
    { 
        ChargerId = chargerId, 
        NewMaxKw = newMaxKw, 
        ModifiedBy = modifiedBy 
    });
}
```

### Scenario 3: Soft Deletes
In SaaS, data is rarely hard-deleted (DELETE FROM). Instead, an `IsDeleted` flag is set.

```csharp
public async Task SoftDeleteSiteAsync(Guid siteId)
{
    // Dapper command for soft delete
    const string sql = @"
        UPDATE Sites 
        SET IsDeleted = 1, DeletedAt = GETUTCDATE() 
        WHERE Id = @SiteId";

    await _connection.ExecuteAsync(sql, new { SiteId = siteId });
}

// In your Repository Read methods, you MUST remember the filter:
public async Task<IEnumerable<SiteDto>> GetActiveSitesAsync()
{
    // RLS handles TenantId, but Application handles IsDeleted
    return await _connection.QueryAsync<SiteDto>("SELECT * FROM Sites WHERE IsDeleted = 0");
}
```

## 7. Performance Implications

### Row-Level Security Overhead
RLS is not free. SQL Server must evaluate the predicate function for every row accessed. 
*   **Poorly Designed RLS:** If the predicate function does a `SELECT` against another table (e.g., checking permissions in a separate `UserRoles` table), it will cause massive performance degradation (Nested Loops).
*   **Optimal RLS:** The predicate should exclusively evaluate `SESSION_CONTEXT` against a column physically on the table (e.g., `WHERE TenantId = SESSION_CONTEXT()`). This adds negligible overhead (nanoseconds per row).

### Connection Reset
When a connection is returned to the ADO.NET pool, `sp_reset_connection` is called. This clears the `SESSION_CONTEXT`. You do not need to worry about Tenant A's context bleeding into Tenant B's request when connections are reused, provided you are properly relying on ADO.NET connection disposal.

## 8. Common Mistakes

### Beginner
*   **Mistake:** Appending a JWT Token or User ID directly into a SQL string.
*   **Correction:** Always use Dapper's parameterized inputs (`new { UserId = id }`) to prevent SQL injection, even if the ID comes from a trusted JWT claim.
*   **Mistake:** Implementing Soft Deletes but forgetting to add `WHERE IsDeleted = 0` to all the `Query` methods.
*   **Correction:** Unlike EF Core, which supports Global Query Filters for Soft Deletes, Dapper requires explicit T-SQL. You must remember to include the filter in every read query, or use an architecture that centralizes query generation.

### Intermediate
*   **Mistake:** Opening a connection, executing `sp_set_session_context`, and then immediately returning it to the pool before executing the business queries.
*   **Correction:** The `SESSION_CONTEXT` lives precisely for the duration of the physical connection session. You must open the connection, set the context, execute the business query, and *then* dispose of the connection, all within the same `using` block lifecycle.
*   **Mistake:** Storing application secrets (like API keys) in SQL Server as plain text, and querying them with Dapper.
*   **Correction:** SQL Server is not a Key Vault. Passwords must be hashed (BCrypt/Argon2). API Keys should be encrypted using Azure Key Vault or AWS KMS before being stored, or stored entirely outside the DB.

### Senior
*   **Mistake:** Creating a generic `GetAll<T>()` Repository method that dynamically builds `SELECT * FROM {typeof(T).Name}` without validating the table name.
*   **Correction:** This is a 2nd-order SQL Injection vulnerability. If a malicious user can manipulate the type name or if the type name is read from an untrusted source, they can query unauthorized tables. Always validate dynamic table/column names against a strict whitelist of known allowed strings.
*   **Mistake:** Performing heavy Role-Based Access Control (RBAC) checks in C# memory after fetching 100,000 rows via Dapper.
*   **Correction:** RBAC should be pushed to the database. Join the target table with the `UserRoles` or `Permissions` table in the Dapper query so SQL Server filters the result set before it hits the network wire.

### Architect
*   **Mistake:** Designing an Audit Logging system that uses a C# background thread to `INSERT` an audit log record into a different database context after a Dapper `UPDATE` completes.
*   **Correction:** If the C# application crashes between the `UPDATE` and the Audit `INSERT`, the audit trail is permanently corrupt and non-compliant. Audit logs must be atomically bound to the mutation. Use SQL `OUTPUT` clauses within the exact same Dapper execution, or use Database Triggers if the schema allows it, ensuring 100% durability.
*   **Mistake:** Designing a multi-tenant DB-per-tenant architecture but storing the connection strings in a centralized SQL table, then executing two separate Dapper connections sequentially per request.
*   **Correction:** This doubles database latency (one query to find the connection string, one for the business logic). Cache the Tenant-to-ConnectionString mapping in a distributed cache (Redis) or memory cache, so the Application can instantly inject the correct string into the Dapper `SqlConnectionFactory` at runtime.

## 9. Interview Questions

### Beginner (10)
1.  **What is a Multi-Tenant application?**
    *Answer:* A single instance of an application serving multiple distinct organizations (tenants), ensuring their data is completely isolated from one another.
2.  **What is SQL Injection?**
    *Answer:* A cyberattack where malicious SQL statements are inserted into entry fields for execution, typically bypassing authentication or exfiltrating data.
3.  **How does Dapper prevent SQL Injection?**
    *Answer:* By requiring developers to use parameterized queries (e.g., passing an anonymous object). Dapper translates these into safe `SqlParameter` objects at the ADO.NET level.
4.  **What is a "Soft Delete"?**
    *Answer:* Marking a record as deleted (e.g., `IsDeleted = 1`) rather than physically removing it from the database table (`DELETE FROM`).
5.  **What is Role-Based Access Control (RBAC)?**
    *Answer:* A security paradigm where system access is determined by the Role assigned to a user (e.g., "Admin", "Viewer"), rather than explicitly assigning permissions to individual users.
6.  **Why should you never hardcode a SQL connection string in a Repository?**
    *Answer:* Because connection strings contain secrets (passwords) and change based on the environment (Dev vs. Prod). They should be injected via configuration.
7.  **What does `sp_reset_connection` do?**
    *Answer:* It is called automatically by ADO.NET when a connection is returned to the pool. It clears temporary tables, SET options, and the `SESSION_CONTEXT`.
8.  **What is an Audit Log?**
    *Answer:* An immutable record detailing who changed what data, when the change occurred, and the old/new values.
9.  **If a user inputs `; DROP TABLE Users;--` into a search field, and you use Dapper parameters, what happens?**
    *Answer:* Dapper safely sends it as a string parameter. SQL Server searches for a record where the column literally matches the string `"; DROP TABLE Users;--"`. No table is dropped.
10. **Can you use Dapper to execute Data Definition Language (DDL) like `CREATE TABLE`?**
    *Answer:* Yes, you can use `ExecuteAsync` to run DDL, though this is typically handled by migration tools.

### Intermediate (10)
11. **Explain what Row-Level Security (RLS) is in SQL Server.**
    *Answer:* RLS allows database administrators to control access to rows in a database table based on the characteristics of the user executing a query (e.g., based on `SESSION_CONTEXT`), applying predicates invisibly at the engine level.
12. **How do you pass a Tenant ID to SQL Server's `SESSION_CONTEXT` using Dapper?**
    *Answer:* `await connection.ExecuteAsync("EXEC sp_set_session_context @key=N'TenantId', @value=@Id", new { Id = tenantId });`
13. **Why is using RLS more secure than relying on `WHERE TenantId = @id` in all your Dapper queries?**
    *Answer:* RLS is enforced at the database engine level. If a new developer forgets the `WHERE` clause in a single Dapper repository method, RLS will still block the query, preventing a catastrophic data leak.
14. **What is the `OUTPUT` clause in T-SQL, and how is it useful for Audit Logging with Dapper?**
    *Answer:* The `OUTPUT` clause returns information from, or expressions based on, each row affected by an `INSERT`, `UPDATE`, or `DELETE`. It allows Dapper to execute a mutation and instantly insert the old/new values into an Audit table without requiring separate SELECT/INSERT round-trips.
15. **How would you implement a "White-Label" feature flag (changing the UI branding per tenant) using Dapper?**
    *Answer:* I would create a `TenantConfigs` table. Upon user login, I would query `connection.QuerySingle<TenantThemeDto>("SELECT LogoUrl, PrimaryColor FROM TenantConfigs WHERE TenantId = @Id", new { Id = tenantId })` and return it in the JWT payload or a dedicated /config API endpoint.
16. **Why is it dangerous to dynamically concatenate a column name for an `ORDER BY` clause, even if using Dapper?**
    *Answer:* Dapper parameters cannot be used for structural SQL components like table names, column names, or `ASC/DESC` keywords. If you concatenate untrusted user input directly into an `ORDER BY` string, it creates a SQL Injection vulnerability.
17. **How do you safely implement dynamic sorting in Dapper?**
    *Answer:* Validate the user input against a hardcoded C# dictionary or whitelist of allowed column names. Only concatenate the trusted, validated string into the SQL query.
18. **If you have a multi-tenant application with a database-per-tenant architecture, how does the DI container resolve the correct `IDbConnection`?**
    *Answer:* The `ISqlConnectionFactory` must inject the `IHttpContextAccessor`. It extracts the Tenant identifier from the request, looks up the corresponding specific Connection String from a cache or configuration, and instantiates the `SqlConnection` using that specific string.
19. **What is Principle of Least Privilege regarding Dapper's SQL Server connection?**
    *Answer:* The database user account defined in the connection string used by Dapper should only have `SELECT`, `INSERT`, `UPDATE`, `DELETE` permissions, and `EXECUTE` on specific SPs. It should never be `db_owner` or `sysadmin`.
20. **Explain how to perform an RBAC check in a Dapper `Query` instead of in C#.**
    *Answer:* By wrapping the business query in an `EXISTS` check: `SELECT * FROM Documents d WHERE d.Id = @DocId AND EXISTS (SELECT 1 FROM UserPermissions up WHERE up.UserId = @UserId AND up.Permission = 'ReadDocuments')`.

### Senior (10)
21. **Analyze the performance implications of an RLS Predicate Function that performs a `JOIN` to a `UserRoles` table.**
    *Answer:* It is highly detrimental. SQL Server must execute that `JOIN` for *every single row* evaluated by the outer query. For a table with 1 million rows, this causes a massive Nested Loop. RLS predicates should ideally only evaluate static `SESSION_CONTEXT` values against physical columns on the target table.
22. **Design an architecture for managing long-running background tasks (e.g., generating end-of-year tax reports for a tenant) where there is no HTTP Context to provide the `TenantId`.**
    *Answer:* Background workers (e.g., Hangfire or Azure Functions) process queued messages. The message payload itself MUST contain the `TenantId`. The worker's connection factory detects it is not in an HTTP context, reads the `TenantId` from the message payload, and executes the `sp_set_session_context` Dapper command before executing the report queries.
23. **You must implement data anonymization for a "Right to be Forgotten" (GDPR) request. How do you structure this Dapper command safely?**
    *Answer:* Soft-deletion is insufficient for GDPR. You must overwrite PII. Create a specific Dapper command: `UPDATE Users SET FirstName = 'ANON', LastName = 'ANON', Email = NEWID(), IsAnonymized = 1 WHERE Id = @UserId`. Never use a generic UPDATE method where a developer might accidentally pass the old values back in.
24. **How do you prevent a malicious developer with access to the production codebase from bypassing RLS by executing `sp_set_session_context` with another Tenant's ID?**
    *Answer:* You cannot solve a compromised execution environment purely with software architecture. However, defense-in-depth dictates separating the API's Database Login from the Migration Login. Furthermore, code reviews, PR policies, and utilizing Managed Identities for Azure resources (no passwords in code) limit the blast radius.
25. **Explain a 2nd-Order SQL Injection vulnerability in a Dapper context.**
    *Answer:* First order: A user inputs malicious text, but Dapper parameterizes it, so it's safely saved to the database as a string. Second order: Later, a background C# job reads that string from the DB, assumes it is trusted because it came from the DB, and dynamically concatenates it into a *new* SQL query string (e.g., executing it as a dynamic table name). The injection payload executes successfully during the second step.
26. **How would you architect a Dapper-based read model for an application that requires highly complex, hierarchical Data Entitlements (e.g., User -> Region -> Branch -> Department -> specific rows)?**
    *Answer:* Doing dynamic JOINs for complex entitlements at runtime is too slow. I would architect an asynchronous materialization process. When entitlements change, a background worker computes a flattened list of allowed `ResourceIds` for a `UserId` and stores them in an `UserAccessMatrix` table or a NoSQL store. The Dapper API query simply does a fast `INNER JOIN` against the pre-computed `UserAccessMatrix` table.
27. **What is the architectural danger of using Dapper to implement a custom authentication system (password hashing/salting) instead of using ASP.NET Core Identity or a third-party IdP (Auth0, Entra)?**
    *Answer:* Building custom crypto and auth flows is a massive security liability. It requires managing salt generation, iteration counts (PBKDF2/Argon2), timing attacks, and token signing. Dapper is a data access tool, not a security framework. The architecture should delegate authentication to a hardened Identity Provider (IdP) via OIDC/OAuth2, and only use Dapper to query business data based on the validated Claims provided by the IdP.
28. **In a database-per-tenant architecture, how do you handle running Dapper database migrations (schema updates) across 5,000 separate databases without causing massive downtime?**
    *Answer:* You cannot use a simple synchronous `foreach` loop on application startup. I would architect an asynchronous migration pipeline using a tool like DbUp or FluentMigrator. The pipeline groups tenants into rings (Canary, Early Adopters, Main, Laggards). It uses parallel fan-out (e.g., Azure Durable Functions) to run the Dapper/DbUp migration scripts against multiple DBs concurrently, verifying each ring's health metrics before proceeding to the next.
29. **How do you ensure Dapper connections are encrypted in transit?**
    *Answer:* This is a connection string configuration. Add `Encrypt=True` (which is default in modern `Microsoft.Data.SqlClient`) and `TrustServerCertificate=False`. Dapper utilizes the underlying driver's TLS encryption automatically.
30. **What is Data Masking, and how does it interact with Dapper?**
    *Answer:* Dynamic Data Masking is a SQL Server feature that obfuscates sensitive data (e.g., returning `XXXX-XXXX-XXXX-1234` for a credit card) for non-privileged users. Dapper interacts with it completely transparently. Dapper simply maps the masked string returned by SQL Server into the C# string property.

### Staff Engineer (5)
31. **Architect a completely immutable, tamper-evident Audit Log for a financial SaaS using Dapper and SQL Server features.**
    *Answer:* I would utilize SQL Server Temporal Tables (System-Versioned tables). The application uses Dapper to perform standard `INSERT/UPDATE/DELETE` operations on the `Invoices` table. SQL Server automatically, transparently moves the previous row versions and exact timestamps into an immutable `InvoicesHistory` table. This provides a tamper-evident audit log with zero C# code changes, zero Dapper mapping overhead, and guaranteed atomicity.
32. **A penetration test reveals that while Dapper prevents SQL Injection, attackers are performing API Enumeration attacks by guessing sequential integer IDs. How do you re-architect the data layer to fix this?**
    *Answer:* Sequential INT primary keys (1, 2, 3) are inherently guessable (Insecure Direct Object Reference - IDOR). We must migrate public-facing identifiers. I would alter the database schema to include an `ExternalId (UNIQUEIDENTIFIER)` for all entities. Dapper queries from the API must strictly use `WHERE ExternalId = @Id`. The internal integer PKs remain for optimal JOIN performance, but are never exposed in DTOs.
33. **Explain the memory and CPU security implications (Denial of Service) of executing `QueryAsync` with `buffered: true` on an unbounded tenant data table.**
    *Answer:* If a tenant has 10 million rows, and an API endpoint lacks pagination, a single HTTP GET request will cause Dapper to execute the query, pull 10 million rows across the network, and allocate a `List<T>` containing 10 million objects. This causes immediate memory exhaustion (OOM), garbage collection pauses, and crashes the entire application instance for all tenants (a classic application-level DoS attack). Strict pagination (OFFSET/FETCH) is mandatory for all multi-tenant list APIs.
34. **Design a mechanism to enforce Tenant Isolation in a shared caching layer (Redis) that sits in front of Dapper Read models.**
    *Answer:* Tenant isolation in cache is just as critical as the database. Every cache key must be deterministically prefixed with the Tenant ID. I would implement an `ITenantCacheService`. When the MediatR handler asks for data, the service constructs the key: `$"tenant:{_tenantContext.Id}:users:active"`. If a cache miss occurs, it calls the Dapper repository, caches the result under the tenanted key, and returns it.
35. **Your SaaS platform is targeted by a massive DDoS attack. The SQL Server connection pool is instantly exhausted, bringing down the DB. Architect a throttling mechanism in front of Dapper.**
    *Answer:* We must implement Bulkhead and Rate Limiting patterns at the Application layer. Using ASP.NET Core Rate Limiting middleware, we enforce a strict request limit per Tenant IP/JWT. Furthermore, we implement a SemaphoreSlim or Polly Bulkhead policy specifically around the `ISqlConnectionFactory`. If concurrent Dapper requests exceed 90 (leaving a buffer in a 100-size pool), the Bulkhead instantly rejects the request (HTTP 429 Too Many Requests) before it even attempts to open a database connection, saving SQL Server from collapse.

### Architect (5)
36. **Architect a compliance-driven data deletion pipeline (GDPR) for a multi-tenant platform spanning SQL Server (via Dapper), Azure Blob Storage, and Elasticsearch.**
    *Answer:* A synchronous API call cannot guarantee atomic deletion across distributed systems. I design a Saga or Choreography. The API inserts a `DeletionRequest` into a SQL table (via Dapper). A durable background orchestrator picks this up. It sends parallel commands to SQL (Execute Dapper Hard Delete/Anonymization), Blob Storage, and Elastic. It tracks the status of each. If a system is down, it retries exponentially. Only when all systems report success does it mark the `DeletionRequest` as Completed.
37. **Evaluate the security and performance trade-offs of using Always Encrypted with Secure Enclaves in SQL Server, and its impact on Dapper queries.**
    *Answer:* Always Encrypted ensures data is encrypted at rest, in transit, and *in memory* within SQL Server. The encryption keys live in the C# application (or Azure Key Vault). Dapper supports this seamlessly via the `Microsoft.Data.SqlClient` driver which intercepts parameters and encrypts them before sending. Trade-offs: Extreme security (even DBAs can't read the data), but severe performance penalties (client-side encryption overhead) and loss of database engine functionality (cannot do `LIKE` searches or complex math on encrypted columns unless using Enclaves, which add further compute overhead).
38. **A multi-tenant architecture uses a Shared Database. Tenant A runs an extremely complex reporting query via Dapper that consumes 100% of the SQL Server CPU, causing timeouts for Tenant B. Architect a solution.**
    *Answer:* This is the "Noisy Neighbor" problem. To solve it without migrating to DB-per-tenant, we must implement SQL Server Resource Governor. We create Resource Pools. Our Dapper Connection Factory is modified: when Tenant A authenticates, we connect using a specific connection string (or set session context) that maps to Tenant A's Resource Pool. The Resource Governor hard-caps Tenant A's CPU usage to 20%, ensuring the other 80% remains available for other tenants, entirely managed at the DB engine level.
39. **Defend the architectural decision to strictly separate the "Read/Write User" connection string from the "Migration User" connection string in a CI/CD pipeline.**
    *Answer:* Principle of Least Privilege. The connection string used by the ASP.NET Core application (and Dapper) should only have DML permissions (Data Manipulation Language). If the application is compromised via a zero-day RCE, the attacker cannot drop tables, alter schemas, or create admin users. DDL permissions (Data Definition Language) must be strictly reserved for a separate "Migration User" credential that is only injected into the ephemeral CI/CD pipeline runner (e.g., GitHub Actions running DbUp) and is never accessible to the runtime application.
40. **Design a mechanism to cryptographically prove to a regulator that an Audit Log row inserted via Dapper has not been tampered with by a rogue DBA modifying the database directly.**
    *Answer:* We must implement a Blockchain-like cryptographic hash chain natively in the application. When Dapper inserts an Audit Log row, it calculates a SHA-256 hash of the current row's data combined with the hash of the *previous* row's hash. This hash is inserted with the row. If a rogue DBA alters any data in a historical row, the hash of that row becomes invalid, and every subsequent row's hash chain is broken. A nightly compliance job runs a Dapper query to recalculate the chain and alerts if a cryptographic mismatch is found.

## 10. Exercises

### Easy
1.  **Session Context Basics:** In a console app, open a connection. Execute `EXEC sp_set_session_context 'MyKey', '123'`. Then, write a Dapper query: `SELECT CAST(SESSION_CONTEXT(N'MyKey') AS VARCHAR)`. Print the result to prove the context was set and retrieved within the session.

### Medium
1.  **OUTPUT Clause Audit:** Create a `Products` table and a `ProductsAudit` table. Write a single C# method that executes a Dapper `UPDATE` command to change a product's price. Use the SQL `OUTPUT` clause within the command to insert the Old Price and New Price directly into the Audit table.

### Hard
1.  **Simulating RLS:** Create a `Tenants` table and a `Users` table (with a `TenantId`). Configure SQL Server Row Level Security based on `SESSION_CONTEXT(N'TenantId')`. Write a Dapper Repository that only executes `SELECT * FROM Users`. Prove that changing the session context before the query automatically filters the users returned, without altering the Dapper `SELECT` string.

### Enterprise
1.  **The Tenanted Connection Factory:** In an ASP.NET Core API, implement the `TenantedConnectionFactory` demonstrated in this chapter. Create a JWT authentication setup. Inject the factory into a CQRS query handler. Issue requests with different Tenant JWTs and verify (using SQL Profiler or the RLS setup from the previous exercise) that the context is correctly scoped to the active HTTP request, and no cross-tenant leakage occurs under concurrent load testing.

## 11. Summary

In an enterprise SaaS environment, trusting the application layer to enforce data isolation is a critical vulnerability waiting to be exploited. By combining the speed of Dapper with the engine-level security of SQL Server Row-Level Security (RLS), you create a defensive architecture that protects against human error and malicious injection.

Furthermore, mastering advanced T-SQL features like the `OUTPUT` clause allows you to satisfy rigorous enterprise compliance requirements—such as immutable audit logging—without sacrificing the performance benefits of a Micro ORM. 

In the final section of this book, we will focus on Production Readiness, exploring how to thoroughly test Dapper repositories using Dockerized Testcontainers, and how to deploy and observe these high-performance applications in the cloud.
