# Appendix 10: Enterprise Architecture Interview Questions

This section focuses entirely on high-level architecture: scaling Angular for massive teams, managing monorepos, micro-frontends, and cloud-native deployments. These are questions expected of a Staff or Principal UI Architect.

---

## Architect Level Questions

### 1. What is the difference between a Polyrepo and a Monorepo architecture?
**Answer:**
* **Polyrepo (Multi-repo):** Every application and shared library has its own Git repository. 
  * *Drawbacks:* Version hell. If the UI team updates the `Shared-Button` library to v2.0, they must publish it to npm, and then manually go to 15 different application repositories, update `package.json`, and fix breaking changes in isolation.
* **Monorepo:** All applications and shared libraries exist in a single Git repository. 
  * *Benefits:* Single source of truth. If you introduce a breaking change to the `Shared-Button` library, the CI/CD pipeline immediately fails for all 15 applications that use it. You fix the breaking changes atomically in a single Pull Request. Teams like Google and Meta operate almost entirely on massive Monorepos.

### 2. Why is Nx considered the industry standard for Angular Monorepos over the default Angular CLI workspaces?
**Answer:**
While the standard Angular CLI supports workspaces, it lacks enterprise scaling tools. Nx provides:
1. **Computation Caching:** If a library's source code hasn't changed since the last build, Nx skips compiling it and pulls the cached output, reducing a 30-minute build to 3 seconds.
2. **Distributed Task Execution (Nx Cloud):** It splits heavy E2E and build tasks across multiple CI agents in parallel.
3. **Module Boundaries:** You can enforce strict linting rules (e.g., "The `Billing` domain cannot import code from the `Shipping` domain"), preventing massive spaghetti-code interdependencies.

### 3. What are Micro-Frontends (MFE), and when should you use them?
**Answer:**
Micro-Frontends involve splitting a massive monolithic SPA into smaller, independently deployable applications that are stitched together at runtime (usually via Webpack Module Federation).
* **When to use them:** Only when you have massive, autonomous teams (e.g., 50+ UI engineers) that need entirely independent release cycles. The "Checkout Team" can deploy their MFE to production on Tuesday without communicating with the "Search Team".
* **When NOT to use them:** If you are a small team of 10 developers, MFEs introduce catastrophic complexity (version mismatches, duplicated dependencies, complex CI/CD routing). A standard Nx Monorepo is vastly superior for small-to-medium enterprise teams.

### 4. Explain how Webpack Module Federation enables Micro-Frontends.
**Answer:**
Module Federation allows a JavaScript application (the "Host" or "Shell") to dynamically load code from another independently deployed application (the "Remote") at runtime over the network.
Unlike lazy-loading a chunk from the *same* build, the Host app literally downloads a JavaScript file from a completely different server (e.g., `checkout.mycompany.com/remoteEntry.js`). 
Crucially, Module Federation manages shared dependencies. If the Host already loaded Angular core, it will *not* force the Remote to download Angular core again, preventing browser memory bloat.

### 5. In a Multi-Tenant SaaS application, how do you handle White-Labeling (custom branding per tenant) without destroying performance?
**Answer:**
**Anti-Pattern:** Building 100 different Angular apps with 100 different SCSS files (`tenant-A.scss`, `tenant-B.scss`). This destroys the CI/CD pipeline and increases build times exponentially.
**Modern Architecture:** Build exactly **one** Angular application. Use **CSS Custom Properties (CSS Variables)** for all colors, fonts, and border radii. 
When the user logs in, the API returns their tenant configuration. The Angular app dynamically injects a `<style>` tag into the DOM or updates the `document.documentElement.style.setProperty('--primary-color', apiColor)`. This changes the theme instantly at runtime with zero build overhead.

