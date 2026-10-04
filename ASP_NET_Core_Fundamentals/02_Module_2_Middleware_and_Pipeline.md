# Module 2: The Request Pipeline and Middleware

If Module 1 was about understanding the *structure* of an ASP.NET Core application, Module 2 is about understanding its *soul*. **Middleware is the single most important concept in ASP.NET Core.** Every feature — authentication, routing, logging, error handling, CORS — is implemented as middleware. Master this, and everything else clicks.

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
> 2. **Passport Control** (Authentication) — "Who are you? Show me your JWT token."
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

## 2. How It Works: The Russian Doll Model

The middleware pipeline is often called the **"Russian Doll" model** (or "Onion" model). Each middleware wraps around the next one. The request travels *inward* through each layer, hits the center (your endpoint), and the response travels *outward* through each layer in reverse.

```
Request →  [Exception Handler]
               →  [HTTPS Redirect]
                      →  [Authentication]
                             →  [Authorization]
                                    →  [Your Controller / Endpoint]
                             ←  [Authorization]
                      ←  [Authentication]
               ←  [HTTPS Redirect]
           ←  [Exception Handler]  → Response
```

### The Three Types of Middleware Registration

ASP.NET Core provides three methods for adding middleware:

```csharp
// 1. app.Use() — Runs code, then passes to the next middleware
app.Use(async (context, next) =>
{
    // Do something before
    await next();  // ← MUST call this to continue the pipeline
    // Do something after
});

// 2. app.Map() — Branches the pipeline based on URL path
app.Map("/healthcheck", app =>
{
    app.Run(async context => await context.Response.WriteAsync("OK"));
});

// 3. app.Run() — Terminal middleware. NEVER calls next(). The pipeline STOPS here.
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello World");
});
```

> **Critical Rule:** If a middleware calls `next()`, the pipeline continues. If it does NOT call `next()`, the pipeline **short-circuits** — no further middleware runs, and the response immediately starts traveling back outward.

## 3. Why Middleware?

### Without Middleware (The Nightmare)
Imagine you have 50 controller actions. Without middleware, you would need to manually add authentication checks, logging, error handling, and CORS headers **inside every single action method:**

```csharp
// ❌ WITHOUT MIDDLEWARE: Every action repeats the same boilerplate
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

The authentication, authorization, logging, CORS, and error handling all happen automatically in the middleware pipeline before and after your controller runs.

### The Principle: Separation of Concerns
*   **Middleware** handles infrastructure concerns (security, logging, error handling).
*   **Controllers/Endpoints** handle business logic (fetching products, creating orders).

## 4. Middleware Order — Where It Goes Matters!

**This is where beginners make the most dangerous mistakes.** The order you register middleware in `Program.cs` is the *exact* order requests flow through. Get it wrong, and you create security vulnerabilities.

### The Standard Production Pipeline Order

```csharp
var app = builder.Build();

// ┌─────────────────────────────────────────────────────────────┐
// │  1. EXCEPTION HANDLER (outermost layer)                     │
// │  WHY first? So it catches errors from ALL downstream        │
// │  middleware, including authentication failures.              │
// └─────────────────────────────────────────────────────────────┘
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();    // Rich HTML error page (DEV ONLY!)
else
    app.UseExceptionHandler("/error");  // Safe JSON error response

// ┌─────────────────────────────────────────────────────────────┐
// │  2. SECURITY HEADERS                                        │
// │  WHY early? Every response — even error responses — must    │
// │  have security headers.                                     │
// └─────────────────────────────────────────────────────────────┘
app.UseHsts();                          // Tell browsers: "Always use HTTPS"
app.UseHttpsRedirection();              // Redirect HTTP → HTTPS

// ┌─────────────────────────────────────────────────────────────┐
// │  3. STATIC FILES (potential short-circuit)                  │
// │  WHY before routing? If the request is for style.css,       │
// │  serve it immediately. No need to run auth, routing, etc.   │
// └─────────────────────────────────────────────────────────────┘
app.UseStaticFiles();

// ┌─────────────────────────────────────────────────────────────┐
// │  4. ROUTING                                                 │
// │  WHAT: Examines the URL and determines WHICH endpoint       │
// │  matches. Does NOT execute it yet.                          │
// └─────────────────────────────────────────────────────────────┘
app.UseRouting();

