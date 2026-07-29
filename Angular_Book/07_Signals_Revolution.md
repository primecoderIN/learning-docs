# Chapter 7: The Signals Revolution

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Understand why Signals were introduced and how they solve the limitations of RxJS and Zone.js.
* Master the three reactive primitives: Writable Signals, Computed Signals, and Effects.
* Migrate legacy RxJS `BehaviorSubject` architectures to Modern Signals.
* Understand the paradigm shift to Signal Inputs (`input()`), Outputs (`output()`), and Models (`model()`).

---

## 2. Introduction: Why Signals?

For years, Angular relied on **Zone.js** to trigger change detection. Zone.js monkey-patches every asynchronous API in the browser (e.g., `setTimeout`, `addEventListener`, XHR requests). Whenever an async event finishes, Zone.js tells Angular: *"Something might have changed, go check the entire component tree."*

This "Top-Down" dirty checking works well for small apps but scales terribly in Enterprise applications. If a user clicks a button deep in the application, Angular checks thousands of components to see if their state changed.

**RxJS** was heavily used to mitigate this via the `async` pipe and `OnPush` change detection, but RxJS has a notoriously steep learning curve and causes memory leaks if unsubscribed improperly.

### Enter Signals (Angular v16+)
A Signal is a wrapper around a value that notifies interested consumers when that value changes. 

Unlike Zone.js, which checks the entire application blindly, Signals provide **Fine-Grained Reactivity**. When a Signal changes, Angular knows exactly which specific component (or even which exact DOM node) depends on that Signal, and updates *only* that piece of the UI. No Zone.js needed.

---

## 3. The Three Primitives

### 1. Writable Signals (`signal`)
A writable signal allows you to read its value and directly update it.

```typescript
import { signal } from '@angular/core';

const counter = signal(0);

// Reading a signal requires executing it like a function
console.log(counter()); // Output: 0

// Updating directly
counter.set(5);

// Updating based on the previous value
counter.update(current => current + 1);
```

### 2. Computed Signals (`computed`)
A computed signal derives its value from other signals. It is **lazy** and **memoized**. 
* **Lazy:** It doesn't calculate the value until someone actually reads it.
* **Memoized:** Once calculated, it caches the value. If the underlying signals haven't changed, reading it again returns the cached value instantly without recalculation.

```typescript
import { signal, computed } from '@angular/core';

const price = signal(100);
const quantity = signal(2);

// Derived state. Cannot be manually updated via .set()
const totalPrice = computed(() => price() * quantity());

console.log(totalPrice()); // 200

price.set(150);
console.log(totalPrice()); // 300
```
> **Architect Rule:** Never use methods in your HTML templates (`{{ calculateTotal() }}`). This destroys performance. Always use `computed()` signals to derive UI state.

### 3. Effects (`effect`)
An effect is an operation that runs whenever one or more signal values change. It is designed for side effects (like writing to `localStorage`, drawing to a canvas, or analytics).

```typescript
import { effect, signal } from '@angular/core';

const theme = signal('dark');

effect(() => {
  // Angular tracks that this effect depends on the 'theme' signal.
  // Whenever 'theme' changes, this code runs automatically.
  document.body.className = theme();
});
```
> **Architect Rule:** Avoid using `effect()` to update other signals (this can cause infinite loops). State derivation should be handled by `computed()`.

---

## 4. Modern Component API (Angular v17+)

Signals revolutionized how data flows into and out of components. The old `@Input()` and `@Output()` decorators are rapidly being replaced by Signal-based functions.

### Signal Inputs
Instead of `@Input()`, we use `input()`. It returns a read-only Signal.

**Old Way:**
```typescript
@Input() chargerId!: string; // Type is string
```
**New Way:**
```typescript
chargerId = input.required<string>(); // Type is Signal<string>
```

**Why it's better:** 
With decorators, if you needed to react to an input changing, you had to use `ngOnChanges` or a setter. With Signal inputs, you just use `computed()`!

```typescript
chargerId = input.required<string>();
// Automatically recalculates whenever chargerId changes!
apiEndpoint = computed(() => `/api/chargers/${this.chargerId()}`);
```

### Signal Outputs
Instead of `EventEmitter`, we use `output()`.

```typescript
// Old
@Output() delete = new EventEmitter<string>();

// New
delete = output<string>();

triggerDelete() {
  this.delete.emit('CHARGER_123');
}
```

### Model Inputs (Two-Way Binding)
Angular v17.2 introduced `model()`. It is a writable signal that automatically emits an event back to the parent when its value changes, perfecting two-way data binding.

