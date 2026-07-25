# Chapter 34: Event Sourcing & CQRS

## Learning Objectives
By the end of this chapter, you will be able to:
*   Understand the fundamental flaw of State-Based persistence (destructive updates).
*   Architect an **Event Sourced** system where data is stored as an append-only log of immutable events.
*   Implement **CQRS (Command Query Responsibility Segregation)** to separate Write workloads from Read workloads.
*   Design Event Projections to build highly optimized Read Models.
*   Embrace the architectural reality of **Eventual Consistency**.

---

## 34.1 The Flaw in State-Based Persistence

For the past 33 chapters, we have built a **State-Based** database. 
If a charging station goes offline, we run: `UPDATE core.Stations SET Status = 'Offline'`.

**The Flaw:** `UPDATE` is a destructive operation. We have permanently erased the fact that it was 'Online' 5 minutes ago. 
When the CEO asks, *"How many times did the Berlin station go offline last year?"*, we cannot answer. The data is gone. (Temporal Tables from Chapter 4 are a bandage for this, but they are incredibly slow to query for complex business logic).

---

## 34.2 Introduction to Event Sourcing

To fix this, Architects use **Event Sourcing**.
In an Event Sourced system, **you never run an `UPDATE` or `DELETE` statement.**

Instead of storing the *current state* of a Station, you store the *history of events* that happened to it.
Your table is an append-only log:
1. `StationCreatedEvent (Name: Berlin, Status: Online)`
2. `StationFaultedEvent (ErrorCode: 502)`
3. `StationRepairedEvent ()`

