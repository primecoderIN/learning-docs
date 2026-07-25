# Chapter 25: Concurrency Control

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the "Lost Update" problem in distributed, stateless web applications.
*   Contrast Pessimistic Concurrency with Optimistic Concurrency.
*   Implement Optimistic Concurrency in EF Core using SQL Server's `RowVersion` data type.
*   Catch and handle `DbUpdateConcurrencyException` to provide graceful conflict resolution to the end user.

---

## 25.1 The "Lost Update" Problem

Imagine two Tenant Administrators (Alice and Bob) are managing the same EV Charging Station via your React Dashboard.
1.  **10:00 AM:** Alice opens the Station Configuration page. She sees the Station is named "Lobby Charger" and is set to **Offline**.
2.  **10:01 AM:** Bob opens the exact same page on his laptop. He also sees "Lobby Charger" / **Offline**.
3.  **10:02 AM:** Alice changes the status to **Online** and clicks "Save". The database updates.
4.  **10:03 AM:** Bob changes the name to "Main Lobby Charger" but leaves his screen's dropdown on **Offline**. He clicks "Save".

*Result:* Bob's `UPDATE` statement overwrites Alice's changes. The station is now named "Main Lobby Charger", but its status has been forced back to **Offline**. 
Alice's update was completely lost. This is the **Lost Update Problem**.

---

## 25.2 Pessimistic Concurrency

In the 1990s, desktop applications solved this using **Pessimistic Concurrency**.
When Alice opened the record at 10:00 AM, the application would tell the database to place an Exclusive (X) lock on that row. When Bob tried to open the record at 10:01 AM, the database would block him, and he would see a message: *"Record locked by Alice."*

**Why this fails in the Cloud:**
Modern Web APIs (REST/GraphQL) are **Stateless**. Alice opens the page, the API serves the data, and the HTTP request immediately ends. The API cannot hold a database lock open while Alice goes to lunch. If it did, your connection pool would exhaust instantly, and a dropped Wi-Fi connection would leave a row locked forever.

Therefore, we cannot use Pessimistic Concurrency in web applications.

---

## 25.3 Optimistic Concurrency

The modern architectural standard is **Optimistic Concurrency**.
We are "optimistic" that collisions are rare. We allow Alice and Bob to read the data simultaneously. We do not lock anything. Instead, we enforce a rule during the `UPDATE`:

*You can only update this record if nobody else has changed it since you read it.*

### How it works: Concurrency Tokens
To enforce this, we add a hidden column to our table called a **Concurrency Token**. In SQL Server, this is the `ROWVERSION` (formerly `TIMESTAMP`) data type. 
SQL Server automatically generates a new, unique binary sequence in this column every single time the row is `INSERTED` or `UPDATED`.

1.  Alice reads the row. `RowVersion = 0x01`.
2.  Bob reads the row. `RowVersion = 0x01`.
3.  Alice saves. EF Core generates:
    `UPDATE Stations SET Status = 'Online' WHERE StationId = 1 AND RowVersion = 0x01;`
    *Result: Success. SQL Server auto-increments the row's RowVersion to `0x02`.*
4.  Bob saves. EF Core generates:
    `UPDATE Stations SET Name = 'Main Lobby Charger' WHERE StationId = 1 AND RowVersion = 0x01;`
    *Result: Failure! The `WHERE` clause finds 0 rows (because the DB RowVersion is now `0x02`).*

When EF Core executes an `UPDATE` and 0 rows are affected, it throws a **`DbUpdateConcurrencyException`**.

---

## 25.4 The Code: EF Core Implementation

### 1. Database Schema & Fluent API
First, add the byte array to your Entity, and configure it as a Concurrency Token in the Fluent API.

```csharp
public class Station
{
    public Guid StationId { get; set; }
    public string Name { get; set; }
    public string Status { get; set; }
    
    // The Concurrency Token
    public byte[] Version { get; set; } 
}

public void Configure(EntityTypeBuilder<Station> builder)
{
    builder.Property(s => s.Version)
           .IsRowVersion(); // Instructs EF Core to use this for Optimistic Concurrency
}
```

