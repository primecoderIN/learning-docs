# ASP.NET Core Fundamentals & Software Architecture Course

**Target Audience:** Complete Beginners to C# Web Development up to Aspiring and Senior Software Architects  
**Technology Stack:** .NET 8 / 9 / 10, ASP.NET Core, C# 12 / 13  
**Teaching Approach:** Every concept is explained with the **"What, How, Why, and Where"** framework — using clear beginner analogies, progressive code snippets, architectural trade-offs, security invariants, and real-world battle-tested production code from **NextEvent** and **Normora** projects.  
**Goal:** Take someone who knows nothing about ASP.NET Core and guide them all the way to understanding, designing, and building enterprise-grade, high-throughput, secure web architectures.

---

## 🏛️ Course Architecture & Learning Journey

1. **How Kestrel, Reverse Proxies, and Docker interact** to ensure security, correct IP forwarding, and zero-downtime container rollouts.
2. **How the Request Pipeline and Middleware order dictate system security**, including the exact nuances of rate-limit partitioning, anti-forgery, and multi-tenant isolation.
3. **How Dependency Injection lifetimes govern memory allocation, thread safety, and concurrency**, and how the Captive Dependency trap brings down production systems.
4. **How Endpoint Routing and Minimal APIs optimize throughput** and integrate with real-time WebSockets (SignalR) and modern AI agent endpoints.
5. **How Configuration, the Options Pattern, and Cloud Secrets** enforce fail-fast startup and runtime configuration safety.
6. **How Model Binding, FluentValidation, and MediatR Pipeline Behaviors** eliminate boilerplate and enforce clean CQRS request validation.
7. **How Modern Error Handling (RFC 7807 Problem Details, .NET 8+ `IExceptionHandler`), Structured Logging, and BOLA/BFLA security sanitization** prevent sensitive data leakage.
8. **How Modular Monoliths enforce bounded contexts, schema isolation, and cross-module communication** without succumbing to microservice operational overhead.
9. **How Enterprise Identity Architecture shifts from naive JWT in browser storage to the Backend-For-Frontend (BFF) pattern with Keycloak and HttpOnly cookies** to eliminate XSS token theft.
10. **How Architectural Fitness Functions (Architecture Tests) programmatically guarantee clean architecture rules and module boundaries** across continuous integration pipelines.

---

## 📚 Course Outline

### [Module 1: Introduction to ASP.NET Core & Cloud-Native Foundations](01_Module_1_Introduction.md)
*   **What is ASP.NET Core?** The evolution from .NET Framework to modern, cross-platform .NET.
*   **Why ASP.NET Core?** Performance benchmarks, cross-platform, unified programming model, built-in DI, cloud-ready design.
*   **The Hosting Model:** Kestrel internals (socket transport, thread pool, zero-allocation parsing), Reverse Proxies (Nginx/IIS), and Docker considerations.
*   **Server Hardening:** Hiding server banners (`options.AddServerHeader = false`) to defeat OSINT reconnaissance.
*   **Forwarded Headers & Reverse Proxy Security (Normora vs. Naive approach):** Why `KnownProxies.Clear()` creates IP spoofing vulnerabilities; production validation with `ReverseProxy:KnownProxies`, `IPAddress` parsing, `ForwardLimit = 1`, and fail-fast startup.
*   **Cloud-Native Migration Strategy (Normora CLI Pattern):** Why in-process migrations fail in multi-pod Kubernetes deployments; executing ephemeral migrations via `dotnet Normora.Api.dll --migrate`.
*   **Anatomy of a Project:** `Program.cs` (three phases with restaurant analogy), `appsettings.json`, `.csproj`, `launchSettings.json`.
*   **Minimal APIs vs. Controllers:** When, why, and where to use each approach; memory allocation differences and source generation.
*   **Best Practices:** Layered extension methods, environment-specific behavior, fail-fast startup.

### [Module 2: The Request Pipeline, Middleware & Security Invariants](02_Module_2_Middleware_and_Pipeline.md)
*   **What is Middleware?** The airport security analogy — every request flows through a pipeline.
*   **The Russian Doll Model:** `next()`, bidirectional flow, `app.Use()`, `app.Run()`, `app.Map()`.
*   **Why Middleware?** Separation of Concerns — controllers only do business logic.
*   **Order Matters!** The standard pipeline order with detailed explanations for each position.
*   **Short-Circuiting:** Performance optimization by stopping the pipeline early.
*   **The Critical Rate Limiter Placement Invariant:** Why `UseRateLimiter()` MUST run AFTER `UseAuthentication()` and `UseBff()`. Explaining anonymous partition collapse and platform-wide DoS vulnerabilities.
*   **Class-Based vs. Factory-Based Middleware:** RequestDelegate singleton lifetime vs. scoped method parameters.
*   **Custom Middleware in NextEvent:** Domain exception mapping with `ExceptionMiddleware`.
*   **Custom Middleware in Normora:** High-performance `TenantResolutionMiddleware` leveraging cached `ITenantResolutionService` to prevent per-request database exhaustion.
*   **Defense-in-Depth:** OWASP security headers (`X-Content-Type-Options: nosniff`) via inline pipeline middleware.

