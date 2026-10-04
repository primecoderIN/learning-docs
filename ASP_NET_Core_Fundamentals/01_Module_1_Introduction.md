# Module 1: Introduction to ASP.NET Core

## 1. What is ASP.NET Core?

Imagine you want to build a house. You need a foundation, walls, plumbing, and electricity. You *could* build everything from raw bricks and copper wire yourself, or you could use a **construction framework** — a proven system of pre-built components that snap together so you can focus on designing your specific house.

ASP.NET Core is that construction framework, but for web applications. It is a **cross-platform, high-performance, open-source framework** built by Microsoft for creating modern web applications and APIs.

### A Brief History (Why It Exists)
*   **2002 – ASP.NET (Old):** Microsoft created ASP.NET, which ran only on Windows and only inside IIS (Internet Information Services). It was powerful but heavy, slow to start, and impossible to run on Linux or Mac.
*   **2016 – ASP.NET Core 1.0 (The Rewrite):** Microsoft threw away the old codebase and rewrote everything from scratch. The new framework was cross-platform (Windows, Mac, Linux), open-source, modular, and *blazingly fast*.
*   **Today – .NET 8/9/10:** ASP.NET Core is now the **only** web framework Microsoft actively develops. The old ASP.NET is considered legacy.

### What Can You Build With It?
| Type | Technology | Example |
|---|---|---|
| **REST APIs** | Controllers or Minimal APIs | A backend for an Angular/React/Mobile app |
| **Real-time Apps** | SignalR (WebSockets) | Live chat, notifications (like Normora's `NotificationHub`) |
| **Web Pages** | Razor Pages, MVC, Blazor | Server-rendered HTML websites |
| **Microservices** | Lightweight APIs in Docker | Individual services in a Kubernetes cluster |
| **Background Jobs** | Hosted Services, Hangfire | Scheduled email sending, document processing |

## 2. Why ASP.NET Core?

If you are deciding which backend technology to learn, here is why ASP.NET Core should be at the top of your list:

### Performance
ASP.NET Core routinely ranks at the **top of the TechEmpower benchmarks** (the industry-standard web framework benchmark). It is significantly faster than Node.js, Django, Ruby on Rails, and standard Java Spring.

> **Analogy:** Think of web frameworks like cars. Most frameworks are family sedans — they get the job done. ASP.NET Core is a Formula 1 car with the comfort of a sedan. You get raw speed *and* developer productivity.

### Cross-Platform & Open Source
*   Develop on Windows, Mac, or Linux.
*   Deploy to any cloud: Azure, AWS, GCP, or your own bare-metal servers.
*   The entire source code is on GitHub. You can read, contribute, or fork it.

### Unified Programming Model
Whether you are building an API endpoint, a web page, or a real-time WebSocket hub, the underlying building blocks are **exactly the same:**
*   **Middleware** (Module 2)
*   **Dependency Injection** (Module 3)
*   **Routing** (Module 4)
*   **Configuration** (Module 5)

Learn them once, use them everywhere.

### Built-in Dependency Injection
Unlike older .NET or Java frameworks where you needed to install third-party DI containers (Autofac, Ninject, Spring), ASP.NET Core includes a fast, lightweight DI container **out of the box**. Every single service in the framework is designed to be injected.

### Cloud-Ready by Design
ASP.NET Core apps are designed for Docker containers and cloud deployments from day one. They start fast, consume minimal memory, and integrate natively with health checks, configuration providers, and distributed tracing.

## 3. Where Does It Run? The Hosting Model

When you hear "web server," you might think of Apache or Nginx. ASP.NET Core has its own story.

### Kestrel: The Built-in Web Server

When you run an ASP.NET Core application, you are actually starting a **console application** that internally boots a web server called **Kestrel**.

> **Analogy:** Imagine Kestrel as a very efficient receptionist sitting at the front desk. Every time an HTTP request (a visitor) arrives, Kestrel receives it, converts it into a format the .NET code can understand, hands it to your application code, waits for the response, and sends it back to the visitor.

*   **What it does:** Listens on a port (e.g., `http://localhost:5000`), receives raw HTTP bytes, converts them into an `HttpContext` object, and passes them through the middleware pipeline.
*   **Why it's great:** Kestrel is asynchronous and non-blocking. It can handle **millions of requests per second** on modern hardware.
*   **What it lacks:** Advanced features like load balancing, SSL certificate management at scale, request buffering, and URL rewriting.

### The Reverse Proxy Pattern (Production)

Because Kestrel lacks some advanced features, in production environments, you place Kestrel **behind a Reverse Proxy** — a more mature web server that handles the internet-facing concerns.

```
[Internet] → [Nginx / IIS / Apache] → [Kestrel (.NET App)]
                 (Reverse Proxy)         (Internal Port)
```

*   **The Reverse Proxy handles:** SSL termination, IP filtering, request buffering, static file caching, load balancing across multiple Kestrel instances.
*   **Kestrel handles:** Running your .NET code as fast as possible.

#### 🏗️ Real-World: How Normora Does This
Normora runs behind Nginx inside Docker containers. Because Nginx is a separate container on Docker's internal network (not `localhost`), the API must explicitly configure forwarded headers so it knows the *real* client IP and the *real* URL scheme (`https`):

```csharp
// From Normora's Program.cs
var forwardedHeadersOptions = new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor 
                     | ForwardedHeaders.XForwardedHost 
                     | ForwardedHeaders.XForwardedProto
};
// Docker containers use internal IPs (172.x.x.x), not localhost.
// Without clearing these, ASP.NET Core silently ignores the headers!
forwardedHeadersOptions.KnownIPNetworks.Clear();
forwardedHeadersOptions.KnownProxies.Clear();
app.UseForwardedHeaders(forwardedHeadersOptions);
```

> **Why this matters:** If you skip this configuration, your app thinks every request comes from `172.18.0.3` (Docker's Nginx container) instead of the real user. OAuth redirect URIs break. Logging becomes useless. Rate limiting targets the wrong IP.

## 4. Anatomy of an ASP.NET Core Project

When you create a new ASP.NET Core Web API project, you get a handful of files. Let's understand every single one.

### `Program.cs` — The Heart of Everything

In modern .NET (6+), `Program.cs` is the single entry point. There is no `Startup.cs` anymore. Everything happens in this one file, broken into **three distinct phases:**

```csharp
// ============================================
// PHASE 1: THE BUILDER (Configure Services)
// ============================================
var builder = WebApplication.CreateBuilder(args);

// Register services into the Dependency Injection container.
// Think of this as "hiring staff" for your restaurant before opening.
builder.Services.AddControllers();         // Hire the "controllers" staff
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();          // Hire the "documentation" staff

// ============================================
// PHASE 2: THE APP (Configure Middleware Pipeline)
// ============================================
var app = builder.Build();

// Configure HOW requests flow through your application.
// Think of this as "designing the workflow" — who does what, in what order.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

// ============================================
// PHASE 3: RUN (Start Listening)
// ============================================
app.Run(); // Kestrel starts listening on the configured port
```

> **Analogy:** Think of opening a restaurant.
> - **Phase 1 (Builder):** You hire the chef, waiter, dishwasher, and security guard.
> - **Phase 2 (App):** You design the workflow — "When a guest arrives, the security guard checks them first, then the waiter seats them, then the chef prepares food."
> - **Phase 3 (Run):** You open the doors and start serving customers.

### `appsettings.json` — The Configuration File

This is a JSON file where you store settings that change between environments.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;..."
  },
  "AllowedOrigins": ["http://localhost:4200"]
}
```

*   **What:** Key-value pairs for your app's configuration.
*   **Where:** Database connection strings, API keys, feature flags, CORS origins.
*   **Why not hardcode?** Because the same code must work in Development, Staging, and Production with different settings. We cover this deeply in Module 5.
*   **Security Warning:** NEVER put real passwords or API keys in this file if it's committed to Git. Use **User Secrets** (development) or **Environment Variables** (production).

### The `.csproj` File — The Project Definition

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  </ItemGroup>
</Project>
```

