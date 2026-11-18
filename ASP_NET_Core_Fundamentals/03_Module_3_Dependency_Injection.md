# Module 3: Dependency Injection, Lifetimes & Architectural Invariants

If Middleware is the *soul* of ASP.NET Core, **Dependency Injection (DI) is the skeleton**. It holds everything together. Every middleware, every controller, every service, every database context — they are all created, connected, and destroyed by the DI container. Understanding DI is non-negotiable.

In enterprise software architecture, understanding DI goes far beyond knowing how to register an interface. You must master **object lifetimes, allocation dynamics, thread safety, the captive dependency trap, and automated architectural rules enforcement.**

---

## 1. What Problem Does DI Solve?

### The Problem: Tight Coupling
Imagine you're building an `OrderService` that needs to send emails when an order is placed. The naive approach:

```csharp
// ❌ TIGHTLY COUPLED — OrderService creates its own dependencies
public class OrderService
{
    public void PlaceOrder(Order order)
    {
        // Business logic...
        
        // Send confirmation email
        var emailSender = new SmtpEmailSender("smtp.gmail.com", 587, "password123");
        emailSender.Send(order.CustomerEmail, "Order Confirmed!");
    }
}
```

**What's wrong with this?**
1. **Can't test it.** To unit test `PlaceOrder`, you'd actually send real emails over the network!
2. **Can't swap implementations.** If you switch from Gmail SMTP to SendGrid or AWS SES, you must find and change every `new SmtpEmailSender(...)` in your entire codebase.
3. **Hard-coded secrets.** The SMTP password is buried directly inside business logic.
4. **Lifecycle chaos.** Who disposes of the `SmtpEmailSender`? What if it should be shared? What if it's expensive to create?

> **Analogy — The Chef Who Grows His Own Tomatoes:**  
> Imagine a chef who, instead of ordering tomatoes from a supplier, personally farms, harvests, and transports tomatoes every time a customer orders pasta. The chef is now tightly coupled to farming — he can't focus on cooking!  
>  
> DI is like hiring a supplier (the DI container). The chef says, "I need tomatoes" (an interface), and the supplier delivers them. The chef doesn't care if they come from a local farm or an imported batch — he just cooks.

### The Solution: Inversion of Control
Instead of a class creating its own dependencies, **you invert the control** — the framework creates and provides them.

```csharp
// ✅ LOOSELY COUPLED — Dependencies are injected
public class OrderService
{
    private readonly IEmailSender _emailSender;

    // The constructor DECLARES what it needs (an IEmailSender).
    // It doesn't know or care about the underlying implementation.
    public OrderService(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public void PlaceOrder(Order order)
    {
        // Business logic...
        _emailSender.Send(order.CustomerEmail, "Order Confirmed!");
    }
}
```

Now:
1. **Testable.** In unit tests, pass a `FakeEmailSender` that records emails in memory instead of sending them.
2. **Swappable.** Change the implementation in one place (the DI registration in `Program.cs`).
3. **Clean.** No hard-coded configuration values or connection strings in business logic.

---

## 2. The Built-in DI Container: Two Distinct Phases

Unlike Spring (Java) or older .NET where you needed third-party containers (Autofac, Ninject, Unity), ASP.NET Core ships with a **built-in, high-performance DI container**. It operates in **two strictly separated phases**:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. REGISTRATION PHASE (Mutable)                             │
│    IServiceCollection (builder.Services)                    │
│    - Register interfaces, classes, and factory delegates   │
│    - Configure service lifetimes                            │
└─────────────────────────────────────────────────────────────┘
                               │
                builder.Build() │ FREEZES CONTAINER
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. RESOLUTION PHASE (Immutable & Thread-Safe)               │
│    IServiceProvider (app.Services)                          │
│    - Resolves object graphs on demand                      │
│    - Manages scope boundaries and object disposal           │
└─────────────────────────────────────────────────────────────┘
```

| Interface | Purpose |
|---|---|
| `IServiceCollection` | The **registration** side — you tell the container what services exist (`builder.Services`) |
| `IServiceProvider` | The **resolution** side — the container creates and provides instances (`app.Services`) |

```csharp
var builder = WebApplication.CreateBuilder(args);

