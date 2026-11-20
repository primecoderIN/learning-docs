# JWT Fundamentals: From Scratch to Security Architect

**Target Audience:** Beginner to Senior C# / Full-Stack Developers who want to design, validate, and defend token-based security in production systems
**Technology Stack:** .NET 10, ASP.NET Core (`JwtBearer`, Authorization Policies), Duende BFF, Keycloak 24 (OIDC / OAuth 2.0), Angular 22, Redis, PostgreSQL, Python 3 / FastAPI (PyJWT, python-jose)
**Goal:** A step-by-step journey from "what is a token?" to architect-level decisions about signing algorithms, key rotation, token lifecycles, OAuth 2.0 / OpenID Connect flows, the BFF pattern, multi-tenant authorization, attack mitigation, and sender-constrained tokens.

> **Running Example — Normora:** Every module uses **Normora**, an enterprise multi-tenant knowledge management and conversational-AI platform: a .NET 10 Modular Monolith (`Tenants`, `Documents`, `Conversations`) with an Angular SPA, Keycloak OIDC, a Duende BFF host (`Normora.Api`) that keeps tokens out of the browser, `TenantResolutionMiddleware`, MediatR cross-module contracts, CQRS folder-per-operation, FluentValidation, OpenTelemetry, and the BOLA → `404` / BFLA → `403` rules with `ApiMessages` constants (no magic strings).

> **About this course:** The former `JWT_Fundamentals` and `JWT_Handbook` tracks are merged into this one course. Modules 1–6 are the "Handbook" (what a JWT *is* and how it is verified); Modules 7–10 are the "Integration" track (how tokens flow through OAuth 2.0, Keycloak, the BFF and ASP.NET Core); Modules 11–12 are the "Architect" track (polyglot validation, attacks, sender-constrained tokens, and production checklists).

## Course Outline

### Module 1: Tokens and Sessions
*File: `01_Module_1_Tokens_and_Sessions.md`*
*   **What is a Token?** Credentials vs. tokens, bearer semantics ("whoever holds it, uses it"), and reference (opaque) tokens vs. self-contained (value) tokens.
*   **Authentication vs. Authorization:** Identity proof vs. permission decisions, and where each happens in a request.
*   **Server-Side Sessions:** Session IDs, session stores, cookies, and why sessions are trivially revocable.
*   **Stateless Tokens:** What "stateless" really means, horizontal scaling, and the hidden cost of irrevocability.
*   **Sessions vs. Tokens Decision Matrix:** Latency, revocation, scaling, mobile/SPA/service clients, and blast radius.
*   **The Normora Hybrid:** Why Normora's browser holds only an encrypted `HttpOnly`, `SameSite=Strict` session cookie while Keycloak-issued JWTs live server-side in the Duende BFF session store.

### Module 2: Base64URL and JWT Anatomy
*File: `02_Module_2_Base64URL_and_JWT_Anatomy.md`*
*   **Encoding Is Not Encryption:** Base64 vs. Base64URL (`+/` → `-_`, padding removal) and why anyone can read a JWT payload.
*   **Decoding by Hand:** Splitting on `.` and decoding header/payload in C# (`Base64UrlEncoder`, `WebEncoders`) and Python (`base64.urlsafe_b64decode`).
*   **The Header (JOSE Header):** `alg`, `typ`, `kid`, `cty`, and why the header is attacker-controlled input.
*   **The Payload (Claims Set):** JSON claims, NumericDate timestamps, and size limits (cookies, headers, proxies).
*   **The Signature:** Signing input = `base64url(header) + "." + base64url(payload)`, and what integrity does and does not guarantee.
*   **Compact Serialization:** 3-part JWS vs. 5-part JWE at a glance, plus a safe local-decoding workflow (why pasting production tokens into online debuggers is a leak).