### 2. The API Controller
Your UI (React/Angular) must store the `Version` it received when it loaded the page, and send that *exact same `Version` back* in the PUT request.

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> UpdateStation(Guid id, StationUpdateDto dto)
{
    var station = await _context.Stations.FindAsync(id);
    
    station.Name = dto.Name;
    station.Status = dto.Status;
    
    // CRITICAL: We overwrite the DB's current version with the version the Client sent us.
    // This primes the EF Core Change Tracker to use the Client's version in the WHERE clause.
    _context.Entry(station).Property("Version").OriginalValue = dto.ClientVersion;

    try
    {
        await _context.SaveChangesAsync();
        return Ok();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Bob hit the exception!
        return Conflict(new { Message = "Another user modified this record before you. Please refresh." });
    }
}
```

---

## 25.5 Architect Perspective: Conflict Resolution

Returning a `409 Conflict` (as shown above) is known as the **"Client Wins / Abort"** strategy. It forces Bob to refresh his screen and try again. This is acceptable for simple configuration screens.

For complex, high-value transactions (e.g., merging code in Git, or collaborative document editing), architects must implement more complex resolution strategies:
1.  **Store Wins:** The database always wins. (The default behavior if no concurrency token is used—the Lost Update).
2.  **Client Wins:** You catch the exception, update the `OriginalValue` to match the current DB version, and call `SaveChanges` again, intentionally crushing Alice's changes.
3.  **Merge (The hardest):** You catch the exception, read Alice's new values from the DB, and attempt to mathematically merge them with Bob's values before saving.

---

## 25.6 Performance & Security Analysis

### Performance Analysis
Optimistic Concurrency has zero read-blocking overhead. The `ROWVERSION` column adds 8 bytes to every row, which is negligible. The only performance cost is the exception handling during a collision, but because we are *optimistic* that collisions are rare, the overall system throughput is vastly superior to Pessimistic locking.

### Security Implications
*   **Tampering:** The `ROWVERSION` is a binary array. If a malicious user manipulates the `ClientVersion` in their HTTP payload to try and bypass concurrency controls, the EF Core `WHERE` clause will simply fail to find a match, treating it exactly like a concurrency collision and rejecting the update. It is tamper-proof by design.

---

## 25.7 Common Mistakes & Production Pitfalls

1.  **Using `DateTime` for Concurrency:** 
    A common mistake is using a `LastModifiedUtc` `DATETIME2` column as a concurrency token. If two requests hit the database in the exact same millisecond, the `DateTime` might not have enough precision to differentiate them, allowing a Lost Update. SQL Server's `ROWVERSION` is guaranteed to be mathematically unique and sequential across the entire database.
2.  **Forgetting to pass the token:** The entire pattern fails if the frontend React application does not store the `Version` array and pass it back in the `PUT/POST` payload.

---

## 25.8 Production Checklist

*   [ ] Highly collaborative entities (e.g., Station Configurations, User Roles) include a `byte[] Version` property mapped via `.IsRowVersion()`.
*   [ ] The API's `PUT` endpoints expect a `ClientVersion` from the frontend and explicitly map it to the EF Core `OriginalValue`.
*   [ ] `DbUpdateConcurrencyException` is globally caught (either in the controller or a global exception middleware) and translated into a clean `HTTP 409 Conflict` response.
*   [ ] `DateTime` columns are *never* used as the sole concurrency token.

---

## 25.9 Exercises

1.  **Code Tracing:** In the code example in Section 25.4, why is it absolutely mandatory to execute `_context.Entry(station).Property("Version").OriginalValue = dto.ClientVersion;`? What would happen to Bob's update if we forgot that line?
2.  **Resolution Logic:** Bob submits an update to a Station, and a `DbUpdateConcurrencyException` is thrown. Write the C# catch block logic that implements a "Client Wins" strategy (forcing Bob's changes into the database regardless of Alice's changes). *Hint: You will need to use `ex.Entries.Single().GetDatabaseValues()`.*

---

## 25.10 Interview Questions

**Q1: Explain the difference between Pessimistic and Optimistic concurrency control, and why Pessimistic is inappropriate for REST APIs.**
*Answer:* Pessimistic concurrency assumes collisions are frequent and prevents them by locking the database row when a user reads it, blocking all other writers. Optimistic concurrency assumes collisions are rare; it allows concurrent reads and checks for modifications only at the moment of the `UPDATE` using a version token. Pessimistic concurrency is inappropriate for REST APIs because HTTP is stateless. The server cannot know if the user closed their browser or lost connection. If a lock was held across HTTP requests, the database would quickly suffer from orphaned locks, exhausting the connection pool and freezing the application.

**Q2: How does Entity Framework Core implement Optimistic Concurrency under the hood?**
*Answer:* EF Core implements it by injecting the original concurrency token (like a `RowVersion`) into the `WHERE` clause of the generated `UPDATE` or `DELETE` statement. When `SaveChanges` is called, EF Core executes `UPDATE Table SET Col = 'New' WHERE Id = 1 AND RowVersion = @OriginalVersion`. If another user modified the row first, the database's `RowVersion` will have changed, so the `WHERE` clause will affect 0 rows. EF Core detects that 0 rows were affected and throws a `DbUpdateConcurrencyException`.

---
**Next up in Chapter 26:** We will wrap up Part 7 by tackling the most difficult aspect of Entity Framework Core: testing. We will discuss why the InMemory Database provider is a trap, and how to architect Integration Tests using Docker and Testcontainers.