### [Module 3: Dependency Injection, Lifetimes & Architectural Invariants](03_Module_3_Dependency_Injection.md)
*   **What Problem Does DI Solve?** Tight coupling, the "chef who grows his own tomatoes" analogy.
*   **The Built-in DI Container:** `IServiceCollection` (registration phase) vs. `IServiceProvider` (read-only execution phase).
*   **Service Lifetimes:** Transient, Scoped, Singleton — visual diagrams, analogies, allocation footprints, and thread safety rules.
*   **Registration and Resolution:** Constructor injection, method injection, `[FromServices]`, factory registrations.
*   **The Captive Dependency Trap:** Why Singletons must never capture Scoped services (database context leaks, stale entities, multithread crashes, cross-tenant data leaks).
*   **Resolving Scoped Services in Singletons:** Proper manual scope management via `IServiceScopeFactory`.
*   **Real-World Modular DI:** Extension methods per architectural layer (`AddApplicationServices`, `AddDatabaseServices`, `AddIdentityServices`).
*   **Automated Assembly Scanning:** Auto-discovering MediatR handlers and FluentValidation validators.
*   **Mockable Abstractions:** Abstracting system clock with `IDateTimeProvider` and security principals with `ICurrentUser`.
*   **Architectural Fitness Functions:** Programmatic enforcement via `Normora.Architecture.Tests` ensuring controllers never inject `DbContext` directly.

### [Module 4: Routing, Endpoints, Minimal APIs & Real-Time SignalR](04_Module_4_Routing_and_Endpoints.md)
*   **What is Routing?** The Post Office analogy — mapping URLs to C# methods.
*   **Endpoint Routing:** The two-step process (matching vs. execution) and why the split empowers authorization and CORS.
*   **Controller Routing:** `[ApiController]` behaviors, `[Route]`, why static routes beat `[controller]` tokens.
*   **Route Parameters & Defensive Constraints:** `{id:int}`, `{slug:alpha}`, `{id:guid}`, optional and default values.
*   **Minimal APIs:** `MapGet`, `MapPost`, Route Groups, dependency injection in delegates, Native AOT compatibility.
*   **HTTP Return Types:** `Ok()`, `Created()`, `NotFound()`, `BadRequest()`, `NoContent()`, `Problem()`, `TypedResults`.
*   **Real-Time Endpoints with SignalR:** Hub routing (`DocumentHub`, `NotificationHub`), securing hubs as BFF endpoints, and scaling horizontally with Redis backplanes.
*   **AI Agent Endpoints:** CopilotKit & Microsoft Agent Framework AG-UI endpoint integration in Normora (`MapNormoraAguiServer()`).
*   **Modern API Documentation:** Migrating from Swagger to native OpenAPI and **Scalar** (`MapScalarApiReference()`).

