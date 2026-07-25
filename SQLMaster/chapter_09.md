# Part 3: Enterprise Analytical Patterns

# Chapter 9: CTEs & Recursive CTEs

## Learning Objectives
By the end of this chapter, you will be able to:
*   Refactor nested, unreadable subqueries into modular Common Table Expressions (CTEs).
*   Understand the physical execution differences between CTEs, Subqueries, and Temporary Tables.
*   Master **Recursive CTEs** to traverse hierarchical tree structures (like Station Groups or Fleet organizations) in a single query.
*   Prevent catastrophic infinite loops when querying self-referencing tables.

---

## 9.1 Introduction to Common Table Expressions

As our EV SaaS reporting requirements grow, so does the complexity of our SQL. Stacking multiple subqueries inside the `FROM` or `WHERE` clause quickly results in "spaghetti SQL" that is impossible to maintain.

A **Common Table Expression (CTE)** provides a way to define a temporary, named result set that exists only for the duration of a single `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement.

### The Syntax
A CTE is defined using the `WITH` keyword, followed by the query that populates it.

```sql
WITH HighUsageStations AS (
    SELECT StationId, SUM(TotalKwh) as TotalEnergy
    FROM core.Sessions
    WHERE TenantId = 'T1-UUID'
    GROUP BY StationId
    HAVING SUM(TotalKwh) > 5000
)
-- Immediately after the CTE, you must write the outer query
SELECT st.Name, h.TotalEnergy
FROM HighUsageStations h
INNER JOIN core.Stations st ON h.StationId = st.StationId;
```

---

## 9.2 CTEs vs. Subqueries vs. Temp Tables

**Architect Perspective:** A standard CTE is *not* a temporary table. It does not materialize data to disk (TempDB). Under the hood, the SQL Server Query Optimizer treats a CTE exactly like an inline view or a subquery. It simply expands the text of the CTE into the main query during compilation.

*   **Subquery:** Hard to read when nested deeply. No performance penalty.
*   **CTE:** Highly readable. Can be referenced multiple times in the same query. No performance penalty.
*   **Temp Table (`#Table`):** Actually writes data to TempDB. Has its own statistics and indexes. **Use Temp Tables instead of CTEs when processing millions of rows in complex batch jobs**, as the optimizer can use the Temp Table's statistics to build a much better execution plan.

---

## 9.3 Chaining Multiple CTEs

You can define multiple CTEs in a single statement by separating them with a comma. This allows you to build complex logic step-by-step.

```sql
WITH SessionTotals AS (
    SELECT UserId, SUM(TotalCost) AS LifetimeSpend
    FROM core.Sessions
    GROUP BY UserId
),
VIPUsers AS (
    SELECT UserId
    FROM SessionTotals
    WHERE LifetimeSpend > 1000.00
)
SELECT u.Email
FROM VIPUsers v
INNER JOIN core.Users u ON v.UserId = u.UserId;
```

---

## 9.4 Recursive CTEs (Navigating Hierarchies)

The true superpower of the CTE is **Recursion**. 
In our SaaS, a Tenant might organize their EV Chargers into a hierarchy: 
*   Global Fleet -> North America -> US -> California -> Bay Area -> specific Stations.

If we store this in a table using a `ParentGroupId` (a self-referencing Foreign Key), it is impossible to traverse this tree using standard `JOIN`s, because you don't know how deep the tree goes.

Enter the Recursive CTE.

### The Recursive Syntax
A Recursive CTE has three mandatory parts:
1.  **The Anchor Member:** The starting point (e.g., the Root node where `ParentGroupId IS NULL`).
2.  **`UNION ALL`:** The operator that ties the anchor to the recursive step.
3.  **The Recursive Member:** A query that joins the CTE back onto itself.

```sql
WITH FleetHierarchy AS (
    -- 1. Anchor Member (Get the Root level)
    SELECT GroupId, Name, ParentGroupId, 1 AS Level
    FROM core.StationGroups
    WHERE ParentGroupId IS NULL AND TenantId = 'T1-UUID'
    
    UNION ALL
    
    -- 2. Recursive Member (Get the children of the previous level)
    SELECT child.GroupId, child.Name, child.ParentGroupId, parent.Level + 1
    FROM core.StationGroups child
    INNER JOIN FleetHierarchy parent -- Joins to the CTE itself!
        ON child.ParentGroupId = parent.GroupId
)
-- 3. Outer Query
SELECT GroupId, Name, Level 
FROM FleetHierarchy
ORDER BY Level;
```

