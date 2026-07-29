# Appendix 1: Fundamentals & CLI Interview Questions

This section covers the absolute basics of Angular, transitioning into architect-level questions regarding the compilation process, bootstrapping, and workspace configurations.

---

## Junior Level Questions

### 1. What is Angular and how does it differ from AngularJS?
**Answer:**
Angular is a TypeScript-based, component-driven framework for building scalable web applications, created by Google. 
AngularJS (v1.x) was the original JavaScript-based framework. It relied heavily on two-way data binding (`ng-model`), controllers, and `$scope`, which led to severe performance issues at scale. Angular (v2+) is a complete rewrite. It enforces unidirectional data flow, uses a virtual-DOM-like change detection strategy (Zone.js/Signals), and is strictly typed using TypeScript.

### 2. What is the difference between a Component, a Directive, and a Pipe?
**Answer:**
* **Component:** A class with a view (HTML template) and styles. It is the core building block of the UI (e.g., `<app-header>`). It is technically a Directive with a template.
* **Directive:** A class that modifies the behavior or appearance of an existing DOM element. It has no template of its own (e.g., `<div appHighlight>`).
* **Pipe:** A pure function used in templates to transform data for display without altering the underlying data source (e.g., `{{ date | date:'short' }}`).

### 3. What does `ng serve` do behind the scenes?
**Answer:**
`ng serve` compiles the Angular application into JavaScript bundles and starts a local development server. In modern Angular (v17+), it uses Vite and Esbuild for blazing-fast Hot Module Replacement (HMR). It runs the application in memory, meaning it does not output physical files to the `dist/` folder unless you run `ng build`.

### 4. What is the difference between `constructor` and `ngOnInit`?
**Answer:**
* **`constructor`:** A standard JavaScript feature used for instantiating the class and injecting dependencies. Angular has not yet resolved the component's `@Input()` properties when the constructor runs. You should avoid putting business logic here.
* **`ngOnInit`:** An Angular lifecycle hook that fires exactly once after Angular has initialized the component's data-bound input properties. This is the correct place to fetch data from an API or set up RxJS subscriptions.

**Example:**
```typescript
export class UserComponent implements OnInit {
  @Input() userId!: string;

  constructor(private api: ApiService) {
    console.log(this.userId); // undefined!
  }

  ngOnInit() {
    console.log(this.userId); // '123'
    this.api.getUser(this.userId).subscribe();
  }
}
```

---

## Mid-Level Questions

### 5. Explain the difference between Ahead-of-Time (AOT) and Just-in-Time (JIT) compilation.
**Answer:**
* **JIT (Just-in-Time):** The Angular compiler is shipped to the browser. The browser downloads the app, parses the HTML templates, and compiles them into JavaScript at runtime. This causes massive bundle sizes and slow initial rendering.
* **AOT (Ahead-of-Time):** The Angular compiler runs on the build server during `ng build`. It converts all HTML templates into optimized JavaScript instructions *before* the code is shipped to the browser. The compiler itself is not shipped. This results in tiny bundle sizes, instant rendering, and catches template errors at compile time. Modern Angular strictly uses AOT.

### 6. What are Standalone Components and why were they introduced in Angular 14?
**Answer:**
Prior to Angular 14, every component, directive, and pipe had to be declared in an `NgModule`. This created a massive mental overhead, made lazy-loading difficult, and led to bloated `SharedModules`.
Standalone Components (`standalone: true`) eliminate the need for `NgModules`. A component now directly imports its own dependencies.

**Example:**
```typescript
@Component({
  selector: 'app-standalone',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, CustomPipe],
  template: `<p>I manage my own dependencies!</p>`
})
export class StandaloneComponent {}
```

### 7. What is `trackBy` in an `*ngFor` loop, and why is it crucial for performance?
**Answer:**
When an array bound to `*ngFor` is updated (e.g., fetching a fresh list from the API), Angular destroys the entire list of DOM elements and recreates them, even if only one item changed. This is highly expensive.
By providing a `trackBy` function, you tell Angular how to uniquely identify each item (usually by its ID). Angular will then only update, add, or remove the specific DOM elements that actually changed.
*(Note: In modern Angular, the `@for` control block natively enforces tracking via the `track` keyword).*

### 8. What is the purpose of the `environment.ts` files?
**Answer:**
They provide build-time configuration variables. When you run `ng build --configuration=production`, the Angular CLI replaces the contents of `environment.ts` with the contents of `environment.prod.ts`. This is commonly used to swap out the API base URL.
However, for enterprise Docker deployments, `environment.ts` should be avoided in favor of runtime `config.json` fetching, so the same Docker image can be promoted across environments without rebuilding.

---

## Senior Level Questions

### 9. How does Angular's Dependency Injection (DI) system resolve a dependency? Explain the injector hierarchy.
**Answer:**
Angular's DI system is hierarchical. When a component requests a dependency (e.g., `constructor(private http: HttpClient)`), Angular searches for it in this order:
1. **ElementInjector (NodeInjector):** Checks if the component itself or any parent component in the DOM tree provided the service in its `@Component({ providers: [] })` array.
2. **EnvironmentInjector (ModuleInjector):** If not found in the DOM hierarchy, it checks the global routing scope and application configuration. This is where services registered with `providedIn: 'root'` or provided in `app.config.ts` live.
3. **NullInjector:** If it reaches the top and finds nothing, it throws a `NullInjectorError`.

