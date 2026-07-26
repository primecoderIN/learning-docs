# Chapter 9: Deployment, Azure SQL, and Observability

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Deploy a Dapper-based ASP.NET Core application to Microsoft Azure seamlessly utilizing Azure SQL Database.
*   Implement passwordless authentication using Azure Managed Identities (Entra ID) within your Dapper `ISqlConnectionFactory`.
*   Architect an automated CI/CD pipeline that safely executes database migrations (DbUp/FluentMigrator) prior to deploying Dapper code.
*   Instrument Dapper queries with OpenTelemetry to achieve distributed tracing and monitor execution times across microservices.
*   Analyze SQL Server execution plans directly from production telemetry without modifying repository code.

## 2. Introduction

Building a high-performance data access layer with Dapper on your local development machine is only half the battle. The true test of an Enterprise SaaS application is how it behaves in a distributed cloud environment under heavy, unpredictable load. 

When transitioning to production—specifically platforms like Azure SQL—the rules change. Network connections are no longer perfectly stable; they become transient. Hardcoded connection strings with passwords become severe security liabilities. Furthermore, when a query that took 10 milliseconds locally suddenly takes 5 seconds in production, you cannot simply attach a debugger. You need rigorous, proactive observability.

This chapter bridges the gap between writing Dapper code and operating it at scale. We will explore how to secure database access using Azure Managed Identities, how to configure resilience for cloud databases, and how to illuminate Dapper's internal execution using OpenTelemetry.

## 3. Core Concepts

### Azure Managed Identities (Passwordless Auth)
Historically, connection strings contained a `User ID` and `Password`. These secrets were frequently leaked in source control. Azure Managed Identities solve this by assigning an Azure Active Directory (Entra ID) identity directly to your compute resource (e.g., Azure App Service). Your application requests a temporary access token from Azure and passes this token to the `SqlConnection`. Dapper operates exactly the same, but passwords are completely eliminated from your configuration.

### Cloud Transient Faults
Cloud environments are multi-tenant at the hardware level. Azure SQL databases are routinely shifted between physical nodes for load balancing or patching. When this happens, open TCP connections are forcefully terminated. A robust Dapper architecture must expect, catch, and automatically retry these transient network errors (like Error 40613).

### Observability and OpenTelemetry
Observability is the ability to understand internal system states by analyzing external outputs (metrics, logs, and traces). OpenTelemetry (OTel) is the industry standard for this. Because Dapper operates directly on ADO.NET (`Microsoft.Data.SqlClient`), we do not need to instrument Dapper specifically. By instrumenting the underlying SQL driver via OTel, every single Dapper query's SQL string, execution duration, and parameter list is automatically intercepted and exported to monitoring tools like Azure Application Insights, Jaeger, or Datadog.

## 4. Visual Diagrams

```text
=============================================================================
             CLOUD DEPLOYMENT & OBSERVABILITY ARCHITECTURE
=============================================================================

[ Azure App Service (Compute) ]
       │
       ├── 1. Requests Auth Token via Managed Identity
       │
[ Azure Entra ID (Active Directory) ]
       │
       ├── 2. Issues short-lived access token
       │
[ ASP.NET Core Application ]
       │
       ├── 3. ISqlConnectionFactory injects token into SqlConnection
       ├── 4. OpenTelemetry intercepts the DbCommand
       │
       ├── 5. DAPPER executes query using the token
       │          │
       │          ▼
       │  [ Azure SQL Database ]
       │          │
       ├── 6. OpenTelemetry ends the span (calculates duration)
       │
[ Azure Application Insights (or Datadog/Jaeger) ]
       ▲
       │
       └── 7. Telemetry exported: { "Query": "SELECT...", "DurationMs": 42 }
```

## 5. Complete Examples: EV Charging Platform

### Scenario 1: Managed Identity Connection Factory
We must refactor our `ISqlConnectionFactory` from Chapter 6 to support Azure Managed Identities, allowing our API to connect to Azure SQL without a password.