// ┌─────────────────────────────────────────────────────────────┐
// │  5. CORS (Cross-Origin Resource Sharing)                    │
// │  WHY after routing? CORS policies can be endpoint-specific. │
// │  Routing must run first so CORS knows which endpoint to     │
// │  check policies for.                                        │
// └─────────────────────────────────────────────────────────────┘
app.UseCors("CorsPolicy");

// ┌─────────────────────────────────────────────────────────────┐
// │  6. AUTHENTICATION — "Who are you?"                         │
// │  WHAT: Reads the JWT token / cookie, validates it, and      │
// │  populates HttpContext.User with the claims.                │
// └─────────────────────────────────────────────────────────────┘
app.UseAuthentication();

// ┌─────────────────────────────────────────────────────────────┐
// │  7. AUTHORIZATION — "Are you allowed?"                      │
// │  WHAT: Checks if the authenticated user has the required    │
// │  roles/policies for the matched endpoint.                   │
// └─────────────────────────────────────────────────────────────┘
app.UseAuthorization();

// ┌─────────────────────────────────────────────────────────────┐
// │  8. ENDPOINT EXECUTION                                      │
// │  WHAT: Finally runs the matched controller action or        │
// │  Minimal API delegate.                                      │
// └─────────────────────────────────────────────────────────────┘
app.MapControllers();

app.Run();
```

### What Happens If You Get The Order Wrong?

**Scenario: Authorization BEFORE Authentication**
```csharp
app.UseAuthorization();    // ← Checks permissions
app.UseAuthentication();   // ← Reads the JWT token
```
Result: Authorization runs *before* the user's identity is established. `HttpContext.User` is empty. Every request is treated as anonymous. **Your entire API is unprotected.**

### Short-Circuiting (Stopping the Pipeline Early)
Some middleware intentionally does NOT call `next()`. This is called **short-circuiting**.

*   `UseStaticFiles` — If the URL matches a physical file (`/css/style.css`), it serves the file directly and stops. No routing, no authentication, no controller.
*   `UseAuthentication` — If the JWT token is invalid, it can return `401 Unauthorized` immediately without ever reaching your controller.
*   `UseExceptionHandler` — If an exception is thrown, it catches it and returns an error response without calling further middleware.

> **Why this matters for performance:** If 40% of your requests are for static files (CSS, JS, images), `UseStaticFiles` saves you from running authentication, routing, and authorization on those requests. That's a massive performance win.

## 5. Writing Custom Middleware

Built-in middleware covers most scenarios, but real-world applications always need custom middleware. You've already seen examples in NextEvent and Normora. Let's understand exactly how to build one.

### Method 1: Inline Middleware (Quick & Dirty)
Good for simple, one-line concerns:

```csharp
// From Normora — Adding a security header to every response
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    await next();
});
```

### Method 2: Class-Based Middleware (The Professional Way)
For anything more than a few lines, create a dedicated middleware class. This is what production code uses.

**The Rules:**
1. The class must have a **constructor** that accepts `RequestDelegate next`.
2. The class must have a public method named `Invoke` or `InvokeAsync`.
3. Additional dependencies (like `ILogger`) can be injected via the constructor (Singleton lifetime) or the `InvokeAsync` method parameters (Scoped lifetime).

**Example: Request Timing Middleware**
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

        await _next(context);  // Let the rest of the pipeline run

        stopwatch.Stop();
        _logger.LogInformation(
            "HTTP {Method} {Path} responded {StatusCode} in {ElapsedMs}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            stopwatch.ElapsedMilliseconds);
    }
}
```

**Create an Extension Method (Convention):**
```csharp
public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestTimingMiddleware>();
    }
}
```

**Register in Program.cs:**
```csharp
app.UseRequestTiming();  // Clean, readable, matches the app.UseXxx() convention
```

## 6. Real-World Middleware from Your Projects

Let's see how NextEvent and Normora use custom middleware to solve real problems.

### 🏗️ NextEvent: Exception Middleware
NextEvent uses a custom `ExceptionMiddleware` that catches **domain-specific exceptions** and maps them to appropriate HTTP status codes:

