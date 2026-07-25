# Chapter 39: Multi-Tenant Architecture

## Learning Objectives
By the end of this chapter, you will be able to:
*   Define the three primary architectural models for Multi-Tenant SaaS databases.
*   Evaluate the security, scalability, and cost trade-offs of Shared vs. Isolated databases.
*   Understand how **Azure Elastic Pools** solve the massive cost problem of Database-per-Tenant models.
*   Implement dynamic connection string routing in EF Core to support an Isolated Database architecture.

---

## 39.1 The Spectrum of Isolation

When building a global SaaS platform like NextEvent or a multi-tenant EV charging network, your most critical architectural decision is how to store data for different customers (Tenants). 
If Acme Corp and Bob's Coffee both use your platform, how do you keep their data separate?

There is no "perfect" answer. You must choose from a spectrum that balances **Cost & Complexity** against **Security & Scale**.

---

## 39.2 Model 1: Shared Database, Shared Schema

This is the model we have used throughout the book.
Every tenant's data lives in the exact same `core.Stations` table. The only thing separating Acme Corp from Bob's Coffee is a `TenantId` column.

*   **Pros:** 
    *   *Cost:* You only pay for one SQL Server. It is incredibly cheap.
    *   *Maintenance:* When you add a new column, you deploy the migration once. 
*   **Cons:** 
    *   *Security:* Massive risk of "Tenant Data Bleed" (Chapter 27). You must rely on RLS and EF Core Global Query Filters perfectly.
    *   *Noisy Neighbor:* If Bob runs a terrible query, it spikes the CPU for the entire server, taking Acme Corp offline too.
    *   *Restore limits:* If Bob accidentally deletes all his data, you cannot restore a database backup. Doing so would overwrite Acme Corp's data with yesterday's backup. You must write a custom Point-in-Time data extraction script.

---

## 39.3 Model 2: Shared Database, Isolated Schema

In this model, all tenants share the same physical database engine, but you create separate schemas for each.
`acme.Stations` and `bob.Stations`.

*   **Pros:** 
    *   *Security:* Better logical isolation. Data bleed is harder to accidentally trigger in code.
*   **Cons:** 
    *   *The Worst of Both Worlds:* You still have the Noisy Neighbor problem (shared CPU/RAM), but now your EF Core migrations are an absolute nightmare. EF Core strongly prefers one schema. You have to write custom DDL generation logic to loop through 50 schemas every time you deploy a new feature.

*Architect Rule:* Avoid this model. Either share the schema, or completely isolate the databases.

---

## 39.4 Model 3: Isolated Database-per-Tenant

This is the Enterprise Standard. 
Acme Corp gets their own physical database (`DB_Acme`). Bob's Coffee gets their own database (`DB_Bob`).

*   **Pros:** 
    *   *Ultimate Security:* It is mathematically impossible for data to bleed. 
    *   *Noisy Neighbor solved:* Bob's terrible query only locks up Bob's database. Acme Corp is unaffected.
    *   *Point-in-Time Restore:* If Bob deletes his data, you just click "Restore Backup" on `DB_Bob` to 5 minutes ago.
    *   *Data Residency:* Acme can request their database be hosted in Germany, while Bob's is hosted in New York.
*   **Cons:** 
    *   *Cost:* If you have 5,000 tenants, you must provision and pay for 5,000 SQL Databases.
    *   *Migrations:* A CI/CD deployment means running a database schema migration script 5,000 times concurrently.

---

## 39.5 Architect Perspective: Azure Elastic Pools

The biggest barrier to the Database-per-Tenant model is cost. If a standard Azure SQL database costs $15/month, and you have 10,000 tenants, you are paying $150,000/month just for idle compute (because most tenants only log in once a week).

**Azure Elastic Pools** solve this entirely.
You provision a massive pool of CPU and RAM (e.g., a 40 vCore Pool for $4,000/month). You then deploy all 10,000 databases *into that single pool*. 
The 10,000 databases share the 40 vCores dynamically. Because it's statistically impossible for all 10,000 tenants to spike their CPU at the exact same millisecond, the Elastic Pool absorbs the peaks and valleys seamlessly. You get full physical isolation at the cost of a shared server.

---

## 39.6 The Code: Dynamic DbContext Routing

If you choose the Database-per-Tenant model, you cannot hardcode the Connection String in `appsettings.json`. The API must figure out which database to talk to on every HTTP request.

You use a central **Tenant Map** (similar to the Shard Map in Chapter 29) and a custom EF Core Connection Interceptor.

