# Chapter 16: Enterprise Architecture Patterns

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Contrast Monoliths, Monorepos (Nx), and Micro Frontends.
* Understand the Domain-Driven Design (DDD) philosophy for structuring large applications.
* Implement a Clean Architecture layer separation (UI vs. State vs. Data Access).
* Understand Module Federation and when to use Micro Frontends.

---

## 2. Introduction: Scaling Beyond the Basics

Building a 10-page application is easy. Building a 500-page enterprise platform maintained by 50 different developers across 10 different teams is an entirely different engineering discipline.

Without a strict architectural strategy, an enterprise codebase quickly devolves into a "Big Ball of Mud." Components become tightly coupled to specific API endpoints, the `shared/` folder becomes a dumping ground for highly specific business logic, and deploying a change to the User Profile breaks the Admin Dashboard.

In Part V of this book, we elevate our perspective from *Writing Code* to *Architecting Systems*.

---

## 3. The Enterprise Workspace: Nx Monorepo

As briefly discussed in Chapter 2, enterprise applications rarely consist of a single UI. 
Our EV Charging Platform consists of:
1. `driver-app` (Mobile-first PWA for drivers to find chargers).
2. `operator-portal` (Dashboard for station owners to set pricing).
3. `admin-portal` (Super-admin dashboard to manage the whole system).

### Why a Monorepo?
If we put these in 3 separate Git repositories (Polyrepo), sharing the `ButtonComponent` or the `AuthService` requires publishing them to a private NPM registry. This causes versioning nightmares.

An **Nx Monorepo** stores all 3 applications in a single Git repository. 

```text
ev-platform-monorepo/
├── apps/
│   ├── driver-app/
│   ├── operator-portal/
│   └── admin-portal/
├── libs/
│   ├── shared-ui/       (Buttons, Modals, Badges)
│   ├── shared-auth/     (Login logic, JWT Interceptor)
│   ├── domain-chargers/ (Charger Interfaces, State Store, API Services)
│   └── domain-billing/  (Invoices, Stripe Integration)
```

### The Magic of Nx: Computation Caching
The historic problem with Monorepos was CI/CD time. If you change a button, do you have to run tests and builds for all 3 apps?

Nx solves this using a Dependency Graph. When you commit a change to `shared-auth`, Nx statically analyzes the graph and determines that only the `operator-portal` and `admin-portal` use that library. It skips building the `driver-app` entirely, retrieving its previous build output from a remote cache.

---

## 4. Domain-Driven Design (DDD) in Angular

DDD is a software engineering philosophy that says the structure of your code should match the structure of the business domain.

### The Bounded Context
In a massive application, a "Charger" means different things to different people.
* To the Driver, a Charger has a Location and a Current Price.
* To the Admin, a Charger has an IP Address, Firmware Version, and Maintenance Schedule.

Instead of creating one massive `Charger` interface and one massive `ChargerService`, we create Bounded Contexts.

* `libs/driver-domain/` gets its own `Charger` interface tailored to the driver.
* `libs/admin-domain/` gets its own `Charger` interface tailored to the admin.

### Access Rules
Using tools like Nx, we can enforce strict architectural linting rules:
* The `driver-app` is strictly forbidden from importing anything from `libs/admin-domain/`.
* If a developer tries to import it, the compiler throws an error. This prevents domains from bleeding into each other.

---

## 5. Clean Architecture in Angular

Within a specific domain (e.g., `libs/domain-chargers`), we must separate our concerns into Layers.

### 1. The Presentation Layer (UI Components)
* **What it does:** Renders HTML, listens for clicks.
* **What it knows:** Only knows about the State Layer. It injects the `SignalStore` or a `Facade` service.
* **What it DOES NOT know:** It has zero idea that `HttpClient` exists, or what the API URL is.

### 2. The State Layer (Store / Facade)
* **What it does:** Holds the current state in Signals. Dispatches actions.
* **What it knows:** Knows about the Data Access Layer.

### 3. The Data Access Layer (Services)
* **What it does:** Injects `HttpClient`. Maps raw JSON DTOs from the backend into clean TypeScript domain models.
* **What it knows:** Knows the API URLs and the shape of the backend data.

### Why this matters
If the backend team decides to switch from REST to GraphQL, or changes the shape of the JSON payload, you **only** modify the Data Access Layer. The State Layer and the UI Components remain completely untouched. This is the definition of highly decoupled code.

---

## 6. Micro Frontends and Module Federation

What if your organization is so large that the Driver team and the Admin team want to deploy their applications completely independently, without waiting for a unified Monorepo CI/CD pipeline?

This is where **Micro Frontends** using **Webpack Module Federation** (or the Native Federation equivalent in Esbuild) come in.

### The Architecture
1. **The Shell App:** A lightweight Angular application that provides the Header and Sidebar.
2. **The Remote Apps:** Independent Angular applications (e.g., `BillingApp`, `ChargerApp`) that are deployed to completely different servers.

When a user clicks "Billing" in the Shell App, the Shell dynamically downloads the compiled JavaScript from the Remote App's server and mounts it into the UI as if it were a local lazy-loaded module.

### When to use Micro Frontends
> **Architect Warning:** Micro Frontends are the most complex architectural pattern in frontend engineering. They introduce massive challenges with state sharing, CSS collisions, and version mismatches (what if the Shell uses Angular 17 but the Remote uses Angular 18?). 
> 
> **Rule of Thumb:** Do not use Micro Frontends unless you have over 100 frontend developers and organizational independence is more critical than codebase simplicity. For 95% of enterprise projects, an Nx Monorepo is vastly superior.

---

## 7. Common Mistakes

### Senior Mistakes
* **The "Shared" Dumping Ground:** Creating a massive `libs/shared` folder and putting domain-specific components (like a `ChargerStatusBadge`) inside it. The `shared` folder should only contain absolutely domain-agnostic UI primitives (Buttons, Inputs, Modals). Domain-specific components belong in their respective Domain libraries.

### Architect Mistakes
* **Premature Micro Frontends:** Choosing a Micro Frontend architecture for a team of 10 developers because "it's the new hot trend." This will paralyze the team with DevOps complexity, CI/CD pipeline nightmares, and difficult local development environments.

---

## 8. Interview Questions

### Senior
**Q: Explain the concept of a "Facade Pattern" in Angular architecture.**
> A: A Facade is an `@Injectable` service that acts as the single point of contact between a UI Component and the State/Data Access layers. Instead of a component injecting the `HttpClient`, the `Store`, and the `AuthService`, it simply injects the `ChargerFacade`. The Facade exposes clean Signals for the UI to read, and exposes methods for the UI to call. This abstracts away the complexity of state management (like NgRx) from the UI developer.

### Architect
**Q: What is the primary difference between lazy-loading a module in a standard Angular app versus loading a Remote app via Module Federation?**
> A: In a standard Angular app, the lazy-loaded chunk is generated during the exact same build process as the main app. They are guaranteed to use the exact same version of Angular, RxJS, and shared services. In Module Federation, the Remote app is built and deployed completely separately, often by a different team. It is fetched over the network at runtime from a different URL. This requires strict runtime contracts to ensure shared dependencies (like the Angular core) don't clash.

---

## 9. Summary
In this chapter, we elevated our perspective to Enterprise Architecture. We explored how Nx Monorepos solve code sharing at scale, how DDD prevents the "Big Ball of Mud," how Clean Architecture separates the UI from the API, and the severe trade-offs of Micro Frontends.

In Chapter 17, we will dive into **Multi-Tenant SaaS Patterns**, exploring how to build an application that dynamically changes its branding, features, and authentication based on which enterprise customer is logging in.
