# Module 8: Modular Monolith Architecture & Database Isolation

This is the module where everything comes together. You've learned Middleware, DI, Routing, Configuration, Model Binding, and Error Handling — the individual building blocks. Now we'll learn **how to organize them at scale** so your application doesn't collapse under its own weight as it grows.

---

## 1. The Architecture Spectrum

### Option 1: Traditional Monolith ("Big Ball of Mud")
All code lives in one project with folders like `Controllers/`, `Services/`, `Repositories/`. Every class can reference every other class. The `OrderService` directly queries the `UserRepository`. The `UserController` creates `EmailSender` instances.

**This works** for small apps. But as the codebase grows to 50+ developers and 200+ files, it becomes an unmaintainable mess — changing the `User` table structure breaks the `Billing`, `Shipping`, and `Notification` code.

### Option 2: Microservices
Split the application into separate deployable services — a `UserService`, an `OrderService`, a `NotificationService` — each with its own database, API, and deployment pipeline.

**This sounds great** in theory. In practice, it introduces:
*   **Network complexity** — Services talk over HTTP/gRPC, which is 1000x slower than an in-memory function call.
*   **Distributed transactions** — How do you roll back an order if the payment service succeeds but the inventory service fails?
*   **DevOps nightmare** — 20 services = 20 CI/CD pipelines, 20 Docker images, 20 monitoring dashboards.
*   **Debugging hell** — A single user action spans 5 services. Good luck finding the bug.

### Option 3: Modular Monolith (The Sweet Spot)
A **Modular Monolith** gives you the **organizational discipline of microservices** with the **simplicity of a monolith**.

*   **Single deployable application** — One binary, one database server, one deployment pipeline.
*   **Strict internal boundaries** — Code is divided into isolated **modules** (Bounded Contexts) that cannot directly access each other's internals.
*   **In-memory communication** — Modules talk via function calls and in-memory events, not HTTP.
*   **Microservice-ready** — If a module needs to scale independently later, extract it with minimal effort.

> **Analogy — The Office Building:**  
> - **Monolith** = Open-plan office. Everyone sits together. The marketing team can freely access the engineering team's files. Sounds collaborative, but it's chaotic at scale.  
> - **Microservices** = Separate buildings across the city. Each department has its own building, its own reception, its own parking lot. Organized but expensive and slow to communicate.  
> - **Modular Monolith** = One building with **separate floors for each department**. They share the same reception (API), elevators (middleware), and address (deployment), but each floor has locked doors. Marketing can't walk into Engineering's lab — they have to use the intercom (interfaces/events).

---

## 2. Real-World Bounded Contexts: Normora & NextEvent

Both of your production projects implement the Modular Monolith pattern. Let's compare:

### NextEvent — Project Structure
```
NextEvent/
├── API/                            ← Composition Root (Program.cs)
│   ├── Extensions/                 ← Service registration extension methods
│   │   ├── ApiServiceExtensions.cs
│   │   ├── DatabaseServiceExtensions.cs
│   │   ├── ApplicationServiceExtensions.cs
│   │   ├── IdentityServiceExtensions.cs
│   │   ├── MassTransitServiceExtensions.cs
│   │   ├── RedisServiceExtensions.cs
│   │   └── RateLimiterServiceExtensions.cs
│   ├── Middleware/
│   │   └── ExceptionMiddleware.cs
│   ├── Services/
│   │   └── CurrentUserService.cs
│   └── Program.cs
│
├── Shared/                         ← Cross-cutting concerns (ALL modules can reference)
│   ├── Interfaces/                 ← ICurrentUserService, IDateTimeProvider
│   ├── Exceptions/                 ← NotFoundException, BusinessRuleException
│   ├── Common/                     ← ApiResponse, ValidationBehavior
│   └── Persistence/                ← SqlConnectionFactory, DapperTypeHandlers
│
└── Modules/
    ├── Identity/                   ← Authentication, Users, Roles
    │   ├── Domain/                 ← User entity, value objects
    │   ├── Application/            ← Commands, Queries, Handlers
    │   └── Persistence/            ← IdentityDbContext, Migrations, Seeders
    │
    ├── Organizations/              ← Organizations, Memberships
    │   ├── Domain/
    │   ├── Application/
    │   └── Persistence/            ← OrganizationsDbContext, Migrations
    │
    ├── Events/                     ← Events, Tickets, Sessions
    │   ├── Domain/
    │   ├── Application/
    │   └── Persistence/            ← EventsDbContext, Migrations
    │
    └── AI/                         ← Gemini integration, smart features
        └── Application/
```

