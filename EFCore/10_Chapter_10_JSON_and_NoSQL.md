# Chapter 10: JSON and NoSQL Integration

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Integrate unstructured JSON data natively into a relational SQL Server database using EF Core 7+ `.ToJson()` mapping.
*   Translate C# LINQ queries directly into highly optimized SQL Server `JSON_VALUE` scalar functions.
*   Architect hybrid schemas, balancing strict relational normalization with NoSQL-style dynamic document storage.
*   Evaluate the performance implications of querying JSON columns and implement computed columns for indexability.
*   Understand the fundamentals of swapping the EF Core Relational provider for the Azure Cosmos DB (NoSQL) provider.

## 2. Introduction

Historically, the software architecture world was divided into two warring factions: Relational (SQL) and NoSQL (Document). 
*   **Relational Purists** normalized everything into dozens of tables, creating inflexible schemas that required weeks of DBA approval to change. 
*   **NoSQL Advocates** dumped everything into massive JSON documents, abandoning referential integrity and eventually drowning in data corruption.

Modern Enterprise Architecture embraces the **Hybrid Database**. SQL Server (since 2016) has treated JSON as a first-class citizen. You can store strict relational data (Users, Tenants, Invoices) alongside dynamic, schemaless JSON data (User Preferences, Device Telemetry, Webhook Payloads) *in the exact same table*.

Prior to EF Core 7, interacting with SQL Server JSON required clunky workarounds using Value Converters or raw SQL. EF Core 7+ revolutionized this by introducing native JSON mapping. You map a C# object graph, tell EF Core `.ToJson()`, and it seamlessly handles the serialization, while miraculously translating your C# LINQ queries into native SQL JSON functions.

This chapter teaches you how to break the rules of relational normalization safely and performantly.

## 3. Core Concepts

### The Hybrid Schema
In a Hybrid Schema, the Aggregate Root (e.g., `Charger`) is a standard relational row. However, its complex, deeply nested, or rapidly changing properties (e.g., `HardwareDiagnostics`) are stored as a single JSON string in an `NVARCHAR(MAX)` column.

### Value Converters vs. Native JSON
*   **Value Converters (EF Core 3-6):** You could configure EF Core to serialize an object to JSON before saving to a string column. However, to the EF Core query pipeline, it was just a string. You could not write `Where(c => c.Diagnostics.Temperature > 100)` because EF Core couldn't translate it to SQL.
*   **Native JSON (`.ToJson()` EF7+):** EF Core maps the C# object to an Owned Type, but stores it as JSON. Crucially, EF Core *understands* the JSON structure. It translates `Where(c => c.Diagnostics.Temperature > 100)` into `WHERE CAST(JSON_VALUE([c].[Diagnostics], '$.Temperature') AS float) > 100.0`.

### The Cosmos DB Provider
EF Core is provider-agnostic. While 90% of the market uses EF Core with SQL Server/PostgreSQL, Microsoft maintains a dedicated provider for Azure Cosmos DB. This allows you to use the exact same `DbContext` and LINQ syntax, but the provider translates it into Cosmos DB NoSQL API queries, storing the entire Entity as a single JSON document.

## 4. Visual Diagrams

```text
=============================================================================
             TRADITIONAL RELATIONAL vs. JSON MAPPING
=============================================================================

[ C# Domain Model ]
class Charger { 
  Guid Id; 
  Diagnostics Diagnostics { int CpuTemp; string ErrorCode; } 
}

1. TRADITIONAL MAPPING (Owned Types to Columns)
Table: Chargers
| Id | Diagnostics_CpuTemp | Diagnostics_ErrorCode |
|----|---------------------|-----------------------|
| 1  | 65                  | NULL                  |
(Rigid schema. Adding 'FanSpeed' requires a database migration).

2. NATIVE JSON MAPPING (.ToJson)
Table: Chargers
| Id | Diagnostics (NVARCHAR(MAX))                        |
|----|----------------------------------------------------|
| 1  | {"CpuTemp": 65, "ErrorCode": null, "FanSpeed": 12} |
(Dynamic schema. Adding 'FanSpeed' in C# requires ZERO database migrations).
```