```csharp
using Microsoft.Data.SqlClient;
using Azure.Identity; // Required for DefaultAzureCredential
using Azure.Core;

public class ManagedIdentityConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;

    // This connection string looks like: "Server=tcp:evplatform.database.windows.net,1433;Initial Catalog=EVDB;"
    // Note: There is NO User ID or Password.
    public ManagedIdentityConnectionFactory(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("AzureSql");
    }

    public async Task<IDbConnection> CreateConnectionAsync(CancellationToken ct = default)
    {
        var connection = new SqlConnection(_connectionString);

        // Fetch a token for Azure SQL. 
        // DefaultAzureCredential works locally (Visual Studio/Azure CLI) AND in Azure!
        var credential = new DefaultAzureCredential();
        var tokenRequestContext = new TokenRequestContext(new[] { "https://database.windows.net//.default" });
        var token = await credential.GetTokenAsync(tokenRequestContext, ct);

        // Inject the token into the ADO.NET connection
        connection.AccessToken = token.Token;

        // Note: We don't open it here if we want the consumer to manage it, 
        // but typically in factories we return it opened.
        await connection.OpenAsync(ct);

        return connection;
    }
}
```

### Scenario 2: OpenTelemetry Integration for Dapper
We want every Dapper query to be logged to Application Insights with the exact T-SQL text. We do this entirely in `Program.cs` without changing a single Dapper repository.

```csharp
// Program.cs
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// Configure OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
    {
        tracerProviderBuilder
            .AddAspNetCoreInstrumentation() // Traces incoming HTTP requests
            .AddSqlClientInstrumentation(options => 
            {
                // CRITICAL: This enables capturing the Dapper SQL strings
                options.SetDbStatementForText = true;
                options.RecordException = true;
            })
            .AddAzureMonitorTraceExporter(); // Exports to Application Insights
    });

// (Register Dapper factories and build app...)
```

### Scenario 3: Transient Fault Handling with Polly
Azure SQL occasionally drops connections. We wrap our Dapper factory creation in a Polly retry policy.

```csharp
using Polly;
using Polly.Retry;
using Microsoft.Data.SqlClient;

public class ResilientConnectionFactory : ISqlConnectionFactory
{
    private readonly ISqlConnectionFactory _innerFactory;
    private readonly AsyncRetryPolicy _retryPolicy;

    public ResilientConnectionFactory(ISqlConnectionFactory innerFactory)
    {
        _innerFactory = innerFactory;
        
        // Retry 3 times, with exponential backoff
        _retryPolicy = Policy
            .Handle<SqlException>(ex => IsTransient(ex))
            .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
    }

    public async Task<IDbConnection> CreateConnectionAsync(CancellationToken ct = default)
    {
        // If the connection attempt fails due to a transient Azure network drop, 
        // Polly will automatically wait and try again.
        return await _retryPolicy.ExecuteAsync(async () => 
            await _innerFactory.CreateConnectionAsync(ct));
    }

    private bool IsTransient(SqlException ex)
    {
        // Common Azure SQL transient error codes (e.g., 40613 for database unavailable)
        int[] transientErrorNumbers = { 40613, 40197, 40501, 49918, 40540, 40143 };
        return transientErrorNumbers.Contains(ex.Number);
    }
}
```

## 6. Continuous Integration / Continuous Deployment (CI/CD)

Dapper assumes your database schema matches your C# DTOs. Therefore, database schema migrations must execute *before* your ASP.NET Core application restarts with the new Dapper queries.

**The Golden CI/CD Pipeline Rule:**
1. **Build Phase:** Compile C#. Run Unit Tests. Run Testcontainer Integration Tests (as seen in Chapter 8).
2. **Release Phase (Database):** Execute `DbUp` or `FluentMigrator` against the Production Azure SQL database to apply `ALTER TABLE` scripts.
3. **Release Phase (Application):** Deploy the new ASP.NET Core DLLs to the Azure App Service.

If you reverse steps 2 and 3, the new Dapper code will query columns that do not yet exist, instantly crashing production.

## 7. Performance Implications

### Token Caching (Managed Identity)
Calling `DefaultAzureCredential.GetTokenAsync()` on every single Dapper repository call introduces a massive latency spike (often 50-100ms per token request). 
*Azure.Identity* internally caches tokens, but it is highly recommended to configure token lifetime caching manually in highly concurrent environments to ensure connection acquisition stays under 5ms.

