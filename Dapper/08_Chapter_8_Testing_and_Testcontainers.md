# Chapter 8: Testing, Mocking, and Testcontainers

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Understand why attempting to mock Dapper extension methods is an architectural anti-pattern.
*   Design a testing strategy that strictly separates Application Layer Unit Testing from Infrastructure Layer Integration Testing.
*   Utilize **Testcontainers** to spin up ephemeral, production-equivalent SQL Server instances within your test suite.
*   Implement the Transaction Rollback Pattern to ensure pristine database state between test runs without rebuilding the schema.
*   Evaluate the dangers of using In-Memory databases (like SQLite) for testing SQL Server Dapper repositories.

## 2. Introduction

Testing data access code is historically one of the most frustrating aspects of enterprise software development. For years, the .NET community debated the merits of mocking the database versus testing against a real database. 

Entity Framework Core attempted to solve this by providing an InMemory provider. However, this often led to a false sense of security; a LINQ query that worked perfectly against EF's InMemory provider would frequently crash in production because SQL Server couldn't translate a specific function.

With Dapper, the problem is even more acute. Dapper is not an abstraction over your database—it is a direct conduit to it. Because Dapper relies entirely on raw T-SQL strings (which are dialect-specific to SQL Server, PostgreSQL, etc.), testing a SQL Server query against a SQLite in-memory database is fundamentally invalid. Furthermore, because Dapper consists of static extension methods on `IDbConnection`, it cannot be mocked using standard libraries like Moq or NSubstitute.

This chapter defines the modern enterprise standard for testing Dapper applications: **Mock the Repositories to unit-test the business logic, and use Dockerized Testcontainers to integration-test the Dapper code against a real database engine.**

## 3. Core Concepts

### The Impossibility of Mocking Dapper
`connection.QueryAsync<T>()` is a static extension method. Standard mocking frameworks (Moq, NSubstitute, FakeItEasy) can only mock interfaces and virtual methods on classes. Attempting to mock Dapper directly requires writing heavy wrapper classes around `IDbConnection` extension methods, which adds unnecessary boilerplate to your production code just to satisfy a testing framework. 

### The Solution: Mock the Abstraction
In Clean Architecture (as discussed in Chapter 6), your Application Layer depends on an interface (e.g., `ISiteRepository`). To test your business logic, you mock the `ISiteRepository`. You do *not* test Dapper here.

### Testcontainers
Testcontainers is a library that provides lightweight, throwaway instances of common databases running in Docker containers. When you run your Integration Tests, Testcontainers automatically spins up a genuine SQL Server Linux container, provides a dynamic connection string, runs your Dapper queries against it, and destroys the container when the tests finish. This provides 100% parity with production behavior.

## 4. Visual Diagrams

```text
=============================================================================
             DAPPER ENTERPRISE TESTING STRATEGY
=============================================================================

[ 1. UNIT TESTING (Fast, Isolated) ]
Target: Application Layer (MediatR Handlers, Domain Logic)

  [ GetSiteQueryHandler ] ───calls───▶ [ Mock<ISiteRepository> ]
                                             │
                                             └── Returns static fake SiteDto
(Result: Tests business logic, validation, and error handling instantly. Zero DB involvement.)


[ 2. INTEGRATION TESTING (Slower, High Fidelity) ]
Target: Infrastructure Layer (Dapper Repositories, SQL Scripts)

  [ xUnit / NUnit Test Runner ]
        │
        ├── 1. Spins up Docker Container (SQL Server 2022)
        ├── 2. Runs DB Migrations (DbUp) on Container
        │
        ├── 3. Instantiates [ SiteRepository ] with Container Connection String
        ├── 4. Executes Dapper Query
        │
        ├── 5. Asserts Results
        └── 6. Destroys Docker Container
(Result: Guarantees the T-SQL is valid, parameters bind correctly, and Dapper maps exactly as expected.)
```

## 5. Complete Examples: EV Charging Platform

### Scenario 1: Unit Testing the Application Layer
We want to test our CQRS Command Handler that creates a Charger. We will use `Moq` to mock the repository.

