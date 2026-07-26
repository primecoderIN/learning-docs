# Chapter 7: Database Migrations and DevOps

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Understand how EF Core tracks schema changes via the `ModelSnapshot` and the `__EFMigrationsHistory` table.
*   Master the EF Core CLI tools for creating, updating, and scripting migrations.
*   Implement Idempotent Data Seeding securely within the migration pipeline.
*   Inject Custom SQL (Views, Stored Procedures, Triggers) into standard EF Core migrations.
*   Architect an enterprise CI/CD deployment pipeline that applies migrations safely to production without utilizing the dangerous `context.Database.Migrate()` runtime method.
*   Design and execute Zero-Downtime database schema evolutions.

## 2. Introduction

Writing optimized LINQ queries and configuring the Fluent API only matters if the physical database schema actually matches your C# models. Keeping the C# codebase and the SQL Server schema in perfect synchronization across Development, Staging, and Production environments is one of the most perilous aspects of software engineering.

Historically, DBAs maintained a massive folder of `.sql` scripts. Developers had to manually write `CREATE TABLE` and `ALTER TABLE` scripts to match their code changes. This disconnected process inevitably led to deployment failures when a script was forgotten or executed out of order.

Entity Framework Core solves this with **Migrations**. Migrations are a version control system for your database schema. When you change a C# class, EF Core calculates the difference between your new code and the current schema, and automatically generates the exact C# code required to upgrade (or downgrade) the database. 

However, running migrations on a developer's laptop is vastly different from executing them against a 5-Terabyte production database with 10,000 active concurrent users. This chapter bridges the gap between local development and enterprise DevOps.

## 3. Core Concepts

### The Migration File (`Up` and `Down`)
When you generate a migration, EF Core creates a C# class with two methods. `Up()` contains the instructions to apply the new schema changes (e.g., `CreateTable`). `Down()` contains the instructions to reverse them (e.g., `DropTable`).

### The Model Snapshot (`ModelSnapshot.cs`)
EF Core maintains a single file in your project called the `ModelSnapshot`. This file is a C# representation of your *entire* database schema as it currently exists. When you create a new migration, EF Core does not look at your live database; it compares your current C# entities against this Snapshot file to calculate the diff.

### The History Table (`__EFMigrationsHistory`)
When a migration is applied to a physical database, EF Core inserts a row into a hidden table named `__EFMigrationsHistory` containing the Migration ID and the EF Core version. Before applying migrations, EF Core queries this table. If a migration is already in the table, it skips it. This guarantees that migrations are only applied once.

### Idempotent Scripts
A SQL script is idempotent if it can be executed multiple times without changing the result beyond the initial application, and without throwing errors. Idempotent scripts check if an object exists before creating it (e.g., `IF NOT EXISTS (SELECT * FROM sys.tables...) CREATE TABLE...`).

## 4. Visual Diagrams

```text
=============================================================================
             THE MIGRATION GENERATION LIFECYCLE
=============================================================================

1. Developer modifies C#: 
   public class Tenant { public string NewColumn { get; set; } }

2. Developer runs: `dotnet ef migrations add AddNewColumn`

3. EF Core computes the Diff:
   [Current C# Domain Models]  <-- COMPARES -->  [ModelSnapshot.cs]

4. EF Core generates files:
   ├── 20261015_AddNewColumn.cs (Contains Up() and Down() using MigrationBuilder)
   └── ModelSnapshot.cs (Overwritten with the new schema state)
```

```text
=============================================================================
             ENTERPRISE CI/CD DEPLOYMENT PIPELINE
=============================================================================

[ GitHub / Azure DevOps ]
       │
       ▼ (1) Build Code
[ Compile C# DLLs ] 
       │
       ▼ (2) Generate Artifacts
[ dotnet ef migrations script --idempotent ] ──▶ Generates `update.sql`
       │
       ▼ (3) Release Pipeline Starts
[ Approval Gate (DBA Review) ] 
       │
       ▼ (4) Pre-Deployment Step
[ Azure SQL Action / SqlCmd ] ──▶ Executes `update.sql` against Production DB
       │
       ▼ (5) App Deployment Step
[ Deploy new ASP.NET Core Binaries to Azure App Service ]
```

