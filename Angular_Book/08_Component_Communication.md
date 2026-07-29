# Chapter 8: Component Communication

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Architect data flow between Smart (Container) and Dumb (Presentational) components.
* Use `@ViewChild` and `@ContentChild` (and their Signal equivalents) to query the DOM and child components.
* Implement robust parent-to-child and child-to-parent communication strategies.
* Understand when to use Component Communication versus global State Management (Services).

---

## 2. Introduction: The Container / Presentational Pattern

Before diving into *how* components communicate, we must discuss *why* and *when* they communicate. The gold standard for Enterprise Angular architecture is the **Container / Presentational Pattern** (often called Smart / Dumb components).

### 1. Smart Components (Containers)
These are typically routed components (e.g., `ChargerDashboardComponent`).
* **Responsibilities:** Inject services, dispatch HTTP calls, manage state, and handle routing.
* **Characteristics:** They have heavy dependencies (RxJS, State Stores) and minimal HTML. They pass data *down* to dumb components.

### 2. Dumb Components (Presentational)
These are reusable UI blocks (e.g., `ChargerCardComponent`, `StatusBadge`).
* **Responsibilities:** Render UI based strictly on inputs and emit events on user interaction.
* **Characteristics:** They inject zero services (except maybe a formatter). They know nothing about APIs or routing. They are highly testable.

Component communication is primarily the act of passing state down from Smart to Dumb components, and passing events up from Dumb to Smart components.

---

## 3. Parent to Child Communication

Passing data from a parent component down to a child component is achieved using Inputs.

### Modern Signal Inputs (Recommended)
As discussed in Chapter 7, Signal inputs are the modern standard.

**Child Component (`charger-card.component.ts`)**
```typescript
@Component({
  selector: 'app-charger-card',
  template: `<h2>{{ charger().name }}</h2>`
})
export class ChargerCardComponent {
  // Required input. The compiler throws an error if the parent forgets to bind it.
  charger = input.required<Charger>(); 
  
  // Optional input with a default value
  showDetails = input<boolean>(false);
}
```

**Parent Component Template**
```html
<!-- Passing data DOWN to the child -->
<app-charger-card [charger]="selectedCharger()" [showDetails]="true" />
```

---

## 4. Child to Parent Communication

Passing data or events back up from a child to a parent is achieved using Outputs.

### Modern Signal Outputs (Recommended)
Angular v17.3 introduced the `output()` function, replacing the clunky `@Output()` decorator and `EventEmitter` class.

**Child Component (`charger-card.component.ts`)**
```typescript
@Component({
  selector: 'app-charger-card',
  template: `<button (click)="onReboot()">Reboot Charger</button>`
})
export class ChargerCardComponent {
  charger = input.required<Charger>();
  
  // Defines the event and the type of data it emits
  rebootRequested = output<string>(); 

  onReboot() {
    this.rebootRequested.emit(this.charger().id);
  }
}
```

**Parent Component Template**
```html
<!-- Listening to events going UP from the child -->
<app-charger-card 
  [charger]="charger()" 
  (rebootRequested)="handleReboot($event)" 
/>
```

### Two-Way Binding (`model`)
If a child needs to read a value from a parent *and* update that exact value in the parent, use `model()`.

```typescript
// Child: statusToggle.component.ts
isActive = model<boolean>(false);

toggle() {
  this.isActive.update(v => !v); // Automatically emits change to parent!
}
```
```html
<!-- Parent HTML: Uses the "banana in a box" syntax for two-way binding -->
<app-status-toggle [(isActive)]="chargerIsActive" />
```

---

## 5. Querying the View: `viewChild` and `viewChildren`

Sometimes, a parent component needs to directly invoke a method on a child component, or gain direct access to a DOM element. We achieve this by querying the View.

### The Modern Signal View Queries
Angular v17.2 introduced signal-based view queries, replacing the `@ViewChild` decorator.

**Child Component (`map-viewer.component.ts`)**
```typescript
@Component({...})
export class MapViewerComponent {
  zoomIn() { /* Complex map logic */ }
}
```

