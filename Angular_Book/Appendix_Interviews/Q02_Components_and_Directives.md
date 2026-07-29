# Appendix 2: Components & Directives Interview Questions

This section covers the core building blocks of the UI, including component lifecycles, structural and attribute directives, content projection, and View queries.

---

## Junior Level Questions

### 1. What are the key differences between `@Input()` and `@Output()`?
**Answer:**
* `@Input()` is used to pass data **down** from a parent component to a child component.
* `@Output()` is used to emit events **up** from a child component to a parent component. It is always instantiated as an `EventEmitter`.

**Example:**
```typescript
@Component({
  selector: 'app-child',
  template: `<button (click)="notifyParent()">{{ title }}</button>`
})
export class ChildComponent {
  @Input() title = 'Default Title';
  @Output() buttonClicked = new EventEmitter<void>();

  notifyParent() {
    this.buttonClicked.emit();
  }
}
```

*(Note: In modern Angular, these are being replaced by the `input()` and `output()` signal-based functions).*

### 2. What is the difference between an Attribute Directive and a Structural Directive?
**Answer:**
* **Attribute Directives** change the appearance or behavior of a DOM element, but they do not add or remove the element from the DOM. Example: `ngClass`, `ngStyle`, or a custom `[appHighlight]` directive.
* **Structural Directives** actually shape or alter the DOM structure by adding, removing, or manipulating elements. They are prefixed with an asterisk (`*`). Examples: `*ngIf`, `*ngFor`, `*ngSwitch`. *(Note: Modern Angular uses the `@if` and `@for` control flow syntax instead).*

### 3. Explain the basic Angular Component Lifecycle hooks in order.
**Answer:**
1. **`ngOnChanges`**: Fires when an `@Input` bound property changes.
2. **`ngOnInit`**: Fires once after the first `ngOnChanges`. Used for initialization (API calls).
3. **`ngDoCheck`**: Fires during every change detection run.
4. **`ngAfterContentInit`**: Fires after external content is projected into the component.
5. **`ngAfterViewInit`**: Fires after the component's own views (and child views) are fully initialized.
6. **`ngOnDestroy`**: Fires just before the component is destroyed. Used for cleanup (unsubscribing).

---

## Mid-Level Questions

### 4. What is Content Projection and how does `<ng-content>` work?
**Answer:**
Content Projection allows a parent component to inject HTML content directly into a designated spot inside a child component. This is how you build reusable wrapper components like Modals or Cards.

**Child (`app-card`):**
```html
<div class="card">
  <div class="header">
    <ng-content select="[card-header]"></ng-content>
  </div>
  <div class="body">
    <!-- Un-selected content goes here -->
    <ng-content></ng-content> 
  </div>
</div>
```

**Parent:**
```html
<app-card>
  <h2 card-header>Title</h2>
  <p>This paragraph goes into the body.</p>
</app-card>
```

### 5. What is the difference between `@ViewChild` and `@ContentChild`?
**Answer:**
* `@ViewChild` queries elements or components that exist directly inside the component's **own HTML template**.
* `@ContentChild` queries elements or components that were **projected** into the component from a parent using `<ng-content>`.

Additionally, `@ViewChild` is only fully available in the `ngAfterViewInit` lifecycle hook, whereas `@ContentChild` is available slightly earlier in `ngAfterContentInit`.

### 6. How do you trigger an Angular lifecycle hook manually during a Unit Test?
**Answer:**
When using `TestBed`, simply instantiating the component does not fire lifecycle hooks. You must call `fixture.detectChanges()`, which triggers `ngOnInit` and the initial rendering cycle. To trigger `ngOnChanges`, you must manually set the input properties on the component instance (or use `fixture.componentRef.setInput()`), and then call `fixture.detectChanges()` again.

### 7. What is the `HostBinding` and `HostListener` decorator?
**Answer:**
They are used primarily inside Directives (and sometimes components) to interact with the host DOM element the directive is attached to, without needing to inject `ElementRef` directly.
* `@HostBinding('class.active')`: Binds a class or style on the host element to a boolean variable inside the directive.
* `@HostListener('click', ['$event'])`: Listens for DOM events on the host element and triggers a function.

---

## Senior Level Questions

### 8. Explain the "Banana in a Box" syntax `[()]` and how to implement custom two-way data binding.
**Answer:**
`[(ngModel)]` is syntactical sugar for combining an Input binding `[]` and an Output event binding `()`.
To create a custom two-way bound property, you must define an `@Input` and an `@Output` where the Output name is exactly the Input name plus the suffix `Change`.

