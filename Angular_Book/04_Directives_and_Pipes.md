# Chapter 4: Directives & Pipes

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Distinguish between Components, Attribute Directives, and Structural Directives.
* Build custom Attribute Directives to encapsulate DOM behaviors using `HostBinding` and `HostListener`.
* Understand how Pipes transform data in the template.
* Differentiate between Pure and Impure Pipes and understand their severe performance implications.
* Build a custom Signal-compatible Pipe for our EV Charging Platform.

---

## 2. Introduction to Directives

In Angular, a Component is actually just a Directive with a template. Directives are classes that add additional behavior to elements in your Angular applications. When Angular renders a template, it transforms the DOM according to the instructions provided by directives.

There are three kinds of directives in Angular:
1. **Components:** Directives with a template. (Covered in Chapter 3).
2. **Attribute Directives:** Change the appearance or behavior of an element, component, or another directive.
3. **Structural Directives:** Change the DOM layout by adding and removing DOM elements (e.g., `*ngIf`, `*ngFor`). *Note: As discussed in Chapter 3, Modern Angular (v17+) replaces structural directives with built-in Control Flow syntax (`@if`, `@for`).*

---

## 3. Attribute Directives Deep Dive

An Attribute Directive is used to encapsulate DOM interactions. Instead of writing messy event listeners and DOM queries inside a component class, you extract that logic into a highly reusable Directive.

### Core API Concepts

* **`ElementRef`**: Injects a reference to the host DOM element so the directive can manipulate it directly (use with caution).
* **`Renderer2`**: A safer abstraction over direct DOM manipulation, ensuring your code remains compatible with Server-Side Rendering (SSR).
* **`@HostListener`**: Decorator that declares a DOM event to listen for, and provides a handler method to run when that event occurs.
* **`@HostBinding`**: Decorator that allows you to bind a directive's property to a property on the host DOM element (e.g., a CSS class, an attribute, or a style).

### Case Study Example: The Access Role Directive

In our **EV Charging Platform**, certain buttons should only be clickable if the user has a specific RBAC permission (e.g., `chargers.delete`). If they don't, the button should appear disabled and show a "Not Authorized" tooltip on hover.

Instead of writing this logic into every component, we can create an Attribute Directive.

**`require-permission.directive.ts`**
```typescript
import { Directive, Input, ElementRef, Renderer2, OnInit, inject } from '@angular/core';
import { AuthService } from '../core/auth/auth.service';

@Directive({
  selector: '[appRequirePermission]',
  standalone: true
})
export class RequirePermissionDirective implements OnInit {
  @Input('appRequirePermission') requiredPermission!: string;

  private authService = inject(AuthService);
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);

  ngOnInit(): void {
    const hasPermission = this.authService.hasPermission(this.requiredPermission);

    if (!hasPermission) {
      // Disable the button safely using Renderer2 (SSR safe)
      this.renderer.setProperty(this.el.nativeElement, 'disabled', true);
      this.renderer.setStyle(this.el.nativeElement, 'opacity', '0.5');
      this.renderer.setAttribute(this.el.nativeElement, 'title', 'Not Authorized');
    }
  }
}
```

**Usage in HTML:**
```html
<button 
  appRequirePermission="chargers.delete" 
  (click)="deleteCharger()">
  Delete Station
</button>
```
By encapsulating this behavior, the component remains ignorant of authorization logic, adhering perfectly to the Single Responsibility Principle.

---

## 4. Introduction to Pipes

Pipes are functions that accept an input value and return a transformed output for display in the UI. They are declared in the HTML template using the pipe operator (`|`).

Angular provides several built-in pipes:
* `DatePipe` (`value | date:'short'`)
* `CurrencyPipe` (`value | currency:'USD'`)
* `DecimalPipe` (`value | number:'1.2-2'`)
* `AsyncPipe` (`value$ | async`) - *Crucial for RxJS unwrapping.*
* `JsonPipe` (`value | json`) - *Excellent for debugging objects.*

---

## 5. Pure vs Impure Pipes (The Performance Trap)

Understanding the difference between Pure and Impure pipes is one of the most critical concepts for an Angular Architect.

### Pure Pipes (Default)
By default, all pipes in Angular are **Pure**. A Pure pipe is only executed by Angular's change detection cycle when it detects a **pure change** to the input value.
A pure change is either:
1. A primitive value change (e.g., `String`, `Number`, `Boolean`).
2. A changed object reference (e.g., `Date`, `Array`, `Function`, `Object`).