```text
=============================================================================
             JSON QUERY TRANSLATION PIPELINE
=============================================================================

C# LINQ:
context.Chargers.Where(c => c.Diagnostics.CpuTemp > 80)

       │ EF Core Query Compiler
       ▼

T-SQL:
SELECT [c].[Id], [c].[Diagnostics]
FROM [Chargers] AS [c]
WHERE CAST(JSON_VALUE([c].[Diagnostics], '$.CpuTemp') AS int) > 80
```

## 5. API Deep Dive: Native JSON Mapping

### 5.1 Configuring the JSON Column
JSON mapping utilizes the existing "Owned Types" feature. You simply append `.ToJson()` to the ownership configuration.

```csharp
public class Charger
{
    public int Id { get; set; }
    // Complex object graph
    public ChargerConfiguration Config { get; set; } 
}

public class ChargerConfiguration
{
    public int MaxKw { get; set; }
    public List<SupportedProtocol> Protocols { get; set; } // Nested Collections!
}

public class SupportedProtocol { public string Name { get; set; } }

// In DbContext OnModelCreating:
builder.Entity<Charger>().OwnsOne(
    c => c.Config, 
    ownedBuilder =>
    {
        // This single line changes everything.
        ownedBuilder.ToJson(); 
        
        // Optional: override the property name in the JSON
        ownedBuilder.Property(x => x.MaxKw).HasJsonPropertyName("max_output_kw");
        
        // EF Core 8+ supports nested collections inside the JSON!
        ownedBuilder.OwnsMany(x => x.Protocols); 
    });
```

### 5.2 Querying JSON
You write standard LINQ. EF Core handles the magic.

```csharp
// 1. Filtering by a nested JSON property
var hotChargers = await context.Chargers
    .Where(c => c.Config.MaxKw > 150)
    .ToListAsync();

// 2. Querying inside a nested JSON array (EF8+)
var ocppChargers = await context.Chargers
    .Where(c => c.Config.Protocols.Any(p => p.Name == "OCPP 2.0.1"))
    .ToListAsync();
```

### 5.3 Updating JSON
EF Core's Change Tracker monitors the JSON object. If you load the `Charger`, modify a property deep inside the `Config` object, and call `SaveChanges()`, EF Core will serialize the *entire* `Config` object and issue an `UPDATE Chargers SET Config = @p0 WHERE Id = 1`. 
*(Note: As of EF8, it updates the whole JSON string, it does not use `JSON_MODIFY` to patch specific properties).*

## 6. EF Core Internals: JSON Translation Constraints

While `.ToJson()` is powerful, it has architectural limits:
1.  **Strict Serialization:** You cannot use `[JsonIgnore]` or custom `System.Text.Json` converters on properties mapped via `.ToJson()`. EF Core uses its own internal relational mapper, not the standard .NET JSON serializer, to guarantee it understands the schema for query translation.
2.  **No Navigation Properties Out:** The classes inside the JSON column cannot contain foreign key navigation properties pointing back to standard relational tables. The JSON document is a strict boundary.
3.  **Client-Side Evaluation Risks:** If you use a C# string method inside the LINQ query that EF Core cannot translate to `JSON_VALUE`, the query will fail or evaluate locally.

## 7. Complete Examples: EV Platform Case Study

In the EV Platform, chargers send massive bursts of telemetry data. Different hardware manufacturers send completely different telemetry fields. Manufacturer A sends `{"battery_temp": 45}`, Manufacturer B sends `{"cooling_fluid_pressure": 2.1}`.

We cannot create a relational table with 500 nullable columns. We must use a Hybrid Schema.

### Domain Model
```csharp
public class TelemetrySnapshot
{
    public Guid Id { get; private set; }
    public int ChargerId { get; private set; }
    public DateTime Timestamp { get; private set; }
    
    // Dynamic payload
    public DynamicPayload Payload { get; private set; } 
}

public class DynamicPayload
{
    public decimal? BatteryTemp { get; set; }
    public decimal? CoolingPressure { get; set; }
    // Catch-all dictionary (EF Core 9 supports Dictionary to JSON)
    public Dictionary<string, string> ExtendedData { get; set; } 
}
```

### Configuration
```csharp
builder.Entity<TelemetrySnapshot>()
       .OwnsOne(t => t.Payload, p => p.ToJson());
```

### The Query
A dashboard needs to find all snapshots where Battery Temp exceeded 60 degrees.
```csharp
var criticalEvents = await _context.TelemetrySnapshots
    .Where(t => t.Payload.BatteryTemp > 60.0m)
    .OrderByDescending(t => t.Timestamp)
    .Take(100)
    .ToListAsync();
```

