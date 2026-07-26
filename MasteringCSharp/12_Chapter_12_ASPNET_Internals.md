# Chapter 12: ASP.NET Core Integration & Internals

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Trace the lifecycle of an HTTP request through the Kestrel web server.
- Evaluate the internal performance differences between Minimal APIs and traditional MVC Controllers.
- Architect scalable Dependency Injection (DI) lifetimes (Singleton, Scoped, Transient).
- Build the foundation of a robust, secured, and heavily customized Middleware pipeline.

## 2. Introduction

Up to this point, we have focused heavily on the C# language, the runtime, and the CPU. However, enterprise systems do not run in isolation. They communicate over networks. For the last decade, the standard for building cloud-native web services in .NET has been **ASP.NET Core**.

ASP.NET Core is not the bloated System.Web of the early 2000s. It was completely rewritten from scratch to be one of the fastest web frameworks on the planet (frequently topping the TechEmpower benchmarks). 

As a Software Architect, you must understand how ASP.NET Core leverages everything we've learned so far—`Span<T>`, `ValueTask`, the ThreadPool, and Asynchronous State Machines—to process millions of HTTP requests per second.

## 3. The Kestrel Web Server Internals

When you run an ASP.NET Core application, you are starting a web server named **Kestrel**.

Kestrel is an asynchronous, cross-platform, event-driven I/O server.
When an HTTP request arrives over the network:
1. The OS notifies Kestrel via IOCP (Windows) or epoll (Linux).
2. Kestrel does **not** allocate a massive byte array. It uses `System.IO.Pipelines` (an evolution of `Span<T>` and `Memory<T>`) to read the raw bytes directly from the socket into chained memory buffers.
3. Kestrel parses the HTTP headers using zero-allocation algorithms.
4. Kestrel wraps this data in an `HttpContext` object and hands it off to the ThreadPool.
5. A ThreadPool thread begins executing your Application Pipeline.

## 4. The Request Pipeline and Middleware

ASP.NET Core uses a Russian-doll model called the **Middleware Pipeline**.
Every request flows through a series of delegates. Each middleware component can process the request, pass it to the `next` component, and then process the response on the way back out.

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Custom Middleware 1: Execution Time Logger
app.Use(async (context, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    
    // Call the next middleware in the pipeline
    await next(context); 
    
    // Executes on the way OUT
    Console.WriteLine($"Request to {context.Request.Path} took {sw.ElapsedMilliseconds}ms");
});

// Terminal Middleware (Ends the pipeline)
app.MapGet("/", () => "Hello, Enterprise!");

