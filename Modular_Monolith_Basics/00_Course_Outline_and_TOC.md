# Modular Monolith Architecture & Basics Course

**Target Audience:** Beginner to Intermediate C# and .NET Developers looking to design robust, clean, scalable applications without the operational nightmare of premature microservices.  
**Technology Stack:** .NET 8 / 9 / 10, C#, ASP.NET Core, EF Core, MediatR, MassTransit, PostgreSQL / SQL Server, NetArchTest.  
**Teaching Approach:** Every concept is explained with **"What, How, Why, and Where"** — combining crystal-clear analogies, architectural diagrams, concrete C# code implementations, and battle-tested production blueprints from **NextEvent** and **Normora**.  
**Goal:** Master the Modular Monolith architecture from ground zero to production deployment: domain boundaries, independent data isolation, cross-module communication, automated architectural testing, and seamless microservice extraction.

---

## Course Outline & Table of Contents

### Module 1: Foundations of Modular Monolith Architecture
*   **The Architecture Spectrum:** Monolith ("Big Ball of Mud") vs. Microservices vs. Modular Monolith ("The Citadel").
*   **The Microservices Fallacy:** Latency overhead, distributed transactions, debugging complexity, and DevOps taxation.
*   **Domain-Driven Design (DDD) Core Building Blocks:** Bounded Contexts, Ubiquitous Language, Aggregates, and Subdomains.
*   **High Cohesion and Loose Coupling:** What they mean in code and why they dictate architectural success.
*   **When to Choose a Modular Monolith:** Decision matrix and business/team criteria.

### Module 2: Solution Architecture & Project Structure
*   **Solution Anatomy:** The Composition Root (`API`), Cross-Cutting Layer (`Shared`), and Bounded Contexts (`Modules/`).
*   **Internal Module Organization:** Clean Architecture (Domain, Application, Infrastructure) vs. Vertical Slice Architecture.
*   **Comparing Real-World Layouts:** Deep dive into the folder structures of **NextEvent** and **Normora**.
*   **Encapsulation & Visibility:** Using C# `internal`, `public`, `InternalsVisibleTo`, and contract assemblies.
*   **Modular Monolith vs. Multi-Repo:** Why a single mono-repo/solution accelerates development velocity.

### Module 3: Database Isolation & Persistence Strategies
*   **The Golden Rule:** Why modules must NEVER directly query or join another module's database tables.
*   **Database Partitioning Strategies:**
    *   Single Database with Table Prefixes.
    *   Single Database with Schema Separation (`identity.*`, `events.*`, `tenants.*`).
    *   Multiple Physical Databases.
*   **Independent EF Core DbContexts:** Designing, configuring, and isolating a `DbContext` per module.
*   **Managing Migrations at Scale:** Independent migration histories, CLI workflows, and automatic startup migrations.
*   **Referencing Data Across Modules:** Referencing by Identity (`Guid eventId`) vs. prohibited EF navigation properties.

### Module 4: Synchronous Cross-Module Communication
*   **The Module Contract Pattern:** Exposing public contracts/interfaces in `Contracts` / `Shared` assemblies.
*   **Public Interfaces vs. Private Implementations:** How Module A requests data from Module B via DI.
*   **DTOs and Read Models:** Why domain entities must never leak outside their module boundary.
*   **Performance Benefits of In-Process Calls:** Zero network serialization, nanosecond function execution, and call-stack clarity.
*   **The Anti-Corruption Layer (ACL):** Translating foreign module concepts into local domain models.

### Module 5: In-Process Asynchronous Communication & Domain Events
*   **Why Asynchronous Messaging?** Decoupling modules in time and preventing cascading failures.
*   **Publish-Subscribe In-Memory:** Publishing Domain and Integration Events with MediatR (`INotification` and `INotificationHandler`).
*   **Handling Side Effects Gracefully:** Keeping write transactions slim and offloading notifications.
*   **The Dual-Write Problem:** What happens when the database commits but the event handler fails?
*   **The Transactional Outbox Pattern:** Guaranteeing reliable message delivery in-memory and in-database.

### Module 6: Distributed Messaging & Background Processing
*   **Stepping Beyond In-Process:** When and why to introduce external brokers (RabbitMQ / Azure Service Bus / Kafka).
*   **MassTransit in a Modular Monolith:** Configuring publish/subscribe for cross-module integration.
*   **Background Jobs & Schedulers:** Integrating Hangfire, Quartz.NET, and .NET `BackgroundService` without violating module isolation.
*   **Idempotency & Resilience:** Handling duplicate messages, retries, and dead-letter queues.

### Module 7: The Composition Root & Dependency Injection
*   **The Role of the Host Application:** Keeping `Program.cs` slim, clean, and agnostic of module internals.
*   **Modular Service Registration:** Creating fluent extension methods (`AddIdentityModule()`, `AddDocumentsModule()`).
*   **Configuration & Options Isolation:** Scoped `IConfiguration` sections and strongly-typed Options patterns per module.
*   **Endpoint Discovery & Mapping:** Registering Minimal APIs or Controllers cleanly from within each module.
*   **Assembly Scanning & Avoiding Circular Dependencies:** Keeping dependency graphs strictly unidirectional.

### Module 8: Shared Cross-Cutting Concerns
*   **The `Shared` Kernel:** What belongs in the shared library and what MUST remain private.
*   **Authentication & Authorization:** Contextual claims, `ICurrentUserService`, and multi-module permissions.
*   **Multi-Tenancy in a Modular Monolith:** **Normora** case study with `ITenantContext`, tenant resolution, and data partitioning.
*   **Standardized Error Handling:** RFC 7807 Problem Details and domain-to-HTTP exception mapping.
*   **Observability & Diagnostics:** Structured logging with Serilog, Activity IDs, and distributed tracing with OpenTelemetry.

### Module 9: Architecture Governance & Automated Architecture Tests
*   **Architectural Drift:** How well-intentioned teams accidentally introduce spaghetti dependencies.
*   **Automated Governance with NetArchTest.Rules:** Writing unit tests that assert architectural boundaries.
*   **Key Architecture Assertions:**
    *   Module Domain layers must not reference Infrastructure or API.
    *   Module A cannot reference internal classes or repositories of Module B.
    *   DbContexts must only reference entities from their own bounded context.
*   **Enforcing Rules in CI/CD:** Failing the build when an architectural boundary is violated.

### Module 10: Real-World Case Studies & Production Lessons
*   **Case Study 1: NextEvent:** Event management, tickets, organization memberships, and AI integration.
*   **Case Study 2: Normora:** Multi-tenant workspace, Keycloak auth, document management, and conversation streaming.
*   **Common Anti-Patterns to Avoid:** The "Shared Database Trap", the "God Module", and the "Fake Module" illusion.
*   **Refactoring Legacy Monoliths:** Practical strangler patterns to extract modules from legacy codebases.

### Module 11: The Microservices Extraction Playbook
*   **When Should You Actually Extract a Module?** Explicit criteria: independent team autonomy, divergent scaling profiles, security isolation.
*   **When You Should NOT Extract:** Organizational traps and premature optimization.
*   **Step-by-Step Extraction Guide:**
    *   Step 1: Replace in-memory module contracts with HTTP/gRPC or message consumers.
    *   Step 2: Move outbox dispatch from in-process handlers to message bus topics.
    *   Step 3: Extract the module's database schema into its own physical database server.
    *   Step 4: Deploy as an independent container/service.
*   **Why Starting with a Modular Monolith Guarantees Microservice Success.**