### Execution Flow
1.  SQL Server executes the Anchor. (Finds the Root node).
2.  It takes the Root node, and feeds it into the Recursive member to find its children.
3.  It takes those children, and feeds them back into the Recursive member to find *their* children.
4.  It repeats this loop until the Recursive member returns 0 rows.

---

## 9.5 The Code: EF Core and CTEs

Standard LINQ does not natively support generating Recursive CTEs. To use this enterprise pattern in C#, you must use raw SQL.

```csharp
// Scenario: We want to load the entire Fleet Hierarchy for a UI Tree View
var tenantId = new SqlParameter("tenantId", userTenantId);

var hierarchy = await context.StationGroups
    .FromSqlRaw(@"
        WITH FleetHierarchy AS (
            SELECT GroupId, Name, ParentGroupId, 1 AS Level
            FROM core.StationGroups
            WHERE ParentGroupId IS NULL AND TenantId = @tenantId
            
            UNION ALL
            
            SELECT child.GroupId, child.Name, child.ParentGroupId, parent.Level + 1
            FROM core.StationGroups child
            INNER JOIN FleetHierarchy parent ON child.ParentGroupId = parent.GroupId
        )
        SELECT GroupId, Name, ParentGroupId
        FROM FleetHierarchy", tenantId)
    .ToListAsync();
```

---

## 9.6 Performance & Security Analysis

### Performance Analysis: The Infinite Loop
If your hierarchical data is corrupted (e.g., Group A is the parent of Group B, and Group B is the parent of Group A), the Recursive CTE will bounce between them forever in an infinite loop, maxing out CPU and crashing the server.
**The Fix:** SQL Server has a built-in safety net called `MAXRECURSION`. By default, it is 100. If a CTE loops 101 times, SQL Server throws an error.
You can override it by adding `OPTION (MAXRECURSION 0)` to the outer query (0 means infinite), but **never do this in a SaaS** unless you mathematically guarantee tree validity.

### Security Implications
*   **Tenant Isolation in Recursion:** Notice in the Anchor Member we explicitly filtered `WHERE TenantId = @tenantId`. If you forget this filter on the anchor, the CTE might start traversing trees belonging to other organizations, resulting in a catastrophic data breach.

---

## 9.7 Common Mistakes & Production Pitfalls

1.  **Thinking CTEs cache data:** Developers often write a CTE and `JOIN` it 4 times in the outer query, thinking it acts as a cached variable. It does not. The SQL Optimizer will execute the CTE's underlying logic 4 separate times. If the CTE is doing a heavy aggregation, use a `#TempTable` instead.
2.  **Using `UNION` instead of `UNION ALL` in Recursion:** The SQL syntax strictly requires `UNION ALL` between the anchor and recursive members. `UNION` will fail compilation.

---

## 9.8 Production Checklist

*   [ ] Unreadable, nested subqueries are refactored into chained CTEs for maintainability.
*   [ ] Recursive CTEs rely on the default `MAXRECURSION` limit (100) to protect against infinite loops caused by bad data.
*   [ ] EF Core integration of CTEs uses parameterization (`SqlParameter`) in `.FromSqlRaw()` to prevent SQL injection.

---

## 9.9 Exercises

1.  **CTE Translation:** Rewrite the following correlated subquery as a CTE with an `INNER JOIN`:
    ```sql
    SELECT st.Name, 
           (SELECT MAX(StartTime) FROM core.Sessions s WHERE s.StationId = st.StationId) 
    FROM core.Stations st;
    ```
2.  **Recursive Safety:** A Junior DBA wrote a recursive CTE and appended `OPTION (MAXRECURSION 0)` at the end because the query kept failing with a depth limit error. Explain why this is dangerous in a production environment and what the actual root cause of the error likely is.

---

## 9.10 Interview Questions

**Q1: What is the physical execution difference between a CTE and a Temporary Table? When would you use one over the other?**
*Answer:* A CTE is purely a logical construct; it is not materialized to disk. The optimizer expands it into the main query like a view. A Temporary Table is physically written to TempDB, has its own statistics, and can be indexed. You should use a CTE to improve readability or for recursive queries. You should use a Temp Table when you have a multi-step batch process analyzing millions of rows, as breaking the query apart and materializing the intermediate steps gives the optimizer better statistics for the final joins.

**Q2: Describe the mandatory components of a Recursive CTE.**
*Answer:* A Recursive CTE requires three parts: an Anchor Member (the base query establishing the starting point), the `UNION ALL` operator, and the Recursive Member (a query that references the CTE itself to traverse the next level of the hierarchy).

---
**Next up in Chapter 10:** We will explore the pinnacle of SQL reporting: Window Functions. We will learn how to calculate running totals, rank data, and compare current rows to previous rows without using self-joins.
