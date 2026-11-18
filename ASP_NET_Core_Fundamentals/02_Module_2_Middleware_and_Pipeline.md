# Module 2: The Request Pipeline, Middleware & Security Invariants

If Module 1 was about understanding the *structure* of an ASP.NET Core application, Module 2 is about understanding its *soul*. **Middleware is the single most important concept in ASP.NET Core.** Every feature — authentication, routing, logging, error handling, rate limiting, and CORS — is implemented as middleware. Master this, and everything else clicks.

For a software architect, ordering middleware is not an aesthetic preference — **it is a matter of system security, performance, and operational correctness.**

---

## 1. What is Middleware?

### The Concept
When a browser sends an HTTP request to your server (e.g., `GET /api/products`), that request doesn't magically teleport to your Controller. It has to travel through a **pipeline** of software components. Each component in the pipeline is called a **middleware**.

Each middleware can:
1.  **Inspect or modify** the incoming request.
2.  **Decide** whether to pass the request to the *next* middleware.
3.  **Inspect or modify** the outgoing response on the way back.

> **Analogy — The Airport Security Model:**  
> Imagine arriving at an airport for a flight.  
> 1. **Check-in Counter** (HTTPS Redirect) — "Do you have the right ticket format? Let me upgrade you to HTTPS."  
> 2. **Passport Control** (Authentication) — "Who are you? Show me your credentials/token."  
> 3. **Security Screening** (Authorization) — "Are you allowed to enter this gate?"  
> 4. **Boarding Gate** (Endpoint/Controller) — "Welcome aboard. Here's your seat (response)."  
>  
> If you fail at *any* checkpoint, you're sent back immediately — you never reach the plane. This is exactly how middleware works.

### The Technical Definition
Middleware is a **function** (called a Request Delegate) that receives an `HttpContext` and optionally calls the `next` delegate to pass control to the next middleware.

```csharp
// The simplest possible middleware — an inline lambda
app.Use(async (HttpContext context, Func<Task> next) =>
{
    // === INBOUND (Request traveling toward the endpoint) ===
    Console.WriteLine($"→ Request arriving: {context.Request.Method} {context.Request.Path}");

    await next(); // Pass to the next middleware

    // === OUTBOUND (Response traveling back to the client) ===
    Console.WriteLine($"← Response leaving: {context.Response.StatusCode}");
});
```

---

## 2. How It Works: The Russian Doll Model

The middleware pipeline is often called the **"Russian Doll" model** (or "Onion" model). Each middleware wraps around the next one. The request travels *inward* through each layer, hits the center (your endpoint), and the response travels *outward* through each layer in reverse.

```
Request →  [Exception Handler]
               →  [Forwarded Headers]
                      →  [CORS]
                             →  [Authentication]
                                    →  [BFF Session]
                                           →  [Rate Limiter]
                                                  →  [Tenant Resolution]
                                                         →  [Authorization]
                                                                →  [Your Controller / Endpoint]
                                                         ←  [Authorization]
                                                  ←  [Tenant Resolution]
                                           ←  [Rate Limiter]
                                    ←  [BFF Session]
                             ←  [Authentication]
                      ←  [CORS]
               ←  [Forwarded Headers]
           ←  [Exception Handler]  → Response
```

### The Three Types of Middleware Registration

ASP.NET Core provides three methods for adding middleware:

```csharp
// 1. app.Use() — Runs code, then passes to the next middleware
app.Use(async (context, next) =>
{
    // Do something before (Inbound)
    await next();  // ← MUST call this to continue the pipeline
    // Do something after (Outbound)
});

// 2. app.Map() — Branches the pipeline based on URL path
app.Map("/healthz", branch =>
{
    branch.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status200OK;
        await context.Response.WriteAsync("OK");
    });
});

// 3. app.Run() — Terminal middleware. NEVER calls next(). The pipeline STOPS here.
app.Run(async context =>
{
    context.Response.StatusCode = StatusCodes.Status404NotFound;
    await context.Response.WriteAsync("Resource not found");
});
```

> **Critical Rule:** If a middleware calls `next()`, the pipeline continues. If it does NOT call `next()`, the pipeline **short-circuits** — no further downstream middleware runs, and the response immediately starts traveling back outward.

---

## 3. Why Middleware?

### Without Middleware (The Nightmare)
Imagine you have 50 controller actions. Without middleware, you would need to manually add authentication checks, logging, error handling, tenant scoping, and CORS headers **inside every single action method:**