*   **What:** An XML file that tells MSBuild how to compile your project.
*   **Why it matters:** It defines the target .NET version and lists all NuGet packages (third-party libraries) your project uses.

### `launchSettings.json` — Local Development Only

This file inside `Properties/` controls how the app launches locally (which ports, which environment name).

```json
{
  "profiles": {
    "https": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:7001;http://localhost:5001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

*   **Important:** This file is **never deployed** to production. Production settings come from environment variables or `appsettings.Production.json`.

## 5. Minimal APIs vs. Controllers

ASP.NET Core gives you two fundamentally different ways to define HTTP endpoints.

### Controllers (The Traditional Approach)
You create a C# class that inherits from `ControllerBase` and decorate methods with HTTP verb attributes.

```csharp
[ApiController]                    // Enables automatic model validation
[Route("api/[controller]")]       // URL: /api/weather
public class WeatherController : ControllerBase
{
    private readonly IWeatherService _service;

    // Dependencies are injected via constructor
    public WeatherController(IWeatherService service)
    {
        _service = service;
    }

    [HttpGet]                      // GET /api/weather
    public IActionResult GetAll()
    {
        return Ok(_service.GetForecasts());
    }

    [HttpGet("{id:int}")]          // GET /api/weather/5
    public IActionResult GetById(int id)
    {
        var forecast = _service.GetById(id);
        return forecast is null ? NotFound() : Ok(forecast);
    }
}
```

**When to use Controllers:**
*   Large, complex APIs with many endpoints.
*   When you want Action Filters (cross-cutting logic per endpoint).
*   When your team comes from an MVC or older ASP.NET background.

### Minimal APIs (The Modern, Lightweight Approach)
Introduced in .NET 6. You define endpoints directly in `Program.cs` using lambda expressions.

```csharp
app.MapGet("/api/weather", (IWeatherService service) => 
    service.GetForecasts());

