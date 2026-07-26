# Chapter 13: Clean Architecture and CQRS

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Architect an enterprise application using Clean Architecture principles.
- Apply the Dependency Inversion Principle to isolate business logic from infrastructure.
- Implement Command Query Responsibility Segregation (CQRS) using MediatR.
- Design Domain Events to decouple complex state transitions.
- Implement the Outbox Pattern for guaranteed message delivery in distributed systems.

## 2. Introduction

Writing C# code that compiles is easy. Writing C# code that runs fast is harder. Writing C# code that can survive 10 years of shifting business requirements, framework updates, and team turnover is the true test of a Software Architect.

In traditional N-Tier architecture (Presentation -> Business Logic -> Data Access), the Business Logic layer usually depends heavily on the Data Access layer. If you decide to swap SQL Server for MongoDB, or swap Entity Framework for Dapper, you often have to rewrite your entire Business layer because the database concerns have leaked into your core logic.

**Clean Architecture** (popularized by Robert C. Martin) flips this on its head. It mandates that the Core Domain (your business logic) must not depend on *anything*. Infrastructure (databases, APIs, UI) must depend on the Core.

## 3. Dependency Inversion and Clean Architecture Layers

To achieve Clean Architecture, we rely on the **Dependency Inversion Principle** (The "D" in SOLID). 
*"High-level modules should not depend on low-level modules. Both should depend on abstractions (interfaces)."*

### The Four Layers
In our EV Platform, we will structure our C# Solution (`.sln`) into four distinct projects:

1. **Domain (Core):** Contains Entities (`ChargingSession`), Value Objects (`MeterValue`), and Domain Interfaces (`IChargerRepository`). This project has **zero** dependencies on NuGet packages like Entity Framework or ASP.NET. It is pure C#.
2. **Application (Use Cases):** Contains business workflows (e.g., `StartChargingCommand`). It depends on the Domain. It still has no concept of SQL or HTTP.
3. **Infrastructure:** Contains the actual implementations of the Domain interfaces (e.g., `SqlChargerRepository`). It depends on the Application and Domain layers. It references EF Core, Redis, etc.
4. **Presentation (API):** The ASP.NET Core project. It depends on Application and Infrastructure (solely to wire up the Dependency Injection container).

By defining `IChargerRepository` in the Domain, but implementing it in Infrastructure, the Application layer can save data without ever knowing *how* it is being saved.

## 4. CQRS: Command Query Responsibility Segregation

In traditional CRUD (Create, Read, Update, Delete) architectures, you often use the same Entity to read data and write data. 

However, in enterprise systems, **Reads** and **Writes** have vastly different requirements.
- **Writes (Commands):** Need strict validation, complex domain logic, transaction scopes, and event firing. They process a single entity at a time.
- **Reads (Queries):** Need to be incredibly fast. They often aggregate data from multiple tables and return flattened Data Transfer Objects (DTOs). They don't need validation or tracking.

**CQRS** separates these two concerns completely.

### Implementing CQRS with MediatR
MediatR is an open-source library that implements the Mediator pattern. It allows your API controllers to send a Request (Command or Query) into the ether, and MediatR automatically routes it to the correct Handler.

```csharp
// 1. The Command (Application Layer)
public record StartChargingCommand(string ChargerId, string UserId) : IRequest<Guid>;

// 2. The Handler (Application Layer)
public class StartChargingHandler : IRequestHandler<StartChargingCommand, Guid>
{
    private readonly IChargerRepository _repository;

    public StartChargingHandler(IChargerRepository repository)
    {
        _repository = repository;
    }

    public async Task<Guid> Handle(StartChargingCommand request, CancellationToken cancellationToken)
    {
        // Fetch domain entity
        var charger = await _repository.GetByIdAsync(request.ChargerId);
        
        // Execute pure business logic
        var session = charger.StartSession(request.UserId);
        
        // Save via abstraction
        await _repository.SaveAsync(charger);
        
        return session.Id;
    }
}
```

**The API Controller (Presentation Layer):**
```csharp
[ApiController]
[Route("api/chargers")]
public class ChargerController : ControllerBase
{
    private readonly IMediator _mediator;

    public ChargerController(IMediator mediator) => _mediator = mediator;

    [HttpPost("{id}/start")]
    public async Task<IActionResult> Start(string id, [FromBody] string userId)
    {
        // The API knows NOTHING about repositories or business logic. 
        // It simply dispatches the command.
        var sessionId = await _mediator.Send(new StartChargingCommand(id, userId));
        return Ok(sessionId);
    }
}
```

