# Chapter 15: Handling JSON & XML

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand when to use relational tables versus semi-structured JSON storage.
*   Store, query, and extract data from JSON payloads using `JSON_VALUE` and `JSON_QUERY`.
*   Index JSON properties for high-performance searching using Computed Columns.
*   Generate API-ready JSON payloads directly from the database engine using `FOR JSON PATH`.
*   Map JSON columns to C# objects natively using EF Core 7+ `ToJson()`.

---

## 15.1 Introduction to Semi-Structured Data

The EV charging protocol (OCPP) frequently evolves. New hardware vendors send custom telemetry metrics (e.g., "InternalTemperature", "CableResistance"). 
If we add a new column to our `core.Sessions` table every time a vendor introduces a new metric, our table will quickly have 300 mostly-NULL columns. This is known as "Sparse Data."

For sparse, highly variable data, we embrace **Semi-Structured Storage** by saving the raw payload as JSON.

---

## 15.2 Storing JSON Data

Unlike PostgreSQL (which has a dedicated `jsonb` data type), SQL Server stores JSON as standard text.
To optimize space, we use `NVARCHAR(MAX)` only if necessary, or `VARCHAR(MAX)` if we guarantee the JSON payload contains no Unicode characters.

To guarantee data integrity, we apply a `CHECK` constraint using the `ISJSON()` function.

```sql
ALTER TABLE core.Sessions
ADD CustomMetrics VARCHAR(MAX) NULL;

-- Ensure nobody inserts malformed XML or plain text into this column
ALTER TABLE core.Sessions
ADD CONSTRAINT CHK_Sessions_Json 
CHECK (CustomMetrics IS NULL OR ISJSON(CustomMetrics) = 1);
```

---

## 15.3 Querying JSON

SQL Server provides built-in functions to parse JSON strings at runtime.

### `JSON_VALUE` (Extracting Scalar values)
Use `JSON_VALUE` to extract a single string, number, or boolean.

```sql
-- Assuming CustomMetrics = {"Vendor":"Tesla", "MaxTemp": 85.5}
SELECT 
    SessionId,
    JSON_VALUE(CustomMetrics, '$.Vendor') AS HardwareVendor,
    CAST(JSON_VALUE(CustomMetrics, '$.MaxTemp') AS DECIMAL(5,2)) AS MaxTemperature
FROM core.Sessions
WHERE TenantId = 'T1-UUID' 
  AND JSON_VALUE(CustomMetrics, '$.Vendor') = 'Tesla';
```

### `JSON_QUERY` (Extracting Objects/Arrays)
If you need to extract a nested JSON object or a JSON Array (not a single scalar value), you must use `JSON_QUERY`. `JSON_VALUE` will return NULL if you point it at an array.

---

## 15.4 Indexing JSON Data

**The Architect's Problem:** Look at the `WHERE` clause in Section 15.3. 
`WHERE JSON_VALUE(CustomMetrics, '$.Vendor') = 'Tesla'`
We applied a function to a column in the `WHERE` clause. Based on Chapter 5 (SARGability), we know this will cause a massive **Index Scan**.

To fix this, we cannot index the JSON string directly. Instead, we create a **Computed Column**, and then index that column.

```sql
-- Step 1: Create a non-persisted computed column
ALTER TABLE core.Sessions
ADD VendorName AS CAST(JSON_VALUE(CustomMetrics, '$.Vendor') AS VARCHAR(50));

-- Step 2: Create a Non-Clustered Index on the computed column
CREATE INDEX IX_Sessions_VendorName ON core.Sessions(VendorName);
```
Now, if you query `WHERE VendorName = 'Tesla'`, SQL Server will perform a blazing-fast **Index Seek**.

---

## 15.5 Formatting SQL Results as JSON

In modern microservices, the API layer (ASP.NET Core) queries the database, converts the rows to C# Objects, serializes them to JSON, and sends them to the client.
If you need to export millions of rows, allocating those C# objects in RAM will cause severe Garbage Collection pauses.

SQL Server can generate the JSON string directly, allowing the API to simply stream the raw text to the client.

```sql
SELECT 
    SessionId,
    TotalCost,
    st.Name AS [Station.Name] -- The dot notation creates nested JSON objects!
FROM core.Sessions s
INNER JOIN core.Stations st ON s.StationId = st.StationId
WHERE s.TenantId = 'T1-UUID'
FOR JSON PATH, ROOT('Sessions');
```
*Output:*
```json
{
  "Sessions": [
    {
      "SessionId": "F9168C5E...",
      "TotalCost": 15.50,
      "Station": {
        "Name": "Lobby Charger"
      }
    }
  ]
}
```

---

## 15.6 A Note on XML