If you mutate an array (e.g., `array.push(item)`) instead of replacing the array reference (`array = [...array, item]`), a pure pipe **will not run**.

### Impure Pipes
You can make a pipe impure by setting `pure: false` in the `@Pipe` metadata.
An impure pipe runs on **every single change detection cycle**, regardless of whether the input value changed. 

> **Architect Warning:** Never use an impure pipe to filter or sort arrays in a template. If a user moves their mouse across the screen, change detection runs, and your array will be resorted hundreds of times a second. This will instantly freeze large enterprise dashboards. Always sort/filter data in the Component class (preferably via Computed Signals), not via a Pipe in the template.

---

## 6. Case Study Example: The KwH Formatter Pipe

In our EV platform, we constantly display energy metrics (Kilowatt-hours). Let's build a pure pipe to format raw decimal values into clean KwH strings.

**`kwh.pipe.ts`**
```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'kwh',
  standalone: true
})
export class KwhPipe implements PipeTransform {
  
  transform(value: number | string | null | undefined, precision: number = 2): string {
    if (value == null || value === '') {
      return '0.00 kWh';
    }

    const numValue = typeof value === 'string' ? parseFloat(value) : value;
    
    if (isNaN(numValue)) {
      return '0.00 kWh';
    }

    return `${numValue.toFixed(precision)} kWh`;
  }
}
```

**Usage:**
```html
<p>Total Energy Delivered: {{ session.energyDelivered | kwh:3 }}</p>
```

---

## 7. The AsyncPipe

The `AsyncPipe` is arguably the most important built-in pipe in Angular when working with RxJS. It subscribes to an `Observable` or `Promise` and returns the latest emitted value. 

Crucially, when the component is destroyed, the `AsyncPipe` **automatically unsubscribes** to prevent memory leaks.

```typescript
@Component({ ... })
export class DashboardComponent {
  // An Observable stream of chargers
  chargers$ = this.chargerService.getActiveChargers();
}
```
```html
<!-- The async pipe subscribes, unwraps the array, and assigns it to 'chargers' -->
@if (chargers$ | async; as chargers) {
  <ul>
    @for (charger of chargers; track charger.id) {
      <li>{{ charger.name }}</li>
    }
  </ul>
}
```
*Note: As we transition to Signals, the `AsyncPipe` becomes less necessary, but it remains heavily used in legacy codebases and specific RxJS Interop scenarios.*

---

## 8. Common Mistakes

### Beginner Mistakes
* **Mutating Arrays into Pure Pipes:** Pushing a new item into an array bound to a pure pipe, and wondering why the view doesn't update. You must reassign the array reference to trigger the pipe.
* **Direct DOM Manipulation in Components:** Using `document.querySelector` inside a component instead of abstracting the behavior into an Attribute Directive using `ElementRef` and `Renderer2`.

### Intermediate / Senior Mistakes
* **Using Pipes for Sorting/Filtering:** Creating a `SortPipe` or `FilterPipe` and using it on a large dataset in an `*ngFor` loop. Even if pure, it triggers heavy computation on array reference changes. This logic belongs in the Component class (or ideally, a computed signal).

---

## 9. Interview Questions

### Intermediate
**Q: What is the difference between `@HostBinding` and `@HostListener`?**
> A: `@HostBinding` allows a directive to set properties, attributes, or CSS classes on the DOM element it is attached to. `@HostListener` allows a directive to listen for native DOM events (like `click` or `mouseenter`) on the host element and execute a method in response.

### Senior
**Q: Why does the Angular team strongly advise against creating a custom FilterPipe for arrays?**
> A: Filtering an array inside a pipe forces the filtering logic to execute during the template rendering phase. If the pipe is pure, it creates an aggressive constraint: the array reference must change to trigger a re-filter. If the pipe is impure, the filter runs on every change detection cycle (e.g., mouse movements), completely destroying performance. State derivation (filtering/sorting) should happen in the component class (via Computed Signals or RxJS map operators), providing the template with pre-computed, immutable data.

---

## 10. Summary
In this chapter, we explored Directives and Pipes. We learned how Attribute Directives encapsulate complex DOM behaviors and how Pipes format data cleanly for the UI. We also covered the critical performance difference between pure and impure pipes. In Chapter 5, we will dive into the beating heart of Angular's architecture: Dependency Injection.
