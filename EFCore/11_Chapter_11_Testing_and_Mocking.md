# Chapter 11: Testing and Mocking Strategies

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Explain the architectural fallacy of mocking `DbContext` or `DbSet` using frameworks like Moq.
*   Evaluate the limitations of the EF Core In-Memory provider and why it fails to catch critical production bugs.
*   Implement integration tests utilizing SQLite in-memory mode as a lightweight relational alternative.
*   Master **Testcontainers**, deploying real SQL Server Docker containers for absolute parity between integration tests and production environments.
*   Architect automated testing pipelines that validate EF Core configurations, Global Query Filters, and Execution Strategies flawlessly.

## 2. Introduction: The Mocking Fallacy

For over a decade, .NET developers were taught to achieve 100% Code Coverage by writing Unit Tests for everything. This dogma led to one of the most destructive anti-patterns in data access: **Mocking the DbContext**.

Developers would use libraries like Moq to create a fake `DbSet`, feed it a `List<T>`, and write a unit test to verify that `context.Users.Add()` was called. 

This test is entirely worthless. It proves only that the developer typed `Add()`. It does not prove that the SQL Server connection works. It does not prove that the Foreign Key constraint is satisfied. It does not prove that the Global Query Filter is applied. It does not prove that the `.ToJson()` mapping translates correctly.

When you mock the DbContext, you are mocking the very thing that is most likely to fail in production.

This chapter redefines how to test data access. We will abandon the fragile practice of mocking the ORM. Instead, we will write robust Integration Tests using real, isolated database instances, ensuring that if our tests pass, our production deployment will succeed.

## 3. Core Concepts

### Test Doubles for EF Core
*   **Moq/NSubstitute:** Used for mocking interfaces (like an `ITenantProvider`). **Never** used for mocking `DbContext`.
*   **EF Core In-Memory Provider:** A non-relational data store provided by Microsoft (`UseInMemoryDatabase`). It does not understand Foreign Keys, Unique Constraints, or raw SQL.
*   **SQLite In-Memory:** A real relational database that runs entirely in RAM. Fast, but speaks a different SQL dialect than SQL Server.
*   **Testcontainers (Docker):** A library that dynamically spins up a real SQL Server Docker container for your test suite, applies migrations, runs the tests, and destroys the container. (The Enterprise Gold Standard).

### Integration Testing
Unit tests isolate a single class. Integration tests verify that multiple components (the Controller, the Command Handler, EF Core, and the Database) work together seamlessly. For data access, Integration Testing is the only testing that matters.

### `WebApplicationFactory<T>`
An ASP.NET Core testing utility that boots up your entire `Program.cs` in memory, allowing you to execute HTTP requests against your API and intercept the DI container to swap out the production connection string for a test database connection string.

## 4. Visual Diagrams

```text
=============================================================================
             THE EVOLUTION OF EF CORE TESTING
=============================================================================

1. THE DARK AGES (Mocking)
[ Test ] ──▶ [ Mock<DbSet> (List<T>) ]
* Speed: Instant
* Reliability: 0% (Fails to catch SQL syntax errors, constraint violations, translations).

2. THE MIDDLE AGES (EF In-Memory)
[ Test ] ──▶ [ UseInMemoryDatabase() ]
* Speed: Fast
* Reliability: 30% (Catches basic LINQ issues. Fails on Foreign Keys, Transactions, Raw SQL).

3. THE RENAISSANCE (SQLite In-Memory)
[ Test ] ──▶ [ UseSqlite("DataSource=:memory:") ]
* Speed: Fast
* Reliability: 70% (Catches relational issues. Fails on SQL Server specific features like JSON_VALUE).

4. THE ENTERPRISE STANDARD (Testcontainers)
[ Test ] ──▶ [ Docker: mcr.microsoft.com/mssql/server:2022 ]
* Speed: Moderate (Container boot takes 3 seconds)
* Reliability: 100% (Absolute parity with Production Azure SQL).
```

## 5. API Deep Dive: Testing Providers

### 5.1 The EF Core In-Memory Provider (Anti-Pattern)
Microsoft explicitly discourages using this provider for relational testing in their official documentation. It is not a relational database.

