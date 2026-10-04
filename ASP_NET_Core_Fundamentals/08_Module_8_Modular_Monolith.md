# Module 8: Modular Monolith Architecture

This is the module where everything comes together. You've learned Middleware, DI, Routing, Configuration, Model Binding, and Error Handling — the individual building blocks. Now we'll learn **how to organize them at scale** so your application doesn't collapse under its own weight as it grows.

## 1. The Architecture Spectrum

### Option 1: Traditional Monolith ("Big Ball of Mud")
All code lives in one project with folders like `Controllers/`, `Services/`, `Repositories/`. Every class can reference every other class. The `OrderService` directly queries the `UserRepository`. The `UserController` creates `EmailSender` instances.

**This works** for small apps. But as the codebase grows to 50+ developers and 200+ files, it becomes an unmaintainable mess — changing the `User` table structure breaks the `Billing`, `Shipping`, and `Notification` code.

### Option 2: Microservices
Split the application into separate deployable services — a `UserService`, an `OrderService`, a `NotificationService` — each with its own database, API, and deployment pipeline.

**This sounds great** in theory. In practice, it introduces:
*   **Network complexity** — Services talk over HTTP/gRPC, which is 1000x slower than a function call.
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

## 2. How NextEvent and Normora Structure Their Modules

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
│   └── Persistence/               ← SqlConnectionFactory, DapperTypeHandlers
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
├── Normora.Api/                    ← Composition Root
│   ├── Extensions/                 ← Service registration
│   ├── Middleware/                  ← GlobalExceptionHandler, TenantResolutionMiddleware
│   ├── Hubs/                       ← SignalR (DocumentHub, NotificationHub)
│   ├── Features/                   ← Vertical Slices (Commands, Queries, DTOs)
│   ├── Services/                   ← CurrentUser, TenantContext, NotificationService
│   └── Program.cs
│
├── Normora.Shared/                 ← Cross-cutting concerns
│   ├── Interfaces/                 ← ITenantContext, ICurrentUser
│   ├── Exceptions/                 ← NotFoundException, BolaException, BflaException
│   └── Constants/                  ← ApiMessages
│
└── Modules/
    ├── Normora.Modules.Auth/       ← Keycloak integration, SSO
    ├── Normora.Modules.Tenants/    ← Multi-tenancy, Memberships, Departments
    │   └── Persistence/            ← TenantsDbContext
    ├── Normora.Modules.Documents/  ← Document upload, processing, search
    │   └── Persistence/            ← DocumentsDbContext
    ├── Normora.Modules.Conversations/ ← AI chat, conversation history
    │   └── Persistence/            ← ConversationsDbContext
    └── Normora.Modules.Users/      ← User profiles, preferences
```

### Key Observations
1. **Each module has its own `DbContext`** — This is the enforcement mechanism for data isolation.
2. **Each module has its own `Domain`, `Application`, and `Persistence` layers** — Clean Architecture within each module.
3. **The `Shared` project contains only interfaces and exceptions** — Not implementations. Modules depend on abstractions.
4. **The `API` project is the Composition Root** — It references all modules and wires them together. No business logic lives here.

## 3. The Golden Rule: Database Isolation

This is the **most important rule** of a Modular Monolith. Break it, and you've built a traditional monolith with extra folders.

### The Rule
**A module must NEVER directly access another module's database tables.**

Even though all modules share the same physical database server, each module has its own `DbContext` that only knows about its own tables.

```csharp
// From NextEvent's DatabaseServiceExtensions.cs
void ConfigureSqlOptions(DbContextOptionsBuilder options, string migrationsAssembly)
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        // Each module stores its migrations in its own assembly
        sqlOptions.MigrationsAssembly(migrationsAssembly);
        sqlOptions.EnableRetryOnFailure(maxRetryCount: 5, ...);
        sqlOptions.CommandTimeout(60);
    });
}

// Three separate DbContexts, three separate migration histories
services.AddDbContext<IdentityDbContext>(o => ConfigureSqlOptions(o, "NextEvent.Modules.Identity"));
services.AddDbContext<OrganizationsDbContext>(o => ConfigureSqlOptions(o, "NextEvent.Modules.Organizations"));
services.AddDbContext<EventsDbContext>(o => ConfigureSqlOptions(o, "NextEvent.Modules.Events"));
```

### Why Separate DbContexts?
1. **Prevents accidental cross-module queries.** If the `Events` module's `DbContext` doesn't have a `DbSet<User>`, a developer physically cannot write `.Include(e => e.User)` — the compiler won't allow it.
2. **Independent migrations.** Each module evolves its schema independently. Adding a column to the `Events` table doesn't require the `Identity` module to know about it.
3. **Microservice extraction readiness.** If you later need to extract the `Events` module into a separate service, you simply point its `DbContext` at a different database. No code changes needed.

### Per-Module Migration and Seeding
```csharp
// From NextEvent's Program.cs
using var scope = app.Services.CreateScope();
var services = scope.ServiceProvider;