### 6. How do you implement strict Role-Based Access Control (RBAC) in the UI?
**Answer:**
RBAC requires a two-pronged approach:
1. **Routing:** Implement a `canMatch` Functional Guard. The guard checks the user's role against the required roles for the route. If they lack the role, the lazy-loaded chunk is never downloaded.
2. **DOM Visibility:** Create a custom structural directive (e.g., `*appHasRole="['ADMIN']"`). This directive injects the `TemplateRef` and `ViewContainerRef`. It subscribes to the user's active role. If the role matches, it calls `createEmbeddedView()`. If not, it calls `clear()`, completely removing the element (like an "Edit" button) from the DOM.
*(Note: UI RBAC is purely for UX. The backend API is the only true security boundary).*

### 7. What is the difference between deploying an Angular app to a static host (S3/Azure Blob) versus a compute host (Docker/Kubernetes)?
**Answer:**
* **Static Host:** Used for standard Client-Side Rendered (CSR) applications. The build output is just static HTML/JS/CSS. You upload these to a cheap storage bucket behind a CDN. It scales infinitely and costs almost nothing.
* **Compute Host:** Required for Server-Side Rendered (SSR) applications (Angular Universal). Because the HTML is generated dynamically per-request, you must run a Node.js server. This requires Dockerizing the app and deploying it to a compute service (Kubernetes, AWS Fargate, Azure App Service), which introduces scaling complexity, CPU monitoring, and higher costs.

### 8. How do you handle environment-specific variables (like API URLs) in a "Build Once, Run Anywhere" Docker strategy?
**Answer:**
You cannot use Angular's built-in `environment.ts` files, because they are hardcoded into the JavaScript bundle at compile time. 
Instead, you rely on **Runtime Configuration Injection**:
1. Remove API URLs from `environment.ts`.
2. Add a `config.json` file in the `src/assets` folder.
3. In `app.config.ts`, use the `APP_INITIALIZER` token to make an HTTP GET request to `/assets/config.json` before the Angular application finishes booting.
4. When deploying to Kubernetes, use a **ConfigMap** to mount over the `/assets/config.json` file inside the Docker container. 
Now, the exact same Docker image can boot up in Staging, read the Staging ConfigMap, and connect to Staging APIs.

### 9. What is a BFF (Backend-For-Frontend) and why might an Angular Architect request one?
**Answer:**
A BFF is a dedicated API gateway (often built in Node.js/NestJS or .NET) that sits between the Angular UI and the heavy backend microservices.
**Why it's needed:**
If a UI needs to display a complex dashboard, it might require data from the Billing Service, the User Service, and the Inventory Service. 
Without a BFF, the Angular app must make 3 separate HTTP requests over the slow public internet, combining the data on the client device (which drains battery on mobile).
With a BFF, the Angular app makes 1 single HTTP request to the BFF. The BFF (sitting in the same high-speed internal datacenter as the microservices) fetches the 3 sources, aggregates them, strips out unnecessary secure data, and returns a single, perfectly formatted JSON object to the UI.

### 10. You inherit a legacy Angular v12 application that is 500,000 lines of code. It takes 20 seconds to boot the app in the browser. Detail your optimization strategy.
**Answer:**
1. **Upgrade Angular:** Iteratively upgrade to v17+ to unlock the Ivy engine, native Esbuild, and Standalone components.
2. **Implement Lazy Loading:** Break the monolithic `AppModule` down. Ensure every major feature route uses `loadChildren` or `loadComponent`. 
3. **Analyze the Bundle:** Run `ng build --stats-json` and use `webpack-bundle-analyzer` or `esbuild-analyze`. Identify massive third-party libraries (e.g., Moment.js, massive charting libraries) that are blocking the main thread. Replace them with lighter alternatives (e.g., date-fns) or `@defer` them.
4. **Change Detection:** Enforce `ChangeDetectionStrategy.OnPush` globally to prevent CPU thrashing during user interactions.
5. **Server-Side Rendering:** Implement Angular SSR to send a fully rendered HTML page to the browser immediately, drastically improving the First Contentful Paint (FCP) and perceived performance while the massive JavaScript payload downloads in the background.