```csharp
// System Under Test: CreateChargerCommandHandler (Application Layer)
using Moq;
using Xunit;
using FluentAssertions;

public class CreateChargerCommandHandlerTests
{
    [Fact]
    public async Task Handle_WhenSiteDoesNotExist_ThrowsNotFoundException()
    {
        // Arrange
        var mockSiteRepo = new Mock<ISiteRepository>();
        var mockChargerRepo = new Mock<IChargerRepository>();

        // Mock the interface, not Dapper!
        mockSiteRepo
            .Setup(repo => repo.SiteExistsAsync(It.IsAny<Guid>()))
            .ReturnsAsync(false);

        var handler = new CreateChargerCommandHandler(mockSiteRepo.Object, mockChargerRepo.Object);
        var command = new CreateChargerCommand(Guid.NewGuid(), "SN-123");

        // Act
        Func<Task> act = async () => await handler.Handle(command, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<NotFoundException>();
        
        // Verify Dapper was never involved, and the Charger was never saved
        mockChargerRepo.Verify(repo => repo.InsertAsync(It.IsAny<ChargerDto>()), Times.Never);
    }
}
```

### Scenario 2: Integration Testing Dapper with Testcontainers
We need to test the actual `ChargerRepository.InsertAsync` method to ensure the SQL string is valid. We use `Testcontainers.MsSql` and `xUnit`.

**1. Setup the Database Fixture:**
```csharp
using Testcontainers.MsSql;
using Xunit;

public class SqlServerFixture : IAsyncLifetime
{
    public MsSqlContainer DbContainer { get; }

    public SqlServerFixture()
    {
        DbContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .WithPassword("StrongP@ssw0rd!")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await DbContainer.StartAsync();
        
        // Run your schema migrations here (e.g., DbUp or raw SQL scripts)
        var connectionString = DbContainer.GetConnectionString();
        await DatabaseMigrator.RunAsync(connectionString); 
    }

    public async Task DisposeAsync()
    {
        await DbContainer.StopAsync();
    }
}
```

**2. Write the Integration Test using the Transaction Rollback Pattern:**
Instead of truncating tables after every test, we start a transaction, run the test, and roll it back. This ensures blazing-fast test isolation.

```csharp
using System.Data.SqlClient;
using Xunit;
using FluentAssertions;

[Collection("DatabaseCollection")] // Shares the fixture across tests
public class ChargerRepositoryTests : IAsyncLifetime
{
    private readonly SqlServerFixture _fixture;
    private SqlConnection _connection;
    private SqlTransaction _transaction;
    private ChargerRepository _sut; // System Under Test

    public ChargerRepositoryTests(SqlServerFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        // Open connection and start transaction before EVERY test
        _connection = new SqlConnection(_fixture.DbContainer.GetConnectionString());
        await _connection.OpenAsync();
        _transaction = _connection.BeginTransaction();

        // Inject a fake ConnectionFactory that returns our transacted connection
        var factory = new FakeTransactedConnectionFactory(_connection, _transaction);
        _sut = new ChargerRepository(factory);
    }

    public async Task DisposeAsync()
    {
        // Rollback after EVERY test to reset DB state instantly
        await _transaction.RollbackAsync();
        await _connection.DisposeAsync();
    }

    [Fact]
    public async Task InsertAsync_WithValidData_SavesAndMapsCorrectly()
    {
        // Arrange
        var siteId = Guid.NewGuid();
        // Assume test data setup here...

        var charger = new ChargerDto 
        { 
            Id = Guid.NewGuid(), 
            SiteId = siteId, 
            SerialNumber = "TEST-999" 
        };

        // Act - Executes real Dapper code against Docker SQL Server
        await _sut.InsertAsync(charger, CancellationToken.None);

        // Assert - Use Dapper to verify the data was inserted
        var inserted = await _connection.QuerySingleOrDefaultAsync<ChargerDto>(
            "SELECT * FROM Chargers WHERE Id = @Id", 
            new { Id = charger.Id }, 
            transaction: _transaction);

        inserted.Should().NotBeNull();
        inserted.SerialNumber.Should().Be("TEST-999");
    }
}
```

## 6. Performance Implications