```typescript
// Child Component
volume = model(50); // Initial value 50

increase() {
  // Updating the signal automatically notifies the parent!
  this.volume.update(v => v + 10);
}
```

---

## 5. Enterprise Case Study: EV Dashboard State

Let's convert a legacy RxJS `BehaviorSubject` architecture into a clean Signal-based architecture for our EV Charger dashboard.

**Legacy RxJS Store:**
```typescript
export class ChargerStore {
  private chargersSubject = new BehaviorSubject<Charger[]>([]);
  chargers$ = this.chargersSubject.asObservable();

  private filterSubject = new BehaviorSubject<string>('ALL');
  filter$ = this.filterSubject.asObservable();

  // Awkward RxJS combination to derive state
  filteredChargers$ = combineLatest([this.chargers$, this.filter$]).pipe(
    map(([chargers, filter]) => chargers.filter(c => c.status === filter))
  );
}
```

**Modern Signal Store:**
```typescript
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ChargerStore {
  // 1. Writable state
  chargers = signal<Charger[]>([]);
  filter = signal<string>('ALL');

  // 2. Computed derived state (Clean, readable, synchronous)
  filteredChargers = computed(() => {
    const currentFilter = this.filter();
    const allChargers = this.chargers();
    
    if (currentFilter === 'ALL') return allChargers;
    return allChargers.filter(c => c.status === currentFilter);
  });
  
  updateFilter(newFilter: string) {
    this.filter.set(newFilter);
  }
}
```
No `.subscribe()`, no `pipe()`, no `combineLatest`, and no memory leaks. The template simply binds to `store.filteredChargers()`.

---

## 6. Signals vs RxJS (The Golden Rule)

With Signals arriving, developers often ask: *"Is RxJS dead?"*
Absolutely not. They solve different problems.

### When to use Signals:
* **Synchronous State:** Storing arrays, objects, strings, booleans.
* **Derived State:** Computing totals, filtering lists, UI presentation logic.
* **DOM Binding:** Anything displayed directly in the HTML template.

### When to use RxJS:
* **Asynchronous Events:** HTTP Requests, WebSockets.
* **Time-based Logic:** Debouncing user input, intervals, throttling.
* **Race Condition Handling:** `switchMap`, `concatMap`.

> **The Enterprise Pattern:** Use RxJS for the *flow* (fetching data, debouncing searches), and map the final result into a *Signal* for the *state* (binding to the UI). Angular provides `toSignal()` and `toObservable()` for seamless interoperability.

---

## 7. Common Mistakes

### Beginner Mistakes
* **Forgetting the parenthesis:** Writing `{{ user.name }}` in the template instead of `{{ user().name }}` when reading a signal. 
* **Mutating Objects inside Signals:**
  ```typescript
  const user = signal({ name: 'John' });
  user().name = 'Jane'; // BAD: Angular doesn't know the signal changed!
  
  user.set({ name: 'Jane' }); // GOOD: New object reference.
  ```

### Architect Mistakes
* **Overusing Effects:** Trying to synchronize state by using `effect()` to update another signal. This leads to infinite change detection loops. `effect()` is for *outside-the-framework* side effects (like updating `localStorage` or drawing a chart). State synchronization must be handled by `computed()`.

---

## 8. Interview Questions

### Senior
**Q: Explain how `computed()` signals optimize performance compared to using a getter function in a template.**
> A: If you bind a standard getter function (e.g., `get total()`) in an Angular template, it is executed on *every single change detection cycle*, even if the underlying data hasn't changed. A `computed()` signal is memoized. It calculates its value once, caches it, and tracks its dependencies. It will only recalculate if one of its dependencies notifies it of a change, saving massive amounts of CPU cycles.

### Architect
**Q: How do Signals enable "Zoneless" Angular?**
> A: Historically, Zone.js monkey-patched the browser to know *when* an async event happened, but it didn't know *what* changed, forcing Angular to check the entire component tree top-down. Signals create a dependency graph. When a Signal is updated via `.set()`, it directly notifies the exact views that depend on it. This granular notification means Angular no longer needs Zone.js to guess when to run change detection.

---

## 9. Summary
In this chapter, we explored the Signals revolution. We learned the three core primitives (Writable, Computed, Effects), upgraded to Signal Inputs/Outputs, and established the golden rule of state management: **RxJS for async streams, Signals for synchronous state.** 

In Chapter 8, we will explore Component Communication, mastering how complex components talk to each other in large-scale applications.