## 5. API Deep Dive: CLI and MigrationBuilder

### The Essential CLI Commands
To use these commands, you must install the CLI tool globally: `dotnet tool install --global dotnet-ef` and the `Microsoft.EntityFrameworkCore.Design` NuGet package in your project.

*   `dotnet ef migrations add <Name>`: Scans models, compares to the snapshot, and generates a new migration class.
*   `dotnet ef database update`: Reads `__EFMigrationsHistory` from the database, determines which migrations are missing, and executes their `Up()` methods sequentially.
*   `dotnet ef migrations remove`: Deletes the last unapplied migration and rolls back the `ModelSnapshot.cs`.
*   `dotnet ef migrations script <From> <To>`: Generates a raw T-SQL script containing the changes between two migrations.

### The MigrationBuilder API
Inside the generated migration class, EF Core uses the `MigrationBuilder` to define schema changes.

```csharp
public partial class AddChargerStatus : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // EF Core generated this:
        migrationBuilder.AddColumn<string>(
            name: "Status",
            table: "Chargers",
            type: "nvarchar(50)",
            maxLength: 50,
            nullable: false,
            defaultValue: "Offline");

        // YOU can add Custom SQL here!
        migrationBuilder.Sql(@"
            CREATE VIEW vw_ActiveChargers AS 
            SELECT Id, SerialNumber FROM Chargers WHERE Status = 'Active';
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP VIEW vw_ActiveChargers;");
        migrationBuilder.DropColumn(name: "Status", table: "Chargers");
    }
}
```

## 6. Complete Examples: EV Platform Case Study

We are deploying a new feature to the EV Platform: **Tariffs**. We need a new table to store pricing rules, and we need to seed the database with a default "Free" tariff so the application doesn't crash upon deployment.

### 6.1 Creating the Entity and Configuration
```csharp
public class Tariff
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal PricePerKwh { get; set; }
}

public class TariffConfiguration : IEntityTypeConfiguration<Tariff>
{
    public void Configure(EntityTypeBuilder<Tariff> builder)
    {
        builder.HasKey(t => t.Id);
        // Best Practice: Seed Data in the Configuration
        builder.HasData(
            new Tariff { Id = 1, Name = "Default Free Tier", PricePerKwh = 0.00m }
        );
    }
}
```

### 6.2 Generating the Migration
We run: `dotnet ef migrations add AddTariffs`