app.MapGet("/api/weather/{id:int}", (int id, IWeatherService service) => 
    service.GetById(id) is { } forecast ? Results.Ok(forecast) : Results.NotFound());
```

**When to use Minimal APIs:**
*   Microservices with a handful of endpoints.
*   Serverless functions or simple CRUD APIs.
*   When you want maximum performance with minimal ceremony.

> **Key Insight:** Under the hood, both Controllers and Minimal APIs compile down to the **exact same Endpoint Routing pipeline**. The performance difference is negligible. Choose based on team preference and project size.

## 6. Best Practices from Production Projects

After 25 years of building .NET applications, here are the patterns I've seen work best — and both **NextEvent** and **Normora** demonstrate them beautifully:

### ✅ Keep `Program.cs` Clean with Extension Methods
Real applications register dozens (sometimes hundreds) of services. If you dump them all into `Program.cs`, it becomes unreadable within weeks. Both projects solve this with **Extension Methods** that group registrations by architectural layer:

```csharp
// From NextEvent's Program.cs — Clean, scannable, organized
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiServices(builder.Configuration);      // CORS, Controllers, JSON
builder.Services.AddDatabaseServices(builder.Configuration); // EF Core, SQL Server
builder.Services.AddApplicationServices();                   // MediatR, FluentValidation
builder.Services.AddSwaggerServices();                       // OpenAPI documentation
builder.Services.AddIdentityServices(builder.Configuration); // JWT Auth, Identity
builder.Services.AddMassTransitServices(builder.Configuration); // Message bus
builder.Services.AddRedisServices(builder.Configuration);    // Distributed caching
builder.Services.AddRateLimiterServices();                   // API rate limiting
```

Each extension method is a separate C# file (e.g., `ApiServiceExtensions.cs`). You can read, test, and maintain each concern independently.

### ✅ Hide Server Identity Headers
By default, Kestrel broadcasts `Server: Kestrel` in every HTTP response header. This tells attackers exactly what technology you're running. Normora disables this:

```csharp
// From Normora's Program.cs
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false; // Don't tell the world we're running Kestrel
});
```

### ✅ Environment-Specific Behavior
Never run development tools in production. NextEvent gates Swagger and auto-migrations:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();    // Only expose API docs locally
    app.UseSwaggerUI();
    // Auto-apply database migrations (NEVER do this in production!)
    identityContext.Database.Migrate();
}
else
{
    app.UseHsts();              // Enforce HTTPS via browser headers
    app.UseHttpsRedirection();  // Redirect HTTP → HTTPS
}
```

### ✅ Fail Fast on Startup
If your database connection is broken, don't let the app start and silently fail on every request. Crash immediately with a clear error:

```csharp
// From NextEvent's Program.cs
var connectionString = configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException(
        "Connection string 'DefaultConnection' was not found.");
```

---
**Key Takeaways:**
1. ASP.NET Core is a complete rewrite — fast, cross-platform, and modern.
2. Kestrel is your built-in web server; always put a Reverse Proxy in front in production.
3. `Program.cs` has three phases: Build services → Configure pipeline → Run.
4. Keep `Program.cs` clean by extracting service registration into extension methods.

**Next up:** In Module 2, we dive into the single most important concept in ASP.NET Core — the **Middleware Pipeline**. Understanding middleware is the key to understanding *everything else*.