### Docker Container Startup Time
Spinning up a SQL Server Docker container takes 5 to 10 seconds. If you spin up a new container for every single test `[Fact]`, a suite of 100 tests will take 15 minutes. 

To optimize this, you must use xUnit's **Class Fixtures** or **Collection Fixtures** (as shown above). This spins up the container exactly once at the beginning of the test run, runs all 100 tests against it, and destroys it at the end. The total run time drops from 15 minutes to about 12 seconds.

### The Transaction Rollback Pattern vs Truncation
When sharing a container across tests, test data from `TestA` can corrupt `TestB`. 
*   **Truncation:** Running `DELETE FROM Tables` between tests is slow and causes identity seed issues.
*   **Respawn:** Using a library like *Respawn* to intelligently clear the database is better, but still takes a few milliseconds per test.
*   **Transaction Rollback:** Passing an open `SqlTransaction` into your repository, executing the test, and calling `Rollback()` is virtually instantaneous. SQL Server simply discards the transaction log in memory. The physical tables are never permanently mutated.

## 7. Common Mistakes

### Beginner
*   **Mistake:** Attempting to write a Mock wrapper around Dapper (e.g., `IDapperWrapper.QueryAsync<T>`).
*   **Correction:** This is an anti-pattern that creates massive maintenance overhead. You are just recreating the Dapper API. Instead, mock the business-specific interface (e.g., `IUserRepository`) that *uses* Dapper internally.
*   **Mistake:** Writing Integration Tests that connect to a shared development database (e.g., `DevDb01.corp.local`).
*   **Correction:** If two developers run the test suite simultaneously, their tests will interfere with each other, causing flaky failures. Tests must be isolated. Testcontainers solves this by giving every test run its own ephemeral database instance.

### Intermediate
*   **Mistake:** Using SQLite InMemory mode to test SQL Server Dapper repositories.
*   **Correction:** Dapper executes raw SQL. SQL Server uses `ISNULL()`, `GETUTCDATE()`, and `DATETIME2`. SQLite uses `IFNULL()`, `CURRENT_TIMESTAMP`, and strings for dates. A Dapper query that passes in SQLite is almost guaranteed to fail in SQL Server. You must test against the exact database engine you run in production.
*   **Mistake:** Hardcoding test connection strings in the test project repository.
*   **Correction:** With Testcontainers, the port mapped to the SQL Server container is dynamically assigned by Docker to prevent port conflicts on CI/CD build agents. You must use `container.GetConnectionString()` programmatically.

### Senior
*   **Mistake:** Using the Transaction Rollback pattern to test a Dapper method that internally contains explicit transaction commits (e.g., a CQRS Command Handler that calls `transaction.Commit()`).
*   **Correction:** The Rollback pattern only works if the code under test is unaware of the transaction's lifecycle (i.e., it simply receives the transaction and executes against it). If the code under test calls `Commit()`, the test cannot roll it back. For code that manages its own transactions, you must use data clearing tools like **Respawn** between test runs instead of the Rollback pattern.
*   **Mistake:** Testing Dapper mappings with perfectly formatted data, but missing null-handling edge cases.
*   **Correction:** Databases are messy. Your integration tests must explicitly insert `NULL` values into optional columns and assert that Dapper correctly maps them to C# nullable types without throwing a `DataException`.

### Architect
*   **Mistake:** Relying solely on Unit Tests because "Testcontainers are too slow for CI/CD."
*   **Correction:** Without Dapper Integration Tests, a developer renaming a database column will pass the CI build (because the Unit Test mocks return fake data), but will instantly crash production with an "Invalid column name" `SqlException`. Integration tests are mandatory in Dapper architectures. Modern CI/CD runners (GitHub Actions, Azure DevOps) support Docker natively and can spin up Testcontainers in seconds.
*   **Mistake:** Not integrating schema migrations (DbUp/FluentMigrator) into the Integration Test setup.
*   **Correction:** If your Testcontainer uses an Entity Framework script to generate the schema, but production uses DbUp, you are testing a false reality. The Testcontainer initialization phase must run the exact same migration pipeline that runs in production.

## 8. Interview Questions

