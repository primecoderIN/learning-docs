# ASP.NET Core Fundamentals Course

**Target Audience:** Complete Beginners to C# Web Development
**Technology Stack:** .NET 8/9/10, ASP.NET Core
**Teaching Approach:** Every concept is explained with "What, How, Why, and Where" — using analogies, code snippets, best practices, and real-world production code from **NextEvent** and **Normora** projects.
**Goal:** Take someone who knows nothing about ASP.NET Core and teach them everything needed to understand and build production-grade web applications.

## Course Outline

### Module 1: Introduction to ASP.NET Core
*   **What is ASP.NET Core?** The evolution from .NET Framework to modern, cross-platform .NET.
*   **Why ASP.NET Core?** Performance benchmarks, cross-platform, unified programming model.
*   **The Hosting Model:** Kestrel, Reverse Proxies (Nginx/IIS), and Docker considerations.
*   **Anatomy of a Project:** `Program.cs` (three phases), `appsettings.json`, `.csproj`, `launchSettings.json`.
*   **Minimal APIs vs. Controllers:** When and where to use each approach.
*   **Best Practices:** Extension methods, hiding server headers, environment-specific behavior, fail-fast startup.

### Module 2: The Request Pipeline and Middleware
*   **What is Middleware?** The airport security analogy — every request flows through a pipeline.
*   **The Russian Doll Model:** `next()`, inbound/outbound flow, `app.Use()`, `app.Run()`, `app.Map()`.
*   **Why Middleware?** Separation of Concerns — controllers only do business logic.
*   **Order Matters!** The standard pipeline order with detailed explanations for each position.
*   **Short-Circuiting:** Performance optimization by stopping the pipeline early.
*   **Custom Middleware:** Class-based middleware with extension methods.
*   **Real-World:** NextEvent's `ExceptionMiddleware`, Normora's `TenantResolutionMiddleware`, OWASP security headers.

### Module 3: Dependency Injection (DI)
*   **What Problem Does DI Solve?** Tight coupling, the "chef who grows his own tomatoes."
*   **Service Lifetimes:** Transient, Scoped, Singleton — with visual diagrams and analogies.
*   **Registration and Resolution:** Constructor injection, method injection, `[FromServices]`, `IServiceScopeFactory`.
*   **The Captive Dependency Trap:** Why Singletons must never capture Scoped services.
*   **Real-World:** Modular registration, manual scoping for startup, `IDateTimeProvider`, `TenantContext` as Scoped.

### Module 4: Routing and Endpoints
*   **Endpoint Routing:** The two-step process (matching vs. execution) and why the split matters.
*   **Controller Routing:** `[ApiController]` behaviors, `[Route]`, `[controller]` token.
*   **Route Parameters and Constraints:** `{id:int}`, `{slug:alpha}`, `{id:guid}`, optional and default values.
*   **Minimal APIs:** `MapGet`, `MapPost`, Route Groups, dependency injection in delegates.
*   **HTTP Return Types:** `Ok()`, `Created()`, `NotFound()`, `BadRequest()`, `NoContent()`, `Problem()`.
*   **Real-World:** Lowercase URLs, CamelCase JSON, SignalR hub routing, BFF anti-forgery.

### Module 5: Configuration and the Options Pattern
*   **The Hierarchical Model:** `appsettings.json` → Environment JSON → User Secrets → Env Vars → CLI args.
*   **Configuration Sources:** JSON files, User Secrets, Environment Variables (double underscore convention).
*   **Raw `IConfiguration` (and why it's bad):** Magic strings, manual parsing, no IntelliSense.
*   **The Options Pattern:** Strongly-typed binding with `IOptions<T>`.
*   **`IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`:** Singleton vs. Scoped vs. live-reloading.
*   **Options Validation on Startup:** `.ValidateDataAnnotations().ValidateOnStart()`.
*   **Real-World:** Fail-fast on missing config, securing JWT keys, CORS from config, database retry logic.

### Module 6: Model Binding and Validation
*   **What is Model Binding?** Automatic mapping from HTTP data to C# objects.
*   **Binding Sources:** `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`, `[FromServices]`.
*   **`[ApiController]` Inference Rules:** How complex types default to `[FromBody]`, simple types to `[FromRoute]`/`[FromQuery]`.
*   **Data Annotations:** `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, automatic 400 responses.
*   **FluentValidation:** Separate validator classes, conditional validation, database-dependent validation.
*   **MediatR + FluentValidation Pipeline:** `ValidationBehavior<,>` — automatic validation before every command handler.
*   **Real-World:** NextEvent's assembly scanning for validators, razor-thin controllers with CQRS.

### Module 7: Error Handling and Logging
*   **Structured Logging:** `ILogger<T>`, message templates vs. string interpolation, log levels.
*   **Logging Providers:** Console, Serilog, multiple sinks (File, Seq, Datadog).
*   **Log Level Configuration:** Per-namespace filtering in `appsettings.json`.
*   **Custom Exception Middleware (NextEvent):** Domain exceptions mapped to HTTP status codes.
*   **`IExceptionHandler` (.NET 8+ — Normora):** DI-friendly, BOLA/BFLA security patterns.
*   **Problem Details (RFC 7807):** Standardized error responses.
*   **Startup Error Handling:** `LogCritical` + `throw` — fail fast.
*   **Distributed Tracing (OpenTelemetry):** TraceId propagation across the entire system.

### Module 8: Modular Monolith Architecture
*   **The Architecture Spectrum:** Monolith vs. Microservices vs. Modular Monolith.
*   **Project Structure:** NextEvent and Normora directory layouts (Modules, Shared, API).
*   **The Golden Rule — Database Isolation:** One `DbContext` per module, independent migrations.
*   **Cross-Module Communication:** Shared interfaces (sync), domain events via MediatR (async), MassTransit (distributed).
*   **The Composition Root:** `Program.cs` wires all modules but contains zero business logic.
*   **Multi-Tenancy (Normora):** `TenantResolutionMiddleware`, `ITenantContext`, global query filters.
*   **Microservice Extraction:** When and how to extract a module into a separate service.

### Module 9: Authentication and Authorization
*   **AuthN vs AuthZ:** Identity verification vs Permissions (The Nightclub analogy).
*   **Claims-Based Identity:** Claims, ClaimsIdentity, and ClaimsPrincipal explained.
*   **JWT Authentication (NextEvent):** Token signing, issuer validation, audience checks.
*   **BFF and Cookie Authentication (Normora):** Mitigating XSS, external identity providers (Keycloak), secure HttpOnly cookies.
*   **Roles vs Policies:** Why Policies are the modern, flexible way to secure endpoints.
*   **Resource-Based & Multi-Tenant Auth:** Dealing with dynamic permissions requiring database lookups or scoped contexts.
*   **Abstraction (`ICurrentUserService`):** Decoupling business logic from `HttpContext`.