```csharp
// From NextEvent's ExceptionMiddleware.cs
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
        catch (ValidationException ex)     { /* → 400 Bad Request  */ }
        catch (NotFoundException ex)       { /* → 404 Not Found    */ }
        catch (BusinessRuleException ex)   { /* → 409 Conflict     */ }
        catch (UnauthorizedException ex)   { /* → 401 Unauthorized */ }
        catch (ForbiddenAccessException ex){ /* → 403 Forbidden    */ }
        catch (Exception ex)
        {
            // Generic fallback: 500 Internal Server Error
            logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);

            // CRITICAL: Never leak stack traces to production clients!
            var message = env.IsDevelopment()
                ? ex.Message
                : "An unexpected error occurred. Please try again later.";

            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(ApiResponse.Fail(message));
        }
    }
}
```

**Why this pattern is brilliant:**
*   Every exception type maps to a specific HTTP status code.
*   In Development, you see the real error message for debugging.
*   In Production, clients get a safe, generic message — no stack trace leaks.
*   Your controllers **never need try-catch blocks** — the middleware handles everything globally.

### 🏗️ Normora: Tenant Resolution Middleware
Normora is a **multi-tenant** application. Every API request must be scoped to a specific tenant (company). This is handled by middleware:

```csharp
// From Normora's TenantResolutionMiddleware.cs
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(
        HttpContext context,
        ITenantContext tenantContext,    // Scoped — injected via method params
        ICurrentUser currentUser,
        TenantsDbContext tenantsDbContext)
    {
        if (currentUser.IsAuthenticated)
        {
            // 1. Read the X-Tenant-Id header from the request
            if (context.Request.Headers.TryGetValue("X-Tenant-Id", out var tenantIdValues))
            {
                if (Guid.TryParse(tenantIdValues.FirstOrDefault(), out var tenantId))
                {
                    // 2. Verify the user actually belongs to this tenant
                    var membership = await tenantsDbContext.TenantMemberships
                        .Include(m => m.User)
                        .Include(m => m.MembershipDepartments)
                        .FirstOrDefaultAsync(m => 
                            m.TenantId == tenantId && 
                            m.User.KeycloakUserId == currentUser.KeycloakUserId);

                    if (membership != null)
                    {
                        // 3. Set the tenant context for this request
                        tenantContext.SetContext(tenantId, membership.Role.ToString(), ...);
                    }
                }
            }
            // If header is missing or invalid, context stays unresolved.
            // [RequireTenant] attribute on endpoints will then reject the request.
        }

        await _next(context); // Continue the pipeline
    }
}
```

**The pipeline order is critical here:**
```csharp
// From Normora's Program.cs
app.UseAuthentication();                            // Step 1: WHO is the user?
app.UseBff();                                       // Step 2: BFF session management
app.UseMiddleware<TenantResolutionMiddleware>();     // Step 3: WHICH tenant?
app.UseAuthorization();                             // Step 4: CAN they access this?
```

If Tenant Resolution ran *before* Authentication, the middleware couldn't check `currentUser.IsAuthenticated` because the user identity wouldn't be established yet.

## 7. Best Practices Summary

| Practice | Why |
|---|---|
| **Put Exception Handler first** | It must catch errors from every other middleware |
| **Put Static Files before Routing** | Short-circuit static requests to avoid unnecessary work |
| **Authentication before Authorization** | You must know WHO before checking WHAT they can do |
| **Custom middleware before Authorization** | Tenant resolution, custom claims must populate before authz checks |
| **Use class-based middleware** | Testable, organized, follows Single Responsibility |
| **Never leak details in Production** | Check `env.IsDevelopment()` before exposing error messages |
| **Add security headers via middleware** | OWASP headers (`X-Content-Type-Options`, `X-Frame-Options`) on every response |

---
**Key Takeaways:**
1. Middleware is the pipeline through which **every** HTTP request flows.
2. Order is **sacred** — wrong order = security vulnerabilities.
3. Short-circuiting saves performance by stopping the pipeline early.
4. Use class-based middleware for production code. Inline for quick one-liners.

**Next up:** Module 3 dives into **Dependency Injection** — the mechanism that makes middleware, controllers, and services all work together seamlessly.