## 8. Performance: Indexing JSON Columns

**The Architectural Trap:**
If you execute the query above on a table with 10 million rows, SQL Server must execute `JSON_VALUE(Payload, '$.BatteryTemp')` on *every single row* to evaluate the `WHERE` clause. This is a catastrophic Full Table Scan causing massive CPU spikes.

SQL Server does not natively support indexing properties *inside* an `NVARCHAR(MAX)` JSON column.

**The Solution: Computed Columns**
To index a JSON property, the Architect must create a Computed Column in SQL Server, and then index that computed column.

```csharp
// 1. In EF Core Configuration:
builder.Entity<TelemetrySnapshot>()
    .Property<decimal?>("ComputedBatteryTemp") // Shadow property
    // Instruct SQL Server to compute this column on the fly
    .HasComputedColumnSql("CAST(JSON_VALUE(Payload, '$.BatteryTemp') AS decimal(18,2))");

// 2. Create a standard index on the computed column
builder.Entity<TelemetrySnapshot>()
    .HasIndex("ComputedBatteryTemp");
```

Now, when you query `Where(t => t.Payload.BatteryTemp > 60.0m)`, SQL Server's query optimizer is smart enough to realize that `JSON_VALUE(Payload, '$.BatteryTemp')` matches the definition of the `ComputedBatteryTemp` indexed column, and it will execute an ultra-fast Index Seek!

## 9. ASP.NET Core Integration: API Payloads

When a frontend SPA sends a JSON payload to update a Charger, the ASP.NET Core Model Binder (`System.Text.Json`) deserializes the HTTP Request body into your C# DTO.

It is critical to decouple the API's JSON representation from EF Core's JSON storage.
Never expose the EF Core mapped Owned Type directly in your API signatures. If you rename a property in C# to satisfy a frontend change, EF Core will change the JSON schema in the database, potentially corrupting historical data or breaking reporting tools reading the raw SQL column.

Always map the incoming API DTO to the EF Core Owned Type explicitly in the Application Layer.

## 10. Clean Architecture Perspective

When should an Architect use Relational Tables vs. JSON Columns?

**Use Relational Tables When:**
1.  The data is a first-class Aggregate Root (e.g., User, Invoice, Site).
2.  The data requires Foreign Key constraints to maintain referential integrity.
3.  You frequently use the data in complex `JOIN` clauses.
4.  The schema is rigid and well-understood.

**Use JSON Columns (`.ToJson()`) When:**
1.  The data is a complex Value Object strictly owned by the parent (e.g., `UserSettings`, `ShippingAddress`).
2.  The schema is highly dynamic or dictated by external 3rd-party integrations (e.g., Webhook payloads).
3.  The data is frequently read/written as a single atomic unit alongside the parent row.
4.  You have a deep inheritance hierarchy and want to avoid the JOIN penalty of TPT (Table-Per-Type), flattening the varying properties into a single JSON column on a TPH base table.

## 11. Enterprise SaaS Perspective: Dynamic Form Builders

A common SaaS requirement is allowing tenants to define their own custom fields (e.g., "Add a 'SecurityGateCode' field to the Site form").

Adding physical columns to the SQL schema dynamically at runtime is a catastrophic security and stability risk.

The enterprise solution utilizes EF Core JSON mapping. You define a `Dictionary<string, string> CustomFields` property and map it using `.ToJson()`. The application stores the tenant's dynamic form data securely within the JSON column. You can then query it using EF Core (e.g., `Where(s => s.CustomFields["SecurityGateCode"] == "1234")`), providing dynamic schemaless flexibility while maintaining a strictly compiled C# backend.

## 12. Real Production Case Study

In our EV Platform, the `Invoice` entity had a complex relationship graph. An Invoice has `InvoiceLines`, which have `TaxDetails`, which have `TaxRates`. 

Originally, these were modeled as 4 separate relational tables. When a user viewed an invoice, EF Core executed a Split Query joining 4 tables.

**The Bug:** A Tax Rate changed globally from 5% to 6%. When users viewed historical invoices from last year, the invoice dynamically recalculated using the new 6% rate, altering historical financial records (a massive compliance violation).

**The Architectural Fix (Document Snapshotting):**
An Invoice is an immutable historical record. We refactored the Domain Model. `InvoiceLine` and `TaxDetail` were changed to strictly owned Value Objects. We configured them using `.ToJson()`. 