// ── REGISTRATION PHASE ──
// "Dear container, when someone asks for IEmailSender, create a SmtpEmailSender."
builder.Services.AddScoped<IEmailSender, SmtpEmailSender>();

var app = builder.Build();
// ── RESOLUTION PHASE ──
// From this point on, the container is locked and read-only. No more registrations allowed.
```

---

## 3. Service Lifetimes — The Most Critical Concept

When you register a service, you must tell the container **how long** each instance should live. This is called the **Lifetime**. Getting this wrong causes memory leaks, thread collisions, data corruption, and cross-tenant security breaches.

### Transient — New Instance Every Time
```csharp
builder.Services.AddTransient<ITaxCalculator, StandardTaxCalculator>();
```
*   **What:** A **brand new instance** is created every time any class requests it. If 3 classes in the same request ask for `ITaxCalculator`, they each get a different instance.
*   **When to use:** Lightweight, stateless utility services. Things that don't hold any state between calls.
*   **Memory impact:** Higher — many instances created and garbage collected.
*   **Thread safety:** Not a concern — each consumer gets its own private instance.

> **Analogy:** Disposable paper cups. Every time you want water, you grab a new cup.

### Scoped — One Instance Per HTTP Request
```csharp
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
```
*   **What:** A **single instance** is created when the first class in an HTTP request asks for it. Every subsequent request for the same service **within the same HTTP request** gets the **exact same instance**. When the HTTP request ends, the instance is automatically disposed.
*   **When to use:** This is the **most common lifetime** for web applications. EF Core's `DbContext` uses Scoped by default, ensuring all database operations in a single request share the same connection and transaction.
*   **Memory impact:** Moderate — exactly one instance per request.
*   **Thread safety:** Generally safe — web requests are typically processed single-threaded.

> **Analogy:** A table at a restaurant. Everyone seated at the same table (same HTTP request) shares the same waiter. Different tables (different requests) get different waiters.

### Singleton — One Instance Forever
```csharp
builder.Services.AddSingleton<ICacheService, InMemoryCacheService>();
```
*   **What:** A **single instance** is created the very first time it's requested, and that **same instance** is shared across **every HTTP request** for the entire lifetime of the application.
*   **When to use:** Caching services (Redis), configuration wrappers (`IOptions<T>`), `HttpClient` factories, and services that are expensive to create.
*   **Memory impact:** Minimal — only one instance ever.
*   **Thread safety:** **CRITICAL!** Multiple concurrent requests will access the same instance simultaneously. You **must** ensure thread safety (use `ConcurrentDictionary`, `lock`, or immutable data).

> **Analogy:** The restaurant's kitchen. There's only one kitchen shared by every customer. If two waiters bring orders simultaneously, the kitchen must handle them without mixing up dishes (thread safety).

### Visual Summary

```
HTTP Request 1                           HTTP Request 2
┌─────────────────────────────┐          ┌─────────────────────────────┐
│ Transient A (Instance #1)   │          │ Transient A (Instance #3)   │
│ Transient A (Instance #2)   │          │ Transient A (Instance #4)   │
│                             │          │                             │
│ Scoped B (Instance #1)      │          │ Scoped B (Instance #2)      │
│ Scoped B (Instance #1) ◄────┼── Same   │ Scoped B (Instance #2) ◄────┼── Same for this request
│                             │          │                             │
│ Singleton C (Instance #1)   │          │ Singleton C (Instance #1)   │
│ Singleton C (Instance #1) ◄─┴──────────┴─────────────────────────────┴── Shared across ALL requests
└─────────────────────────────┘          └─────────────────────────────┘
```

---

## 4. How to Register Services

### Basic Registration
```csharp
// Interface → Implementation
builder.Services.AddTransient<ITaxCalculator, StandardTaxCalculator>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<ICacheService, InMemoryCacheService>();
```

### Registering Without an Interface
Sometimes you don't need an interface (e.g., concrete utility classes):
```csharp
builder.Services.AddTransient<PdfGenerator>();
```

### Factory Registration (Complex Creation Logic)
When construction requires dynamic branching or configuration inspection:
```csharp
builder.Services.AddScoped<IPaymentGateway>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    var environment = config["Environment"];
    
    return environment == "Production"
        ? new StripePaymentGateway(config["Stripe:ApiKey"])
        : new FakePaymentGateway(); // Use fake in development
});
```

---

## 5. How to Resolve (Consume) Services

### Constructor Injection — The Standard Way
ASP.NET Core automatically examines your constructor, finds registered services matching parameter types, and injects them.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repository;
    private readonly IEmailSender _emailSender;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        IOrderRepository repository,
        IEmailSender emailSender,
        ILogger<OrdersController> logger)
    {
        _repository = repository;
        _emailSender = emailSender;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderDto dto)
    {
        var order = await _repository.CreateAsync(dto);
        await _emailSender.Send(order.Email, "Order confirmed!");
        _logger.LogInformation("Order {OrderId} created", order.Id);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}
```

### Minimal API Injection — Method Parameter Injection
With Minimal APIs, you inject directly into the handler delegate:

```csharp
app.MapPost("/api/orders", async (
    CreateOrderDto dto,
    IOrderRepository repository,
    IEmailSender emailSender) =>
{
    var order = await repository.CreateAsync(dto);
    await emailSender.Send(order.Email, "Order confirmed!");
    return Results.Created($"/api/orders/{order.Id}", order);
});
```

### `[FromServices]` — Inject Into a Specific Action
When only one specific action method needs a heavy service:
```csharp
[HttpGet("report")]
public IActionResult GenerateReport([FromServices] IReportGenerator generator)
{
    return Ok(generator.Generate());
}
```

### Manual Resolution with `IServiceProvider`
For startup code or background processing:
```csharp
using var scope = app.Services.CreateScope();
var repository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
```

---

## 6. The Captive Dependency Trap ⚠️

This is the **#1 DI architectural pitfall**. It occurs when a **longer-lived service captures a shorter-lived service**.

```csharp
// ❌ DANGEROUS: Singleton captures a Scoped service
builder.Services.AddSingleton<IBackgroundProcessor, BackgroundProcessor>();
builder.Services.AddScoped<AppDbContext>(); // Scoped = one per request

public class BackgroundProcessor : IBackgroundProcessor
{
    private readonly AppDbContext _dbContext; // ⚠️ Captured!

    public BackgroundProcessor(AppDbContext dbContext)
    {
        _dbContext = dbContext; // This DbContext will NEVER be disposed
    }
}
```

### What Happens in Production:
1.  `BackgroundProcessor` is created once as a Singleton on startup.
2.  It captures the `DbContext` that was meant to live for a single request.
3.  **Memory Leak:** EF Core accumulates tracked entities in memory forever, eventually causing an Out-Of-Memory (OOM) crash.
4.  **Multithreaded Crash:** When Request A and Request B call `BackgroundProcessor` simultaneously, both execute queries on the same `DbContext`. EF Core throws:  
    `InvalidOperationException: A second operation was started on this context instance before a previous operation completed.`
5.  **Tenant Data Contamination:** If a Singleton captures `ITenantContext`, Tenant B executes operations using Tenant A's cached tenant ID!

### The DI Container Rule:
```
Transient  ──▶  Can safely consume: Transient, Scoped, Singleton
Scoped     ──▶  Can safely consume: Scoped, Singleton
Singleton  ──▶  Can ONLY consume: Singleton
```

> **Analogy:** A paper cup (Transient) can hold water from any source. A reusable bottle (Scoped) should only use filtered water. A water tower (Singleton) must only connect to the main supply — it can't depend on individual bottles that come and go.

### The Fix: Manual Scoping via `IServiceScopeFactory`
When a Singleton needs a Scoped service, inject `IServiceScopeFactory` and construct a manual scope:

```csharp
public class BackgroundProcessor : IBackgroundProcessor
{
    private readonly IServiceScopeFactory _scopeFactory;

    public BackgroundProcessor(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory; // Singleton-safe!
    }

    public async Task ProcessAsync()
    {
        // Create an explicit, temporary scope just for this operation
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        // ... use dbContext safely ...
        await dbContext.SaveChangesAsync();
        // Disposed automatically when using block exits!
    }
}
```

---

## 7. Real-World DI Patterns from Your Projects

### 🏗️ NextEvent: Modular Service Registration
NextEvent bundles DI registrations into **extension methods by architectural layer**. This keeps `Program.cs` scannable:

```csharp
// From NextEvent's API/Extensions/ApplicationServiceExtensions.cs
public static class ApplicationServiceExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // 1. FluentValidation: Auto-discover all validator classes across modules
        services.AddValidatorsFromAssemblyContaining<CreateEventCommandValidator>();
        services.AddValidatorsFromAssemblyContaining<CreateOrganizationCommandValidator>();
        services.AddValidatorsFromAssemblyContaining<RegisterCommandValidator>();

        // 2. Typed HttpClient for external API calls
        services.AddHttpClient<IGeminiService, GeminiService>();

        // 3. Singleton clock — mockable in tests
        services.AddSingleton<IDateTimeProvider, SystemDateTimeProvider>();

        // 4. MediatR: Register all CQRS handlers across modules
        services.AddMediatR(x =>
        {
            x.RegisterServicesFromAssemblyContaining<GetEventsListQueryHandler>();
            x.RegisterServicesFromAssemblyContaining<GetOrganizationByIdQueryHandler>();
            x.RegisterServicesFromAssemblyContaining<LoginCommandHandler>();

            // Bridge: FluentValidation ↔ MediatR
            x.AddOpenBehavior(typeof(ValidationBehavior<,>));
        });

        return services;
    }
}
```

### 🏗️ NextEvent: Manual Scope for Database Seeding
During startup, no HTTP request scope exists. NextEvent creates a manual scope to migrate and seed:

```csharp
// From NextEvent's Program.cs
using var scope = app.Services.CreateScope();
var services = scope.ServiceProvider;

try
{
    var identityContext = services.GetRequiredService<IdentityDbContext>();
    var userManager = services.GetRequiredService<UserManager<User>>();
    var roleManager = services.GetRequiredService<RoleManager<IdentityRole>>();

    identityContext.Database.Migrate();
    await IdentityDataSeeder.SeedAsync(roleManager, userManager);
}
catch (Exception ex)
{
    var logger = services.GetRequiredService<ILogger<Program>>();
    logger.LogCritical(ex, "A critical error occurred during database initialization.");
    throw; // FAIL FAST — don't start a broken application
}
```

### 🏗️ Normora: Scoped Tenant Context
Normora registers `TenantContext` as **Scoped**, ensuring each HTTP request gets its own isolated tenant state:

```csharp
// From Normora's TenantContext.cs
public class TenantContext : ITenantContext
{
    public Guid? TenantId { get; private set; }
    public string? TenantRole { get; private set; }
    public bool IsTenantResolved => TenantId.HasValue;

    public void SetContext(Guid tenantId, string role, IReadOnlyCollection<Guid> departments)
    {
        // Prevent accidental double-setting within the same request
        if (IsTenantResolved)
            throw new InvalidOperationException("Tenant context is already set for this request.");

        TenantId = tenantId;
        TenantRole = role;
    }
}
```

**Why Scoped?** If it were Singleton, Request A from Tenant "Acme" would overwrite the context, and Request B from Tenant "Globex" would see Acme's confidential data. **Scoped guarantees each request gets its own clean `TenantContext` instance.**

### 🏗️ NextEvent: Abstracted Clock (`IDateTimeProvider`)
Instead of calling `DateTime.UtcNow` directly:

```csharp
// Interface (in Shared project)
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

// Real implementation (registered as Singleton)
public class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

// In unit tests, freeze time deterministically:
public class FakeDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow { get; set; } = new(2026, 1, 1);
}
```

---

## 8. Architectural Fitness Functions: Enforcing DI Invariants in CI/CD

In **Normora**, the test suite `Normora.Architecture.Tests` programmatically inspects compiled assemblies during CI/CD to enforce architectural invariants:

### Rule 1: Controllers Must NEVER Inject `DbContext` Directly
```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void Controllers_MustNeverInject_DbContexts()
{
    var assemblies = new[] { ConversationsAssembly, DocumentsAssembly, TenantsAssembly };

    var controllerTypes = assemblies
        .SelectMany(a => a.GetTypes())
        .Where(t => t.IsClass && !t.IsAbstract && typeof(ControllerBase).IsAssignableFrom(t))
        .ToList();

    Assert.NotEmpty(controllerTypes);

    foreach (var controller in controllerTypes)
    {
        var constructors = controller.GetConstructors();
        foreach (var ctor in constructors)
        {
            foreach (var param in ctor.GetParameters())
            {
                Assert.False(
                    typeof(DbContext).IsAssignableFrom(param.ParameterType),
                    $"Controller '{controller.FullName}' must not inject DbContext '{param.ParameterType.Name}'. " +
                    $"Controllers must use MediatR contracts or module service abstractions.");
            }
        }
    }
}
```

### Rule 2: Entity Configurations Must Have Parameterless Constructors
```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void EntityConfigurations_MustHaveParameterlessConstructors()
{
    var assemblies = new[] { ConversationsAssembly, DocumentsAssembly, TenantsAssembly };

    var configTypes = assemblies
        .SelectMany(a => a.GetTypes())
        .Where(t => t.IsClass && !t.IsAbstract && t.GetInterfaces()
            .Any(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IEntityTypeConfiguration<>)))
        .ToList();

    Assert.NotEmpty(configTypes);

    foreach (var configType in configTypes)
    {
        var defaultCtor = configType.GetConstructor(Type.EmptyTypes);
        Assert.True(
            defaultCtor != null,
            $"Entity configuration '{configType.FullName}' must have a parameterless constructor to allow auto-discovery via ApplyConfigurationsFromAssembly.");
    }
}
```

---

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Program to interfaces, not implementations** | Swappable, testable, decoupled |
| **Use Scoped for database contexts** | One connection per request, auto-disposed |
| **Use Singleton only for thread-safe services** | Protect concurrent requests from race conditions |
| **Never inject Scoped into Singleton** | Captive dependency → memory leaks & concurrency crashes |
| **Use `IServiceScopeFactory` in Singletons** | Create manual scopes safely |
| **Group registrations into extension methods** | Clean `Program.cs`, maintainable code |
| **Use Assembly Scanning** | Auto-discover validators, handlers, and mappings |
| **Abstract `DateTime.UtcNow` behind an interface** | Enables pure, deterministic unit testing of time-based logic |
| **Automate Architecture Tests** | Programmatically enforce DI and layer boundaries in CI/CD |

---
**Key Takeaways:**
1. DI removes the `new` keyword from your business logic — the container creates, provides, and manages everything.
2. Lifetimes (Transient/Scoped/Singleton) determine how long instances live — get this wrong and you'll leak memory or crash under load.
3. The Captive Dependency trap is the #1 DI pitfall — Singletons must never capture Scoped services.
4. Real projects use extension methods, assembly scanning, and manual scoping.
5. Senior architects use architecture tests to guarantee controllers never bypass CQRS or inject database contexts directly.

**Next up:** In [Module 4](04_Module_4_Routing_and_Endpoints.md), we explore **Routing and Endpoints** — how ASP.NET Core matches incoming URLs to C# actions and powers real-time WebSockets with SignalR.
