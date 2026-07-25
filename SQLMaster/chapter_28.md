# Chapter 28: High Availability & Disaster Recovery

## Learning Objectives
By the end of this chapter, you will be able to:
*   Differentiate between High Availability (HA) and Disaster Recovery (DR).
*   Define the business critical metrics: RTO (Recovery Time Objective) and RPO (Recovery Point Objective).
*   Architect an **Always On Availability Group (AG)** across multiple data centers.
*   Evaluate the performance trade-offs of Synchronous vs. Asynchronous commits.
*   Configure EF Core to utilize Read-Scale Out by routing reporting queries to read-only secondary replicas.

---

## 28.1 HA vs. DR

As a Database Architect, it is your responsibility to ensure the EV SaaS platform stays online when physical hardware inevitably fails.

*   **High Availability (HA):** Handling local, transient failures seamlessly. If a hard drive fails on Server A, the database automatically fails over to Server B in the same data center within 5 seconds. The user never notices an outage.
*   **Disaster Recovery (DR):** Handling catastrophic, site-wide failures. If a hurricane destroys the entire US-East data center, you must manually fail over to a server in US-West. Users *will* experience an outage, but the goal is to bring the system back online without losing data.

---

## 28.2 The Business Metrics: RTO & RPO

Before designing an architecture, you must ask the business two questions:

1.  **RPO (Recovery Point Objective):** *"How much data can we legally lose?"* 
    *   If RPO is 0, every single transaction must be replicated to a remote server before the user receives a "Success" message.
    *   If RPO is 15 minutes, you can rely on async replication or log backups.
2.  **RTO (Recovery Time Objective):** *"How long can the system be completely offline?"*
    *   If RTO is 10 seconds, you need automatic HA clustering.
    *   If RTO is 4 hours, you can manually restore from backups.

---

## 28.3 Always On Availability Groups (AGs)

The enterprise standard for SQL Server HA/DR is the **Always On Availability Group**.
It utilizes Windows Server Failover Clustering (WSFC).

1.  You provision a **Primary Replica** (Server A). This handles all Read and Write traffic.
2.  You provision one or more **Secondary Replicas** (Server B, Server C).
3.  When a transaction commits on Server A, SQL Server captures the Transaction Log (LDF) blocks and streams them over the network to Server B and C.
4.  Server B and C replay those log blocks into their own physical MDF files.

There is no shared storage (like in the older SAN-based Failover Cluster Instances). Every server has its own independent hard drives, meaning there is no single point of failure.

---

## 28.4 Synchronous vs. Asynchronous Commit

The replication of the transaction log dictates your RPO and your performance.

### Synchronous Commit (For HA)
*   **How it works:** Thread A runs an `UPDATE`. The Primary Server writes it to its local log. The Primary *waits*. It streams the log to the Secondary. The Secondary writes it to its local log and sends an ACK back. The Primary finally sends a "Success" message to the user.
*   **RPO:** 0 data loss guaranteed.
*   **Trade-off:** High latency. If the network between Primary and Secondary is slow, every single `UPDATE` in your application becomes slow.
*   **Usage:** Used between nodes in the *same* data center.

### Asynchronous Commit (For DR)
*   **How it works:** Thread A runs an `UPDATE`. The Primary Server writes it to its local log and immediately sends "Success" to the user. In the background, it streams the log to the Secondary.
*   **RPO:** Potential data loss. If the Primary blows up before the background stream finishes, the Secondary is missing data.
*   **Trade-off:** Blazingly fast. No network latency applied to the user's transaction.
*   **Usage:** Used between nodes in *different* data centers (e.g., US-East to US-West).

---

## 28.5 Read-Scale Out (Readable Secondaries)

Always On AGs provide a massive architectural bonus: **Readable Secondaries**.
Since Server B is constantly receiving and replaying data, you can configure it to accept Read-Only connections.

Instead of running massive, CPU-heavy reporting dashboards against the Primary Server (competing with IoT telemetry inserts), you route all reporting API calls to the Secondary Server. This doubles the computing power of your cluster for free.

---

## 28.6 The Code: EF Core Read-Only Routing

To utilize Read-Scale Out, your API does not connect directly to Server A or Server B. It connects to the **AG Listener** (a virtual IP address).

To route a query to the Secondary replica, you simply append `ApplicationIntent=ReadOnly` to your connection string. 

### Multi-Context Architecture
In ASP.NET Core, architect your application with two DbContexts.

```csharp
// 1. Connection Strings in appsettings.json
{
  "ConnectionStrings": {
    "WriteDb": "Server=tcp:ag-listener;Database=VoltCore;Integrated Security=SSPI;",
    "ReadDb": "Server=tcp:ag-listener;Database=VoltCore;Integrated Security=SSPI;ApplicationIntent=ReadOnly;"
  }
}

// 2. Dependency Injection
services.AddDbContext<VoltCoreWriteContext>(options => options.UseSqlServer("WriteDb"));
services.AddDbContext<VoltCoreReadContext>(options => 
{
    options.UseSqlServer("ReadDb");
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking); // Force NoTracking globally for reads!
});
```
When your CQRS MediatR Queries run, they inject `VoltCoreReadContext`, automatically routing the traffic to the Secondary replica without any code changes.

