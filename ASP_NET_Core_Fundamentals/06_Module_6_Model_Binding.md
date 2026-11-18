# Module 6: Model Binding, Validation & CQRS Pipeline

Your API receives raw HTTP data — bytes over the wire. A JSON body, query string parameters, URL segments, HTTP headers. Your C# code works with strongly-typed objects. **Model Binding** is the bridge between these two worlds. And **Validation** is the guard that ensures the data is sane before your business logic touches it.

In enterprise software architecture, validation must be **isolated, automated, and testable**. This module demonstrates how to implement model binding, compare DataAnnotations with FluentValidation, and integrate automated validation and audit logging directly into a **MediatR CQRS pipeline**.

---

## 1. What is Model Binding?

Model Binding is the **automatic process** by which ASP.NET Core reads incoming HTTP request data and converts it into C# method parameters and objects.

> **Analogy — The Mail Room:**  
> Imagine your office receives packages (HTTP requests). Each package has a shipping label (URL), a customs form (headers), and contents inside the box (body). The mail room (Model Binder) reads all of these, assembles the information into a structured internal form (C# object), and delivers it to the right department (Controller action).

### Before Model Binding (Manual Parsing)
```csharp
// ❌ Without model binding — Painful, error-prone
[HttpPost]
public async Task<IActionResult> CreateUser()
{
    var body = await new StreamReader(Request.Body).ReadToEndAsync();
    var json = JsonSerializer.Deserialize<Dictionary<string, object>>(body);
    var username = json["username"].ToString();
    var age = int.Parse(json["age"].ToString());
    var email = json["email"].ToString();
    // ... endless manual parsing, type conversion, null checks
}
```

### With Model Binding (Automatic)
```csharp
// ✅ With model binding — Clean, automatic
[HttpPost]
public IActionResult CreateUser(CreateUserDto dto)
{
    // 'dto' is already populated with data from the JSON body!
    // ASP.NET Core did all the parsing, deserializing, and type conversion.
    return Ok(dto);
}
```

---

## 2. Binding Sources — Where Data Comes From

HTTP requests carry data in **multiple places**. ASP.NET Core can read from all of them.

### The Four Primary Sources

| Attribute | Source | Example | Common Use |
|---|---|---|---|
| `[FromRoute]` | URL path segments | `/api/users/42` → `id = 42` | Resource identifiers |
| `[FromQuery]` | Query string | `?page=2&size=10` → `page = 2, size = 10` | Filters, pagination, search |
| `[FromBody]` | Request body (JSON) | `{ "name": "Alice" }` → `dto.Name = "Alice"` | POST/PUT payloads |
| `[FromHeader]` | HTTP headers | `X-Tenant-Id: abc123` | Metadata, tenant IDs, auth |

### Additional Sources

| Attribute | Source | Use Case |
|---|---|---|
| `[FromForm]` | Form-encoded body | File uploads, multipart forms |
| `[FromServices]` | DI container | Inject a service into a specific action only |

### How `[ApiController]` Changes the Rules
When a controller is annotated with `[ApiController]`, ASP.NET Core applies **inference rules** so you don't have to write binding attributes for every parameter:

| Parameter Type | Inferred Source |
|---|---|
| Simple types (`int`, `string`, `Guid`) | `[FromRoute]` if a matching route parameter exists, otherwise `[FromQuery]` |
| Complex types (classes, records) | `[FromBody]` |
| `IFormFile` or `IFormFileCollection` | `[FromForm]` |
| `CancellationToken` | Injected automatically by the framework |

This means:
```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // ASP.NET Core automatically infers:
    //   - 'role' comes from the route (matches {role})
    //   - 'sendEmail' comes from the query string (simple type, no route match)
    //   - 'dto' comes from the JSON body (complex type)
    [HttpPost("{role}")]
    public IActionResult CreateUser(string role, bool sendEmail, CreateUserDto dto)
    {
        // role = "admin" (from /api/users/admin)
        // sendEmail = true (from ?sendEmail=true)
        // dto = { Name: "Alice", Email: "alice@test.com" } (from JSON body)
        return Ok();
    }
}
```

### Explicit Binding (When You Need Explicit Contracts)
```csharp
[HttpPost("{role}")]
public IActionResult CreateUser(
    [FromRoute]  string role,                               // From URL: /api/users/admin
    [FromQuery]  bool sendEmail,                            // From query: ?sendEmail=true
    [FromBody]   CreateUserDto dto,                         // From JSON body
    [FromHeader(Name = "X-Request-Id")] string requestId)   // From HTTP header
{
    return Ok();
}
```

---

## 3. What is Validation? (And Why It's Non-Negotiable)

### The Golden Rule: Never Trust Client Data
Every byte of data coming from the internet is potentially malicious or malformed. A client could send:
*   An empty username
*   A negative age
*   A SQL injection payload in an email field
*   A 50MB string where a short title should be

**Validation** ensures that incoming data meets business rules **before** it touches your core domain logic.

> **Analogy — The Bouncer at a Club:**  
> Your business logic is the VIP section. The bouncer (validator) stands at the velvet rope. He checks ID, dress code, and guest list *before* anyone enters. If you don't meet the rules, you are turned away at the door (400 Bad Request) — you never enter the club.

---

## 4. Approach 1: Data Annotations (Trivial Validation)

DataAnnotations use C# attributes directly on DTO properties:

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(50, MinimumLength = 2, ErrorMessage = "Name must be 2-50 characters")]
    public string Name { get; set; } = string.Empty;

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; } = string.Empty;

    [Range(18, 120, ErrorMessage = "Must be between 18 and 120")]
    public int Age { get; set; }
}
```

### Common Data Annotation Attributes:
| Attribute | What it checks |
|---|---|
| `[Required]` | Value is not null, empty, or whitespace |
| `[StringLength(max, MinimumLength = min)]` | String length is within bounds |
| `[Range(min, max)]` | Numeric value is within bounds |
| `[EmailAddress]` | String matches standard email format |
| `[RegularExpression(pattern)]` | String matches custom regex |
| `[Compare("OtherProperty")]` | Two properties match (e.g., password confirmation) |

### Limitations of Data Annotations:
1.  **Pollutes DTOs:** Validation logic is coupled to the data transfer object.
2.  **No Complex Logic:** Conditional rules (*"If payment method is CreditCard, Cvv is required"*) become an unreadable mess of custom attributes.
3.  **Cannot Query External Services:** Attributes cannot easily perform async queries (e.g., checking if an email is already registered in PostgreSQL).
4.  **Difficult to Unit Test:** Testing requires instantiating validation contexts rather than pure unit tests.

---

## 5. Approach 2: FluentValidation (Enterprise Standard)

**FluentValidation** decouples validation into dedicated, strongly-typed validator classes.

```csharp
// Clean DTO
public record CreateUserDto(string Name, string Email, int Age, string ContactMethod, string? Phone);