```csharp
// ❌ WITHOUT MIDDLEWARE: Every action repeats identical boilerplate
[HttpGet]
public IActionResult GetProducts()
{
    try 
    {
        if (!User.Identity.IsAuthenticated) return Unauthorized();  // Auth check
        if (!User.IsInRole("Admin")) return Forbid();                // Authz check
        _logger.LogInformation("Fetching products");                 // Logging
        Response.Headers.Add("Access-Control-Allow-Origin", "*");    // CORS

        var products = _service.GetAll();
        return Ok(products);
    }
    catch (Exception ex) 
    {
        _logger.LogError(ex, "Failed");                              // Error handling
        return StatusCode(500);
    }
}
```

### With Middleware (The Clean Way)
All those cross-cutting concerns are handled **once**, centrally, in the pipeline. Your controller becomes razor-thin:

```csharp
// ✅ WITH MIDDLEWARE: Controller only contains business logic
[HttpGet]
public IActionResult GetProducts()
{
    var products = _service.GetAll();
    return Ok(products);
}
```

The authentication, authorization, logging, CORS, rate limiting, and error handling all happen automatically in the middleware pipeline before and after your controller runs.

### The Principle: Separation of Concerns
*   **Middleware** handles infrastructure and cross-cutting concerns (security, protocol normalization, rate limiting, error handling).
*   **Controllers/Endpoints** handle business orchestration (fetching products, creating orders, delegating commands to CQRS handlers).

---

## 4. Middleware Order — Where It Goes Matters!

**This is where developers make the most dangerous mistakes.** The order you register middleware in `Program.cs` is the *exact* sequential order requests flow through. Get it wrong, and you introduce security vulnerabilities or subtle platform bugs.

### What Happens If You Get The Order Wrong?

**Scenario 1: Authorization BEFORE Authentication**
```csharp
app.UseAuthorization();    // ❌ Checks permissions
app.UseAuthentication();   // ❌ Reads identity
```
Result: Authorization runs *before* the user's identity is established. `HttpContext.User` is empty. Every request is treated as anonymous. **Your entire API is unprotected or returns 401/403 for everyone.**

---

### The Critical Rate Limiter Placement Invariant

Why is `app.UseRateLimiter()` placed strictly **after** `UseAuthentication()` and `UseBff()`, but **before** `TenantResolutionMiddleware`?

```csharp
// Typical user-partitioned rate limiter rule:
options.AddPolicy("user-policy", context =>
{
    // Extract the unique user ID claim ('sub')
    var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

    if (!string.IsNullOrEmpty(userId))
    {
        return RateLimitPartition.GetTokenBucketLimiter(userId, _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,
            ReplenishmentPeriod = TimeSpan.FromMinutes(1),
            TokensPerPeriod = 100
        });
    }

    // Fallback: Partition by IP for anonymous requests
    return RateLimitPartition.GetTokenBucketLimiter(
        context.Connection.RemoteIpAddress?.ToString() ?? "anonymous", 
        _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 10,
            ReplenishmentPeriod = TimeSpan.FromMinutes(1),
            TokensPerPeriod = 10
        });
});
```

#### ⚠️ The Anonymous Partition Collapse Hazard:
*   **If you register `app.UseRateLimiter()` BEFORE `app.UseAuthentication()`:**
    1.  The request enters `UseRateLimiter`.
    2.  The rate limiter checks `context.User.FindFirst(ClaimTypes.NameIdentifier)`.
    3.  Because `UseAuthentication()` has not run yet, `context.User` is completely empty!
    4.  The rate limiter assumes the user is **anonymous** and places them into the anonymous fallback bucket.
    5.  Every single authenticated user on your platform now shares the same tiny anonymous rate limit bucket!
    6.  As soon as one user performs 10 clicks, the rate limiter triggers HTTP 429 for **every user on the platform**, creating a catastrophic **self-inflicted Denial of Service (DoS)**!

**The Golden Architect Rule:** Always authenticate the caller before evaluating identity-partitioned rate limits.

---

## 5. The Production Pipeline Order: Normora Case Study

Here is the exact, hardened production pipeline from **Normora's `Program.cs`**:

