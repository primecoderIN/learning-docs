# Chapter 26: Testing Databases

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand why Mocking `DbSet` and using the EF Core InMemory Database are architectural anti-patterns that provide false confidence.
*   Architect true Integration Tests against a real SQL Server instance.
*   Implement **Testcontainers** to spin up ephemeral SQL Server Docker containers directly from C# xUnit/NUnit tests.
*   Utilize libraries like *Respawn* to reset database state between tests for maximum isolation and performance.

---

## 26.1 The Database Testing Dilemma

Unit Testing is about testing pure business logic (e.g., `CalculateInvoiceAmount()`) in isolation, executing in milliseconds.
However, when you need to test code that interacts with the database (e.g., `UpdateStationStatus()`), you cross the boundary from Unit Testing into **Integration Testing**.

Historically, developers have tried to bend Integration Tests to act like Unit Tests by faking the database. This leads to three distinct anti-patterns.

---

## 26.2 Anti-Pattern 1: Mocking `DbSet`

Early on, developers used libraries like Moq to mock `DbContext` and `DbSet`.

```csharp
// DANGEROUS ANTI-PATTERN
var mockDbSet = new Mock<DbSet<Station>>();
var mockContext = new Mock<VoltCoreDbContext>();
mockContext.Setup(c => c.Stations).Returns(mockDbSet.Object);
```
**Why it fails:** EF Core translates LINQ to SQL using a highly complex expression tree parser. If you mock the `DbSet`, your LINQ query executes against `IEnumerable` in C# RAM, not `IQueryable` in SQL. 
If you write `.Where(s => SomeCSharpMethod(s.Name))`, the Mock will pass perfectly. But in Production, EF Core will throw a `SqlTranslationException` because SQL Server doesn't know what `SomeCSharpMethod` is. Mocking the database proves absolutely nothing about production readiness.

---

## 26.3 Anti-Pattern 2: The InMemory Provider

Microsoft provides an InMemory database provider (`options.UseInMemoryDatabase("TestDb")`). It is widely used. **And it is an architectural trap.**

The InMemory provider behaves like a NoSQL document store. It is *not* a relational database.
1.  **No Constraints:** It ignores Foreign Keys, Unique Constraints, and `VARCHAR` length limits. You can insert a string 500 characters long into a `VARCHAR(50)` column, and the test will pass. Production will crash.
2.  **No Transactions:** It does not support `BeginTransaction()`. If your code uses transactions (Chapter 16), your tests will throw exceptions.
3.  **No Raw SQL:** You cannot use `ExecuteUpdateAsync` or `FromSqlInterpolated` (Chapter 24).

*Architect Rule:* Never use the InMemory provider for enterprise SaaS testing. It provides a false sense of security.

---

## 26.4 The Architect's Standard: True Integration Testing

To test database code, you must test against the exact same database engine you run in Production. If Production uses SQL Server 2022, your tests must run against SQL Server 2022.

In the past, this meant maintaining a shared "Dev Database." But if Alice runs her tests and deletes a Tenant, and Bob runs his tests and expects that Tenant to exist, the tests randomly fail due to race conditions.

The modern solution is **Ephemeral Containerization**.

---

## 26.5 Introduction to Testcontainers

**Testcontainers** is a library that allows your C# test project to programmatically spin up Docker containers.

Before your test suite runs, Testcontainers downloads the official Microsoft SQL Server Docker image, spins up an isolated container, and provides your C# code with a dynamic Connection String. 
When the test suite finishes, Testcontainers automatically destroys the container.

**Pros:**
*   You are testing against a real SQL Server engine. Constraints, Transactions, and Raw SQL work perfectly.
*   Zero infrastructure management. CI/CD pipelines (GitHub Actions, Azure DevOps) support Docker natively.
*   Total isolation.

---

## 26.6 The Code: Testcontainers in xUnit

### 1. Setting up the Container
We use xUnit's `IAsyncLifetime` to start the container before any tests run.

```csharp
using Testcontainers.MsSql;
using Xunit;

public class DatabaseFixture : IAsyncLifetime
{
    // Configure the SQL Server container
    private readonly MsSqlContainer _dbContainer = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public string ConnectionString => _dbContainer.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync(); // Spins up Docker!
        
        // Run EF Core Migrations to build the schema
        var options = new DbContextOptionsBuilder<VoltCoreDbContext>()
            .UseSqlServer(ConnectionString).Options;
        using var context = new VoltCoreDbContext(options);
        await context.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _dbContainer.StopAsync(); // Destroys Docker!
    }
}
```

