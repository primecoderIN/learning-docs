# Chapter 29: Database Sharding & Horizontal Scale

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the physical limits of Vertical Scaling (Scale-Up).
*   Architect a Database Sharding strategy (Horizontal Partitioning) for multi-tenant SaaS applications.
*   Contrast Tenant-Based Sharding with Hash-Based Sharding.
*   Design a Shard Map Manager to route API traffic to the correct database instance dynamically.
*   Implement dynamic Connection String generation using EF Core's `IDbContextFactory`.

---

## 29.1 The Limits of Vertical Scaling

For the first three years of your SaaS startup, performance problems are usually solved by **Vertical Scaling** (Scale-Up). 
If the database is slow, you slide a slider in the Azure Portal to give the VM more CPU and more RAM.

However, Vertical Scaling has a physical and financial ceiling. 
Eventually, you will hit a point where you cannot buy a larger server, or the cost of the next tier is astronomically prohibitive. When a single SQL Server instance can no longer handle the sheer volume of global IoT telemetry, you must transition to **Horizontal Scaling** (Scale-Out).

---

## 29.2 Horizontal Partitioning (Sharding)

As discussed in Chapter 22, Table Partitioning splits a table across disks on the *same* server.
**Sharding** splits a database across entirely *different* physical servers.

If we have four database servers, we divide our global EV SaaS Tenant data among them.
*   **Shard 1:** Houses Acme Corp and 50 other mid-size tenants.
*   **Shard 2:** Houses Bob's Coffee and 1,000 tiny tenants.
*   **Shard 3:** Houses solely "MegaCorp", our largest enterprise client who demands dedicated hardware.

This architecture provides virtually infinite scalability. If Shard 1 runs out of CPU, you simply provision Shard 4 and migrate half of Shard 1's tenants to it.

---

## 29.3 Sharding Strategies

How do you decide which row goes to which Shard? You need a **Shard Key**. In a B2B SaaS, the Shard Key is almost always the `TenantId`.

### Strategy 1: Tenant-Based (Lookup) Sharding
You maintain a central lookup table (The Shard Map) that maps a `TenantId` to a specific database Connection String.
*   *Pros:* Complete control. You can give a massive VIP tenant their own dedicated server (Tenant Isolation), while grouping 1,000 free-tier tenants onto a shared server.
*   *Cons:* Requires a fast, highly available lookup mechanism before every API request.

### Strategy 2: Hash-Based Sharding
You pass the `TenantId` through a cryptographic hashing function (e.g., MD5), apply modulo math based on the number of shards, and route the traffic algorithmically.
*   *Pros:* No central lookup table required. Perfect mathematical distribution.
*   *Cons:* If you add a new Shard, the math changes, and you must migrate massive amounts of data to rebalance the cluster. You cannot isolate a "noisy neighbor" tenant to their own hardware easily.

*Architect's Choice:* B2B SaaS applications universally prefer **Tenant-Based (Lookup) Sharding**.

---

## 29.4 The Shard Map Manager

The Shard Map Manager is a critical infrastructure component. It is a tiny, ultra-fast database (or Redis cache) that holds the routing table.

**The Routing Table (Global Database):**
| TenantId | ShardName | ConnectionString |
| :--- | :--- | :--- |
| T1-UUID | Shard-US-East-1 | Server=db1.core.com;Database=Shard1;... |
| T2-UUID | Shard-US-East-1 | Server=db1.core.com;Database=Shard1;... |
| T3-UUID | Shard-EU-West-2 | Server=db4.core.com;Database=Shard4;... |

