# Module 7: Error Handling, Logging & Security Sanitization

Every distributed system encounters failures: database failovers occur, third-party AI services trigger rate limits, network partitions disconnect microservices, and invalid inputs penetrate validation guards.

In production architecture, error handling and logging serve two critical mandates:
1.  **Observability & Diagnostics:** Engineers must be able to trace exactly what failed, where it failed, and why it failed within seconds.
2.  **Security & Information Shielding (CWE-209 Mitigation):** The system must **never leak internal implementation details, stack traces, database schemas, or infrastructure topology** to untrusted clients.

This module details the modern evolution from custom exception middleware to **.NET 8+ `IExceptionHandler`**, RFC 7807 Problem Details, **BOLA/BFLA security protections**, and reflection-cached safe error propagation.

---

## 1. What is Logging?

Logging is the practice of recording **structured events** while your application runs. Think of it as your application's black box flight recorder — when something goes wrong, you read the logs to understand exactly what happened and why.

> **Analogy — The Hospital Chart:**  
> A patient walks into a hospital. Every action is recorded on their chart: vitals taken, tests ordered, medications given, symptoms observed. If the patient's condition worsens, the doctor reads the chart to trace what happened. Without it, the doctor is guessing. Logs are your application's medical chart.

---

## 2. Logging with `ILogger<T>` — The Built-in System

ASP.NET Core provides a built-in logging abstraction via `ILogger<T>`. The generic `<T>` parameter tags every log message with the class name, making it easy to filter logs by component.

### Basic Usage
```csharp
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;
    private readonly IOrderRepository _repository;

    public OrdersController(ILogger<OrdersController> logger, IOrderRepository repository)
    {
        _logger = logger;
        _repository = repository;
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        _logger.LogInformation("Fetching order {OrderId}", id);  // Structured!
        
        var order = await _repository.GetByIdAsync(id);
        
        if (order is null)
        {
            _logger.LogWarning("Order {OrderId} not found", id);
            return NotFound();
        }

        return Ok(order);
    }
}
```

### Structured Logging — Why `{OrderId}` and NOT String Interpolation
```csharp
// ❌ BAD: String interpolation creates flat text
_logger.LogInformation($"Fetching order {id}");
// Output: "Fetching order 42"  ← Just a string. Can't query by OrderId.

// ✅ GOOD: Message template with named placeholders
_logger.LogInformation("Fetching order {OrderId}", id);
// Output (JSON): { "OrderId": 42, "Message": "Fetching order {OrderId}" }
```

**Why does this matter?** With structured logging, you can query your log platform: *"Show me all logs where `OrderId == 42`."* With string interpolation, you can only do a full-text search for `"42"` — which matches order 42, user ID 42, port 42, and every other 42.

### Log Levels — When to Use Each

| Level | Severity | When to use | Example |
|---|---|---|---|
| `Trace` | 0 | Extremely detailed debugging info. Almost never on in production. | "Entering method GetById with id=42" |
| `Debug` | 1 | Useful during development. Turned off in production. | "Cache miss for key 'user:42'" |
| `Information` | 2 | **Normal business events.** Use this for important milestones. | "Order 42 created by user alice@test.com" |
| `Warning` | 3 | **Something unexpected happened, but the app recovered.** | "Retry attempt 2 for database connection" |
| `Error` | 4 | **An operation failed.** The request couldn't be completed. | "Failed to charge credit card for order 42" |
| `Critical` | 5 | **Application is dying.** Entire system is compromised. | "Database server is unreachable. App shutting down." |

### Log Level Configuration
You control which levels are recorded per category in `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",                    // Everything: Info and above
      "Microsoft.AspNetCore": "Warning",           // Framework noise: Warning only
      "Microsoft.EntityFrameworkCore": "Warning",  // EF Core: Warning only
      "MyApp.Services": "Debug"                    // Your code: Debug and above
    }
  }
}
```

This means in production, you don't see hundreds of framework-level `Information` logs — only warnings and errors from ASP.NET Core, but full `Information` from your own code.

---

## 3. Logging Providers — Where Logs Go

The built-in `ILogger<T>` is just the **interface**. Where logs actually go depends on which **providers** you configure.

### Built-in Providers
*   **Console** — Prints to the terminal. Standard for Docker containers (collected by container log forwarders).
*   **Debug** — Prints to the IDE's debug output window.
*   **EventLog** — Windows Event Log (Windows only).

### Third-Party Providers (Production: Serilog)
In production, enterprise systems almost always use **Serilog** configured with sinks for **Seq, Elasticsearch, Datadog, or Azure Application Insights**:

```csharp
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console(new JsonFormatter()));
```

---

## 4. The Architectural Evolution: `ExceptionMiddleware` to `IExceptionHandler`

ASP.NET Core has evolved its error-handling mechanisms across major versions:

```
ASP.NET Core 2.0 - 7.0                     Modern .NET 8 / 9 / 10
┌─────────────────────────────┐            ┌─────────────────────────────┐
│ Custom ExceptionMiddleware  │            │ IExceptionHandler Interface │
│ - Class-based try-catch     │   ──────▶  │ - Centralized, DI-friendly  │
│ - Manual Response.Write     │            │ - ValueTask<bool> TryHandle │
│ - Boilerplate reflection    │            │ - Native ProblemDetails     │
└─────────────────────────────┘            └─────────────────────────────┘
```