var identityContext = services.GetRequiredService<IdentityDbContext>();
var orgContext = services.GetRequiredService<OrganizationsDbContext>();
var eventsContext = services.GetRequiredService<EventsDbContext>();

if (app.Environment.IsDevelopment())
{
    // Each module migrates independently
    identityContext.Database.Migrate();
    orgContext.Database.Migrate();
    eventsContext.Database.Migrate();

    // Each module seeds its own data
    await IdentityDataSeeder.SeedAsync(roleManager, userManager);
    await OrganizationsDataSeeder.SeedAsync(orgContext);
    await EventsDataSeeder.SeedAsync(eventsContext, userManager);
}
```

## 4. Cross-Module Communication

If modules can't access each other's databases, how do they share data?

### Method 1: Shared Interfaces (Synchronous)
A module exposes a **public interface** in the `Shared` project. Other modules call it.

```csharp
// In Shared/Interfaces/
public interface ICurrentUserService
{
    string? GetCurrentUserId();
    Guid? GetCurrentUserOrganizationId();
    bool HasRole(string role);
}

// Implemented in API/Services/ (not inside any module)
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public string? GetCurrentUserId()
    {
        var user = _httpContextAccessor.HttpContext?.User;
        return user?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }
}
```

Any module can inject `ICurrentUserService` to get the current user without depending on the `Identity` module directly.

### Method 2: Domain Events (Asynchronous, In-Memory)
When something important happens in one module, it raises an **event**. Other modules subscribe.

```
Module A (Identity)                 Module B (Organizations)
┌─────────────────────┐            ┌──────────────────────────┐
│ User Registers      │            │ Event Handler            │
│   ↓                 │  ──event──→│   ↓                      │
│ Publish:            │            │ Creates default          │
│ UserRegisteredEvent │            │ Organization for user    │
└─────────────────────┘            └──────────────────────────┘
```

This happens **in-memory** via MediatR's `INotification`:

```csharp
// The event (in Shared project)
public class UserRegisteredEvent : INotification
{
    public Guid UserId { get; init; }
    public string Email { get; init; }
}

// Published by the Identity module
await _mediator.Publish(new UserRegisteredEvent { UserId = user.Id, Email = user.Email });

// Handled by the Organizations module
public class CreateDefaultOrgHandler : INotificationHandler<UserRegisteredEvent>
{
    private readonly OrganizationsDbContext _db;

    public async Task Handle(UserRegisteredEvent notification, CancellationToken ct)
    {
        var org = new Organization { OwnerId = notification.UserId, Name = "My Organization" };
        _db.Organizations.Add(org);
        await _db.SaveChangesAsync(ct);
    }
}
```

**Why not direct method calls?** Direct calls create compile-time dependencies between modules. Events keep modules completely decoupled — the Identity module doesn't even know the Organizations module exists.

### Method 3: Message Bus (Asynchronous, Over Network)
For truly distributed scenarios, replace MediatR with a real message broker like **RabbitMQ** (via MassTransit, which NextEvent uses):

```csharp
// From NextEvent's Program.cs
builder.Services.AddMassTransitServices(builder.Configuration);
```

MassTransit can use MediatR-like in-memory mode for the monolith, and switch to RabbitMQ/Kafka when you extract a module into a microservice. **Zero code changes in your handlers.**

## 5. The Composition Root — Where Modules Meet

The `API` project (or `Normora.Api`) is the **Composition Root**. It's the only project that references all modules. Its sole job is to wire everything together in `Program.cs`.

```csharp
// From NextEvent's Program.cs — The Composition Root
var builder = WebApplication.CreateBuilder(args);

