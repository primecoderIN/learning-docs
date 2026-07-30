# Appendix 4: Signals & Reactivity Interview Questions

This section covers the revolutionary reactivity model introduced in Angular v16+, focusing on Signals, Computeds, Effects, and the philosophical shift from RxJS to fine-grained reactivity.

---

## Junior Level Questions

### 1. What is an Angular Signal?
**Answer:**
A Signal is a wrapper around a value that notifies interested consumers when that value changes. It is the new reactive primitive in Angular (v16+). Unlike regular variables, when a Signal's value changes, Angular knows exactly where that Signal is used in the template and can update the UI instantly without needing a global change detection check.

### 2. How do you read, update, and mutate a `WritableSignal`?
**Answer:**
* **Read:** Call the signal as a function: `mySignal()`.
* **Update (set):** Replaces the entire value: `mySignal.set(newValue)`.
* **Update (compute):** Updates based on the previous value: `mySignal.update(old => old + 1)`.

**Example:**
```typescript
count = signal(0);
user = signal({ name: 'Alice', age: 30 });

increment() {
  this.count.update(c => c + 1);
}

changeName() {
  // Always create a new object reference to trigger change detection!
  this.user.update(u => ({ ...u, name: 'Bob' }));
}
```
*(Note: In earlier versions of Angular 16, `mutate()` existed for objects/arrays, but it was deprecated in v17 in favor of strict immutability using `update()`).*

### 3. What is the difference between `signal()` and `computed()`?
**Answer:**
* `signal()` creates a **Writable Signal**. You can manually change its value using `.set()` or `.update()`.
* `computed()` creates a **Read-Only Signal** derived from one or more other Signals. You *cannot* manually set its value. Angular automatically recalculates it only when its underlying dependencies change.

```typescript
price = signal(100);
quantity = signal(2);
// Automatically updates if price or quantity changes!
total = computed(() => this.price() * this.quantity()); 
```

---

## Mid-Level Questions

### 4. What is an `effect()`, and when should you use it?
**Answer:**
An `effect()` is a function that runs at least once, and then automatically re-runs whenever any Signal read inside it changes. 
* **When to use it:** For side-effects that need to react to state changes, such as synchronizing data with `localStorage`, custom DOM manipulation (like initializing a 3rd party charting library), or logging.
* **When NOT to use it:** You should rarely use an effect to update *another* Signal (use a `computed` instead). You should also avoid using it for HTTP requests, which are better handled by RxJS.

```typescript
constructor() {
  effect(() => {
    // Re-runs automatically every time user() changes
    localStorage.setItem('currentUser', JSON.stringify(this.user()));
  });
}
```