```csharp
// DO NOT DO THIS FOR RELATIONAL DOMAINS
var options = new DbContextOptionsBuilder<EvDbContext>()
    .UseInMemoryDatabase(databaseName: "TestDb")
    .Options;

using var context = new EvDbContext(options);
// If you add a Charger with a SiteId that doesn't exist, this SUCCEEDS. 
// It ignores Foreign Key constraints completely!
```

### 5.2 SQLite In-Memory
A massive step up. It is a true relational database.

```csharp
// Requires Microsoft.EntityFrameworkCore.Sqlite
var connection = new SqliteConnection("DataSource=:memory:");
connection.Open(); // Must keep connection open for the lifetime of the test!

var options = new DbContextOptionsBuilder<EvDbContext>()
    .UseSqlite(connection)
    .Options;

using var context = new EvDbContext(options);
context.Database.EnsureCreated(); // Creates the schema in RAM

// This will now correctly throw an exception if a Foreign Key is violated.
```
*Limitation:* If your code uses SQL Server features (`EF.Functions.FreeText()`, `.ToJson()`, or ExecuteUpdate bulk operations), SQLite will throw a translation exception because its SQL dialect is different.

### 5.3 Testcontainers (The Gold Standard)
Testcontainers is a NuGet package (`Testcontainers.MsSql`) that leverages Docker Desktop (or CI/CD Docker services) to spin up a temporary database.

```csharp
// In your test Setup/Initialize method:
var msSqlContainer = new MsSqlBuilder()
    .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
    .Build();

await msSqlContainer.StartAsync();

// Get the dynamic, randomized connection string (e.g., localhost:49153)
string connectionString = msSqlContainer.GetConnectionString();

// Configure EF Core to use the REAL SQL Server running in Docker
var options = new DbContextOptionsBuilder<EvDbContext>()
    .UseSqlServer(connectionString)
    .Options;

using var context = new EvDbContext(options);
await context.Database.MigrateAsync(); // Apply real migrations!
```

## 6. EF Core Internals: Provider Translation Limits

Why does swapping the database provider in tests cause false positives/negatives?

EF Core's LINQ translation is highly coupled to the specific Database Provider. 
If you write `context.Users.Where(u => u.Name.Contains("A", StringComparison.OrdinalIgnoreCase))`, the SQL Server provider translates this flawlessly based on the database collation.

If you run that exact same C# code against the EF Core In-Memory provider, it evaluates the `Contains` method locally in C#.

If you run it against SQLite, it might throw an `InvalidOperationException` because SQLite doesn't natively support case-insensitive `Contains` without custom function mapping.

**Architectural Rule:** If you want your tests to prove your production application works, you must test against the exact same database engine used in production.

## 7. Complete Examples: EV Platform Case Study

We need to write an Integration Test to verify our "Book a Charging Slot" CQRS Command. This command utilizes EF Core's Optimistic Concurrency (RowVersion) to prevent double-booking. 

Mocking `DbContext` cannot test RowVersion concurrency. In-Memory database does not support RowVersion concurrency natively. We **must** use Testcontainers.

### Setup using xUnit and `IAsyncLifetime`
```csharp
public class BookingIntegrationTests : IAsyncLifetime
{
    private MsSqlContainer _dbContainer;
    private EvDbContext _context;

    public async Task InitializeAsync()
    {
        // 1. Boot up SQL Server
        _dbContainer = new MsSqlBuilder().Build();
        await _dbContainer.StartAsync();

        // 2. Configure DbContext
        var options = new DbContextOptionsBuilder<EvDbContext>()
            .UseSqlServer(_dbContainer.GetConnectionString())
            .Options;
            
        // 3. Mock the TenantProvider (It's OK to mock interfaces, NOT the DbContext)
        var mockTenantProvider = new Mock<ITenantProvider>();
        mockTenantProvider.Setup(x => x.GetTenantId()).Returns(Guid.NewGuid());

        _context = new EvDbContext(options, mockTenantProvider.Object);
        
        // 4. Apply Schema
        await _context.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _context.DisposeAsync();
        await _dbContainer.DisposeAsync(); // Destroys the database!
    }

    [Fact]
    public async Task BookSlot_ConcurrentRequests_ThrowsConcurrencyException()
    {
        // Arrange: Seed a Slot
        var slot = new ChargingSlot { Id = 1, IsBooked = false };
        _context.ChargingSlots.Add(slot);
        await _context.SaveChangesAsync();

        // Simulate User A reading the slot
        var userAContext = new EvDbContext(..., ...);
        var slotA = await userAContext.ChargingSlots.FindAsync(1);

        // Simulate User B reading the slot concurrently
        var userBContext = new EvDbContext(..., ...);
        var slotB = await userBContext.ChargingSlots.FindAsync(1);

        // Act: User A books it and saves (Success)
        slotA.IsBooked = true;
        await userAContext.SaveChangesAsync();

        // Act: User B tries to book it
        slotB.IsBooked = true;
        
        // Assert: EF Core MUST throw the concurrency exception because the RowVersion changed
        await Assert.ThrowsAsync<DbUpdateConcurrencyException>(() => 
            userBContext.SaveChangesAsync());
    }
}
```