```csharp
// 1. A service that extracts the TenantId from the JWT Token
public interface ITenantResolver { Guid GetCurrentTenantId(); }

// 2. A factory that dynamically builds the DbContext
public class MultiTenantDbContextFactory
{
    private readonly ITenantResolver _resolver;
    private readonly ITenantCatalogDatabase _catalog; // The central lookup DB

    public async Task<VoltCoreDbContext> CreateContextAsync()
    {
        var tenantId = _resolver.GetCurrentTenantId();
        
        // Lookup the specific database connection string for this tenant
        string connectionString = await _catalog.GetConnectionStringForTenant(tenantId);

        var options = new DbContextOptionsBuilder<VoltCoreDbContext>()
            .UseSqlServer(connectionString)
            .Options;

        return new VoltCoreDbContext(options);
    }
}
```
*Note: In ASP.NET Core, you register this factory as a Scoped service, so every HTTP request gets a perfectly isolated `DbContext` pointing directly to their private database.*

---

## 39.7 Performance & Security Analysis

### Performance Analysis: Cross-Tenant Reporting
The primary performance flaw of the Database-per-Tenant model is global reporting. If the CEO wants a report of "Total Revenue Across All Tenants", you cannot write a simple SQL `SELECT SUM()` query. You must execute a Fan-Out query against 10,000 databases, which will crash your application. 
*The Architect's Fix:* You must implement a CQRS architecture (Chapter 34) where all 10,000 databases asynchronously publish their data (via CDC) to a single, centralized Data Warehouse or Data Lake (e.g., Azure Synapse / Snowflake) designed exclusively for global reporting.

### Security Implications
*   **Tenant Catalog Spoofer:** The `ITenantCatalogDatabase` is the keys to the kingdom. If a malicious user manages to trick the `ITenantResolver` into providing Acme Corp's `TenantId` (e.g., by tampering with the JWT if the signing key is weak), the factory will gladly connect them to Acme's database. JWT signing keys and the Catalog database must be secured with the highest possible RBAC and encryption standards.

---

## 39.8 Common Mistakes & Production Pitfalls

1.  **Shared Master Data:** In a Database-per-Tenant model, developers often duplicate lookup tables (e.g., `CountryCodes`, `CurrencyRates`) into every single tenant's database. When a new country is added, you have to run an `INSERT` script across 10,000 databases. Keep globally shared reference data in a central "Master" database, and cache it heavily in the API's RAM.
2.  **Migration Failures in the Middle:** When deploying a schema update to 10,000 databases, what happens if database #5,432 fails due to a locked table? Half your customers are on Schema V1, half are on Schema V2. Your API code must be robust enough (Expand-and-Contract, Chapter 38) to handle both schemas seamlessly while the DBA team fixes the broken database.

---

## 39.9 Production Checklist

*   [ ] The architectural model (Shared vs Isolated) is chosen before Day 1 of development; migrating from Shared to Isolated later takes thousands of engineering hours.
*   [ ] If using Database-per-Tenant, Azure Elastic Pools (or AWS Aurora Serverless) are utilized to control idle compute costs.
*   [ ] EF Core is configured with dynamic connection string resolution based on a secure, authenticated Tenant ID.
*   [ ] Global cross-tenant reporting is offloaded to a separate, centralized Data Warehouse.

---

## 39.10 Exercises

1.  **Architectural Choice:** You are building a SaaS for Hospitals. HIPAA regulations state that Patient Data for Hospital A must be encrypted with a customer-managed key (CMK) that Hospital A controls, and Hospital A can revoke that key at any time, instantly destroying access to their data. Which multi-tenant model must you choose, and why do the other two models fail?
2.  **Data Recovery:** In a Shared Database (Model 1), a customer support agent accidentally deletes all 5,000 charging sessions for Tenant B. You have a full database backup from 1 hour ago. Explain exactly why you cannot simply run `RESTORE DATABASE VoltCore FROM DISK` to fix the problem.

---

## 39.11 Interview Questions

**Q1: Compare the "Shared Database, Shared Schema" model with the "Database-per-Tenant" model regarding the "Noisy Neighbor" problem and Disaster Recovery.**
*Answer:* In the Shared model, all tenants share physical resources (CPU/RAM). A single massive query from one tenant (the Noisy Neighbor) can saturate the server, degrading performance for all other tenants. Furthermore, Disaster Recovery is highly complex; if one tenant corrupts their data, you cannot restore the shared database backup without overwriting the good data of all other tenants. The Database-per-Tenant model solves both: physical resource isolation prevents the noisy neighbor problem (especially when combined with Elastic Pools resource limits), and you can restore a single tenant's database to a specific point-in-time without impacting anyone else.

**Q2: What is an Azure Elastic Pool, and why is it critical for the financial viability of a Database-per-Tenant architecture?**
*Answer:* An Azure Elastic Pool is a shared pool of CPU, memory, and I/O resources that hosts multiple independent databases. In a Database-per-Tenant architecture, most databases are idle 90% of the day. Paying for dedicated compute for thousands of idle databases is financially ruinous. An Elastic Pool allows thousands of databases to share the cost of the compute pool, automatically absorbing the usage spikes of individual databases, providing strict physical isolation at a fraction of the cost.

---
**Next up in Chapter 40:** We have reached the final chapter of the book. We will wrap up everything we have learned by building a comprehensive **Architectural Review Board (ARB) Checklist**. This is the ultimate grading rubric for any enterprise SQL Server application.
