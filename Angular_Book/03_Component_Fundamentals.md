# Chapter 3: Component Fundamentals & View Encapsulation

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Understand the lifecycle of an Angular Component and when to hook into it.
* Master the modern Control Flow syntax (`@if`, `@for`).
* Understand the internals of View Encapsulation (Emulated vs Shadow DOM).
* Use Content Projection (`<ng-content>`) to build reusable enterprise UI components.

---

## 2. Introduction to Components

In Angular, a Component is a TypeScript class adorned with the `@Component` decorator. It is the absolute core primitive of the framework. Every Angular application is a hierarchical tree of these components, starting from the Root Component (`AppComponent`).

Unlike React, where a component is just a function returning JSX, an Angular component strictly separates its concerns:
* **The Class (`.ts`)**: Manages state and behavior.
* **The Template (`.html`)**: Defines the presentation layer.
* **The Styles (`.scss`)**: Scoped specifically to this component.

---

## 3. The Component Lifecycle

Angular creates, updates, and destroys components. You can tap into key moments of a component's existence by implementing Lifecycle Hook interfaces.

### Core Lifecycle Hooks

1. **`ngOnInit()`**: Called exactly once after Angular has initialized all data-bound inputs (`@Input`). **Best practice:** Place initialization logic (like HTTP calls) here, *not* in the constructor. The constructor should only be used for Dependency Injection.
2. **`ngOnChanges()`**: Called before `ngOnInit()` and whenever one or more data-bound input properties change. Provides a `SimpleChanges` object. *(Note: In modern Angular, Signal Inputs are rapidly replacing the need for this hook).*
3. **`ngAfterViewInit()`**: Called after Angular has fully initialized the component's view and child views. Essential when querying the DOM via `@ViewChild`.
4. **`ngOnDestroy()`**: Called exactly once before Angular destroys the component. **Crucial:** Used to unsubscribe from RxJS Observables to prevent catastrophic memory leaks.

### Visual Diagram: Component Lifecycle

```text
[ Component Instantiated ] -> constructor() runs (DI resolved)
           ↓
[ Inputs Bound ] -> ngOnChanges() runs (first time)
           ↓
[ Initialization ] -> ngOnInit() runs
           ↓
[ View Rendered ] -> ngAfterViewInit() runs
           ↓
( ... User Interaction / State Changes ... )
           ↓
[ Component Removed ] -> ngOnDestroy() runs
```

---

## 4. Modern Control Flow (Angular v17+)

For years, Angular relied on structural directives like `*ngIf` and `*ngFor` to manipulate the DOM. While powerful, they required importing `CommonModule` and had slightly clunky syntax. 

Angular v17 introduced **Built-in Control Flow**, a native template syntax heavily inspired by Svelte and Razor. It is faster, requires no imports, and provides significantly better type-checking.

### The `@if` Block
```html
@if (sessionStatus() === 'CHARGING') {
  <div class="badge success">Active Charging</div>
} @else if (sessionStatus() === 'FAULT') {
  <div class="badge error">Hardware Fault</div>
} @else {
  <div class="badge default">Idle</div>
}
```

### The `@for` Block
The `@for` block requires a `track` expression. This is a massive performance optimization. It tells the Angular rendering engine exactly which item in the array corresponds to which DOM node, allowing Angular to move DOM nodes instead of destroying and recreating them.

```html
<ul>
  @for (charger of chargers(); track charger.id) {
    <li>{{ charger.name }} - {{ charger.status }}</li>
  } @empty {
    <li>No chargers found for this site.</li>
  }
</ul>
```

---

## 5. View Encapsulation Deep Dive

One of Angular's most powerful enterprise features is **View Encapsulation**. When you write CSS in an Angular component, it does not leak out and affect the rest of the application. 

### How does Angular do this?

Angular supports three encapsulation strategies (`ViewEncapsulation` enum):

#### 1. Emulated (Default)
Angular emulates the Shadow DOM by rewriting your CSS and HTML at compile time.
* **Component HTML:** `<div _ngcontent-c42 class="card">...</div>`
* **Compiled CSS:** `.card[_ngcontent-c42] { color: blue; }`

By appending a unique attribute (`_ngcontent-*`) to both the HTML elements and the CSS selectors, the styles are scoped exclusively to that component.

#### 2. ShadowDom
Angular uses the browser's native Web Components Shadow DOM API.
* **Pros:** True isolation. Even global styles (`styles.scss`) will not affect the component.
* **Cons:** Difficult to theme. Heavy framework styles (like Angular Material) cannot penetrate the Shadow DOM.

#### 3. None
Disables encapsulation entirely. The styles are injected into the global `<head>` and will affect the entire application.

---

## 6. Content Projection

