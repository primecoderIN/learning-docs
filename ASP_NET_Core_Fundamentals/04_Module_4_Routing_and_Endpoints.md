# Module 4: Routing, Endpoints, Minimal APIs & Real-Time SignalR

Routing is the **GPS and dispatch system** of your web application. When an HTTP request arrives at your server saying `GET /api/events/42/tickets`, routing is the mechanism that looks at that URL, decodes it, and determines: *"This request should be handled by the `GetTickets` method of the `EventsController`, and the `id` parameter is `42`."*

For a software architect, routing is not just about string matching: it governs **RESTful API contracts, performance benchmarks, real-time bidirectional communication, defense-in-depth parameter constraints, and modern AI agent interfaces.**

---

## 1. What is Routing?

### The Core Problem
Your ASP.NET Core application has dozens (or hundreds) of C# methods that handle different operations: creating users, fetching products, deleting orders. An HTTP request is just a URL string and an HTTP verb. **Something** has to map `/api/users/5` to a specific `GetUserById(int id)` method.

That "something" is the **Router**.

> **Analogy — The Post Office:**  
> Think of routing like a post office. Letters (HTTP requests) arrive with an address (URL). The post office (router) reads the address, looks it up in its delivery registry, and dispatches the letter to the correct mailbox (C# method). If the address doesn't match any mailbox, the letter is returned with "Address Unknown" (404 Not Found).

### What Routing Does, Step by Step
1. **Receives** an incoming URL: `GET /api/products/42`
2. **Matches** it against all registered route templates: `"/api/products/{id:int}"`
3. **Extracts** parameters: `id = 42`
4. **Selects** the endpoint (the C# method or delegate to execute)
5. **Binds** the extracted values to the method's parameters

---

## 2. Endpoint Routing — The Modern Two-Stage Architecture

Prior to ASP.NET Core 3.0, routing was an internal subsystem of MVC (`app.UseMvc()`). MVC performed route matching, authentication, authorization, and action execution all inside a single monolithic block. This made it impossible for other middleware to inspect endpoint metadata.

Modern ASP.NET Core uses **Endpoint Routing**, which decouples routing into **two distinct pipeline stages**:

```
[ Incoming Request: GET /api/v1/documents/3fa85f64 ]
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. ROUTE MATCHING (app.UseRouting())                        │
│    - Evaluates URL path against registered route tree       │
│    - Finds matching Endpoint (e.g. GetDocumentByIdAction)   │
│    - Attaches Endpoint Metadata to HttpContext              │
│    - ⚠️ DOES NOT EXECUTE THE ENDPOINT YET!                  │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ INTERMEDIATE MIDDLEWARE (Can Inspect Endpoint Metadata)     │
│    - UseCors: Checks if endpoint has [EnableCors]           │
│    - UseAuthentication: Establishes ClaimsPrincipal        │
│    - UseRateLimiter: Checks endpoint rate limit policies    │
│    - UseAuthorization: Checks [Authorize] & [RequireTenant] │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ENDPOINT EXECUTION (app.MapControllers() / MapEndpoints) │
│    - Model binding parses route values & body payload       │
│    - Action filters execute                                 │
│    - Controller action / Minimal API lambda executes        │
└─────────────────────────────────────────────────────────────┘
```

### Why Split Them?
Because **middleware between routing and execution** can make decisions based on *which* endpoint was matched. This is revolutionary.

```csharp
app.UseRouting();           // "This request matches the GetSecretData endpoint"
app.UseAuthentication();    // "Who is the caller?"
app.UseAuthorization();     // "Does the caller have permission for GetSecretData?"
app.MapControllers();       // "Permission granted — execute GetSecretData"
```

Without this split, the Authorization middleware wouldn't know *which* endpoint needs checking until it's already executed — which is too late.

---

## 3. Controller Routing (Attribute Routing)

For larger APIs, Controllers are the standard approach. You annotate your classes and methods with route attributes.

### Basic Controller Setup
```csharp
[ApiController]                    // Enables API behaviors (auto 400, auto binding)
[Route("api/users")]              // Base route (explicit contract)
public class UsersController : ControllerBase
{
    private readonly IUserRepository _repository;

    public UsersController(IUserRepository repository)
    {
        _repository = repository;
    }

    // GET /api/users
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var users = await _repository.GetAllAsync();
        return Ok(users);
    }

    // GET /api/users/5
    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var user = await _repository.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }

    // POST /api/users
    [HttpPost]
    public async Task<IActionResult> Create(CreateUserDto dto)
    {
        var user = await _repository.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // PUT /api/users/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, UpdateUserDto dto)
    {
        await _repository.UpdateAsync(id, dto);
        return NoContent();
    }

    // DELETE /api/users/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _repository.DeleteAsync(id);
        return NoContent();
    }
}
```

### The `[ApiController]` Attribute — What It Actually Does
This single attribute enables **critical behaviors** that streamline API development:

| Behavior | What it does |
|---|---|
| **Automatic Model Validation** | If `ModelState` is invalid, automatically returns `400 Bad Request` before your code runs |
| **Binding Source Inference** | Complex types default to `[FromBody]`, simple types default to `[FromRoute]` or `[FromQuery]` |
| **Problem Details Responses** | Automatically formats error responses according to RFC 7807 Problem Details specification |
| **Multipart/Form-Data Inference** | Parameters of type `IFormFile` or `IFormFileCollection` automatically bind from `[FromForm]` |

### ⚠️ The Renaming Trap: `[Route("api/[controller]")]`
Many beginner templates use token replacement:
```csharp
[Route("api/[controller]")] // ❌ FRAGILE
public class TenantsController : ControllerBase
```
If a developer refactors or renames `TenantsController` to `WorkspacesController`, the public API route silently changes from `/api/tenants` to `/api/workspaces`, breaking external clients!

**Architectural Standard:** Always declare explicit, immutable URI paths:
```csharp
[Route("api/tenants")] // ✅ IMMUTABLE PRODUCTION CONTRACT
public class TenantsController : ControllerBase
```

---

## 4. Route Parameters & Defensive Constraints

Route constraints enforce data typing and structural invariants directly within the URL template. If the incoming URL fails the constraint, the routing engine does not match the endpoint, returning `404 Not Found` immediately.

```csharp
[HttpGet("{id:int}")] // Matches only valid integers: /api/users/42
public async Task<IActionResult> GetById(int id)

[HttpGet("{id:guid}")] // Matches only valid GUIDs: /api/documents/3fa85f64-...
public async Task<IActionResult> GetById(Guid id)

[HttpGet("slug/{slug:alpha:minlength(3)}")] // Alphabetical only, min length 3
public async Task<IActionResult> GetBySlug(string slug)

// Optional parameters (?) and default values:
[HttpGet("reports/{year:int?}")] // year is optional
public async Task<IActionResult> GetReports(int? year)

[HttpGet("search")] // query string parameters with defaults
public async Task<IActionResult> Search([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
```

### Why Route Constraints are an Architect's Shield:
1.  **Prevents Database Query Waste:** If a client requests `/api/tenants/invalid-text`, a `{id:guid}` constraint rejects the request at the routing level. The application never attempts to parse a string into a GUID or query the PostgreSQL database.
2.  **Disambiguates Overloaded Routes:** You can map `GET /api/documents/{id:guid}` to a fetch-by-id action, and `GET /api/documents/{tag:alpha}` to a search-by-tag action without route collision errors.

---

## 5. Minimal APIs: Architecture, Route Groups & Performance

Introduced in .NET 6 and expanded in .NET 8/9, **Minimal APIs** allow developers to define HTTP endpoints directly as lambda delegates, bypassing the entire MVC controller reflection pipeline.

### Architectural Performance Characteristics:
*   **Zero MVC Overhead:** Skips the Controller Activator, Action Descriptors, and filter pipeline allocation.
*   **Native AOT Ready:** Can be pre-compiled into native machine code (Ahead-Of-Time compilation) with zero JIT overhead and instant cold starts.
*   **Low Allocation:** Benchmark tests demonstrate up to 3x higher raw throughput and significantly reduced GC allocations compared to standard MVC controllers for simple operations.

### Clean Organization with Route Groups
Senior architects organize Minimal APIs using **Route Groups** (`MapGroup`) to apply shared security, prefixes, and tags across modular slices:

```csharp
public static class DocumentsEndpointExtensions
{
    public static RouteGroupBuilder MapDocumentsEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/documents")
            .WithTags("Documents")
            .RequireAuthorization("RequireTenantAccess"); // Applies policy to all routes in group

        group.MapGet("/", async (IMediator mediator, CancellationToken ct) =>
        {
            var results = await mediator.Send(new GetDocumentsQuery(), ct);
            return TypedResults.Ok(results);
        });

        group.MapPost("/", async (CreateDocumentDto dto, IMediator mediator, CancellationToken ct) =>
        {
            var created = await mediator.Send(new CreateDocumentCommand(dto), ct);
            return TypedResults.Created($"/api/v1/documents/{created.Id}", created);
        });

        return group;
    }
}
```

---

## 6. HTTP Return Types and Status Codes

Understanding what to return from your endpoints is crucial for building proper REST APIs:

### Controller Return Types
```csharp
return Ok(data);                    // 200 — Success with data
return NoContent();                 // 204 — Success, no data to return (PUT/DELETE)
return CreatedAtAction(nameof(GetById), new { id = data.Id }, data); // 201 — Resource created with Location header
return BadRequest(errors);          // 400 — Client sent invalid data
return Unauthorized();              // 401 — Identity not authenticated
return Forbid();                    // 403 — Authenticated but unauthorized
return NotFound();                  // 404 — Resource does not exist
return Conflict(message);           // 409 — Business rule or concurrency conflict
```

### Minimal API Return Types
```csharp
return TypedResults.Ok(data);
return TypedResults.NoContent();
return TypedResults.Created($"/api/documents/{id}", data);
return TypedResults.BadRequest(errors);
return TypedResults.NotFound();
return TypedResults.Problem(detail: "Database timeout", statusCode: 500);
```

---

## 7. Real-Time Endpoints with SignalR: WebSockets at Enterprise Scale

REST APIs follow a strict request-response paradigm: the client must initiate every interaction. For real-time applications — live document processing indicators, notifications, chat, and agent streaming — ASP.NET Core provides **SignalR**.

SignalR abstracts transport protocols (WebSockets, Server-Sent Events, Long Polling) behind a strongly-typed **Hub** abstraction.

### 🏗️ SignalR Routing in Normora:
Normora maps multiple dedicated real-time hubs in `Program.cs`:

```csharp
// From Normora's Program.cs
app.MapHub<DocumentHub>("/hubs/documents").AsBffApiEndpoint();
app.MapHub<NotificationHub>("/hubs/notifications").AsBffApiEndpoint();
```

Notice the `.AsBffApiEndpoint()` modifier: this ensures that WebSocket upgrade requests validate the client's Duende BFF anti-forgery tokens and cookie sessions!

### ⚠️ The Multi-Node Scaling Problem: Why SignalR Needs a Backplane

In production, you run multiple container replicas of your API behind a load balancer:

```
[ User Alice ] ── (WebSocket Conn) ──▶ [ API Pod 1 ]
                                              ⚡ (How does Pod 1 tell Pod 2?)
[ User Bob ]   ── (WebSocket Conn) ──▶ [ API Pod 2 ]
```

If User Alice uploads a document on **Pod 1**, and a background worker completes its processing, Pod 1 attempts to broadcast `Clients.User(Bob).SendAsync("DocumentReady")`. But Bob is connected to **Pod 2**! Pod 1 has no idea Bob exists, and the real-time notification is silently dropped.

### 🛡️ The Architectural Fix: Redis Backplane

Normora scales SignalR horizontally across multi-replica clusters by configuring a **Redis Backplane**. When Pod 1 publishes a message, it writes to a Redis Pub/Sub channel. Every API pod subscribes to Redis, receives the broadcast, and dispatches it to any locally connected client.

```csharp
// From Normora.Infrastructure/Extensions/ApplicationServiceExtensions.cs
public static IServiceCollection AddNormoraSignalR(
    this IServiceCollection services, 
    IOptions<RedisOptions> redisOptions)
{
    var signalRBuilder = services.AddSignalR(options =>
    {
        options.EnableDetailedErrors = false; // Security: Never leak internals in production
    });

    // If Redis connection is configured, wire the distributed Redis backplane
    if (!string.IsNullOrWhiteSpace(redisOptions.Value.ConnectionString))
    {
        signalRBuilder.AddStackExchangeRedis(redisOptions.Value.ConnectionString, options =>
        {
            // Dedicated channel prefix to prevent collisions with other Redis databases
            options.Configuration.ChannelPrefix = StackExchange.Redis.RedisChannel.Literal(redisOptions.Value.ChannelPrefix);
        });
    }

    return services;
}
```

---

## 8. Modern AI Agent Endpoints: CopilotKit & AG-UI

As enterprise software incorporates AI agents, backend architectures must host streaming protocols designed for multi-turn AI interactions and tool invocations.

In **Normora**, the API integrates the **Microsoft Agent Framework** to power workplace intelligence agents via CopilotKit. Rather than writing custom WebSocket parsers, the endpoint is routed cleanly into the ASP.NET Core pipeline:

```csharp
// From Normora's Program.cs
// Microsoft Agent Framework AG-UI endpoint for CopilotKit Workplace Intelligence:
app.MapNormoraAguiServer()
    .RequireAuthorization()
    .AsBffApiEndpoint();
```

---

## 9. Real-World Routing from Your Projects

### 🏗️ NextEvent: Lowercase URLs & Configurable CORS Origins
```csharp
// From NextEvent's ApiServiceExtensions.cs
services.AddRouting(options =>
{
    options.LowercaseUrls = true;  // /api/Users/5 → /api/users/5
});

// CORS origins loaded from configuration, not hardcoded
var allowedOrigins = configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [];
services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials()
              .WithOrigins(allowedOrigins);
    });
});
```

**Why `LowercaseUrls = true`?** REST API conventions dictate lowercase URLs. Without this, a controller named `UsersController` generates routes like `/api/Users` (uppercase U), which looks unprofessional and causes caching anomalies across CDNs.

### 🏗️ NextEvent: JSON Serialization Configuration
```csharp
// From NextEvent's ApiServiceExtensions.cs
services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.PropertyNameCaseInsensitive = true;
    });
```

*   **`CamelCase`:** C# properties (`FirstName`) are serialized as JSON (`firstName`), matching JavaScript frontend expectations.
*   **`PropertyNameCaseInsensitive`:** When receiving JSON from clients, `firstname`, `FirstName`, and `FIRSTNAME` all bind correctly, preventing frustrating 400 errors from casing mismatches.

### 🏗️ Modern API Documentation: Migrating from Swagger to Scalar
In **Normora**, API documentation is generated using native .NET 9 OpenAPI and exposed through modern **Scalar UI**, gated strictly to Development:

```csharp
// From Normora's Program.cs
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
    app.MapGet("/scalar", () => Results.Redirect("/scalar/v1")).ExcludeFromDescription();
}
```

---

## 10. Best Practices Summary

| Practice | Why |
|---|---|
| **Use `LowercaseUrls = true`** | Professional REST API convention, prevents CDN cache fragmentation |
| **Always use route constraints (`{id:guid}`)** | Shields database from garbage input, returns clean 404s |
| **Use `[ApiController]` on every API controller** | Enables auto-validation, binding inference, Problem Details |
| **Use `CamelCase` JSON** | Standard JavaScript convention for frontends |
| **Group Minimal API endpoints with `MapGroup`** | Keeps `Program.cs` organized and applies shared security |
| **Load CORS origins from configuration** | Same binary works across local dev, staging, and production |
| **Use `CreatedAtAction` for POST endpoints** | Returns proper `201 Created` with a `Location` header |
| **SignalR Redis Backplane** | Ensures multi-replica container pods route WebSockets to all users |
| **Gate Documentation in Production** | Never expose OpenAPI / Swagger / Scalar UIs publicly in production |

---
**Key Takeaways:**
1. Routing maps URLs to C# methods. Endpoint Routing splits matching from execution so authorization can run in between.
2. Controllers give you structure and features. Minimal APIs give you speed and simplicity.
3. Route constraints (`{id:guid}`) prevent type mismatches from reaching your database.
4. Scale real-time WebSockets across multi-node clusters using a Redis backplane.
5. Always configure `LowercaseUrls`, `CamelCase` JSON, and route constraints from day one.

**Next up:** In [Module 5](05_Module_5_Configuration_and_Options.md), we dive into **Configuration and the Options Pattern** — how to manage database credentials, API keys, and environment settings without hardcoding secrets.