### Beginner (10)
1.  **Why can't you easily mock Dapper's `QueryAsync` using Moq?**
    *Answer:* Because `QueryAsync` is a static extension method on `IDbConnection`, and mocking frameworks rely on overriding virtual methods or implementing interfaces, which is impossible for static methods.
2.  **What is a Unit Test in the context of a Data Access architecture?**
    *Answer:* A test that isolates the business logic from the database, typically by mocking the Repository interface. It executes purely in memory and is extremely fast.
3.  **What is an Integration Test?**
    *Answer:* A test that verifies the interaction between your application code (Dapper) and external infrastructure (a real SQL Server database).
4.  **What is Testcontainers?**
    *Answer:* A .NET library that allows developers to spin up throwaway Docker containers (like SQL Server, Redis, RabbitMQ) programmatically within their test code.
5.  **Why is it dangerous to use a shared development database for automated integration tests?**
    *Answer:* It causes flaky tests due to data collisions (two tests running concurrently modifying the same rows) and inconsistent state if a previous test failed to clean up after itself.
6.  **What does the Transaction Rollback pattern achieve in testing?**
    *Answer:* It ensures every test starts with a clean database state by starting a transaction, running the test, and issuing a rollback, discarding the changes instantly without deleting tables.
7.  **If you rename a C# property but forget to update the Dapper SQL string, will a Unit Test catch this?**
    *Answer:* No. The Unit Test mocks the repository. Only an Integration Test running against a real database will catch the mapping failure.
8.  **What is the `IAsyncLifetime` interface in xUnit used for with Testcontainers?**
    *Answer:* It allows you to run asynchronous code (like `container.StartAsync()`) before any tests in the class execute, and `container.StopAsync()` after they finish.
9.  **Should Integration Tests connect to your Production database?**
    *Answer:* Absolutely not. Integration tests write dummy data and often delete data. They must connect to isolated, ephemeral databases.
10. **Why should you avoid creating a `IDapperWrapper` interface just for testing?**
    *Answer:* It introduces "test damage"—modifying your production architecture (adding unnecessary abstraction layers and boilerplate) purely to satisfy a testing tool, rather than testing the actual behavior.

### Intermediate (10)
11. **Explain the difference between a Class Fixture and a Collection Fixture in xUnit when using Testcontainers.**
    *Answer:* A Class Fixture shares the Docker container across all tests within a single class. A Collection Fixture shares the container across multiple test classes, which is vastly more efficient for large test suites as it spins up SQL Server only once for the entire run.
12. **Why is testing a SQL Server Dapper application against an in-memory SQLite database an anti-pattern?**
    *Answer:* Because Dapper executes raw T-SQL. SQL Server and SQLite have entirely different dialects, functions, and data types. A query using `GETUTCDATE()` or `MERGE` will fail in SQLite, rendering the tests useless.
13. **How do you run database schema migrations (e.g., DbUp) against a Testcontainer?**
    *Answer:* Inside the `InitializeAsync` method of your test fixture, after calling `container.StartAsync()`, you retrieve the dynamic connection string and execute your DbUp migrator programmatically against it.
14. **If a test using the Transaction Rollback pattern fails, what happens to the data?**
    *Answer:* The `DisposeAsync` method still executes `transaction.RollbackAsync()`, safely discarding the data regardless of test success or failure.
15. **What is the `Respawn` library, and when would you use it over the Transaction Rollback pattern?**
    *Answer:* Respawn intelligently clears database tables by generating an optimized `DELETE` script considering foreign key relationships. You must use it when testing code that manages its own transactions (like a Unit of Work), because such code will commit transactions, rendering the Rollback pattern ineffective.
16. **How do you assert that Dapper correctly mapped a One-to-Many relationship in an Integration Test?**
    *Answer:* Execute the Dapper repository method, assert that the returned parent object is not null, assert that the child collection `Count` matches the inserted test data, and assert specific properties on the child objects.
17. **What is a "Flaky Test"?**
    *Answer:* A test that passes sometimes and fails other times without any code changes, usually due to shared state, race conditions, or unisolated database environments.
18. **How does Testcontainers handle port mapping conflicts on CI build servers?**
    *Answer:* It dynamically binds the container's SQL Server port (1433) to a random available high port on the host machine. You must use `container.GetConnectionString()` to discover this port at runtime.