### 10. What is the difference between `providedIn: 'root'` and adding a Service to a Component's `providers` array?
**Answer:**
* `providedIn: 'root'`: Creates a single application-wide **Singleton**. The service is tree-shakeable (if never injected, the bundler removes it). State inside the service is shared across the entire app.
* `@Component({ providers: [MyService] })`: Creates a **new instance** of the service every time that specific component is instantiated. The service dies when the component is destroyed. It is used to scope state to a specific UI tree.

### 11. Explain "View Encapsulation" and the difference between `Emulated`, `None`, and `ShadowDom`.
**Answer:**
View Encapsulation determines how CSS styles are isolated to a component.
* **`Emulated` (Default):** Angular adds a unique attribute (e.g., `_ngcontent-c1`) to all HTML elements in the component and scopes the CSS to that attribute. Styles do not leak out, nor do they leak in.
* **`None`:** CSS defined in the component is injected globally into the `<head>`. It affects the entire application. Highly dangerous unless intentional.
* **`ShadowDom`:** Uses the browser's native Web Components Shadow DOM API. Provides absolute strict isolation, but breaks global styles (like Bootstrap or Tailwind) from entering the component.

### 12. How do you prevent a highly specific CSS style from being stripped out during an AOT build if it's dynamically applied?
**Answer:**
If you dynamically apply a class (e.g., `[class]="dynamicClass"`), the Angular compiler (or PurgeCSS/Tailwind) might remove the CSS class during Tree Shaking because it doesn't see a static reference in the HTML. 
You must either:
1. Safelist the class in your build configuration.
2. Use standard `[class.my-class]="condition"` binding so the compiler can statically analyze the dependency.

---

## Architect Level Questions

### 13. What is the Ivy Rendering Engine, and how does it achieve "Locality"?
**Answer:**
Ivy is Angular's third-generation compilation and rendering pipeline. 
**Locality** means that Ivy compiles a component strictly based on its own `@Component` decorator metadata, without needing global knowledge of the application or the `NgModule` it resides in. 
This locality is what enables rapid incremental builds, efficient tree-shaking, and the modern Standalone Component architecture. Under the hood, it compiles HTML templates into raw JavaScript instructions (like `ɵɵelementStart`), bypassing HTML parsing at runtime.

### 14. You are migrating a massive legacy application from AngularJS to modern Angular. What is the architectural strategy?
**Answer:**
A "Big Bang" rewrite is almost always a failure for massive applications. The correct strategy is **Incremental Migration using `@angular/upgrade`**.
1. **Bootstrapping:** Run both frameworks side-by-side. The Angular (v2+) application bootstraps the legacy AngularJS application inside it.
2. **Bottom-Up Migration:** Start migrating the "leaf" components (dumb components with no dependencies) to modern Angular.
3. **Downgrading:** Use `downgradeComponent` to allow the legacy AngularJS code to render the new modern Angular components.
4. **Upgrading:** Use `UpgradeComponent` to allow new modern Angular views to render legacy AngularJS components until they can be rewritten.
5. Once all routes and components are migrated, remove the `@angular/upgrade` package and AngularJS dependency entirely.

### 15. Explain how the Angular Compiler handles "Dead Code Elimination" (Tree Shaking) regarding Services vs. Components.
**Answer:**
For **Services**, if they use `@Injectable({ providedIn: 'root' })`, they are fully tree-shakeable. The bundler analyzes the import graph; if no component actively `injects()` the service, it is entirely omitted from the production bundle. However, if you explicitly add a service to a `providers: []` array, it creates a hard reference, and the bundler cannot tree-shake it, even if you never inject it.

For **Components**, in the legacy `NgModule` system, importing an `NgModule` pulled in *every* component declared in that module, breaking tree shaking. In the modern Standalone architecture, a component's `imports: []` array creates a strict dependency graph, allowing Esbuild to aggressively tree-shake and remove any standalone component that isn't explicitly imported into a route or another component.

### 16. How does Angular's Bootstrap process work under the hood in a standalone application?
**Answer:**
1. The browser executes `main.js`.
2. It encounters `bootstrapApplication(AppComponent, appConfig)`.
3. Angular creates the root **EnvironmentInjector** using the providers defined in `appConfig` (e.g., `provideHttpClient`, `provideRouter`).
4. It creates the root HTML element (`<app-root>`) and attaches the `AppComponent` to it.
5. It triggers the first Change Detection cycle (Top-Down check) to render the initial state.
6. If Zone.js is enabled, it monkey-patches all browser APIs to listen for future events. If Zoneless, it waits for Signals to notify the view.

### 17. Your CI/CD build is taking 45 minutes on a large enterprise monorepo. How do you architect a solution to reduce this to under 5 minutes?
**Answer:**
You must implement an **Nx Monorepo with Computation Caching**.
1. **Dependency Graph Analysis:** Nx statically analyzes the codebase to determine exactly which libraries and applications were affected by the current Git commit.
2. **Affected Builds:** Run `npx nx affected:build`. Nx will only build the specific apps/libs that changed, rather than the entire monorepo.
3. **Remote Caching (Nx Cloud):** If another developer (or a previous CI run) already built a specific library with the exact same source code, Nx skips the build entirely and downloads the compiled output from a remote cache in milliseconds. 
4. **Esbuild:** Ensure the project has been migrated off the legacy Webpack builder onto the native `esbuild` application builder provided in Angular v17+, which builds code up to 67% faster by leveraging native Go binaries.
