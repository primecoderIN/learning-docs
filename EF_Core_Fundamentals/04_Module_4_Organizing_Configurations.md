# Module 4: Organizing Fluent API Configurations

When configuring your database models using the Fluent API, you will quickly run into a scaling problem. As your application grows from 2 tables to 50 tables, how do you manage all that configuration code?

There are two primary ways to write Fluent API configurations in EF Core. Let's explore the differences, the syntax, and why one is considered the enterprise best practice.

---

## Approach 1: Inline Configuration (`OnModelCreating`)

This is the standard approach you see in basic tutorials. All configuration is written directly inside the `OnModelCreating` method of your `DbContext`.

### The Syntax
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // You must repeatedly call modelBuilder.Entity<T>() to get the builder
    modelBuilder.Entity<Movie>()
        .ToTable("Pictures")
        .HasKey(m => m.Identifier);

    modelBuilder.Entity<Movie>()
        .Property(m => m.Title)
        .HasColumnType("varchar")
        .HasMaxLength(128)
        .IsRequired();
        
    // Imagine 50 more entities configured exactly like this...
}
```

### The Problem
If you have 50 entities, and each entity has 5 properties to configure, your `OnModelCreating` method will easily grow to **thousands of lines of code**. It becomes a massive, unreadable dumping ground that violates the Single Responsibility Principle.

---

## Approach 2: Grouped Configuration (`IEntityTypeConfiguration<T>`)

This is the **Enterprise Best Practice**. Instead of dumping everything into the `DbContext`, EF Core provides an interface called `IEntityTypeConfiguration<T>`. This allows you to extract the configuration for *one specific entity* into its own dedicated class.

### The Syntax
You create a new class that implements the interface, and EF Core injects an `EntityTypeBuilder<T>` directly into the `Configure` method.

```csharp
public class MovieConfiguration : IEntityTypeConfiguration<Movie>
{
    public void Configure(EntityTypeBuilder<Movie> builder)
    {
        // Notice we don't need modelBuilder.Entity<Movie>() anymore!
        // We just use 'builder' directly.
        
        builder.ToTable("Pictures")
               .HasKey(m => m.Identifier);

        builder.Property(m => m.Title)
               .HasColumnType("varchar")
               .HasMaxLength(128)
               .IsRequired();
               
        builder.Property(m => m.Synopsis)
               .HasColumnType("varchar(max)")
               .HasColumnName("Plot");
    }
}
```

### How to apply it in DbContext
Once you move your configurations to separate classes, you must "wire them up" inside the `OnModelCreating` method of your `DbContext`. There are two ways to do this:

#### 1. The Manual Way (`ApplyConfiguration`)
You can explicitly register each configuration class one by one. 

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new MovieConfiguration());
    modelBuilder.ApplyConfiguration(new TenantConfiguration());
    modelBuilder.ApplyConfiguration(new UserConfiguration());
    // This gets tedious if you have 50 tables...
}
```

#### 2. The Enterprise Magic Way (`ApplyConfigurationsFromAssembly`)
Instead of wiring them manually, you can tell EF Core to scan your entire project (Assembly), find *every* class that implements `IEntityTypeConfiguration`, and wire them all up automatically with just one line of code:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // This scans the assembly where TenantsDbContext lives.
    // It finds MovieConfiguration automatically and applies it!
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(TenantsDbContext).Assembly);
}
```
This second method is the ultimate best practice because when a developer adds a new table and a new configuration class, they **never have to touch `DbContext` at all**. It just works magically!

### Which one should you prefer? (The "Multiple Database Engines" edge case)

Choosing between these two methods depends entirely on how many database engines your application supports.

#### 1. If you have ONLY ONE Database Engine (e.g., SQL Server)
**Always prefer `ApplyConfigurationsFromAssembly`.**
Since every configuration in your project is meant for that one specific database, you want EF Core to blindly grab all of them and apply them. It saves time and prevents human error (like forgetting to register a file).

#### 2. If you have MORE THAN ONE Database Engine
If you are supporting multiple database engines (e.g., SQL Server and PostgreSQL) and they require *different* configurations, **the "Assembly" method can actually break your app.** If you write `MovieSqlServerConfig` and `MoviePostgresConfig`, the Assembly scanner will blindly apply both, causing conflicts.

In a multi-engine scenario, you have two choices:

**Choice A: The Manual Way (Preferred for small multi-engine apps)**
Use `if/else` statements in your `DbContext` to selectively load configurations based on the active provider.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    if (Database.IsSqlServer()) 
    {
        modelBuilder.ApplyConfiguration(new MovieSqlServerConfiguration());
        modelBuilder.ApplyConfiguration(new UserSqlServerConfiguration());
    }
    else if (Database.IsNpgsql()) // Postgres
    {
        modelBuilder.ApplyConfiguration(new MoviePostgresConfiguration());
        modelBuilder.ApplyConfiguration(new UserPostgresConfiguration());
    }
}
```