## 8. Performance: Parallel Test Execution

Spinning up a SQL Server Docker container takes roughly 3 to 5 seconds. If you have 500 integration tests, and you spin up a new container for *every single test*, your test suite will take 40 minutes to run.

**The Solution: Container Re-use and Respawning**
Instead of creating a container per test, you create one container per *Test Class* (using xUnit `IClassFixture`) or globally for the entire test run (using `ICollectionFixture`).

But if all tests share one database, Test A might insert a `Charger`, and Test B might query `Count()`, expecting 0 but getting 1 because Test A's data is still there. Tests must be isolated!

**The Respawn Library:**
Respawn is a NuGet package created by Jimmy Bogard. Between every single test, Respawn executes a highly optimized SQL command that instantly truncates all tables and resets all IDENTITY sequences, wiping the database clean in a few milliseconds without destroying the container.

```csharp
// Inside the test constructor or test Initialize method:
await _checkpoint.Reset(_dbContainer.GetConnectionString());
```
*Result:* You boot Docker once (3 seconds). You run 500 tests. Respawn wipes the data between each test (5ms overhead). The entire test suite completes in 10 seconds with 100% production parity.

## 9. ASP.NET Core Integration: WebApplicationFactory

Testing the DbContext directly is great, but testing the entire HTTP API pipeline (Middleware, Auth, CQRS, and EF Core) is better.

```csharp
public class CustomApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private MsSqlContainer _dbContainer;

    public async Task InitializeAsync()
    {
        _dbContainer = new MsSqlBuilder().Build();
        await _dbContainer.StartAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Remove the production DbContext configuration
            var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<EvDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            // Add the DbContext using the Docker connection string
            services.AddDbContext<EvDbContext>(options =>
                options.UseSqlServer(_dbContainer.GetConnectionString()));
        });
    }

    public Task DisposeAsync() => _dbContainer.DisposeAsync();
}
```
*Usage:* You use `factory.CreateClient()` to get an `HttpClient`. You execute `await client.PostAsJsonAsync("/api/chargers", dto)`. The request flows through the entire ASP.NET Core pipeline and saves into the Docker database. You then query the Docker database to assert the data was saved correctly.

## 10. Clean Architecture Perspective

In Clean Architecture, the **Domain Layer** contains the Entities and business logic (`public void Deactivate() { ... }`). The Domain layer does not reference EF Core. Therefore, you should use standard, instantaneous **Unit Tests** for the Domain Layer. No database required.

The **Infrastructure Layer** (where EF Core configurations and migrations live) and the **Application Layer** (where MediatR Command Handlers orchestrate DbContext saves) MUST be tested using **Integration Tests** with Testcontainers.

Do not write Unit Tests for Repositories using Moq. It is a waste of engineering time.

## 11. Enterprise SaaS Perspective: Testing Migrations

In a SaaS environment, deploying a bad migration causes a system-wide outage.
Your CI/CD pipeline must include a test that strictly verifies your migrations.

**The Migration Test:**
Create an Integration Test that boots an empty SQL Server Testcontainer.
Execute `await context.Database.MigrateAsync()`.
If the developer made a mistake in the C# configuration (e.g., trying to map an index on a column that doesn't exist, or writing invalid raw SQL in the `Up()` method), `MigrateAsync()` will throw a `SqlException` and fail the CI/CD build before it ever reaches production.

