# Chapter 38: Zero-Downtime Migrations

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand why the traditional "Weekend Maintenance Window" is no longer acceptable in global SaaS applications.
*   Identify the locking and breaking risks of standard Entity Framework Core migrations.
*   Architect and execute the **Expand-and-Contract Pattern** to rename columns or split tables without any application downtime.
*   Implement Blue/Green application deployments safely against a mutating database schema.
*   Generate Idempotent Migration Scripts for secure, reliable CI/CD execution.

---

## 38.1 The "Maintenance Window" is Dead

In the 2010s, if you needed to rename a column from `ZipCode` to `PostalCode`, you sent an email to all your customers: *"The system will be down on Sunday from 2:00 AM to 4:00 AM for scheduled maintenance."*
During that window, you took the API offline, ran the EF Core migration script, deployed the new API code, and brought the system back online.

In a global SaaS application, there is no "2:00 AM." When it is 2:00 AM in New York, it is 8:00 AM in Berlin, and 4:00 PM in Tokyo. If your EV charging platform goes offline for 5 minutes, cars cannot charge, and your company loses money and trust.

Architects must design deployments that achieve **Zero Downtime**.

---

## 38.2 Why Standard Migrations Cause Downtime

Let's look at the standard EF Core migration for renaming a column:
`EXEC sp_rename 'core.Stations.ZipCode', 'PostalCode', 'COLUMN';`

If you run this script *before* the new API is deployed, the old API (currently live in production) will try to execute `SELECT ZipCode FROM core.Stations`. It will crash with a "Invalid column name" error.
If you deploy the new API *before* the script runs, the new API will try to execute `SELECT PostalCode`. It will crash.

Standard schema modifications are inherently breaking changes. You cannot deploy the code and the database at the exact same millisecond. Therefore, standard migrations guarantee downtime.

---

## 38.3 The Expand-and-Contract Pattern

To achieve zero downtime, we must decouple the database deployment from the application deployment. We do this using the **Expand-and-Contract** pattern, executed over multiple independent CI/CD deployments.

### Phase 1: Expand (Database Deployment)
We do not rename the old column. We *expand* the schema by adding the new column alongside it.
```sql
ALTER TABLE core.Stations ADD PostalCode VARCHAR(20) NULL;
```
*Result:* The old API is still running perfectly. It doesn't know `PostalCode` exists.

### Phase 2: Migrate Data (Database Deployment)
We must move the data from `ZipCode` to `PostalCode`.
If the table has 100 million rows, we do not run a single `UPDATE` statement (which would lock the table). We run a background SQL Agent job that batches the update in chunks of 5,000 rows.
To ensure new inserts are caught, we add a temporary SQL Trigger:
```sql
CREATE TRIGGER trg_SyncZipCode ON core.Stations AFTER INSERT, UPDATE AS
BEGIN
    UPDATE core.Stations SET PostalCode = inserted.ZipCode
    FROM core.Stations JOIN inserted ON core.Stations.Id = inserted.Id
END
```

### Phase 3: Switch Application (Code Deployment)
Now that both columns exist and the data is synced, we execute a **Blue/Green Deployment**.
We spin up the new API (Green) which reads and writes exclusively to `PostalCode`. The load balancer slowly shifts traffic from Old API (Blue) to Green.
*Result:* Zero downtime. Users transition seamlessly.

### Phase 4: Contract (Database Deployment)
Days or weeks later, when we are 100% sure the new API is stable and we will not need to rollback, we clean up the database.
```sql
DROP TRIGGER trg_SyncZipCode;
ALTER TABLE core.Stations DROP COLUMN ZipCode;
```

---

## 38.4 Architect Perspective: EF Core in CI/CD

Many developers wire up their application startup code (in `Program.cs`) to execute `_context.Database.Migrate()`.
**Architect Rule: Never execute EF Core migrations on application startup in a production environment.**

If you have 10 API pods spinning up simultaneously in Kubernetes during a scale-out event, they will all attempt to run the migration concurrently. They will clash, causing Deadlocks and corrupting the `__EFMigrationsHistory` table.

### The CI/CD Standard: Idempotent Scripts
Migrations must be executed by your CI/CD pipeline (e.g., GitHub Actions) *before* the application pods are updated.
Furthermore, the CI/CD pipeline should not use the `dotnet ef database update` command directly against production. It should generate an **Idempotent SQL Script**.

```bash
# Generate a script that checks if a migration has already been applied before executing it
dotnet ef migrations script --idempotent --output ./migrations.sql
```

Your pipeline then takes `migrations.sql`, hands it to a secure DBA tool (like DbUp, Flyway, or Azure DevOps SQL Task), which executes it safely against the database.

---

## 38.5 The Code: Handling Breaking Changes in EF Core