### Approach 1: Custom Exception Middleware (NextEvent)
In **NextEvent**, error handling is wrapped in a class-based middleware:

```csharp
// From NextEvent's API/Middleware/ExceptionMiddleware.cs
public class ExceptionMiddleware(
    RequestDelegate next,
    ILogger<ExceptionMiddleware> logger,
    IHostEnvironment env)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Errors));
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);
            var message = env.IsDevelopment() ? ex.Message : "An unexpected server error occurred.";
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(message));
        }
    }
}
```

---

### Approach 2: Modern .NET 8+ `IExceptionHandler` (Normora)

In .NET 8+, Microsoft introduced the `IExceptionHandler` interface. Instead of manual try-catch wrappers, you implement `IExceptionHandler` and register it with `app.UseExceptionHandler()`.

Here is the production implementation of `GlobalExceptionHandler` in **Normora**:

```csharp
// From Normora.Api/Middleware/GlobalExceptionHandler.cs
public sealed class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger, 
    IWebHostEnvironment env) 
    : IExceptionHandler
{
    // Reflection-cached allowlist of safe, deliberate domain error messages
    private static readonly HashSet<string> SafeDomainMessages = typeof(ApiMessages)
        .GetFields(BindingFlags.Public | BindingFlags.Static)
        .Where(f => f.IsLiteral && !f.IsInitOnly && f.FieldType == typeof(string))
        .Select(f => (string)f.GetValue(null)!)
        .ToHashSet(StringComparer.Ordinal);

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // 1. Validation Failures (FluentValidation from MediatR pipeline)
        if (exception is ValidationException validationException)
        {
            var errors = validationException.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

            httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
            await httpContext.Response.WriteAsJsonAsync(
                ApiResponse<Dictionary<string, string[]>>.Failure(ApiMessages.ValidationFailed, errors), 
                cancellationToken);
            return true;
        }

        // 2. Business Invariant Violations (InvalidOperationException)
        if (exception is InvalidOperationException)
        {
            httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
            // Production Information Shielding: Verify if message is a deliberate constant
            var message = env.IsProduction() && !SafeDomainMessages.Contains(exception.Message)
                ? ApiMessages.BadRequest
                : exception.Message;
            await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(message), cancellationToken);
            return true;
        }

        // 3. OWASP API 1: BOLA (Broken Object Level Authorization)
        if (exception is BolaException || exception is NotFoundException)
        {
            httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
            await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(ApiMessages.NotFound), cancellationToken);
            return true;
        }

        // 4. OWASP API 5: BFLA (Broken Function Level Authorization)
        if (exception is BflaException)
        {
            httpContext.Response.StatusCode = StatusCodes.Status403Forbidden;
            await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(ApiMessages.Forbidden), cancellationToken);
            return true;
        }

        // 5. Upstream AI / External Dependency Fault Isolation (e.g. Gemini 429)
        if (exception is HttpRequestException httpException && 
            httpException.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
        {
            logger.LogWarning("Upstream AI dependency rate limit encountered (429).");
            httpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(ApiMessages.RateLimitExceeded), cancellationToken);
            return true;
        }

        // 6. Generic Fallback: Unhandled Server Errors (500)
        logger.LogError(exception, "An unhandled exception occurred in the API pipeline.");
        
        var serverErrorMessage = env.IsProduction() ? ApiMessages.InternalServerError : exception.Message;
        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(serverErrorMessage), cancellationToken);
        return true;
    }
}
```

---

## 5. Security Deep Dive: Information Shielding & CWE-209 Mitigation

### The Vulnerability: Information Exposure Through Error Messages (CWE-209)
In insecure APIs, when an unexpected database error occurs, the server transmits the raw exception to the client:
```json
{
  "error": "Npgsql.PostgresException: 42P01: relation 'tenants.billing_secrets' does not exist at line 4 column 12..."
}
```
This informs attackers of the internal database technology, internal schemas, table names, and query structures.

### 🛡️ The Architectural Solution: Reflection-Cached `SafeDomainMessages`
In **Normora**, the architect implements a **Reflection-Cached Allowlist**:
1.  All intentional business error strings are centralized as constants in a single class: `ApiMessages.cs`.
2.  At boot time, `GlobalExceptionHandler` scans `ApiMessages` via reflection and caches all string constants into a static `HashSet<string>`.
3.  In Production, when an exception occurs:
    *   If `exception.Message` exists in `SafeDomainMessages`, it is safe to display to the user.
    *   If `exception.Message` does NOT exist in the set (e.g., an unhandled EF Core or Npgsql exception), it is automatically masked and replaced with `ApiMessages.BadRequest` or `ApiMessages.InternalServerError`!

---

## 6. OWASP API Security: BOLA & BFLA Mitigation

The OWASP API Security Top 10 lists **Broken Object Level Authorization (BOLA)** as the #1 most exploited API vulnerability globally.

