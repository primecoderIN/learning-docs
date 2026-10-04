# Module 6: Model Binding and Validation

Your API receives raw HTTP data — bytes over the wire. A JSON body, query string parameters, URL segments, HTTP headers. Your C# code works with strongly-typed objects. **Model Binding** is the bridge between these two worlds. And **Validation** is the guard that ensures the data is sane before your business logic touches it.

## 1. What is Model Binding?

Model Binding is the **automatic process** by which ASP.NET Core reads incoming HTTP request data and converts it into C# method parameters and objects.

> **Analogy — The Mail Room:**
> Imagine your office receives packages (HTTP requests). Each package has a shipping label (URL), a customs form (headers), and contents inside the box (body). The mail room (Model Binder) reads all of these, assembles the information into a structured internal form (C# object), and delivers it to the right department (Controller action).

### Before Model Binding (Manual Parsing)
```csharp
// ❌ Without model binding — Painful, error-prone
[HttpPost]
public IActionResult CreateUser()
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
}
```

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
| `[FromForm]` | Form-encoded body | File uploads, HTML forms |
| `[FromServices]` | DI container | Inject a service into a specific action only |

### How `[ApiController]` Changes the Rules
When a controller is annotated with `[ApiController]`, ASP.NET Core applies **inference rules** so you don't have to write binding attributes for every parameter:

| Parameter Type | Inferred Source |
|---|---|
| Simple types (`int`, `string`, `Guid`) | `[FromRoute]` if a matching route parameter exists, otherwise `[FromQuery]` |
| Complex types (classes, records) | `[FromBody]` |
| `IFormFile` | `[FromForm]` |
| `CancellationToken` | Special — injected by the framework |

This means:
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // ASP.NET Core automatically infers:
    //   - 'role' comes from the route (because it matches {role})
    //   - 'sendEmail' comes from the query string (simple type, no route match)
    //   - 'dto' comes from the JSON body (complex type)
    [HttpPost("{role}")]
    public IActionResult CreateUser(string role, bool sendEmail, CreateUserDto dto)
    {
        // role = "admin" (from /api/users/admin)
        // sendEmail = true (from ?sendEmail=true)
        // dto = { Name: "Alice", Email: "alice@test.com" } (from JSON body)
    }
}
```

### Explicit Binding (When You Need Control)
Sometimes inference guesses wrong. Explicitly specify the source:

```csharp
[HttpPost("{role}")]
public IActionResult CreateUser(
    [FromRoute]  string role,              // From URL: /api/users/admin
    [FromQuery]  bool sendEmail,           // From query: ?sendEmail=true
    [FromBody]   CreateUserDto dto,        // From JSON body
    [FromHeader(Name = "X-Request-Id")] string requestId)  // From HTTP header
{
    // ...
}
```

## 3. What is Validation? (And Why It's Non-Negotiable)

### The Golden Rule: Never Trust Client Data
Every byte of data coming from the internet is potentially malicious or malformed. A client could send:
*   An empty username
*   A negative age
*   A SQL injection in the email field
*   A 50MB string where a name should be

**Validation** ensures that incoming data meets your rules **before** it reaches your business logic.

> **Analogy — The Bouncer at a Club:**
> Your business logic is the VIP section. The bouncer (validator) stands at the door checking everyone's ID. No ID? You're not getting in. Fake ID? Rejected. Under 18? Go home. Only verified guests enter the VIP area.

## 4. Data Annotations — The Built-in Approach

Data Annotations are attributes you place directly on your DTO (Data Transfer Object) properties.

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "Username is required")]
    [StringLength(50, MinimumLength = 3, 
        ErrorMessage = "Username must be between 3 and 50 characters")]
    public string Username { get; set; } = string.Empty;

    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; set; }

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; } = string.Empty;

    [Url(ErrorMessage = "Invalid URL format")]
    public string? Website { get; set; }

    [RegularExpression(@"^[A-Za-z0-9]+$", 
        ErrorMessage = "Only alphanumeric characters allowed")]
    public string? DisplayName { get; set; }
}
```

### Common Validation Attributes

| Attribute | What it checks |
|---|---|
| `[Required]` | Value is not null or empty |
| `[StringLength(max, MinimumLength)]` | String length within range |
| `[Range(min, max)]` | Numeric value within range |
| `[EmailAddress]` | Valid email format |
| `[Url]` | Valid URL format |
| `[Phone]` | Valid phone number format |
| `[RegularExpression(pattern)]` | Matches a regex pattern |
| `[Compare("OtherProperty")]` | Two properties match (e.g., Password confirmation) |
| `[CreditCard]` | Valid credit card number (Luhn algorithm) |

### Automatic 400 Bad Request (The Magic of `[ApiController]`)
When you use `[ApiController]`, ASP.NET Core **automatically validates** the model before your action runs. If validation fails, a `400 Bad Request` is returned immediately — your action method **never executes**.

