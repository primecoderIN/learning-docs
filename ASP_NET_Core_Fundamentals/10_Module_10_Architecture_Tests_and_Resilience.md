# Module 10: Architectural Fitness Functions, Resiliency & Production Engineering

Architectures degrade over time. As deadlines loom, engineers make expedient compromises: a developer injects `DbContext` directly into a controller to bypass a query handler, a module references an internal entity of another bounded context, or an error handler leaks an internal SQL exception into an email notification.

In enterprise software engineering, senior architects do not rely on code reviews alone to protect the integrity of the system. **They write Architectural Fitness Functions — automated unit tests that fail CI/CD builds whenever architectural rules are violated.**

This module examines how **Normora** programmatically enforces clean architecture rules, handles distributed cache resilience, configures health probes for Kubernetes, and executes zero-downtime database migrations.

---

## 1. What are Architectural Fitness Functions?

First introduced in *Building Evolutionary Architectures* (Neal Ford, Rebecca Parsons, Patrick Kua), an **Architectural Fitness Function** is an objective, automated metric or test used to evaluate and preserve critical architectural characteristics (modularity, layer boundaries, loose coupling, security invariants) as a codebase evolves.

In .NET, architects write fitness functions using reflection or libraries like **NetArchTest.Rules** or **ArchUnit.NET**.

```
┌─────────────────────────────────────────────────────────────┐
│ GitHub Actions / Azure DevOps CI Pipeline                   │
│                                                             │
│   ├── Step 1: dotnet build                                  │
│   │                                                         │
│   ├── Step 2: Unit & Integration Tests                      │
│   │                                                         │
│   └── Step 3: Normora.Architecture.Tests                    │
│       ├── Rule: Controllers never inject DbContext?         │
│       ├── Rule: Module boundaries strictly isolated?        │
│       ├── Rule: Infrastructure doesn't reference hosts?     │
│       └── Rule: Zero technical errors in email templates?   │
│                                                             │
│   If ANY rule fails ──▶ BUILD FAILS! PULL REQUEST BLOCKED!  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Real-World Architecture Tests from Normora

Let's examine the exact, battle-tested architecture tests from `Normora.Architecture.Tests`:

### Invariant 1: Controllers Must NEVER Inject `DbContext`
Controllers must be razor-thin dispatchers. Injecting a `DbContext` into a controller bypasses MediatR validation, audit logging, and domain event dispatching.

```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void Controllers_MustNeverInject_DbContexts()
{
    var assemblies = new[] { ConversationsAssembly, DocumentsAssembly, TenantsAssembly };

    var controllerTypes = assemblies
        .SelectMany(a => a.GetTypes())
        .Where(t => t.IsClass && !t.IsAbstract && typeof(ControllerBase).IsAssignableFrom(t))
        .ToList();

    Assert.NotEmpty(controllerTypes);

    foreach (var controller in controllerTypes)
    {
        var constructors = controller.GetConstructors();
        foreach (var ctor in constructors)
        {
            foreach (var param in ctor.GetParameters())
            {
                Assert.False(
                    typeof(DbContext).IsAssignableFrom(param.ParameterType),
                    $"Violation: Controller '{controller.FullName}' must not inject DbContext '{param.ParameterType.Name}'. " +
                    $"Controllers must dispatch MediatR contracts or service abstractions.");
            }
        }
    }
}
```

---

### Invariant 2: Strict Module Boundary Isolation
Modules in a Modular Monolith must never have direct assembly references to one another. Communication must flow strictly through `Normora.Shared.Contracts` or in-memory domain events.

```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void BusinessModules_MustNotReferenceEachOther()
{
    var moduleAssemblies = new[] { TenantsAssembly, DocumentsAssembly, ConversationsAssembly };

    foreach (var module in moduleAssemblies)
    {
        var otherModuleNames = moduleAssemblies
            .Where(m => m != module)
            .Select(m => m.GetName().Name!)
            .ToList();

        var referencedAssemblies = module.GetReferencedAssemblies()
            .Select(r => r.Name)
            .ToList();

        foreach (var other in otherModuleNames)
        {
            Assert.DoesNotContain(
                other, 
                referencedAssemblies, 
                $"Architectural Violation: Module '{module.GetName().Name}' references '{other}'. " +
                $"Modules must be completely decoupled.");
        }
    }
}
```

---

### Invariant 3: Decoupling Hosts from Infrastructure
The `Worker` host and the `Api` host are executable processes. The `Infrastructure` project must never reference executable hosts:

```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void Infrastructure_MustNotReferenceExecutableHosts()
{
    var hostReferences = InfrastructureAssembly.GetReferencedAssemblies()
        .Where(reference =>
            string.Equals(reference.Name, "Normora.Api", StringComparison.Ordinal) ||
            string.Equals(reference.Name, "Normora.Worker", StringComparison.Ordinal))
        .Select(reference => reference.Name)
        .ToList();

    Assert.Empty(hostReferences);
}
```

---

### Invariant 4: Security Invariant — Zero Error Detail Leakage in Emails
To prevent **CWE-209 (Information Exposure Through Error Messages)**, the architecture test inspects rendered email notification templates to guarantee that raw HTTP status codes, monospace stack trace blocks, or unescaped HTML tags never appear:

```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void DocumentProcessingFailedEmail_MustNotExpose_TechnicalErrorDetails_Or_RawStatusCodes()
{
    const string tenantName = "Acme Corp <script>";
    const string fileName = "policy_handbook.pdf";

    var emailHtml = DocumentEmailTemplates.DocumentProcessingFailedEmail(tenantName, fileName);

    // Security check: Raw error detail container must never exist in notification emails
    Assert.DoesNotContain("Error Details", emailHtml, StringComparison.OrdinalIgnoreCase);
    Assert.DoesNotContain("font-family: monospace", emailHtml, StringComparison.OrdinalIgnoreCase);
    Assert.DoesNotContain("429 (Too Many Requests)", emailHtml, StringComparison.OrdinalIgnoreCase);
    Assert.DoesNotContain("500 (Internal Server Error)", emailHtml, StringComparison.OrdinalIgnoreCase);

    // Security check: Dynamic inputs must be properly HTML escaped to mitigate XSS
    Assert.DoesNotContain("<script>", emailHtml, StringComparison.OrdinalIgnoreCase);
    Assert.Contains("&lt;script&gt;", emailHtml, StringComparison.OrdinalIgnoreCase);
}
```

---

## 3. Distributed Caching & Environment Resiliency

In cloud deployments, external services like Redis might occasionally be unavailable or absent during local development and testing.

A resilient system must support **graceful environment degradation**:

```csharp
// From Normora.Infrastructure/Extensions/DatabaseServiceExtensions.cs
var redisOptions = configuration.GetSection(RedisOptions.SectionName).Get<RedisOptions>() 
    ?? new RedisOptions();