### OpenTelemetry Overhead
Capturing the raw SQL text (`SetDbStatementForText = true`) is invaluable, but the telemetry payload must be serialized and transmitted to the observability backend (like Application Insights). Under extreme load (e.g., 10,000 queries per second), this telemetry can consume more CPU than Dapper itself. In such scenarios, use OTel sampling (e.g., only tracing 10% of requests) to maintain performance.

## 8. Common Mistakes

### Beginner
*   **Mistake:** Pushing `appsettings.json` containing a production Azure SQL password to a public GitHub repository.
*   **Correction:** Never put secrets in source control. Use Azure Key Vault, GitHub Secrets, or Azure Managed Identities (Passwordless auth).
*   **Mistake:** Writing a `try/catch` block around every single Dapper query to manually retry on failure.
*   **Correction:** This pollutes business logic. Centralize retry logic using Polly, either via a Decorator on the Connection Factory or MediatR Pipeline Behaviors.

### Intermediate
*   **Mistake:** Assuming Dapper has built-in execution logging like EF Core's `.LogTo(Console.WriteLine)`.
*   **Correction:** Dapper is silent by design. You must either use a `DbCommand` interceptor (like MiniProfiler), or rely on `Microsoft.Data.SqlClient` EventSources (via OpenTelemetry) to see the generated SQL.
*   **Mistake:** Trying to use `CommandFlags.Pipelined` to improve Azure SQL performance over high-latency networks.
*   **Correction:** Standard SQL Server and Azure SQL do not support true command pipelining on the TDS protocol. It will not improve performance and may cause connection state corruption.

### Senior
*   **Mistake:** Enabling `SetDbStatementForText = true` in OpenTelemetry but using string concatenation in Dapper instead of parameters.
*   **Correction:** If you concatenate user input into the SQL string, that PII (Personally Identifiable Information) or password will be exported directly into Application Insights in plain text, violating GDPR and PCI-DSS compliance. When you use proper Dapper parameters (`new { Id = 1 }`), OTel only logs the parameter names (`@Id`), keeping PII out of the logs.
*   **Mistake:** Using `System.Data.SqlClient` instead of `Microsoft.Data.SqlClient`.
*   **Correction:** `System.Data.SqlClient` is legacy and lacks modern Azure authentication features (like Managed Identity) and modern telemetry hooks. Always ensure your Dapper projects reference the newer `Microsoft` namespace.

### Architect
*   **Mistake:** Deploying Dapper schema migrations that contain `DROP COLUMN` while the old version of the API is still serving traffic (Zero-Downtime Deployment failure).
*   **Correction:** A `DROP COLUMN` breaks the old API instantly. You must use the Expand and Contract pattern. Deploy the schema change (Expand), deploy the new Dapper code, and then drop the column days later in a separate migration (Contract) once all traffic has drained from the old application instances.
*   **Mistake:** Implementing distributed tracing (TraceId) in HTTP headers, but losing the trace context when a background worker executes a Dapper query.
*   **Correction:** Traces must flow across asynchronous boundaries. The background worker (e.g., RabbitMQ consumer) must extract the `TraceId` from the message payload and manually start a new OpenTelemetry Activity linked to that parent ID before executing the Dapper connection, ensuring the database query is correctly attached to the original HTTP request in Jaeger/App Insights.

## 9. Interview Questions

### Beginner (10)
1.  **What is a Transient Fault in a cloud database?**
    *Answer:* A temporary network or infrastructure error (like a momentary connection drop) that resolves itself if the operation is retried a few seconds later.
2.  **How do you prevent passwords from being stored in your connection string?**
    *Answer:* By using Azure Managed Identities (Entra ID) to acquire short-lived OAuth tokens for authentication instead of a static password.
3.  **What is OpenTelemetry?**
    *Answer:* An open-source, vendor-agnostic framework for generating, capturing, and exporting telemetry data (traces, metrics, logs) from applications.
4.  **Does Dapper require a specific logging framework to log SQL queries?**
    *Answer:* No. Dapper doesn't log anything natively. You must instrument the underlying ADO.NET driver.
