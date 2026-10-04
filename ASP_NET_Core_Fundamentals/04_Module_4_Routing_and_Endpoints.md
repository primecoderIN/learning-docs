# Module 4: Routing and Endpoints

Routing is the **GPS system** of your web application. When an HTTP request arrives at your server saying `GET /api/events/42/tickets`, routing is the mechanism that looks at that URL, decodes it, and says: "Ah, this request should be handled by the `GetTickets` method of the `EventsController`, and the `id` parameter is `42`."

## 1. What is Routing?

### The Core Problem
Your ASP.NET Core application has dozens (or hundreds) of C# methods that handle different operations: creating users, fetching products, deleting orders. An HTTP request is just a URL string and an HTTP verb. **Something** has to map `/api/users/5` to a specific `GetUserById(int id)` method.

That "something" is the **Router**.

> **Analogy — The Post Office:**
> Think of routing like a post office. Letters (HTTP requests) arrive with an address (URL). The post office (router) reads the address, looks it up in its delivery registry, and dispatches the letter to the correct mailbox (C# method). If the address doesn't match any mailbox, the letter is returned with "Address Unknown" (404 Not Found).

### What Routing Does, Step by Step
1. **Receives** an incoming URL: `GET /api/products/42`
2. **Matches** it against all registered route templates: `"/api/products/{id:int}"`
3. **Extracts** parameters: `id = 42`
4. **Selects** the endpoint (the C# method to execute)
5. **Binds** the extracted values to the method's parameters

## 2. Endpoint Routing — The Modern Architecture

Modern ASP.NET Core (3.0+) uses a system called **Endpoint Routing**. It is critically important to understand that routing is split into **two separate steps**:

### Step 1: Route Matching (UseRouting)
The routing middleware examines the URL and determines *which* endpoint matches. **It does NOT execute the endpoint yet.** It simply stores the matched endpoint in the `HttpContext` for later.

### Step 2: Endpoint Execution (MapControllers / MapGet)
After other middleware runs (like Authorization), the endpoint execution middleware actually *invokes* the matched C# method.

### Why Split Them?
Because **middleware between routing and execution** can make decisions based on *which* endpoint was matched. This is revolutionary.

```csharp
app.UseRouting();           // "This request matches the GetSecretData endpoint"
app.UseAuthentication();    // "Who is the caller?"
app.UseAuthorization();     // "Does the caller have permission for GetSecretData?"
app.MapControllers();       // "Permission granted — execute GetSecretData"
```

Without this split, the Authorization middleware wouldn't know *which* endpoint needs checking until it's already executed — which is too late.

## 3. Controller Routing (Attribute Routing)

For larger APIs, Controllers are the standard approach. You annotate your classes and methods with route attributes.

### Basic Controller Setup
```csharp
[ApiController]                    // Enables API behaviors (auto 400, auto binding)
[Route("api/[controller]")]       // Base route: /api/users (derived from class name)
public class UsersController : ControllerBase
{
    private readonly IUserRepository _repository;

    public UsersController(IUserRepository repository)
    {
        _repository = repository;
    }

    // GET /api/users
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var users = await _repository.GetAllAsync();
        return Ok(users);
    }

    // GET /api/users/5
    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var user = await _repository.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }

    // POST /api/users
    [HttpPost]
    public async Task<IActionResult> Create(CreateUserDto dto)
    {
        var user = await _repository.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // PUT /api/users/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, UpdateUserDto dto)
    {
        await _repository.UpdateAsync(id, dto);
        return NoContent();
    }

    // DELETE /api/users/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _repository.DeleteAsync(id);
        return NoContent();
    }
}
```

### The `[ApiController]` Attribute — What It Actually Does
This single attribute enables **5 critical behaviors** that beginners often don't know about:

| Behavior | What it does |
|---|---|
| **Automatic Model Validation** | If `ModelState` is invalid, automatically returns `400 Bad Request` before your code runs |
| **Binding Source Inference** | Complex types default to `[FromBody]`, simple types default to `[FromRoute]` or `[FromQuery]` |
| **Problem Details Responses** | Error responses use the standardized RFC 7807 format |
| **Attribute Routing Required** | Forces you to use `[Route]` instead of convention-based routing |
| **Multipart/form-data Inference** | File uploads with `IFormFile` are automatically inferred |

### `[controller]` Token — Dynamic Route Segments
The `[controller]` token in `[Route("api/[controller]")]` is automatically replaced with the controller's name (minus the "Controller" suffix).

```csharp
// Class: ProductsController → Route: /api/products
// Class: OrdersController  → Route: /api/orders
// Class: UsersController   → Route: /api/users
```

## 4. Route Parameters and Constraints

### Route Parameters (Extracting Data from URLs)
Use curly braces `{}` to define dynamic segments in routes:

```csharp
// Single parameter
[HttpGet("{id}")]                   // /api/users/42
public IActionResult GetById(int id)

// Multiple parameters
[HttpGet("{year}/{month}")]         // /api/reports/2026/10
public IActionResult GetReport(int year, int month)

// Optional parameter
[HttpGet("{id?}")]                  // /api/users OR /api/users/42
public IActionResult Get(int? id)

// Default value
[HttpGet("{page=1}")]               // /api/users → page=1; /api/users/3 → page=3
public IActionResult GetPaged(int page)
```

### Route Constraints (Type Safety in URLs)
Without constraints, `/api/users/abc` would match `{id}` and then crash when trying to parse "abc" as an integer. Constraints prevent the match entirely:

```csharp
[HttpGet("{id:int}")]               // Only matches integers: /api/users/42
[HttpGet("{slug:alpha}")]           // Only matches letters: /api/articles/hello-world
[HttpGet("{id:guid}")]              // Only matches GUIDs: /api/orders/a1b2c3d4-...
[HttpGet("{name:minlength(3)}")]    // Minimum 3 characters
[HttpGet("{age:range(18,99)}")]     // Integer between 18 and 99
[HttpGet("{email:regex(^.+@.+$)}")] // Matches a regex pattern
```

> **Best Practice:** Always use `{id:int}` instead of plain `{id}` for numeric parameters. Without the constraint, `/api/users/abc` returns a confusing 400 error. With it, you get a clean 404 — "this route doesn't exist" — which is correct.

## 5. Minimal API Routing

Minimal APIs define routes directly in `Program.cs` using extension methods on the `app` object. They're more concise and slightly faster (less overhead than Controller infrastructure).

### Basic CRUD
```csharp
var app = builder.Build();

app.MapGet("/api/products", async (IProductService service) =>
    Results.Ok(await service.GetAllAsync()));

app.MapGet("/api/products/{id:int}", async (int id, IProductService service) =>
    await service.GetByIdAsync(id) is { } product
        ? Results.Ok(product)
        : Results.NotFound());

app.MapPost("/api/products", async (CreateProductDto dto, IProductService service) =>
{
    var product = await service.CreateAsync(dto);
    return Results.Created($"/api/products/{product.Id}", product);
});

app.MapPut("/api/products/{id:int}", async (int id, UpdateProductDto dto, IProductService service) =>
{
    await service.UpdateAsync(id, dto);
    return Results.NoContent();
});

app.MapDelete("/api/products/{id:int}", async (int id, IProductService service) =>
{
    await service.DeleteAsync(id);
    return Results.NoContent();
});
```

### Route Groups (Organizing Minimal APIs)
When you have many Minimal API endpoints, `Program.cs` gets messy. Use **Route Groups** to organize:

```csharp
var products = app.MapGroup("/api/products");

products.MapGet("/", GetAll);
products.MapGet("/{id:int}", GetById);
products.MapPost("/", Create);

// You can apply middleware/filters to the entire group
products.RequireAuthorization();
```

## 6. HTTP Return Types and Status Codes

Understanding what to return from your endpoints is crucial for building proper REST APIs:

### Controller Return Types
```csharp
return Ok(data);                    // 200 — Success with data
return NoContent();                 // 204 — Success, no data to return
return Created(uri, data);          // 201 — Resource created successfully
return BadRequest(errors);          // 400 — Client sent invalid data
return Unauthorized();              // 401 — Not authenticated
return Forbid();                    // 403 — Authenticated but not authorized
return NotFound();                  // 404 — Resource doesn't exist
return Conflict(message);           // 409 — Business rule violation
```

### Minimal API Return Types
```csharp
return Results.Ok(data);
return Results.NoContent();
return Results.Created(uri, data);
return Results.BadRequest(errors);
return Results.NotFound();
return Results.Problem(detail, statusCode: 500);
```

## 7. Real-World Routing from Your Projects

### 🏗️ NextEvent: Lowercase URLs & Configurable CORS Origins
```csharp
// From NextEvent's ApiServiceExtensions.cs
services.AddRouting(options =>
{
    options.LowercaseUrls = true;  // /api/Users/5 → /api/users/5
});

// CORS origins loaded from configuration, not hardcoded
var allowedOrigins = configuration.GetSection("AllowedOrigins").Get<string[]>() ?? [];
services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials()
              .WithOrigins(allowedOrigins);
    });
});
```

**Why `LowercaseUrls = true`?** REST API conventions dictate lowercase URLs. Without this, a controller named `UsersController` generates routes like `/api/Users` (uppercase U), which looks unprofessional and can cause caching issues.

### 🏗️ Normora: SignalR Hub Routing + BFF Security
Normora routes real-time WebSocket hubs alongside REST controllers, applying the same security:

```csharp
// From Normora's Program.cs
app.MapControllers().AsBffApiEndpoint();                                    // REST APIs
app.MapBffManagementEndpoints();                                           // /bff/login, /bff/logout
app.MapHub<DocumentHub>("/hubs/documents").AsBffApiEndpoint();             // WebSocket
app.MapHub<NotificationHub>("/hubs/notifications").AsBffApiEndpoint();     // WebSocket
```

**Key insight:** `.AsBffApiEndpoint()` enforces anti-forgery (CSRF) protection on every endpoint. This means a malicious website can't make requests to your API on behalf of a logged-in user, whether the endpoint is REST or WebSocket.

### 🏗️ NextEvent: JSON Serialization Configuration
```csharp
// From NextEvent's ApiServiceExtensions.cs
services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.PropertyNameCaseInsensitive = true;
    });
```

*   **`CamelCase`:** C# properties (`FirstName`) are serialized as JSON (`firstName`). This follows JavaScript/API conventions.
*   **`PropertyNameCaseInsensitive`:** When receiving JSON from clients, `firstname`, `FirstName`, and `FIRSTNAME` all bind correctly. This prevents frustrating 400 errors from case mismatches.

## 8. Best Practices Summary

| Practice | Why |
|---|---|
| **Use `LowercaseUrls = true`** | Professional REST API convention |
| **Always use route constraints (`{id:int}`)** | Return clean 404s instead of confusing 400s |
| **Use `[ApiController]` on every API controller** | Enables auto-validation, binding inference, Problem Details |
| **Use `CamelCase` JSON** | JavaScript convention, frontend developers expect it |
| **Group Minimal API endpoints with `MapGroup`** | Keeps `Program.cs` organized |
| **Load CORS origins from configuration** | Same binary, different environments |
| **Use `CreatedAtAction` for POST endpoints** | Returns proper `201 Created` with a `Location` header |

---
**Key Takeaways:**
1. Routing maps URLs to C# methods. Endpoint Routing splits matching from execution so authorization can run in between.
2. Controllers give you structure and features. Minimal APIs give you speed and simplicity.
3. Route constraints (`{id:int}`) prevent type mismatches from reaching your code.
4. Always configure `LowercaseUrls`, `CamelCase` JSON, and route constraints from day one.

**Next up:** Module 5 — **Configuration and the Options Pattern**. How do you manage database passwords, API keys, and environment-specific settings without hardcoding them?
