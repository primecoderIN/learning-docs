# Chapter 5: Dependency Injection Internals

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Understand the core philosophy of Dependency Injection (DI) and Inversion of Control (IoC).
* Differentiate between the two Angular Injector hierarchies: `EnvironmentInjector` and `NodeInjector`.
* Configure advanced providers using `useClass`, `useValue`, and `useFactory`.
* Understand the paradigm shift from constructor injection to the modern `inject()` function.

---

## 2. Introduction to Dependency Injection

Dependency Injection (DI) is a design pattern used to implement Inversion of Control (IoC). Instead of a class creating its own dependencies via the `new` keyword, the framework provides (injects) those dependencies from the outside.

**Without DI (Tightly Coupled & Hard to Test):**
```typescript
export class DashboardComponent {
  private apiService: ApiService;

  constructor() {
    // Hardcoded dependency. Impossible to mock in unit tests.
    this.apiService = new ApiService(); 
  }
}
```

**With DI (Decoupled & Testable):**
```typescript
@Component({...})
export class DashboardComponent {
  // Angular's DI system provides the instance. Easily mockable.
  constructor(private apiService: ApiService) {}
}
```

Angular is unique among frontend frameworks because it ships with a massive, hierarchical, enterprise-grade DI container built directly into the core runtime.

---

## 3. The Two Injector Hierarchies

Angular does not just have one global bucket of dependencies. It has two parallel, hierarchical trees of injectors. Understanding these trees is what separates Junior Angular developers from Architects.

### 1. EnvironmentInjector (The Global Tree)
This tree configures providers for the application environment. It is established during application bootstrap or when lazy-loading a route.

* **Root Injector:** Provided at bootstrap (`provideHttpClient()`, `provideRouter()`). Services with `@Injectable({ providedIn: 'root' })` live here. They are true Singletons.
* **Route Injectors:** Created when a lazy-loaded route defines its own `providers` array.

### 2. NodeInjector (The UI Tree)
This tree mirrors your Component DOM tree. Every time you create a component, Angular creates a `NodeInjector` for it.
* If a component defines a `providers: []` array in its `@Component` decorator, Angular creates a *new instance* of that service scoped entirely to that component and its children.

### Resolution Algorithm
When a component requests a dependency (e.g., `ApiService`), Angular looks for it in this exact order:
1. The Component's own NodeInjector.
2. The Parent Component's NodeInjector (recursively up the DOM tree).
3. The Route EnvironmentInjector (if lazy loaded).
4. The Root EnvironmentInjector.
5. If not found anywhere, it throws a `NullInjectorError`.

---

## 4. Configuring Providers

When you inject a service, you are asking the DI container for a token. You can tell the DI container exactly how to satisfy that token.

### Standard Provision (`useClass`)
```typescript
@Injectable({ providedIn: 'root' })
export class ApiService {}
```
By default, the token is the class type, and Angular instantiates that exact class.

### Overriding for Testing or Mocking
You can override what instance is returned for a given token.

```typescript
providers: [
  // When a component asks for ApiService, give it MockApiService instead
  { provide: ApiService, useClass: MockApiService }
]
```

### Providing Values (`useValue`)
Useful for passing configuration objects or environment variables.

```typescript
export const API_URL = new InjectionToken<string>('API_URL');

providers: [
  { provide: API_URL, useValue: 'https://api.ev-platform.com/v1' }
]
```

### Factory Providers (`useFactory`)
Useful when dependency creation requires complex logic or conditional setup.

```typescript
providers: [
  {
    provide: LoggerService,
    useFactory: (env: EnvironmentConfig) => {
      return env.production ? new DataDogLogger() : new ConsoleLogger();
    },
    deps: [EnvironmentConfig]
  }
]
```

---

## 5. The Modern `inject()` Function

In Angular v14, a massive paradigm shift occurred. The introduction of the `inject()` function allowed developers to request dependencies *outside* of the class constructor.

**Legacy Constructor Injection:**
```typescript
@Component({...})
export class ChargerComponent {
  constructor(
    private chargerService: ChargerService,
    private authService: AuthService,
    private router: Router
  ) {}
}
```

