# Part 8: Security & High Availability

# Chapter 27: Row-Level Security (RLS)

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the risks of relying entirely on Application-Level isolation (Global Query Filters) for multi-tenant data.
*   Implement SQL Server **Row-Level Security (RLS)** to enforce tenant isolation at the engine level.
*   Construct an Inline Table-Valued Function (iTVF) as a Security Predicate using `SESSION_CONTEXT`.
*   Apply Security Policies using `FILTER` and `BLOCK` predicates.
*   Configure Entity Framework Core Interceptors to automatically inject the `TenantId` into the SQL Server session.

---

## 27.1 The Threat: Tenant Data Bleed

In our EV SaaS platform, Tenant A (Acme Corp) and Tenant B (Bob's Coffee) share the same `core.Stations` table (Shared Database, Shared Schema architecture).

**The Threat:** If a developer accidentally writes `_context.Stations.ToList()` and forgets to add `.Where(s => s.TenantId == tenantId)`, Acme Corp will see Bob's financial data on their dashboard. This is called **Tenant Data Bleed**, and it is a company-ending event.

Developers usually fix this using EF Core's **Global Query Filters**:
```csharp
// Inside OnModelCreating
builder.Entity<Station>().HasQueryFilter(s => s.TenantId == _currentTenantId);
```
While this is good, it is easily bypassed in code by calling `.IgnoreQueryFilters()` or by executing a raw SQL query. 
To achieve "Defense in Depth" (Chapter 3), we must enforce isolation at the database engine level, so it is mathematically impossible to bypass, even if the application is compromised.

---

## 27.2 Introduction to Row-Level Security (RLS)

SQL Server **Row-Level Security (RLS)** allows you to restrict read and write access to specific rows in a table based on the execution context of the query.

If RLS is enabled, a query like `SELECT * FROM core.Stations` will magically only return rows belonging to the currently authenticated Tenant. The filtering happens deep inside the Storage Engine, completely transparent to the application layer.

---

## 27.3 Implementing RLS

RLS requires three components:
1.  **Session Context:** A secure key/value pair stored in the database session.
2.  **Security Predicate:** A SQL Function that evaluates the row against the Session Context.
3.  **Security Policy:** The object that binds the Predicate to the Table.

### Step 1: The Session Context
When our API opens a connection to SQL Server, we execute a system procedure to set the current Tenant ID into the session memory.
```sql
EXEC sp_set_session_context @key = N'TenantId', @value = 'T1-UUID';
```

### Step 2: The Security Predicate
We write an Inline Table-Valued Function (Chapter 13) that takes the row's `TenantId` and compares it to the Session Context.

```sql
CREATE SCHEMA security;
GO

CREATE FUNCTION security.fn_TenantAccessPredicate (@RowTenantId UNIQUEIDENTIFIER)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessResult
    WHERE 
        -- Allow access if the row's TenantId matches the Session's TenantId
        @RowTenantId = CAST(SESSION_CONTEXT(N'TenantId') AS UNIQUEIDENTIFIER)
        -- Escape hatch: Allow super-admins (Support Team) to see all rows
        OR CAST(SESSION_CONTEXT(N'IsSuperAdmin') AS INT) = 1
);
```

### Step 3: The Security Policy
We bind the function to the `core.Stations` table.

```sql
CREATE SECURITY POLICY security.TenantIsolationPolicy
ADD FILTER PREDICATE security.fn_TenantAccessPredicate(TenantId) ON core.Stations,
ADD BLOCK PREDICATE security.fn_TenantAccessPredicate(TenantId) ON core.Stations
WITH (STATE = ON);
```

*   **FILTER Predicate:** Silently filters out rows during `SELECT`, `UPDATE`, and `DELETE`.
*   **BLOCK Predicate:** Prevents a user from `INSERT`ing a row for a different Tenant, throwing an immediate error.

---

## 27.4 Architect Perspective: The Performance Hit

RLS is incredibly secure, but it incurs a performance penalty.

Because RLS appends an invisible `WHERE` clause to *every single query* run against that table, it acts similarly to a View. If the Security Predicate is complex (e.g., performing lookups into a `Roles` table), it will destroy the Query Optimizer's ability to create efficient execution plans, causing massive RBAR (Row-By-Agonizing-Row) processing.

**Architect Rule:** Keep the Security Predicate as simple as humanly possible. Relying purely on `SESSION_CONTEXT` is the fastest method because it doesn't require any physical disk reads to evaluate.

---

## 27.5 The Code: EF Core Interceptors

How do we ensure that every time EF Core opens a database connection, it executes `sp_set_session_context` before running the actual query?
We use an **EF Core `DbConnectionInterceptor`**.

```csharp
public class TenantSessionInterceptor : DbConnectionInterceptor
{
    private readonly ITenantProvider _tenantProvider; // Reads TenantId from JWT HttpContext

    public TenantSessionInterceptor(ITenantProvider tenantProvider)
    {
        _tenantProvider = tenantProvider;
    }

    public override async Task ConnectionOpenedAsync(
        DbConnection connection, ConnectionEndEventData eventData, CancellationToken cancellationToken)
    {
        var tenantId = _tenantProvider.GetCurrentTenantId();
        
        using var command = connection.CreateCommand();
        command.CommandText = "EXEC sp_set_session_context @key=N'TenantId', @value=@tenantId;";
        
        var param = command.CreateParameter();
        param.ParameterName = "@tenantId";
        param.Value = tenantId;
        command.Parameters.Add(param);
        
        await command.ExecuteNonQueryAsync(cancellationToken);
    }
}
```
*Registration:* You register this interceptor in your `AddDbContext` configuration. Now, it is absolutely impossible for any developer query to bypass the Tenant isolation, even if they use raw SQL!

---

## 27.6 Performance & Security Analysis

### Security Implications: Side-Channel Attacks
RLS is not perfect. A malicious user with query execution access can perform a Side-Channel (Timing) Attack. 
If an attacker writes: 
`SELECT * FROM core.Stations WHERE 1/(CASE WHEN Name = 'SecretProject' THEN 0 ELSE 1 END) = 1;`
Even if RLS filters out the 'SecretProject' row so the attacker can't see it, the `WHERE` clause might evaluate *before* the RLS filter is applied. The query will throw a "Divide by Zero" error. The attacker now knows that a station named 'SecretProject' exists in another Tenant's database! 
*Fix:* Never allow external users to execute ad-hoc SQL against an RLS-protected database.

---

## 27.7 Common Mistakes & Production Pitfalls

1.  **Connection Pooling Data Bleed:** If you use a standard SQL connection string, SQL Server pools connections to save CPU. If Request A sets the `SESSION_CONTEXT` to 'T1', and that connection goes back to the pool, Request B might pick up that exact same connection. If Request B fails to set its own `SESSION_CONTEXT`, it will execute as 'T1'! 
    *Fix:* Always ensure the `sp_set_session_context` interceptor fires *every single time* the connection is opened from the pool.
2.  **Using `CONTEXT_INFO` instead of `SESSION_CONTEXT`:** Legacy systems used `CONTEXT_INFO`, which requires converting GUIDs to `VARBINARY(128)`, leading to massive encoding bugs. Always use the modern `SESSION_CONTEXT` which stores native SQL types.

---

## 27.8 Production Checklist

*   [ ] Multi-tenant isolation is enforced at the Storage Engine level using Row-Level Security (RLS).
*   [ ] The RLS Security Predicate is implemented as an Inline Table-Valued Function (iTVF) without external table joins to maximize performance.
*   [ ] EF Core utilizes a `DbConnectionInterceptor` to safely and consistently inject `SESSION_CONTEXT` upon connection open.
*   [ ] Both `FILTER` (Read) and `BLOCK` (Write) predicates are applied to the Security Policy.

---

## 27.9 Exercises

1.  **Block Predicate Testing:** A Security Policy with a `BLOCK PREDICATE` is applied to `core.Stations`. The current `SESSION_CONTEXT` has TenantId = 'T1'. A developer executes: `INSERT INTO core.Stations (TenantId, Name) VALUES ('T2', 'Rogue Station')`. What exact error will SQL Server throw, and why?
2.  **Performance Tuning:** A Junior DBA attempts to make RLS more robust by changing the Security Predicate function to perform a `JOIN` against the `core.Users` table to verify the user's role on every row evaluation. Why will this crash the production server during high load?

---

## 27.10 Interview Questions

**Q1: Contrast EF Core Global Query Filters with SQL Server Row-Level Security (RLS). Why might an architect choose to implement both?**
*Answer:* EF Core Global Query Filters are an application-level constraint. EF Core automatically appends a `WHERE` clause to generated LINQ queries. However, they are easily bypassed intentionally (`IgnoreQueryFilters()`) or accidentally (using raw SQL/Dapper). RLS is a database-level constraint enforced by the Storage Engine. It cannot be bypassed by the application layer. An architect implements both for "Defense in Depth": the EF Core filter provides early, clean filtering and predictable LINQ translation, while RLS provides a foolproof, impenetrable safety net at the bare metal in case of an application compromise.

**Q2: How do you pass the application's authenticated Tenant ID to SQL Server in a secure way that integrates seamlessly with Connection Pooling?**
*Answer:* You use the `sp_set_session_context` system stored procedure. In EF Core, you implement a `DbConnectionInterceptor`. Every time a connection is pulled from the connection pool and opened, the interceptor reads the Tenant ID from the HTTP Context and executes the stored procedure. This ensures the SQL session memory is overwritten with the correct Tenant ID before any business queries are executed, preventing cross-tenant data bleed within the shared connection pool.

---
**Next up in Chapter 28:** Now that we have isolated our tenants securely, we must ensure our database is highly available. We will explore SQL Server High Availability (HA) architectures, focusing on Always On Availability Groups and Disaster Recovery (DR) strategies.