When using the Expand-and-Contract pattern, EF Core can be frustrating because it tries to map everything.
During Phase 1 (Expand), your C# Entity must have *both* properties mapped, so that EF Core can read from the old and write to the new.

```csharp
public class Station
{
    public Guid Id { get; set; }
    
    // The Old Column (Keep it for reading during the transition)
    [Obsolete("Use PostalCode instead. Scheduled for removal in Sprint 42.")]
    public string ZipCode { get; set; }
    
    // The New Column
    public string PostalCode { get; set; }
}
```
You must carefully orchestrate your EF Core LINQ queries to transition smoothly between these properties across your Sprint deployments.

---

## 38.6 Performance & Security Analysis

### Performance Analysis: Online Index Operations
During Phase 4 (Contract), when you drop a column, SQL Server must modify the table metadata. If you need to drop an Index associated with that old column, or create a new index for the new column, do not use standard `CREATE/DROP INDEX` commands on massive tables. They take exclusive locks. Always use the `WITH (ONLINE = ON)` flag (available in Enterprise Edition/Azure SQL) to build indexes in the background without blocking concurrent API traffic.

### Security Implications
*   **Pipeline Credentials:** Generating the SQL script in CI/CD is safe. But the pipeline runner that *executes* the script against production needs `ALTER` permissions on the database. If your GitHub Actions pipeline is compromised, the attacker can drop your entire database. Ensure the service principal used by the pipeline has strict conditional access policies, network isolation (VNet integration), and absolutely no access to the application data (only schema alteration).

---

## 38.7 Common Mistakes & Production Pitfalls

1.  **Adding a NOT NULL Column without a Default:** If you run `ALTER TABLE Users ADD Age INT NOT NULL;` on a table with 1 million rows, SQL Server will immediately reject it because the existing 1 million rows do not have an Age. You must either add it as `NULL` first (and backfill the data), or supply a `DEFAULT` constraint: `ADD Age INT NOT NULL DEFAULT 0;`.
2.  **Rolling Back Code, but not DB:** You deploy a new API and a new database migration. The API has a critical bug. You instantly roll back the API via Kubernetes. But the old API crashes because the database schema was already changed! *Always make your database migrations backwards-compatible (Expand-and-Contract).*

---

## 38.8 Production Checklist

*   [ ] EF Core `_context.Database.Migrate()` is strictly removed from the ASP.NET Core startup pipeline.
*   [ ] CI/CD pipelines generate Idempotent SQL scripts to execute schema changes.
*   [ ] Destructive schema changes (Renames, Drops, Type changes) are broken down into multi-phase Expand-and-Contract deployments.
*   [ ] `ONLINE = ON` is enforced for all Index creation and rebuild operations.

---

## 38.9 Exercises

1.  **The Drop Column Disaster:** A developer merges a PR that deletes the `Description` property from a C# Entity. CI/CD runs the EF Core migration, executing `ALTER TABLE Stations DROP COLUMN Description`. The Blue/Green deployment is currently shifting traffic; 50% of users are on the new API, 50% are on the old API. Exactly what will happen to the users on the old API, and how should this deployment have been structured?
2.  **Idempotency:** Open the `migrations.sql` file generated by EF Core with the `--idempotent` flag. What specific `IF` statement surrounds every block of `CREATE TABLE` or `ALTER TABLE` code, and why is this critical for CI/CD reruns?

---

## 38.10 Interview Questions

**Q1: Why is executing `_context.Database.Migrate()` on application startup a critical anti-pattern in modern cloud deployments?**
*Answer:* In modern cloud environments (Kubernetes, Azure App Service), scaling out means multiple instances (pods) of the application start up simultaneously. If they all run `Migrate()` at the same time, they will clash, causing race conditions, deadlocks, and potential corruption of the schema or the EF Core migrations history table. Migrations must be executed sequentially by a dedicated CI/CD pipeline step prior to the application code deployment.

**Q2: Explain the Expand-and-Contract pattern and how it achieves Zero-Downtime database migrations.**
*Answer:* Traditional database migrations (like dropping or renaming a column) are instantly breaking changes for the currently running application. Expand-and-Contract solves this by decoupling the change into phases. First, you "Expand" the database by adding the new schema (e.g., adding a new column alongside the old one). This is backwards-compatible. Second, you migrate the data. Third, you switch the application code to use the new column via a seamless Blue/Green deployment. Finally, weeks later, you "Contract" the database by dropping the old, unused column. This guarantees the database is always compatible with whatever version of the API is currently handling traffic.

---
**Next up in Chapter 39:** We will explore Multi-Tenant SaaS patterns from an operational perspective, comparing the tradeoffs of Shared Schema, Shared Database, and Isolated Database (Database-per-Tenant) models.