When an Invoice is generated, the complex hierarchy is serialized into a single `DocumentData` JSON column on the `Invoices` table. 
1.  **Immutability:** The historical tax rates are permanently frozen inside the JSON string.
2.  **Performance:** Viewing an invoice now requires querying exactly 1 table and 1 row. No JOINs. No Split Queries. 
We transformed a highly relational model into a NoSQL Document model stored natively inside SQL Server, leveraging EF Core's JSON capabilities to handle the translation seamlessly.

## 13. Common Mistakes

### Beginner
*   **Mistake:** Confusing `[NotMapped]` with JSON storage.
*   **Correction:** If you add `[NotMapped]` to a complex object, EF Core ignores it entirely; it is not saved to the database. You must use `builder.OwnsOne(x => x.Prop, b => b.ToJson())` to store it as JSON.

### Intermediate
*   **Mistake:** Executing `ExecuteUpdate` on a specific JSON property.
*   **Correction:** In current versions of EF Core, `ExecuteUpdate(s => s.SetProperty(c => c.Config.MaxKw, 100))` is often unsupported or highly inefficient because EF Core must overwrite the entire JSON string, not issue a SQL Server `JSON_MODIFY` command. Avoid bulk updates on JSON internals.

### Senior
*   **Mistake:** Storing 5MB JSON documents in a SQL Server row.
*   **Correction:** SQL Server stores `NVARCHAR(MAX)` off-row if it exceeds 8KB, meaning a separate disk read is required to fetch it. Fetching a 5MB JSON string through EF Core will absolutely destroy memory and network throughput. If documents exceed a few megabytes, the Architect must move them to Azure Blob Storage and store only the Blob URI in the SQL database.

### Architect
*   **Mistake:** Using Cosmos DB (NoSQL) for the entire application because "JSON is flexible", only to realize later that the business requires complex transactional reporting and ad-hoc joins across aggregates.
*   **Correction:** Cosmos DB is incredible for massive scale and point-reads. It is terrible for relational ad-hoc querying. The Architect should utilize a Hybrid SQL Server database using EF Core JSON columns to get 80% of the flexibility of NoSQL while retaining 100% of the relational querying power of SQL.

## 14. Interview Questions

### Beginner (10)
1.  **Can SQL Server store JSON data?**
    *Answer:* Yes, typically in an `NVARCHAR(MAX)` column.
2.  **What does the `.ToJson()` method do in EF Core?**
    *Answer:* It instructs EF Core to map an Owned Type complex object graph to a single JSON string column in the database, while still allowing LINQ queries against its properties.
3.  **What was the old way of storing JSON before EF Core 7?**
    *Answer:* Using Value Converters (`.HasConversion()`), which serialized the object but prevented EF Core from translating LINQ queries into SQL JSON functions.
4.  **Can you query inside a nested JSON array using EF Core?**
    *Answer:* Yes, in EF Core 8+, you can use LINQ `.Any()` to query inside nested JSON collections.
5.  **Is the JSON column strongly typed in C#?**
    *Answer:* Yes. You interact with standard C# classes and properties. The JSON serialization is completely abstracted by EF Core.
6.  **Can an object inside a JSON column have a Foreign Key pointing to another table?**
    *Answer:* No. JSON mapped types are strictly isolated value objects.
7.  **What SQL Server function does EF Core use to extract a value from the JSON?**
    *Answer:* `JSON_VALUE`.
8.  **What is a Hybrid Database?**
    *Answer:* A database (like SQL Server) that utilizes strict relational tables for primary entities, but leverages JSON columns for dynamic or highly nested data within those tables.
9.  **If you rename a C# property mapped via `.ToJson()`, what happens to the database?**
    *Answer:* Future saves will use the new property name in the JSON. Historical data will still have the old property name, meaning queries might return nulls unless a data migration is performed.
10. **How do you explicitly set the JSON property name?**
    *Answer:* `ownedBuilder.Property(x => x.Prop).HasJsonPropertyName("custom_name");`

### Intermediate (10)
11. **Explain the performance risk of `Where(c => c.JsonPayload.Status == "Active")`.**
    *Answer:* SQL Server must evaluate the `JSON_VALUE` function on every row in the table, resulting in a full table scan and massive CPU usage.