### [Module 5: Configuration, the Options Pattern & Cloud Secrets](05_Module_5_Configuration_and_Options.md)
*   **What is Configuration?** The car dashboard analogy — externalizing runtime knobs and dials.
*   **The Hierarchical Model:** `appsettings.json` → Environment JSON → User Secrets → Env Vars → CLI args.
*   **Cloud & Docker Conventions:** Double underscore convention (`Section__Key`) for nested settings in Linux containers.
*   **Raw `IConfiguration` (and why it's bad):** Magic strings, manual parsing, no IntelliSense, runtime failure hazards.
*   **The Options Pattern:** Strongly-typed binding with POCO classes.
*   **The Options Trinity:** `IOptions<T>` (Singleton) vs. `IOptionsSnapshot<T>` (Scoped) vs. `IOptionsMonitor<T>` (Singleton with live change notifications).
*   **Options Validation on Startup:** Fail-fast boot checks with `.ValidateDataAnnotations().ValidateOnStart()`.
*   **Production Implementations:** `RedisOptions`, `GeminiOptions`, `MinioOptions`, and `KeycloakOptions` in Normora.
*   **Architecture Testing for Options:** Enforcing consistent default options values in CI/CD.

### [Module 6: Model Binding, Validation & CQRS Pipeline](06_Module_6_Model_Binding.md)
*   **What is Model Binding?** The mail room analogy — automatic mapping from HTTP bytes to C# objects.
*   **Binding Sources:** `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`, `[FromServices]`.
*   **`[ApiController]` Inference Rules:** How complex types default to `[FromBody]`, simple types to `[FromRoute]`/`[FromQuery]`.
*   **Validation Architecture:** Data Annotations vs. FluentValidation — why complex business rules belong in dedicated validators.
*   **MediatR Pipeline Behaviors:** Open generic `ValidationBehavior<TRequest, TResponse>` executing validation automatically before command handlers.
*   **Audit Logging in Pipeline:** Tracking mutations, actor IDs, timestamps, and request telemetry with `AuditBehavior<TRequest, TResponse>`.
*   **Assembly Scanning for Validators:** Auto-registering validators across modular monolith boundaries.

### [Module 7: Error Handling, Logging & Security Sanitization](07_Module_7_Error_Handling.md)
*   **What is Logging?** The hospital chart analogy — recording structured flight recorder events.
*   **Structured Logging with `ILogger<T>` and Serilog:** Message templates vs. string interpolation, log level hierarchy, sink configuration.
*   **Log Level Configuration:** Per-namespace filtering in `appsettings.json`.
*   **The Error Handling Evolution:** Custom `ExceptionMiddleware` (NextEvent) vs. modern .NET 8+ `IExceptionHandler` (Normora).
*   **Problem Details (RFC 7807):** Standardized error responses across services.
*   **Information Leakage Prevention (CWE-209 Mitigation):**
    *   Reflection-cached `SafeDomainMessages` allowlist using `ApiMessages` constants to mask internal database/framework errors while serving safe business messages in production.
    *   Sanitizing background job failure notifications and emails.
*   **BOLA (Broken Object Level Authorization / OWASP API 1):** Why unauthorized access must return `404 Not Found` instead of `403 Forbidden` to prevent resource ID enumeration.
*   **BFLA (Broken Function Level Authorization / OWASP API 5):** Returning `403 Forbidden` for unauthorized actions.
*   **Distributed Tracing & Observability:** Correlating requests across logs, metrics, and traces using OpenTelemetry.

### [Module 8: Modular Monolith Architecture & Database Isolation](08_Module_8_Modular_Monolith.md)
*   **The Architecture Spectrum:** Monolith ("Big Ball of Mud") vs. Microservices vs. Modular Monolith (the office building analogy).
*   **Project Structure & Bounded Contexts:** NextEvent (`Identity`, `Organizations`, `Events`) and Normora (`Tenants`, `Documents`, `Conversations`, `Worker`, `Infrastructure`).
*   **The Golden Rule — Database Isolation:** One `DbContext` per module, separate PostgreSQL schemas (`CREATE SCHEMA tenants;`), independent migrations, prohibiting cross-module SQL joins.
*   **Cross-Module Communication:** Shared interfaces and DTOs (sync), domain events via MediatR notifications (in-memory async), MassTransit with RabbitMQ and Transactional Outbox (distributed async).
*   **The Composition Root:** `Program.cs` wires all modules but contains zero business logic.
*   **Multi-Tenancy Architecture (Normora):** Tenant resolution pipeline, scoped `ITenantContext`, EF Core Global Query Filters (`e.HasQueryFilter(x => x.TenantId == _tenantContext.TenantId)`), and safe bypassing via `IgnoreQueryFilters()`.
*   **Decoupled Background Workers:** Isolating heavy workloads into `Normora.Worker` and Hangfire.
*   **Microservice Extraction:** When and how to extract a module into a separate service.

### [Module 9: Authentication, Authorization & Identity Architecture](09_Module_9_Authentication_and_Authorization.md)
*   **AuthN vs. AuthZ:** Identity verification vs. Permissions (The Nightclub analogy).
*   **Claims-Based Identity:** Claims, ClaimsIdentity, and ClaimsPrincipal explained.
*   **JWT Authentication (NextEvent):** Stateless token signing, issuer/audience validation, clock skew elimination.
*   **BFF and Cookie Authentication (Normora):**
    *   Why storing JWTs in browser `localStorage` is an XSS vulnerability.
    *   How Duende BFF encapsulates tokens on the server, issues encrypted `HttpOnly` `SameSite=Strict` cookies, and validates anti-forgery headers (`X-CSRF: 1`).
    *   Keycloak IAM and OpenID Connect authorization code flow with PKCE.
*   **Roles vs. Policies:** Why declarative Policies are the modern, flexible way to secure endpoints.
*   **Resource-Based & Dynamic Multi-Tenant Auth:** Dealing with dynamic permissions requiring database lookups or scoped contexts (`[RequireTenant(TenantRoles.Admin)]`).
*   **Abstraction (`ICurrentUser` / `ICurrentUserService`):** Decoupling business logic from `HttpContext`.

### [Module 10: Architectural Fitness Functions, Resiliency & Production Engineering](10_Module_10_Architecture_Tests_and_Resilience.md)
*   **Architectural Fitness Functions (Architecture Tests):** Writing automated tests in `Normora.Architecture.Tests` to prevent architectural degradation:
    *   Enforcing parameterless constructors on entity configurations for auto-discovery.
    *   Guaranteeing Controllers NEVER inject `DbContext` directly.
    *   Enforcing strict isolation: Modules must never reference another module's internal classes.
    *   Ensuring Worker hosts never reference the API host, and Infrastructure never references executable hosts.
    *   Guaranteed error sanitization in email templates (CWE-209 mitigation).
*   **Distributed Caching & Failover:** Redis distributed cache with in-memory fallback for local development and integration tests, with fail-fast enforcement in production.
*   **System Health Checks:** Liveness vs. Readiness probes (`/healthz`) for Kubernetes orchestrators.
*   **Zero-Downtime Database Migration Deployments:** Orchestrating blue/green and rolling deployments with ephemeral migration runners and the Expand/Contract pattern.