// Each extension method wires up a specific concern, pulling from multiple modules
builder.Services.AddApiServices(builder.Configuration);       // API layer
builder.Services.AddDatabaseServices(builder.Configuration);  // All module DbContexts
builder.Services.AddApplicationServices();                    // All module handlers
builder.Services.AddIdentityServices(builder.Configuration);  // Identity module auth
builder.Services.AddMassTransitServices(builder.Configuration); // Event bus
builder.Services.AddRedisServices(builder.Configuration);     // Caching
builder.Services.AddRateLimiterServices();                    // Rate limiting
```

### Assembly Scanning Across Modules
Because each module lives in a separate assembly, you must tell libraries like FluentValidation and MediatR to scan **multiple assemblies**:

```csharp
// From NextEvent's ApplicationServiceExtensions.cs
// FluentValidation: Scan each module's assembly separately
services.AddValidatorsFromAssemblyContaining<CreateEventCommandValidator>();      // Events module
services.AddValidatorsFromAssemblyContaining<CreateOrganizationCommandValidator>(); // Orgs module
services.AddValidatorsFromAssemblyContaining<RegisterCommandValidator>();          // Identity module

// MediatR: Same pattern
services.AddMediatR(x =>
{
    x.RegisterServicesFromAssemblyContaining<GetEventsListQueryHandler>();
    x.RegisterServicesFromAssemblyContaining<GetOrganizationByIdQueryHandler>();
    x.RegisterServicesFromAssemblyContaining<LoginCommandHandler>();
    x.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

## 6. Multi-Tenancy in a Modular Monolith (Normora)

Normora adds another layer of complexity: **multi-tenancy**. The same application serves multiple companies (tenants), each with isolated data.

### How It Works
```
Request → Authentication → TenantResolutionMiddleware → Authorization → Controller
                                    │
                                    ▼
                           ITenantContext (Scoped)
                           ┌─────────────────────┐
                           │ TenantId: abc-123    │
                           │ TenantRole: Admin    │
                           │ Departments: [...]   │
                           └─────────────────────┘
                                    │
                            Injected into every
                            DbContext, Service,
                            and Repository
```

1. **Middleware reads** the `X-Tenant-Id` header from the request.
2. **Middleware verifies** the authenticated user actually belongs to that tenant.
3. **Middleware populates** the `ITenantContext` (a Scoped service).
4. **DbContexts use** `ITenantContext` in global query filters to automatically scope all queries to the current tenant.
5. **Authorization** checks that the user's tenant role allows the requested action.

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

**Why `throw` if already set?** Prevents a sneaky attack where a malicious middleware or service tries to switch tenants mid-request to access another company's data.

## 7. When to Extract a Module into a Microservice

The beauty of a Modular Monolith is that extraction is surgical, not a rewrite:

| Signal | Action |
|---|---|
| One module needs different scaling (e.g., AI processing) | Extract to a separate service |
| One module needs a different database technology | Give it its own database |
| One module has a different deployment cadence | Deploy it independently |
| One module is maintained by a different team | Give that team ownership |

**How to extract:** Because the module already has its own `DbContext`, its own handlers, and communicates via interfaces/events, you simply:
1. Create a new ASP.NET Core project for the module.
2. Move the module's code into it.
3. Replace in-memory MediatR events with RabbitMQ/Kafka messages.
4. Point the module's `DbContext` at its own database.

## 8. Best Practices Summary

| Practice | Why |
|---|---|
| **Organize by business domain, not technical layer** | `/Modules/Events/` not `/Controllers/EventsController.cs` |
| **One DbContext per module** | Enforces data isolation at compile time |
| **Use a Shared project for interfaces only** | Modules depend on abstractions, not each other |
| **API project = Composition Root only** | No business logic, only wiring |
| **Use extension methods for service registration** | One file per concern, clean `Program.cs` |
| **Communicate via events, not direct calls** | Loose coupling, microservice-ready |
| **Scan assemblies per module** | FluentValidation and MediatR need to know about each module |
| **Fail fast with `throw` on missing critical config** | Don't start a broken application |
| **Use MassTransit for the event bus** | Switches between in-memory and RabbitMQ transparently |

---
**Key Takeaways:**
1. A Modular Monolith is the best starting architecture for most applications — monolith simplicity with microservice discipline.
2. Each module gets its own `DbContext`, preventing accidental cross-module database access.
3. Modules communicate via shared interfaces (sync) or domain events (async), never via direct database queries.
4. The Composition Root (API project) wires everything together but contains zero business logic.
5. Extracting a module to a microservice is a deployment change, not a rewrite.

---
**🎓 Congratulations!** You've completed the ASP.NET Core Fundamentals course. You now understand:
- How HTTP requests flow through the **Middleware Pipeline**
- How **Dependency Injection** creates and manages every service
- How **Routing** maps URLs to your C# code
- How **Configuration** externalizes settings safely
- How **Model Binding** and **Validation** protect your API
- How **Error Handling** and **Logging** make your app production-ready
- How to **structure a real-world application** using the Modular Monolith pattern

These aren't just concepts — they're the exact patterns used in production applications like **NextEvent** and **Normora**.
