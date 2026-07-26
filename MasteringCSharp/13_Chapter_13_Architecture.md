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

## 8. Summary
Enterprise Architecture is about managing dependencies. Clean Architecture protects your business logic from the volatile world of databases and UI frameworks. CQRS allows you to tune the performance of your Reads independently from the strict validation of your Writes. By leveraging MediatR and the Outbox Pattern, we build systems that are testable, reliable, and scalable.

But eventually, data must be saved to disk. In the next chapter, we descend into the Data Access Layer, comparing the two titans of .NET data access: Entity Framework Core and Dapper.
