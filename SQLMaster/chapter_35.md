# Chapter 35: Graph Databases in SQL Server

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the performance limitations of standard relational `JOIN`s when querying deep, unbounded hierarchical data.
*   Architect a Graph Database schema using SQL Server **Node** and **Edge** tables.
*   Construct complex relationship queries using the T-SQL `MATCH` clause.
*   Compare the capabilities of SQL Server Graph features against dedicated graph databases like Neo4j.
*   Integrate Graph queries within an Entity Framework Core application.

---

## 35.1 The Limits of Relational Hierarchies

Relational databases are fantastic at 1-to-Many and Many-to-Many relationships. But they struggle massively with **Deep Hierarchies** or **Highly Interconnected Data**.

**The Scenario:**
In our EV SaaS platform, we want to build a social feature: *"Find all users who charge their cars at the same Station as my friends, or friends-of-my-friends, up to 3 degrees of separation."*

Using standard SQL tables, you would have to write:
```sql
SELECT ...
FROM Users u1
JOIN Friendships f1 ON u1.Id = f1.UserId1
JOIN Users u2 ON f1.UserId2 = u2.Id
JOIN Friendships f2 ON u2.Id = f2.UserId1
JOIN Users u3 ON f2.UserId2 = u3.Id
JOIN ChargingSessions s ON u3.Id = s.UserId
JOIN Stations st ON s.StationId = st.Id
```
This is a nightmare. It requires 6 hardcoded `JOIN` statements. If you want to go 4 degrees deep, you have to rewrite the SQL query. Furthermore, executing 6 self-joins on a 10-million row table will cause SQL Server to consume massive amounts of CPU and likely time out.

*(Note: In Chapter 9, we used Recursive CTEs for hierarchies, but they are notoriously slow for highly connected "web" data).*

---

## 35.2 Node and Edge Tables

To solve this, SQL Server 2017 introduced native **Graph Database** capabilities.

A Graph Database models data as:
1.  **Nodes:** The entities (e.g., Users, Stations).
2.  **Edges:** The relationships connecting the entities (e.g., "FriendsWith", "ChargesAt").

Instead of standard tables, you explicitly create Node and Edge tables. SQL Server automatically generates hidden columns (`$node_id`, `$edge_id`, `$from_id`, `$to_id`) to optimize traversal.

```sql
-- 1. Create the Nodes
CREATE TABLE Users (
    UserId INT PRIMARY KEY,
    Name VARCHAR(100)
) AS NODE;

CREATE TABLE Stations (
    StationId INT PRIMARY KEY,
    Location VARCHAR(100)
) AS NODE;

-- 2. Create the Edges (Relationships)
CREATE TABLE FriendsWith AS EDGE;
CREATE TABLE ChargesAt AS EDGE;
```

---

## 35.3 The `MATCH` Clause

Once the data is populated, you query it using the new `MATCH` clause. The syntax is designed to look like ASCII art representing the graph connections.

To find the stations where a user's friends charge:
```sql
SELECT Friend.Name, Station.Location
FROM 
    Users AS Person, 
    FriendsWith, 
    Users AS Friend, 
    ChargesAt, 
    Stations AS Station
WHERE MATCH(Person-(FriendsWith)->Friend-(ChargesAt)->Station)
AND Person.Name = 'Alice';
```

**Shortest Path:**
SQL Server 2019 added the `SHORTEST_PATH` function, which allows unbounded traversal without hardcoding the depth!
```sql
-- Find connections up to 3 levels deep
SELECT Person.Name, Friend.Name
FROM Users AS Person, FriendsWith FOR PATH, Users FOR PATH AS Friend
WHERE MATCH(SHORTEST_PATH(Person-(FriendsWith)->+Friend))
AND Person.Name = 'Alice';
```

---

## 35.4 Architect Perspective: SQL Server vs Neo4j

If Graph capabilities are built into SQL Server, why do massive enterprises use dedicated graph databases like **Neo4j**?

**The Case for SQL Server Graph:**
*   **Unified Ecosystem:** You don't have to sync data out to a separate system. You can write a query that uses a Graph `MATCH` and joins it to a standard relational `Invoices` table in the exact same transaction.
*   **Security:** Existing Row-Level Security (RLS), Backup strategies, and High Availability (Always On AGs) apply instantly to your graph data.

**The Case for Neo4j:**
*   **Index-Free Adjacency:** Under the hood, SQL Server Graph is still executing relational joins using hidden B-Tree indexes. If the graph gets massive (billions of edges), those B-Trees become a bottleneck. Neo4j uses "Index-Free Adjacency" (physical memory pointers linking nodes), which makes traversing 10 levels deep lightning fast regardless of the total data size.

*Architect Rule:* Use SQL Server Graph features for localized, moderately deep relationship traversals (e.g., 2-4 levels) to avoid the operational nightmare of maintaining two separate database systems. If your *entire* business model is a massive social network or recommendation engine, migrate to Neo4j.