// Dedicated Validator Class
public class CreateUserDtoValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserDtoValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .Length(2, 50).WithMessage("Name must be 2-50 characters");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .When(x => x.ContactMethod == "Email"); // Conditional rule!

        RuleFor(x => x.Phone)
            .NotEmpty().WithMessage("Phone is required when contact method is Phone")
            .When(x => x.ContactMethod == "Phone");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Age must be between 18 and 120");
    }
}
```

### Database-Dependent Validation (Injecting Services)
Because validators are standard classes registered in the DI container, they can inject database repositories or services:

```csharp
public class RegisterCommandValidator : AbstractValidator<RegisterCommand>
{
    public RegisterCommandValidator(IUserRepository repository)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .MustAsync(async (email, ct) =>
            {
                var exists = await repository.EmailExistsAsync(email, ct);
                return !exists; // Must return true to pass validation
            })
            .WithMessage("This email address is already registered.");
    }
}
```

---

## 6. The MediatR Pipeline Behavior Pattern

In Clean Architecture and Modular Monoliths, controllers do not perform validation. Instead, a **MediatR Pipeline Behavior** intercepts every command and query before it reaches the handler.

```
HTTP Request → Controller → MediatR → [ValidationBehavior] → [AuditBehavior] → Command Handler
                                            ↓
                                     Runs FluentValidation
                                            ↓
                                     If invalid → throws ValidationException
                                     If valid → continues down pipeline
```

### The Production `ValidationBehavior`:

Here is the exact implementation used in **Normora** and **NextEvent**:

```csharp
// From Normora.Shared/ValidationBehavior.cs
using FluentValidation;
using MediatR;

namespace Normora.Shared;

