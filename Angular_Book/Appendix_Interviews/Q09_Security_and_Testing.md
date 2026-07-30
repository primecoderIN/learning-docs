# Appendix 9: Security & Testing Interview Questions

This section covers protecting your Angular application from common enterprise vulnerabilities, along with the testing pyramid (Unit, Component, and E2E Testing).

---

## Junior Level Questions

### 1. What is XSS (Cross-Site Scripting), and how does Angular prevent it?
**Answer:**
XSS occurs when an attacker tricks a web application into executing malicious JavaScript in a user's browser (e.g., submitting a comment like `<script>alert('hacked')</script>`).
Angular prevents this by treating all bound values (`{{ value }}`) as untrusted strings. It performs **strict contextual escaping**, meaning the script tags are rendered visually on the screen as plain text, but the browser will never execute them.

### 2. What is the difference between a Unit Test and an End-to-End (E2E) Test?
**Answer:**
* **Unit Test:** Tests an isolated piece of code (like a Pipe or a Service function) without rendering the DOM or making real network requests. It runs in milliseconds.
* **E2E Test:** Uses a tool like Playwright or Cypress to spin up a real browser, navigate to the application, and simulate a user clicking buttons and filling out forms. It tests the entire stack (Frontend + Backend) and takes much longer to run.

### 3. What is the role of `TestBed` in Angular testing?
**Answer:**
`TestBed` is Angular's primary testing API. It creates a dynamic, simulated Angular module environment for the test. It allows you to compile components, resolve Dependency Injection tokens, and interact with the DOM via `ComponentFixture`.

---

## Mid-Level Questions

### 4. How does the `DomSanitizer` handle `[innerHTML]`, and when should you bypass it?
**Answer:**
If you bind HTML data to `[innerHTML]="userData"`, Angular recognizes that you want to render actual HTML tags. However, it runs the string through the `DomSanitizer` first. It strips out dangerous tags (like `<script>`, `<iframe>`) and dangerous attributes (like `onclick="stealTokens()"`).
You should only bypass it (using `bypassSecurityTrustResourceUrl` or `bypassSecurityTrustHtml`) if you are 100% certain the content is safe (e.g., embedding a known YouTube video or rendering a PowerBI dashboard iframe). **Never bypass the sanitizer using data input directly by a user.**

### 5. Why must you never store sensitive data (like a JWT) in `localStorage` in highly secure apps?
**Answer:**
If your application suffers even a minor XSS vulnerability (e.g., through a poorly implemented third-party component), an attacker's script can simply execute `localStorage.getItem('token')` and send the user's authentication token to their server. 
In highly secure enterprise apps (Banking, Healthcare), JWTs must be stored in **HTTP-Only Cookies**. The browser will automatically attach the cookie to backend requests, but JavaScript (and thus, XSS attacks) cannot read HTTP-Only cookies.

### 6. How do you prevent tests from making real HTTP requests?
**Answer:**
You configure `TestBed` to provide the `HttpTestingController` (using `provideHttpClientTesting()`).
When the component or service makes an HTTP call, the request is intercepted. You can then use `httpMock.expectOne('/api/users')` to verify the request was made, and call `req.flush(mockData)` to synchronously return fake data to the component. Finally, you should call `httpMock.verify()` in `afterEach` to ensure no unexpected HTTP requests were made.

---

## Senior Level Questions

### 7. What is CSRF (Cross-Site Request Forgery), and how does Angular protect against it?
**Answer:**
CSRF happens when a malicious website tricks a user's browser into sending a request to your application (using the user's existing session cookies).
If your backend uses HTTP-Only Cookies for authentication, you are vulnerable.
Angular protects against this using the **Double Submit Cookie pattern**. Angular's `HttpClient` automatically looks for a cookie named `XSRF-TOKEN`. If it finds it, it extracts the value and attaches it to a custom HTTP Header (`X-XSRF-TOKEN`) on every mutating request (POST/PUT/DELETE). The backend verifies that the cookie matches the header. Because the malicious site cannot read the cookie due to Same-Origin Policy, it cannot forge the header, and the backend blocks the request.

### 8. Explain the "Ice Cream Cone" testing anti-pattern vs. the Testing Pyramid.
**Answer:**
* **Testing Pyramid (Good):** A massive base of extremely fast, isolated Unit Tests covering every logic edge case; a smaller middle layer of Component DOM tests; and a tiny peak of slow E2E tests covering only the critical "Happy Paths" (e.g., Login, Checkout).
* **Ice Cream Cone (Anti-Pattern):** Writing hundreds of slow, flakey E2E tests and almost zero Unit Tests. This makes the CI/CD pipeline take hours to run, and when an E2E test fails, it's incredibly difficult to pinpoint exactly which line of code broke.

### 9. What is the difference between Shallow Testing and Deep Testing a Component?
**Answer:**
* **Deep Testing:** Rendering a parent component and allowing all its child components to render natively. This is essentially an integration test. It tests the real interaction, but it's slow and fragile because a bug in a child component will fail the parent's test.
* **Shallow Testing:** Rendering a parent component but mocking out (stubbing) all its child components. This tests the parent strictly in isolation. In modern Standalone Angular, you achieve this by overriding the component's imports in `TestBed.overrideComponent()`.

---

## Architect Level Questions

### 10. You are tasked with migrating an enterprise app from Protractor to Playwright. What are the core architectural differences and benefits?
**Answer:**
Protractor (deprecated) relied heavily on Selenium WebDriver. It was notoriously slow, flakey, and required manual `browser.wait()` commands everywhere because the DOM was often out of sync with the test.
Playwright operates directly over the browser's DevTools Protocol. 
**Key Benefits:**
1. **Auto-Waiting:** Playwright inherently waits for elements to be actionable (visible, enabled, not animating) before interacting with them, eliminating the need for `setTimeout` or manual waits.
2. **Parallelization:** It runs tests in parallel natively across multiple CPU cores.
3. **Network Interception:** Playwright can intercept and mock backend API responses directly at the browser network layer, allowing you to run E2E tests without needing to spin up a real backend database.

### 11. Explain how to securely implement a "Feature Flag" system in an Angular application.
**Answer:**
A Feature Flag system dictates whether a user can see a specific UI feature.
**The Security Trap:** If the backend sends `{ canSeeBilling: false }`, and the Angular developer simply uses `*ngIf="canSeeBilling"` in the template, it is **not secure**. An attacker can open Chrome DevTools, modify the Angular state, or manually change the HTML to reveal the Billing component.
**The Solution:**
1. Use `*ngIf` (or `@if`) combined with `@defer` to ensure the JavaScript chunk for the Billing component is never even downloaded to the browser if the flag is false.
2. Most importantly, the Backend API must *always* re-verify the feature flag during the HTTP POST request. The frontend flag is purely for UX; the backend is the only true authority on authorization.

### 12. How do you test a Component that heavily utilizes Signals without running into "stale data" issues in the DOM?
**Answer:**
When a Signal updates in a component, the DOM does not update immediately; Angular waits for the change detection cycle.
In a test environment, if you do `fixture.componentInstance.mySignal.set(true)`, you must immediately call `fixture.detectChanges()` to force Angular to flush the Signal graph and update the DOM. 
In modern Angular (v17+), you can also use `ComponentFixture.autoDetectChanges()` which automatically hooks into the test's `Zone` (or Zoneless scheduler) to flush Signal updates automatically, mimicking real browser behavior more closely.
