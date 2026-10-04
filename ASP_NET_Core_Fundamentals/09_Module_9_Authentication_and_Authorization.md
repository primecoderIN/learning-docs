# Module 9: Authentication and Authorization

If there is one area where developers guess, copy-paste from StackOverflow, and accidentally create massive security holes, it is Authentication and Authorization. ASP.NET Core has an incredibly powerful, flexible, and robust security system, but it requires understanding the core concepts: **Claims, Identities, and Principals**.

## 1. Authentication vs. Authorization

Let's clear this up immediately because people mix them up constantly:

*   **Authentication (AuthN):** "Who are you?" (Identity verification)
    *   *Example:* Checking a username and password, validating a JWT token, verifying a Google OAuth login.
*   **Authorization (AuthZ):** "Are you allowed to do this?" (Permissions)
    *   *Example:* Can you view this document? Are you an Administrator? Do you belong to this Tenant?

> **Analogy — The Nightclub:**
> **Authentication** is the bouncer at the front door checking your ID. He verifies you are who you say you are and that your ID is real.
> **Authorization** is the VIP room attendant. He knows who you are (you got past the front door), but he checks his list to see if you have the *permission* to enter the VIP area.

## 2. The Claims-Based Identity Model

In older frameworks, your identity was usually just a Username and a Role (e.g., `User.IsInRole("Admin")`). Modern ASP.NET Core uses **Claims-Based Identity**.

### What is a Claim?
A Claim is a **statement about the user**. It is a key-value pair.
*   "My Name is Alice"
*   "My Email is alice@example.com"
*   "My Date of Birth is 1990-01-01"
*   "My Role is Manager"
*   "My OrganizationId is 12345"

