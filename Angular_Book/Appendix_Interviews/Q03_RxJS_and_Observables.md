# Appendix 3: RxJS & Observables Interview Questions

This section covers Reactive Programming using RxJS, a core dependency of Angular. It progresses from basic Observables to complex Higher-Order Mapping operators and advanced memory management.

---

## Junior Level Questions

### 1. What is the difference between an Observable and a Promise?
**Answer:**
* **Promise:** Represents a single future value. It executes immediately upon creation (eager), cannot be cancelled once started, and resolves exactly once (or rejects).
* **Observable:** Represents a stream of values over time. It does not execute until someone calls `.subscribe()` (lazy), it can emit multiple values over time, and it can be cancelled at any time by calling `.unsubscribe()`.

### 2. What is the difference between a `Subject` and a `BehaviorSubject`?
**Answer:**
* **`Subject`:** A special type of Observable that allows values to be multicasted to many Observers. However, it does not store the current value. If you subscribe *after* a value was emitted, you miss that value.
* **`BehaviorSubject`:** Requires an initial value upon creation and always stores the latest emitted value. If you subscribe late, it immediately synchronously emits its current value to you.

**Example:**
```typescript
const bSubject = new BehaviorSubject<number>(0);
bSubject.next(1);
bSubject.subscribe(val => console.log(val)); // Instantly logs '1'
```

### 3. What does the `AsyncPipe` (`| async`) do in Angular templates?
**Answer:**
The `AsyncPipe` automatically subscribes to an Observable or Promise directly in the HTML template and returns the latest emitted value. 
Most importantly, when the component is destroyed, the `AsyncPipe` **automatically unsubscribes**, preventing memory leaks without writing any manual `ngOnDestroy` logic.

```html
<!-- Subscribes automatically, displays value, unsubscribes on destroy -->
<h2>{{ currentUser$ | async }}</h2>
```

---

## Mid-Level Questions

### 4. Explain the difference between `map` and `switchMap` in RxJS.
**Answer:**
* **`map`:** Used to transform the emitted values of a stream (like standard JavaScript `Array.map`). It returns a primitive value or an object.
* **`switchMap`:** A Higher-Order Mapping operator. It is used when an emission requires you to subscribe to a *new* inner Observable (like an HTTP call). If the outer Observable emits again before the inner Observable finishes, `switchMap` instantly **cancels (aborts)** the previous inner Observable and switches to the new one.

### 5. When should you use `mergeMap` (formerly `flatMap`) instead of `switchMap`?
**Answer:**
* Use **`switchMap`** for read operations (e.g., Search Autocomplete). If a user types "T" then "Te", you want to cancel the HTTP request for "T".
* Use **`mergeMap`** for write operations (e.g., saving multiple documents). If a user clicks "Save" on 3 files rapidly, you do *not* want to cancel the first two saves. `mergeMap` allows all inner Observables to execute concurrently in parallel.

### 6. What is a "Cold" Observable vs a "Hot" Observable?
**Answer:**
* **Cold Observable:** The data producer is created *inside* the Observable. Every time you call `.subscribe()`, a brand new producer is created, and the stream starts from the beginning. (e.g., `HttpClient.get()` executes a new network request for every subscriber).
* **Hot Observable:** The data producer exists *outside* the Observable (e.g., a DOM click event or a `Subject`). Multiple subscribers share the same producer and receive the exact same data emissions in real-time.

### 7. How do you convert a Cold Observable into a Hot Observable?
**Answer:**
You use multicasting operators, most commonly `shareReplay()`. 
If you have an HTTP request that you want to share among 5 different components without making 5 separate network calls, you pipe it through `shareReplay(1)`. The first subscriber triggers the HTTP call, and all subsequent subscribers instantly receive the cached value.

```typescript
this.config$ = this.http.get('/config').pipe(
  shareReplay(1) // Makes it Hot and caches the last 1 value
);
```

---

## Senior Level Questions

### 8. What is the difference between `forkJoin` and `combineLatest`?
**Answer:**
* **`forkJoin`:** Waits for all provided Observables to **complete**. Once they all complete, it emits a single array (or object) containing the *final* value of each Observable. Used for parallel HTTP requests where you need all data before rendering the page.
* **`combineLatest`:** Does not wait for completion. As soon as all provided Observables have emitted at least one value, it emits an array of the latest values. Every time *any* of the input Observables emit a new value thereafter, it emits a new array. Used for reacting to multiple changing inputs (e.g., updating a filtered list when either the search term OR the category dropdown changes).