To find out the current status of the Station, you query all its events from the database, load them into C# memory, and "replay" them in chronological order. 
*   *Start:* Online.
*   *Next:* Faulted (Now it's Offline).
*   *Next:* Repaired (Now it's Online).
*   *Result:* The station is currently Online.

This provides a perfect, mathematically provable audit trail. You can answer *any* question about the past.

---

## 34.3 CQRS (Command Query Responsibility Segregation)

Replaying 10,000 events just to see if a Station is Online is terribly slow. You cannot build a UI Dashboard by replaying events for 5,000 stations every time a user refreshes the page.

Event Sourcing requires **CQRS**.
CQRS dictates that the system must have two distinct sides:
*   **The Command (Write) Side:** This is the Event Store. It only appends events. It is optimized for 100,000 writes per second.
*   **The Query (Read) Side:** This is the Read Model. It stores data exactly as the UI needs to see it. It is optimized for 100,000 reads per second.

---

## 34.4 Event Projections (The Read Model)

How does the data get from the Write Side to the Read Side? **Projections**.

When a `StationFaultedEvent` is saved to the Event Store, a background worker picks up that event and updates the Read Model database. It essentially executes `UPDATE ReadModel.Stations SET Status = 'Offline'`.

**Why is this brilliant?**
1.  **Tailored Models:** You can create 5 different Read Models from the exact same Event Stream. One for the UI, one for the Data Warehouse, one for ElasticSearch.
2.  **Rebuildability:** If a developer introduces a bug that corrupts the Read Model, you just delete the entire Read Model database, press a button to replay all events from the beginning of time, and regenerate a perfect Read Model in minutes.

---

## 34.5 Architect Perspective: Eventual Consistency

CQRS introduces a massive architectural paradigm shift: **Eventual Consistency**.

Because the Projection runs in the background, there is a delay (usually 50ms, but sometimes seconds) between writing an event and the Read Model updating.
1. The user clicks "Take Station Offline".
2. The Event is saved.
3. The UI instantly redirects to the Dashboard.
4. *The Dashboard still shows the station as Online!*

The background projection hasn't finished yet. The system is "eventually" consistent.
Architects must train frontend teams to handle this (e.g., using SignalR websockets to push the update to the UI when the projection finishes, or optimistically updating the UI in Javascript).

---

## 34.6 The Code: Event Sourcing in EF Core

While dedicated databases like EventStoreDB exist, you can build an Event Store in SQL Server using EF Core.

```csharp
// 1. The Append-Only Table
public class EventStream
{
    public Guid AggregateId { get; set; } // The Station Id
    public int Version { get; set; }      // Optimistic Concurrency Token!
    public string EventType { get; set; } // "StationFaultedEvent"
    public string EventDataJson { get; set; } 
    public DateTime CreatedAtUtc { get; set; }
}

// 2. Appending an Event (The Command Side)
public async Task HandleFault(Guid stationId, int currentVersion)
{
    var newEvent = new EventStream 
    {
        AggregateId = stationId,
        Version = currentVersion + 1, // Enforces sequence
        EventType = "StationFaultedEvent",
        EventDataJson = "{ 'ErrorCode': 502 }"
    };
    
    _context.EventStreams.Add(newEvent);
    // If two threads try to write Version 5 simultaneously, 
    // SQL Server's Unique Constraint on (AggregateId, Version) will throw an exception,
    // protecting us from concurrency bugs!
    await _context.SaveChangesAsync(); 
}
```

---

## 34.7 Performance & Security Analysis

### Performance Analysis: Snapshots
If a Station has been online for 10 years, it might have 500,000 events. Loading and replaying 500,000 events in C# memory to figure out its current state (to validate a new Command) will cause massive CPU/RAM spikes and latency. 
*The Fix:* Implement **Snapshots**. Every 1,000 events, you serialize the calculated state and save it. When you need the state, you load the Snapshot, and only replay the events that occurred *after* the snapshot.

### Security Implications
*   **The "Right to be Forgotten" (GDPR):** Event Sourcing creates an immutable ledger. You cannot run `DELETE`. If a user requests their PII (Personally Identifiable Information) be deleted, you are legally trapped. 
*   *The Architect's Fix (Crypto-Shredding):* You encrypt the PII in the event payload using a unique symmetric key per user. When the user requests deletion, you simply delete their encryption key from Azure Key Vault. The immutable events remain in the database, but the PII is permanently mathematically scrambled (crypto-shredded).

---

## 34.8 Common Mistakes & Production Pitfalls

1.  **Overusing Event Sourcing:** Junior architects read about Event Sourcing and try to apply it to everything. Do not event-source your `Lookup_Countries` table. Only use Event Sourcing for core business domains where the history and audit trail provide massive financial or operational value (e.g., Financial Ledgers, Shopping Carts, IoT Telemetry status).
2.  **Modifying Past Events:** A developer discovers a bug in how an event was written 3 months ago, and tries to write a SQL script to `UPDATE` the JSON payload of that old event. *Never do this.* It destroys the cryptographic integrity of the stream. You must write a new "Correction Event" (like a bank reversing a bad transaction with a negative transaction) and append it to the end of the stream.

---

## 34.9 Production Checklist

*   [ ] The Write Database (Event Store) strictly enforces an Append-Only architecture; `UPDATE` and `DELETE` permissions are revoked at the SQL Server level.
*   [ ] Optimistic Concurrency is enforced on the Event Stream using a Unique Constraint on `(AggregateId, Version)`.
*   [ ] Projections (Read Models) are designed to be completely disposable and rebuildable from the ground up.
*   [ ] The UI/UX is explicitly designed to tolerate Eventual Consistency.

---

## 34.10 Exercises

1.  **Concurrency Control:** Look at the `EventStream` table in Section 34.6. Why is the `Version` integer column fundamentally superior to a `DateTime` column for guaranteeing event ordering and preventing concurrency collisions in a high-throughput system?
2.  **CQRS Architecture:** A business requirement states: "Users must be able to search for charging stations by City, and filter by minimum voltage." In a CQRS system, explain why you would execute this query against the Read Model, and why executing it against the Write Model (Event Store) would be impossible.

---

## 34.11 Interview Questions

**Q1: Explain the fundamental difference between State-Based persistence and Event Sourcing.**
*Answer:* State-Based persistence stores only the current state of an entity. When an entity changes, destructive `UPDATE` or `DELETE` commands permanently overwrite the previous data, losing the history of *how* the entity arrived at that state. Event Sourcing stores data as an append-only log of immutable business events. The current state is derived by loading and replaying the events in chronological order, providing a perfect audit trail and the ability to project the data into multiple different read models.

**Q2: What is Eventual Consistency, and why is it mandatory in a CQRS architecture?**
*Answer:* Eventual Consistency is the architectural reality that a read operation might return stale data immediately following a write operation, but will "eventually" become consistent. In CQRS, the Write Model (Event Store) is physically separated from the Read Model (Projections). Because the process that updates the Read Model runs asynchronously in the background, there is a microsecond to second delay before the Read Model reflects the newly written event.

---
**Next up in Chapter 35:** We will explore graph data in SQL Server. We will learn how to query complex hierarchical relationships (like "Friends of Friends") using SQL Server Graph features and Recursive CTEs.