**Modern `inject()` Approach:**
```typescript
@Component({...})
export class ChargerComponent {
  private chargerService = inject(ChargerService);
  private authService = inject(AuthService);
  private router = inject(Router);
}
```

### Why `inject()` is Better:
1. **Inheritance without pain:** In constructor injection, if a child class extends a base class, the child must inject all of the parent's dependencies and pass them up via `super()`. With `inject()`, the base class handles its own dependencies natively.
2. **Reusable Functions:** You can write reusable functional code that injects services (like functional router guards or HTTP interceptors) without needing a class.

*Note: `inject()` can only be called during the **injection context** (the construction phase of a class). You cannot call it later inside `ngOnInit` or an event handler.*

---

## 6. Enterprise SaaS Case Study: Multi-Tenant Configuration

In our EV Charging Platform, we might deploy the same application for different enterprise tenants (e.g., "Tesla" vs "ChargePoint"). We need to inject different branding and feature flags depending on the tenant.

**`tenant.token.ts`**
```typescript
import { InjectionToken } from '@angular/core';

export interface TenantConfig {
  tenantId: string;
  primaryColor: string;
  enableBetaFeatures: boolean;
}

export const TENANT_CONFIG = new InjectionToken<TenantConfig>('TENANT_CONFIG');
```

**`main.ts`** (Bootstrapping dynamically based on URL)
```typescript
const host = window.location.hostname; // e.g., tesla.ev-platform.com
const tenantConfig = fetchTenantConfigSynchrnously(host); 

bootstrapApplication(AppComponent, {
  providers: [
    { provide: TENANT_CONFIG, useValue: tenantConfig }
  ]
});
```

**`header.component.ts`**
```typescript
@Component({
  selector: 'app-header',
  standalone: true,
  template: `<header [style.backgroundColor]="config.primaryColor">...</header>`
})
export class HeaderComponent {
  config = inject(TENANT_CONFIG);
}
```
By leveraging DI, our `HeaderComponent` is completely decoupled from *how* the tenant config is retrieved. It just asks the DI container for it.

---

## 7. Common Mistakes

### Beginner Mistakes
* **Circular Dependencies:** Service A injects Service B, and Service B injects Service A. The compiler crashes. Fix this by extracting shared logic into a Service C, or utilizing RxJS/Signals to decouple the flow.
* **Not understanding `@Injectable()`:** Forgetting to add `@Injectable({ providedIn: 'root' })` and wondering why the `NullInjectorError` appears.

### Architect Mistakes
* **Providing Singletons in `Component` providers:** Putting a service in the `providers: []` array of the `AppComponent` or a layout shell, thinking it makes it a singleton. If that component is destroyed and recreated, the state in that service is completely wiped out. True singletons must be provided in `providedIn: 'root'` or the `ApplicationConfig`.

---

## 8. Interview Questions

### Senior
**Q: Explain the difference between `ElementRef` and standard dependency injection.**
> A: Standard DI injects class instances (Services) from the logical Injector trees (Environment or Node). `ElementRef` is a special token provided exclusively by the NodeInjector. Angular dynamically creates it at runtime to wrap the physical DOM element associated with that specific component/directive instance, allowing direct DOM manipulation.

### Architect
**Q: How does Tree Shaking interact with `@Injectable({ providedIn: 'root' })` vs adding a service to a `providers` array?**
> A: When using `providedIn: 'root'`, the service tells Angular "If someone injects me, put me in the root injector." If no component ever injects the service, the build optimizer (Esbuild) can prove it is dead code and tree-shake it out of the production bundle. Conversely, if you explicitly add a service to an `EnvironmentProviders` array or a component's `providers` array, the compiler creates a hard reference to that class, preventing it from being tree-shaken, even if it is never actually used.

---

## 9. Summary
In this chapter, we conquered Angular's Dependency Injection system. We mapped the two injector hierarchies (Environment and Node), learned how to configure complex providers with Factories and Tokens, and adopted the modern `inject()` function for cleaner code. 

With the fundamentals complete, Part I is finished. In Chapter 6, we will enter Part II: Reactive State, beginning with Mastering RxJS in Angular.