### 1. BOLA (OWASP API 1): Why BOLA Must Return 404, NOT 403!
Imagine an attacker accesses `/api/documents/8888`. Document `8888` belongs to Tenant B; the attacker belongs to Tenant A.

*   **If you return `403 Forbidden`:** The attacker learns that Document `8888` **actually exists in the database**, but they lack permission to view it. The attacker can now write a script iterating from 1 to 10,000, enumerating which document IDs exist across all competitors!
*   **If you return `404 Not Found` (The Normora Standard):** The API reveals **zero information**. The attacker cannot distinguish between a document that belongs to another tenant and a document that does not exist. The enumeration attack is completely neutralized.

In Normora's `GlobalExceptionHandler`:
```csharp
// Maps BOLA attempts to generic 404 Not Found to prevent resource enumeration
if (exception is BolaException || exception is NotFoundException)
{
    httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
    await httpContext.Response.WriteAsJsonAsync(ApiResponse.Failure(ApiMessages.NotFound), cancellationToken);
    return true;
}
```

### 2. BFLA (OWASP API 5): Broken Function Level Authorization
Occurs when a normal user attempts to execute an administrative action (e.g., `/api/tenants/billing/upgrade`).
*   Mapped strictly to **`403 Forbidden`** via `BflaException`.

---

## 7. Problem Details (RFC 7807) — The Industry Standard

REST APIs require a **standardized error format** so that clients can parse errors consistently regardless of which endpoint threw them.

**RFC 7807 (Problem Details for HTTP APIs)** defines this standard:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "detail": "The Workspace Slug field is required.",
  "instance": "/api/tenants",
  "traceId": "00-84c1fd4063c38d9d7fa57c42-00"
}
```

Enable native Problem Details in `Program.cs`:
```csharp
builder.Services.AddProblemDetails();
```

---

## 8. Startup Error Handling — Fail Fast

If the database is unreachable during boot, should the app start and fail on every request? **No. Crash immediately.**

```csharp
// From NextEvent's Program.cs
using var scope = app.Services.CreateScope();
var services = scope.ServiceProvider;

try
{
    var identityContext = services.GetRequiredService<IdentityDbContext>();
    identityContext.Database.Migrate();
    await IdentityDataSeeder.SeedAsync(roleManager, userManager);
}
catch (Exception ex)
{
    var logger = services.GetRequiredService<ILogger<Program>>();
    logger.LogCritical(ex, 
        "A critical error occurred during database initialization. " +
        "The application will terminate.");
    throw;  // RE-THROW — kill the process immediately
}
```

**Why `LogCritical` + `throw`?**
*   `LogCritical` ensures the error is flushed to container logs before the process dies.
*   `throw` ensures the process exits with a non-zero exit code, telling Docker/Kubernetes the container is unhealthy.

---

## 9. Distributed Tracing & OpenTelemetry

In modern architectures, a single user request spans multiple systems (API, MinIO, PostgreSQL, Hangfire, Gemini AI). How do you trace the entire flow?

### The Solution: OpenTelemetry
OpenTelemetry assigns a unique **TraceId** to each incoming request, which is propagated across all services and background jobs.

```csharp
// From Normora's Program.cs
builder.Services.AddNormoraTelemetry(builder.Environment);
```

When an error occurs, the `traceId` is included in the RFC 7807 Problem Details response. When a user reports an issue, you can locate the exact end-to-end trace in your telemetry dashboard.

---

## 10. Best Practices Summary

| Practice | Why |
|---|---|
| **Use structured logging (`{OrderId}`, not `$"{id}"`)** | Queryable, indexable, machine-readable |
| **Configure log levels per namespace** | Silence framework noise while capturing business logs |
| **Use Serilog for production** | Multiple sinks, structured JSON, enrichers |
| **Map domain exceptions to HTTP status codes** | Clean, consistent API responses |
| **Never expose stack traces in production** | Use `env.IsDevelopment()` or reflection-cached constant allowlists |
| **Return 404 for authorization failures (BOLA)** | Prevent ID enumeration attacks (OWASP API 1) |
| **Fail fast on startup** | `LogCritical` + `throw` for unrecoverable errors |
| **Use `IExceptionHandler` (.NET 8+)** | DI-friendly, testable, modern centralized error handling |
| **Enable Problem Details (RFC 7807)** | Standardized error format for API consumers |
| **Integrate OpenTelemetry** | Trace requests across distributed boundaries with a single TraceId |

---
**Key Takeaways:**
1. Logging is structured data, not flat text. Use message templates, not string interpolation.
2. Global exception handling maps domain exceptions to HTTP status codes — controllers never need try-catch blocks.
3. Never leak stack traces to production clients. Use reflection-cached constant allowlists (`SafeDomainMessages`).
4. Fail fast on startup errors. A broken app that starts is worse than an app that crashes with a clear message.
5. Prevent BOLA enumeration attacks by returning 404 Not Found for unauthorized resource access.

**Next up:** In [Module 8](08_Module_8_Modular_Monolith.md), we explore **Modular Monolith Architecture** — how to structure a large ASP.NET Core application so it doesn't degrade into an unmaintainable "Big Ball of Mud."