EF Core generates the following `Up()` method:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Tariffs",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                .Annotation("SqlServer:Identity", "1, 1"),
            Name = table.Column<string>(type: "nvarchar(max)", nullable: false),
            PricePerKwh = table.Column<decimal>(type: "decimal(18,2)", nullable: false)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Tariffs", x => x.Id);
        });

    // EF Core automatically generates the seeding SQL based on HasData!
    migrationBuilder.InsertData(
        table: "Tariffs",
        columns: new[] { "Id", "Name", "PricePerKwh" },
        values: new object[] { 1, "Default Free Tier", 0.00m });
}
```

## 7. EF Core Internals: Data Seeding

There are two ways to seed data in EF Core:
1.  **Runtime Seeding (Anti-Pattern):** Writing a `DbInitializer` class that runs `context.Tariffs.Any()` on application startup, and calls `context.Tariffs.Add()` if empty. This causes race conditions if 5 API instances start simultaneously.
2.  **Migration Seeding (Enterprise Standard):** Using `builder.HasData()` in the Fluent API. 

When you use `HasData`, EF Core tracks the seeded data exactly like it tracks schema changes. If you change the `PricePerKwh` of the seed data in C# and generate a new migration, EF Core calculates the diff and generates an `UPDATE` statement inside the new migration. This guarantees that data required for the application to function is perfectly synchronized with the schema version.

## 8. ASP.NET Core Integration: The `Migrate()` Anti-Pattern

In many tutorials, you will see this code in `Program.cs`:
```csharp
// DO NOT DO THIS IN PRODUCTION!
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<EvDbContext>();
    db.Database.Migrate(); 
}
```
**Why this is a catastrophic architectural failure:**
1.  **Race Conditions:** In Azure App Service or Kubernetes, a deployment might spin up 10 instances of your API simultaneously. All 10 instances will start up, call `Migrate()`, and attempt to execute `CREATE TABLE` at the exact same millisecond. This causes database deadlocks and application crashes.
2.  **Security:** To run `Migrate()`, the API's connection string must have Database Owner (DDL) privileges (e.g., `ALTER TABLE`). In a secure SaaS, the API connection string should ONLY have Data Manipulation (DML) privileges (`SELECT, INSERT, UPDATE, DELETE`). If the API is compromised via SQL Injection, the attacker cannot drop tables.
3.  **Observability:** If a migration fails mid-execution because a table is locked, the API fails to start. You have no visibility into the SQL failure without digging through application logs.

## 9. Enterprise DevOps: Idempotent SQL Scripts

The Architect's solution to deployment is generating an Idempotent SQL Script during the CI/CD Build pipeline.

```bash
dotnet ef migrations script --idempotent --output update.sql
```

**What `--idempotent` does:**
It wraps every single migration in a SQL `IF` statement checking the `__EFMigrationsHistory` table.

```sql
IF NOT EXISTS(SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20261015_AddTariffs')
BEGIN
    CREATE TABLE [Tariffs] (
        [Id] int NOT NULL IDENTITY,
        [Name] nvarchar(max) NOT NULL,
        [PricePerKwh] decimal(18,2) NOT NULL,
        CONSTRAINT [PK_Tariffs] PRIMARY KEY ([Id])
    );
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20261015_AddTariffs', N'9.0.0');
END;
GO
```

**The CI/CD Workflow:**
1. The Build Pipeline generates this `update.sql` artifact.
2. The Release Pipeline downloads the artifact.
3. The Release Pipeline executes `update.sql` against the Production Azure SQL database using a highly privileged DevOps Service Principal.
4. If it succeeds, the Release Pipeline deploys the compiled ASP.NET Core binaries (which use a low-privilege connection string).

## 10. Clean Architecture Perspective

Where do migrations live?
Because Migrations generate C# code that references the `Microsoft.EntityFrameworkCore` namespace, they **must** live in the Infrastructure layer. 

If your solution has an `EV.Infrastructure` project, you should run CLI commands specifying the project:
`dotnet ef migrations add AddTariffs --project EV.Infrastructure --startup-project EV.Api`

(The `--startup-project` is required so the EF tooling can boot up ASP.NET Core, resolve the DI container, and find your connection string to verify the provider, even if it's not actually applying the migration to the DB).

## 11. Enterprise SaaS Perspective: Zero-Downtime Migrations

In a high-availability SaaS, you cannot take the system offline to run a migration. This introduces the concept of **Expand and Contract**.

**Scenario:** We need to rename the `Name` column in `Tenants` to `CompanyName`.
If you rename it in C# and run a migration, EF Core generates `sp_rename 'Tenants.Name', 'CompanyName'`.
If you run this against Production, the running API instantly crashes because its compiled SQL queries are still looking for `[Name]`.

**The Zero-Downtime Workflow (4 Deployments):**
1.  **Expand (Migration 1):** Add `CompanyName` as a *new*, nullable column. Do not remove `Name`. Deploy.
2.  **Dual-Write (App Code):** Update the API to write to *both* `Name` and `CompanyName`, but read from `Name`. Deploy.
3.  **Backfill (Custom SQL):** Run a background script: `UPDATE Tenants SET CompanyName = Name WHERE CompanyName IS NULL`.
4.  **Transition (App Code):** Update the API to read and write exclusively to `CompanyName`. Deploy.
5.  **Contract (Migration 2):** Drop the `Name` column. Deploy.

*Architectural Rule:* Never rename columns, drop columns, or change column types in a single deployment if zero-downtime is required. Migrations must be purely additive until the API is fully migrated away from the old schema.

## 12. Real Production Case Study

In the EV Platform, we needed to implement full-text search across all `Sites`. We decided to use SQL Server Full-Text Indexes, which EF Core does not natively support via the Fluent API.

We generated an empty migration: `dotnet ef migrations add AddFullTextIndex`.
We opened the generated file and manually wrote the Custom SQL.

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Create the catalog
    migrationBuilder.Sql("CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT;", suppressTransaction: true);
    
    // Create the index on the LocationName column
    migrationBuilder.Sql(
        "CREATE FULLTEXT INDEX ON Sites(LocationName) KEY INDEX PK_Sites ON ftCatalog;", 
        suppressTransaction: true);
}
```
*Note: We must pass `suppressTransaction: true` because SQL Server does not allow Full-Text Catalog creation inside a standard user transaction, which EF Core wraps migrations in by default.*

## 13. Common Mistakes

### Beginner
*   **Mistake:** Deleting a migration file from the Solution Explorer because you made a mistake.
*   **Correction:** This breaks the `ModelSnapshot.cs`. EF Core will still think the migration exists. Always use the CLI command `dotnet ef migrations remove` to safely remove the last generated migration and rollback the snapshot.

### Intermediate
*   **Mistake:** Attempting to merge a Git branch where a colleague added a migration, while you also added a migration locally. The `ModelSnapshot.cs` is now in a severe merge conflict.
*   **Correction:** Do not try to manually resolve complex snapshot merge conflicts. Accept the incoming branch's changes. Delete your local migration file. Then, re-run `dotnet ef migrations add YourMigrationName`. EF Core will instantly recalculate the diff against the newly merged snapshot and generate a fresh, perfectly aligned migration.

### Senior
*   **Mistake:** Using `DbContext.Database.EnsureCreated()` in a production application.
*   **Correction:** `EnsureCreated()` completely bypasses the Migrations history table. It generates the schema directly. If you use it, you can *never* use Migrations on that database in the future, as EF Core has no baseline history. It is meant exclusively for integration testing or throwaway prototypes.

### Architect
*   **Mistake:** Storing application-level lookup data (e.g., a list of 5,000 zip codes) using EF Core `HasData` seeding.
*   **Correction:** `HasData` generates explicit `INSERT` statements inside the migration files. Seeding 5,000 rows will generate a 50,000-line C# migration file that crashes Visual Studio and takes 10 minutes to compile. `HasData` is exclusively for critical, static system constants (e.g., 3 standard Application Roles). Large datasets must be seeded via raw SQL scripts or separate data-import pipelines.

## 14. Interview Questions

### Beginner (10)
1.  **What is an EF Core Migration?**
    *Answer:* A way to keep the database schema in sync with the C# EF Core model. It generates code that represents the differences between the current model and the previous state.
2.  **What CLI command generates a new migration?**
    *Answer:* `dotnet ef migrations add <MigrationName>`
3.  **What CLI command applies the migration to the database?**
    *Answer:* `dotnet ef database update`
4.  **How does EF Core know which migrations have already been applied to a database?**
    *Answer:* By querying the `__EFMigrationsHistory` table in the database.
5.  **What is the purpose of the `Up()` and `Down()` methods?**
    *Answer:* `Up()` contains instructions to apply the new changes. `Down()` contains instructions to revert those exact changes.
6.  **What is the `ModelSnapshot.cs` file?**
    *Answer:* A C# representation of the entire current database schema. EF Core compares your entities against this file to determine what changed.
7.  **Is it safe to manually modify the generated `Up()` method?**
    *Answer:* Yes. It is common practice to inject raw SQL using `migrationBuilder.Sql()` for features EF Core doesn't support.
8.  **How do you safely delete a migration you just created but haven't applied yet?**
    *Answer:* `dotnet ef migrations remove`.
9.  **What does `DbContext.Database.EnsureCreated()` do?**
    *Answer:* It creates the database schema directly without using migrations. It is incompatible with the migrations workflow.
10. **Where should migration files live in a Clean Architecture solution?**
    *Answer:* In the Infrastructure project.

### Intermediate (10)
11. **Explain what an Idempotent script is in the context of EF Core.**
    *Answer:* A SQL script that checks the `__EFMigrationsHistory` table before applying a migration. It can be run repeatedly against the same database without causing errors or duplicate data.
12. **Why is calling `context.Database.Migrate()` at application startup dangerous in production?**
    *Answer:* It causes race conditions if multiple application instances start simultaneously, and requires the application connection string to have dangerous DDL (Schema modification) privileges.
13. **How do you seed data using migrations?**
    *Answer:* By using the `HasData()` method in the Fluent API configuration. EF Core will translate this into `INSERT` or `UPDATE` statements inside the migration.
14. **What happens if you change a seeded value in `HasData()` and generate a new migration?**
    *Answer:* EF Core detects the change in the C# model and generates an `UPDATE` statement in the new migration's `Up()` method.
15. **How do you resolve a Git merge conflict in the `ModelSnapshot.cs`?**
    *Answer:* Discard your local migration and snapshot changes, pull the remote branch, and then regenerate your local migration.
16. **What is a "pending" migration?**
    *Answer:* A migration that exists in your C# codebase but has not yet been recorded in the `__EFMigrationsHistory` table of the target database.
17. **Can you apply a migration directly from code without using the CLI?**
    *Answer:* Yes, using `context.Database.Migrate()`, though it is an anti-pattern for production. You can also generate the script programmatically via `context.Database.GenerateCreateScript()`.
18. **How do you specify a different name for the migrations history table?**
    *Answer:* In the `UseSqlServer` configuration: `options.UseSqlServer(conn, x => x.MigrationsHistoryTable("MyHistoryTable"));`
19. **If you rename a property from `FirstName` to `GivenName`, what does EF Core do when generating the migration?**
    *Answer:* EF Core cannot confidently know if you renamed a column or dropped one and added another. It usually generates a `DropColumn` and `AddColumn` (causing data loss!). You must manually edit the generated migration to use `RenameColumn` instead.
20. **How do you apply migrations to a database that is entirely offline from your development machine?**
    *Answer:* Generate an idempotent SQL script (`dotnet ef migrations script --idempotent`) and hand the `.sql` file to the DBA to execute via SSMS.

### Senior (10)
21. **Architect a deployment pipeline for EF Core migrations using Azure DevOps.**
    *Answer:* The Build Pipeline compiles the code and runs `dotnet ef migrations script --idempotent` to produce an artifact (`update.sql`). The Release Pipeline uses an Azure SQL task to execute `update.sql` using a Service Principal with DDL rights. Upon success, the pipeline deploys the App Service binaries, which run with a separate Managed Identity containing only DML rights.
22. **Explain the Expand and Contract pattern for Zero-Downtime schema migrations.**
    *Answer:* You never make destructive changes (renames, drops, type changes) in a single deployment. You Expand (add new column), update app to dual-write, backfill data, transition app to read from new column, and finally Contract (drop old column) in a subsequent deployment.
23. **You need to create a SQL Server Stored Procedure. Where is the correct place to define the `CREATE PROCEDURE` script in an EF Core project?**
    *Answer:* Generate an empty migration. Add `migrationBuilder.Sql("CREATE PROCEDURE...");` to the `Up()` method, and `migrationBuilder.Sql("DROP PROCEDURE...");` to the `Down()` method. This ensures the procedure is version-controlled and deployed sequentially with table changes.
24. **Why might you need to pass `suppressTransaction: true` to `migrationBuilder.Sql()`?**
    *Answer:* EF Core wraps the entire `Up()` method in a SQL Transaction by default. Certain SQL Server commands (like `CREATE FULLTEXT CATALOG`, `ALTER DATABASE`, or modifying memory-optimized tables) cannot be executed within a user transaction. Suppressing the transaction allows these commands to execute.
25. **Evaluate the performance differences between seeding data via `HasData` versus writing a custom SQL script executed by DbUp.**
    *Answer:* `HasData` parses the C# objects into `INSERT` statements inside the migration file. For small datasets (Roles, Statuses), it is excellent. For large datasets (e.g., 10,000 zip codes), it creates a massive C# file that compiles slowly and executes slowly via ADO.NET parameterization. DbUp executing a bulk `.sql` file (or `SqlBulkCopy`) is orders of magnitude faster for large data seeding.
26. **How do you handle migrations when different developers are working on features that target different bounded contexts (DbContexts) within the same physical database?**
    *Answer:* You must ensure each `DbContext` has its own unique Migrations History table. Configure `x.MigrationsHistoryTable("__EFMigrationsHistory_Billing")` for the Billing context, and `__EFMigrationsHistory_Identity` for the Identity context.
27. **What happens if a migration fails halfway through execution on SQL Server?**
    *Answer:* Because EF Core wraps the `Up()` execution in a transaction, the entire migration rolls back (including structural changes), leaving the database in its previous pristine state. The `__EFMigrationsHistory` table is not updated.
28. **A developer accidentally dropped a column in a previous migration that has already been deployed to production. They want to edit the old migration file to remove the `DropColumn` command. Why is this a catastrophic idea?**
    *Answer:* Editing a previously applied migration does absolutely nothing to the production database, because EF Core skips it based on the history table. However, it completely breaks the `ModelSnapshot`, corrupting future diff calculations. The Architect must enforce a "Roll Forward Only" policy: fix the mistake by creating a *new* migration that re-adds the column.
29. **How do you configure a migration to execute a specific data-scrubbing script before dropping a table?**
    *Answer:* Open the generated migration file. Insert your `migrationBuilder.Sql("UPDATE OtherTable SET Ref = NULL WHERE...");` command *above* the `migrationBuilder.DropTable()` command in the `Up()` method. EF Core guarantees sequential execution within the transaction.
30. **Explain how EF Core migrations interact with SQL Server Temporal Tables (System-Versioned).**
    *Answer:* If you configure an entity with `.ToTable(tb => tb.IsTemporal())`, the initial migration generates the `CREATE TABLE` with `GENERATED ALWAYS AS ROW START/END` and creates the associated History table. If you subsequently drop a column, EF Core correctly handles disabling system-versioning, dropping the column from both main and history tables, and re-enabling system-versioning natively.

### Staff Engineer (5)
31. **Architect a mechanism to automatically apply migrations in a Kubernetes cluster without using `context.Database.Migrate()` in the API pods, ensuring high availability and zero race conditions.**
    *Answer:* Do not apply migrations from the API pods. Use a Kubernetes **Init Container** or a **Pre-Sync Helm Hook** (Kubernetes Job). This dedicated short-lived container contains the EF Core CLI (or a small compiled worker service). It executes the migration script sequentially. Only when this Job completes successfully does the deployment proceed to spin up the actual API pods. This guarantees single-threaded execution and strict separation of privileges.
32. **A SaaS platform uses a separate database per tenant (10,000 databases). The schema must be identical across all. Formulate a strategy to apply EF Core migrations to 10,000 databases efficiently.**
    *Answer:* Using EF Core CLI to apply migrations to 10k databases sequentially would take hours. The Architect must build a custom control-plane service. This service generates the Idempotent SQL Script from EF Core once. It then utilizes a distributed worker queue (e.g., RabbitMQ + Hangfire) or Azure Elastic Jobs to fan-out the execution of the raw SQL script across 10,000 databases in parallel using raw ADO.NET, tracking success/failure for each tenant individually.
33. **Analyze the risks of using EF Core Migrations for extensive data manipulation (DML) alongside schema definition (DDL) within the same `Up()` method.**
    *Answer:* DDL commands (like `CREATE INDEX`) can require schema locks. DML commands (like `UPDATE 1,000,000 rows`) take immense time and transaction log space. Combining them inside a single implicit EF Core transaction risks massive lock escalation, potentially blocking the entire database for minutes, leading to an application outage. The Architect should segregate heavy data migrations from schema migrations, often executing data backfills via background jobs outside the strict deployment window.
34. **Design a solution to reverse a deployed migration that caused a catastrophic production issue, assuming you cannot restore from a database backup due to strict data-loss constraints.**
    *Answer:* Never attempt to run `dotnet ef database update <PreviousMigrationName>` in production. This executes the `Down()` method, which usually drops tables or columns, causing irreversible data loss. The Architect mandates "Roll Forward". You revert the C# code in Git (e.g., `git revert`). You run `dotnet ef migrations add RevertBadFeature`. EF Core computes the diff (which looks identical to the original `Down()` method). You manually review and modify this new `Up()` method to ensure it preserves data (e.g., moving data to a temp table before dropping the main column), and deploy it as a standard new release.
35. **Evaluate the internal diffing algorithm of EF Core when determining if an index has changed, and how it impacts CI/CD pipelines.**
    *Answer:* EF Core calculates diffs by comparing the exact state of the `IModel` against the `ModelSnapshot`. If you change an index from non-unique to unique, EF Core detects this. However, it does not detect changes to database-level constraints applied manually via raw SQL in previous migrations. If your CI/CD pipeline relies on `dotnet ef migrations has-pending-model-changes` (a command to check if a developer forgot to generate a migration), it will only validate the EF Core metadata, completely ignoring any manual SQL drift that a DBA might have introduced directly into production.

## 15. Exercises

### Easy
1.  **Generate and Apply:** Add a new property `IsPremium` to an existing entity. Run the CLI command to generate the migration. Run the CLI command to apply it to your local database. Verify in SSMS that the column exists.

### Medium
1.  **Custom SQL:** Generate an empty migration (`dotnet ef migrations add AddUserView`). In the `Up()` method, write `migrationBuilder.Sql()` to create a SQL View. In the `Down()` method, write the SQL to drop the view. Apply the migration.
2.  **Idempotent Scripts:** Run the command to generate an idempotent script for all your migrations. Open the `.sql` file and identify the `IF NOT EXISTS` block surrounding the `__EFMigrationsHistory` check.

### Hard
1.  **Zero-Downtime Simulation:** Rename a property on an entity. Generate the migration. Observe that EF Core generated a `RenameColumn` command. Delete the migration. Now, execute the Expand and Contract pattern: Add the new property, generate a migration (Expand). Then drop the old property, and generate a second migration (Contract).

### Enterprise
1.  **Automated Script Generation (CI/CD):** Create a simple GitHub Actions YAML file (or PowerShell script) that executes during the "Build" phase. The script must compile your EF Core project, run the `dotnet ef migrations script --idempotent` command, and output an `update.sql` file as a build artifact, proving you have decoupled migration generation from execution.

## 16. Production Checklist

- [ ] Are migrations executed via Idempotent SQL scripts in the CI/CD pipeline rather than via `context.Database.Migrate()` at runtime?
- [ ] Does the ASP.NET Core application use a connection string with reduced privileges (no DDL rights) in production?
- [ ] Is data seeding limited to essential system constants, avoiding massive datasets in migration files?
- [ ] Are destructive changes (renames, drops) managed via the Expand and Contract pattern to ensure zero downtime?
- [ ] Are custom SQL objects (Views, Procedures) managed inside explicit migration files to ensure they are version controlled?

## 17. Summary

Migrations transform database management from a chaotic, manual process into a predictable, version-controlled engineering discipline. By understanding the `ModelSnapshot`, you can resolve merge conflicts and orchestrate complex, zero-downtime schema evolutions. By mastering Idempotent Scripts, you bridge the gap between developer tooling and enterprise DevOps pipelines, guaranteeing secure, race-condition-free deployments.

With our schema securely deployed and our queries optimized, we must now address the most critical aspect of Enterprise SaaS: Data Integrity. In the next chapter, we will master Transactions, Concurrency Control, and Resilience against cloud network failures.