## 12. Real Production Case Study

In the EV Platform, we had a bug where the API successfully returned `200 OK` when deleting a `Tenant`, but the related `Sites` were not deleted in the database, resulting in orphaned records.

**Why the Unit Test missed it:**
The developer wrote a unit test using Moq:
`mockContext.Verify(m => m.Tenants.Remove(tenant), Times.Once);`
The test passed because the code did call `Remove()`.

**Why the In-Memory Test missed it:**
Another developer wrote a test using `UseInMemoryDatabase()`.
The In-Memory database does not enforce Foreign Key constraints. It happily deleted the Tenant and ignored the child Sites.

**The Fix (Testcontainers):**
We refactored the test suite to use Testcontainers (SQL Server). The test instantly failed. Why? Because the SQL Server Database threw an FK Constraint violation! We realized we had configured the relationship with `DeleteBehavior.Restrict` instead of implementing a domain-driven soft-delete cascade. 
We fixed the application logic to correctly coordinate the deletion of child aggregates. The Testcontainers test now passes, and we have absolute confidence the bug is fixed in production.

## 13. Common Mistakes

### Beginner
*   **Mistake:** Mocking `DbSet` and trying to implement `.Where` or `.Include` via Moq setups.
*   **Correction:** It is nearly impossible to accurately mock EF Core's `IQueryable` extension methods. Stop trying. Use SQLite in-memory or Testcontainers.

### Intermediate
*   **Mistake:** Using `UseInMemoryDatabase` to test a query that utilizes `EF.Functions.DateDiffDay()`.
*   **Correction:** The In-Memory provider cannot translate SQL Server specific functions. It will throw an exception. You must use Testcontainers.

### Senior
*   **Mistake:** Running Integration Tests sequentially because sharing a database causes state conflicts.
*   **Correction:** Sequential tests kill developer productivity. Implement the `Respawn` library to truncate tables between tests, allowing you to run hundreds of integration tests in seconds against a single Docker container.

### Architect
*   **Mistake:** Allowing developers to merge PRs without an automated test that applies EF Core Migrations against an empty database.
*   **Correction:** The Architect must enforce a CI gate. The CI server must spin up a Docker database, run `dotnet ef database update`, and only proceed to compile the application if the migrations succeed. This catches manual typos in migration files immediately.

## 14. Interview Questions

### Beginner (10)
1.  **Why is mocking a `DbContext` considered an anti-pattern?**
    *Answer:* Because the DbContext is an abstraction over a physical database. Mocking it hides all SQL translation errors, constraint violations, and concurrency issues, providing zero confidence that the code will work in production.
2.  **What is the EF Core In-Memory provider designed for?**
    *Answer:* It is a general-purpose, non-relational datastore primarily useful for testing very basic CRUD operations without foreign key complexities, though its use is largely discouraged for enterprise systems.
3.  **Why is SQLite In-Memory better than the EF Core In-Memory provider?**
    *Answer:* SQLite is a true relational database. It enforces Foreign Key constraints and unique indexes, making tests much more realistic.
4.  **What is a limitation of using SQLite for testing a SQL Server application?**
    *Answer:* SQLite cannot execute SQL Server specific functions (like `JSON_VALUE`, Full-Text search, or specific `EF.Functions`).
5.  **What is Testcontainers?**
    *Answer:* A library that orchestrates Docker containers from C# code, allowing you to dynamically spin up a real SQL Server database for integration tests.
6.  **What does `WebApplicationFactory` do?**
    *Answer:* It boots up an ASP.NET Core application in memory, allowing you to write integration tests that send HTTP requests to the API and test the entire stack end-to-end.
7.  **How do you override the database connection string when using `WebApplicationFactory`?**
    *Answer:* By overriding `ConfigureWebHost` and using `services.ConfigureTestServices()` to remove the production `DbContext` and inject a new one pointing to the test database.
8.  **Should Domain Entities be tested with Unit Tests or Integration Tests?**
    *Answer:* Unit tests. Domain entities (Aggregates) should be pure C# classes without external dependencies, making them perfect for fast, isolated unit tests.
