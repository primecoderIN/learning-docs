# Appendix 8: Performance & Change Detection

This section covers the hardest engineering challenges in Angular: identifying bottlenecks, mastering the `OnPush` change detection strategy, and utilizing modern load-time optimizations like `@defer`.

---

## Junior Level Questions

### 1. What is `Zone.js` and what is its role in Angular?
**Answer:**
`Zone.js` is a library that Angular uses under the hood to monkey-patch (intercept) asynchronous browser events (like `setTimeout`, HTTP requests, and DOM clicks). 
Whenever one of these asynchronous events completes, `Zone.js` notifies Angular, which then triggers a global "Top-Down" change detection cycle to check if any data in the application has changed and update the DOM accordingly.

### 2. How do you prevent a function in an HTML template from ruining performance?
**Answer:**
**Anti-pattern:** `<p>{{ calculateTax() }}</p>`
If you bind a function directly in the HTML template, Angular will execute that function on *every single change detection cycle*, which can happen dozens of times per second. If the function contains heavy logic, the UI will freeze.
**Solution:** Never bind functions in the template. Instead, use a **Pure Pipe** (`{{ price | calculateTax }}`) or a **Computed Signal** (`{{ taxAmount() }}`). Pure Pipes and Computed Signals are memoized, meaning they only recalculate if their specific inputs change.

### 3. What is the `ChangeDetectionStrategy.OnPush`?
**Answer:**
By default, Angular checks every component in the tree during a change detection cycle. 
Setting a component to `OnPush` tells Angular: *"Do not check this component or its children UNLESS an `@Input` reference changes, an event originates from this component, or an Observable/Signal bound in the template updates."*
This drastically reduces the number of components Angular has to check, massively improving performance in large applications.

---

## Mid-Level Questions

### 4. You are using `OnPush`, and you update a property inside `setTimeout`. Why doesn't the UI update, and how do you fix it?
**Answer:**
Because `setTimeout` is an asynchronous event, `Zone.js` triggers a global change detection cycle. However, because the component is set to `OnPush`, Angular skips checking it (since no `@Input` changed and no DOM event was fired inside it).
To fix it, you must inject the `ChangeDetectorRef` and manually tell Angular to check the component during the next cycle:
```typescript
constructor(private cdr: ChangeDetectorRef) {}

ngOnInit() {
  setTimeout(() => {
    this.status = 'Ready';
    this.cdr.markForCheck(); // Marks the path to this component as dirty
  }, 1000);
}
```
*(Note: If `status` was a Signal, `this.status.set('Ready')` would update the UI automatically without needing `ChangeDetectorRef`).*

### 5. What is the `@defer` control block?
**Answer:**
Introduced in Angular v17, `@defer` allows you to declaratively lazy-load components, directives, or pipes directly from the HTML template. 
Instead of bundling a massive chart component into the main JavaScript file, `@defer (on viewport)` tells the Angular compiler to put the chart into a separate chunk, and only download and render it when the user scrolls it into view.

### 6. What is the difference between `ChangeDetectorRef.markForCheck()` and `detectChanges()`?
**Answer:**
* `markForCheck()`: Does **not** trigger change detection immediately. It simply walks up the component tree to the root, marking every component in that path as "dirty." When the *next* normal change detection cycle runs, Angular will check those specific components.
* `detectChanges()`: **Immediately and synchronously** runs change detection on the current component and all its children, bypassing the normal Angular schedule. It is mostly used in Unit Testing or highly complex dynamic component rendering.

---

## Senior Level Questions

### 7. What causes the `ExpressionChangedAfterItHasBeenCheckedError`?
**Answer:**
This is Angular's most famous error (only visible in development mode). 
It occurs when a bound value in the template is modified *after* Angular has already checked it, but *before* the current change detection cycle finishes. 
For example, if a Parent component renders a Child component, and the Child component synchronously modifies a property on the Parent inside `ngAfterViewInit`, the Parent's view is now out of sync with its data. Angular throws this error to warn you that the UI is inconsistent.
**Fixes:** 
1. Use Signals (which natively prevent this).
2. Defer the update to the next JavaScript macro-task using `setTimeout(() => updateValue(), 0)`.
3. Refactor the architecture so data flows strictly top-down.

### 8. Explain how the Angular CDK Virtual Scroller improves performance for massive lists.
**Answer:**
If you render 10,000 `<div class="row">` elements using `@for`, the browser DOM will bloat, consuming hundreds of megabytes of RAM, and scrolling will stutter.
The `cdk-virtual-scroll-viewport` fixes this by only rendering the elements that are currently visible on the screen (e.g., 20 elements). As the user scrolls, it does not create new DOM nodes; it recycles the same 20 DOM nodes, simply swapping the data inside them. This keeps the DOM tiny and scrolling locked at 60 FPS.

### 9. How does "Event Coalescing" work in modern Angular?
**Answer:**
By default, if a user clicks a button, `Zone.js` triggers a change detection cycle. If an HTTP request finishes 1 millisecond later, `Zone.js` triggers *another* full cycle.
Event Coalescing (enabled via `provideZoneChangeDetection({ eventCoalescing: true })`) intercepts these triggers and uses `requestAnimationFrame` to delay the cycle slightly. This batches multiple rapid asynchronous events into a single, highly efficient change detection pass, saving CPU cycles.

---

## Architect Level Questions

### 10. How do you identify and prove a Memory Leak in an Angular Single Page Application?
**Answer:**
Memory leaks in Angular are typically caused by untracked RxJS subscriptions or detached DOM nodes holding references to components that have been destroyed.
To prove it:
1. Open the **Chrome DevTools Memory tab**.
2. Take a **Heap Snapshot**.
3. Navigate to the suspected component, then navigate away (do this 3-5 times).
4. Take a second **Heap Snapshot** and compare it to the first.
5. If you search for the Component's class name and see 5 retained instances in memory (when there should be 0), you have a leak. Inspecting the "Retainers" tree will usually point to a global Observable (like an NgRx Store or Router) that was not unsubscribed from in `ngOnDestroy`.

### 11. What is "Non-Destructive Hydration" in Server-Side Rendering (SSR)?
**Answer:**
In older SSR implementations (Angular Universal), the Node server would generate the HTML and send it to the browser. Once the browser downloaded the JavaScript, Angular would completely destroy the server-rendered DOM and recreate it from scratch, causing a visible UI flicker.
Modern Angular uses **Non-Destructive Hydration**. It downloads the JavaScript, intelligently maps it to the *existing* server-rendered DOM nodes, and attaches the necessary event listeners to make the page interactive, all without destroying the DOM. This results in a flawless, flicker-free user experience.

### 12. Explain the architectural steps to migrate an enterprise application to be fully "Zoneless".
**Answer:**
Zoneless Angular (v18+) completely removes the `Zone.js` dependency, drastically improving performance. To migrate:
1. Ensure the application is strictly using `ChangeDetectionStrategy.OnPush` everywhere.
2. Remove all reliance on native asynchronous browser events triggering UI updates. If a component uses `setTimeout` or `setInterval` to update a raw variable, it must be migrated to update a **Signal**, or you must manually call `ChangeDetectorRef.markForCheck()`.
3. Ensure all RxJS streams bound to the UI use the `AsyncPipe` (which internally marks components for check) or are converted to Signals via `toSignal()`.
4. In `app.config.ts`, replace `provideZoneChangeDetection()` with `provideExperimentalZonelessChangeDetection()`.
5. Remove `zone.js` from the `polyfills` array in `angular.json` to eliminate the bundle payload.
