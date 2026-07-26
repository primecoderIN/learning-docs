# Chapter 6: ASP.NET Core Integration and Clean Architecture

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Integrate Dapper securely and efficiently into the ASP.NET Core Dependency Injection (DI) container.
*   Manage `SqlConnection` lifecycles properly to avoid connection pool starvation and thread contention.
*   Architect an enterprise solution using Clean Architecture, ensuring that Dapper strictly resides within the Infrastructure layer.
*   Implement the `IDbConnectionFactory` pattern to abstract away ADO.NET from the Application and Domain layers.
*   Configure Dapper globally within the ASP.NET Core `Program.cs` pipeline for custom type handlers and mappings.

## 2. Introduction

Dapper is exceptionally fast, but it is architecturally unopinionated. Unlike Entity Framework Core, which provides a strongly opinionated `DbContext` that integrates natively into ASP.NET Core's DI container via `AddDbContext<T>`, Dapper provides absolutely nothing out of the box for architectural scaffolding.

If you simply drop `using Dapper;` into your ASP.NET Core Controllers or Minimal APIs and start instantiating `SqlConnection` objects, you will rapidly create a monolithic, tightly coupled, and untestable application. T-SQL strings will leak into your business logic, and database connection strings will be hardcoded across the application.

To build production-scale SaaS applications, we must impose strict boundaries. This chapter explores how to combine Dapper with Clean Architecture and CQRS, ensuring that the blistering speed of Dapper does not compromise the maintainability and testability of your codebase.

## 3. Core Concepts

### Dependency Inversion Principle (DIP)
The foundation of Clean Architecture is the Dependency Inversion Principle. High-level modules (Business Logic) should not depend on low-level modules (Data Access/Dapper). Both should depend on abstractions (Interfaces). Your Application layer must never reference the `Dapper` or `Microsoft.Data.SqlClient` NuGet packages.

### Connection Management in DI
In ASP.NET Core, services can be registered as Transient, Scoped, or Singleton. 
*   A `SqlConnection` must **never** be registered as a Singleton, as it is not thread-safe. 
*   Registering it as Scoped is common, but can lead to issues if multiple independent database calls within a single HTTP request accidentally share a connection state or a transaction.
*   **The Best Practice:** Inject a Factory (e.g., `IDbConnectionFactory`) or the configuration string itself, and instantiate the `SqlConnection` directly inside the Repository method using a `using` block. This leverages ADO.NET connection pooling perfectly.

## 4. Visual Diagrams

```text
=============================================================================
             CLEAN ARCHITECTURE + DAPPER IN ASP.NET CORE
=============================================================================

                   [ 1. Presentation Layer (Web API) ]
                   - Controllers / Minimal APIs
                   - Program.cs (DI Container Setup)
                           │
                           ▼ (Depends On)
                           
               [ 2. Application Layer (Use Cases) ]
               - MediatR Handlers
               - DTOs & Validation
               - Interfaces (IChargerRepository)
               - *NO DAPPER REFERENCES HERE*
                           │
                           ▼ (Depends On)
                           
                 [ 3. Domain Layer (Core) ]
                 - Entities & Value Objects
                 - Business Rules & Invariants
                 - *NO DATA ACCESS OR INFRA REFERENCES HERE*

─────────────────────────────────────────────────────────────────────────────
                             ▲ (Implements)
                             │
            [ 4. Infrastructure Layer (Data Access) ]
            - References Dapper & Microsoft.Data.SqlClient
            - ChargerRepository : IChargerRepository
            - Dapper TypeHandlers
            - Raw T-SQL Strings
```

## 5. API Deep Dive

### Global Dapper Configuration in ASP.NET Core
In ASP.NET Core, global configurations should happen as early as possible in the application lifecycle, typically in `Program.cs`. Dapper relies on static configuration for Type Handlers and Custom Mappers.

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Configure Dapper Globally (BEFORE building the app)
DefaultTypeMap.MatchNamesWithUnderscores = true; // snake_case to PascalCase
SqlMapper.AddTypeHandler(new JsonTypeHandler<TelemetryConfig>());
SqlMapper.AddTypeHandler(new GuidTypeHandler()); 

// Register the Connection Factory
builder.Services.AddSingleton<ISqlConnectionFactory, SqlConnectionFactory>(
    sp => new SqlConnectionFactory(builder.Configuration.GetConnectionString("Default")));