5.  **Which NuGet package should you use for SQL Server connections in modern .NET?**
    *Answer:* `Microsoft.Data.SqlClient` (not the legacy `System.Data.SqlClient`).
6.  **What is Polly?**
    *Answer:* A .NET resilience and transient-fault-handling library that allows developers to express policies such as Retry, Circuit Breaker, and Timeout in a fluent manner.
7.  **In a CI/CD pipeline, should you run database schema migrations before or after deploying the new C# code?**
    *Answer:* Before. The new Dapper queries might rely on tables or columns that the migration creates.
8.  **What is a TraceId?**
    *Answer:* A globally unique identifier assigned to a single transaction (like a web request) that flows through all microservices and database calls, allowing you to trace the entire lifecycle of the request.
9.  **Can I see the exact SQL generated by Dapper in Application Insights?**
    *Answer:* Yes, if you enable `SetDbStatementForText = true` in the OpenTelemetry SQL Client instrumentation configuration.
10. **What is a "Cold Start" in Azure App Services or Azure Functions?**
    *Answer:* The delay experienced when a cloud resource spins up from zero instances. During this time, the initial Dapper connection might take significantly longer to establish.

### Intermediate (10)
11. **Explain the Expand and Contract pattern for database migrations.**
    *Answer:* A strategy for zero-downtime deployments. You first "Expand" the database (add new columns/tables) without breaking existing ones. You deploy the new Dapper code to use them. Later, you "Contract" the database (drop old columns) in a subsequent deployment.
12. **How does `DefaultAzureCredential` work locally versus in production?**
    *Answer:* It attempts a chain of authentications. Locally, it uses your Visual Studio, VS Code, or Azure CLI login. In production, it seamlessly detects the environment and uses the Managed Identity of the Azure resource.
