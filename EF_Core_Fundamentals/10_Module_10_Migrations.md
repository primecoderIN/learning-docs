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

### The `__EFMigrationsHistory` Table
How does EF Core know which migrations have already been applied to your database? 
The very first time you run `Update-Database`, EF Core creates a special hidden table in your SQL Server called `__EFMigrationsHistory`. Every time it successfully runs an `Up()` method, it inserts the name of the migration into this table. The next time you run `Update-Database`, it checks this table to ensure it doesn't accidentally run the same migration twice!

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
If your Dev database is simply 3 migrations ahead of Prod, you don't actually need to "compare" them. Because EF Core tracks history in the `__EFMigrationsHistory` table, all you do is run: `dotnet ef migrations script`. 
This generates a massive SQL script. When you run this script on the Prod database, it checks the history table, automatically sees that Prod is missing the last 3 migrations, and perfectly syncs Prod up to Dev's level.

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

## 8. Interview Questions

**Q1: What is the difference between `EnsureCreated()` and Migrations?**
> `dbContext.Database.EnsureCreated()` completely bypasses migrations. It looks at your C# models and instantly creates the database schema from scratch. However, if the database already exists, it does nothing—it cannot *update* an existing schema. Migrations, on the other hand, allow for incremental updates to an existing database schema over time without losing data. `EnsureCreated` is for prototyping and testing; Migrations are for production.

**Q2: You just ran `Add-Migration`, but you realize you made a typo in your C# class name. You haven't updated the database yet. How do you fix it?**
> You should delete the migration by running `Remove-Migration` (PMC) or `dotnet ef migrations remove` (CLI). Then, fix the typo in your C# class, and run `Add-Migration` again. You should *never* just delete the generated `.cs` file manually from the Solution Explorer, because EF Core also updates a `ModelSnapshot.cs` file behind the scenes, and deleting the migration file manually leaves the snapshot corrupted.

**Q3: How does EF Core know which migrations to apply when you run `Update-Database`?**
> EF Core looks at the `__EFMigrationsHistory` table in the target database. It compares the list of migrations in that table against the list of migration `.cs` files in your compiled project. Any migration file that exists in the project but is *not* listed in the history table is executed in chronological order.

**Q4: Is it a good practice for your ASP.NET Core API to call `dbContext.Database.Migrate()` automatically when the app starts up?**
> In small side-projects, yes, it is convenient. In enterprise/production environments, **absolutely not**. If you run multiple instances of your API (e.g., behind a load balancer or in Kubernetes), multiple instances might try to run the migration at the exact same time, causing deadlocks, database corruption, or race conditions. In enterprise systems, migrations should be applied via generated SQL scripts executed by a DBA, or as a strict, isolated step in an automated CI/CD pipeline.

---

## Summary
Migrations act as version control for your relational database schema. By using tools like the `.NET CLI` or `Package Manager Console`, you can seamlessly generate incremental updates (`Up()`) and rollbacks (`Down()`) that keep your SQL database perfectly aligned with your C# Domain Models, enabling safe teamwork and reliable production deployments.