### Normora — Project Structure
```
Normora/server/
├── Normora.Api/                    ← Composition Root (Program.cs, Middleware, Extensions)
│   ├── Extensions/
│   ├── Middleware/                 ← GlobalExceptionHandler, TenantResolutionMiddleware
│   ├── Services/                   ← CurrentUser, TenantContext
│   └── Program.cs
│
├── Normora.Infrastructure/         ← External Adapters (MinIO, Gemini, Redis, Keycloak)
│   ├── Agents/
│   ├── BackgroundJobs/
│   ├── Hubs/                       ← SignalR (DocumentHub, NotificationHub)
│   ├── Options/                    ← RedisOptions
│   └── Providers/
│
├── Normora.Shared/                 ← Cross-cutting interfaces & contracts
│   ├── Constants/                  ← ApiMessages
│   ├── Contracts/                  ← Cross-module interfaces & DTOs
│   ├── Exceptions/                 ← BolaException, BflaException
│   ├── Interfaces/                 ← ITenantContext, ICurrentUser
│   ├── AuditBehavior.cs
│   └── ValidationBehavior.cs
│
├── Normora.Worker/                 ← Independent Background Host (Hangfire, OCR, Vector Embeddings)
│   └── Program.cs
│
├── Normora.Architecture.Tests/     ← Automated Architectural Fitness Functions
│
└── Modules/
    ├── Tenants/                    ← Workspace boundaries, subscriptions, billing, departments
    │   ├── API/                    ← TenantsController, DepartmentsController
    │   ├── Application/            ← Commands, Queries, Handlers, Validators
    │   ├── Domain/                 ← Tenant, Department, TenantMembership entities
    │   └── Persistence/            ← TenantsDbContext (Schema: 'tenants'), Migrations
    │
    ├── Documents/                  ← Document ingestion, chunking, pgvector embeddings
    │   ├── API/                    ← DocumentsController
    │   ├── Application/
    │   ├── Domain/                 ← Document, DocumentChunk entities
    │   └── Persistence/            ← DocumentsDbContext (Schema: 'documents'), Migrations
    │
    └── Conversations/              ← Chat sessions, AI agent orchestration
        ├── API/                    ← ConversationsController
        ├── Application/
        ├── Domain/                 ← Conversation, Message entities
        └── Persistence/            ← ConversationsDbContext (Schema: 'conversations'), Migrations
```

---

## 3. The Golden Rule: Strict Database Isolation

The single most common mistake in modular architecture is sharing a single `DbContext` and writing SQL queries that join tables across module boundaries:

```csharp
// ❌ ARCHITECTURAL DISASTER: Cross-Module SQL Join
// Documents module directly querying Tenants table via EF navigation property
var doc = await dbContext.Documents
    .Include(d => d.Tenant) // Leaking foreign module entity!
    .FirstOrDefaultAsync(d => d.Id == id);
```

### Why Cross-Module Database Joins Destroy Your Architecture:
1.  **Tight Coupling at the Data Layer:** If the `Tenants` module modifies its schema, the `Documents` module breaks.
2.  **Prevents Microservice Extraction:** If you ever need to extract `Documents` into an independent microservice, you cannot, because the database tables are physically coupled by foreign keys and queries.