### Module 3: Claims — Registered, Public, and Private
*File: `03_Module_3_Claims_Registered_Public_and_Private.md`*
*   **Registered Claims (RFC 7519):** `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti` — purpose and semantics of each.
*   **Public Claims:** The IANA JWT Claims Registry, OIDC standard claims (`email`, `name`, `preferred_username`), and collision-resistant names.
*   **Private Claims:** Application-specific claims such as `tenant_id`, `roles`, `scope`, and `department`, and naming conventions.
*   **Anatomy of a Keycloak Token:** `realm_access`, `resource_access`, `azp`, `sid`, `acr`, and protocol mappers.
*   **Claim Design Principles:** Minimal payloads, no PII or secrets, stable identifiers (`sub` → Normora's `ExternalUserId`), and avoiding stale authorization data.
*   **Claims in Token vs. Claims on Server:** Why Normora resolves memberships and RBAC through `TenantResolutionMiddleware` + Redis instead of trusting a long-lived `roles` claim.

### Module 4: JWS vs. JWE and Signing Algorithms
*File: `04_Module_4_JWS_JWE_and_Signing_Algorithms.md`*
*   **The JOSE Family:** JWS, JWE, JWK, JWA, and JWT — how the RFCs (7515–7519) fit together.
*   **JWS (Signed Tokens):** Integrity and authenticity without confidentiality; JSON vs. compact serialization.
*   **JWE (Encrypted Tokens):** Key management (`alg`: `RSA-OAEP`, `ECDH-ES`, `dir`) vs. content encryption (`enc`: `A256GCM`), and nested (signed-then-encrypted) JWTs.
*   **HMAC (HS256/384/512):** Shared secrets, secret length/entropy, and why symmetric keys break in multi-party systems.
*   **RSA (RS256, PS256):** Public/private key pairs, PKCS#1 v1.5 vs. PSS, key sizes, and verification-heavy performance.
*   **ECDSA (ES256) and EdDSA (Ed25519):** Smaller keys and signatures, nonce pitfalls, and library/IdP support (including Keycloak and .NET).
*   **Algorithm Selection Guide:** A decision matrix for Normora's IdP tokens, internal service tokens, and when (if ever) to reach for JWE.

### Module 5: Key Management, JWKS, and Key Rotation
*File: `05_Module_5_Key_Management_JWKS_and_Key_Rotation.md`*
*   **Key Lifecycle:** Generation, storage (Key Vault / KMS / HSM / Kubernetes secrets), usage, rotation, retirement, and destruction.
*   **JSON Web Keys (JWK):** `kty`, `use`, `alg`, `kid`, `n`/`e`, `crv`/`x`/`y`, and private-key fields that must never be published.
*   **JWKS Endpoints:** OIDC discovery (`/.well-known/openid-configuration`) → `jwks_uri`, and Keycloak's realm certificate endpoint.
*   **The `kid` Header:** Selecting the right verification key, and handling unknown `kid` values safely.
*   **Key Rotation Without Downtime:** Overlap windows, pre-publishing next keys, Keycloak active/passive keys, and in-flight token survival.
*   **Caching and Refresh:** `ConfigurationManager` refresh intervals in .NET, refresh-on-unknown-`kid` with rate limiting, and outage behavior.
*   **Compromise Response:** Emergency rotation runbook, forced re-authentication, and auditing.

### Module 6: Token Validation Rules
*File: `06_Module_6_Token_Validation_Rules.md`*
*   **The Validation Pipeline:** Parse → algorithm allow-list → key resolution → signature → claims, and why order matters.
*   **Issuer (`iss`) and Audience (`aud`):** Exact matching, multi-audience tokens, Keycloak audience mappers, and confused-deputy risks.
*   **Time Claims (`exp`, `nbf`, `iat`):** NumericDate math, clock skew (`ClockSkew` defaults of 5 minutes and why to tighten them), and NTP.
*   **Type and Algorithm Pinning:** `typ` (`at+jwt` per RFC 9068), `ValidAlgorithms`, and rejecting `none`.
*   **`TokenValidationParameters` Deep Dive:** Every important flag in `Microsoft.IdentityModel.Tokens`, `JsonWebTokenHandler` vs. legacy `JwtSecurityTokenHandler`, and claim-type mapping (`MapInboundClaims`).
*   **Failure Handling:** `401` vs. `403`, `WWW-Authenticate` error codes, OpenTelemetry logging of failures without leaking tokens, and `ApiMessages`-based responses.

### Module 7: Access, Refresh, and ID Tokens — Rotation and Revocation
*File: `07_Module_7_Access_Refresh_and_ID_Tokens_Rotation_and_Revocation.md`*
*   **Three Tokens, Three Jobs:** Access tokens (for APIs), ID tokens (for clients, never for APIs), and refresh tokens (for obtaining new access tokens).
*   **Lifetime Design:** Short-lived access tokens, sliding vs. absolute refresh lifetimes, and Keycloak SSO session/idle timeouts.
*   **Refresh Token Rotation:** One-time-use refresh tokens and token families.
*   **Reuse Detection:** Detecting replayed refresh tokens, revoking the entire family, and Keycloak's "Revoke Refresh Token" setting.
*   **Revocation Strategies:** Short TTL, `jti` denylists in Redis, per-user/tenant session versions, token introspection (RFC 7662), and revocation endpoints (RFC 7009).
*   **Logout Done Right:** Front-channel vs. back-channel logout, and how Duende BFF server-side sessions make Normora's logout and membership-removal effective immediately.

### Module 8: OAuth 2.0 and OpenID Connect with Keycloak
*File: `08_Module_8_OAuth2_and_OpenID_Connect_with_Keycloak.md`*
*   **OAuth 2.0 Roles:** Resource owner, client, authorization server, and resource server — mapped onto Normora's components.
*   **Authorization Code + PKCE:** `code_verifier` / `code_challenge`, `state`, `nonce`, redirect URI validation, and why the Implicit and Password grants are deprecated (OAuth 2.1).
*   **Client Credentials:** Machine-to-machine tokens for workers and services, scopes, and confidential client authentication (secret vs. `private_key_jwt`).
*   **OpenID Connect:** ID tokens, the UserInfo endpoint, discovery documents, scopes (`openid profile email`), and `acr`/`amr` step-up.
*   **Keycloak in Practice:** Realms, confidential vs. public clients, client scopes, protocol mappers (adding `tenant_id`/audience), roles, and theming (`keycloak-theme/normora`).
*   **Advanced Grants:** Token Exchange (RFC 8693) and Device Authorization at a glance, plus Pushed Authorization Requests (PAR).

### Module 9: The BFF Pattern with Duende
*File: `09_Module_9_The_BFF_Pattern_with_Duende.md`*
*   **Why Tokens Don't Belong in the Browser:** XSS exfiltration, `localStorage`/`sessionStorage` risks, and the IETF "OAuth 2.0 for Browser-Based Apps" guidance.
*   **The BFF Architecture:** Confidential client on the server, cookie session to the SPA, and server-side token storage.
*   **Duende BFF in `Normora.Api`:** `/bff/login`, `/bff/user`, `/bff/logout`, server-side sessions (`SessionDatabaseMigrator`), and `AsBffApiEndpoint()` for embedded module controllers.
*   **Cookie Hardening:** `HttpOnly`, `Secure`, `SameSite=Strict`, `__Host-` prefixes, and data protection key management.
*   **Anti-CSRF:** The `X-CSRF: 1` header requirement, why it works, and why skip-antiforgery is reserved for externally signed callbacks (e.g. PayU) only.
*   **Token Management & Real-Time Channels:** Automatic refresh, remote API proxying, and authenticating SignalR / SSE / AG-UI streams with the BFF cookie.
*   **The Angular Side:** HTTP interceptors, session-expiry handling with `ApiMessages`, and Signals-based auth state without ever touching a token.

### Module 10: ASP.NET Core JwtBearer, Authorization Policies, and Multi-Tenant Claims
*File: `10_Module_10_ASP_NET_Core_JwtBearer_Policies_and_Multi_Tenant_Claims.md`*
*   **When Normora Accepts Bearer Tokens:** Cookie vs. bearer schemes, policy schemes (`ForwardDefaultSelector`), and machine callers such as external MCP clients on `/api/mcp`.
*   **Configuring `AddJwtBearer`:** Authority, audience, metadata, `TokenValidationParameters`, `JwtBearerEvents`, and typed options with `ValidateOnStart`.
*   **Claims Transformation:** `IClaimsTransformation` for flattening Keycloak `realm_access` roles and mapping `sub` → `ExternalUserId`.
*   **Authorization Policies:** Roles vs. claims vs. requirement/handler policies, `IAuthorizationHandler`, resource-based authorization, and fallback policies.
*   **Multi-Tenant Claims:** `tenant_id` in the token vs. tenant resolved from the route/slug, cross-checking both, and EF Core global query filters bound to `ITenantContext`.
*   **BOLA and BFLA:** Object-level checks throwing `BolaException` (→ `404`, `ApiMessages.NotFound`) and function-level checks throwing `BflaException` (→ `403`, `ApiMessages.Forbidden`) inside CQRS handlers and MediatR pipeline behaviors.
*   **Testing Auth:** Integration tests with test signing keys, `WebApplicationFactory`, and architecture tests (`Normora.Architecture.Tests`) that enforce `[Authorize]` coverage.

### Module 11: Validating JWTs in FastAPI
*File: `11_Module_11_Validating_JWTs_in_FastAPI.md`*
*   **The Polyglot Scenario:** A Python AI/analytics sidecar service that trusts the same Keycloak realm as Normora, called by `Normora.Worker` with client-credentials tokens.
*   **PyJWT with JWKS:** `PyJWKClient`, `jwt.decode()` with explicit `algorithms`, `audience`, `issuer`, `leeway`, and required claims.
*   **python-jose:** `jwt.decode`, JWKS key selection by `kid`, JWE support, and maintenance/security trade-offs vs. PyJWT (and `joserfc`/`authlib` alternatives).
*   **FastAPI Integration:** `HTTPBearer` / `OAuth2AuthorizationCodeBearer`, dependency injection with `Depends`, and scope/role dependencies.
*   **Multi-Tenant Enforcement in Python:** Propagating `tenant_id`, mirroring the 404 (BOLA) vs. 403 (BFLA) contract, and sharing message constants across stacks.
*   **Operational Concerns:** JWKS caching, async HTTP clients, key-rotation tests, and cross-language token interop tests against the .NET stack.

### Module 12: JWT Attacks, Sender-Constrained Tokens, and Production Checklists
*File: `12_Module_12_JWT_Attacks_Sender_Constrained_Tokens_and_Production_Checklists.md`*
*   **Signature Bypass Attacks:** `alg=none`, RS256 → HS256 key confusion, and weak HMAC secret brute-forcing.
*   **Header Injection Attacks:** `kid` injection (path traversal / SQL injection), `jku` and `x5u` abuse, and embedded `jwk` headers.
*   **Token Theft:** Token sidejacking, leakage via logs/URLs/referrers, XSS + `localStorage`, and CSRF against cookie-based sessions.
*   **Sender-Constrained Tokens:** DPoP (RFC 9449) proofs, `cnf`/`jkt` binding, and nonces; mTLS certificate-bound tokens (RFC 8705) with `x5t#S256`.
*   **Choosing a Defense:** BFF vs. DPoP vs. mTLS for browsers, mobile clients, and service-to-service traffic.
*   **Production Checklists:** Issuer/IdP, resource server, BFF/cookie, key management, logging/OpenTelemetry, and incident response checklists.
*   **Architect Capstone:** A full threat model and security review of Normora's token architecture, with an Architecture Decision Record (ADR) template.