### 9. Explain the RxJS `debounceTime` vs `throttleTime` operators.
**Answer:**
Both are used to rate-limit emissions, typically for DOM events (like typing or scrolling).
* **`debounceTime(500)`:** Waits for 500ms of *silence* before emitting. If the user keeps typing, the timer resets. Used for Search Autocomplete to wait until the user pauses.
* **`throttleTime(500)`:** Emits the first value immediately, then ignores all subsequent values for 500ms. Used for button clicks to prevent double-submissions, or for window scroll events to limit rapid firing.

### 10. You have a component with 5 manual `.subscribe()` blocks in `ngOnInit`. How do you cleanly manage unsubscribing to prevent memory leaks?
**Answer:**
While the `AsyncPipe` is preferred, manual subscriptions must be cleaned up in `ngOnDestroy`.
The modern (Angular v16+) solution is the **`takeUntilDestroyed()`** operator from `@angular/core/rxjs-interop`.

```typescript
export class MyComponent {
  constructor() {
    this.apiService.getData().pipe(
      takeUntilDestroyed() // Automatically unsubscribes when component dies
    ).subscribe(data => console.log(data));
  }
}
```
*Legacy Answer:* Create a `private destroy$ = new Subject<void>()`, pipe all subscriptions through `takeUntil(this.destroy$)`, and call `this.destroy$.next()` inside `ngOnDestroy`.

---

## Architect Level Questions

### 11. Explain the "Imperative vs Declarative" RxJS paradigm and why nested subscribes are an anti-pattern.
**Answer:**
**Imperative (Anti-Pattern):**
```typescript
// BAD: Nested Subscribes
this.route.params.subscribe(params => {
  this.http.get('/users/' + params.id).subscribe(user => {
    this.user = user;
  });
});
```
This causes race conditions, prevents cancellation, and makes the code difficult to test.

**Declarative (RxJS Native):**
```typescript
// GOOD: Declarative Pipeline
this.user$ = this.route.params.pipe(
  switchMap(params => this.http.get('/users/' + params.id))
);
```
In declarative RxJS, we define a "blueprint" of how data flows through the application using operators (`switchMap`, `map`, `catchError`), and let the template `AsyncPipe` execute the flow. This eliminates race conditions because `switchMap` inherently handles cancellation.

### 12. Explain the danger of using `catchError` incorrectly in a persistent stream (like an NgRx Effect or a UI event listener).
**Answer:**
If an error occurs inside an Observable stream and is caught by `catchError`, **the original stream completes and dies forever**. 
If you are listening to a "Search Button Clicks" Subject, map it to an HTTP call, and the HTTP call fails with a 404, the `catchError` handles it, but the Search Button stream dies. The user can never click the search button again without refreshing the page.

**Solution:** You must catch the error on the *inner* Observable (inside the `switchMap`), not on the *outer* stream.

```typescript
// CORRECT ERROR HANDLING
this.searchClicks$.pipe(
  switchMap(query => this.http.get('/search?q=' + query).pipe(
    // Catch error here, returning a fallback value (like an empty array).
    // The inner stream completes, but the outer searchClicks$ stream stays alive!
    catchError(() => of([])) 
  ))
).subscribe();
```

### 13. What is a "Memory Leak" in the context of Angular routing, and how can RxJS cause it?
**Answer:**
A memory leak occurs when a Component is destroyed (e.g., the user navigates to a different page), but the JavaScript Garbage Collector cannot remove the component from memory because an active reference still exists.
If a component subscribes to a global service (like `Router.events` or an NgRx Store) and fails to unsubscribe in `ngOnDestroy`, the global service retains a reference to the component's callback function. Every time the user visits that route, a new component instance is created and permanently retained in memory, eventually crashing the browser tab.

### 14. How do you implement robust HTTP Retry logic with exponential backoff using RxJS?
**Answer:**
Instead of a simple `retry(3)`, enterprise apps use `retryWhen` (or `retry({ delay: ... })` in modern RxJS) to implement exponential backoff, preventing server flooding during an outage.

```typescript
this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => {
      // If it's a 404, don't retry, throw instantly
      if (error.status === 404) return throwError(() => error);
      
      // Otherwise, retry with exponential backoff (1s, 2s, 4s...)
      console.warn(`Retry attempt ${retryCount}`);
      return timer(Math.pow(2, retryCount - 1) * 1000);
    }
  })
);
```
