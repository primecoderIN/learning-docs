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

## 9. Interview Questions

### Beginner Tier (ASP.NET Core Basics)

**1. What is ASP.NET Core?**
*Answer:* It is a cross-platform, high-performance, open-source framework for building modern, cloud-based, internet-connected applications (APIs, web apps, microservices) using C# and .NET.

**2. What is Kestrel?**
*Answer:* Kestrel is the default, cross-platform web server for ASP.NET Core. It is highly optimized for asynchronous I/O and receives incoming HTTP requests from the network before passing them to the ASP.NET Core application pipeline.

**3. What is the `Program.cs` file used for in modern ASP.NET Core?**
*Answer:* It is the entry point of the application. It is used to configure the WebHost, register services in the Dependency Injection (DI) container, and build the HTTP request pipeline (Middleware).

**4. What is Dependency Injection (DI)?**
*Answer:* DI is a design pattern (Inversion of Control) where an object's dependencies are provided to it (usually via the constructor) rather than the object creating them itself. ASP.NET Core has a built-in DI container.

**5. Name the three DI service lifetimes in ASP.NET Core.**
*Answer:* Transient, Scoped, and Singleton.

**6. What is a Singleton service?**
*Answer:* A Singleton is created the very first time it is requested, and that exact same instance is shared across every single HTTP request for the lifetime of the application.

**7. What is the standard return type for an asynchronous ASP.NET Core endpoint?**
*Answer:* `Task<IResult>` or `Task<IActionResult>`.

### Intermediate Tier (Middleware and Lifetimes)

**8. What is Middleware?**
*Answer:* Middleware is software assembled into an application pipeline to handle requests and responses. Each component chooses whether to pass the request to the next component in the pipeline and can perform work before and after the next component in the chain is invoked.

**9. In what order is Middleware executed?**
*Answer:* It is executed in the exact order it is added in `Program.cs` (top to bottom for requests, and then bottom to top for responses on the way back out).

**10. What happens if a Middleware does NOT call `next(context)`?**
*Answer:* The pipeline short-circuits. The request does not proceed to the rest of the middleware or the endpoint logic, and the response immediately travels back up the pipeline to the client.

**11. Explain a Scoped service.**
*Answer:* A Scoped service is created once per client request (HTTP request). All classes that ask for this service *during that specific HTTP request* will receive the exact same instance.

**12. Explain a Transient service.**
*Answer:* A Transient service is created every single time it is requested from the DI container, even within the same HTTP request.

**13. What is a Captive Dependency?**
*Answer:* It occurs when a long-lived service (Singleton) injects a short-lived service (Scoped or Transient). The Singleton holds onto the Scoped service forever, effectively promoting it to a Singleton. If the Scoped service was a `DbContext`, it will cause thread-safety crashes.

**14. How do you access Configuration data (like `appsettings.json`)?**
*Answer:* Through the `IConfiguration` interface, which is automatically registered in the DI container. You can inject it and call `_config.GetValue<string>("MyKey")` or use the Options pattern.

### Senior Tier (Internals and Performance)

**15. Why are Minimal APIs significantly faster than traditional MVC Controllers?**
*Answer:* MVC Controllers rely heavily on runtime Reflection to inspect attributes (`[HttpGet]`), instantiate the controller, and invoke the method. Minimal APIs use C# Source Generators to inspect delegates at compile-time, emitting hardcoded mapping and DI resolution logic, entirely bypassing Reflection.

**16. What is the `HttpContext`?**
*Answer:* It is an object that encapsulates all information about an individual HTTP request and response. It contains the headers, body streams, user claims, and DI service provider specific to that single request lifecycle.

**17. Explain how Kestrel achieves massive concurrency without running out of memory.**
*Answer:* Traditional servers allocated a new `byte[]` for every request. Under load, this crushes the Garbage Collector. Kestrel uses `System.IO.Pipelines` and `MemoryPool<byte>`. It reads socket data directly into pinned, reusable buffers. As data is parsed, the buffer is returned to the pool, ensuring zero-allocation networking.

**18. What is the Options Pattern?**
*Answer:* Instead of injecting the massive `IConfiguration` interface into every class, the Options pattern binds a specific section of `appsettings.json` to a strongly typed C# class, and you inject `IOptions<MyConfig>` into your service. It adheres to the Interface Segregation Principle.

**19. What is the difference between `IOptions`, `IOptionsSnapshot`, and `IOptionsMonitor`?**
*Answer:* `IOptions` is a Singleton and does not detect changes to `appsettings.json`. `IOptionsSnapshot` is Scoped and reads the latest config file values at the start of each HTTP request. `IOptionsMonitor` is a Singleton that provides a callback event exactly when the config file changes.

**20. How do you run background tasks in ASP.NET Core?**
*Answer:* Implement the `IHostedService` interface or inherit from the `BackgroundService` base class, and register it using `builder.Services.AddHostedService<MyJob>()`. It runs independently of the HTTP request pipeline.

**21. Why is it dangerous to inject a Scoped service (like `DbContext`) into an `IHostedService`?**
*Answer:* `IHostedService` is a Singleton. Injecting a Scoped `DbContext` creates a Captive Dependency, which is strictly forbidden. Instead, you must inject `IServiceScopeFactory`, call `CreateScope()`, and manually resolve the `DbContext` within a `using` block inside your background loop.

### Staff Engineer Tier (Customizing the Pipeline)

**22. You need to log the total execution time of every HTTP request. How do you implement this?**
*Answer:* I would write a Custom Middleware. It would start a `Stopwatch`, `await next(context)`, stop the stopwatch on the line below the await, and log the elapsed time.