**Child Component:**
```typescript
@Input() size: number;
@Output() sizeChange = new EventEmitter<number>();

increase() {
  this.sizeChange.emit(this.size + 1);
}
```
**Parent Component:**
```html
<app-child [(size)]="currentSize"></app-child>
<!-- Which Angular compiles down to: -->
<!-- <app-child [size]="currentSize" (sizeChange)="currentSize = $event"></app-child> -->
```
*(Note: Modern Angular achieves this elegantly using the `model()` signal).*

### 9. Why is mutating `@Input` arrays or objects a bad practice, and how does it break `OnPush` change detection?
**Answer:**
If a parent component passes an array `@Input() users` to a child, and the parent does `this.users.push(newUser)`, the *reference* in memory to the array has not changed. 
If the child component is using `ChangeDetectionStrategy.OnPush`, Angular strictly checks for referential equality (`oldUsers === newUsers`). Because the reference is the same, Angular assumes nothing changed, skips checking the component, and the new user will not appear on the screen.
**Solution:** Always treat Inputs as immutable. Create a new reference: `this.users = [...this.users, newUser]`.

### 10. How do you dynamically create and render a Component at runtime (without knowing about it in the HTML)?
**Answer:**
In modern Angular (v14+), you inject a `ViewContainerRef`, which represents a spot in the DOM. You can then use `createComponent()` to instantiate the component dynamically.

```typescript
@Component({
  template: `<ng-container #dynamicAnchor></ng-container>`
})
export class ParentComponent {
  @ViewChild('dynamicAnchor', { read: ViewContainerRef }) vcr!: ViewContainerRef;

  loadComponent() {
    this.vcr.clear();
    const componentRef = this.vcr.createComponent(DynamicModalComponent);
    componentRef.instance.title = 'Hello';
  }
}
```

---

## Architect Level Questions

### 11. Explain how the `@defer` control block handles the chunking of dependencies.
**Answer:**
When the Angular Compiler encounters an `@defer` block in a template, it statically analyzes everything inside that block (components, custom pipes, directives). It removes them from the main JavaScript bundle and compiles them into a separate, lazy-loaded chunk file. 
When the defer trigger fires (e.g., `on viewport`), Angular dynamically requests that chunk over the network. If multiple `@defer` blocks use the same shared component, the compiler is smart enough to extract that shared component into a common chunk so it is only downloaded once.

### 12. You are building a complex UI library (like Angular Material). How do you safely interact with the DOM without breaking Server-Side Rendering (SSR)?
**Answer:**
You must never access `window` or `document` directly inside your components, because those globals do not exist when the component is being rendered on a Node.js server.
1. Inject the `DOCUMENT` token (`@Inject(DOCUMENT) document: Document`) to interact with the DOM safely.
2. Use `Renderer2` to add classes or modify elements instead of `element.nativeElement.classList.add()`.
3. If you absolutely must use browser-specific APIs (like `IntersectionObserver` or `Chart.js`), you must wrap the execution inside `afterNextRender()`, which guarantees the code will only execute in the browser and will be skipped entirely on the server.

### 13. What is the `ControlValueAccessor` interface, and when is it architecturally necessary?
**Answer:**
`ControlValueAccessor` is the bridge that allows a custom UI component to hook directly into Angular's Reactive Forms API (`[formControl]` or `formControlName`). 
If you build a highly complex custom Datepicker component, you cannot simply use standard `@Input()` and `@Output()`. To make it behave like a native `<input>`, you must implement `ControlValueAccessor`.
This requires implementing 4 methods:
* `writeValue()`: Angular writes a new value to the UI.
* `registerOnChange()`: The component tells Angular the user changed the value.
* `registerOnTouched()`: The component tells Angular the user blurred the input.
* `setDisabledState()`: Angular tells the component to disable itself.
You must also provide it via the `NG_VALUE_ACCESSOR` injection token.

### 14. What are the performance implications of having a deeply nested component tree, and how do Smart/Dumb components solve it?
**Answer:**
A deeply nested tree forces Angular's Change Detection to traverse many nodes. If the root component modifies data, the default change detection checks every single child component down to the leaf nodes.
The **Container/Presentational (Smart/Dumb) Pattern** solves this. The Smart component sits at the top, connects to the State/API, and passes data down via Inputs. The Dumb components at the bottom are strictly configured with `ChangeDetectionStrategy.OnPush`. 
Because Dumb components only rely on Inputs, they form a "barrier". Angular will completely skip checking the deeply nested Dumb components unless their specific Input reference changed or a DOM event originated from them, drastically reducing CPU cycles.