if (string.IsNullOrWhiteSpace(redisOptions.ConnectionString))
{
    redisOptions.ConnectionString = configuration.GetConnectionString(RedisOptions.SectionName)
        ?? configuration[RedisOptions.EnvironmentVariable];
}

if (!string.IsNullOrWhiteSpace(redisOptions.ConnectionString))
{
    // Production / Staging: Distributed Redis Cache
    services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = redisOptions.ConnectionString;
        options.InstanceName = redisOptions.InstanceName;
    });
}
else
{
    // In Production, running without Redis is fatal: Fail-Fast!
    if (!environment.IsDevelopment())
    {
        throw new InvalidOperationException(ApiMessages.RedisRequired);
    }

    // Local Development & Integration Tests: Graceful fallback to memory cache
    services.AddDistributedMemoryCache();
}

services.AddSingleton<ICacheService, DistributedCacheService>();
```

### Architectural Principles Applied:
1.  **Seamless Local Dev:** Developers can clone and run the application locally without requiring a Docker Redis instance if they only need to test basic features.
2.  **Fail-Fast Production Invariant:** If the production environment lacks a Redis connection string, the container crashes immediately on boot with `ApiMessages.RedisRequired`. It never silently degrades to an in-memory cache that would break distributed session sharing across multiple pods.

---

## 4. Cloud Health Checks: Liveness vs. Readiness Probes

In container orchestrators like Kubernetes, container health is monitored using **Health Probes**. ASP.NET Core provides a native health check subsystem:

```csharp
// In Composition Root:
builder.Services.AddHealthChecks()
    .AddNpgsql(connectionString, name: "postgresql")
    .AddRedis(redisConnectionString, name: "redis");

// In Request Pipeline:
app.MapHealthChecks("/healthz").AllowAnonymous();
```

### The Architect's Distinction: Liveness vs. Readiness
*   **Liveness Probe (`/healthz/liveness`):** "Is the process alive or deadlocked?"
    *   If this returns 500 or times out, Kubernetes immediately kills the container and creates a replacement pod.
    *   *Rule:* Never check external dependencies (database, Redis) in a liveness probe! If PostgreSQL has a 5-second network hiccup, all 20 API pods would fail their liveness probe simultaneously, causing Kubernetes to restart every pod at once — turning a minor network blip into a catastrophic cluster-wide outage!
*   **Readiness Probe (`/healthz/readiness`):** "Is this pod ready to accept user traffic?"
    *   Checks whether PostgreSQL, Redis, and message queues are accessible.
    *   If this fails, Kubernetes leaves the container alive, but stops routing user requests to it until it recovers.

---

## 5. Zero-Downtime Database Migration Deployments

How do software architects upgrade database schemas in production without taking the application offline?

### The Two Golden Rules:
1.  **Decouple Migration from Web Pod Boot:** Use the CLI pattern (`dotnet Normora.Api.dll --migrate`) in an ephemeral Kubernetes `Job` or CI/CD deployment step *before* updating web pods.
2.  **The Expand / Contract Pattern (Parallel Change):** Never alter or delete a database column in the same deployment where application code stops using it.

```
PHASE 1: EXPAND (Deployment N)
- Add new nullable column: `payment_reference`
- Write code that reads from old column `stripe_id`, but writes to both `stripe_id` and `payment_reference`.
- Database schema is backward-compatible with older replica pods!

PHASE 2: DATA BACKFILL (Background Worker)
- Background script copies existing `stripe_id` values to `payment_reference`.

PHASE 3: SWITCH (Deployment N+1)
- Application code now reads and writes exclusively from `payment_reference`.

PHASE 4: CONTRACT (Deployment N+2)
- Remove old column `stripe_id` via a subsequent migration.
```

---

## 6. Summary Checklist for Software Architects

* [ ] **Automate Architecture Tests:** Enforce that Controllers never inject `DbContext`, module boundaries are strictly isolated, and hosts are decoupled.
* [ ] **Sanitize Technical Information (CWE-209):** Verify that notification templates and client-facing error payloads never expose raw technical details or stack traces.
* [ ] **Resilient Caching with Fail-Fast:** Provide in-memory cache fallbacks for local development, while strictly enforcing Redis requirements in production.
* [ ] **Partition Health Checks:** Separate Liveness (process health) from Readiness (dependency health) to avoid cascading container restarts.
* [ ] **Zero-Downtime Schema Evolutions:** Use ephemeral migration runners and the Expand/Contract pattern for zero-downtime rolling upgrades.
