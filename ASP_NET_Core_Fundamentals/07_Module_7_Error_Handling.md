# Module 7: Error Handling and Logging

Every application fails. Databases go down. External APIs timeout. Users send garbage data. A null reference sneaks past your tests. The question isn't *if* your app will encounter errors — it's *how gracefully it handles them*. This module covers the two pillars of production readiness: **Logging** (recording what happened) and **Error Handling** (responding to failures safely).

## 1. What is Logging?

Logging is the practice of recording **structured events** while your application runs. Think of it as your application's black box flight recorder — when something goes wrong, you read the logs to understand exactly what happened and why.

> **Analogy — The Hospital Chart:**
> A patient walks into a hospital. Every action is recorded on their chart: vitals taken, tests ordered, medications given, symptoms observed. If the patient's condition worsens, the doctor reads the chart to trace what happened. Without it, the doctor is guessing. Logs are your application's medical chart.

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
// Output: "Fetching order 42"  ← Looks the same, but...
// Structured data: { "OrderId": 42, "Message": "Fetching order {OrderId}" }
```

**Why does this matter?** With structured logging, you can query your log platform: "Show me all logs where `OrderId == 42`." With string interpolation, you can only do a full-text search for "42" — which matches order 42, user ID 42, port 42, and every other 42.

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

## 3. Logging Providers — Where Logs Go

The built-in `ILogger<T>` is just the **interface**. Where logs actually go depends on which **providers** you configure.

### Built-in Providers
*   **Console** — Prints to the terminal. Good for Docker (logs are collected by container orchestrators).
*   **Debug** — Prints to the IDE's debug output window.
*   **EventLog** — Windows Event Log (Windows only).

### Third-Party Providers (Production)
For production, you almost always use a structured logging library like **Serilog**:

```csharp
// In Program.cs, replace the built-in logging with Serilog
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)  // Read settings from appsettings.json
        .WriteTo.Console()                                // Still write to console
        .WriteTo.File("logs/app-.log",                   // Write to rolling log files
            rollingInterval: RollingInterval.Day)
        .WriteTo.Seq("http://localhost:5341");            // Send to Seq log server
});
```

**Why Serilog?**
*   **Structured JSON output** — Logs are queryable JSON, not flat text.
*   **Multiple sinks** — Write to Console, File, Seq, Elasticsearch, Datadog, Azure Application Insights simultaneously.
*   **Enrichers** — Automatically attach machine name, thread ID, request ID to every log entry.

## 4. Global Exception Handling

When an unhandled exception occurs (e.g., `NullReferenceException`, `SqlException`), ASP.NET Core's default behavior is to return a `500 Internal Server Error` with an ugly HTML stack trace. In production, this is **unacceptable** because:

1. **Security risk** — Stack traces expose internal code paths, database names, and framework versions.
2. **Bad UX** — API consumers expect structured JSON errors, not HTML.
3. **No logging** — The error might not be recorded if you don't catch it.

### Approach 1: Developer Exception Page (Development Only)
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();  // Rich HTML with stack trace, source code snippets
}
```

**This page shows your actual source code and line numbers.** Never, ever enable this in production.

### Approach 2: Custom Exception Middleware (The NextEvent Approach)
NextEvent uses a hand-crafted middleware class that maps **domain exceptions** to **HTTP status codes**:

```csharp
// From NextEvent's ExceptionMiddleware.cs (full production code)
public class ExceptionMiddleware(
    RequestDelegate next,
    ILogger<ExceptionMiddleware> logger,
    IHostEnvironment env)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);  // Let the entire pipeline run
        }
        catch (ValidationException ex)
        {
            // FluentValidation errors → 400 Bad Request
            var errors = ex.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail("Validation failed", errors));
        }
        catch (NotFoundException ex)
        {
            // Resource not found → 404 Not Found
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (BusinessRuleException ex)
        {
            // Domain logic violation → 409 Conflict
            context.Response.StatusCode = StatusCodes.Status409Conflict;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (UnauthorizedException ex)
        {
            // Not authenticated → 401 Unauthorized
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (ForbiddenAccessException ex)
        {
            // Not authorized → 403 Forbidden
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            // Everything else → 500 Internal Server Error
            logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);

            // CRITICAL: Never expose stack traces in production
            var message = env.IsDevelopment()
                ? ex.Message
                : "An unexpected error occurred. Please try again later.";

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(message));
        }
    }
}
```

**The mapping pattern:**

| Domain Exception | HTTP Status | Meaning |
|---|---|---|
| `ValidationException` | 400 Bad Request | Client sent invalid data |
| `UnauthorizedException` | 401 Unauthorized | Not logged in |
| `ForbiddenAccessException` | 403 Forbidden | Logged in, but not allowed |
| `NotFoundException` | 404 Not Found | Resource doesn't exist |
| `BusinessRuleException` | 409 Conflict | Business logic violation |
| Any other `Exception` | 500 Internal Server Error | Unexpected crash |

### Approach 3: `IExceptionHandler` Interface (.NET 8+ — The Normora Approach)
.NET 8 introduced a DI-friendly interface for exception handling:

