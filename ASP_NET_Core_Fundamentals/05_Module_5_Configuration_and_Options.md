# Module 5: Configuration and the Options Pattern

Every real-world application needs settings that change based on where it runs. Your development machine uses `localhost:5432` for the database; production uses `prod-db.company.com:5432`. Your local environment uses a test API key; production uses the real one. **Configuration** is the system that manages these values, and ASP.NET Core has one of the most sophisticated configuration systems of any web framework.

## 1. What is Configuration?

Configuration is **externalized data** that your application reads at runtime to control its behavior. Think of it as a set of knobs and dials that you can adjust without recompiling your code.

Examples of configuration:
*   Database connection strings
*   Third-party API keys (Stripe, SendGrid, Gemini)
*   Feature flags (enable/disable features)
*   Logging verbosity levels
*   CORS allowed origins
*   JWT token signing keys

> **Analogy — The Car Dashboard:**
> Your car's engine (application code) is the same whether you're driving in the city or on the highway. But the dashboard settings (configuration) change — you adjust the AC, the mirrors, and the seat position. You don't rebuild the engine to change these.

## 2. The Hierarchical Configuration Model

### How It Works
ASP.NET Core doesn't load configuration from a single source. It layers **multiple sources** on top of each other. When the same key exists in multiple sources, the **last one loaded wins** (overrides).

`WebApplication.CreateBuilder(args)` sets up these sources **in this order:**

```
Priority (lowest → highest):

1. appsettings.json                    ← Base defaults
2. appsettings.{Environment}.json      ← Environment overrides
3. User Secrets                        ← Local dev secrets (NOT in Git)
4. Environment Variables               ← Server/Docker/Cloud
5. Command-line arguments              ← Highest priority
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
*   **Development** — Your local machine. Enables detailed error pages, Swagger, hot-reload.
*   **Staging** — Pre-production environment for testing.
*   **Production** — The live environment. Enables HTTPS enforcement, optimized logging, error hiding.

## 3. The Configuration Sources in Detail

### `appsettings.json` — The Base
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
For sensitive data during development:
```bash
# Store a secret (stored in your user profile, NOT in the project folder)
dotnet user-secrets set "PaymentGateway:ApiKey" "sk_live_real_key_here"
```

**Why?** Because `appsettings.json` is committed to Git. If you put a real API key there, it's visible to everyone with repository access. User Secrets are stored in your OS user profile directory, completely outside the project folder.

### Environment Variables — The Production Standard
In production (Docker, Kubernetes, Azure, AWS), you inject configuration via environment variables. ASP.NET Core automatically reads them.

```bash
# In Docker Compose or Kubernetes config:
ConnectionStrings__DefaultConnection=Server=prod-db;Database=MyApp;Password=s3cret;
PaymentGateway__ApiKey=sk_live_production_key
```

Note the **double underscore `__`** — it replaces the colon `:` used in JSON nesting. `PaymentGateway__ApiKey` maps to `PaymentGateway:ApiKey` in configuration.

## 4. Reading Configuration — The Raw Way (And Why It's Bad)

### Using `IConfiguration` Directly
```csharp
public class PaymentsController : ControllerBase
{
    private readonly IConfiguration _config;

    public PaymentsController(IConfiguration config) => _config = config;

    [HttpGet]
    public IActionResult GetConfig()
    {
        var apiKey = _config["PaymentGateway:ApiKey"];          // Magic string!
        var timeout = int.Parse(_config["PaymentGateway:TimeoutSeconds"]); // Manual parsing!
        
        return Ok(new { apiKey, timeout });
    }
}
```

**What's wrong with this?**
1. **Magic strings** — `"PaymentGateway:ApiKey"` has no compile-time checking. Typo? Silent `null` at runtime.
2. **Manual parsing** — You must convert `"30"` to `int` yourself. Miss it? Runtime crash.
3. **No IntelliSense** — Your IDE can't help you discover available config keys.
4. **Scattered access** — Every class that needs config reads it differently. No single source of truth.
5. **Hard to test** — How do you mock `IConfiguration` in unit tests? It's painful.

## 5. The Options Pattern — The Professional Way

The Options Pattern solves every problem above by **binding configuration sections to strongly-typed C# classes**.

### Step 1: Define the Options Class
Create a plain C# class (a POCO) whose properties match the JSON structure:

```csharp
public class PaymentOptions
{
    // These property names must EXACTLY match the JSON keys
    public string ApiKey { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; } = 30;  // Default value!
    public int RetryCount { get; set; } = 3;
}
```

### Step 2: Bind in Program.cs
Tell the DI container to map the `"PaymentGateway"` JSON section to the `PaymentOptions` class:

```csharp
builder.Services.Configure<PaymentOptions>(
    builder.Configuration.GetSection("PaymentGateway"));
```

### Step 3: Inject `IOptions<T>`
```csharp
public class PaymentsController : ControllerBase
{
    private readonly PaymentOptions _options;

    public PaymentsController(IOptions<PaymentOptions> options)
    {
        _options = options.Value;  // Access the bound object
    }

    [HttpPost("charge")]
    public IActionResult Charge(ChargeDto dto)
    {
        // ✅ Strongly typed! IntelliSense! Compile-time safety!
        var client = new PaymentClient(_options.ApiKey, _options.TimeoutSeconds);
        client.Charge(dto.Amount);
        return Ok();
    }
}
```

### Step 4 (Advanced): Validation on Startup
You can validate your configuration at startup to fail fast if required values are missing:

```csharp
builder.Services.AddOptions<PaymentOptions>()
    .Bind(builder.Configuration.GetSection("PaymentGateway"))
    .ValidateDataAnnotations()  // Use [Required], [Range], etc. on the Options class
    .ValidateOnStart();         // Crash at startup if validation fails