---

## 28.7 Architect Perspective: Transient Fault Handling

When the Primary Server crashes, the AG Listener takes about 5 to 10 seconds to detect the failure, promote the Secondary to Primary, and move the Virtual IP.
During those 10 seconds, any EF Core queries will throw a `SqlException` (Network transport error).

To prevent the user from seeing a 500 Error, you must implement **Transient Fault Handling**. EF Core has this built-in via Execution Strategies.

```csharp
optionsBuilder.UseSqlServer(connectionString, sqlServerOptionsAction: sqlOptions =>
{
    // If the database fails over, EF Core will silently catch the exception
    // and retry the query up to 5 times over 30 seconds before failing.
    sqlOptions.EnableRetryOnFailure(
        maxRetryCount: 5,
        maxRetryDelay: TimeSpan.FromSeconds(30),
        errorNumbersToAdd: null);
});
```

---

## 28.8 Performance & Security Analysis

### Performance Analysis: Redo Queue Build-up
On a Secondary Replica, a background thread continuously "Redoes" the incoming log blocks to update the data pages. If you run massive reporting queries on the Secondary, those queries take Shared (S) locks on the tables. Because readers block writers (unless RCSI is enabled, as discussed in Chapter 18), the reporting queries will block the Redo thread. The Secondary will fall further and further behind the Primary. **You MUST enable RCSI on Availability Groups to ensure Read-Scale Out functions properly.**

### Security Implications
*   **Orphaned Logins:** Logins (Server-level security) are not replicated by Availability Groups. Only Users (Database-level security) are. If you fail over to Server B, but forgot to create the API's Login on Server B, the API will fail to authenticate. You must synchronize Logins and Passwords across all nodes using Contained Databases or manual DBA scripts.

---

## 28.9 Common Mistakes & Production Pitfalls

1.  **Ignoring Split-Brain Syndrome:** If the network link between Server A and Server B dies, but both servers are still running, they might both assume the other is dead and try to act as the Primary. This is "Split-Brain," and it causes massive data corruption. AGs fix this using a **Quorum** (usually a small File Share Witness). You must configure Quorum correctly so the cluster can mathematically vote on who the true Primary is.
2.  **Transactions spanning databases:** Always On AGs replicate at the *database* level. If a single transaction updates `core.Sessions` (Database A) and `billing.Invoices` (Database B), and a failover happens exactly in the middle, the databases can become out of sync, breaking ACID rules. You must enable Distributed Transaction Coordinator (DTC) support for the AG if you do this.

---

## 28.10 Production Checklist

*   [ ] Primary and Secondary replicas within the same region are configured with Synchronous Commit for zero data loss (HA).
*   [ ] The Disaster Recovery replica in a remote region is configured with Asynchronous Commit to prevent network latency from destroying write performance.
*   [ ] CQRS Read queries utilize `ApplicationIntent=ReadOnly` to offload CPU from the primary server.
*   [ ] EF Core `EnableRetryOnFailure` is configured to handle the 10-second transient blip during a failover.

---

## 28.11 Exercises

1.  **RPO vs. RTO:** The CEO states: "If the data center burns down, I don't care if we are offline for 12 hours while we spin up a new server in AWS, but we absolutely cannot lose a single charging session." Define the RTO and RPO requested by the CEO. Can Asynchronous Commit AG replication satisfy this requirement?
2.  **Connection Strings:** You are writing an EF Core background worker that generates a massive monthly Excel report. Write the exact parameter you must append to the SQL Server connection string to ensure this background worker does not consume CPU on the Primary replica.

---

## 28.12 Interview Questions

**Q1: Explain the architectural trade-off between Synchronous and Asynchronous commit modes in an Always On Availability Group.**
*Answer:* Synchronous commit guarantees zero data loss (RPO = 0) because the Primary server will not acknowledge a transaction as successful until the Secondary server confirms it has hardened the transaction log to disk. The trade-off is high write latency; the application must wait for the network round-trip. Asynchronous commit eliminates this latency, as the Primary acknowledges success instantly while streaming the log to the Secondary in the background. The trade-off is potential data loss; if the Primary crashes before the log arrives at the Secondary, that data is permanently lost.

**Q2: What is the purpose of an Availability Group "Listener", and how does it assist with High Availability?**
*Answer:* The Listener is a virtual network name (VNN) and virtual IP address that floats between the nodes in the cluster. Instead of hardcoding the API connection string to "Server A", the application connects to "The Listener." If Server A crashes, the cluster promotes Server B to Primary and instantly moves the Listener's IP address to point to Server B. The application automatically reconnects to the new Primary without any manual configuration changes.

---
**Next up in Chapter 29:** As our SaaS scales globally, a single Availability Group is no longer enough. We will explore advanced Database Sharding and Horizontal Partitioning patterns to achieve infinite scale across multiple geographic regions.