When an API request arrives for `T3-UUID`, the application queries the Shard Map Manager (usually cached in C# Memory), gets the connection string for `db4`, and instantiates Entity Framework Core pointing to that specific server.

---

## 29.5 Cross-Shard Queries and Fan-out

What happens when the CEO asks for a report: *"Show me the total revenue across all tenants globally."*

Because the data is now split across 4 physical servers, a simple `SELECT SUM(Revenue)` no longer works. You must execute a **Fan-out Query**.
The application (or a specialized engine like Azure Elastic Query) must fire 4 independent SQL queries simultaneously to all 4 shards, wait for 4 numbers to return over the network, and then sum them together in C# memory.

*Architect Warning:* Fan-out queries are complex, slow, and prone to partial failures (what if Shard 3 is offline?). If your application requires frequent cross-tenant reporting, Sharding will introduce massive architectural pain. Use Sharding only when data strictly aligns with the Shard Key boundaries.

---

## 29.6 The Code: EF Core DbContext Factory

In a non-sharded application, you inject a single `DbContext` into your controllers. In a sharded application, the Connection String isn't known until the HTTP Request provides the `TenantId`.

You must use EF Core's `IDbContextFactory` to dynamically build the context per request.

```csharp
public class ShardedDataService
{
    private readonly IDbContextFactory<VoltCoreDbContext> _contextFactory;
    private readonly IShardMapManager _shardMap;

    public ShardedDataService(
        IDbContextFactory<VoltCoreDbContext> contextFactory, 
        IShardMapManager shardMap)
    {
        _contextFactory = contextFactory;
        _shardMap = shardMap;
    }

    public async Task UpdateStationAsync(Guid tenantId, Guid stationId)
    {
        // 1. Lookup the correct physical server
        string connectionString = await _shardMap.GetConnectionStringAsync(tenantId);

        // 2. Dynamically build the DbContext options
        var optionsBuilder = new DbContextOptionsBuilder<VoltCoreDbContext>()
            .UseSqlServer(connectionString);

        // 3. Create the context pointing to the specific Shard
        using var context = new VoltCoreDbContext(optionsBuilder.Options);

        // 4. Execute the query
        var station = await context.Stations.FindAsync(stationId);
        // ... logic ...
    }
}
```

---

## 29.7 Performance & Security Analysis

### Performance Analysis: The Noisy Neighbor
A primary driver for Sharding is mitigating the "Noisy Neighbor" problem. If Bob's Coffee runs a massive, unoptimized report that consumes 100% of the CPU on Shard 1, it will slow down Acme Corp (who is also on Shard 1). However, MegaCorp (who is on Shard 3) will not notice anything. The blast radius of bad queries is localized to the physical shard.

### Security Implications
*   **Shard Map Spoofing:** If an attacker can manipulate the HTTP headers or JWT payload to forge a different `TenantId`, they might trick the Shard Map Manager into routing them to a different physical database. This emphasizes why Row-Level Security (RLS - Chapter 27) must still be applied *within* every single Shard as a defense-in-depth measure.

---

## 29.8 Common Mistakes & Production Pitfalls

1.  **Unique Constraints Across Shards:** You cannot enforce a global `UNIQUE` constraint across physical servers. If you need emails to be globally unique across your entire SaaS, a unique constraint on `Users.Email` in Shard 1 does not prevent that same email from being registered in Shard 2. You must handle global uniqueness in the application layer (or the Shard Map database).
2.  **Schema Drift:** If you have 50 Shards, you must deploy database schema migrations (e.g., adding a column) to 50 servers simultaneously. If Shard 1 upgrades successfully but Shard 2 fails, your application must be able to handle "Schema Drift." Deployment pipelines for sharded databases are incredibly complex.

---

## 29.9 Production Checklist

*   [ ] The Shard Key (`TenantId`) is present on *every single table* in the database to guarantee data locality.
*   [ ] The Shard Map Manager routing table is aggressively cached in API memory (e.g., `IMemoryCache`) to prevent lookup latency.
*   [ ] EF Core is configured using `IDbContextFactory` or a custom Multi-Tenant Connection Provider to support dynamic connection strings.
*   [ ] Global uniqueness requirements (e.g., User Emails, API Keys) are validated outside of the individual shards.

---

## 29.10 Exercises

1.  **Architectural Design:** A multi-national SaaS must comply with strict data residency laws (EU customer data cannot physically leave Germany; US customer data cannot leave Virginia). Explain how a Tenant-Based Sharding strategy solves this regulatory requirement seamlessly, whereas Hash-Based Sharding completely fails.
2.  **Fan-out Calculation:** You have 10 shards. You need to calculate the average charging session duration globally. Why is it mathematically incorrect to execute `SELECT AVG(Duration)` on all 10 shards and then average those 10 numbers together in C#? How must you construct the fan-out queries to get the accurate global average?

---

## 29.11 Interview Questions

**Q1: What is Database Sharding, and how does it differ from Table Partitioning?**
*Answer:* Database Sharding is a horizontal scaling technique that splits a logical database into multiple physical database servers (nodes) based on a Shard Key (like `TenantId`). It increases overall system CPU, RAM, and storage capacity. Table Partitioning (vertical) splits a massive table into multiple filegroups on the *same* physical server, primarily to aid in data archiving (Sliding Window) and disk I/O management, but does not provide additional compute resources.

**Q2: Explain the "Noisy Neighbor" problem in multi-tenant architecture and how Sharding mitigates it.**
*Answer:* The Noisy Neighbor problem occurs when one massive tenant executes heavy, unoptimized queries that saturate a shared server's CPU or memory, degrading the performance for all other tenants on that server. Sharding mitigates this by isolating the blast radius. You can physically segregate large, noisy tenants onto their own dedicated Shard (hardware), ensuring their resource consumption has zero impact on the smaller tenants housed on shared Shards.

---
**Next up in Chapter 30:** We will explore Asynchronous processing and the Database as a Queue pattern. We will cover the risks of polling the database and introduce Service Broker and Change Data Capture (CDC).