### 🛡️ The Architectural Rule: One `DbContext` and Schema Per Module

In **Normora**, each module owns its **private PostgreSQL schema and dedicated `DbContext`**:
*   `Tenants` module owns `TenantsDbContext` targeting PostgreSQL schema `tenants`.
*   `Documents` module owns `DocumentsDbContext` targeting PostgreSQL schema `documents`.
*   `Conversations` module owns `ConversationsDbContext` targeting PostgreSQL schema `conversations`.

```csharp
// Inside TenantsDbContext.cs
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Isolates all tables of this module into the 'tenants' schema
    modelBuilder.HasDefaultSchema("tenants");
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(TenantsDbContext).Assembly);
}
```

```csharp
// Inside DocumentsDbContext.cs
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Isolates all tables of this module into the 'documents' schema
    modelBuilder.HasDefaultSchema("documents");
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(DocumentsDbContext).Assembly);
}
```

**The Foreign Key Invariant:**  
A table in `documents` **never declares a physical foreign key constraint** to a table in `tenants`. Instead, `documents.Document` stores a plain `Guid TenantId`. Foreign key integrity within a module is enforced by the database; cross-module consistency is managed by application contracts and events.

---

## 4. Cross-Module Communication Patterns

If modules cannot reference each other's internal classes or perform database joins, how do they communicate?

```
                       CROSS-MODULE COMMUNICATION
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         ▼                                                   ▼
1. SYNCHRONOUS QUERIES                              2. ASYNCHRONOUS EVENTS
   (Needs immediate data)                              (State changes occurred)
   - Expose public interface in Shared/Contracts       - In-Memory: MediatR INotification
   - Call abstraction: ITenantPlanQueryService         - Distributed: MassTransit + RabbitMQ
```

### Pattern 1: Synchronous Queries via Contract Interfaces
If the `Documents` module needs to verify a tenant's subscription tier before allowing a 100MB file upload:
1.  Define an interface and contract DTO in `Normora.Shared.Contracts`:
    ```csharp
    public interface ITenantPlanQueryService
    {
        Task<TenantPlanDto> GetTenantPlanAsync(Guid tenantId, CancellationToken ct);
    }
    ```
2.  The `Tenants` module implements the interface using its private `TenantsDbContext`.
3.  The `Documents` module injects `ITenantPlanQueryService` via DI, without referencing any internal entity from the `Tenants` module.

### Pattern 2: Asynchronous Events via In-Memory MediatR Notifications
When an event occurs in one module that other modules need to react to, publish a **Domain Notification**:
```csharp
// Published by Tenants module when a workspace is deactivated:
public record TenantDeactivatedEvent(Guid TenantId) : INotification;

// Handled independently by Documents module:
public class TenantDeactivatedNotificationHandler(DocumentsDbContext dbContext) 
    : INotificationHandler<TenantDeactivatedEvent>
{
    public async Task Handle(TenantDeactivatedEvent notification, CancellationToken ct)
    {
        // Archive or soft-delete documents belonging to deactivated tenant
        await dbContext.Documents
            .Where(d => d.TenantId == notification.TenantId)
            .ExecuteUpdateAsync(s => s.SetProperty(d => d.IsArchived, true), ct);
    }
}
```

### Pattern 3: Distributed Asynchronous Events via MassTransit (NextEvent)
In **NextEvent**, distributed events communicate over a message broker using **MassTransit** and the **Transactional Outbox Pattern**:
```csharp
// Event published when a ticket is purchased:
await publishEndpoint.Publish(new TicketPurchasedEvent(orderId, eventId, attendeeId));
```
MassTransit writes the event to a local database outbox table within the same transaction as the order creation. A background publisher polls the outbox and publishes the event to RabbitMQ, guaranteeing **at-least-once delivery** without distributed 2-phase commits.

---

## 5. Multi-Tenancy Architecture: EF Core Global Query Filters

In enterprise multi-tenant systems, data isolation between rival tenants is a paramount security invariant.