app.Run();
```

**Architectural Rule:** Order matters intensely. You must place Authentication before Authorization, and both must be placed before your Endpoint Routing (so you don't execute business logic for unauthenticated users).

## 5. The Dependency Injection (DI) Container

Modern software uses **Inversion of Control**. Instead of a class instantiating its own dependencies (`var db = new SqlDatabase()`), it asks the framework to provide them via the constructor.

ASP.NET Core includes a built-in DI container (`IServiceProvider`). To master it, you must understand the three object lifetimes:

### 1. Transient
A new instance is created **every single time** it is requested. 
- *Use Case:* Lightweight, stateless utility classes.
- *Danger:* Heavy GC pressure if you request many of them per HTTP request.

### 2. Scoped
A new instance is created **once per HTTP Request**.
- *Use Case:* Entity Framework Core `DbContext`, Repository classes, User Contexts.
- *Danger:* If you inject a Scoped service into a Singleton, the Scoped service becomes trapped forever, leaking memory and sharing data across requests (Captive Dependency).

### 3. Singleton
A single instance is created when the app starts and is shared by **every HTTP Request**.
- *Use Case:* In-memory caches, background queue channels, Redis connection multiplexers.
- *Danger:* Must be strictly **Thread-Safe**. If two requests hit a Singleton simultaneously, race conditions will occur unless you use locks or concurrent collections (as covered in Chapter 10).

```csharp
// Registering services
builder.Services.AddTransient<ITelemetryParser, FastTelemetryParser>();
builder.Services.AddScoped<IEvDatabase, SqlEvDatabase>();
builder.Services.AddSingleton<ICache, RedisCache>();
```

## 6. Minimal APIs vs MVC Controllers

Historically, developers built APIs using MVC Controllers.
```csharp
[ApiController]
[Route("api/[controller]")]
public class ChargerController : ControllerBase
{
    private readonly IEvDatabase _db;
    public ChargerController(IEvDatabase db) => _db = db;

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(string id)
    {
        return Ok(await _db.GetChargerAsync(id));
    }
}
```

**The Cost of MVC:**
Controllers rely heavily on Reflection. At startup, the framework scans your assembly, finds all classes ending in `Controller`, inspects their `[HttpGet]` attributes, and builds a routing tree. At runtime, the framework uses Reflection to instantiate the Controller and invoke the method, adding significant CPU overhead.

**Minimal APIs (C# 10+):**
Minimal APIs bypass the MVC framework entirely. They map a URL directly to a Delegate.

```csharp
app.MapGet("/api/chargers/{id}", async (string id, IEvDatabase db) => 
{
    return await db.GetChargerAsync(id);
});
```

**Compiler Internals:**
Using C# Source Generators, the compiler analyzes the Delegate at build time. It generates hardcoded mapping logic that retrieves `id` from the route and `db` from the DI container. There is zero Reflection at runtime. This results in dramatically lower memory allocation and higher requests-per-second (RPS).

## 7. Real Production Case Study: Secure API Endpoints

In our EV Platform, we will expose a highly optimized Minimal API to start a charging session. It will use a Scoped database, a Singleton cache, and enforce Role-Based Access Control (RBAC).

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using System.Security.Claims;

var builder = WebApplication.CreateBuilder(args);

// Setup DI
builder.Services.AddSingleton<RedisConnection>();
builder.Services.AddScoped<ChargingService>();
builder.Services.AddAuthorization();
builder.Services.AddAuthentication("JwtBearer").AddJwtBearer();

var app = builder.Build();

// Setup Pipeline
app.UseAuthentication();
app.UseAuthorization();

// Route Grouping with strict Authorization
var api = app.MapGroup("/api/v1").RequireAuthorization("AdminPolicy");

// Minimal API Endpoint
api.MapPost("/chargers/{id}/start", async (
    string id, 
    ChargingService service, 
    ClaimsPrincipal user) =>
{
    var tenantId = user.FindFirst("TenantId")?.Value;
    if (tenantId == null) return Results.Unauthorized();

    var result = await service.StartSessionAsync(tenantId, id);
    
    return result.Success 
        ? Results.Ok(result.SessionId) 
        : Results.BadRequest(result.Error);
});

app.Run();
```

## 8. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Putting complex business logic directly in the API endpoint. | Untestable code. | Keep endpoints thin. Delegate logic to Scoped Service classes. |
| Intermediate | Captive Dependencies (Injecting a Scoped DbContext into a Singleton BackgroundService). | Data corruption. `DbContext` is not thread-safe. One request's data bleeds into another. | Create an `IServiceScopeFactory` in the Singleton, and manually create a scope when needed. |
| Senior | Performing synchronous I/O (like `Thread.Sleep` or `.Result`) inside a middleware. | Thread Pool starvation. The web server will stop responding to new requests under load. | Always use `async/await` for everything in the pipeline. |
| Architect | Ignoring payload sizes on Kestrel buffers. | Large JSON POSTs can cause LOH allocations, triggering Gen 2 GC pauses. | Enforce strict request size limits (`[RequestSizeLimit]`). Stream large files directly to blob storage, never buffering them entirely in RAM. |

## 9. Summary
ASP.NET Core represents the culmination of all .NET performance optimizations. We explored how Kestrel leverages pipelines for raw network speed, how to construct a robust middleware chain, and how the DI container manages object lifetimes. We also compared the reflection-heavy MVC paradigm to the modern, source-generated Minimal APIs.

Now that we have a high-performance HTTP entry point, we must structure the business logic behind it. In the next chapter, we will architect the core of the enterprise: Clean Architecture, CQRS, and Domain Events.