// Register Repositories
builder.Services.AddScoped<IChargerRepository, ChargerRepository>();
```

### The ISqlConnectionFactory Pattern
Instead of injecting an open `IDbConnection`, we inject a factory. This completely hides ADO.NET instantiation from the rest of the application while guaranteeing that connections are fresh from the pool.

```csharp
public interface ISqlConnectionFactory
{
    IDbConnection CreateConnection();
}

public class SqlConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;
    public SqlConnectionFactory(string connectionString) => _connectionString = connectionString;
    
    public IDbConnection CreateConnection()
    {
        return new SqlConnection(_connectionString);
    }
}
```

## 6. Complete Examples: EV Charging Platform

Let's implement a complete Vertical Slice of Clean Architecture for retrieving a Site and its Chargers.

### Step 1: The Domain Layer (Core)
The Domain layer defines the shape of the data and the interface required to fetch it. It does not know about SQL or Dapper.

```csharp
// Project: EVPlatform.Domain

public class SiteDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public List<ChargerDto> Chargers { get; set; } = new();
}

public class ChargerDto
{
    public Guid Id { get; set; }
    public string Status { get; set; }
}

public interface ISiteRepository
{
    Task<SiteDto?> GetSiteWithChargersAsync(Guid siteId, CancellationToken ct);
}
```

### Step 2: The Application Layer (Use Cases)
The Application layer orchestrates the business logic via CQRS (using MediatR). It relies entirely on the interface.

```csharp
// Project: EVPlatform.Application

using MediatR;
using EVPlatform.Domain;

public record GetSiteQuery(Guid SiteId) : IRequest<SiteDto>;

public class GetSiteQueryHandler : IRequestHandler<GetSiteQuery, SiteDto>
{
    private readonly ISiteRepository _siteRepository;

    public GetSiteQueryHandler(ISiteRepository siteRepository)
    {
        _siteRepository = siteRepository;
    }

    public async Task<SiteDto> Handle(GetSiteQuery request, CancellationToken ct)
    {
        var site = await _siteRepository.GetSiteWithChargersAsync(request.SiteId, ct);
        
        if (site == null)
            throw new NotFoundException($"Site {request.SiteId} not found.");
            
        return site;
    }
}
```

### Step 3: The Infrastructure Layer (Dapper Implementation)
This is the ONLY project in the solution that references Dapper. It implements the interface defined in the Domain.

```csharp
// Project: EVPlatform.Infrastructure

using Dapper;
using Microsoft.Data.SqlClient;
using EVPlatform.Domain;

public class SiteRepository : ISiteRepository
{
    private readonly ISqlConnectionFactory _connectionFactory;

