# Module 5: Configuration, the Options Pattern & Cloud Secrets

Every real-world application needs settings that change based on where it runs. Your development machine uses `localhost:5432` for the database; production uses `prod-db.company.com:5432`. Your local environment uses a test API key; production uses the real one. **Configuration** is the system that manages these values, and ASP.NET Core has one of the most sophisticated, extensible configuration systems of any modern web framework.

---

## 1. What is Configuration?

Configuration is **externalized data** that your application reads at runtime to control its behavior. Think of it as a set of knobs and dials that you can adjust without recompiling your code.

Examples of configuration:
*   Database connection strings
*   Third-party API keys (Stripe, SendGrid, Gemini AI)
*   Feature flags (enable/disable features dynamically)
*   Logging verbosity levels
*   CORS allowed origins
*   JWT token signing keys

> **Analogy — The Car Dashboard:**  
> Your car's engine (application code) is the same whether you're driving in the city or on the highway. But the dashboard settings (configuration) change — you adjust the AC, the mirrors, and the seat position. You don't rebuild the engine to change these.

---

## 2. The Hierarchical Configuration Model

### How It Works
ASP.NET Core does not read configuration from a single source. It layers **multiple sources** on top of each other. When the same key exists in multiple sources, the **last one loaded wins** (overrides).

`WebApplication.CreateBuilder(args)` sets up these sources **in this order:**

```
PRIORITY LADDER (Lowest → Highest):

1. appsettings.json                    ← Base defaults for all environments
2. appsettings.{Environment}.json      ← Environment overrides (e.g. Development)
3. User Secrets                        ← Local dev secrets (NOT in Git)
4. Environment Variables               ← Server / Docker / Kubernetes cloud secrets
5. Command-line arguments              ← Highest priority (Overrides everything)
```

### Why This Layered Model is Brilliant

**Scenario:** You have a database connection string.

| Source | Value | Purpose |
|---|---|---|
| `appsettings.json` | `Server=localhost;Database=MyApp;...` | Default for local development |
| `appsettings.Production.json` | `Server=prod-db.internal;Database=MyApp;...` | Production override |
| Environment Variable | `ConnectionStrings__DefaultConnection=...` | Docker/Kubernetes secret injection |

The same application binary works in every environment without recompilation. The only thing that changes is *which configuration sources are available.*

### Environment Names
ASP.NET Core uses the `ASPNETCORE_ENVIRONMENT` environment variable to determine the current environment. The three standard names are:
*   **Development** — Your local machine. Enables detailed error pages, Scalar/Swagger docs, hot-reload.
*   **Staging** — Pre-production environment for testing.
*   **Production** — The live environment. Enables HTTPS enforcement, optimized logging, error shielding.

---

## 3. The Configuration Sources in Detail

### `appsettings.json` — The Base Defaults
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;Trusted_Connection=true;"
  },
  "AllowedOrigins": ["http://localhost:4200"],
  "PaymentGateway": {
    "ApiKey": "test_key_123",
    "TimeoutSeconds": 30,
    "RetryCount": 3
  }
}
```

### `appsettings.Development.json` — Dev Overrides
Only loaded when `ASPNETCORE_ENVIRONMENT=Development`:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"  // More verbose logging in dev
    }
  }
}
```

### User Secrets — Local Secrets (Never Committed to Git)
For sensitive data during local development:
```bash
# Initialize and store a secret (stored in your OS user profile, NOT in the repo)
dotnet user-secrets init
dotnet user-secrets set "PaymentGateway:ApiKey" "sk_test_secret_key_here"
```

**Why?** Because `appsettings.json` is committed to Git. If you put a real API key or password there, it is visible to everyone with repository access. User Secrets are stored outside the project folder.

### Environment Variables — The Production & Container Standard
In production (Docker, Kubernetes, Azure, AWS), you inject configuration via environment variables:

```bash
# In Docker Compose or Kubernetes deployment manifests:
ConnectionStrings__DefaultConnection=Server=prod-db.internal;Database=MyApp;Password=s3cret;
PaymentGateway__ApiKey=sk_live_production_key
```

Note the **double underscore `__`** — it replaces the colon `:` used in JSON nesting because Linux shells cannot parse colons in variable names. `PaymentGateway__ApiKey` automatically maps to `PaymentGateway:ApiKey`.

---

## 4. Reading Configuration — The Raw Way (And Why It's Bad)

### Using `IConfiguration` Directly (The Anti-Pattern)
```csharp
// ❌ RAW ICONFIGURATION ANTI-PATTERN
public class PaymentService
{
    private readonly IConfiguration _configuration;

    public PaymentService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void ProcessPayment()
    {
        // 1. Magic strings: Typos compile successfully but fail at runtime!
        var apiKey = _configuration["PaymentGateway:ApiKey"];

        // 2. Manual parsing: Fragile type conversions
        var timeout = int.Parse(_configuration["PaymentGateway:TimeoutSeconds"] ?? "30");
        var retries = int.Parse(_configuration["PaymentGateway:RetryCount"] ?? "3");
    }
}
```