public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators) 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // 1. If no validators are registered for this request, pass through immediately
        if (!validators.Any())
        {
            return await next();
        }

        var context = new ValidationContext<TRequest>(request);

        // 2. Execute all validators asynchronously in parallel
        var validationResults = await Task.WhenAll(
            validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        // 3. Aggregate failures across all validator classes
        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        // 4. If any validation rules fail, throw ValidationException
        if (failures.Count != 0)
        {
            throw new ValidationException(failures);
        }

        // 5. All validation passed — continue to the actual Command Handler
        return await next();
    }
}
```

### The Controller in a CQRS Pipeline:
```csharp
[HttpPost]
public async Task<IActionResult> Register(RegisterCommand command, CancellationToken ct)
{
    // The controller contains ZERO validation boilerplate!
    var result = await _mediator.Send(command, ct);
    return Ok(result);
    // If command is invalid, ValidationBehavior throws ValidationException.
    // The global exception handler intercepts it and returns 400 Bad Request.
    // This action never executes with invalid data!
}
```

---

## 7. Cross-Cutting Pipeline Behaviors: The `AuditBehavior`

In **Normora**, any command mutating system state must record an immutable audit log. Rather than adding audit logging code inside every single command handler, Normora implements an **`AuditBehavior`**:

```csharp
// From Normora.Shared/AuditBehavior.cs
public sealed class AuditBehavior<TRequest, TResponse>(
    ILogger<AuditBehavior<TRequest, TResponse>> logger,
    ICurrentUser currentUser,
    ITenantContext tenantContext) 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;

        // Execute the command
        var response = await next();

        // If the command mutates state, record the audit entry
        if (requestName.EndsWith("Command", StringComparison.Ordinal))
        {
            logger.LogInformation(
                "Audit: Action {CommandName} executed by User {UserId} in Tenant {TenantId}",
                requestName,
                currentUser.ExternalUserId ?? "System",
                tenantContext.TenantId);
        }

        return response;
    }
}
```

---

## 8. Wiring Behaviors via Assembly Scanning

In your Composition Root (`ApplicationServiceExtensions.cs`), auto-discover validators and wire the pipeline:

```csharp
// From NextEvent's ApplicationServiceExtensions.cs
public static IServiceCollection AddApplicationServices(this IServiceCollection services)
{
    // Scan assemblies and auto-register all FluentValidation validators
    services.AddValidatorsFromAssemblyContaining<RegisterCommandValidator>();

    // Register MediatR with pipeline behaviors
    services.AddMediatR(cfg =>
    {
        cfg.RegisterServicesFromAssemblyContaining<LoginCommandHandler>();

        // Behaviors execute in the sequential order registered:
        cfg.AddOpenBehavior(typeof(ValidationBehavior<,>)); // 1. Validate inputs
        cfg.AddOpenBehavior(typeof(AuditBehavior<,>));      // 2. Audit execution
    });

    return services;
}
```

---

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Always use `[ApiController]`** | Automatic validation inference, binding rules, Problem Details |
| **Use DTOs, not Domain Entities** | Never expose internal database entities to external clients |
| **Prefer FluentValidation over Data Annotations** | Testable, injectable, supports conditionals and async rules |
| **Use a `ValidationBehavior` with MediatR** | Centralized validation before any business handler runs |
| **Always validate on the server** | Client-side validation is a UX feature, not a security boundary |
| **Use `[FromRoute]` for identifiers** | Clean, RESTful URLs: `/api/users/42` |
| **Use `[FromQuery]` for filters/pagination** | Standard URI query strings: `?page=2&pageSize=10` |
| **Use `[FromBody]` for command payloads** | Deserializes JSON payloads into strongly-typed commands |
| **Implement Cross-Cutting Audit Behaviors** | Record mutation telemetry centrally without cluttering handlers |

---
**Key Takeaways:**
1. Model Binding automatically converts HTTP data into C# objects — you rarely need to parse anything manually.
2. `[ApiController]` enables automatic binding inference and model validation.
3. FluentValidation is the enterprise standard — it separates validation from models and supports DI, conditionals, and async rules.
4. In CQRS architectures, validation runs in a MediatR pipeline behavior, keeping controllers razor-thin and business handlers protected.

**Next up:** In [Module 7](07_Module_7_Error_Handling.md), we explore **Error Handling and Logging** — how to record structured logs, adopt .NET 8+ `IExceptionHandler`, and prevent sensitive technical data leaks.