    public SiteRepository(ISqlConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<SiteDto?> GetSiteWithChargersAsync(Guid siteId, CancellationToken ct)
    {
        // 1. Get connection from Factory
        using var connection = _connectionFactory.CreateConnection();
        
        // 2. Define raw T-SQL
        const string sql = @"
            SELECT s.Id, s.Name, c.Id, c.Status
            FROM Sites s
            LEFT JOIN Chargers c ON s.Id = c.SiteId
            WHERE s.Id = @SiteId";

        // 3. Create CommandDefinition for cancellation support
        var command = new CommandDefinition(sql, new { SiteId = siteId }, cancellationToken: ct);

        // 4. Execute Multi-Mapping via Dapper
        SiteDto? targetSite = null;

        await connection.QueryAsync<SiteDto, ChargerDto, SiteDto>(
            command,
            (site, charger) =>
            {
                if (targetSite == null) targetSite = site;
                if (charger != null && charger.Id != Guid.Empty)
                {
                    targetSite.Chargers.Add(charger);
                }
                return targetSite;
            },
            splitOn: "Id"
        );

        return targetSite;
    }
}
```

## 7. Performance Implications

### ASP.NET Core Thread Pool vs Database Connection Pool
In ASP.NET Core, incoming HTTP requests are served by Thread Pool threads. Database connections are managed by the ADO.NET Connection Pool. 

If you register your `SqlConnection` as a `Scoped` service and implicitly open it via Dapper, the connection remains checked out of the ADO.NET pool until the HTTP request completes and the DI container disposes of the scoped services. If your API does network I/O *after* the Dapper call (e.g., calling a third-party HTTP API), you are needlessly holding a database connection open. 

By using `ISqlConnectionFactory` and an explicit `using var connection` inside the Repository method, the connection is instantly returned to the ADO.NET pool the microsecond the Dapper query finishes, vastly improving scalability.

## 8. Common Mistakes

### Beginner
*   **Mistake:** Hardcoding the connection string directly inside the Repository class.
*   **Correction:** Always inject the connection string (or the factory) via Dependency Injection. It must be configurable per environment (Development, Staging, Production) via `appsettings.json` or Environment Variables.
*   **Mistake:** Returning an Unbuffered Dapper query directly out of a Repository method into a Controller.
*   **Correction:** The Controller (or its JSON Serializer) will attempt to iterate the `IEnumerable`. If the `using` block in the repository has closed the connection, it will throw an exception. Always return materialized lists (e.g., `ToList()`) from Repositories unless building a highly specialized streaming pipeline.

### Intermediate
*   **Mistake:** Writing raw SQL directly inside MediatR Handlers in the Application Layer.
*   **Correction:** This violates Clean Architecture. SQL is an infrastructure concern. It binds your Application Layer to a specific database technology (SQL Server). Move all SQL and Dapper calls into a Repository or a dedicated Query Object in the Infrastructure layer.
*   **Mistake:** Catching `SqlException` in the Application Layer.
*   **Correction:** `SqlException` belongs to the `Microsoft.Data.SqlClient` namespace. If the Application Layer references it, you have broken the Dependency Inversion Principle. The Infrastructure layer should catch `SqlException` and wrap it in a domain-specific exception (e.g., `DuplicateKeyException`) before throwing it up to the Application Layer.

### Senior
*   **Mistake:** Using `IServiceCollection.AddTransient<IDbConnection>` and injecting `IDbConnection` directly into Repositories.
*   **Correction:** If a Repository requires two consecutive Dapper calls, injecting `IDbConnection` transiently means it gets the same connection instance. However, if the repository is transient and instantiated multiple times, you lose explicit control over when the connection is opened and closed. A Factory is much more explicit and safer for controlling the exact lifetime.
*   **Mistake:** Configuring `SqlMapper.AddTypeHandler` inside a Repository constructor.
*   **Correction:** `AddTypeHandler` modifies global static state. Calling it inside a transient repository constructor will cause it to be called thousands of times per second under load, leading to severe thread contention and exceptions indicating the handler is already registered. It must be called exactly once in `Program.cs`.

### Architect
*   **Mistake:** Sharing Data Transfer Objects (DTOs) between the Presentation Layer (Swagger/API responses) and the Infrastructure Layer (Dapper mapping targets).
*   **Correction:** While tempting for rapid development, this tightly couples the database schema directly to the public API contract. If you rename a column in the database, the public API JSON response changes, breaking external clients. Dapper should map to internal Application Layer DTOs or Domain Entities. The Presentation Layer should independently map those to API ViewModels/Responses.
*   **Mistake:** Attempting to force Dapper to act as a Generic Repository (e.g., `IRepository<T>`) across the entire Clean Architecture.
*   **Correction:** Dapper is designed for specific, optimized SQL. A generic `GetById(int id)` might work, but generic `GetAll(Expression<Func<T, bool>> predicate)` requires building a complex LINQ-to-SQL translator, which completely defeats the purpose of using Dapper. Use explicit repositories (e.g., `IGetActiveChargersQuery`).

## 9. Interview Questions

### Beginner (10)
1.  **What is Dependency Injection in ASP.NET Core?**
    *Answer:* It is a design pattern used to achieve Inversion of Control between classes and their dependencies. ASP.NET Core provides a built-in container to register and resolve these dependencies automatically.
2.  **Which architectural layer should contain the Dapper NuGet package?**
    *Answer:* The Infrastructure layer.
3.  **Should your Domain Layer (Entities) know about SQL Server or Dapper?**
    *Answer:* No. The Domain Layer should be completely ignorant of how data is persisted or retrieved.
4.  **How do you read a connection string in `Program.cs`?**
    *Answer:* `builder.Configuration.GetConnectionString("DefaultConnection")`.
5.  **Why should you avoid returning `IEnumerable<T>` from a repository if the connection is closed inside the repository?**
    *Answer:* Because Dapper's default `Query` method returns a lazily-evaluated sequence if unbuffered (or even if buffered, passing it out can be risky depending on implementation). If the connection is disposed, iterating the sequence later throws an exception.
6.  **What is a Transient service in ASP.NET Core?**
    *Answer:* A service created anew every single time it is requested from the DI container.
7.  **What is a Scoped service?**
    *Answer:* A service created once per client request (e.g., once per HTTP request).
8.  **What is a Singleton service?**
    *Answer:* A service created once the first time it is requested, and that single instance is used for the entire lifetime of the application.
9.  **Why is registering `SqlConnection` as a Singleton dangerous?**
    *Answer:* `SqlConnection` is not thread-safe. Concurrent HTTP requests trying to execute commands on a single shared connection will cause exceptions and data corruption.
10. **Where is the correct place to call `SqlMapper.AddTypeHandler` in ASP.NET Core?**
    *Answer:* In `Program.cs` or `Startup.cs`, during application initialization, before any requests are served.

### Intermediate (10)
11. **Explain the Dependency Inversion Principle (DIP).**
    *Answer:* High-level modules should not depend on low-level modules; both should depend on abstractions. Details should depend on abstractions, not vice-versa. In practice, the Application layer defines an Interface, and the Infrastructure layer implements it.
12. **Why is an `IDbConnectionFactory` preferred over injecting an `IDbConnection` directly?**
    *Answer:* It delays the creation of the connection until it is strictly needed, preventing connections from being held open unnecessarily during non-database operations within the same HTTP request.
13. **How does Dapper interact with the ADO.NET Connection Pool?**
    *Answer:* Dapper does not manage the pool. It relies on the underlying `Microsoft.Data.SqlClient`. When Dapper or a `using` block calls `Dispose()` on the connection, it is returned to the pool, not physically destroyed.
14. **In Clean Architecture, where do you place the raw T-SQL strings?**
    *Answer:* Inside the Repository classes within the Infrastructure Layer.
15. **If a Dapper query fails due to a Unique Constraint violation, how should the Repository handle it?**
    *Answer:* The Repository should catch the `SqlException`, check the error number (e.g., 2601), and throw a custom, infrastructure-agnostic exception (like `DuplicateEntityException`) defined in the Application or Domain layer.
16. **What is the purpose of the `CancellationToken` in an ASP.NET Core request?**
    *Answer:* It signals if the client has aborted the request (e.g., closed the browser). Passing this token down to Dapper via `CommandDefinition` allows SQL Server to abort the query, saving resources.
17. **Can you use EF Core and Dapper in the same ASP.NET Core application?**
    *Answer:* Absolutely. It is a highly recommended pattern (Polyglot Persistence). EF Core is used for Commands (Writes) to leverage Change Tracking and invariants, while Dapper is used for Queries (Reads) for maximum performance.
18. **Why might injecting `IConfiguration` directly into a Repository to get the connection string be considered an anti-pattern?**
    *Answer:* It violates the Single Responsibility Principle. The Repository should only care about data access, not configuration management. It makes unit testing harder. Injecting a specific factory or an `IOptions<DatabaseSettings>` object is superior.
19. **How do you handle environment-specific database configurations (e.g., Dev vs Prod) with Dapper?**
    *Answer:* By managing the connection strings via ASP.NET Core's `appsettings.Development.json` vs `appsettings.Production.json`, or via Environment Variables in the cloud provider. Dapper simply consumes whatever connection string is provided by the DI container.
20. **What is MediatR, and how does it complement Dapper?**
    *Answer:* MediatR is an implementation of the Mediator pattern used to achieve CQRS. It decouples the HTTP Controllers from the business logic. MediatR Query Handlers can inject Dapper repositories to fetch optimized DTOs, keeping the architecture extremely clean.

### Senior (10)
21. **Analyze the architectural flaw of returning Domain Entities directly from Dapper Read/Query operations.**
    *Answer:* Domain Entities encapsulate behavior and invariants (e.g., private setters, validation logic). Dapper bypasses this by using reflection to forcefully inject state from the database. Furthermore, Read models often require joined, flattened data that doesn't match the Entity graph. Dapper should map to flat Read-Specific DTOs, leaving Domain Entities exclusively for the Write/Command stack where they can protect their own invariants.
22. **Design a Resiliency Strategy for Dapper inside an ASP.NET Core microservice connecting to Azure SQL.**
    *Answer:* I would integrate **Polly**. I would create an `IAsyncPolicy` that handles `SqlException` and `TimeoutException`. Instead of decorating every repository method, I would build an `ExecuteWithRetryAsync` extension method on my `IDbConnectionFactory`, or use a Decorator pattern over the Repositories, ensuring that all Dapper calls automatically retry transient cloud connectivity failures with exponential backoff.
23. **How do you unit test an Application Layer service that relies on a Dapper repository?**
    *Answer:* Because the Application Layer relies on the Repository *Interface* (e.g., `IUserRepository`), you use a mocking framework like Moq or NSubstitute to create a fake implementation of the interface. You configure the mock to return predefined DTOs. This tests the Application Layer's business logic in complete isolation from Dapper and SQL Server.
24. **How do you write Integration Tests for a Dapper Repository itself?**
    *Answer:* You cannot mock Dapper. You must use a real database. The modern enterprise standard is using **Testcontainers**. The test spins up a SQL Server Docker container, executes database migrations (e.g., DbUp), instantiates the Dapper Repository pointing to the container's connection string, executes the Dapper query, and asserts the results against the real schema.
25. **Explain the architectural impact of changing `DefaultTypeMap.MatchNamesWithUnderscores = true` in a massive monolith.**
    *Answer:* This changes Dapper's global mapping behavior. If the monolith has hundreds of queries relying on exact mapping or custom `AS` aliases, flipping this flag globally might accidentally map columns that were previously ignored, or break existing custom TypeMaps. Global configuration changes in Dapper must be backed by 100% integration test coverage.
26. **You have a high-traffic API. A developer registers `IDbConnection` as `Scoped` and injects it into 5 different services used during a single HTTP request. What is the performance risk?**
    *Answer:* The connection is opened on the first use and held open for the entire duration of the HTTP request. If the request performs long-running non-DB operations (e.g., uploading a file to Azure Blob Storage), the database connection is held hostage. Under heavy load, this will instantly exhaust the ADO.NET connection pool (default size 100), causing subsequent API requests to timeout waiting for a connection.
27. **Architect a mechanism to automatically apply Row-Level Security (RLS) Tenant Context to every Dapper query in a Clean Architecture.**
    *Answer:* Implement a custom `ITenantConnectionFactory`. This factory injects the `IHttpContextAccessor` to retrieve the current TenantId from the JWT claim. When `CreateConnectionAsync()` is called, it opens the connection and executes `connection.ExecuteAsync("EXEC sp_set_session_context 'TenantId', @TenantId")` via Dapper. The Repositories use this factory and execute standard Dapper queries, inherently protected by SQL Server RLS.
28. **In CQRS, why should the Query stack completely bypass the Domain Layer?**
    *Answer:* The Domain Layer is designed to enforce business rules during state changes (Writes). Querying data (Reads) does not change state. Forcing Queries through Domain Entities adds massive performance overhead (ORM hydration, mapping to entities, mapping entities to DTOs). Bypassing the Domain and using Dapper to map SQL directly to UI-optimized DTOs maximizes performance and perfectly aligns with CQRS principles.
29. **How do you manage Database Migrations (schema changes) when using Dapper in ASP.NET Core?**
    *Answer:* Dapper is agnostic to schemas. You must integrate a dedicated migration tool like **DbUp** or **FluentMigrator**. I configure DbUp to run synchronously during `Program.cs` startup (or via a CI/CD pipeline step). It executes `.sql` scripts to update the database schema. Dapper repositories simply expect the schema to match their queries.
30. **What is the `SqlMapper.TypeMapProvider` and how can it be used architecturally?**
    *Answer:* It allows you to completely override how Dapper discovers properties and fields on a type. Architecturally, you could use this to enforce strict mapping conventions, such as only mapping properties decorated with a custom `[DataColumn]` attribute, or automatically stripping specific prefixes from database columns company-wide.

### Staff Engineer (5)
31. **Architect a globally distributed Read-Heavy API using Dapper, Azure SQL Geo-Replication, and ASP.NET Core. How do you route Dapper queries to the closest read replica?**
    *Answer:* I would configure Azure Traffic Manager to route users to the closest Azure Region. In each region's ASP.NET Core App Service, the `appsettings.json` connection string points to the local Azure SQL Read Replica (using `ApplicationIntent=ReadOnly`). The DI container injects this localized string into the `ISqlConnectionFactory`. Dapper executes standard queries unaware of the geo-distribution, achieving ultra-low latency reads. Write operations must be explicitly routed to the Primary Region's connection string via a separate `IWriteConnectionFactory`.
32. **A monolith has 500 Dapper Repositories. You must implement Distributed Tracing (OpenTelemetry) to trace exactly which SQL query is causing latency across microservices. How do you instrument Dapper globally without modifying 500 classes?**
    *Answer:* Dapper uses the underlying ADO.NET `DbCommand`. OpenTelemetry provides automatic instrumentation for `Microsoft.Data.SqlClient`. By adding `builder.Services.AddOpenTelemetry().WithTracing(t => t.AddSqlClientInstrumentation(options => options.SetDbStatementForText = true));` in `Program.cs`, every single Dapper execution is automatically intercepted at the ADO.NET driver level. The raw SQL, parameters, and execution time are published as Spans to Jaeger/Zipkin without touching a single Dapper repository.
33. **Evaluate the memory implications of Dapper's static Identity Cache in an Enterprise Multi-Tenant application where thousands of unique dynamic SQL strings are generated per second.**
    *Answer:* Dapper caches IL delegates in a static `ConcurrentDictionary` keyed by the exact SQL string. If the application dynamically concatenates values (e.g., `WHERE TenantId = 'abc'`) instead of using parameters, every query generates a unique string. The static dictionary will grow infinitely, eventually causing an `OutOfMemoryException` and crashing the ASP.NET Core process. The architecture MUST enforce strict parameterization to stabilize the hash keys and cap the cache size.
34. **Design a high-performance feature flag system that dictates whether a CQRS Query Handler uses an old EF Core read path or a new Dapper read path, changeable at runtime without restarting the application.**
    *Answer:* I use the Strategy Pattern and `IOptionsSnapshot` (or Azure App Configuration). I define `IOrderQueryStrategy` implemented by both `EfCoreOrderQuery` and `DapperOrderQuery`. The MediatR handler injects both strategies and the `IFeatureManager`. Inside `Handle()`, it evaluates `await _featureManager.IsEnabledAsync("UseDapperReadPath")`. If true, it invokes the Dapper strategy. This allows gradual, percentage-based rollouts and instant rollbacks in production.
35. **Explain the thread starvation risk of executing synchronous Dapper `Query()` inside an ASP.NET Core synchronous Controller action under high load.**
    *Answer:* Synchronous `Query()` blocks the executing thread while waiting for database network I/O. In high load, all Thread Pool threads block simultaneously. ASP.NET Core's Thread Pool injection heuristic (adding new threads) is slow (~1-2 per second). This causes a "Thread Pool Starvation" death spiral; requests queue up, latency spikes to 30+ seconds, and the API returns 503s. The architectural mandate is strict asynchronous execution: `QueryAsync()` with `async/await` from Controller to Repository to ADO.NET.

### Architect (5)
36. **Architect a polyglot persistence strategy for a complex SaaS application. Where does EF Core fit, where does Dapper fit, and how do you prevent the engineering team from fragmenting into warring camps?**
    *Answer:* The architecture is CQRS. The boundary is explicit. 
    **Commands (Writes):** Handled exclusively by EF Core. Why? EF Core provides Change Tracking, Unit of Work, Optimistic Concurrency tokens, and deep domain graph validation. It enforces data integrity.
    **Queries (Reads):** Handled exclusively by Dapper. Why? Reads do not require state tracking. Dapper maps complex SQL Views directly to flat DTOs with zero allocation overhead.
    I prevent fragmentation by making this an immutable Architectural Decision Record (ADR) enforced via automated PR linting tools (e.g., ArchUnitNET) that fail the build if a Query Handler references `DbContext` or a Command Handler references `IDbConnection`.
37. **Your enterprise requires that every SQL query executed must be logged with the ID of the user who initiated the HTTP request. How do you architect this requirement securely with Dapper?**
    *Answer:* I create a custom `DbConnection` wrapper (Decorator Pattern). The Application DI registers an `ITenantedConnectionFactory`. This factory injects `IHttpContextAccessor` and a logger. When `CreateCommand()` is called on the underlying connection, the decorator returns a custom `DbCommand` wrapper. This wrapper intercepts `ExecuteReaderAsync`, extracts the UserID from the HTTP Context, logs the UserID alongside the `CommandText`, and then executes the real command. Dapper operates on the `DbConnection` abstraction, remaining completely unaware of the telemetry interception.
38. **Evaluate the decision to use Dapper vs a Source-Generated ORM (like EF Core Compiled Models or Dapper.AOT) for a microservice deployed to Azure Container Apps with Native AOT compilation enabled.**
    *Answer:* Standard Dapper is fundamentally incompatible with Native AOT because it relies on `System.Reflection.Emit` to generate Intermediate Language at runtime. Native AOT requires all code to be compiled ahead of time and strips out the JIT compiler. As an Architect, if the microservice absolutely requires the microsecond cold-start times of Native AOT, I must abandon standard Dapper and adopt a Source-Generated ORM (like Dapper.AOT experimental packages or RepoDb) which generates the mapping code during the MSBuild phase.
39. **In a domain-driven design (DDD) system, how do you handle the mapping of Value Objects (e.g., `Money` which contains `Amount` and `Currency`) stored in separate columns in SQL Server using Dapper?**
    *Answer:* I configure custom `TypeHandlers` for specific primitive types, or use Dapper's Multi-Mapping. However, the cleanest architectural approach for DDD with Dapper is constructor mapping. The Repository query explicitly aliases the columns: `SELECT Amount AS value, Currency as currency FROM...`. The `Money` Value Object has a parameterized constructor: `public Money(decimal value, string currency)`. Dapper automatically invokes this constructor, perfectly hydrating the immutable Value Object without requiring setter properties.
40. **Justify the architectural cost of maintaining raw T-SQL strings across 500 Dapper Repositories versus using EF Core's LINQ-to-SQL generation.**
    *Answer:* The cost is maintenance burden and lack of compile-time schema checking. The ROI (Return on Investment) is absolute control over execution plans, index utilization, lock hints (e.g., `UPDLOCK`), and memory allocations. In a low-traffic internal app, this ROI is negative; EF Core is better. In a high-traffic SaaS where a 10ms database latency improvement saves $10,000/month in Azure Compute scaling costs, the ROI is overwhelmingly positive. We mitigate the maintenance cost by writing exhaustive Integration Tests using Testcontainers that execute every raw SQL string against a real schema during CI/CD.

## 10. Exercises

### Easy
1.  **Factory Setup:** In an ASP.NET Core Web API, create an `ISqlConnectionFactory` interface and implementation. Register it as a Singleton in `Program.cs`. 

### Medium
1.  **Clean Repository:** Create an `IUserRepository` interface in a Domain layer. Create a `UserRepository` in an Infrastructure layer. Inject the `ISqlConnectionFactory` into the repository. Implement a `GetUserByIdAsync` method using Dapper and a `using` block for the connection. Call this from a Controller.

### Hard
1.  **CQRS Implementation:** Install the `MediatR` package. Create a `GetActiveUsersQuery` and a `GetActiveUsersQueryHandler`. Inject the `IUserRepository` into the handler. Ensure the controller only depends on `IMediator`, completely decoupling the API from the database.

### Enterprise
1.  **Polly Resiliency:** Install the `Polly` NuGet package. Create a Decorator around your `ISqlConnectionFactory` (or create an extension method) that wraps every Dapper execution in an `AsyncRetryPolicy`. The policy should catch `SqlException`, check if it's a transient error (e.g., error number 40613), and retry the operation up to 3 times with exponential backoff. Test it by temporarily disabling your LocalDB network adapter during execution.

## 11. Summary

Dapper is a surgical instrument. Without a guiding architecture, it will slice through your application boundaries, leaving a tangled mess of hardcoded SQL strings and leaked database dependencies.

By strictly adhering to Clean Architecture, utilizing the Dependency Inversion Principle, and managing connection lifecycles via the `ISqlConnectionFactory` pattern, you can harness Dapper's extreme performance safely. The resulting architecture ensures that your Domain layer remains pristine, your Application layer orchestration remains database-agnostic via CQRS, and your Infrastructure layer efficiently marshals data to and from SQL Server. 

In the next chapter, we will explore the Enterprise SaaS Perspective, tackling Multi-Tenancy, Row-Level Security, and Audit Logging using Dapper.