## 5. Domain Events and the Outbox Pattern

When a charging session starts, multiple side-effects must occur:
- An email must be sent to the user.
- A push notification must be sent to the mobile app.
- A billing record must be initialized.

If we put all this code inside `StartChargingHandler`, the handler violates the Single Responsibility Principle and becomes massive. Instead, the Domain Entity should raise a **Domain Event**.

```csharp
public class ChargingSession
{
    public List<IDomainEvent> DomainEvents { get; } = new();

    public void Start(string userId)
    {
        // Core state change
        IsActive = true;
        
        // Raise event for side-effects
        DomainEvents.Add(new SessionStartedEvent(Id, userId));
    }
}
```

### The Reliability Problem: The Outbox Pattern
If we save the session to the SQL database, and then try to publish the `SessionStartedEvent` to RabbitMQ, what happens if RabbitMQ is offline? The database transaction is already committed, but the email and billing system are never notified. We have distributed data inconsistency.

**The Outbox Pattern** solves this.
Instead of sending the event to RabbitMQ immediately, we serialize the event to JSON and save it into an `OutboxMessages` table in the *same SQL transaction* as the `ChargingSession` update. 

Because it is a single SQL transaction, it is Atomic. Either both save, or neither save.
A separate Background Worker (using a timer or channels) continuously polls the `OutboxMessages` table and safely forwards the messages to RabbitMQ, retrying if RabbitMQ is down.

## 6. Real Production Case Study: Vertical Slice Architecture

While Clean Architecture organizes code by *technical concern* (Domain vs Data Access), some architects prefer **Vertical Slice Architecture**. 

In Vertical Slice, we organize code by *Feature*. Everything related to "Starting a Session" (the Command, the Handler, the specific DB Query, the API Endpoint) lives in a single folder. 

This minimizes jumping between 4 different projects to add one simple feature. MediatR shines here.

```text
📁 Features
 📁 Charging
    📄 StartSession.cs (Contains Command, Handler, and Endpoint setup)
    📄 StopSession.cs
    📄 GetSessionStatus.cs (Contains Query and fast Dapper logic)
```

**Enterprise Rule:** Clean Architecture is best for highly complex domains with deep business rules. Vertical Slice is best for microservices that have high cohesion and want to optimize developer velocity. You can blend both: Group by Vertical Slice, but enforce Clean dependencies inside the slice.

## 7. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Injecting `DbContext` directly into the API Controller. | Business logic bleeds into the UI layer. Cannot unit test without a real database. | Use MediatR or Service classes to isolate logic. |
| Intermediate | Using EF Core Entities as the return types for API Queries. | Leaks database schema to the client. Over-fetches data, causing high memory usage. | For Queries, project directly to simple Record DTOs using `.Select()`. |
| Senior | Calling external APIs (Stripe, Twilio) inside a database transaction scope. | If the external API takes 5 seconds, you lock the SQL table for 5 seconds, causing massive deadlocks. | Publish a Domain Event (via Outbox). Call external APIs asynchronously in the event handler. |
| Architect | Dogmatic adherence to Clean Architecture for simple CRUD. | Over-engineering. 10 files created just to update a user's name. | Use CQRS. Allow simple Queries to bypass the Domain layer entirely and read straight from the DB. |

## 8. Interview Questions

### Beginner Tier (Architecture and SOLID Basics)

**1. What is the Dependency Inversion Principle (DIP)?**
*Answer:* DIP states that high-level modules (business logic) should not depend on low-level modules (databases, APIs). Both should depend on abstractions (interfaces). This makes the business logic testable and decoupled from infrastructure.

**2. What is the Single Responsibility Principle (SRP)?**
*Answer:* A class or module should have one, and only one, reason to change. For example, a class should not handle both database access and complex business validation.

**3. What is the core philosophy of Clean Architecture?**
*Answer:* The core domain (business logic) is at the center of the application and has absolutely zero dependencies on outer layers (like databases, UI, or web frameworks). Dependencies always point inwards toward the core.

**4. What is a Domain Entity?**
*Answer:* An object that represents a core business concept and possesses a unique identity (e.g., a `User` with a `Guid Id`). Its identity remains the same even if its properties change.

**5. What is a Value Object?**
*Answer:* An object that contains attributes but has no conceptual identity. Two Value Objects are considered equal if all their properties are identical (e.g., an `Address` or `Money` struct). Value Objects should be immutable.