### Why Raw `IConfiguration` Fails Enterprise Standards:
*   **No Type Safety:** Everything is retrieved as an untyped string.
*   **No IntelliSense:** You have to guess or remember exact JSON property names.
*   **No Validation:** If a key is missing, you get `null`, leading to a `NullReferenceException` deep in production.
*   **Scattered Secrets:** Configuration paths are hardcoded across dozens of business classes.

---

## 5. The Options Pattern: Strongly-Typed Configuration

The **Options Pattern** binds JSON configuration sections directly to plain C# classes (POCOs).

### Step 1: Create the Options Class
```csharp
public class PaymentGatewayOptions
{
    public const string SectionName = "PaymentGateway";

    public string ApiKey { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; } = 30;
    public int RetryCount { get; set; } = 3;
}
```

### Step 2: Register in Program.cs
```csharp
// Binds the "PaymentGateway" section of appsettings.json to PaymentGatewayOptions
builder.Services.Configure<PaymentGatewayOptions>(
    builder.Configuration.GetSection(PaymentGatewayOptions.SectionName));
```

### Step 3: Inject `IOptions<T>` into Your Service
```csharp
// ✅ CLEAN, STRONGLY-TYPED INJECTION
public class PaymentService
{
    private readonly PaymentGatewayOptions _options;

    public PaymentService(IOptions<PaymentGatewayOptions> options)
    {
        _options = options.Value; // Strongly-typed, verified, IntelliSense-enabled!
    }

    public void ProcessPayment()
    {
        var apiKey = _options.ApiKey;
        var timeout = _options.TimeoutSeconds; // Already an int! No parsing needed.
    }
}
```

---

## 6. The Options Trinity: `IOptions` vs. `IOptionsSnapshot` vs. `IOptionsMonitor`

ASP.NET Core provides three distinct interfaces to consume options:

| Interface | Service Lifetime | Reads Changes at Runtime? | Injected into Singletons? | Best Use Case |
|---|---|---|---|---|
| **`IOptions<T>`** | **Singleton** | ❌ No (Reads once at startup) | ✅ **Yes** | Infrastructure (DB connections, Redis, AWS/Azure URLs) |
| **`IOptionsSnapshot<T>`** | **Scoped** | ✅ **Yes** (Re-read per HTTP request) | ❌ **NO!** (Captive dependency hazard) | Per-request business rules, tenant feature flags |
| **`IOptionsMonitor<T>`** | **Singleton** | ✅ **Yes** (Live push notifications) | ✅ **Yes** | Background workers, live-reloading Singleton services |

### Using `IOptionsSnapshot<T>` for Live Reloading in Web Requests
```csharp
// Automatically re-reads configuration on every HTTP request
public class DiscountService(IOptionsSnapshot<DiscountOptions> options)
{
    public decimal GetDiscount() => options.Value.CurrentPercentage;
}
```

### Using `IOptionsMonitor<T>` in Singletons & Background Workers
```csharp
public class PollingWorker : BackgroundService
{
    private readonly IOptionsMonitor<PaymentGatewayOptions> _optionsMonitor;

    public PollingWorker(IOptionsMonitor<PaymentGatewayOptions> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;

        // Register a callback for when configuration changes on disk:
        _optionsMonitor.OnChange(newOptions =>
        {
            Console.WriteLine($"Config changed on disk! New timeout: {newOptions.TimeoutSeconds}");
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Always reads the LATEST value without app restart
            var currentTimeout = _optionsMonitor.CurrentValue.TimeoutSeconds;
            await Task.Delay(TimeSpan.FromSeconds(currentTimeout), stoppingToken);
        }
    }
}
```

---

## 7. Fail-Fast Startup Validation: `ValidateOnStart()`

A fundamental rule of resilient software architecture is: **Fail Fast, Fail Loudly.**

If an application is missing its database connection string, JWT signing secret, or third-party AI credentials, **the container must refuse to start**.

### Implementing Startup Validation via DataAnnotations:
```csharp
public class KeycloakOptions
{
    public const string SectionName = "Keycloak";

    [Required(ErrorMessage = "Keycloak Authority URL is mandatory.")]
    [Url(ErrorMessage = "Keycloak Authority must be a valid URL.")]
    public string Authority { get; set; } = string.Empty;

    [Required(ErrorMessage = "Keycloak ClientId is mandatory.")]
    public string ClientId { get; set; } = string.Empty;

    [Required(ErrorMessage = "Keycloak ClientSecret is mandatory.")]
    public string ClientSecret { get; set; } = string.Empty;

    [Range(1, 60, ErrorMessage = "TokenClockSkewSeconds must be between 1 and 60.")]
    public int TokenClockSkewSeconds { get; set; } = 5;
}
```