9.  **Should Repositories be tested with Unit Tests or Integration Tests?**
    *Answer:* Integration Tests. A repository's entire purpose is database communication, which cannot be validated via a mock.
10. **What happens if you don't clear the database between integration tests?**
    *Answer:* Test State Leakage. Test A inserts data that Test B accidentally queries, causing random, flaky test failures depending on execution order.

### Intermediate (10)
11. **Explain how the `Respawn` library solves test state leakage.**
    *Answer:* `Respawn` intelligently analyzes the database schema and generates a highly optimized script to truncate all tables and reset identity sequences. You execute this script between tests to guarantee a pristine database state in milliseconds.
12. **You write a test for `AsSplitQuery()`. Can you use the In-Memory provider?**
    *Answer:* No. The In-Memory provider ignores relational concepts like `JOINs` and Split Queries.
13. **How do you test Optimistic Concurrency (RowVersion) in EF Core?**
    *Answer:* You must use an actual relational database (like Testcontainers). You read the entity in two separate DbContext instances, update both, save the first, and assert that saving the second throws a `DbUpdateConcurrencyException`.
14. **How do you ensure a Testcontainer is destroyed even if the test fails and throws an exception?**
    *Answer:* By implementing `IAsyncLifetime` (in xUnit) or equivalent teardown methods, calling `await _container.DisposeAsync()`, which sends a command to Docker to kill and remove the container.
15. **Can you use Testcontainers in a CI/CD pipeline like GitHub Actions?**
    *Answer:* Yes, GitHub Actions runners have Docker installed by default. Testcontainers will automatically detect the Docker daemon and spin up the SQL Server container during the build pipeline.
16. **Why does `context.Database.EnsureCreated()` break migration testing?**
    *Answer:* `EnsureCreated()` bypasses the migrations pipeline and builds the schema directly. If you want to test that your migration files actually work, you must use `context.Database.MigrateAsync()` in your test setup.
17. **How do you test a Global Query Filter (e.g., TenantId)?**
    *Answer:* Create an Integration test. Mock the `ITenantProvider` to return Tenant A. Insert data for Tenant A and Tenant B into the Testcontainer. Execute a query and assert that only Tenant A's data is returned.
18. **What is the `HasConversion` risk when testing with SQLite?**
    *Answer:* If you use `HasConversion` to map a complex object to a JSON string, SQLite stores it as a string. SQL Server stores it as `NVARCHAR`. If your LINQ query attempts to query *inside* that string, SQLite might evaluate it locally, whereas SQL Server might throw an error.
19. **How do you verify that an EF Core query generates the exact T-SQL you expect?**
    *Answer:* You can call `.ToQueryString()` on the `IQueryable` in your test and `Assert.Contains("EXPECTED SQL", result)`.
20. **Is it possible to mock an `ExecutionStrategy` to simulate a transient fault in a test?**
    *Answer:* Yes, but it is complex. You can inject a custom `IExecutionStrategy` via DbContext Options specifically for the test, forcing it to execute the retry delegate multiple times to verify application behavior during network drops.

### Senior (10)
21. **Architect a test suite strategy for an Enterprise SaaS utilizing Database-per-Tenant. How do you test the dynamic connection string resolution?**
    *Answer:* Boot a SQL Server Testcontainer. Manually create two distinct logical databases inside the container (`CREATE DATABASE Tenant1; CREATE DATABASE Tenant2;`). In the `WebApplicationFactory`, mock the `ITenantProvider` to return Tenant 1's ID. Execute an HTTP POST to create data. Assert the data exists in `Tenant1`. Change the mock to Tenant 2, execute a GET, and assert it returns empty. This proves the DbContext routing logic works perfectly.
22. **Evaluate the performance trade-offs of using `IClassFixture` versus `ICollectionFixture` with Testcontainers in xUnit.**
    *Answer:* `IClassFixture` boots a new Docker container for *every test class*. If you have 50 classes, that's 50 container boots (150 seconds overhead). `ICollectionFixture` boots one single container for the *entire test suite*. However, `ICollectionFixture` forces all tests in that collection to run sequentially on a single thread to avoid database lock contention. The Architect must balance container boot time vs. parallel execution capabilities, often opting for `ICollectionFixture` coupled with `Respawn` for optimal speed.