19. **If you are testing a Dapper query that utilizes `CommandTimeout = 120`, how do you test the timeout behavior without waiting 2 minutes?**
    *Answer:* You can write a specific Integration Test that alters the command to have a `CommandTimeout = 1` second, and pass a SQL command that intentionally sleeps (`WAITFOR DELAY '00:00:05'`), asserting that a `SqlException` is thrown.
20. **Is it necessary to write Unit Tests for Dapper Repositories?**
    *Answer:* Generally, no. A repository's sole responsibility is executing SQL. A Unit Test with a mocked database connection tests nothing of value. Repositories should strictly be covered by Integration Tests.

### Senior (10)
21. **Analyze the performance impact of executing 500 Integration Tests using Testcontainers if you failed to use a Shared Collection Fixture.**
    *Answer:* If each test spins up a SQL Server container (taking ~7 seconds), the suite will take approximately 1 hour to run. By sharing the fixture, the container starts once (7 seconds), and the 500 tests (using transaction rollbacks) complete in less than 5 seconds, reducing CI pipeline time by 99%.
22. **You need to integration test a Dapper query that relies on SQL Server's Full-Text Search. How do you configure this with Testcontainers?**
    *Answer:* The standard SQL Server Linux image does not have Full-Text Search enabled. You must configure the Testcontainers builder to use a specific Docker image tag (e.g., `mcr.microsoft.com/mssql/server:2022-latest`) and run a custom startup script or use a custom Dockerfile that enables the FTS daemon before the tests begin.
23. **Explain how to use the `IAsyncLifetime` interface in xUnit to properly manage `SqlConnection` and `SqlTransaction` boundaries for the Rollback pattern.**
    *Answer:* In `InitializeAsync`, you instantiate the connection, call `OpenAsync`, and call `BeginTransaction`. You inject this transacted connection into the Subject Under Test. In `DisposeAsync`, you call `RollbackAsync` and `DisposeAsync` on the connection. This guarantees setup and teardown run asynchronously per test.
24. **How do you handle testing Dapper queries that rely on `SESSION_CONTEXT` for Row-Level Security (RLS)?**
    *Answer:* In the Integration Test arrange phase, you must use the test connection to execute `sp_set_session_context` with a mock Tenant ID *before* calling the Repository method. You then assert that the repository only returns data belonging to that specific mock Tenant ID.
25. **Your Dapper repository uses a `TypeHandler` to map JSON strings to complex objects. How do you ensure this is tested?**
    *Answer:* The `TypeHandler` must be registered globally. In your Test Project, you must have an assembly initialization step (or within the Fixture startup) that calls `SqlMapper.AddTypeHandler()` exactly as the production `Program.cs` does. If you forget, the integration tests will fail with parsing exceptions.
26. **What is Mutation Testing, and how does it prove the value of Dapper Integration Tests?**
    *Answer:* Mutation testing involves tools intentionally changing your production code (e.g., changing a `>` to `<` in a SQL string) and ensuring your tests fail. If you only mocked the repository, the mutation in the SQL string would not cause a test failure. Integration tests catch these mutations, proving the test suite's robustness.
27. **How do you integration test a Dapper Stored Procedure that takes a Table-Valued Parameter (TVP)?**
    *Answer:* The Testcontainer must have the User-Defined Table Type and the Stored Procedure created via DbUp. The test code creates a `DataTable`, populates it with test data, passes it via `AsTableValuedParameter`, executes the repository method, and then queries the target table to assert the bulk data was inserted correctly.
28. **In an Event-Driven architecture, your Dapper repository implements the Transactional Outbox pattern. How do you test it?**
    *Answer:* You write an Integration Test that calls the repository method. The assertion phase must verify two things: 1. The primary domain entity was saved correctly. 2. The `OutboxMessages` table contains exactly one new row with the serialized event payload.
29. **Why might `ExecuteScalarAsync<int>` return unexpected results in a Testcontainer environment if you use `IDENTITY` columns and truncation?**
    *Answer:* `DELETE FROM Table` does not reset the `IDENTITY` seed. If Test A inserts a row, its ID is 1. If Test A deletes it, and Test B inserts a row, its ID is 2. If Test B asserts `Id == 1`, it fails. This is why the Transaction Rollback pattern or tools like Respawn (which can reseed identities) are critical for predictable assertions.