Before JSON dominated the web, SOAP and XML were the standard. Many legacy enterprise systems still rely on it.
Unlike JSON, SQL Server *does* have a native `XML` data type.

```sql
-- Extracting data from XML using XQuery
SELECT 
    SessionId,
    XmlPayload.value('(/SessionData/Vendor)[1]', 'VARCHAR(50)') AS VendorName
FROM core.LegacySessions;
```
*Architect Rule:* If building a new SaaS today, strictly use JSON. Only use the `XML` data type when integrating with legacy SOAP APIs.

---

## 15.7 The Code: EF Core JSON Mapping

Historically, developers had to create custom Value Converters to serialize/deserialize JSON strings in EF Core.
As of **EF Core 7**, JSON columns are natively supported using the `.ToJson()` mapping.

```csharp
// 1. The C# Classes
public class Session
{
    public Guid SessionId { get; set; }
    public decimal TotalCost { get; set; }
    public SessionMetrics Metrics { get; set; } // The JSON object
}

public class SessionMetrics
{
    public string Vendor { get; set; }
    public decimal MaxTemp { get; set; }
}

// 2. The EF Core Configuration
public void Configure(EntityTypeBuilder<Session> builder)
{
    builder.OwnsOne(s => s.Metrics, metricsBuilder =>
    {
        // Instructs EF Core to store this object as a JSON string
        metricsBuilder.ToJson();
    });
}

// 3. Querying (EF Core automatically translates this to JSON_VALUE!)
var teslaSessions = await context.Sessions
    .Where(s => s.Metrics.Vendor == "Tesla")
    .ToListAsync();
```

---

## 15.8 Performance & Security Analysis

### Performance Analysis: The 8KB Limit
JSON strings are stored in `VARCHAR(MAX)`. If the JSON string is small (under 8KB), SQL Server stores it "In-Row". If it grows larger than 8KB, SQL Server moves it to "Out-Of-Row" LOB (Large Object) storage. Reading LOB data is significantly slower than reading In-Row data. **Keep your JSON payloads small.**

### Security Implications
*   **JSON Injection:** If you manually construct JSON strings in T-SQL via concatenation (e.g., `'{ "name": "' + @Name + '" }'`), a user passing a name like `" } , { "admin": true ` can alter the payload structure. Always use `FOR JSON` to construct JSON safely, as it automatically escapes illegal characters.

---

## 15.9 Common Mistakes & Production Pitfalls

1.  **Over-using JSON:** Developers who prefer MongoDB often treat SQL Server like a Document DB, storing everything in massive JSON columns. This destroys the Query Optimizer's ability to create efficient Execution Plans, makes Foreign Keys impossible, and eliminates relational data integrity. **Only use JSON for sparse, unpredictable, or append-only telemetry data.**
2.  **Case Sensitivity:** JSON keys are strictly case-sensitive. `JSON_VALUE(Col, '$.vendor')` will return NULL if the actual JSON key is `"Vendor"`.

---

## 15.10 Production Checklist

*   [ ] Highly variable, sparse data (like IoT sensor metrics) is stored in JSON columns rather than 50 nullable relational columns.
*   [ ] JSON columns have an `ISJSON()` CHECK constraint applied to guarantee structural integrity.
*   [ ] Properties inside the JSON that are frequently used in `WHERE` clauses are promoted to Computed Columns and Indexed.
*   [ ] EF Core configurations utilize `.ToJson()` for seamless querying.

---

## 15.11 Exercises

1.  **JSON Extraction:** Write a `SELECT` statement that extracts the nested value "Firmware" from the following JSON structure stored in `CustomMetrics`: 
    `{ "Hardware": { "Firmware": "v1.2" } }`
2.  **Computed Column Indexing:** Write the T-SQL to add a computed column named `FirmwareVersion` based on the extraction in Exercise 1, and create a Non-Clustered index on it to make searches SARGable.

---

## 15.12 Interview Questions

**Q1: In SQL Server, how do you index a specific property inside a JSON string to ensure queries filtering on that property perform Index Seeks instead of Index Scans?**
*Answer:* SQL Server does not allow you to index a JSON string directly. You must first create a Computed Column that uses `JSON_VALUE` to extract the specific property from the JSON. Then, you create a standard Non-Clustered Index on that computed column. When you query the computed column, the optimizer uses the index.

**Q2: What is the architectural difference between using `JSON_VALUE` and `JSON_QUERY`?**
*Answer:* `JSON_VALUE` is designed strictly for extracting scalar values (strings, integers, booleans). If you point it at an object or an array, it returns NULL. `JSON_QUERY` is designed to extract complex types (objects or arrays).

---
**Next up in Chapter 16:** We enter Part 5 of the book, which covers the heart of the database engine: Transactions, Concurrency, and Internals. We will start with the ACID properties and explicit transaction management.