### 2. Writing the Test
```csharp
public class StationRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public StationRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task AddStation_WithDuplicateMac_ThrowsException()
    {
        // Arrange
        var options = new DbContextOptionsBuilder<VoltCoreDbContext>()
            .UseSqlServer(_fixture.ConnectionString).Options;
        using var context = new VoltCoreDbContext(options);
        
        context.Stations.Add(new Station { MacAddress = "AA:BB" });
        await context.SaveChangesAsync();

        // Act & Assert (Testing REAL Database constraints!)
        context.Stations.Add(new Station { MacAddress = "AA:BB" });
        
        await Assert.ThrowsAsync<DbUpdateException>(() => context.SaveChangesAsync());
    }
}
```

---

## 26.7 Performance & Security Analysis

### Performance Analysis: State Resetting (Respawn)
Spinning up a Docker container takes ~5 seconds. We only do this *once* for the entire test suite.
However, we must reset the data between individual tests so Alice's test doesn't interfere with Bob's test.
Deleting data using EF Core `RemoveRange` is slow. Recreating the database is slower.
Architects use the **Respawn** library. Respawn intelligently analyzes your schema and generates optimized `TRUNCATE` and `DELETE` SQL commands, wiping all data from the database in 50 milliseconds, ensuring a pristine state for the next test.

### Security Implications
*   **Docker Socket:** Testcontainers requires access to the Docker Daemon. When running in a CI/CD pipeline, ensure the runners are securely configured (e.g., using rootless Docker or secure Docker-in-Docker DIND setups) to prevent container escape vulnerabilities.

---

## 26.8 Common Mistakes & Production Pitfalls

1.  **Testing with SQLite:** Some developers realize InMemory is bad, so they use the EF Core SQLite provider for testing, while Production uses SQL Server. This is also a trap. SQLite treats all data as strings, ignores schema boundaries, and does not support T-SQL syntax. Your tests will pass, but production will fail.
2.  **Seeding in Migrations:** If your EF Core migrations insert default data (e.g., `INSERT INTO Roles`), Respawn will wipe those tables out during test reset. You must configure Respawn's `TablesToIgnore` property to avoid deleting static reference data.

---

## 26.9 Production Checklist

*   [ ] The EF Core InMemory provider (`UseInMemoryDatabase`) has been eradicated from the integration testing suite.
*   [ ] Testcontainers is implemented to run integration tests against the exact version of SQL Server used in production.
*   [ ] The database is wiped to a clean state between every test execution using a high-performance library like Respawn.
*   [ ] CI/CD pipelines (GitHub Actions/Azure DevOps) are configured with Docker support to run the Testcontainers suite on pull requests.

---

## 26.10 Exercises

1.  **Anti-Pattern Identification:** A developer writes a test using `UseInMemoryDatabase`. They test inserting a `Station` with a 2,000-character name, even though the EF Core configuration maps it to `VARCHAR(50)`. The test passes. Why does it pass, and what exactly will happen when that same code runs in Production?
2.  **Test Isolation:** In an xUnit class with 5 tests, explain the performance difference between spinning up a new Testcontainer for *each* test (using the constructor) versus spinning it up once for the class (using `IClassFixture`) and resetting the data with Respawn.

---

## 26.11 Interview Questions

**Q1: Why is it an architectural anti-pattern to use `Mock<DbSet>` or the EF Core InMemory database for integration testing?**
*Answer:* Both approaches provide a false sense of security. Mocking the `DbSet` tests LINQ against C# in-memory lists, completely bypassing EF Core's SQL translation engine; queries that work in memory will often throw translation exceptions in production. The InMemory provider acts as a document store, not a relational database. It ignores Foreign Key constraints, Unique constraints, data type length limitations, and does not support database transactions or raw SQL execution. Testing against them proves nothing about the code's behavior in a real SQL Server environment.

**Q2: Describe how the Testcontainers library solves the "Shared Dev Database" testing problem.**
*Answer:* A shared Dev Database suffers from race conditions (tests interfering with each other's data) and schema drift. Testcontainers solves this by programmatically spinning up an isolated, ephemeral Docker container running the exact production database engine (e.g., SQL Server 2022) at the start of the test run. It dynamically provides the connection string, allows EF Core to migrate the schema from scratch, and cleanly destroys the container when the test suite completes. This guarantees perfect isolation and reproducibility in CI/CD pipelines.

---
**Next up in Chapter 27:** We begin Part 8: Security and High Availability. We will explore Authentication vs Authorization, and implement Row-Level Security (RLS) to guarantee strict multi-tenant data isolation at the engine level.