13. **Why must you be careful when logging Dapper SQL strings regarding GDPR?**
    *Answer:* If you concatenate PII (like a user's name or email) directly into the SQL string instead of using parameters, that PII is permanently written to your monitoring logs, causing severe compliance violations.
14. **What is a Circuit Breaker?**
    *Answer:* A resilience pattern. If a Dapper query fails repeatedly (e.g., the database is down), the circuit "opens" and immediately fails fast for subsequent requests without attempting the DB connection, preventing system overload while the DB recovers.
15. **How do you inject the Azure AD Token into the database connection?**
    *Answer:* By setting the `AccessToken` property on the `SqlConnection` object before calling `OpenAsync()`.
16. **What is the difference between Application Insights and OpenTelemetry?**
    *Answer:* Application Insights is a proprietary Microsoft monitoring platform. OpenTelemetry is a vendor-neutral standard for collecting data. Modern Application Insights utilizes OpenTelemetry as its data collection engine.
17. **If a transient network error occurs *during* the execution of an `ExecuteAsync` `INSERT` command, is it safe to automatically retry it via Polly?**
    *Answer:* Only if the operation is Idempotent. If the network dropped after SQL Server committed the insert, but before the ACK reached the application, retrying will result in a duplicate record.
18. **How do you identify a transient SQL Exception in code?**
    *Answer:* By checking the `SqlException.Number` property against a known list of transient error codes provided by Microsoft (e.g., 40613 for Database Unavailable).
19. **What is DbUp?**
    *Answer:* A .NET library that helps you deploy changes to SQL Server databases by tracking which SQL scripts have already been run, making schema migrations predictable in CI/CD.
20. **Does Dapper cause N+1 query problems in monitoring dashboards?**
    *Answer:* If a developer writes a `foreach` loop that executes a Dapper `QuerySingle` 100 times, OpenTelemetry will faithfully report 100 distinct database spans, drastically increasing the duration of the parent HTTP request span.

### Senior (10)
21. **Analyze the performance implications of acquiring a Managed Identity token on every Dapper repository invocation under a load of 5,000 requests per second.**
    *Answer:* Acquiring a token requires an HTTP call to the Azure Instance Metadata Service (IMDS). Doing this 5,000 times a second will cause massive latency and potentially throttle the IMDS endpoint. Tokens must be cached in memory. The `Azure.Identity` SDK does some caching, but high-throughput systems require explicit token lifetime caching to ensure connection acquisition remains strictly local.
22. **Architect a mechanism to automatically inject a `CorrelationId` into SQL Server Extended Events to map Dapper queries to application logs without changing the SQL string.**
    *Answer:* We can use `sp_set_session_context` via Dapper. When the connection opens, we execute `EXEC sp_set_session_context 'CorrelationId', @TraceId`. In SQL Server, we configure an Extended Events session that captures `SESSION_CONTEXT(N'CorrelationId')` alongside the query text, seamlessly linking DB engine performance metrics to application traces.
23. **You are migrating a monolithic Dapper application to Kubernetes. How do you manage the database schema migrations?**
    *Answer:* Database migrations should not execute on application startup (e.g., in `Program.cs`) in Kubernetes, because scaling up 10 pods simultaneously causes race conditions and DB locks as all 10 try to run the migration. Migrations must run as a Kubernetes `Job` (init container) or a dedicated step in the deployment pipeline, ensuring it runs exactly once before the API pods are updated.
24. **Explain how OpenTelemetry Context Propagation works across asynchronous Dapper calls in C#.**
    *Answer:* OpenTelemetry relies on `System.Diagnostics.Activity`. `Activity` flows implicitly through asynchronous code via `AsyncLocal<T>`. When you `await` a Dapper call, the underlying SQL client instrumentation reads `Activity.Current` to attach the database span to the current execution context, ensuring the trace hierarchy remains intact regardless of thread-pool thread switching.
25. **If an Azure SQL database hits its DTU/vCore limit, what behavior will Dapper exhibit?**
    *Answer:* Dapper will experience severe performance degradation. Simple queries will exceed their `CommandTimeout` (default 30s) and throw `SqlException` (Timeout). You will also likely see error 10928 (Resource ID: %d. The %s limit for the database is %d and has been reached), which should be caught by Polly and handled gracefully via rate limiting.
26. **How do you architect Blue/Green deployments for an application using Dapper and SQL Server?**
    *Answer:* Blue/Green requires two versions of the API running simultaneously against the *same* database. Therefore, the database schema must be compatible with both the V1 and V2 Dapper queries simultaneously. This strictly mandates the Expand and Contract pattern. Destructive schema changes are impossible during the overlap window.
27. **Why might you use `SqlConnectionStringBuilder` instead of raw string manipulation in your Connection Factory?**
    *Answer:* `SqlConnectionStringBuilder` safely handles parsing and modifying connection properties (like overriding `Connect Timeout` or `Max Pool Size` for specific background jobs) without risking syntax errors or injection vulnerabilities that occur with manual string concatenation.
28. **In a highly concurrent system using Polly retries for Dapper, what is "Exponential Backoff with Jitter" and why is it necessary?**
    *Answer:* Exponential backoff increases the wait time between retries (1s, 2s, 4s). If 100 concurrent requests fail simultaneously and all wait exactly 1 second, they will all retry simultaneously, causing a "Thundering Herd" that crushes the recovering database. "Jitter" adds a random millisecond variance (e.g., 1.1s, 0.9s, 2.3s) to desynchronize the retries and smooth out the load.
29. **How do you monitor for Dapper "Cache Bloat" (Identity Dictionary memory leaks) in production?**
    *Answer:* Standard OpenTelemetry doesn't track Dapper's internal dictionaries. You must architect a custom metric. You can periodically monitor `Dapper.SqlMapper.GetCachedSQLCount()` and export it as an OpenTelemetry Metric (Gauge). If this gauge climbs infinitely, you have a dynamic SQL injection issue busting the cache.
30. **What is the `ApplicationIntent=ReadOnly` connection string parameter, and how do you use it architecturally with Dapper?**
    *Answer:* It instructs Azure SQL to route the connection to a geo-replicated Secondary read-only replica. Architecturally, you implement CQRS. The Command handlers inject an `IWriteConnectionFactory` (Primary DB). The Query handlers inject an `IReadConnectionFactory` (configured with `ApplicationIntent=ReadOnly`). Dapper executes identically, but read traffic is entirely offloaded from the primary node.

### Staff Engineer (5)
31. **Architect a seamless failover strategy for Dapper connecting to an Azure SQL Failover Group where the DNS switch takes up to 30 seconds.**
    *Answer:* A 30-second DNS switch will cause total API failure. The architecture must utilize a highly robust Polly policy wrapping the `ISqlConnectionFactory`. The policy catches transport-level exceptions (Error 233, 10054). It applies a Circuit Breaker combined with a Retry policy that waits *at least* 30 seconds (e.g., 3 retries, 10s apart). During the outage, the Circuit Breaker fails fast to shed load. Once the DNS propagates, the retry succeeds, acquiring a connection to the promoted secondary region transparently to the end user.
32. **A production incident reveals that OpenTelemetry SQL instrumentation is dropping spans under extreme load. Analyze the root cause and resolution.**
    *Answer:* OpenTelemetry exporters (like OTLP) use bounded memory queues and background threads to batch and transmit telemetry. Under extreme load, the Dapper query volume exceeds the exporter's transmission bandwidth, filling the queue. The exporter drops spans to prevent OOM exceptions. Resolution: 1. Implement tail-based or head-based sampling to reduce volume. 2. Tune the `BatchExportProcessorOptions` (increase `MaxQueueSize` and `MaxExportBatchSize`), ensuring the compute instance has sufficient RAM.
33. **Your Dapper application requires accessing cross-database tables (e.g., `DB1.dbo.Users` and `DB2.dbo.Orders`). Evaluate the viability of this in Azure SQL Database and the required Dapper architecture.**
    *Answer:* Azure SQL Database (PaaS) does not support native cross-database queries using three-part names (unlike SQL Server on-prem or Managed Instance). To achieve this, you must architect **Elastic Database Queries** (External Tables) at the DB tier, or architect API-level orchestration (using Dapper to fetch Users from DB1, Orders from DB2, and stitching them in C# memory). Elastic Queries introduce severe performance penalties; memory stitching via Dapper is preferred for high-throughput reads.
34. **Design an infrastructure-as-code (Terraform/Bicep) strategy that provisions Azure SQL, assigns a Managed Identity to an App Service, and creates the required database user for Dapper without human intervention.**
    *Answer:* 1. Terraform creates the App Service and enables System-Assigned Managed Identity. 2. Terraform creates Azure SQL. 3. Terraform uses the `azuread` and `azurerm_mssql_database` providers to execute a post-deployment script (via an Azure Container Instance acting as an admin) that runs: `CREATE USER [<AppServiceIdentityName>] FROM EXTERNAL PROVIDER; ALTER ROLE db_datareader ADD MEMBER [<AppServiceIdentityName>];`. This achieves fully automated, zero-touch credential provisioning.
35. **The security team mandates that all SQL connections must originate from a private IP address (VNet) and never traverse the public internet. How does this impact your Dapper API and Azure SQL configuration?**
    *Answer:* Azure SQL must be configured with an **Azure Private Endpoint** (Private Link), disabling public network access entirely. The ASP.NET Core API must be deployed into a VNet (e.g., App Service VNet Integration or AKS). Dapper's connection string does not change, but DNS resolution internally routes `*.database.windows.net` to the 10.0.x.x private IP. This guarantees network-level isolation for all Dapper queries.

### Architect (5)
36. **Architect a global, multi-region active-active SaaS utilizing Dapper and Azure Cosmos DB for PostgreSQL (Citus). How do you adapt Dapper's T-SQL paradigms to a distributed PostgreSQL engine?**
    *Answer:* Dapper works flawlessly with Npgsql. However, active-active distributed databases change the data access paradigm entirely. Dapper queries must be re-architected to include the **Distribution Column** (Shard Key) in every `WHERE` clause to avoid catastrophic broadcast queries. Stored procedures (PL/pgSQL) are preferred for complex operations as they execute locally on the shard. The architecture must abandon SQL Server-specific features (`MERGE`, `OUTPUT`) in favor of Postgres native `INSERT ... ON CONFLICT DO UPDATE RETURNING`.
37. **Defend the choice to use OpenTelemetry over proprietary APM agents (like Application Insights SDK or New Relic Agent) for monitoring Dapper performance.**
    *Answer:* Vendor lock-in is an architectural risk. Proprietary agents require specific binaries, often use reflection/profiling APIs that degrade performance, and couple the codebase to a specific vendor. OpenTelemetry is natively integrated into .NET and `Microsoft.Data.SqlClient`. By using OTel, we export a standardized OTLP payload. We can route this to Jaeger today, Datadog tomorrow, and Application Insights next week, changing only the configuration, guaranteeing observability portability across cloud environments.
38. **A monolithic Dapper application is being strangled into microservices. The monolith uses explicit `SqlTransaction` boundaries spanning 5 aggregate roots. How do you architect the decomposition of these transactions?**
    *Answer:* We cannot stretch an ADO.NET `SqlTransaction` across multiple microservices. We must fundamentally redesign the business process using the **Saga Pattern**. The initiating microservice executes its Dapper update and commits an event to an Outbox. A message broker distributes the event. Subsequent microservices execute their Dapper updates independently. If a downstream service fails, it publishes a Compensating Event, triggering the upstream services to execute Dapper commands to reverse the transaction. ACID is replaced with BASE (Basically Available, Soft state, Eventual consistency).
39. **Evaluate the security perimeter of utilizing Azure SQL Always Encrypted with Secure Enclaves against a Dapper architecture handling highly classified health data (HIPAA).**
    *Answer:* Always Encrypted guarantees the DBA cannot read the data. Dapper supports it via the SqlClient driver. The driver transparently encrypts parameters in application memory before transmission. With Secure Enclaves, SQL Server can perform pattern matching (`LIKE`) and sorting on encrypted data securely inside a hardware-isolated enclave. The architectural trade-off is performance and setup complexity: The C# application must maintain access to the Column Master Key (CMK) in Key Vault, and Dapper queries must perfectly map to the encrypted column types, but it provides the ultimate cryptographically proven defense-in-depth required for HIPAA.
40. **Summarize the ultimate architectural goal of the data access layer in a modern enterprise system.**
    *Answer:* The ultimate goal is absolute transparency and deterministic performance. The data access layer (Dapper) should be an invisible conduit that seamlessly marshals state between the Application Layer's Domain logic and the persistence engine, without leaking technical constraints (SQL strings, connection pooling, transient errors) into the business logic. It must be observable, scalable, secure by default via Identity integration, and testable via CI/CD automation.

## 10. Exercises

### Easy
1.  **Managed Identity Prep:** Modify an existing `ISqlConnectionFactory` to include `DefaultAzureCredential`. Note: You will need the `Azure.Identity` NuGet package.

### Medium
1.  **Polly Integration:** Create a `ResilientConnectionFactory` decorator (as shown in the chapter) using the `Polly` NuGet package. Configure it to retry up to 3 times for transient SQL errors.

### Hard
1.  **OpenTelemetry Tracing:** Create an ASP.NET Core Web API. Install `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, and `OpenTelemetry.Instrumentation.SqlClient`. Configure them in `Program.cs`. Run the API and make a request that executes a Dapper query. Ensure `SetDbStatementForText` is true. (You can view the output in the console using the Console Exporter).

### Enterprise
1.  **Full Cloud Deployment Pipeline:** Architect a GitHub Actions YAML file (or Azure DevOps YAML) that performs the following:
    *   Compiles a .NET Dapper application.
    *   Spins up a SQL Server Testcontainer and runs Integration Tests.
    *   Executes a DbUp console application against a real Azure SQL Development database to run migrations.
    *   Deploys the compiled API to an Azure App Service.
    *   *Conceptual:* Describe how you would securely pass the Azure SQL connection string to the DbUp migration step using GitHub Secrets, while the App Service uses Managed Identity.

## 11. Summary

Transitioning a Dapper application from localhost to the cloud is a true test of architectural rigor. By leveraging Azure Managed Identities, you eliminate the single greatest security vulnerability in data access: hardcoded passwords. By implementing Polly resiliency policies, you harden your application against the realities of transient cloud networks. 

Most importantly, integrating OpenTelemetry pulls Dapper out of the shadows. It provides the empirical data necessary to prove that your queries are performing optimally in production, allowing you to identify bottlenecks before they impact users.

You have now completed the technical journey through Dapper. In the final conclusion, we will review the master checklist for deploying Dapper to production and summarize the core tenets of Enterprise .NET architecture.