**23. What is the `IHttpContextAccessor` and why should you avoid it?**
*Answer:* It is a service that allows you to access the current `HttpContext` from deeply nested business logic classes. You should avoid it because it relies on `AsyncLocal<T>`, which has a performance cost, and it tightly couples your core domain logic to the web framework. Pass required data explicitly as method parameters instead.

**24. Explain the difference between `app.Use()` and `app.Run()` in Middleware.**
*Answer:* `app.Use()` adds a middleware delegate that can call `next()` to pass execution to the next component. `app.Run()` adds a terminal middleware delegate; it does not receive a `next` parameter and permanently ends the pipeline, forcing the response to travel back up to the client.

**25. How do you implement Global Exception Handling in modern ASP.NET Core?**
*Answer:* Instead of wrapping every endpoint in a try/catch, you register `app.UseExceptionHandler()`. In .NET 8+, you implement `IExceptionHandler` and register it in DI. If an unhandled exception occurs anywhere in the pipeline, the framework catches it, routes it to your handler, and you can format a standard JSON `ProblemDetails` response.

**26. What happens if a Middleware performs synchronous I/O?**
*Answer:* It will cause ThreadPool Starvation. Because middleware executes on ThreadPool threads handling HTTP requests, blocking a thread synchronously (e.g., `Thread.Sleep` or `.Result`) prevents the server from processing new incoming HTTP connections, leading to latency spikes or server hangs.

**27. Explain how the `[FromServices]` attribute (or Minimal API automatic DI) works internally.**
*Answer:* When building the endpoint mapping, the framework identifies parameters that are not in the URL/Body. It generates IL/Source code that calls `context.RequestServices.GetRequiredService<T>()`, extracting the required dependency from the specific Scoped DI container attached to that exact HTTP request.

### Architect Tier (Hosting and Extreme Scaling)

**28. You are deploying ASP.NET Core to a Kubernetes cluster that receives 10,000 requests per second. The pod keeps getting OOMKilled (Out of Memory) by Linux, even though C# memory profiling shows the heap is small. What is happening?**
*Answer:* Server Garbage Collection (Server GC) is enabled. Server GC creates a dedicated heap and GC thread for *every logical CPU core* on the host machine. If Kubernetes restricts the pod to 512MB RAM, but the host node has 64 cores, the CLR will allocate 64 heaps, immediately blowing past the 512MB limit. The solution is to upgrade to .NET 8 (which respects cgroup limits automatically) or switch to Workstation GC.

**29. What is a Reverse Proxy, and why should you use one (like NGINX or YARP) in front of Kestrel?**
*Answer:* While Kestrel is extremely fast, a Reverse Proxy provides edge capabilities like SSL Termination, Load Balancing across multiple ASP.NET Core pods, DDOS protection, and port sharing (running multiple apps on port 443). YARP (Yet Another Reverse Proxy) is Microsoft's ultra-fast proxy built entirely in C#.

**30. Explain the architectural trade-offs of NativeAOT in ASP.NET Core 8+.**
*Answer:* NativeAOT compiles the app directly to machine code, completely removing the JIT compiler. This drops startup time to milliseconds and reduces memory usage by 50% (ideal for AWS Lambda/Microservices). However, it absolutely forbids runtime Reflection (`Reflection.Emit`). Heavy legacy libraries (like older EF Core or Newtonsoft.Json) will crash. You must strictly use Source Generators for everything.

**31. How do you prevent large JSON payload attacks (Billion Laughs / Memory Exhaustion)?**
*Answer:* Configure Kestrel limits. Set `MaxRequestBodySize` to a reasonable limit (e.g., 2MB). For endpoints that explicitly need to accept massive file uploads, use the `[DisableRequestSizeLimit]` attribute, but strictly process the `IFormFile` using asynchronous streams directly to disk, never buffering the `byte[]` in memory.

**32. What is Response Compression and when is it a bad idea?**
*Answer:* Response Compression middleware compresses the JSON payload using GZIP or Brotli, saving network bandwidth. It is a bad idea (and a security vulnerability like CRIME/BREACH) if you compress a response that contains both attacker-controlled input and secret data (like an anti-forgery token) over HTTPS. It also trades CPU cycles for network bandwidth, which is bad if your servers are CPU-bound.

**33. How do you share Authentication Cookies across a Web Farm (multiple servers)?**
*Answer:* ASP.NET Core uses Data Protection to encrypt cookies. If Server A encrypts a cookie, Server B cannot decrypt it because they have different encryption keys in memory. The architect must configure a shared `IDataProtectionProvider` (e.g., storing the keys in Azure Key Vault or Redis) so all servers can encrypt/decrypt the same tokens.

**34. Explain the difference between `AddHttpClient` and instantiating `new HttpClient()` manually.**
*Answer:* Instantiating `new HttpClient()` manually for every request causes Socket Exhaustion (the OS runs out of available ports because closed sockets enter a TIME_WAIT state). Using `AddHttpClient` (IHttpClientFactory) delegates socket management to the framework, which pools the underlying `HttpMessageHandler` connections, drastically reducing port consumption and DNS stale caching issues.

## 10. Summary
ASP.NET Core represents the culmination of all .NET performance optimizations. We explored how Kestrel leverages pipelines for raw network speed, how to construct a robust middleware chain, and how the DI container manages object lifetimes. We also compared the reflection-heavy MVC paradigm to the modern, source-generated Minimal APIs.

Now that we have a high-performance HTTP entry point, we must structure the business logic behind it. In the next chapter, we will architect the core of the enterprise: Clean Architecture, CQRS, and Domain Events.