**Choice B: The Advanced "Separate Assemblies" Way (Enterprise multi-engine apps)**
Put your SQL Server configs in one Class Library (`MyProject.Data.SqlServer`) and your Postgres configs in another (`MyProject.Data.Postgres`). Then, dynamically pass the correct assembly into the magic method:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    Assembly configAssembly;

    if (Database.IsSqlServer()) 
        configAssembly = typeof(SomeSqlServerSpecificClass).Assembly;
    else 
        configAssembly = typeof(SomePostgresSpecificClass).Assembly;

    // Magically loads ALL configs, but only from the correct project!
    modelBuilder.ApplyConfigurationsFromAssembly(configAssembly);
}
```

---

## Why is Approach 2 the Best Practice?

1. **Separation of Concerns (SRP):** The `DbContext` is no longer responsible for knowing the intricate database mapping rules for every single entity. Each Configuration class has one job: configuring its specific entity.
2. **Readability:** You don't have to repeatedly type `modelBuilder.Entity<Movie>()`. The injected `builder` variable makes the syntax much cleaner.
3. **Maintainability:** If you need to change how the `Movie` table is mapped, you know exactly where to go: `MovieConfiguration.cs`. You don't have to scroll through a 3,000-line `DbContext` file.
4. **Git Merge Conflicts:** If two developers are configuring different tables, they are editing different files. In Approach 1, they would both be editing `OnModelCreating`, causing painful merge conflicts.

---

## Best Practices for Folder Structure

To keep your project organized, you should separate your pure C# domain models from your database configuration classes.

A standard enterprise folder structure looks like this:

```text
📁 MyProject.Domain
   📁 Entities                  <-- Pure C# Classes (No EF Core logic)
      📄 Movie.cs
      📄 Tenant.cs
      📄 User.cs

📁 MyProject.Infrastructure     <-- Database Layer
   📁 Data
      📄 TenantsDbContext.cs    <-- Your DbContext (Very clean now)
      📁 Configurations         <-- All your IEntityTypeConfiguration classes
         📄 MovieConfiguration.cs
         📄 TenantConfiguration.cs
         📄 UserConfiguration.cs
```

### Key Takeaways for Folder Structure:
1. **Never put Configurations in the Domain layer.** The Domain layer should be ignorant of the database. `IEntityTypeConfiguration` is an EF Core interface, so it belongs in your Data/Infrastructure layer.
2. **Match the Names:** Always name your configuration class `<EntityName>Configuration` (e.g., `MovieConfiguration`) so it is instantly discoverable.
3. **One File Per Entity:** Never group multiple entity configurations into a single file. Keep it strict: one entity, one configuration file.

---

## Migrations: Versioning Your Database Schema

EF Core Migrations are a **source-controlled, incremental history of changes** to your database schema. Every time your model changes, you generate a migration that captures the `Up` (apply) and `Down` (rollback) operations.

### Essential CLI Commands

```bash
# Install the EF Core tools globally (one-time setup)
dotnet tool install --global dotnet-ef

# Create a new migration snapshot
dotnet ef migrations add InitialCreate --project src/Infrastructure --startup-project src/Api

# Apply all pending migrations to the database
dotnet ef database update

# Roll back to a specific migration
dotnet ef database update PreviousMigrationName