### The Problem: Accidental Tenant Data Leaks
If an engineer forgets to append `WHERE TenantId = @currentTenant` to a single query:
```csharp
// ❌ HIGH RISK: If where clause is forgotten, Tenant A sees Tenant B's confidential documents!
var docs = await dbContext.Documents.ToListAsync();
```

### 🛡️ The Architectural Solution: Global Query Filters
In **Normora**, multi-tenant isolation is enforced automatically at the database context level via **EF Core Global Query Filters**:

```csharp
public class DocumentsDbContext(
    DbContextOptions<DocumentsDbContext> options,
    ITenantContext tenantContext) // Scoped to the current HTTP request
    : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Enforce Multi-Tenant Isolation across all queries automatically:
        modelBuilder.Entity<Document>()
            .HasQueryFilter(doc => doc.TenantId == tenantContext.TenantId);
    }
}
```

Whenever *any* query executes against `DocumentsDbContext`:
```csharp
var docs = await dbContext.Documents.ToListAsync();
```
EF Core automatically rewrites the executed SQL query:
```sql
SELECT d.id, d.title, d.tenant_id 
FROM documents.documents AS d
WHERE d.tenant_id = '3fa85f64-5717-4562-b3fc-2c963f66afa6';
```

### Safe Bypassing in Background Workers
When a system background worker (e.g., Hangfire job running in `Normora.Worker`) needs to reindex all documents across all tenants, the architect explicitly disables the filter:
```csharp
// Explicitly bypass tenant filter for system administrative jobs
var allSystemDocs = await dbContext.Documents
    .IgnoreQueryFilters()
    .ToListAsync(ct);
```

### Immutable Scoped Context Defense
In Normora's `TenantContext`:
```csharp
// From Normora's TenantContext.cs — Immutable once set per request
public class TenantContext : ITenantContext
{
    public Guid? TenantId { get; private set; }
    public string? TenantRole { get; private set; }
    public bool IsTenantResolved => TenantId.HasValue;

    public void SetContext(Guid tenantId, string role, IReadOnlyCollection<Guid> departments)
    {
        if (IsTenantResolved)
            throw new InvalidOperationException("Tenant context is already set for this request.");
        
        TenantId = tenantId;
        TenantRole = role;
    }
}
```

**Why `throw` if already set?** Prevents a subtle attack where a compromised service tries to switch tenants mid-request to access another company's data.

---

## 6. Decoupled Background Workers: The `Normora.Worker` Host

In high-throughput architectures, CPU-intensive tasks must **never run in the user-facing web API process**:
*   PDF OCR text extraction
*   Document chunking and tokenization
*   High-dimensional vector embedding generation via Gemini API

If these workloads execute inside `Normora.Api`, thread pool starvation occurs, causing HTTP request response times to spike for all users.

### The Solution: Headless Worker Process
Normora isolates heavy workloads into a dedicated console worker: **`Normora.Worker`**.
*   `Normora.Api` accepts document uploads, saves them to MinIO storage, enqueues a Hangfire job ID, and returns HTTP 202 Accepted.
*   `Normora.Worker` runs in a separate Docker container, pulls jobs from the PostgreSQL/Hangfire queue, processes embeddings, and notifies the API via Redis backplane and SignalR.

---

## 7. Architectural Fitness Functions: Enforcing Module Boundaries in CI/CD

How do software architects guarantee that developers do not reference internal classes across modules?

In **Normora**, the test suite `Normora.Architecture.Tests` runs in CI/CD pipelines to fail builds if module boundaries are violated:

