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