### Registering with `ValidateOnStart()`:
```csharp
builder.Services.AddOptions<KeycloakOptions>()
    .Bind(builder.Configuration.GetSection(KeycloakOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // 🛡️ Throws OptionsValidationException immediately during builder.Build()!
```

---

## 8. Real-World Configuration from Your Projects

### 🏗️ NextEvent: Fail-Fast on Missing Connection String
```csharp
// From NextEvent's DatabaseServiceExtensions.cs
var connectionString = configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException(
        "Connection string 'DefaultConnection' was not found.");
```

**Why throw immediately?** If the connection string is missing, every database operation will fail. Crash at startup with a clear error message rather than failing minutes later under user load.

### 🏗️ NextEvent: Securing the JWT Token Key
```csharp
// From NextEvent's IdentityServiceExtensions.cs
var tokenKey = config["TokenKey"]
    ?? throw new InvalidOperationException(
        "TokenKey is not configured. Set it via User Secrets (development) " +
        "or an environment variable (production).");
```

The JWT signing key is **never** stored in `appsettings.json`. Developers use `dotnet user-secrets set "TokenKey" "my-super-secret-key"` locally, and in production, it's injected as an environment variable.

### 🏗️ NextEvent: CORS Origins from Configuration
```csharp
// From NextEvent's ApiServiceExtensions.cs
var allowedOrigins = configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [];

services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.WithOrigins(allowedOrigins); // Not hardcoded!
    });
});
```

### 🏗️ NextEvent: Database Resilience Configuration
```csharp
// From NextEvent's DatabaseServiceExtensions.cs
options.UseSqlServer(connectionString, sqlOptions =>
{
    sqlOptions.MigrationsAssembly(migrationsAssembly);
    sqlOptions.EnableRetryOnFailure(
        maxRetryCount: 5,                                // Retry 5 times
        maxRetryDelay: TimeSpan.FromSeconds(10),         // Wait up to 10s between retries
        errorNumbersToAdd: null);                        // Retry on transient network blips
    sqlOptions.CommandTimeout(60);                       // 60-second query timeout
});
```

**Why `EnableRetryOnFailure`?** In cloud environments, database connections can briefly drop during failovers or network rebalancing. With retry logic, the query retries transparently and the user never experiences a 500 error.

### 🏗️ Normora: Production `RedisOptions`
```csharp
// From Normora.Infrastructure/Options/RedisOptions.cs
public class RedisOptions
{
    public const string SectionName = "Redis";
    public const string EnvironmentVariable = "REDIS_CONNECTION";

    public string? ConnectionString { get; set; }
    public string InstanceName { get; set; } = "normora:";
    public string ChannelPrefix { get; set; } = "normora:signalr";
}
```

And verified via architecture tests:
```csharp
// From Normora.Architecture.Tests/ModularMonolithArchitectureTests.cs
[Fact]
public void RedisOptions_DefaultValues_AreStronglyTypedAndConsistent()
{
    var options = new Normora.Infrastructure.Options.RedisOptions();

    Assert.Equal("Redis", Normora.Infrastructure.Options.RedisOptions.SectionName);
    Assert.Equal("REDIS_CONNECTION", Normora.Infrastructure.Options.RedisOptions.EnvironmentVariable);
    Assert.Equal("normora:signalr", options.ChannelPrefix);
    Assert.Equal("normora:", options.InstanceName);
    Assert.Null(options.ConnectionString);
}
```

---

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Never hardcode configuration values** | Use `appsettings.json` + environment variables |
| **Use the Options Pattern, not raw `IConfiguration`** | Type safety, IntelliSense, testability |
| **Use `IOptionsSnapshot<T>` for web apps** | Auto-reloads when config changes on disk |
| **Validate options on startup (`.ValidateOnStart()`)** | Fail fast with clear errors before serving traffic |
| **Store secrets in User Secrets (dev) / Env Vars (prod)** | Never commit passwords or private keys to Git |
| **Throw on missing critical config** | `?? throw new InvalidOperationException(...)` |
| **Use `EnableRetryOnFailure` for cloud databases** | Transparently handle transient network failures |
| **Load CORS origins from config** | Same binary functions cleanly across dev, staging, and prod |

---
**Key Takeaways:**
1. Configuration is layered — `appsettings.json` → `appsettings.{Environment}.json` → User Secrets → Environment Variables. Last one wins.
2. The Options Pattern binds JSON sections to C# classes for type safety and IntelliSense.
3. Use `IOptions<T>` for Singletons, `IOptionsSnapshot<T>` for web requests, and `IOptionsMonitor<T>` for live background workers.
4. Enforce `.ValidateOnStart()` to guarantee missing cloud secrets halt container boot immediately.

**Next up:** In [Module 6](06_Module_6_Model_Binding.md), we explore **Model Binding and Validation** — how ASP.NET Core deserializes HTTP payloads and enforces clean CQRS pipeline validation.