12. **How do you index a specific property inside a JSON column in SQL Server?**
    *Answer:* You cannot index it directly. You must create a Computed Column in SQL Server that extracts the `JSON_VALUE`, and then create a standard non-clustered index on that computed column.
13. **How do you map a Dictionary to a JSON column?**
    *Answer:* In EF Core 9+, standard dictionaries map natively. In older versions, you must use a Value Converter (sacrificing query translation) or map to a list of KeyValuePair objects.
14. **Why might you use JSON storage instead of Table-Per-Type (TPT) inheritance?**
    *Answer:* TPT requires a `LEFT JOIN` for every derived type, degrading performance. Flattening the hierarchy and storing the varying derived properties in a single JSON column on the base table eliminates all JOINs.
15. **Does EF Core use `System.Text.Json` to serialize the `.ToJson()` columns?**
    *Answer:* No. It uses its own internal relational mapper to ensure tight integration with the query translation engine. You cannot easily customize the serialization rules using standard `[JsonIgnore]` attributes.
16. **Can you map a primitive collection (e.g., `List<int>`) directly to a column without `.ToJson()`?**
    *Answer:* Yes, EF Core 8 introduced Primitive Collections mapping. It stores `List<int>` natively as a JSON array in SQL Server without needing a complex Owned Type wrapper.
17. **What is the Cosmos DB Provider in EF Core?**
    *Answer:* A database provider that translates LINQ and DbContext operations into Azure Cosmos DB NoSQL queries, mapping entire aggregates to Cosmos DB JSON documents.
18. **Why is Change Tracking difficult for massive JSON columns?**
    *Answer:* EF Core must compare the current JSON object graph against the snapshot it took when the entity was loaded to detect changes. For very deep/large JSON structures, this `DetectChanges` process consumes significant CPU time.