```csharp
// =========================================================================
// NORMORA PRODUCTION REQUEST PIPELINE (Order is Sacred)
// =========================================================================

// 1. GLOBAL ERROR HANDLING & OWASP HEADERS (Outermost Layer)
// Must be first so it wraps EVERY downstream middleware failure.
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    await next();
});

// Centralized ProblemDetails error handler
app.UseExceptionHandler(); 

// 2. REVERSE PROXY & PROTOCOL NORMALIZATION
// Resolves original HTTPS scheme and client IP from trusted proxies
app.UseForwardedHeaders(forwardedHeadersOptions);

// 3. CROSS-ORIGIN RESOURCE SHARING (CORS)
// Evaluates CORS preflight OPTIONS requests before authentication
app.UseCors("CorsPolicy");

// 4. IDENTITY AUTHENTICATION
// Reads JWT Bearer tokens or Session Cookies; populates HttpContext.User
app.UseAuthentication();

// 5. BACKEND-FOR-FRONTEND (BFF) SESSION VALIDATION
// Validates Duende BFF anti-forgery headers and manages server-side sessions
app.UseBff();

// 6. RATE LIMITING (Must be placed AFTER AuthN & BFF!)
// Partitioners inspect `sub` claim. Placing it here prevents anonymous DoS collapse.
app.UseRateLimiter();

// 7. MULTI-TENANT WORKSPACE RESOLUTION
// Extracts X-Tenant-Id, queries cached ITenantResolutionService, populates ITenantContext
app.UseMiddleware<TenantResolutionMiddleware>();

// 8. ENDPOINT AUTHORIZATION
// Evaluates [Authorize] policies and [RequireTenant] against resolved claims and tenant
app.UseAuthorization();

// 9. ENDPOINT EXECUTION (The Center of the Russian Doll)
app.MapHealthChecks("/healthz").AllowAnonymous();
app.MapControllers().AsBffApiEndpoint();
app.MapBffManagementEndpoints();
app.MapHub<DocumentHub>("/hubs/documents").AsBffApiEndpoint();
app.MapHub<NotificationHub>("/hubs/notifications").AsBffApiEndpoint();
app.MapNormoraAguiServer().RequireAuthorization().AsBffApiEndpoint();

app.Run();
```

---

## 6. Short-Circuiting: The Performance Superpower

Some middleware intentionally does NOT call `next()`. This is called **short-circuiting**.

*   `UseStaticFiles` — If the URL matches a physical file (`/css/style.css`), it serves the file directly and stops. No routing, no authentication, no controller instantiation.
*   `UseRateLimiter` — If an IP or tenant exceeds their allocated quota, it returns `429 Too Many Requests` immediately without touching the database.
*   `UseAuthentication` — If an expired or forged token arrives, it can return `401 Unauthorized` immediately.
*   `UseExceptionHandler` — If an exception is thrown, it catches it and returns a safe ProblemDetails error response without propagating the crash to Kestrel.

> **Why this matters for performance:** If 40% of your requests are for static assets or health checks, short-circuiting saves you from running authentication, database queries, and controller allocations on those requests. That's a massive performance win.

---

## 7. Writing Custom Middleware

Built-in middleware covers standard scenarios, but real-world enterprise applications always require custom middleware.

### Method 1: Inline Middleware (Quick & Lightweight)
Ideal for simple, cross-cutting headers:

```csharp
// From Normora — Adding OWASP security header to every response
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    await next();
});
```

### Method 2: Class-Based Middleware (The Professional Way)
For anything beyond a single line, create a dedicated middleware class.

