# Appendix 6: Routing & Guards Interview Questions

This section covers the Angular Router, focusing on bundle optimization via Lazy Loading, modern Functional Route Guards, and advanced data passing mechanisms.

---

## Junior Level Questions

### 1. What is the Angular Router and how do you define a basic route?
**Answer:**
The Angular Router is the core library (`@angular/router`) that enables navigation from one view to the next as users perform application tasks, without reloading the entire page (SPA).
You define a route by mapping a URL path to a specific Component.

```typescript
export const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: '', redirectTo: '/home', pathMatch: 'full' } // Default fallback
];
```

### 2. Why is `pathMatch: 'full'` required on empty path redirects?
**Answer:**
In Angular, the router checks if the requested URL *starts with* the path defined in the route array. Since every single URL string technically starts with an empty string `''`, a route like `{ path: '', redirectTo: 'home' }` would match *every* URL (e.g., `/about`, `/users`).
Adding `pathMatch: 'full'` tells the router that the URL must equal the empty string *exactly* to trigger the redirect.

### 3. How do you pass data in the URL (Route Parameters)?
**Answer:**
You define a parameter in the route config using a colon (`:`).
```typescript
{ path: 'user/:id', component: UserDetailComponent }
```
To read it, you historically inject `ActivatedRoute` and subscribe to `route.params`. In modern Angular, you can use `withComponentInputBinding()` to read it directly as an `@Input()` or Signal `input()`.

---

## Mid-Level Questions

### 4. What is Lazy Loading, and how do you implement it in modern Standalone Angular?
**Answer:**
Lazy Loading is the process of splitting your application into multiple JavaScript chunks. Instead of downloading all 500 components on the initial page load, the browser only downloads the specific chunk when the user navigates to that route.

In modern Standalone Angular, you use the `loadComponent` property with a dynamic `import()`.
```typescript
export const routes: Routes = [
  { 
    path: 'admin', 
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent) 
  }
];
```
For lazy-loading an entire branch of routes, use `loadChildren`.

### 5. What are Route Guards, and what are the primary types?
**Answer:**
Route Guards are interfaces (or functions) that allow or deny navigation to a route based on custom logic (e.g., Authentication).
1. `canActivate`: Prevents a user from entering a route.
2. `canDeactivate`: Prevents a user from leaving a route (e.g., "You have unsaved changes!").
3. `canMatch` (replaced `canLoad`): Prevents the router from even matching the route, often used to prevent downloading lazy-loaded chunks for unauthorized users.
4. `resolve`: Delays rendering the component until required data is fetched from an API.

### 6. Explain the modern "Functional Route Guards" introduced in Angular v15.
**Answer:**
Historically, Guards were class-based `@Injectable()` services that implemented interfaces like `CanActivate`. This required a lot of boilerplate.
Functional guards are simple JavaScript functions that utilize the `inject()` function to access services. They are significantly cleaner and easier to write.

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) return true;
  return router.parseUrl('/login'); // Redirect
};
```

---

## Senior Level Questions

### 7. What is the `withComponentInputBinding()` feature?
**Answer:**
Introduced in Angular v16, this configuration tells the Router to automatically map URL route parameters (`/user/:id`), query parameters (`?sort=asc`), and static route `data` directly to the Component's `@Input()` properties or Signal `input()`s.
This completely eliminates the need to inject the `ActivatedRoute` service and manage RxJS subscriptions just to read the URL.

```typescript
// If the URL is /users/123?sort=asc
export class UserComponent {
  id = input.required<string>(); // Receives '123'
  sort = input<string>();        // Receives 'asc'
}
```

### 8. Explain the danger of memory leaks in Route Guards that return Observables.
**Answer:**
If a Route Guard returns an `Observable` (e.g., waiting for an HTTP call to check the user's role), the Angular Router subscribes to that Observable and waits for it to **complete** or emit the first value.
If you return a stream that never completes (like a `BehaviorSubject`), the Router will wait infinitely. The navigation will hang, the URL won't change, and the application will appear frozen.
**Solution:** Always pipe the Observable through `take(1)` or `first()` in a Guard to ensure it completes immediately after emitting.

### 9. What is a "Preloading Strategy", and why use `PreloadAllModules`?
**Answer:**
By default, if you strictly lazy-load every route, the user will experience a slight network delay every time they click a navigation link because the browser has to download the chunk.
A Preloading Strategy tells Angular to download lazy-loaded chunks in the background *after* the initial application has booted and the user is idle.
`PreloadAllModules` eagerly downloads every chunk in the background. For massive enterprise apps, you might write a custom strategy that only preloads specific high-traffic modules.

---

## Architect Level Questions

### 10. How does the Angular Router decide whether to instantiate a new Component or reuse an existing one? Can you override this?
**Answer:**
By default, if you navigate from `/chargers/1` to `/chargers/2`, the Router sees that the component configuration hasn't changed—only the route parameter changed. To save performance, it **reuses** the existing component instance. The `constructor` and `ngOnInit` will **not** fire again. (This is why you must subscribe to `ActivatedRoute.params` or use Signal Inputs to detect the ID change).

If you want to force the component to destroy and recreate itself, you must implement a custom `RouteReuseStrategy`. By overriding the `shouldReuseRoute` method, you can instruct Angular to always destroy the component when the parameter changes.

### 11. What is the difference between `canActivate` and `canMatch`, and which one should you use for a lazy-loaded route?
**Answer:**
* `canActivate` determines if a user can access a route, but it executes *after* the Router has matched the URL. If the route is lazy-loaded (`loadChildren`), the Router will download the JavaScript chunk first, and *then* run `canActivate`. This exposes your proprietary code (and increases network usage) for unauthorized users.
* `canMatch` executes *before* the Router even acknowledges the route exists. If `canMatch` returns false, the Router skips the route configuration entirely. This means for a lazy-loaded route, the JavaScript chunk will **not** be downloaded.
**Architect Rule:** Always use `canMatch` for protecting lazy-loaded modules.

### 12. You are building an Enterprise Shell architecture with multiple `<router-outlet>` tags. How do you implement "Named Outlets" (Auxiliary Routes)?
**Answer:**
Sometimes you need to render two completely independent routes on the screen simultaneously (e.g., a main dashboard, and a persistent chat window).
You achieve this using Named Outlets.

**HTML:**
```html
<router-outlet></router-outlet> <!-- Primary -->
<router-outlet name="chat"></router-outlet> <!-- Auxiliary -->
```
**Route Config:**
```typescript
{ path: 'support', component: ChatComponent, outlet: 'chat' }
```
**Navigation:**
The URL syntax becomes complex: `/dashboard(chat:support)`. This tells the router to load the Dashboard in the primary outlet, and the Support component in the 'chat' outlet simultaneously.
