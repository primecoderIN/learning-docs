# Chapter 10: Advanced Routing Architecture

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Configure nested routing for Complex Layouts (Shell architecture).
* Implement Lazy Loading to drastically reduce initial bundle size.
* Protect routes using modern Functional Route Guards (`canActivate`).
* Bind route parameters directly to Component Signal Inputs (`withComponentInputBinding`).
* Understand the Router lifecycle and navigation events.

---

## 2. Introduction to the Angular Router

The Angular Router is one of the most sophisticated routing libraries in the frontend ecosystem. Unlike React Router, which is a third-party library, the Angular Router is a core framework module tightly integrated with Dependency Injection, the Compiler, and the Change Detection cycle.

It handles URL parsing, lazy loading of JavaScript chunks, component instantiation, and complex asynchronous guard checks—all before a single pixel is rendered on the screen.

---

## 3. Shell Architecture (Nested Routing)

In Enterprise SaaS applications, the UI is rarely a single full-screen page. It usually consists of a Shell (a fixed Sidebar and Header) and a dynamic Main Content area. We achieve this using **Nested Routing** and multiple `<router-outlet>` tags.

### The EV Platform Layout Structure

**`app.routes.ts`**
```typescript
export const routes: Routes = [
  {
    path: 'auth',
    component: AuthLayoutComponent, // Full screen, no sidebar
    children: [
      { path: 'login', component: LoginComponent }
    ]
  },
  {
    path: 'dashboard',
    component: MainLayoutComponent, // Contains Sidebar and Header
    children: [
      { path: 'overview', component: OverviewComponent }, // Renders inside MainLayout
      { path: 'chargers', component: ChargerListComponent }
    ]
  },
  { path: '', redirectTo: 'dashboard/overview', pathMatch: 'full' }
];
```

**`main-layout.component.html`**
```html
<div class="enterprise-layout">
  <app-sidebar></app-sidebar>
  <div class="main-column">
    <app-header></app-header>
    <main>
      <!-- Child routes (overview, chargers) render exactly here! -->
      <router-outlet></router-outlet> 
    </main>
  </div>
</div>
```

---

## 4. Lazy Loading (Bundle Splitting)

If your application has 500 components, you **must not** load them all when the user first visits the login page. That would result in a massive multi-megabyte JavaScript payload, ruining performance.

**Lazy Loading** splits your application into smaller chunks. When a user navigates to a route, Angular's router fetches the required JavaScript chunk over the network on the fly.

### Modern Standalone Lazy Loading
With Standalone components, lazy loading is incredibly clean. We use dynamic `import()` statements.

```typescript
export const routes: Routes = [
  {
    path: 'reports',
    // The browser only downloads this JS file if the user navigates to /reports
    loadComponent: () => import('./features/reports/reports.component')
                           .then(m => m.ReportsComponent)
  }
];
```

> **Architect Tip:** You can also lazy-load entire routing branches using `loadChildren`.

```typescript
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes')
                          .then(m => m.adminRoutes)
  }
```

---

## 5. Functional Route Guards

In Enterprise applications, you must prevent unauthorized users from accessing specific URLs. Angular provides Route Guards for this purpose. 

In modern Angular (v15+), Class-based guards were deprecated in favor of **Functional Route Guards**. They are much less boilerplate-heavy and utilize the `inject()` function.

### Case Study: Protecting the EV Admin Panel

Let's write a guard that checks if the user is authenticated AND has the `admin` role. If not, we redirect them to the login page.

**`admin.guard.ts`**
```typescript
import { inject } from '@angular/core';
import { Router, CanActivateFn } from '@angular/router';
import { AuthService } from '../core/auth.service';

export const adminGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated() && authService.hasRole('admin')) {
    return true; // Navigation allowed
  }

  // Navigation denied, redirect to login
  return router.createUrlTree(['/auth/login']); 
};
```

**Applying the Guard in `app.routes.ts`**
```typescript
  {
    path: 'admin',
    canActivate: [adminGuard],
    loadChildren: () => import('./admin.routes').then(m => m.routes)
  }
```
*Note: Guards can return a `boolean`, an `UrlTree` (for redirects), a `Promise`, or an `Observable` (if the auth check requires an API call).*

---

## 6. Route Parameters as Component Inputs

Historically, reading the URL (e.g., `/chargers/123`) required injecting the `ActivatedRoute` service and subscribing to its observables.

Modern Angular allows you to bind route parameters, query parameters, and route data **directly to Component Inputs/Signals**.

### 1. Enable the Feature
You must explicitly enable this in your `app.config.ts`.

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding())
  ]
};
```

### 2. Read the Parameter in the Component
Assume the route is `/chargers/:chargerId`.

```typescript
@Component({
  template: `<h1>Viewing Charger {{ chargerId() }}</h1>`
})
export class ChargerDetailComponent {
  // The router automatically pushes the URL param into this Signal Input!
  chargerId = input.required<string>(); 

  // You can immediately use it in computed signals or effects!
  endpoint = computed(() => `/api/chargers/${this.chargerId()}`);
}
```
This entirely removes the need to inject `ActivatedRoute` or manage RxJS subscriptions for URL parameters.

---

## 7. Common Mistakes

### Beginner Mistakes
* **Forgetting `pathMatch: 'full'` on empty routes:** If you have `{ path: '', redirectTo: 'home' }` without `pathMatch: 'full'`, it will match *every single route* because every URL starts with an empty string. You will get trapped in an infinite redirect loop.
* **Leaking memory in guards:** If a functional guard returns an `Observable` (e.g., waiting for an HTTP call to check roles) and that observable never completes (like a `BehaviorSubject`), the Router will wait forever and the navigation will hang permanently. Use `.pipe(take(1))` to ensure the observable completes.

### Architect Mistakes
* **Not configuring a Preloading Strategy:** If you strictly lazy-load everything, the user will experience a slight network delay every time they click a new menu item. For large apps, use `PreloadAllModules` or build a custom `NetworkAwarePreloader` that fetches lazy chunks in the background *after* the initial app has loaded.

```typescript
provideRouter(routes, withPreloading(PreloadAllModules))
```

---

## 8. Interview Questions

### Intermediate
**Q: What is the difference between `loadComponent` and `loadChildren` in modern routing?**
> A: `loadComponent` is used to lazy-load a single Standalone Component file. `loadChildren` is used to lazy-load an entirely separate routing configuration array (a branch of routes), which allows you to modularize massive sections of an application.

### Senior
**Q: How does the Router decide whether to instantiate a new component or reuse an existing one when the URL changes?**
> A: By default, if you navigate from `/chargers/1` to `/chargers/2`, the Router will *reuse* the existing `ChargerDetailComponent` instance, because the route configuration hasn't changed (only the parameter changed). The constructor and `ngOnInit` will **not** fire again. This is why binding parameters to Signal Inputs (or subscribing to `ActivatedRoute.params`) is critical. If you want to force the component to destroy and recreate, you must implement a custom `RouteReuseStrategy`.

---

## 9. Summary
In this chapter, we conquered the Angular Router. We built a nested Shell architecture for our SaaS platform, utilized dynamic `import()` for lazy loading, secured routes with functional guards, and streamlined component data flow using `withComponentInputBinding`.

In Chapter 11, we will tackle the most complex and powerful feature of Angular: **Enterprise Form Design**.