23. **You are using EF Core `.ToJson()` mapping. You write an integration test using SQLite. The test fails with a SQL syntax error. Why?**
    *Answer:* EF Core 7/8 translates `.ToJson()` LINQ queries into SQL Server specific `JSON_VALUE` functions. The SQLite provider translates them into SQLite `json_extract` functions. If your DbContext is specifically hardcoded to `UseSqlServer` but you override it with `UseSqlite` in testing, the internal relational mappings conflict. To use native JSON mapping safely, you MUST use SQL Server Testcontainers.
24. **Design an automated CI gate that prevents N+1 queries from reaching production.**
    *Answer:* In the Integration Test `DbContext` configuration, enable `options.ConfigureWarnings(w => w.Throw(RelationalEventId.MultipleCollectionIncludeWarning))`. Furthermore, write a custom Interceptor (`DbCommandInterceptor`) specifically for tests that counts the number of SQL queries executed per HTTP Request in the `WebApplicationFactory`. If an endpoint triggers more than 3 queries, throw an exception and fail the test.
25. **How do you write an integration test to verify that `ExecuteUpdate` successfully bypasses domain logic validations enforced by the Change Tracker?**
    *Answer:* Write an entity with business logic (e.g., throwing if `Price < 0`). Insert a valid entity. In the test, use `context.Items.ExecuteUpdateAsync(s => s.SetProperty(i => i.Price, -10))`. Assert that the command succeeds and the database contains the invalid data (-10). This test explicitly documents the architectural trade-off of bypassing the domain layer for performance.

## 15. Exercises

### Easy
1.  **Stop Mocking:** If you have an existing project that mocks `DbSet` using Moq, delete those tests. They are technical debt.

### Medium
1.  **SQLite In-Memory:** Create a simple test class. Initialize a `SqliteConnection` to `:memory:`. Configure the DbContext to use it, call `EnsureCreated()`, insert a record, query it, and assert the values.
2.  **ToQueryString():** Write a complex LINQ query (e.g., filtering, sorting, projection). In a test, call `.ToQueryString()` and use `Console.WriteLine` or `Assert` to output the exact T-SQL EF Core generates.

### Hard
1.  **Testcontainers Setup:** Install the `Testcontainers.MsSql` NuGet package. Implement `IAsyncLifetime` in an xUnit test class. Boot up a SQL Server container, get the dynamic connection string, configure EF Core, call `MigrateAsync()`, and write a test that inserts and reads data from the real Docker database.

### Enterprise
1.  **Respawn Integration:** Take the Testcontainers setup from the previous exercise. Add 5 distinct tests. Install the `Respawn` NuGet package. Configure `Respawn` to execute in an async method before *every* test execution, wiping the data cleanly. Verify that Test 5 passes regardless of what data Test 1 inserted.

## 16. Production Checklist

- [ ] Has mocking the `DbContext` or `DbSet` via libraries like Moq been strictly banned from the codebase?
- [ ] Are all data access paths verified using Integration Tests against a real relational database?
- [ ] Is Testcontainers (Docker) utilized to ensure absolute parity between the testing environment and production SQL Server?
- [ ] Is test state isolation guaranteed between tests using a database wipe tool like `Respawn`?
- [ ] Does the CI/CD pipeline automatically test the execution of EF Core Migration files (`MigrateAsync`) on an empty database before compiling release binaries?

## 17. Summary

Testing data access should provide absolute confidence that your code will execute correctly in production. Mocking EF Core destroys that confidence, hiding SQL translation errors and concurrency conflicts behind fake C# lists.

By embracing Testcontainers and the `WebApplicationFactory`, we elevate our testing strategy to the Enterprise tier. We test the actual SQL executed against an actual database, guaranteeing that our migrations, our query filters, and our high-performance `ExecuteUpdate` commands function flawlessly.

We have reached the end of the technical journey. In the final chapter, we will synthesize everything you have learned into the **Master Architect's Playbook**—a quick-reference guide containing the golden rules, performance checklists, and architectural mandates required to lead a team in building world-class EF Core applications.