---

## 35.5 The Code: EF Core and Graph Queries

Entity Framework Core does *not* have fluent LINQ support for the `MATCH` clause. 
If you try to write `_context.Users.Include(u => u.Friends)`, EF Core will generate standard relational `JOIN`s, completely bypassing the graph engine.

To utilize the Graph engine, you must use Raw SQL via `FromSqlInterpolated`.

```csharp
public async Task<List<string>> GetFriendsStationsAsync(string userName)
{
    // You must map the result of the MATCH query to a DTO, 
    // as it crosses multiple node types.
    var results = await _context.Database.SqlQuery<string>($@"
        SELECT Station.Location AS Value
        FROM Users AS Person, FriendsWith, Users AS Friend, ChargesAt, Stations AS Station
        WHERE MATCH(Person-(FriendsWith)->Friend-(ChargesAt)->Station)
        AND Person.Name = {userName}
    ").ToListAsync();

    return results;
}
```

---

## 35.6 Performance & Security Analysis

### Performance Analysis: Edge Indexes
When you create an `EDGE` table, SQL Server automatically creates a unique index on `($edge_id)`. However, it does *not* automatically create indexes on the `$from_id` or `$to_id` columns. If you execute a `MATCH` clause and do not manually create nonclustered indexes on the Edge's From/To columns, SQL Server will execute a Clustered Index Scan across the entire Edge table. You must manually tune Edge tables exactly like relational tables.

### Security Implications
*   **Graph Traversal Data Leaks:** In a multi-tenant system, Graph queries can easily traverse *across* tenant boundaries if the Edge relationships are not strictly policed. If an Edge is created between `User (Tenant A)` and `Station (Tenant B)`, a simple `MATCH` query will return cross-tenant data. RLS (Row-Level Security) must be applied to both the Node tables *and* the Edge tables to prevent traversal leaks.

---

## 35.7 Common Mistakes & Production Pitfalls

1.  **Treating Edges as Nodes:** Developers sometimes try to add massive amounts of business data (like JSON payloads or foreign keys) to an `EDGE` table. Edges should be incredibly thin (usually just a Weight/Score integer or a Date). If you need complex data, it belongs in a Node.
2.  **Ignoring the T-SQL Limitations:** SQL Server Graph has limitations. You cannot `UPDATE` an edge to point to a different node; you must `DELETE` the old edge and `INSERT` a new one. PolyMorphism (an edge pointing to *either* a User Node or a Station Node) is supported via Edge Constraints, but severely complicates the query plan.

---

## 35.8 Production Checklist

*   [ ] Highly interconnected traversal queries have been refactored from Recursive CTEs / multi-JOINs to Graph `MATCH` clauses.
*   [ ] Explicit Nonclustered Indexes have been created on the hidden `$from_id` and `$to_id` columns of all `EDGE` tables.
*   [ ] EF Core integration utilizes `SqlQuery<T>` or `FromSqlInterpolated` to pass parameters securely into the `MATCH` clause.
*   [ ] Row-Level Security Policies are explicitly applied to `EDGE` tables to prevent cross-tenant traversal leaks.

---

## 35.9 Exercises

1.  **Relational vs Graph:** A business requirement states: "Find all Managers who manage Employees who have filed an IT Ticket assigned to a Technician located in the same City as the Manager." Write out the conceptual `MATCH` clause for this query, identifying the Nodes and Edges.
2.  **Performance Tuning:** You execute a `MATCH` query spanning 3 Nodes and 2 Edges. Query Store shows the query is consuming 100% CPU and executing massive Table Scans. What critical step did the DBA forget to perform when creating the `EDGE` tables?

---

## 35.10 Interview Questions

**Q1: What specific problem do Graph Databases solve that traditional Relational Databases struggle with?**
*Answer:* Relational databases struggle with highly interconnected, deeply hierarchical data (like social networks or recommendation engines). Traversing multiple levels of relationships in SQL requires either massive, hardcoded `JOIN` statements or highly inefficient Recursive CTEs, leading to catastrophic CPU overhead and timeouts. Graph databases solve this by storing data as Nodes and Edges, allowing for highly optimized, unbounded traversal queries using specialized syntax (like the `MATCH` clause).

**Q2: In SQL Server Graph, why is it critical to apply Row-Level Security to the Edge tables, not just the Node tables?**
*Answer:* If you only secure the Node tables, the Graph engine might still use the unprotected Edge table to determine that a relationship path exists. In some execution plans, or if an attacker crafts a malicious `MATCH` query, the presence of the Edge itself can leak the existence of cross-tenant relationships (a side-channel attack). Securing the Edge table ensures the traversal algorithm halts immediately at the tenant boundary.

---
**Next up in Chapter 36:** We will explore In-Memory OLTP (Hekaton). We will learn how to achieve microsecond latency by compiling tables directly into RAM and bypassing the SQL Server locking engine entirely.
