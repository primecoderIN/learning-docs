# Appendix 7: State Management Interview Questions

This section focuses on managing application state, contrasting basic Service-based state with traditional NgRx (Redux) and the modern `@ngrx/signals` architecture.

---

## Junior Level Questions

### 1. What is Application State?
**Answer:**
State is any data that an application needs to remember over time to render the UI correctly. 
* **Local State:** Data restricted to a single component (e.g., whether a specific dropdown is open or closed).
* **Global/Shared State:** Data used across multiple routes or components (e.g., the currently authenticated user's profile, or a shopping cart).

### 2. How do you share state between two completely unrelated sibling components without a State library?
**Answer:**
You use a shared Angular Service provided at the `root` level (a Singleton). The Service holds the data (usually in a `BehaviorSubject` or a `Signal`). Both components inject the Service. Component A updates the Signal, and Component B instantly reacts to the change. This pattern avoids the anti-pattern of "Prop Drilling" (passing data up and down multiple layers of parent/child components).

---

## Mid-Level Questions

### 3. What is the Redux Pattern, and what are its core principles?
**Answer:**
Redux is an architectural pattern for managing global state predictability.
1. **Single Source of Truth:** The entire state of the application is stored in one massive, nested object (the Store).
2. **State is Read-Only:** Components cannot mutate the state directly. They must dispatch an Action.
3. **Changes are made by Pure Functions (Reducers):** A Reducer takes the current State and the dispatched Action, and returns a completely *new* State object (Immutability).

### 4. What are NgRx Actions and Selectors?
**Answer:**
* **Actions:** Plain objects describing unique events that happened in the application. Examples: `[Charger API] Load Chargers`, `[User Settings] Toggle Dark Mode`.
* **Selectors:** Pure, memoized functions used to derive or extract a specific slice of data from the global state tree. Because they are memoized, they only recalculate if that specific slice of state changes, preventing expensive calculations during render cycles.

### 5. What is the purpose of NgRx Effects?
**Answer:**
Reducers must be completely pure, synchronous functions (no API calls, no timers).
NgRx Effects are RxJS streams that listen for dispatched Actions, perform asynchronous side-effects (like calling `HttpClient`), and then dispatch a *new* Action containing the API result (e.g., `Load Chargers Success` or `Load Chargers Failure`).

---

## Senior Level Questions

### 6. Why is mutating the state directly a massive anti-pattern in NgRx, and how does Angular handle it?
**Answer:**
If a developer writes `store.user.name = 'Bob'`, they mutate the existing object reference in memory. 
Because Angular relies heavily on `ChangeDetectionStrategy.OnPush` in enterprise apps, Angular strictly checks for referential equality (`oldState === newState`). If you mutate the object, the reference is identical. Angular assumes nothing changed and the UI will not update.
In NgRx development mode, mutating the state throws a runtime error. You must always use the spread operator (`{ ...state, user: { ...state.user, name: 'Bob' } }`) or a library like Immer to create a new object reference.

### 7. What are the primary drawbacks of traditional NgRx?
**Answer:**
* **Massive Boilerplate:** Adding a single piece of state (like a Counter) requires modifying 4 different files (Actions, Reducers, Selectors, Effects) and importing the module.
* **RxJS Dependency:** Everything is an Observable. Managing multiple subscriptions in the UI and understanding complex higher-order mapping in Effects creates a steep learning curve for junior developers.

### 8. Explain the modern `@ngrx/signals` (SignalStore) and how it differs from traditional NgRx.
**Answer:**
SignalStore is a modern, lightweight state management library built on top of Angular Signals.
* **No Boilerplate:** You define the State, Computeds (Selectors), and Methods (Actions/Reducers/Effects) in a single compact declaration.
* **Synchronous by Default:** It does not use RxJS for basic state reads or writes. State is stored in native Signals, read instantly in the template, and updated synchronously using the `patchState()` function.
* **Local or Global:** A SignalStore can be provided in root (Global), or provided at the Component level (Local state tied to a component's lifecycle).

### 9. How do you handle Asynchronous side-effects in a SignalStore?
**Answer:**
Since Signals are synchronous, SignalStore provides the `rxMethod` bridge.
You define a method that takes an RxJS `pipe`. Inside the pipe, you can use standard RxJS operators (`switchMap`, `debounceTime`) to handle the API call. Finally, you use `tap` and `patchState` to push the result back into the synchronous Signal state.

```typescript
loadChargers = rxMethod<void>(
  pipe(
    switchMap(() => this.api.getChargers().pipe(
      tap(chargers => patchState(this.store, { chargers }))
    ))
  )
);
```

---

## Architect Level Questions

### 10. When should an enterprise application choose a full NgRx Redux Store vs. `@ngrx/signals` vs. a basic Angular Service?
**Answer:**
* **Basic Service with Signals:** Good for small applications or strictly scoped feature state (e.g., a multi-step form). Fails at scale because there are no guardrails preventing developers from directly mutating the state from anywhere.
* **`@ngrx/signals` (SignalStore):** The modern recommendation for 90% of enterprise applications. It enforces immutable state updates via `patchState`, provides structured `computed` selectors, easily bridges to RxJS for effects, and significantly reduces cognitive load.
* **Traditional NgRx (Redux):** Should only be chosen if the application has immensely complex, deeply nested, highly interdependent global state where Actions need to trigger multiple disparate reducers simultaneously, or if the organization relies heavily on the Redux DevTools time-travel debugging.

### 11. Explain the "Facade Pattern" and how it decouples the UI from the State Management library.
**Answer:**
If a UI component directly injects `Store<AppState>` and dispatches NgRx actions, the UI becomes tightly coupled to NgRx. If the architect decides to migrate to SignalStore or Akita later, every single UI component must be rewritten.
A **Facade** is an `@Injectable()` service that acts as a middleman. 
The UI injects the Facade. The Facade exposes simple Signals/Observables (e.g., `facade.user$`) and simple methods (e.g., `facade.updateUser()`). 
Inside the Facade, it handles the messy work of injecting the Store, selecting state, and dispatching actions. If the state library changes, you only rewrite the Facade. The UI components remain completely untouched.

### 12. How do you handle "State Hydration" in Server-Side Rendering (SSR) using TransferState?
**Answer:**
If an Angular app uses SSR, the Node server boots up, dispatches an action to fetch data from the database, populates the Store, renders the HTML, and sends it to the browser. 
However, when the browser boots up the client-side Angular app, the Store is empty again! The browser will dispatch a duplicate HTTP request, causing the UI to flicker.
**TransferState** solves this. On the server, before sending the HTML, you serialize the NgRx Store state into a `<script>` tag inside the HTML payload. When the browser boots, it reads that script tag, instantly hydrates the NgRx Store with the pre-fetched data, and skips the duplicate HTTP request.