```csharp
// From Normora's GlobalExceptionHandler.cs (full production code)
public sealed class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger, 
    IWebHostEnvironment env) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is ValidationException validationException)
        {
            var errors = validationException.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());

            httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;
            await httpContext.Response.WriteAsJsonAsync(
                ApiResponse<Dictionary<string, string[]>>.Failure("Validation failed", errors), 
                cancellationToken);
            return true;
        }

        // BOLA: Return 404 instead of 403 to prevent attackers from enumerating valid IDs
        if (exception is NotFoundException || exception is BolaException)
        {
            httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
            var message = env.IsProduction() ? ApiMessages.NotFound : exception.Message;
            await httpContext.Response.WriteAsJsonAsync(
                ApiResponse.Failure(message), cancellationToken);
            return true;
        }

        // BFLA: Broken Function Level Authorization
        if (exception is BflaException)
        {
            httpContext.Response.StatusCode = StatusCodes.Status403Forbidden;
            var message = env.IsProduction() ? ApiMessages.Forbidden : exception.Message;
            await httpContext.Response.WriteAsJsonAsync(
                ApiResponse.Failure(message), cancellationToken);
            return true;
        }

        // Fallback: 500
        logger.LogError(exception, "An unhandled exception occurred.");
        var serverErrorMsg = env.IsProduction() ? ApiMessages.InternalServerError : exception.Message;
        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(
            ApiResponse.Failure(serverErrorMsg), cancellationToken);
        return true;
    }
}
```

**Registration:**
```csharp
// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
// ...
app.UseExceptionHandler();
```

### Security Insight: BOLA and BFLA
Notice that Normora's handler treats `BolaException` the same as `NotFoundException` (returns 404, not 403). This is an **OWASP security best practice:**

*   **BOLA (Broken Object Level Authorization):** If user A tries to access user B's resource, returning `403 Forbidden` tells the attacker "this resource exists, you just can't access it." Returning `404 Not Found` tells them nothing — preventing ID enumeration attacks.

## 5. Problem Details (RFC 7807) — The Industry Standard

REST APIs need a **standardized error format** so that clients can parse errors consistently regardless of which endpoint threw them.

**RFC 7807 (Problem Details for HTTP APIs)** defines this standard:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "detail": "The Username field is required.",
  "instance": "/api/users",
  "traceId": "00-84c1fd4063c38d9d7fa57c42-00"
}
```

ASP.NET Core natively supports Problem Details. Enable it:
```csharp
builder.Services.AddProblemDetails();
```

## 6. Startup Error Handling — Fail Fast

One of the most overlooked aspects of error handling is **what happens during startup**. If the database is unreachable, should the app start and fail on every request? **No. Crash immediately.**

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
    throw;  // RE-THROW — kill the process
}
```

**Why `LogCritical` + `throw`?**
*   `LogCritical` ensures the error is recorded before the process dies.
*   `throw` ensures the process exits with a non-zero exit code, which tells Docker/Kubernetes "this container is unhealthy — restart it."

## 7. Distributed Tracing — Following Requests Across Systems

In modern architectures (especially modular monoliths and microservices), a single user action might trigger work across multiple systems. How do you trace the entire flow?

### The Problem
1. User clicks "Upload Document"
2. API receives the request
3. API saves metadata to PostgreSQL
4. API uploads file to MinIO
5. API enqueues a background job in Hangfire
6. Background job calls Gemini AI for text extraction

If step 5 fails, how do you find the related logs from steps 1-4?

### The Solution: OpenTelemetry
OpenTelemetry assigns a unique **TraceId** to each incoming request. That TraceId is propagated through every operation, service call, and background job.

```csharp
// From Normora's Program.cs
builder.Services.AddNormoraTelemetry(builder.Environment);
```

This configures:
*   **Traces** — Visual timeline of every operation in a request.
*   **Metrics** — Request counts, response times, error rates.
*   **Exported to** — Jaeger, Zipkin, Azure Monitor, Datadog, etc.

The `ProblemDetails` error response automatically includes the `traceId`, so when a user reports an error, you can search for that exact trace across all your logs.

## 8. Best Practices Summary

| Practice | Why |
|---|---|
| **Use structured logging (`{OrderId}`, not `$"{id}"`)** | Queryable, parseable, machine-readable |
| **Configure log levels per namespace** | Silence noisy framework logs, keep your own |
| **Use Serilog for production** | Multiple sinks, structured JSON, enrichers |
| **Map domain exceptions to HTTP status codes** | Clean, consistent API responses |
| **Never expose stack traces in production** | `env.IsDevelopment() ? ex.Message : "Safe message"` |
| **Return 404 for authorization failures (BOLA)** | Prevent ID enumeration attacks |
| **Fail fast on startup** | `LogCritical` + `throw` for unrecoverable errors |
| **Use `IExceptionHandler` (.NET 8+)** | DI-friendly, testable, modern |
| **Enable Problem Details (RFC 7807)** | Standardized error format for API consumers |
| **Integrate OpenTelemetry** | Trace requests across systems with a single TraceId |

---
**Key Takeaways:**
1. Logging is structured data, not printf debugging. Use message templates, not string interpolation.
2. Global exception handling maps domain exceptions to HTTP status codes — controllers never need try-catch.
3. Never leak stack traces to production clients. Ever.
4. Fail fast on startup errors. A broken app that starts is worse than an app that crashes with a clear message.
5. OpenTelemetry gives you traceability across your entire system.

**Next up:** Module 8 — **Modular Monolith Architecture**. How do you structure a large ASP.NET Core application so it doesn't become an unmaintainable "Big Ball of Mud"?