19. **If you use a Value Converter instead of `.ToJson()`, what happens if you query `Where(x => x.Json.Id == 1)`?**
    *Answer:* The EF Core compiler cannot translate it. It will either throw an exception or execute client-side (fetching all rows into RAM and filtering in C#).
20. **Is it possible to use `.ToJson()` with SQLite?**
    *Answer:* Yes, EF Core supports JSON mapping for SQLite, translating LINQ into SQLite's native `json_extract` functions.

### Senior (10)
21. **Architect a mechanism to handle schema evolution within a `.ToJson()` mapped column.**
    *Answer:* JSON schemas drift. If Property A was an `int` and becomes a `string`, EF Core will throw deserialization errors reading historical rows. The Architect must implement defensive C# properties. Map the raw JSON to loosely typed or dynamic internal properties, and expose strictly typed calculated properties on the domain model that can gracefully handle historical data variations (e.g., parsing a string to an int if necessary).
22. **Evaluate the architectural choice of using Cosmos DB vs. SQL Server with JSON columns for an Event Sourcing Event Store.**
    *Answer:* Cosmos DB is vastly superior for a raw Event Store. It handles massive append-only write throughput infinitely better than SQL Server, and its Change Feed provides native event streaming. SQL Server with JSON is an awkward fit for an Event Store because relational databases struggle with massive lock contention on append-only un-indexed massive tables.
23. **A query against a Computed Column index over a JSON property is still performing a Table Scan. Why?**
    *Answer:* The EF Core LINQ query expression might not perfectly match the SQL Server Computed Column definition. For example, if the Computed Column is `CAST(JSON_VALUE(...) AS decimal(18,2))` but the LINQ query uses a C# `double`, EF Core generates `CAST(... AS float)`. SQL Server sees a type mismatch and cannot use the index. The definitions must match flawlessly.
24. **Design a solution to implement full-text search across all nested string properties within a complex JSON column using EF Core.**
    *Answer:* SQL Server Full-Text Indexes cannot natively parse JSON structure intelligently (it just treats it as a massive string). The Architect must instruct SQL Server to index the `NVARCHAR(MAX)` column using a Full-Text Index. In EF Core, you must drop down to raw SQL or use `EF.Functions.FreeText()` if the provider supports it, acknowledging that the search will hit keys as well as values. For true intelligent JSON search, export the data to Azure AI Search.
25. **Explain how EF Core 8's Primitive Collection mapping translates `context.Users.Where(u => u.Tags.Contains("Admin"))` into SQL Server syntax.**
    *Answer:* It uses the `OPENJSON` function. The generated SQL looks something like: `WHERE EXISTS (SELECT 1 FROM OPENJSON([u].[Tags]) WITH ([Value] nvarchar(max) '$') AS [t] WHERE [t].[Value] = 'Admin')`. This is incredibly powerful as it evaluates the array contents on the database server.
26. **What is the `HasConversion` workaround for storing JSON in EF Core 3-6, and why is it dangerous?**
    *Answer:* You define a conversion: `v => JsonSerializer.Serialize(v), v => JsonSerializer.Deserialize(v)`. The danger is that developers write LINQ queries against the object properties, assuming they will execute on SQL Server. In EF Core 3+, this throws a runtime exception, breaking APIs.
27. **How does EF Core handle `NULL` values when mapping C# objects to JSON columns?**
    *Answer:* By default, missing keys in the database JSON evaluate to `null` in C#. If the entire database column is `NULL`, EF Core instantiates the Owned Type as `null`. You must defensively check for nulls before accessing JSON properties in C#.
28. **In an Enterprise SaaS, you allow tenants to define custom schemas. How do you map this in EF Core so that you can still query the dynamic fields?**
    *Answer:* Define a `Dictionary<string, JsonElement>` (or similar raw type) and map it using `.ToJson()`. When querying, you must use specialized LINQ extensions or EF Functions (if provided) or drop down to raw SQL `JSON_VALUE(CustomFields, '$.TenantField')` via `FromSqlInterpolated` to query the unknown dynamic keys.
29. **Evaluate the locking implications of updating a 1MB JSON column in SQL Server.**
    *Answer:* Updating a large `NVARCHAR(MAX)` column is an expensive operation. It consumes significant transaction log space and holds exclusive row locks longer than updating an `INT`. In highly concurrent systems, frequent updates to massive JSON columns will cause severe blocking and deadlocks. The Architect should segregate volatile data into standard relational columns and reserve JSON for mostly-read/rarely-updated document payloads.
30. **How does the Cosmos DB provider handle relationships (`Include`)?**
    *Answer:* It doesn't. Cosmos DB has no concept of SQL JOINs. If you configure a relationship, the Cosmos provider typically embeds the child entities as a nested JSON array within the parent document. If you try to enforce relational behavior (separate documents requiring a JOIN), EF Core will throw an exception or evaluate the join locally (catastrophically slow).

### Staff Engineer (5)
31. **Architect a CQRS system where EF Core writes to SQL Server, and a mechanism automatically syncs the data to Cosmos DB to serve as the high-speed Read Model. Detail the synchronization mechanism and latency guarantees.**
    *Answer:* The Architect implements the Outbox Pattern in EF Core. The Command transaction saves the relational data and an `OutboxMessage` containing the serialized aggregate state. A background worker (e.g., Azure Function) polls the Outbox (or uses SQL Server Change Tracking/CDC) and upserts the JSON document into Cosmos DB. This guarantees Eventual Consistency. The latency is dictated by the polling interval (typically 1-5 seconds). The Read API queries Cosmos DB natively using the Cosmos SDK (bypassing EF Core) for sub-10ms response times.
32. **A legacy system stores JSON in a `VARCHAR(MAX)` column. A new EF Core `.ToJson()` mapping fails randomly on certain historical rows. Diagnose the root cause at the database engine level.**
    *Answer:* SQL Server does not enforce JSON validity on `VARCHAR(MAX)` columns unless explicitly configured with a `CHECK (ISJSON(column) = 1)` constraint. The legacy system likely inserted truncated, malformed, or invalid JSON strings (e.g., unescaped quotes). When EF Core's materializer attempts to parse this invalid JSON using its strict internal reader, it throws an exception. The Architect must run a data cleansing script using `ISJSON()` to find and fix the corrupted rows before EF Core can safely map the column.
33. **Design a strategy to migrate a deeply nested Table-Per-Type (TPT) inheritance hierarchy (which is causing massive JOIN performance issues) into a single JSON-mapped TPH structure, ensuring zero downtime during the migration.**
    *Answer:* (Expand and Contract). 1. Add an `NVARCHAR(MAX)` JSON column to the Base table. 2. Update the Application Layer to write the derived properties to *both* the old TPT tables and the new JSON column (Dual Write). 3. Write a background SQL script to backfill historical TPT data into the JSON column. 4. Update the EF Core model to use `.ToJson()` mapping on the base table and remove the TPT configurations, instructing it to read from the JSON. 5. Deploy. 6. Drop the old derived TPT tables.
34. **Analyze the execution plan differences between `context.Chargers.Where(c => c.Config.MaxKw == 50)` running on SQL Server (using `JSON_VALUE`) versus running on PostgreSQL (using `jsonb`).**
    *Answer:* SQL Server parses the `NVARCHAR` string at runtime using `JSON_VALUE`, requiring a computed column for efficient indexing. PostgreSQL supports the `jsonb` data type, which stores data in a decomposed binary format. EF Core translates the query to `Config->>'MaxKw'`. Because PostgreSQL natively supports GIN (Generalized Inverted Indexes) on `jsonb` columns, it can execute an index seek directly against the JSON structure without requiring explicit computed columns, making PostgreSQL vastly superior for dynamic JSON querying workloads.
35. **Evaluate the architectural limitations of using the EF Core Cosmos DB Provider for a complex Domain-Driven Design aggregate that requires strict cross-aggregate transactional consistency.**
    *Answer:* It is a fundamental mismatch. Cosmos DB guarantees ACID transactions *only within a single logical partition key*. If an EF Core transaction attempts to `SaveChanges()` containing modifications to multiple Aggregates that reside in different Cosmos DB partitions (or different containers), the operation will fail or execute without atomic guarantees. The Architect must strictly design the Cosmos partition strategy to encapsulate the transactional boundary, or abandon EF Core's implicit transactions in favor of a distributed Saga pattern.

## 15. Exercises

### Easy
1.  **Map a JSON Object:** Create a `UserSettings` class containing `Theme` (string) and `NotificationsEnabled` (bool). Add it as a property to a `User` entity. Use the Fluent API `.OwnsOne().ToJson()` to map it. Generate a migration and inspect the `Up()` method to see the column type.

### Medium
1.  **JSON Querying:** Write a LINQ query to fetch all users where `UserSettings.Theme == "Dark"`. Execute it and inspect the generated SQL string using EF Core logging to observe the `JSON_VALUE` translation.
2.  **Primitive Collections:** Add a `List<string> Tags` property to the `User` entity. Map it (no special config needed in EF8+). Seed a user with tags, then write a query: `context.Users.Where(u => u.Tags.Contains("VIP"))`. Inspect the generated SQL to observe the `OPENJSON` usage.

### Hard
1.  **Computed Column Indexing:** Write raw SQL in a migration to create a Computed Column that extracts the `Theme` value from the JSON column. Then, use EF Core configuration to create a standard Index on that shadow property.

### Enterprise
1.  **The Hybrid Aggregate:** Model an `Order` aggregate. The `Order` has standard columns (`Id`, `Total`, `Date`). It also has an `OrderContext` JSON column containing highly dynamic data (e.g., `BrowserUserAgent`, `ReferringCampaignId`, `GeoLocationData`). Build the API endpoint to create this order, ensuring the dynamic payload is passed cleanly from a generic API `Dictionary` into the strictly typed (or dynamic) JSON internal property.

## 16. Production Checklist

- [ ] Are JSON columns strictly reserved for dynamic, unstructured, or isolated Value Objects, preventing relational normalization decay?
- [ ] Are EF Core `.ToJson()` mappings favored over manual Value Converters to enable native SQL query translation?
- [ ] Have Computed Columns been implemented and indexed for any JSON properties that are frequently used in `WHERE` clauses?
- [ ] Are JSON payloads protected against unbounded growth (e.g., exceeding megabytes) to prevent memory and network saturation?
- [ ] Is the data isolated from ASP.NET Core API DTOs, ensuring API contract changes do not accidentally alter the database JSON schema?

## 17. Summary

The Hybrid Database is the pinnacle of modern data architecture. By mastering EF Core's native JSON integration, you no longer have to choose between the safety of SQL Server and the flexibility of Document databases. You can build strict, transactional aggregates that gracefully encapsulate dynamic, rapidly evolving payloads.

We have traversed the entire spectrum of Entity Framework Core—from the basics of DbContext instantiation, through the depths of the LINQ translation pipeline, into the battleground of high-performance concurrency, and finally into advanced hybrid architectures.

In the final chapter, we will consolidate all this knowledge into the **Master Architect's Playbook**, providing a comprehensive, quick-reference guide to debugging production issues, structuring enterprise solutions, and leading development teams to EF Core mastery.