**6. What is a Data Transfer Object (DTO)?**
*Answer:* A simple object (usually a C# `record`) with no behavior/logic, used exclusively to transfer data between layers or across the network (e.g., returning JSON from an API).

**7. Why should you avoid returning Domain Entities directly from an API?**
*Answer:* It tightly couples your public API contract to your internal domain model. Changing the domain logic breaks mobile clients. It also risks exposing sensitive data (like password hashes) if they are accidentally serialized.

### Intermediate Tier (CQRS and MediatR)

**8. What is CQRS and why is it useful?**
*Answer:* Command Query Responsibility Segregation separates Write operations (Commands) from Read operations (Queries). It is useful because read models and write models have vastly different performance and validation requirements, and CQRS allows them to be optimized and scaled independently.

**9. What is the Mediator Pattern?**
*Answer:* A behavioral pattern that reduces chaotic dependencies between objects. Instead of objects communicating directly with each other (tight coupling), they send messages to a central Mediator object, which routes the message to the appropriate handler.

**10. What is MediatR?**
*Answer:* A popular open-source library in the .NET ecosystem that implements the Mediator pattern, making it trivial to implement CQRS by dispatching `IRequest` objects to `IRequestHandler` classes.

**11. In CQRS, what is the role of a Command?**
*Answer:* A Command is an intent to mutate state (e.g., `CreateUserCommand`). It should contain all the data necessary to execute the operation. It typically returns nothing, or just the ID of the created resource.

**12. In CQRS, what is the role of a Query?**
*Answer:* A Query is a request for data (e.g., `GetUserByIdQuery`). It must never mutate system state (it is idempotent/side-effect free) and returns a DTO.

**13. Why should you avoid injecting a `DbContext` directly into an API Controller?**
*Answer:* It tightly couples the HTTP Presentation layer directly to the Data Access layer, bypassing the Domain/Business layer entirely. This makes it impossible to unit test the controller without a database, and business logic inevitably leaks into the UI layer.

**14. What is an Aggregate Root in Domain-Driven Design (DDD)?**
*Answer:* A specific Domain Entity that acts as a gateway or boundary for a cluster of related entities. For example, an `Order` is the aggregate root for `OrderLineItem`s. External objects may only hold references to the aggregate root, and all modifications to the cluster must go through the root's methods.

### Senior Tier (Domain Events and Decoupling)

**15. What is a Domain Event?**
*Answer:* A message indicating that something significant has happened within the business domain (e.g., `OrderPlacedEvent`). It allows different parts of the system to react to the event without the original component knowing about them.

**16. How do you implement Domain Events in EF Core?**
*Answer:* Instead of dispatching events immediately in business logic, the Domain Entity adds the event object to an internal `List<IDomainEvent>`. During `DbContext.SaveChanges()`, you intercept the save process, extract the events from the tracked entities, and dispatch them via MediatR.

**17. Why is dispatching Domain Events *before* `SaveChanges` risky?**
*Answer:* If you dispatch an `OrderCreatedEvent` and an email is sent to the customer, but the subsequent SQL `INSERT` fails due to a database error, the customer received a confirmation for an order that does not exist.

**18. What is the difference between a Domain Event and an Integration Event?**
*Answer:* A Domain Event triggers side-effects *within the same microservice* (often within the same process and database transaction). An Integration Event is published to an external message broker (like RabbitMQ or Kafka) to notify *other microservices* that something happened.

**19. What is an Anemic Domain Model?**
*Answer:* An anti-pattern where Domain Entities contain only properties (getters/setters) and no behavior. All business logic is stripped out and placed in "Service" classes, turning the entities into glorified DTOs and violating encapsulation.

**20. What is a Rich Domain Model?**
*Answer:* A design where Domain Entities possess both data and behavior. The properties have private setters, and state is mutated only through explicit methods that enforce business invariants (e.g., `public void ApplyDiscount(decimal amount)`).

**21. Explain the Repository Pattern.**
*Answer:* It creates an abstraction layer between the domain and the data access layer. It treats the database like an in-memory collection of Aggregate Roots, hiding the complexities of SQL or EF Core behind an interface (e.g., `IUserRepository.GetById(id)`).

### Staff Engineer Tier (Transactions and The Outbox Pattern)

**22. Explain the Outbox Pattern and why it is critical for Microservices.**
*Answer:* Dual-write scenarios (saving to SQL and publishing an event to RabbitMQ) are prone to distributed inconsistency if the message broker fails. The Outbox Pattern writes the domain event to a SQL `Outbox` table in the *same transaction* as the entity update. A background worker reliably polls and forwards the messages.

**23. How do you handle idempotency when consuming Integration Events?**
*Answer:* Message brokers guarantee "at-least-once" delivery, meaning your consumer might process the exact same message twice. The consumer must record the `EventId` in its database when processing. If it receives a duplicate `EventId`, it acknowledges the message but skips processing it.

**24. In Clean Architecture, where do third-party API clients (e.g., a Stripe SDK) belong?**
*Answer:* The SDK belongs strictly in the Infrastructure layer. The Domain layer defines an interface (`IPaymentGateway`). The Application layer uses the interface. The Infrastructure layer implements the interface and wraps the Stripe SDK.

**25. Why should you avoid referencing the MediatR package in your core Domain project?**
*Answer:* The Domain project should be completely agnostic of external libraries and frameworks. Domain Events should simply be plain C# interfaces (e.g., `public interface IDomainEvent { }`). The Application/Infrastructure layers can map these to MediatR's `INotification`.

**26. What is Vertical Slice Architecture?**
*Answer:* An architecture that organizes code by *Feature* rather than technical layers. Everything related to "Starting a Session" (the Command, Handler, DB Query, API Endpoint) lives in a single folder. It maximizes cohesion and developer velocity for that specific feature.

**27. How does CQRS solve the "N+1 query problem" inherently?**
*Answer:* In standard ORM usage, fetching a complex aggregate root graph can trigger N+1 queries. In CQRS, Queries bypass the ORM and Aggregate Roots completely. A Query handler executes a single, highly-optimized SQL `JOIN` (often using Dapper) and maps the flat result directly to the required UI DTO.

### Architect Tier (Distributed Systems and Saga)

**28. Compare Clean Architecture to Vertical Slice Architecture. When would you use each?**
*Answer:* Clean Architecture organizes horizontally by technical concern. It is best for highly complex, shifting domains with deep business rules across shared entities. Vertical Slice organizes vertically by Feature. It is best for highly cohesive microservices where maximizing developer velocity is the priority, as it prevents scattering files across 4 projects just to add a simple CRUD endpoint.

**29. What is the Saga Pattern?**
*Answer:* A pattern for managing distributed transactions across multiple microservices where standard ACID database transactions cannot exist. It breaks the transaction into a sequence of local transactions. If one step fails, the Saga executes compensating transactions (e.g., `RefundPayment`) to undo the preceding steps.

**30. Explain Choreography vs. Orchestration in Sagas.**
*Answer:* Choreography means microservices publish events and react to each other without a central controller (highly decoupled, but hard to monitor). Orchestration uses a central coordinator service (the Orchestrator) that explicitly sends commands to participants and tracks the state of the entire Saga.

**31. How do you scale the Outbox Pattern's background worker without processing messages twice?**
*Answer:* If multiple pods poll the SQL Outbox table simultaneously, they will read the same messages. You must use `SELECT ... FOR UPDATE SKIP LOCKED` (in PostgreSQL/SQL Server) so the database safely partitions the rows among the workers, ensuring each message is locked and processed by exactly one pod.

**32. What is Event Sourcing?**
*Answer:* Instead of storing the *current state* of an entity in a database table, Event Sourcing stores the *sequence of events* that led to that state in an Event Store. The current state is derived by replaying the events. It provides a perfect audit trail and "time travel" capabilities, but drastically increases system complexity.

**33. How does Event Sourcing integrate with CQRS?**
*Answer:* They are naturally complementary. The Write side appends events to the Event Store. As events are appended, they are published. The Read side consumes these events and updates materialized views (standard SQL/NoSQL tables optimized for rapid querying).

**34. In an enterprise system, should every microservice use Clean Architecture?**
*Answer:* No. Imposing Clean Architecture on a simple, data-driven microservice (like a Reference Data lookup service) is massive over-engineering. An architect must evaluate the domain complexity. Complex domains get Clean Architecture/DDD; simple domains get basic CRUD or Vertical Slices.

## 9. Summary
Enterprise Architecture is about managing dependencies. Clean Architecture protects your business logic from the volatile world of databases and UI frameworks. CQRS allows you to tune the performance of your Reads independently from the strict validation of your Writes. By leveraging MediatR and the Outbox Pattern, we build systems that are testable, reliable, and scalable.

But eventually, data must be saved to disk. In the next chapter, we descend into the Data Access Layer, comparing the two titans of .NET data access: Entity Framework Core and Dapper.