**The Rules:**
1. The class must have a constructor accepting `RequestDelegate next`.
2. The class must have a public method named `Invoke` or `InvokeAsync(HttpContext context, ...)`.
3. Constructor-injected dependencies are **Singletons** (held for the application's life).
4. Method-injected dependencies into `InvokeAsync` are **Scoped** (resolved per request).

#### Example: Request Timing Middleware
```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Constructor-injected dependencies live for the app's lifetime (Singleton).
    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // Method-injected dependencies are resolved per-request (Scoped).
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();

        await _next(context);  // Let the rest of the downstream pipeline run

        stopwatch.Stop();
        _logger.LogInformation(
            "HTTP {Method} {Path} responded {StatusCode} in {ElapsedMs}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            stopwatch.ElapsedMilliseconds);
    }
}

// Create an Extension Method (Convention):
public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestTimingMiddleware>();
    }
}
```

Register cleanly in `Program.cs`:
```csharp
app.UseRequestTiming();
```

---

## 8. Real-World Middleware from Your Projects

### 🏗️ NextEvent: Custom `ExceptionMiddleware`
NextEvent uses a custom `ExceptionMiddleware` that catches **domain-specific exceptions** and maps them to appropriate HTTP status codes:

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
            await next(context); // Let the entire pipeline run
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
        catch (BusinessRuleException ex)
        {
            context.Response.StatusCode = StatusCodes.Status409Conflict;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (UnauthorizedException ex)
        {
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (ForbiddenAccessException ex)
        {
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            // Generic fallback: 500 Internal Server Error
            logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);

            // CRITICAL: Never leak stack traces to production clients! (CWE-209)
            var message = env.IsDevelopment()
                ? ex.Message
                : "An unexpected error occurred. Please try again later.";

            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(message));
        }
    }
}
```

---

### 🏗️ Normora: High-Performance `TenantResolutionMiddleware`
In Normora's multi-tenant architecture, every request must be scoped to an active workspace.

#### The Performance Evolution:
In an early prototype, tenant resolution was queried directly against `TenantsDbContext`:
```csharp
// ❌ EARLY PATTERN: Direct DB query on every HTTP request
var membership = await tenantsDbContext.TenantMemberships.FirstOrDefaultAsync(...);
```
At 5,000 requests per second, this generated 5,000 duplicate SQL queries, rapidly exhausting PostgreSQL's connection pool.

#### The Hardened Production Solution:
Normora decoupled resolution into `ITenantResolutionService`, which caches tenant access and departmental clearances in Redis and L1 memory:

```csharp
// From Normora's server/Normora.Api/Middleware/TenantResolutionMiddleware.cs
public class TenantResolutionMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(
        HttpContext context,
        ITenantContext tenantContext,                          // Scoped to this request
        ICurrentUser currentUser,                              // Scoped identity abstraction
        ITenantResolutionService tenantResolutionService)      // Cached resolution engine
    {
        // 1. Tenant resolution only applies to authenticated security principals
        if (currentUser.IsAuthenticated)
        {
            // 2. Extract active workspace header supplied by client interceptor
            if (context.Request.Headers.TryGetValue("X-Tenant-Id", out var tenantIdValues))
            {
                if (Guid.TryParse(tenantIdValues.FirstOrDefault(), out var tenantId))
                {
                    // 3. Query the cached service to validate membership and compute effective departmental clearances.
                    var resolvedAccess = await tenantResolutionService.ResolveTenantAccessAsync(
                        tenantId,
                        currentUser.ExternalUserId,
                        context.RequestAborted);

                    if (resolvedAccess != null)
                    {
                        // 4. Bind resolved workspace scope to ITenantContext for downstream EF query filters
                        tenantContext.SetContext(tenantId, resolvedAccess.Role, resolvedAccess.EffectiveDepartments);
                    }
                }
            }

            // Unresolved or unauthorized tenant scopes remain unset; 
            // [RequireTenant] action filters will reject downstream access with 403 Forbidden.
        }

        await next(context);
    }
}
```

---

## 9. Middleware Dependency Lifetimes: Singleton vs. Scoped

Software architects must understand the dependency injection rules governing middleware:

### Conventional Class Middleware is a Singleton!
When you write:
```csharp
app.UseMiddleware<TenantResolutionMiddleware>();
```
ASP.NET Core instantiates **exactly one instance** of `TenantResolutionMiddleware` when the application boots and holds it alive forever.

*   **Constructor Injection:** Any service injected into the middleware's **constructor** lives for the lifetime of the application. **You must NEVER inject a Scoped service (like `DbContext` or `ITenantContext`) into the constructor of a conventional middleware!** Doing so creates a Captive Dependency ([Module 3](03_Module_3_Dependency_Injection.md)).
*   **Method Injection:** Injected services in the `InvokeAsync(HttpContext context, ITenantContext tenantContext, ...)` method are resolved **per HTTP request** from the request's scoped container. This is the safe, supported mechanism for consuming scoped dependencies inside middleware.

---

## 10. Best Practices Summary

| Practice | Why |
|---|---|
| **Put Exception Handler first** | It must catch errors from every other downstream middleware component |
| **Put Forwarded Headers early** | Normalizes real client IP and HTTPS scheme before CORS, auth, and rate limiting run |
| **Put Static Files before Routing** | Short-circuits static asset requests to avoid routing, auth, and controller work |
| **Authentication before Authorization** | You must know WHO the caller is before checking WHAT they can do |
| **Rate Limiter after Authentication** | Prevents anonymous partition collapse and platform-wide Denial of Service |
| **Tenant Resolution before Authorization** | Workspaces and clearances must populate before `[RequireTenant]` evaluates |
| **Use class-based middleware** | Testable, modular, follows Single Responsibility Principle |
| **Never leak details in Production** | Check `env.IsDevelopment()` or use constant allowlists before exposing error messages |
| **Add security headers via middleware** | OWASP headers (`X-Content-Type-Options: nosniff`) on every response |

---
**Key Takeaways:**
1. Middleware is the pipeline through which **every** HTTP request flows.
2. Order is **sacred** — wrong order = security vulnerabilities or platform-wide DoS.
3. Short-circuiting saves performance by stopping the pipeline early.
4. Use class-based middleware for production code. Method-inject scoped dependencies into `InvokeAsync`.

**Next up:** In [Module 3](03_Module_3_Dependency_Injection.md), we dive into **Dependency Injection** — the skeletal framework that creates, connects, and destroys services across the application.