**Parent Component**
```typescript
@Component({
  template: `
    <app-map-viewer #mapViewer />
    <button (click)="triggerZoom()">Zoom</button>
  `
})
export class DashboardComponent {
  // Queries the template for the component. Returns a Signal!
  map = viewChild(MapViewerComponent);

  // Queries the template for a specific Template Reference Variable (#mapViewer)
  mapRef = viewChild<MapViewerComponent>('mapViewer');

  triggerZoom() {
    // Read the signal to get the component instance, then call its method
    this.map()?.zoomIn();
  }
}
```

> **Important Lifecycle Note:** In the old `@ViewChild` days, the result was `undefined` inside `ngOnInit` and only available in `ngAfterViewInit`. With `viewChild()` signals, you can read them at any time (though they will evaluate to `undefined` before the view is rendered).

### `viewChildren`
Use `viewChildren` to query multiple instances of a component or element. It returns a `Signal<ReadonlyArray<T>>`.

```typescript
// Finds all ChargerCard components in the template
chargerCards = viewChildren(ChargerCardComponent);
```

---

## 6. Querying Projected Content: `contentChild`

What if the element you want to query isn't directly in your template, but was projected into your component using `<ng-content>`? (See Chapter 3).

You cannot use `viewChild` for projected content. You must use `contentChild` or `contentChildren`.

**Wrapper Component (`app-modal.component.ts`)**
```typescript
@Component({
  template: `
    <div class="modal">
      <ng-content></ng-content>
    </div>
  `
})
export class ModalComponent {
  // Finds the form that the parent projected into this modal
  projectedForm = contentChild(NgForm);
  
  validate() {
    console.log(this.projectedForm()?.valid);
  }
}
```

---

## 7. Sibling Communication (The Service Bus Pattern)

How do two completely unrelated sibling components communicate? (e.g., A Sidebar component needs to tell a Header component to update).

**Do NOT** pass data up to the parent and back down to the other child if the tree is deeply nested. This is called "Prop Drilling" and creates highly coupled, fragile code.

**Solution:** Use a shared Service injected into both components via Dependency Injection.

**`state.service.ts`**
```typescript
@Injectable({ providedIn: 'root' })
export class StateService {
  isSidebarOpen = signal(false);

  toggleSidebar() {
    this.isSidebarOpen.update(v => !v);
  }
}
```

**`header.component.ts`**
```typescript
export class HeaderComponent {
  state = inject(StateService);
  
  onMenuClick() {
    this.state.toggleSidebar();
  }
}
```

**`sidebar.component.ts`**
```typescript
@Component({
  template: `
    <!-- Reacts instantly to the signal change from the Header -->
    <aside [class.open]="state.isSidebarOpen()"></aside>
  `
})
export class SidebarComponent {
  state = inject(StateService);
}
```

---

## 8. Common Mistakes

### Beginner Mistakes
* **Prop Drilling:** Passing an `@Input` down through 4 layers of dumb components just to reach the one child that actually needs it. Use a shared service instead.
* **Mutating Inputs:** Trying to mutate the value of an `@Input()` inside the child component. Data flow must remain unidirectional (Top-Down). If the child needs to change it, it must emit an `@Output` so the parent can change the source of truth.

### Senior Mistakes
* **Overusing `ViewChild` for Data:** Using `ViewChild` to read data properties directly off a child component (`this.child().data`). This tightly couples the parent to the child's internal implementation. The child should instead emit events via `Output`, or both should read from a shared Service.

---

## 9. Interview Questions

### Intermediate
**Q: What is the difference between `viewChild` and `contentChild`?**
> A: `viewChild` queries elements that are directly written in the component's own HTML template. `contentChild` queries elements that were projected into the component from a parent component using `<ng-content>`.

### Architect
**Q: Explain the architectural benefits of the Container/Presentational pattern.**
> A: The Container/Presentational pattern physically separates business logic (state management, API calls) from UI rendering. Presentational (Dumb) components become highly reusable, pure UI functions that rely strictly on Inputs and Outputs, making them incredibly easy to unit test and catalog in tools like Storybook. Container (Smart) components can focus entirely on orchestrating RxJS streams and State, delegating all HTML layout to the dumb components.

---

## 10. Summary
In this chapter, we explored the arteries of an Angular application: Component Communication. We learned the architectural necessity of Smart/Dumb components, mastered the new Signal-based `input`, `output`, and `model` APIs, and learned how to query the DOM securely using `viewChild`.

In Chapter 9, we will scale this up to the enterprise level, exploring State Management architectures (NgRx, SignalStore) for complex applications.