**The automatic error response:**
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Username": ["Username is required", "Username must be between 3 and 50 characters"],
    "Age": ["Age must be between 18 and 120"],
    "Email": ["Invalid email format"]
  }
}
```

### Without `[ApiController]` (Manual Checking)
In older MVC apps or when you don't use `[ApiController]`:
```csharp
[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    if (!ModelState.IsValid)  // Manual check!
    {
        return BadRequest(ModelState);
    }
    // ... business logic
}
```

## 5. FluentValidation — The Enterprise Approach

### Why Not Just Use Data Annotations?
Data Annotations work, but they have drawbacks in large codebases:

1. **Cluttered models** — Your clean DTO class gets buried under attributes.
2. **No conditional logic** — You can't say "Email is required only if ContactMethod == Email."
3. **Can't inject services** — What if validation needs to check the database (e.g., "is this email already taken")?
4. **Hard to unit test** — Annotations are tested by the framework, not by you.

### FluentValidation: Separate Validator Classes
FluentValidation moves all validation logic into dedicated classes:

```csharp
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Username is required")
            .Length(3, 50).WithMessage("Username must be between 3 and 50 characters");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress();
    }
}
```

### Conditional Validation (FluentValidation's Superpower)
```csharp
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .When(x => x.ContactMethod == "Email");  // Only validate email if method is Email!

        RuleFor(x => x.Phone)
            .NotEmpty()
            .When(x => x.ContactMethod == "Phone");
    }
}
```

### Database-Dependent Validation (Injecting Services)
```csharp
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator(IUserRepository repository)  // DI works here!
    {
        RuleFor(x => x.Email)
            .MustAsync(async (email, cancellation) =>
            {
                var exists = await repository.EmailExistsAsync(email);
                return !exists;  // Must return true to pass validation
            })
            .WithMessage("This email is already registered");
    }
}
```

## 6. The MediatR + FluentValidation Pipeline

In architectures that use **CQRS (Command Query Responsibility Segregation)**, the Controller doesn't validate directly. Instead, a **Pipeline Behavior** intercepts every command before it reaches the handler.

### How It Works

```
HTTP Request → Controller → MediatR → [ValidationBehavior] → Command Handler
                                           ↓
                                    Runs FluentValidation
                                           ↓
                                    If invalid → throws ValidationException
                                    If valid → continues to handler
```

### The Validation Pipeline Behavior
```csharp
public class ValidationBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // 1. Run ALL registered validators for this request type
        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(result => result.Errors)
            .Where(f => f != null)
            .ToList();

        // 2. If any fail, throw — the ExceptionMiddleware catches this
        if (failures.Count > 0)
            throw new ValidationException(failures);

        // 3. If all pass, continue to the actual handler
        return await next();
    }
}
```

### 🏗️ Real-World: NextEvent's Registration
```csharp
// From NextEvent's ApplicationServiceExtensions.cs
services.AddMediatR(x =>
{
    x.RegisterServicesFromAssemblyContaining<LoginCommandHandler>();

    // This single line bridges FluentValidation and MediatR:
    x.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

**The beauty:** Your controllers become incredibly thin — they just forward commands to MediatR. Validation happens automatically via the pipeline. The `ExceptionMiddleware` catches `ValidationException` and returns a clean `400 Bad Request`.

```csharp
// A controller in a CQRS architecture — no validation logic at all!
[HttpPost]
public async Task<IActionResult> Register(RegisterCommand command)
{
    var result = await _mediator.Send(command);
    return Ok(result);
    // If command is invalid, ValidationBehavior throws ValidationException
    // ExceptionMiddleware catches it and returns 400 Bad Request
    // This code NEVER executes if validation fails
}
```

## 7. Best Practices Summary

| Practice | Why |
|---|---|
| **Always use `[ApiController]`** | Automatic validation, binding inference, Problem Details |
| **Use DTOs, not Domain Entities** | Don't expose internal models to the outside world |
| **Prefer FluentValidation over Data Annotations** | Testable, injectable, conditional rules |
| **Use a `ValidationBehavior` with MediatR** | Automatic, centralized, no manual checks |
| **Always validate on the server** | Client-side validation is a UX feature, not a security feature |
| **Use `[FromRoute]` for identifiers** | Clean URLs: `/api/users/42` |
| **Use `[FromQuery]` for filters/pagination** | Standard: `?page=2&size=10&sort=name` |
| **Use `[FromBody]` for create/update payloads** | POST/PUT with JSON |

---
**Key Takeaways:**
1. Model Binding automatically converts HTTP data into C# objects — you rarely need to parse anything manually.
2. `[ApiController]` enables automatic validation, returning 400 before your code runs.
3. FluentValidation is the enterprise standard — it separates validation from models and supports DI, conditionals, and async rules.
4. In CQRS architectures, validation runs in a MediatR pipeline behavior, keeping controllers razor-thin.

**Next up:** Module 7 — **Error Handling and Logging**. What happens when things go wrong? How do you log what happened and return safe, standardized error responses?