```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs

[Fact]
public void DocumentsModule_MustNotReference_ConversationsInternals()
{
    var docTypes = DocumentsAssembly.GetTypes()
        .Where(t => t.IsClass && !t.IsAbstract)
        .ToList();

    foreach (var type in docTypes)
    {
        foreach (var ctor in type.GetConstructors())
        {
            foreach (var param in ctor.GetParameters())
            {
                Assert.False(
                    param.ParameterType.Namespace?.StartsWith("Normora.Modules.Conversations") == true,
                    $"Documents type '{type.FullName}' directly references Conversations type '{param.ParameterType.FullName}'. " +
                    $"Cross-module communication must use Normora.Shared.Contracts.");
            }
        }
    }
}

[Fact]
public void BusinessModules_MustNotReferenceEachOther()
{
    var moduleAssemblies = new[] { TenantsAssembly, DocumentsAssembly, ConversationsAssembly };

    foreach (var module in moduleAssemblies)
    {
        var otherModuleNames = moduleAssemblies
            .Where(m => m != module)
            .Select(m => m.GetName().Name!)
            .ToList();

        var referencedAssemblies = module.GetReferencedAssemblies()
            .Select(r => r.Name)
            .ToList();

        foreach (var other in otherModuleNames)
        {
            Assert.DoesNotContain(
                other, 
                referencedAssemblies, 
                $"Architectural Violation: Module '{module.GetName().Name}' references '{other}'. " +
                $"Modules must be completely decoupled.");
        }
    }
}

[Fact]
public void Worker_MustNotReferenceApiHost()
{
    Assert.DoesNotContain(
        WorkerAssembly.GetReferencedAssemblies(),
        reference => string.Equals(reference.Name, "Normora.Api", StringComparison.Ordinal),
        "Worker host must be completely decoupled from the Web API host.");
}
```

---

## 8. When to Extract a Module into a Microservice

The beauty of a Modular Monolith is that extraction is surgical, not an architectural rewrite:

| Signal | Action |
|---|---|
| One module needs independent scaling (e.g., AI document processing) | Extract to a separate service |
| One module requires a different database technology | Give it its own database server |
| One module has an independent release cadence | Deploy it independently |
| One module is owned by a separate engineering team | Hand that team complete ownership |

**How to extract:** Because the module already owns its private `DbContext`, schemas, and communicates via interfaces/events:
1. Create a new ASP.NET Core project for the module.
2. Move the module's folder into the new solution.
3. Replace in-memory MediatR domain events with MassTransit/RabbitMQ distributed messages.
4. Point the module's `DbContext` at its dedicated database instance.

---

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Organize by business domain, not technical layer** | `/Modules/Events/` not `/Controllers/EventsController.cs` |
| **One DbContext per module** | Enforces compile-time and runtime data isolation |
| **Separate database schemas per module** | `tenants`, `documents`, `conversations` schemas in PostgreSQL |
| **Use a Shared project for contracts only** | Modules depend on abstractions, never on each other's internals |
| **API project = Composition Root only** | Wires dependencies and middleware; contains zero business logic |
| **Communicate via events and interfaces** | Decoupled in-memory or distributed messaging |
| **Apply Global Query Filters for multi-tenancy** | Physically prevents accidental cross-tenant data leaks |
| **Decouple CPU-heavy work into Worker hosts** | Prevents background tasks from choking the web API thread pool |
| **Automate boundary verification with Architecture Tests** | Programmatically fails CI/CD builds on boundary violations |

---
**Key Takeaways:**
1. A Modular Monolith provides the modular discipline of microservices with the deployment simplicity of a monolith.
2. Each module owns its dedicated schema and `DbContext` — cross-module SQL joins are strictly forbidden.
3. Multi-tenant isolation is enforced via EF Core Global Query Filters combined with scoped `ITenantContext`.
4. Decouple heavy background workloads into headless worker hosts (`Normora.Worker`).
5. Write architecture tests to programmatically enforce module isolation in CI/CD.

**Next up:** In [Module 9](09_Module_9_Authentication_and_Authorization.md), we explore **Authentication, Authorization & Identity Architecture** — Claims, JWTs, Keycloak IAM, and why the Backend-For-Frontend (BFF) pattern is the modern standard for securing Single Page Applications.