### 5. Why did Angular introduce Signals if RxJS already exists?
**Answer:**
RxJS is incredibly powerful for handling asynchronous events (HTTP requests, debouncing clicks, WebSockets). However, using RxJS for **synchronous UI state** is clunky.
* RxJS streams can emit errors and complete (which UI state shouldn't do).
* RxJS is "lazy" and requires manual subscriptions (`subscribe()` or `| async`).
* RxJS `BehaviorSubject` requires complex `pipe` logic to derive state.
* Signals are "glitch-free" (synchronous), never complete, are inherently readable without a subscription, and trigger true fine-grained UI updates without relying on `Zone.js`.

### 6. Explain the `input()` API introduced in Angular v17.1.
**Answer:**
The `input()` API replaces the legacy `@Input()` decorator. It creates a Signal-based input.
Instead of binding to a raw variable, the component receives a Read-Only Signal. This allows developers to use `computed()` or `effect()` to react to input changes effortlessly, completely eliminating the need for the `ngOnChanges` lifecycle hook.

```typescript
export class UserCardComponent {
  // Optional input
  age = input<number>(0);
  
  // Required input
  name = input.required<string>();

  // Derived state reacts instantly to input changes!
  isAdult = computed(() => this.age() >= 18);
}
```

---

## Senior Level Questions

### 7. How does Angular's `computed()` signal handle caching and memoization?
**Answer:**
`computed()` signals are lazily evaluated and heavily memoized.
1. If the underlying dependencies (e.g., `price` and `quantity`) change, the `computed` signal marks itself as "stale."
2. **Crucially**, it does *not* immediately recalculate its value.
3. It only recalculates the next time a consumer (like the HTML template) actually calls `total()`.
4. If it is called 50 times in the template but the dependencies haven't changed, it returns the cached value instantly without running the computation function again.

### 8. How do you integrate RxJS with Signals?
**Answer:**
Angular provides two interoperability functions in `@angular/core/rxjs-interop`:
* **`toSignal()`:** Converts an RxJS Observable into a read-only Signal. It handles subscribing and automatically unsubscribing when the component is destroyed.
* **`toObservable()`:** Converts a Signal into an RxJS Observable, allowing you to use operators like `debounceTime`.

**Example:**
```typescript
// Converting an HTTP request into a Signal
userData = toSignal(this.http.get('/api/user'), { initialValue: null });

// Using it in the template (No AsyncPipe needed!)
// <h2>{{ userData()?.name }}</h2>
```

### 9. What is the `model()` function, and how does it differ from `input()`?
**Answer:**
`model()` (introduced in v17.2) is the modern Signal replacement for two-way data binding (formerly achieved using an `@Input` and a matching `@Output`).
* `input()` is strictly read-only inside the child component.
* `model()` creates a `WritableSignal`. If the child component updates the model via `this.myModel.set(newValue)`, it automatically emits an event back up to the parent component to update the parent's source of truth.

**Child Component:**
```typescript
export class ToggleComponent {
  isActive = model(false);
  toggle() { this.isActive.update(v => !v); } // Emits to parent implicitly!
}
```

---

## Architect Level Questions

### 10. Explain the concept of "Fine-Grained Reactivity" and how Signals enable "Zoneless" Angular.
**Answer:**
Historically, Angular used `Zone.js` to monkey-patch the browser. When a click event occurred, `Zone.js` triggered a global Top-Down check. Angular would check every single binding on every single component from the Root down to the leaves, just in case a value changed.
**Fine-Grained Reactivity** means the framework knows exactly *what* piece of state changed and exactly *which specific DOM node* relies on it. 
Because Signals track their consumers directly, when a Signal updates, Angular doesn't need to traverse the component tree. It can update the specific view instantly. This allows us to remove `Zone.js` entirely (`provideExperimentalZonelessChangeDetection()`), drastically reducing CPU overhead and bundle size.

### 11. What is a "Glitch" in reactive programming, and how do Angular Signals prevent it?
**Answer:**
A "glitch" occurs when a derived state recalculates multiple times with inconsistent intermediate values before settling on the final correct value. This happens frequently in RxJS if you combine streams synchronously.
Angular Signals prevent glitches using a **Push/Pull mechanism** (specifically, topological sorting of the dependency graph).
1. **Push:** When a WritableSignal changes, it pushes a "Dirty" notification down the graph to all consumers (like Computeds and Effects), but it does *not* push the new value.
2. **Pull:** Later, during the rendering phase, the View pulls the value from the Computed signal. The Computed signal pulls the fresh values from its dependencies, recalculates *once*, and returns the final value. The intermediate broken states are never rendered to the screen.

### 12. You are migrating an enterprise NgRx Store to `@ngrx/signals` (SignalStore). How do you handle asynchronous HTTP calls, since Signals are strictly synchronous?
**Answer:**
Signals are designed for synchronous state. You cannot use an `effect()` to cleanly manage HTTP requests because you lose cancellation, debouncing, and race-condition protections.
In `@ngrx/signals`, you handle this using the **`rxMethod`**. 
`rxMethod` creates a bridge: It takes a Signal (or primitive) as an input, passes it into an RxJS `pipe` where you can use `switchMap` and `catchError`, and then finally uses `patchState` to synchronously push the result back into the SignalStore. This provides the robust async capabilities of RxJS combined with the glitch-free UI rendering of Signals.