```

```csharp
public class PaymentOptions
{
    [Required(ErrorMessage = "PaymentGateway:ApiKey is required")]
    public string ApiKey { get; set; } = string.Empty;

    [Range(1, 120, ErrorMessage = "Timeout must be between 1 and 120 seconds")]
    public int TimeoutSeconds { get; set; } = 30;
}
```

## 6. IOptions vs IOptionsSnapshot vs IOptionsMonitor

This is where most tutorials stop. But in real applications, you need to know **when** configuration values are read.

| Interface | Lifetime | Reloads on change? | Use when... |
|---|---|---|---|
| `IOptions<T>` | Singleton | ❌ No — reads once at startup | Values that never change (JWT signing keys) |
| `IOptionsSnapshot<T>` | Scoped | ✅ Yes — re-reads per request | Most web scenarios (connection strings, API keys) |
| `IOptionsMonitor<T>` | Singleton | ✅ Yes — notifies on change | Singleton services that need live updates |

### When Does Configuration Change?
If you edit `appsettings.json` while the app is running:
*   `IOptions<T>` — **Never sees the change.** It read the value at startup and cached it forever.
*   `IOptionsSnapshot<T>` — **Sees the change on the next HTTP request.** Perfect for web apps.
*   `IOptionsMonitor<T>` — **Sees the change immediately** via a callback. Used in Singleton services like background workers.

```csharp
// IOptionsMonitor example — reacting to live config changes
public class BackgroundWorker : BackgroundService
{
    private readonly IOptionsMonitor<PaymentOptions> _optionsMonitor;

    public BackgroundWorker(IOptionsMonitor<PaymentOptions> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
        
        // Register a callback that fires whenever the config file changes
        _optionsMonitor.OnChange(newOptions =>
        {
            Console.WriteLine($"Config changed! New timeout: {newOptions.TimeoutSeconds}");
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Always reads the LATEST value
            var currentTimeout = _optionsMonitor.CurrentValue.TimeoutSeconds;
            await Task.Delay(TimeSpan.FromSeconds(currentTimeout), stoppingToken);
        }
    }
}
```

## 7. Real-World Configuration from Your Projects

### 🏗️ NextEvent: Fail-Fast on Missing Connection String
```csharp
// From NextEvent's DatabaseServiceExtensions.cs
var connectionString = configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException(
        "Connection string 'DefaultConnection' was not found.");
```

**Why throw immediately?** If the connection string is missing, every database operation will fail. Instead of discovering this on the first user request (which might be minutes later), crash at startup with a clear error message. This is the **Fail Fast** principle.

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

**Why?** The same compiled binary runs in Development (`http://localhost:4200`) and Production (`https://app.nextevent.com`). The CORS origins are read from configuration, not hardcoded.

### 🏗️ NextEvent: Database Resilience Configuration
```csharp
// From NextEvent's DatabaseServiceExtensions.cs
options.UseSqlServer(connectionString, sqlOptions =>
{
    sqlOptions.MigrationsAssembly(migrationsAssembly);  // Module-specific migrations
    sqlOptions.EnableRetryOnFailure(
        maxRetryCount: 5,                                // Retry 5 times
        maxRetryDelay: TimeSpan.FromSeconds(10),         // Wait up to 10s between retries
        errorNumbersToAdd: null);                        // Retry on all transient errors
    sqlOptions.CommandTimeout(60);                       // 60-second query timeout
});
```

**Why `EnableRetryOnFailure`?** In cloud environments, database connections can briefly drop during failovers, scaling events, or network blips. Without retry logic, a single blip returns a 500 error. With retry logic, the query transparently retries and the user never notices.

### 🏗️ Normora: Centralized Configuration Extension
```csharp
// From Normora's Program.cs
builder.Services.AddConfigurationServices(builder.Configuration);
```

Normora groups all `IOptions<T>` bindings (MinIO storage, Keycloak auth, OpenTelemetry, email settings) into a single extension method, keeping `Program.cs` clean.

## 8. Best Practices Summary

| Practice | Why |
|---|---|
| **Never hardcode configuration values** | Use `appsettings.json` + environment variables |
| **Use the Options Pattern, not raw `IConfiguration`** | Type safety, IntelliSense, testability |
| **Use `IOptionsSnapshot<T>` for web apps** | Auto-reloads when config file changes |
| **Validate options on startup (`.ValidateOnStart()`)** | Fail fast with clear errors |
| **Store secrets in User Secrets (dev) / Env Vars (prod)** | Never commit passwords to Git |
| **Throw on missing critical config** | `?? throw new InvalidOperationException(...)` |
| **Use `EnableRetryOnFailure` for cloud databases** | Handle transient network failures |
| **Load CORS origins from config** | Same binary, multiple environments |

---
**Key Takeaways:**
1. Configuration is layered — `appsettings.json` → `appsettings.{Environment}.json` → User Secrets → Environment Variables. Last one wins.
2. The Options Pattern binds JSON sections to C# classes for type safety and IntelliSense.
3. Use `IOptionsSnapshot<T>` for live-reloading in web requests, `IOptionsMonitor<T>` for Singletons.
4. Never store secrets in files committed to source control. Use User Secrets or environment variables.

**Next up:** Module 6 — **Model Binding and Validation**. How does ASP.NET Core take a raw JSON body and magically turn it into a C# object? And how do you ensure that object contains valid data?