Content Projection (similar to `children` in React or `slots` in Vue) allows you to pass HTML content from a parent component into a child component. This is essential for building reusable wrapper components like Modals, Cards, or Accordions.

### Single-Slot Projection
**Card Component Template (`app-card`)**
```html
<div class="enterprise-card">
  <!-- Whatever the parent puts inside <app-card> goes here -->
  <ng-content></ng-content> 
</div>
```

### Multi-Slot Projection
In enterprise design systems, you often need specific sections (Header, Body, Footer). You can project content into specific slots using the `select` attribute.

**Modal Component Template (`app-modal`)**
```html
<div class="modal-dialog">
  <header>
    <ng-content select="[modal-title]"></ng-content>
  </header>
  
  <main>
    <ng-content></ng-content> <!-- Default catch-all -->
  </main>
  
  <footer>
    <ng-content select="[modal-actions]"></ng-content>
  </footer>
</div>
```

**Usage by Parent Component:**
```html
<app-modal>
  <h2 modal-title>Delete Charger</h2>
  
  <p>Are you sure you want to delete this charging station? This cannot be undone.</p>
  
  <div modal-actions>
    <button (click)="cancel()">Cancel</button>
    <button class="danger" (click)="confirm()">Delete</button>
  </div>
</app-modal>
```

---

## 7. Enterprise SaaS Case Study: The Charger Status Badge

Let's build a reusable component for our EV Charging Platform using what we've learned. We will use Standalone components, modern control flow, and strict types.

**`charger-badge.component.ts`**
```typescript
import { Component, Input, ChangeDetectionStrategy } from '@angular/core';

export type ChargerStatus = 'AVAILABLE' | 'CHARGING' | 'FAULTED' | 'OFFLINE';

@Component({
  selector: 'app-charger-badge',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <span class="badge" [class]="statusClass">
      @switch (status) {
        @case ('AVAILABLE') { 🟢 Ready }
        @case ('CHARGING')  { ⚡ In Use }
        @case ('FAULTED')   { 🔴 Hardware Fault }
        @case ('OFFLINE')   { ⚪ Disconnected }
      }
    </span>
  `,
  styles: [`
    .badge {
      padding: 4px 8px;
      border-radius: 12px;
      font-weight: bold;
      font-size: 0.85rem;
    }
    /* Emulated Encapsulation keeps these classes completely isolated */
    .status-available { background: #e6f4ea; color: #137333; }
    .status-charging  { background: #e8f0fe; color: #1a73e8; }
    .status-faulted   { background: #fce8e6; color: #c5221f; }
    .status-offline   { background: #f1f3f4; color: #5f6368; }
  `]
})
export class ChargerBadgeComponent {
  @Input({ required: true }) status!: ChargerStatus;

  get statusClass(): string {
    return `status-${this.status.toLowerCase()}`;
  }
}
```

---

## 8. Common Mistakes

### Beginner Mistakes
* **Memory Leaks:** Failing to implement `ngOnDestroy` and unsubscribe from a global Observable stream. (The component is removed from the DOM, but the subscription lives forever in memory).
* **DOM Manipulation:** Using `document.getElementById` inside an Angular component. **Never do this.** Angular's rendering engine needs to own the DOM. If you mutate it directly, Angular's state becomes out of sync with the view.

### Architect Mistakes
* **Over-Projecting Content:** Overusing `<ng-content>` can make a component too flexible, breaking the design system. If a `CardComponent` allows *anything* to be projected, developers will project badly styled HTML into it. It is often better to use `@Input()` properties to enforce strict UI guidelines.

---

## 9. Interview Questions

### Senior
**Q: Explain how Angular's Emulated View Encapsulation works under the hood.**
> A: At compile time, the Angular compiler generates a unique attribute (e.g., `_ngcontent-abc`) for each component. It modifies the component's HTML to include this attribute on every element, and it rewrites the component's CSS selectors to target that specific attribute (e.g., `.my-class[_ngcontent-abc]`). This prevents the CSS from leaking globally without requiring native Shadow DOM support.

### Architect
**Q: Why does the new `@for` block require a `track` expression, and what is its performance impact?**
> A: The `track` expression tells the Angular rendering engine how to uniquely identify an item in an iterable. During change detection, if the array changes (e.g., an item is moved or added), Angular uses the `track` identity to reuse existing DOM nodes and move them, rather than destroying the entire list and recreating it from scratch. This drastically reduces expensive DOM manipulations and layout thrashing.

---

## 10. Summary
In this chapter, we explored the anatomy of an Angular Component. We learned how to hook into the lifecycle, how to use modern control flow (`@if`, `@for`), and how Angular isolates CSS using View Encapsulation. In Chapter 4, we will explore Directives and Pipes, learning how to attach complex behavior and data transformations directly to HTML elements.