30. **How can you assert that a specific Dapper query did NOT cause the "N+1 query" problem during an Integration Test?**
    *Answer:* You can use SQL Server Extended Events (XEvents) or a custom `DbConnection` interceptor (like MiniProfiler) configured in the test fixture to count the number of SQL statements executed during the repository call. Assert that the `ExecutionCount` is exactly 1 (or the expected batched amount).

### Staff Engineer (5)
31. **Architect an automated CI/CD pipeline step in GitHub Actions that runs Dapper integration tests. How do you handle the Docker daemon requirement?**
    *Answer:* GitHub Actions Ubuntu runners come with Docker pre-installed. The .NET test command (`dotnet test`) will seamlessly communicate with the local Docker daemon. The Testcontainers library will automatically pull the SQL Server image, start it, run the tests, and clean it up. The only architectural requirement is ensuring the runner has sufficient RAM (at least 4GB) to run SQL Server.
32. **Your application uses Azure SQL Active Directory (Entra ID) Managed Identity authentication, which requires a specific token provider. How do you integration test this locally with Testcontainers?**
    *Answer:* Testcontainers runs standard SQL Server Linux, which uses SQL Authentication (Username/Password), not Entra ID. As a Staff Engineer, I must design the `ISqlConnectionFactory` to be environment-aware. In Production, it acquires the Entra token. In the Integration Test environment (driven by env vars), it falls back to standard SQL Authentication using the Testcontainer's connection string, while keeping the core Dapper code identical.
33. **A complex reporting query using Dapper `QueryMultiple` works in production but consistently deadlocks in the Integration Test suite. What architectural difference causes this?**
    *Answer:* Integration tests using the Transaction Rollback pattern wrap *everything* in a `SqlTransaction` with an isolation level (usually ReadCommitted). If the test arranges data by inserting rows, and then executes the multi-query, it might cause read/write lock contention that doesn't exist in production (where reads might use Snapshot isolation or run outside explicit transactions). The test architecture must mimic production isolation levels precisely.
34. **Design a strategy to integration test database migrations that include destructive schema changes alongside Dapper queries to ensure zero-downtime deployments.**
    *Answer:* I architect a multi-stage integration test. Stage 1: Spin up container, apply V1 schema. Insert V1 test data via Dapper. Stage 2: Apply V2 schema migration (e.g., adding a new column, copying data). Stage 3: Run the *V2* Dapper repository code against the migrated data and assert correctness. This guarantees the Dapper code is backwards/forwards compatible with the migration script during a rolling deployment.
35. **Evaluate the memory implications on the CI build agent when running parallel xUnit test collections that each spin up their own SQL Server Testcontainer.**
    *Answer:* SQL Server Linux containers consume ~1.5GB to 2GB of RAM minimum. If xUnit runs 4 test collections in parallel, it spins up 4 containers, demanding 8GB of RAM. Most standard CI agents will crash with an OOM (Out of Memory) error. You must architect the test suite to either disable xUnit collection parallelism (`[assembly: CollectionBehavior(DisableTestParallelization = true)]`) or utilize a single Shared Assembly Fixture so only one container is ever created.

### Architect (5)
36. **Architect a mechanism to completely separate the Domain Logic Unit Tests from the Database Integration Tests so they can run in different stages of a deployment pipeline.**
    *Answer:* I structure the solution with distinct projects: `Platform.Domain.Tests` and `Platform.Infrastructure.IntegrationTests`. I use xUnit Traits (`[Trait("Category", "Unit")]`). In the CI/CD pipeline, the "Build" stage runs `dotnet test --filter Category=Unit`, providing developers feedback in < 5 seconds. Only if successful, the pipeline moves to the "Integration" stage, running the heavier Testcontainers suite, optimizing the feedback loop and CI compute costs.