# Generate a SQL script instead of directly applying (useful for DBA review)
dotnet ef migrations script --output migration.sql
```

### Understanding Migration Files

When you run `dotnet ef migrations add`, EF Core generates a file like `20241004_AddTenantSlug.cs`:

```csharp
public partial class AddTenantSlug : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Applied when running 'dotnet ef database update'
        migrationBuilder.AddColumn<string>(
            name: "Slug",
            table: "SystemTenants",
            type: "varchar(63)",
            nullable: false,
            defaultValue: "");

        migrationBuilder.CreateIndex(
            name: "IX_SystemTenants_Slug",
            table: "SystemTenants",
            column: "Slug",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        // Applied when rolling back
        migrationBuilder.DropIndex(name: "IX_SystemTenants_Slug", table: "SystemTenants");
        migrationBuilder.DropColumn(name: "Slug", table: "SystemTenants");
    }
}
```

### Applying Migrations Automatically on Startup
In some scenarios (e.g., Docker containers, CI/CD pipelines), you want migrations to run automatically when the app starts:

```csharp
// Program.cs
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<TenantsDbContext>();
    await context.Database.MigrateAsync(); // Applies any pending migrations
}
```

> **Warning:** Be careful with `MigrateAsync()` in load-balanced environments. If multiple instances start simultaneously, they may race to apply the same migration. Use a distributed lock or a dedicated migration job in production.

---

## Real-World Scenarios

### Scenario A: Normora's Modular Configuration Setup

In Normora's `TenantsDbContext`, we use `ApplyConfigurationsFromAssembly` to auto-discover all configurations in the Infrastructure assembly:

```csharp
public class TenantsDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // One line to rule them all — scans and applies all IEntityTypeConfiguration classes
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(TenantsDbContext).Assembly);

        // We still manually apply cross-cutting concerns like shadow properties
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            modelBuilder.Entity(entityType.ClrType).Property<DateTime>("CreatedAt");
            modelBuilder.Entity(entityType.ClrType).Property<bool>("IsDeleted").HasDefaultValue(false);
        }
    }
}
```

### Scenario B: A Data Migration (Populating a New Column)
When you add a `Slug` column to `Tenants` and need to populate it from existing `Name` data:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(name: "Slug", table: "SystemTenants", nullable: true);

    // Data migration: populate from existing data
    migrationBuilder.Sql(@"
        UPDATE SystemTenants 
        SET Slug = LOWER(REPLACE(Name, ' ', '-'))
        WHERE Slug IS NULL
    ");

    // Now make it non-nullable after populating
    migrationBuilder.AlterColumn<string>(name: "Slug", table: "SystemTenants", nullable: false);
}
```

---

## Interview Questions

**Q1: What is `IEntityTypeConfiguration<T>` and why is it better than configuring everything in `OnModelCreating`?**
> `IEntityTypeConfiguration<T>` is an interface that moves entity configuration into its own dedicated class. It's better because: (1) it follows the Single Responsibility Principle, (2) the `DbContext` stays clean and focused, (3) it reduces Git merge conflicts when multiple developers work on different entities, (4) it's instantly discoverable — `TenantConfiguration.cs` is obviously the right place to look for `Tenant` configuration.

**Q2: What does `ApplyConfigurationsFromAssembly` do? What's the risk in multi-database scenarios?**
> It scans the given assembly using reflection, finds all classes implementing `IEntityTypeConfiguration<T>`, and automatically calls `ApplyConfiguration` for each one. The risk in multi-database scenarios is that if you have separate configuration classes for SQL Server and PostgreSQL in the same assembly, the scanner will blindly apply both, causing conflicts. The solution is to either use manual registration with `if/else` or separate assemblies per provider.

**Q3: What is a database migration, and why is it superior to `EnsureCreated()`?**
> A migration is a versioned, source-controlled snapshot of a schema change. Each migration has an `Up` (apply) and `Down` (rollback) method. Unlike `EnsureCreated()` (which can only create the schema from scratch), migrations support incremental changes — adding columns, renaming tables, adding indexes — without destroying existing data. They're checked into source control, meaning every environment can be brought to the same state reliably.

**Q4: What is a data migration, and when is it needed?**
> A data migration is custom SQL inside a migration's `Up()` method that transforms existing data as part of a schema change. It's needed when you add a new required column that must be populated from existing data (e.g., generating a `Slug` from a `Name`). Without a data migration, the `NOT NULL` constraint on the new column would cause the migration to fail on existing rows.

**Q5: How would you handle migrations in a zero-downtime deployment?**
> The key is to ensure migrations are backward-compatible: (1) Never drop a column or rename a column in the same migration that stops using it — keep the old column for one deployment cycle. (2) Add new columns as `nullable` first, then populate via a data migration, then add the `NOT NULL` constraint in a subsequent migration. (3) Use the "expand/contract" pattern: expand the schema (add new things), deploy the code, then contract (remove old things). This ensures the old and new app versions can coexist against the same schema.

---

## Summary
In this module, we learned to use `IEntityTypeConfiguration<T>` to keep configuration organized and scalable, `ApplyConfigurationsFromAssembly` for zero-touch auto-discovery, and how migrations provide a safe, versioned, source-controlled way to evolve your database schema. In the next module, we will model the relationships between our entities.
