# Module 10: Database Migrations in EF Core

When you build an application with Entity Framework Core, your primary source of truth is your C# code (your Domain Entities and Fluent API configurations). However, a relational database needs a physical schema (Tables, Columns, Primary Keys, Foreign Keys) to store that data.

How do you keep the C# code and the SQL Server schema perfectly synchronized as your application evolves? The answer is **Migrations**.

---

## 1. What is a Migration?

A **Migration** is an auto-generated set of instructions (written in C#) that tells EF Core exactly how to alter the database schema to match the current state of your `DbContext`. 

Think of a migration as a "git commit" for your database. 

Every time you change your C# models (e.g., adding a new `Age` property to the `User` class), you create a new Migration. EF Core looks at the old state of the database, looks at the new state of your C# code, and generates a file that basically says: *"To get from State A to State B, execute `ALTER TABLE Users ADD Age INT NULL`."*

When you apply the migration, EF Core translates those C# instructions into pure SQL and runs them against your database.

---

## 2. WHY do we need Migrations? 

Before Migrations existed, developers used to manually write SQL scripts to update the database, or they would delete the entire database and recreate it every time they changed a C# class (a feature called `EnsureCreated()`). 

In a production enterprise environment, neither of those approaches work. We need Migrations for three critical reasons:

### 1. Version Control for your Schema
Your database schema evolves over time. Migrations create a historical timeline of exactly how your database changed from Day 1 to Year 5. Because migrations are just `.cs` files, they get committed to your Git repository alongside your normal code. This ensures your code and your database schema are always perfectly aligned in history.

### 2. Team Collaboration
Imagine you are building Normora on a team of 5 developers.
*   Developer A adds a `StripeCustomerId` column to the `Tenants` table.
*   Developer B adds an `IsActive` column to the `Users` table.

Without migrations, Developer A and B would constantly break each other's local databases. With migrations, Developer A generates a migration file, commits it to Git, and Developer B just pulls the code and runs `Update-Database`. EF Core automatically applies Developer A's changes to Developer B's local database.

### 3. Safe Rolling Back (Undo)
If you deploy a new feature to production and realize you made a massive mistake (e.g., a column data type is wrong and crashing the app), you can use migrations to safely "roll back" the database schema to the exact state it was in yesterday, without losing the data in the other tables.

---

## 3. Which Tools are Available for Migrations?

To generate and apply migrations, you don't write the C# migration files yourself. You use command-line tooling provided by Microsoft. 

There are **two entirely different toolsets** you can use. They both do the exact same thing behind the scenes; the choice just depends on which IDE you are using.

### Option A: The .NET Core CLI (`dotnet ef`)
This is the modern, cross-platform command-line tool. You use this if you are developing on a Mac, using Visual Studio Code (VS Code), using Rider, or running automated CI/CD pipelines (like GitHub Actions).

**Setup:** You must install it globally on your machine once:
```bash
dotnet tool install --global dotnet-ef
```

**The Commands (Terminal):**
1. **Create a migration:** `dotnet ef migrations add AddedAgeToUser`
2. **Apply to database:** `dotnet ef database update`
3. **Remove the last migration (if unapplied):** `dotnet ef migrations remove`
4. **Generate a raw SQL script for a DBA:** `dotnet ef migrations script`

### Option B: The Package Manager Console (Visual Studio only)
This is the classic PowerShell-based tool built directly into Visual Studio on Windows. If you are using full Visual Studio, this is usually the preferred and slightly faster method.

**Setup:** You must install the `Microsoft.EntityFrameworkCore.Tools` NuGet package into your startup project.

**The Commands (Package Manager Console):**
1. **Create a migration:** `Add-Migration AddedAgeToUser`
2. **Apply to database:** `Update-Database`
3. **Remove the last migration:** `Remove-Migration`
4. **Generate a raw SQL script:** `Script-Migration`

---

## 4. The Anatomy of a Migration File

When you run `Add-Migration AddedAgeToUser`, EF Core generates a C# file with a timestamp in your project (e.g., `20231027_AddedAgeToUser.cs`). 

If you open it, you will see two very important methods: `Up()` and `Down()`.

```csharp
public partial class AddedAgeToUser : Migration
{
    // The Up() method runs when you are UPGRADING the database
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<int>(
            name: "Age",
            table: "Users",
            type: "int",
            nullable: true);
    }

    // The Down() method runs when you are ROLLING BACK the database
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "Age",
            table: "Users");
    }
}
```

### The `__EFMigrationsHistory` Table (Database Migration History)
How does EF Core know which migrations have already been applied to your database? Does the database itself keep track of this history?

**Yes, absolutely.** The database maintains its own history table to track which migrations have been executed. The very first time you apply a migration (`Update-Database` or `dotnet ef database update`), EF Core creates a special system table called **`__EFMigrationsHistory`**.

#### 1. What the Table Looks Like
The physical structure of `__EFMigrationsHistory` is lightweight and consists of two columns:

| Column Name | Data Type | Description | Example Value |
| :--- | :--- | :--- | :--- |
| **`MigrationId`** *(PK)* | `nvarchar(150)` | Timestamp prefix + Migration name | `20261011140000_AddedAgeToUser` |
| **`ProductVersion`** | `nvarchar(32)` | The EF Core version used to apply the migration | `9.0.0` |

* `MigrationId` acts as the Primary Key. Because it starts with a UTC timestamp (e.g., `YYYYMMDDHHMMSS`), migrations are naturally sorted in chronological order.

#### 2. How EF Core Uses This Table (The Sync Lifecycle)
Whenever an update command or migration bundle runs, EF Core synchronizes your C# code against this database table:

```mermaid
sequenceDiagram
    autonumber
    participant App as EF Core CLI / Bundle
    participant DB as SQL Server (__EFMigrationsHistory)

    App->>App: Scans compiled code for Migration .cs files
    App->>DB: SELECT [MigrationId] FROM [__EFMigrationsHistory]
    DB-->>App: Returns list of already applied migrations
    App->>App: Computes difference (Pending = Code Migrations - Applied Migrations)
    loop For each Pending Migration
        App->>DB: Executes Migration Up() DDL (e.g. ALTER TABLE ...)
        App->>DB: INSERT INTO [__EFMigrationsHistory] VALUES ('20261011140000_AddedAgeToUser', '9.0.0')
    end
```

1. **Scan Code:** EF Core inspects your compiled assembly to discover all migration classes inheriting from `Migration`.
2. **Query Database:** It runs `SELECT [MigrationId] FROM [__EFMigrationsHistory]`.
3. **Compute Diff:** Any migration present in your code that is **missing** from `__EFMigrationsHistory` is flagged as **pending**.
4. **Apply & Record:** EF Core executes the pending migration's `Up()` SQL and immediately inserts a row into `__EFMigrationsHistory` inside the exact same database transaction. If the DDL fails, the insert is rolled back!

#### 3. How Idempotent SQL Scripts Use It
When generating SQL scripts for a DBA or automated CI/CD pipeline (`dotnet ef migrations script -i` or `Script-Migration -Idempotent`), EF Core wraps every migration block with a conditional check against this history table:

```sql
IF NOT EXISTS (
    SELECT * FROM [__EFMigrationsHistory] 
    WHERE [MigrationId] = N'20261011140000_AddedAgeToUser'
)
BEGIN
    -- 1. Apply DDL
    ALTER TABLE [Users] ADD [Age] int NULL;

    -- 2. Record migration in history table
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20261011140000_AddedAgeToUser', N'9.0.0');
END;
```

This ensures the script is completely safe to run against any database environment—if a migration has already been executed, SQL Server simply skips it.

#### 4. Customizing the History Table (Modular Monoliths & Custom Schemas)
By default, the table is named `dbo.__EFMigrationsHistory`. If you are building a Modular Monolith or microservices architecture where multiple `DbContext` instances share one database, you must isolate their migration tracking to avoid collisions:

```csharp
builder.Services.AddDbContext<BillingDbContext>(options =>
    options.UseSqlServer(connectionString, sql =>
        sql.MigrationsHistoryTable("__EFMigrationsHistory_Billing", "billing")));
```

#### 5. Industry Standard Across Tools
Tracking migrations directly within a dedicated database table is not unique to EF Core—it is the universal design pattern across all enterprise database migration tools:
* **EF Core (.NET):** `__EFMigrationsHistory`
* **DbUp (.NET):** `SchemaVersions`
* **Flyway (Java / Polyglot):** `flyway_schema_history`
* **Liquibase:** `DATABASECHANGELOG`

---

## 5. Types of Migrations in Enterprise Applications

When developers talk about "Types of Migrations," they are referring to different strategies for generating and applying schema changes. Here are the 4 main types you will encounter:

### 1. Code-First Migrations (Schema Migrations)
This is the absolute standard and most common type of migration in EF Core.
*   **What it is:** You write your C# domain models and configure your Fluent API first. Then you run `Add-Migration`, and EF Core generates the `Up()` and `Down()` methods to create the SQL tables.
*   **Use Case:** Adding new features. For example, adding a `StripeCustomerId` property to the `Tenant` class and generating a migration to add that column to the database.

### 2. Empty Migrations (For Raw SQL, Views, and Stored Procedures)
EF Core is amazing at generating tables and columns, but it **cannot** auto-generate SQL Views, Stored Procedures, Triggers, or complex SQL Functions from C# code. 
*   **What it is:** You run `Add-Migration CreateUserReportView` *without actually changing any C# code*. EF Core generates a completely blank migration file.
*   **Use Case:** You open the blank file and manually write raw SQL inside the `Up()` method using `migrationBuilder.Sql()`. 
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Manually writing a SQL View in an empty migration!
    migrationBuilder.Sql(@"
        CREATE VIEW ActiveUsersView AS 
        SELECT Id, Name FROM Users WHERE IsActive = 1
    ");
}
```

### 3. Data Migrations (Preventing Data Loss During Refactoring)
The golden rule of migrations is to **never lose production data**. However, if you do a "destructive" refactor (like renaming a column, or splitting `FullName` into `FirstName` and `LastName`), EF Core will generate a migration that drops the old column. 

When you run `Add-Migration` for a destructive change, EF Core will print a bright yellow warning in your terminal:
> ⚠️ **WARNING: An operation was scaffolded that may result in the loss of data. Please review the migration for accuracy.**

If you just run `Update-Database`, you will lose all the users' names! To prevent data loss, you must manually intervene and add a **Data Migration** step *before* the column is dropped:

1. You create a Code-First migration to add the new `FirstName` and `LastName` columns.
2. Before EF Core drops the old column, you manually add a SQL step in the `Up()` method to safely move the data:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // 1. EF Core auto-generates adding the new columns (Safe)
    migrationBuilder.AddColumn<string>(name: "FirstName", table: "Users", ...);
    migrationBuilder.AddColumn<string>(name: "LastName", table: "Users", ...);

    // 2. YOU MANUALLY INTERVENE: Write SQL to move the data safely!
    migrationBuilder.Sql(@"
        UPDATE Users 
        SET FirstName = SUBSTRING(FullName, 1, CHARINDEX(' ', FullName) - 1),
            LastName = SUBSTRING(FullName, CHARINDEX(' ', FullName) + 1, LEN(FullName))
        WHERE FullName IS NOT NULL
    ");

    // 3. EF Core auto-generates dropping the old column. 
    // Now it is safe to drop, because we saved the data in step 2!
    migrationBuilder.DropColumn(name: "FullName", table: "Users");
}
```

### 4. Database-First Migrations (Reverse Engineering / Scaffolding)
This is the exact opposite of Code-First.
*   **What it is:** You don't write any C# code. You already have a massive, pre-existing SQL Server database (perhaps built 10 years ago by a DBA). You use the EF Core CLI command `dotnet ef dbcontext scaffold` to connect to the database.
*   **Use Case:** EF Core reads the SQL Server schema and automatically generates all the C# entity classes and the `DbContext` for you. Once scaffolded, you can switch back to Code-First for future updates.

### Bonus: "Squashed" Migrations
Over 5 years, your enterprise app might accumulate 300 migration files. This slows down build times and clutters the project. "Squashing" is the process of deleting all 300 migration files and generating a single, massive "InitialCreate" migration that represents the entire current state of the database in one file, allowing you to start fresh.

---

## 6. Advanced Architecture: Snapshots, Deltas, and Syncing Databases

To master migrations, you must understand the underlying engine architecture and how to sync different environments (like Dev and Prod).

### Snapshot-Based vs Delta-Based Migrations
Entity Framework Core uses **both** of these concepts working together. It is a **Delta-based** migration system that is *powered* by a **Snapshot**.

1. **Snapshot-Based (State-Based):**
   A "Snapshot" represents the complete, entire state of your database at a specific moment in time. In EF Core, this is the `[YourDbContext]ModelSnapshot.cs` file. Every time you run `Add-Migration`, EF Core does *not* look at your actual SQL Server database to see what changed. Instead, it compares your current C# code against this massive Snapshot file.
2. **Delta-Based (Diff-Based):**
   "Delta" means "Difference." A Delta-based migration is a file that contains **only the changes** between State A and State B. The actual migration files EF Core generates (like `20231027_AddedAge.cs`) are Delta-based. They only contain the exact `Up()` and `Down()` instructions needed to move the database forward one step. 

**The Lifecycle:** You run `Add-Migration` -> EF Core checks the Snapshot -> Generates a Delta migration file -> Updates the Snapshot for next time.

### Syncing Production and Development Databases
Can you compare the Prod and Dev databases and bring Prod up to Dev's level? **Yes, but you don't use EF Core to do it directly.**

EF Core is designed to compare *C# Code vs a Database*. It is not designed to compare Database A vs Database B. You have two ways to sync them:

**1. The EF Core Way (The Standard CI/CD Approach)**
If your Dev database is simply 3 migrations ahead of Prod, you don't actually need to "compare" them. Because EF Core tracks history in the `__EFMigrationsHistory` table, you generate an idempotent SQL script from your migrations and apply it.

#### Clarifying the Terminology: Generating vs. Translating
* `dotnet ef migrations add <Name>` **generates the C# migration files** (`.cs`) representing schema deltas.
* `dotnet ef migrations script` **translates** those C# migrations into a **pure SQL script** so you can run it via a CI/CD pipeline, DBA review, or a container.

#### The 3 Ways to Apply Migrations in Practice

| Method | Command | Where It's Used | Needs .NET SDK on Target Server? |
| :--- | :--- | :--- | :---: |
| **1. Direct CLI Update** | `dotnet ef database update` | **Local Development Only** | **Yes** |
| **2. SQL Script** | `dotnet ef migrations script -i -o script.sql` | **CI/CD Pipelines & DBA Approvals** | **No** (runs via `sqlcmd`, SSMS, or Azure task) |
| **3. Migration Bundle** | `dotnet ef migrations bundle` | **Kubernetes Init Containers / Cloud Pipelines** | **No** (self-contained executable binary) |

---

#### Pro Tip: Always Use `--idempotent` (`-i`)
You might see developers outputting scripts via terminal redirection:
```bash
dotnet ef migrations script > script.txt
```

While redirection (`> script.txt`) technically writes a file, in enterprise CI/CD pipelines you should always use the official CLI flags:

```bash
dotnet ef migrations script --idempotent --output migrate.sql
# or shorthand:
dotnet ef migrations script -i -o migrate.sql
```

##### Why `--idempotent` (`-i`) is Critical:
* **Without `-i`:** The generated script blindly executes `CREATE TABLE` and `ALTER TABLE`. If any of those tables or columns already exist in the target database, SQL Server throws an exception and the deployment crashes.
* **With `-i`:** EF Core wraps every single migration in a safety check against `__EFMigrationsHistory`:
  ```sql
  IF NOT EXISTS (
      SELECT * FROM [__EFMigrationsHistory] 
      WHERE [MigrationId] = N'20261011_AddedUserAge'
  )
  BEGIN
      ALTER TABLE [Users] ADD [Age] int NULL;
      INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
      VALUES (N'20261011_AddedUserAge', N'9.0.0');
  END;
  ```
Now the script is **safe to run anytime**—it will only apply what is currently missing and safely skip what has already been applied!

---

#### How to Run This Script in a Pipeline or Container

##### A. In a CI/CD Pipeline (GitHub Actions / Azure DevOps)
In an enterprise release pipeline, the build stage creates the script as an artifact, and the deployment stage executes it against the production database:

```mermaid
sequenceDiagram
    autonumber
    participant CI as CI Build Step
    participant CD as CD Release Step
    participant DB as Production Database

    CI->>CI: dotnet ef migrations script -i -o migrate.sql
    CI->>CD: Passes migrate.sql as Build Artifact
    CD->>DB: Runs via sqlcmd or pipeline task
    DB-->>CD: Schema Updated Successfully!
```

Example GitHub Actions step:
```yaml
# Step 1: Generate the idempotent script
- name: Generate SQL Migration Script
  run: dotnet ef migrations script --idempotent --output ./migrate.sql

# Step 2: Execute script against target DB using sqlcmd (no .NET SDK needed on DB runner!)
- name: Run SQL Script on Azure SQL / SQL Server
  run: |
    sqlcmd -S "${{ secrets.DB_SERVER }}" -d "${{ secrets.DB_NAME }}" \
           -U "${{ secrets.DB_USER }}" -P "${{ secrets.DB_PASSWORD }}" \
           -i ./migrate.sql
```

##### B. In a Docker / Kubernetes Container
You have two clean approaches:

* **Option 1 (Using the SQL Script):** Use a lightweight container with `sqlcmd` installed as an **init-container** in Kubernetes. It executes `sqlcmd -i ./migrate.sql` before your main ASP.NET Core API pod spins up.
* **Option 2 (The Modern .NET Alternative — Migration Bundles):** Instead of generating a text SQL script and requiring `sqlcmd`, compile a **Migration Bundle** (explained in full detail below).

---

### Deep Dive: EF Core Migration Bundles (The Modern Cloud & Container Standard)

Introduced in EF Core 6, **Migration Bundles** solve the fundamental operational friction of deploying database migrations in modern cloud environments.

#### 1. WHAT is an EF Core Migration Bundle?
A **Migration Bundle** is a **single-file, self-contained executable binary** (e.g., `bundle.exe` on Windows or `./bundle` ELF binary on Linux) that packages:
* All your compiled C# migration classes (`.cs`)
* Your `DbContext` and Fluent API model metadata
* The internal EF Core runtime migration engine
* The compiled database provider ADO.NET driver (e.g., `Microsoft.Data.SqlClient`, `Npgsql`)

It turns your database migrations into a **portable, standalone CLI tool** that can execute against any database instance without needing external tools.

```mermaid
graph LR
    subgraph "CI Build Server (Has .NET SDK & Source Code)"
        Code[C# Migrations + DbContext] --> CLI["dotnet ef migrations bundle"]
        CLI --> Bundle["migrate-bundle<br>(Single Standalone Binary)"]
    end

    subgraph "Production / Kubernetes (NO .NET SDK, NO Source Code)"
        Bundle --> Exec["./migrate-bundle --connection '...'"]
        Exec --> DB[(Production SQL Database)]
    end
```

---

#### 2. WHY Was It Created? (The Problems It Solves)
Before Migration Bundles, developers only had two choices for production deployments, and both introduced friction:

* **The Problem with `dotnet ef database update` in Production:**
  * Requires the **entire .NET SDK** (hundreds of megabytes) installed on the deployment agent or runner.
  * Requires your **entire source code repository** to be present on the production runner to compile on the fly.
  * Having compilers, SDKs, and raw source code on production infrastructure violates enterprise security compliance and creates massive container images.

* **The Problem with SQL Scripts (`dotnet ef migrations script`):**
  * Requires installing and maintaining external database CLI utilities (such as `sqlcmd` for SQL Server or `psql` for PostgreSQL) on deployment runners.
  * Installing and authenticating these utilities inside minimal Linux containers (like Alpine or Distroless images) is brittle and complex.

* **The Migration Bundle Solution:**
  * Compiles once during the CI build stage.
  * Requires **zero .NET SDK, zero source code, and zero SQL client tools** on the target runner.
  * Executes directly against the database with a single command.

---

#### 3. HOW Does It Work?

##### Step 1: Generate the Bundle (in CI / Build Server)
Run this command from your terminal or CI build script:

```bash
# Standard compilation:
dotnet ef migrations bundle --output ./migrate-bundle

# In multi-project / Clean Architecture solutions:
dotnet ef migrations bundle \
  --project src/Infrastructure \
  --startup-project src/Api \
  --output ./artifacts/migrate-bundle
```

###### 💡 Cross-Platform Compiling (Developing on Windows/Mac, Deploying to Linux):
If your production infrastructure or Kubernetes pods run Linux, but you develop on Windows or macOS, cross-compile using `--target-runtime`:
```bash
dotnet ef migrations bundle --target-runtime linux-x64 --output ./migrate-bundle
```

##### Step 2: Execute the Bundle (in CD / Production)
On your deployment server or container runner, execute the binary and pass the connection string:

```bash
# Linux / macOS:
./migrate-bundle --connection "Server=sql-prod;Database=NormoraDb;User Id=sa;Password=Secret123;TrustServerCertificate=True"

# Windows:
.\migrate-bundle.exe --connection "Server=sql-prod;Database=NormoraDb;Integrated Security=True;TrustServerCertificate=True"
```

###### Passing Connection Strings via Environment Variables:
The bundle automatically reads standard ASP.NET Core environment variables without needing explicit CLI arguments:
```bash
export ConnectionStrings__DefaultConnection="Server=sql-prod;Database=NormoraDb;..."
./migrate-bundle
```

###### Previewing the SQL Before Execution (`--script`):
If a DBA or security engineer wants to inspect the exact SQL before applying:
```bash
# Outputs the raw SQL to stdout without executing it against the database!
./migrate-bundle --script
```

###### Targeting a Specific Migration or Rollback:
```bash
# Migrate up to a specific migration:
./migrate-bundle AddedInvoiceTable

# Roll back to an earlier migration:
./migrate-bundle 20261010120000_InitialCreate
```

---

#### 4. Key Benefits of Migration Bundles
1. **Zero Deployment Dependencies:** Does not require the .NET SDK or `dotnet-ef` CLI tool on deployment runners.
2. **No SQL Client Tools Needed:** Executes directly via compiled ADO.NET drivers; no need to install `sqlcmd` or `psql`.
3. **Built-in Idempotency:** Automatically inspects the `__EFMigrationsHistory` table and only applies pending migrations.
4. **Ideal for Kubernetes Init Containers:** Runs inside a lightweight pod/container that exits as soon as migrations complete, before the API pod starts.
5. **Secure & Compliance-Friendly:** No source code is deployed; only compiled MSIL/native binaries.
6. **Cross-Platform:** Cross-compile for `linux-x64`, `linux-arm64`, `win-x64`, or `osx-x64`.

---

#### 5. Comprehensive Comparison: All 3 Migration Approaches

| Feature | 1. Direct CLI (`dotnet ef database update`) | 2. SQL Script (`dotnet ef migrations script -i`) | 3. Migration Bundle (`dotnet ef migrations bundle`) |
| :--- | :--- | :--- | :--- |
| **Primary Use Case** | Local Development & Rapid Prototyping | DBA Review, Gated Approvals, Legacy Pipelines | Modern CI/CD, Docker, Kubernetes, Cloud Deployments |
| **Output Artifact** | None (direct DB execution) | A `.sql` text file | A standalone executable binary (`.exe` / ELF binary) |
| **Requires .NET SDK on Runner?** | **Yes** (mandatory) | **No** | **No** |
| **Requires Source Code on Runner?** | **Yes** (compiles project on the fly) | **No** | **No** |
| **Requires SQL Client (`sqlcmd` / `psql`)?** | No | **Yes** (to execute the script) | **No** (has built-in DB drivers) |
| **DBA Reviewability** | Zero (opaque execution) | **High** (DBA can read raw SQL before running) | Moderate (run with `--script` flag to print SQL) |
| **Idempotent by Default?** | Yes | Yes (when using `-i`) | Yes |
| **Execution Speed in Pipelines** | Slow (must rebuild project) | Fast | **Fastest** (pre-compiled binary) |
| **Container / K8s Init-Container Fit** | Terrible (bloated image needed) | Moderate (requires SQL client installation) | **Best Practice / Ideal** |

##### Architectural Recommendation:
* **For Local Dev:** Use `dotnet ef database update` for instant feedback.
* **For Strict DBA / Regulated Environments (Banking, Healthcare):** Use `dotnet ef migrations script -i -o migrate.sql` so database engineers can review every line of SQL before execution.
* **For Modern Automated CI/CD (Kubernetes, AWS ECS, Azure App Service):** Use **Migration Bundles** (`dotnet ef migrations bundle`) inside a deployment job or init-container for clean, dependency-free automation.

**2. The Schema Compare Way (The DBA Approach)**
What if someone manually logged into Prod and messed with the tables, and now Prod is broken and EF Core's history is corrupted? You must step outside of EF Core and use **Database Tooling**.
You use **Visual Studio Schema Compare (SSDT)** or **Redgate SQL Compare**. You set the Source (Dev DB) and Target (Prod DB). The tool analyzes both live databases, finds the exact differences, and generates a raw `UPDATE` script to force Prod to perfectly match Dev, bypassing EF Core entirely.

---

## 7. Enterprise Migration Commands (Microservices vs Modular Monolith)

When you move away from simple single-project applications and start building Microservices or Modular Monoliths (like Normora), running EF Core migrations becomes significantly more complex. 

Because your `DbContext`, your Migration files, and your `appsettings.json` (Connection String) are split across completely different projects/folders, you **must use specific CLI flags** to tell EF Core where to look.

### The 4 Critical CLI Flags
*   `-p` or `--project`: The project where you want the migration `.cs` files to be saved (Usually the Infrastructure project).
*   `-s` or `--startup-project`: The project that actually runs and holds your `appsettings.json` and Dependency Injection setup (Usually the Web API project).
*   `-c` or `--context`: The exact name of the `DbContext` (Crucial when you have more than one context!).
*   `-o` or `--output-dir`: The specific folder inside the `--project` where you want the files placed.

#### ⚠️ A Note on Command Formatting (Line Continuations)
Because these enterprise commands get extremely long, they are often formatted across multiple lines for readability. To tell the terminal *"this command isn't finished yet, keep reading the next line,"* you must use a **line continuation character**.
*   **Linux / Mac (Bash):** Uses a backslash (`\`)
*   **Windows (PowerShell):** Uses a backtick (`` ` ``)
*   **Windows (Command Prompt):** Uses a caret (`^`)

*(Note: The examples below use the Bash `\` syntax. If you are using Windows PowerShell, replace the `\` with a backtick `` ` `` or paste the command as one single continuous line!)*

---

### Scenario A: Microservices Architecture
In a Microservice architecture, each service is entirely isolated. It has its own database, its own API, and usually its own Solution. You almost always have **exactly one DbContext** per microservice.

#### The Folder Structure
```text
/eCommerce.CatalogService (The Solution)
  ├── /src
  │    ├── /CatalogService.Api             <-- (Startup Project: holds appsettings.json)
  │    │
  │    ├── /CatalogService.Domain
  │    │
  │    ├── /CatalogService.Infrastructure  <-- (Target Project: holds CatalogDbContext)
  │         └── /Data
  │              └── CatalogDbContext.cs
```

#### The Migration Command
Because you only have one DbContext in this solution, you don't need the `--context` flag, but you *must* specify the Project and Startup Project. Execute this from the root `/eCommerce.CatalogService` folder:
```bash
dotnet ef migrations add InitialCreate \
  --project src/CatalogService.Infrastructure \
  --startup-project src/CatalogService.Api \
  --output-dir Data/Migrations
```

**What it does:** 
1. Bootstraps the `CatalogService.Api` to read the connection string.
2. Compiles `CatalogService.Infrastructure` to find the `CatalogDbContext`.
3. Generates the migration files and saves them strictly into `CatalogService.Infrastructure/Data/Migrations/`.

---

### Scenario B: Modular Monolith Architecture (Normora)
A Modular Monolith is a single massive application (one API Host, one process) that is internally divided into strictly isolated modules (Identity, Billing, Inventory). 

Because the modules are strictly isolated, **each module has its own DbContext**, even though they all run in the same API! This means you have multiple DbContexts in the same solution, so you **must** use the `--context` flag, or EF Core will throw an error saying *"More than one DbContext was found"*.

#### The Folder Structure
```text
/Normora.Solution (The Solution)
  ├── /src
  │    ├── /Normora.Host.Api               <-- (Startup Project: runs the whole app)
  │    │
  │    ├── /Modules
  │    │    ├── /Identity
  │    │    │    └── /Normora.Module.Identity.Infrastructure
  │    │    │         └── IdentityDbContext.cs
  │    │    │
  │    │    ├── /Billing
  │    │    │    └── /Normora.Module.Billing.Infrastructure
  │    │    │         └── BillingDbContext.cs
```

#### The Migration Command
If you want to add a new column to a Billing table, you must explicitly target the Billing module's infrastructure and the `BillingDbContext`. Execute this from the root `/Normora.Solution` folder:
```bash
# Generating a migration for the BILLING module
dotnet ef migrations add AddedInvoiceTable \
  --context BillingDbContext \
  --project src/Modules/Billing/Normora.Module.Billing.Infrastructure \
  --startup-project src/Normora.Host.Api \
  --output-dir Migrations

# Generating a migration for the IDENTITY module
dotnet ef migrations add AddedUserAge \
  --context IdentityDbContext \
  --project src/Modules/Identity/Normora.Module.Identity.Infrastructure \
  --startup-project src/Normora.Host.Api \
  --output-dir Migrations
```

#### Updating the Database (`dotnet ef database update`)
When applying the migrations to the database in a Modular Monolith, you also must specify which context you are updating:
```bash
# This applies only the Billing migrations!
dotnet ef database update \
  --context BillingDbContext \
  --project src/Modules/Billing/Normora.Module.Billing.Infrastructure \
  --startup-project src/Normora.Host.Api
```

*(Note: In a true Modular Monolith, even though you have multiple DbContexts, you can choose to point them all at the exact same SQL database using different DB Schemas like `[billing].[Invoices]` and `[identity].[Users]`, or you can point them to entirely different databases!)*

---

## 8. Runtime Schema Validation: The Fail-Fast Migration Guard Pattern

In enterprise environments, calling `context.Database.Migrate()` on application startup is strictly avoided to prevent race conditions, connection pool exhaustion, and privilege escalation. However, this creates a critical operational question: **What happens if a new application version is deployed before the database migration pipeline finishes?**

If the application starts up and immediately handles user traffic against an un-migrated database, queries will fail with runtime SQL exceptions (`Invalid column name`, `Invalid object name`), returning broken HTTP 500 errors to customers.

To prevent this, production applications implement the **Fail-Fast Migration Guard** (Startup Schema Validation).

### The Operational Flow

```mermaid
graph TD
    Deploy[CI/CD deploys new API pod / container] --> Startup[API Starts Up]
    Startup --> Check{Check Pending Migrations<br>GetPendingMigrationsAsync}
    Check -->|Pending Migrations Exist| FailFast[💥 Throw Exception & Stop Startup<br>Pod marks Unready / Crashes]
    Check -->|No Pending Migrations| Ready[✅ Database is Synced<br>Pod marks Ready & Receives Live Traffic]
```

---

### The Code Pattern & Roslyn Analyzer Warning

A developer might write the following startup check:

```csharp
var pendingMigrations = await context.Database.GetPendingMigrationsAsync();
if (pendingMigrations.Count() > 0)
    throw new Exception("Database is not fully migrated for Module...");
```

Notice the **green squiggly underline** under `pendingMigrations.Count() > 0`:

#### 1. Why the Green Squiggly? (IDE Code Analysis)
The green underline is a Roslyn performance analyzer warning (**CA1860** / **CA1829**):
> ⚠️ **Warning:** *"Avoid using Count() > 0 when checking for non-empty collections. Use Any() instead."*

* **The Cause:** `context.Database.GetPendingMigrationsAsync()` returns `IEnumerable<string>`. Calling `pendingMigrations.Count() > 0` iterates through the **entire** collection simply to check if it has at least one element.
* **The Fix:** `pendingMigrations.Any()` short-circuits and stops evaluating as soon as the first element is encountered. It is both more performant and idiomatic modern C#.

```csharp
// ❌ Slower / Triggers CA1860 analyzer warning:
if (pendingMigrations.Count() > 0)

// ✅ Clean, optimized & idiomatic C#:
if (pendingMigrations.Any())
```

---

### Production-Grade Implementations

Depending on your hosting infrastructure (standalone server vs. Kubernetes / Cloud), you implement this pattern in one of two ways:

#### Approach A: In Program.cs (Fail-Fast Startup Guard)
Before calling `app.Run()`, create a temporary DI scope to resolve your `DbContext` and verify that the schema is current:

```csharp
var app = builder.Build();

// Ensure the database is fully migrated before accepting traffic
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<NormoraDbContext>();
    var pendingMigrations = (await context.Database.GetPendingMigrationsAsync()).ToList();

    if (pendingMigrations.Any())
    {
        var migrationList = string.Join(", ", pendingMigrations);
        throw new InvalidOperationException(
            $"Application cannot start: Database is not fully migrated! Pending migrations: [{migrationList}]");
    }
}

app.Run();
```

---

#### Approach B: As an ASP.NET Core Readiness Health Check (Kubernetes / Cloud)
Instead of terminating the container process with an unhandled exception, integrate the check with ASP.NET Core Health Checks. This allows Kubernetes to know whether the container is ready to accept network traffic:

```csharp
public class DatabaseMigrationHealthCheck : IHealthCheck
{
    private readonly NormoraDbContext _context;

    public DatabaseMigrationHealthCheck(NormoraDbContext context) => _context = context;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        var pendingMigrations = await _context.Database.GetPendingMigrationsAsync(cancellationToken);

        if (pendingMigrations.Any())
        {
            return HealthCheckResult.Unhealthy(
                $"Database has {pendingMigrations.Count()} pending migration(s): {string.Join(", ", pendingMigrations)}");
        }

        return HealthCheckResult.Healthy("Database schema is fully synchronized.");
    }
}
```

In `Program.cs`:
```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseMigrationHealthCheck>("database_migrations", tags: ["ready"]);

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

* **Result:** While the independent CI/CD migration job is running, the pod reports `/health/ready` as **503 Unhealthy**. The load balancer routes 0% traffic to it. The moment the migration job finishes, the health check passes (`200 OK`), and traffic seamlessly flows into the pod.

---

## 9. Interview Questions

**Q1: What is the difference between `EnsureCreated()` and Migrations?**
> `dbContext.Database.EnsureCreated()` completely bypasses migrations. It looks at your C# models and instantly creates the database schema from scratch. However, if the database already exists, it does nothing—it cannot *update* an existing schema. Migrations, on the other hand, allow for incremental updates to an existing database schema over time without losing data. `EnsureCreated` is for prototyping and testing; Migrations are for production.

**Q2: You just ran `Add-Migration`, but you realize you made a typo in your C# class name. You haven't updated the database yet. How do you fix it?**
> You should delete the migration by running `Remove-Migration` (PMC) or `dotnet ef migrations remove` (CLI). Then, fix the typo in your C# class, and run `Add-Migration` again. You should *never* just delete the generated `.cs` file manually from the Solution Explorer, because EF Core also updates a `ModelSnapshot.cs` file behind the scenes, and deleting the migration file manually leaves the snapshot corrupted.

**Q3: How does EF Core know which migrations to apply when you run `Update-Database`?**
> EF Core looks at the `__EFMigrationsHistory` table in the target database. It compares the list of migrations in that table against the list of migration `.cs` files in your compiled project. Any migration file that exists in the project but is *not* listed in the history table is executed in chronological order.

**Q4: Is it a good practice for your ASP.NET Core API to call `dbContext.Database.Migrate()` automatically when the app starts up?**
> In small side-projects, yes, it is convenient. In enterprise/production environments, **absolutely not**. If you run multiple instances of your API (e.g., behind a load balancer or in Kubernetes), multiple instances will race to apply migrations simultaneously, causing connection pool exhaustion, deadlocks in `__EFMigrationsHistory`, or schema corruption.
>
> Instead, migrations must be applied independently (via Migration Bundles or CI/CD pipelines). To prevent running against an out-of-sync database, use a read-only check with `GetPendingMigrationsAsync()` or an ASP.NET Core Readiness Health Check (`DatabaseMigrationHealthCheck`).

**Q5: What is an EF Core Migration Bundle, and how does it work under the hood?**
> Introduced in EF Core 6, a Migration Bundle is a single-file, self-contained executable binary that packages your compiled migrations, the `DbContext`, the EF Core runtime, and the database ADO.NET driver. Under the hood, it executes the exact same migration logic as `dotnet ef database update`, but it does not require the .NET SDK, `dotnet-ef` CLI tool, or source code on the target machine. It can be run directly on any server or container via `./migrate-bundle --connection "..."`.

**Q6: Can a DBA or security engineer review the SQL generated by a Migration Bundle before it executes in production?**
> **Yes.** You can run the bundle with the `--script` parameter:
> ```bash
> ./migrate-bundle --script --idempotent > migration_preview.sql
> ```
> This command instructs the bundle to output the exact idempotent SQL statements it would execute to standard output without actually modifying the database. This allows DBAs to inspect, audit, and approve the migration before execution.

**Q7: How do you cross-compile a Migration Bundle on a Windows developer machine to run on a Linux production container?**
> You use the `--target-runtime` (or `-r`) flag with the target Linux Runtime Identifier (RID):
> ```bash
> dotnet ef migrations bundle --target-runtime linux-x64 --output ./migrate-bundle
> ```
> This compiles a Linux ELF binary on Windows, ready to be copied into a Linux Docker image or Kubernetes deployment runner.

**Q8: How do you deploy and run Migration Bundles in Kubernetes?**
> In Kubernetes, there are two standard enterprise patterns:
> 1. **The Init-Container Pattern:** You add an `initContainers` block to your API Pod spec using a lightweight container that runs `./migrate-bundle`. Kubernetes guarantees that the init-container must successfully exit (code 0) before the main API container starts.
> 2. **The Kubernetes Job Pattern (Recommended for Large Teams):** You define a standalone Kubernetes `Job` resource in your Helm chart or deployment manifest. The CI/CD pipeline deploys and monitors the Job. Only after the Job completes successfully does the pipeline trigger the rolling update of the Deployment pods.

**Q9: How do you securely pass database connection strings to a Migration Bundle in CI/CD without hardcoding credentials?**
> You should never hardcode connection strings in commands or commit them to source control. Migration Bundles automatically read standard ASP.NET Core environment variables:
> ```bash
> export ConnectionStrings__DefaultConnection="Server=db;Database=Prod;..."
> ./migrate-bundle
> ```
> In Kubernetes or Docker, this environment variable is mapped directly from a Kubernetes Secret or HashiCorp Vault. In GitHub Actions or Azure DevOps, it is passed via encrypted pipeline secrets.

**Q10: In a Modular Monolith with multiple DbContexts, how do you handle Migration Bundles?**
> Because each `DbContext` represents an isolated module, you generate a **separate bundle for each context** using the `--context` flag:
> ```bash
> # Billing Module Bundle
> dotnet ef migrations bundle --context BillingDbContext --output ./bundle-billing
>
> # Identity Module Bundle
> dotnet ef migrations bundle --context IdentityDbContext --output ./bundle-identity
> ```
> During deployment, your pipeline executes `./bundle-billing` and `./bundle-identity` sequentially or in parallel. Because each context has a dedicated database schema and custom `__EFMigrationsHistory` table, they never collide.

**Q11: Comparing Direct CLI, Idempotent SQL Scripts, and Migration Bundles—how do you choose between them?**
> * **Direct CLI (`dotnet ef database update`):** Choose for **Local Development**. Fast and instant iteration without file artifacts.
> * **Idempotent SQL Scripts (`dotnet ef migrations script -i`):** Choose for **Strict Enterprise / DBA-Governed Environments**. Mandated when DBAs execute scripts manually, or when compliance requires raw SQL audit trails.
> * **Migration Bundles (`dotnet ef migrations bundle`):** Choose for **Automated Cloud / Container / Kubernetes Pipelines**. Eliminates the need for .NET SDK on production nodes, requires no external SQL client tools like `sqlcmd`, and executes natively with zero overhead.

---

## Summary
Migrations act as version control for your relational database schema. By using tools like the `.NET CLI` or `Package Manager Console`, you can seamlessly generate incremental updates (`Up()`) and rollbacks (`Down()`) that keep your SQL database perfectly aligned with your C# Domain Models, enabling safe teamwork and reliable production deployments.