37. **Defend the architectural decision to mandate 100% Integration Test coverage for Dapper repositories, explicitly prohibiting Unit Testing of `IDbConnection`.**
    *Answer:* Unit testing an ORM wrapper tests the mock, not the reality. Dapper's sole purpose is executing raw SQL and mapping results. A Unit Test cannot verify SQL syntax, constraint violations, parameter binding, or IL Emit mapping behaviors. Therefore, a Unit Test provides zero confidence that the code will work in production. Only an Integration Test against a real schema provides the deterministic proof required for enterprise reliability. Any time spent mocking `IDbConnection` is wasted engineering effort.
38. **How do you architect observability into the Integration Test suite so that when a Dapper query fails on the CI server, developers can see the exact SQL Server Execution Plan?**
    *Answer:* I enhance the Testcontainers startup script to enable Query Store on the ephemeral database. If a test fails, the teardown method in the `IAsyncLifetime` fixture catches the exception, executes a query against `sys.query_store_plan` to extract the XML execution plan for the failed query, and writes it to the CI artifact output directory before the container is destroyed.
39. **Your SaaS platform targets both SQL Server and PostgreSQL using Dapper. Architect an Integration Test suite that guarantees both implementations work.**
    *Answer:* I use the Abstract Factory pattern for the tests. I create an abstract base test class `RepositoryTestsBase`. I then create two inherited fixtures: one uses `Testcontainers.MsSql` and the other uses `Testcontainers.PostgreSql`. The test runner automatically executes the entire suite of assertions twice—once against the SQL Server container (exercising the T-SQL Dapper queries) and once against the Postgres container (exercising the PL/pgSQL Dapper queries), guaranteeing cross-DB compatibility.
40. **How do you handle testing temporal (time-based) Dapper queries (e.g., `SELECT * FROM Sessions WHERE EndTime < GETUTCDATE()`) deterministically in an Integration Test?**
    *Answer:* Relying on `GETUTCDATE()` inside Dapper SQL strings makes tests non-deterministic. As an Architect, I enforce that time is a dependency that must be injected. The Application Layer uses an `ITimeProvider` and passes the absolute `DateTime` as a parameter to Dapper: `WHERE EndTime < @CurrentTime`. In the Integration Test, we pass a fixed, known `DateTime` parameter, making the database assertion 100% deterministic regardless of when the CI server runs the test.

## 10. Exercises

### Easy
1.  **Mocking the Interface:** Create an `IUserRepository` with a `GetUserAsync` method. Create a `UserService` (Application Layer) that calls it. Write a Unit Test using Moq to return a fake User and assert the service logic behaves correctly.

### Medium
1.  **Testcontainers Setup:** Create a new xUnit test project. Install the `Testcontainers.MsSql` NuGet package. Write a single test that spins up a container, opens a `SqlConnection` to it, executes `connection.ExecuteScalarAsync<int>("SELECT 1")` via Dapper, and asserts the result is 1.

### Hard
1.  **Transaction Rollback Pattern:** Implement the `SqlServerFixture` and the Transaction Rollback pattern as demonstrated in this chapter. Write a test that inserts a record via Dapper. Write a second test that asserts the table is completely empty (proving the first test rolled back successfully).

### Enterprise
1.  **Schema Migration & Integration:** 
    *   Set up a DbUp console application that creates a `Chargers` table.
    *   In your xUnit test project, configure the `SqlServerFixture` to run the DbUp migration automatically upon container startup.
    *   Write a Dapper repository `Insert` method.
    *   Write an integration test that successfully inserts a charger and retrieves it, proving your Dapper code perfectly matches your production schema migrations.

## 11. Summary

Testing Dapper requires a paradigm shift from traditional Full ORM testing. Because Dapper is intrinsically tied to raw SQL, attempting to mock it or test it against in-memory substitutes like SQLite is an exercise in futility. 

By strictly adhering to Clean Architecture, you can isolate your business logic for lightning-fast Unit Testing. For the Data Access layer, **Testcontainers** combined with the **Transaction Rollback Pattern** provides the holy grail of testing: 100% production fidelity, absolute test isolation, and execution speeds that seamlessly integrate into modern CI/CD pipelines.

In the final chapter of this book, we will focus on Deployment and Observability, exploring how to run these high-performance Dapper applications in Azure, manage Managed Identities, and monitor query execution using OpenTelemetry.
