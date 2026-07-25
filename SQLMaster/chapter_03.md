# Chapter 3: Constraints & Data Integrity

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand the "Defense in Depth" philosophy and why application-layer validation is insufficient for enterprise data integrity.
*   Implement Foreign Keys correctly, understanding why `CASCADE DELETE` is often a disaster in SaaS environments.
*   Design composite `UNIQUE` constraints to enforce multi-tenant boundaries (e.g., ensuring a username is unique only *within* a tenant).
*   Utilize `CHECK` constraints to enforce business domain rules (like preventing negative energy consumption) at the disk level.

---

## 3.1 The "Defense in Depth" Philosophy

A common anti-pattern among full-stack developers is relying entirely on the application layer (e.g., C# FluentValidation, React forms) to enforce data integrity. The rationale is usually: *"My API validates the data before inserting it, so the database doesn't need to."*

**Architect Perspective:** Applications are ephemeral; data is permanent. Over a 10-year lifespan, your SaaS will likely have multiple APIs, background workers, legacy migration scripts, and direct DBA interventions writing to the database. If your data integrity rules only exist in the v1 REST API, a background worker bypassing that API will instantly corrupt your database. 

Constraints are the ultimate firewall. They ensure that no matter who or what is writing to the database, the physical data adheres to business reality.

---

## 3.2 Primary Keys and Foreign Keys (Referential Integrity)

### Primary Keys (PK)
As discussed in Chapter 2, every table needs a Primary Key to uniquely identify a row. In SQL Server, the PK constraint automatically creates a Unique Clustered Index (by default).

### Foreign Keys (FK)
Foreign Keys enforce **Referential Integrity**. In our EV SaaS, a `Session` cannot exist without a valid `UserId` and `PortId`. 

```sql
ALTER TABLE core.Sessions
ADD CONSTRAINT FK_Sessions_Users 
FOREIGN KEY (UserId) REFERENCES core.Users(UserId);
```

#### The `CASCADE DELETE` Trap
When defining an FK, you can specify what happens when the parent record is deleted.
*   `ON DELETE NO ACTION` (Default): The engine throws an error and rolls back the transaction if you try to delete a User who has Sessions.
*   `ON DELETE CASCADE`: The engine automatically deletes all child Sessions when the User is deleted.

**Production Pitfall:** In a multi-tenant SaaS, **never use Cascade Deletes** for business entities. If a disgruntled Admin deletes a `Tenant` record, a cascade delete will recursively wipe out millions of Stations, Ports, Users, and billing Sessions in milliseconds, permanently destroying financial records. 
Instead, we use `NO ACTION` (Restrict) combined with **Soft Deletes** (discussed in Chapter 24).

---

## 3.3 UNIQUE Constraints and Multi-Tenant Boundaries

A `UNIQUE` constraint ensures that no two rows have the same value in a specific column (or combination of columns). It automatically creates a Unique Non-Clustered Index behind the scenes.

### The Single-Column Unique Constraint
For a global system, we want to ensure no two users register the same RFID card.
```sql
ALTER TABLE core.Users
ADD CONSTRAINT UQ_Users_RfidTag UNIQUE (RfidTag);
```

### The Composite (Multi-Column) Unique Constraint
In a multi-tenant SaaS, uniqueness often depends on the Tenant. 
For example, a Tenant might name their charging station "Lobby Charger". Another Tenant might also use the name "Lobby Charger". This is fine. But a *single* Tenant cannot have two stations with the same name.

We enforce this boundary using a composite unique constraint:
```sql
ALTER TABLE core.Stations
ADD CONSTRAINT UQ_Stations_Tenant_Name UNIQUE (TenantId, Name);
```
By including `TenantId` in the constraint, we isolate the business rule to the customer's specific dataset.

---

## 3.4 CHECK Constraints: The Architect's Secret Weapon

A `CHECK` constraint evaluates a Boolean expression for every `INSERT` or `UPDATE`. If the expression evaluates to `FALSE`, the transaction is rejected. This is where we enforce core domain logic.

### Rule 1: No Free Energy (Negative kWh)
A hardware glitch on an OCPP charging station might cause it to report `-500 kWh`. If this hits our billing engine, we would end up owing the customer money.

```sql
ALTER TABLE core.Sessions
ADD CONSTRAINT CHK_Sessions_Kwh_Positive 
CHECK (TotalKwh >= 0);
```

### Rule 2: Time Travel Prevention
A charging session must end *after* it starts.
```sql
ALTER TABLE core.Sessions
ADD CONSTRAINT CHK_Sessions_Timeline 
CHECK (EndTime IS NULL OR EndTime >= StartTime);
```

### Rule 3: Validating Enums
While lookup tables are better for dynamic data, fixed state machines (like Session Status) can be enforced via CHECK constraints to prevent garbage data insertion.

```sql
ALTER TABLE core.Sessions
ADD CONSTRAINT CHK_Sessions_Status 
CHECK (Status IN ('Charging', 'Completed', 'Faulted'));
```

---

## 3.5 The Code: EF Core Implementations

To implement these constraints in EF Core, we continue using the Fluent API.

```csharp
// Persistence/Configurations/SessionConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class SessionConfiguration : IEntityTypeConfiguration<Session>
{
    public void Configure(EntityTypeBuilder<Session> builder)
    {
        // 1. Foreign Keys (Restrict Delete)
        builder.HasOne(s => s.User)
               .WithMany(u => u.Sessions)
               .HasForeignKey(s => s.UserId)
               .OnDelete(DeleteBehavior.Restrict); // NEVER use Cascade

        // 2. CHECK Constraints
        builder.HasCheckConstraint(
            "CHK_Sessions_Kwh_Positive", 
            "[TotalKwh] >= 0");
            
        builder.HasCheckConstraint(
            "CHK_Sessions_Timeline", 
            "[EndTime] IS NULL OR [EndTime] >= [StartTime]");
    }
}

// Persistence/Configurations/StationConfiguration.cs
public class StationConfiguration : IEntityTypeConfiguration<Station>
{
    public void Configure(EntityTypeBuilder<Station> builder)
    {
        // 3. Composite Unique Constraint for Multi-Tenancy
        builder.HasIndex(s => new { s.TenantId, s.Name })
               .IsUnique()
               .HasDatabaseName("UQ_Stations_Tenant_Name");
    }
}
```

---

## 3.6 Performance & Security Analysis

### Performance Analysis
*   **Foreign Key Indexing:** SQL Server does **not** automatically index Foreign Key columns. When a row in the parent table (`Users`) is deleted or updated, SQL Server must check the child table (`Sessions`) to ensure referential integrity. If `Sessions.UserId` is not indexed, this results in a full table scan of `Sessions`, causing massive locks and timeouts. **Always manually create Non-Clustered Indexes on your FK columns.**
*   **Constraint Overhead:** CHECK constraints add a minuscule CPU overhead during writes. This overhead is statistically insignificant compared to the cost of fixing corrupted billing data.

### Security Implications
*   **Information Disclosure:** When a unique constraint violation occurs, the database throws an error (e.g., Error 2627). If your API does not catch this exception, it might return a 500 Internal Server Error to the client containing the raw SQL error message, revealing database schema details to attackers. Always catch `DbUpdateException` in EF Core and return a clean `409 Conflict`.

---

## 3.7 Common Mistakes & Production Pitfalls

1.  **Disabling Constraints for Bulk Inserts:** DBAs sometimes disable constraints (`NOCHECK`) to speed up bulk data imports. If they forget to re-enable them `WITH CHECK`, the constraint becomes "Untrusted." The query optimizer will no longer use untrusted constraints to build better execution plans.
2.  **Soft Deletes Breaking Unique Constraints:** If you use a composite unique constraint on `(TenantId, Name)`, and a user deletes the station "Lobby Charger" (setting `IsDeleted = 1`), they will not be able to create a *new* station named "Lobby Charger" because the old one still exists in the unique index. (We will solve this using Filtered Indexes in Chapter 19).

---

## 3.8 Production Checklist

*   [ ] All Foreign Keys are configured to `ON DELETE NO ACTION` (Restrict) to prevent accidental cascading data loss.
*   [ ] Every Foreign Key column has a dedicated Non-Clustered Index to prevent table scans during parent updates.
*   [ ] Business rules that dictate absolute reality (e.g., values cannot be negative) are enforced via `CHECK` constraints.
*   [ ] Multi-tenant uniqueness is enforced by including `TenantId` in composite `UNIQUE` constraints.

---

## 3.9 Exercises

1.  **Constraint Design:** The business adds a `DiscountPercentage` column to the `Tenant` table to offer wholesale pricing. Write the T-SQL `ALTER TABLE` statement to ensure the discount is between 0.00 and 100.00.
2.  **EF Core Translation:** Write the C# EF Core Fluent API configuration for the `DiscountPercentage` check constraint you designed in Exercise 1.

---

## 3.10 Interview Questions

**Q1: What is the difference between a Primary Key and a Unique Constraint?**
*Answer:* Both enforce uniqueness. However, a table can only have one Primary Key, and PK columns cannot contain `NULL` values. A table can have multiple Unique Constraints, and in SQL Server, a Unique Constraint allows exactly one `NULL` value (unless specifically designed as a filtered index). Additionally, a PK defaults to creating a Clustered Index, while a Unique Constraint defaults to a Non-Clustered Index.

**Q2: Why is it critical to create indexes on Foreign Key columns, even if you never use them in a `WHERE` clause for a `SELECT` statement?**
*Answer:* Indexes on FKs are required for efficient `UPDATE` and `DELETE` operations on the parent table. When you attempt to delete a parent record, SQL Server must check the child table to enforce referential integrity. Without an index on the child's FK column, the engine must perform a full table scan on the child table, which is extremely slow and causes severe locking issues.

---
**Next up in Chapter 4:** We will explore the Core CRUD Operations, analyzing how the engine processes `INSERT`, `UPDATE`, and `DELETE` commands, and how EF Core translates our C# objects into these statements.