### The Hierarchy
1.  **Claim:** A single fact (`Type = "Email", Value = "alice@example.com"`).
2.  **ClaimsIdentity:** A collection of Claims representing a single identity (like a Driver's License or a Passport).
3.  **ClaimsPrincipal:** The actual `User` object in ASP.NET Core (`HttpContext.User`). A Principal can hold *multiple* Identities (e.g., you can have a Driver's License *and* a Passport).

When a user authenticates, ASP.NET Core builds a `ClaimsPrincipal` and attaches it to the `HttpContext.User`.

```csharp
// How to read claims in a controller or service:
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var email = HttpContext.User.FindFirst(ClaimTypes.Email)?.Value;
var orgId = HttpContext.User.FindFirst("OrganizationId")?.Value;
```

## 3. JWT Authentication (The NextEvent Approach)

**JSON Web Tokens (JWT)** are the standard for securing REST APIs. Instead of storing a session on the server (which requires memory and sticky sessions), the server gives the client a cryptographically signed token containing their Claims. The client sends this token in the `Authorization` header of every request.

### The Flow
1. User sends credentials (email/password) to `/api/login`.
2. Server verifies them in the database.
3. Server generates a JWT containing claims (UserId, Role, OrgId) and **signs it** with a secret key.
4. Server returns the JWT.
5. Client sends `Authorization: Bearer <token>` on future requests.
6. Server validates the signature. If valid, it extracts the claims and builds the `ClaimsPrincipal`.

### 🏗️ Real-World: NextEvent's JWT Configuration

NextEvent uses standard JWT Bearer authentication. Notice how it validates the token:

```csharp
// From NextEvent's IdentityServiceExtensions.cs
var tokenKey = config["TokenKey"]; // The secret key used to sign the token
var issuer  = config["Jwt:Issuer"]; // Who created the token? (NextEvent.API)
var audience = config["Jwt:Audience"]; // Who is it for? (NextEvent.Client)

services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt =>
    {
        opt.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true, // Must have a valid signature
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(tokenKey)),
            
            ValidateIssuer = true,           // Must come from our server
            ValidIssuer = issuer,
            
            ValidateAudience = true,         // Must be meant for our client app
            ValidAudience = audience,
            
            ValidateLifetime = true,         // Token must not be expired
            ClockSkew = TimeSpan.Zero        // Don't allow a 5-minute grace period
        };
    });
```

When a request arrives, the `app.UseAuthentication()` middleware reads the `Authorization` header, runs these validation rules, and if successful, populates `HttpContext.User`.

## 4. BFF and Cookie Authentication (The Normora Approach)

While JWTs are great for mobile apps and server-to-server communication, storing a JWT in a browser (like in `localStorage`) is a **massive security risk** due to Cross-Site Scripting (XSS). If an attacker runs malicious JavaScript on your site, they can steal the JWT and impersonate the user.

Normora solves this using the **BFF (Backend-For-Frontend)** pattern with **Secure Cookies**.

### The Flow
1. The Angular SPA redirects the user to Keycloak (an external Identity Provider).
2. User logs in to Keycloak.
3. Keycloak redirects back to the Normora API (the BFF) with a code.
4. The API exchanges the code for a JWT from Keycloak.
5. **CRITICAL:** The API *does not* give the JWT to the Angular app. Instead, it creates a strongly encrypted, HTTP-Only **Cookie** and sends that to the browser.
6. The browser automatically sends the Cookie with every API request.
7. The API reads the Cookie, decrypts the session, finds the JWT, and authenticates the user.

Because the Cookie is `HttpOnly`, malicious JavaScript **cannot read it**. XSS attacks are mitigated.

```csharp
// From Normora's Program.cs (BFF Setup)
builder.Services.AddBff()
    .AddEntityFrameworkServerSideSessions<SessionDbContext>(); // Store sessions in DB

builder.Services.AddAuthentication(options =>
{
    // Use Cookies as the default scheme for the browser
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "https://keycloak.normora.internal...";
    // ... configures OIDC flow with Keycloak
});

// Middleware pipeline order
app.UseAuthentication();
app.UseBff();  // Validates the session cookie and anti-forgery tokens
```

## 5. Authorization — Roles vs. Policies

Once `Authentication` knows *who* the user is, `Authorization` determines what they can *do*.

### The Old Way: Roles
Roles are simple string categories: "Admin", "User", "Manager".

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteUser(int id)
```

**The Problem with Roles:** What happens when the business says, "Managers can delete users, but only if the user is in their department"? A Role ("Manager") isn't enough information. Your controller becomes polluted with `if (user.DepartmentId == ...)` logic.

### The Modern Way: Policies
Policies are **rules** evaluated against the user's Claims. You define the rule once, and apply the policy name to endpoints.

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

If the business rule changes (e.g., "ActiveOrganizers must also have a verified email"), you update the Policy definition in one place. You don't touch the controllers.

## 6. Advanced Authorization: Resource-Based Authorization

Sometimes, a Policy isn't enough because the permission depends on the **data being requested**.

Example: A user can edit a document, but **only if they are the author**. You can't write a Policy for this because the Policy doesn't know *which document* is being edited until the database is queried.

### The Flow
1. Fetch the resource from the database.
2. Call the `IAuthorizationService` to evaluate a rule against that specific resource.

```csharp
// Example (How this looks in a MediatR handler)
public async Task<Unit> Handle(EditDocumentCommand request, CancellationToken ct)
{
    var document = await _db.Documents.FindAsync(request.Id);
    
    // Resource-based check!
    if (document.AuthorId != _currentUser.Id && !_currentUser.HasRole("Admin"))
    {
        throw new ForbiddenAccessException("You can only edit your own documents.");
    }
    
    // ... proceed with edit
}
```

## 7. Multi-Tenant Authorization (Normora)

Normora introduces a highly complex, real-world authorization requirement: **Multi-Tenancy**.
A user might be an "Admin" in Tenant A, but only a "Viewer" in Tenant B. Therefore, global roles (stored in the JWT) don't work.

### How Normora Solves It
Normora delays authorization until the `TenantResolutionMiddleware` figures out which Tenant context we are in.

1. **AuthN Middleware:** "This is user Alice."
2. **Tenant Middleware:** "The request is for Tenant A. I'll query the DB to find Alice's membership in Tenant A. Ah, she is an Admin here."
3. **Custom Authorize Attribute:** We create a `[RequireTenant("Admin")]` attribute that checks the `ITenantContext` (which the middleware just populated).

```csharp
// From Normora's TenantContext
tenantContext.SetContext(tenantId, membership.Role.ToString(), departments);
```

```csharp
// From a Normora Controller
[RequireTenant(TenantRoles.Admin)] // Custom attribute checking the Scoped ITenantContext
[HttpPost("documents")]
public IActionResult UploadDocument()
```

## 8. Abstractions: `ICurrentUserService`

**Never access `HttpContext.User` directly in your Application layer or Domain layer.** It tightly couples your business logic to the ASP.NET Core web framework. Instead, create an interface and inject it.

```csharp
// From NextEvent's CurrentUserService.cs
public class CurrentUserService(IHttpContextAccessor httpContextAccessor) : ICurrentUserService
{
    public string? GetCurrentUserId()
    {
        var user = httpContextAccessor.HttpContext?.User;
        if (user?.Identity?.IsAuthenticated != true) return null;

        // Uses the standard ClaimTypes.NameIdentifier to find the ID
        return user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }
    
    public bool HasRole(string role)
    {
        return httpContextAccessor.HttpContext?.User?.IsInRole(role) == true;
    }
}
```

Now, your MediatR handlers can inject `ICurrentUserService` and safely access user identity without knowing anything about HTTP, JWTs, or Cookies.

## 9. Best Practices Summary

| Practice | Why |
|---|---|
| **Use JWTs for Server-to-Server or Mobile apps** | Stateless, scalable, standard |
| **Use Cookies + BFF for Browser SPAs (React/Angular)** | Prevents XSS token theft |
| **Prefer Policies over Roles** | Business rules change; Policies decouple rules from controllers |
| **Never put sensitive data in a JWT** | JWTs are base64 encoded, not encrypted. Anyone can read them |
| **Abstract `HttpContext.User` behind `ICurrentUserService`** | Keeps business logic testable and framework-agnostic |
| **Always use `[Authorize]` globally** | Make endpoints secure by default. Use `[AllowAnonymous]` explicitly for login/registration |

---
**Key Takeaways:**
1. Authentication verifies identity (JWT/Cookies); Authorization verifies permissions (Policies/Roles).
2. The Claims-Based Identity model uses key-value pairs (Claims) to describe users.
3. JWTs are stateless but vulnerable to XSS in browsers. BFF with Cookies is the modern standard for SPAs (Normora).
4. Policies are superior to Roles because they encapsulate complex rules centrally.
5. Resource-based and Tenant-based authorization require evaluating rules *after* fetching data or resolving context.
