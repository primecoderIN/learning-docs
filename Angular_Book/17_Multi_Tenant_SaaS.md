# Chapter 17: Multi-Tenant SaaS Patterns

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Architect a frontend capable of dynamically adapting to different enterprise tenants.
* Implement dynamic CSS variable injection for White-Labeling.
* Design a robust, signal-based Feature Flag system.
* Implement Role-Based Access Control (RBAC) at the route and component levels using Structural Directives.

---

## 2. Introduction: The SaaS Paradigm

In a true B2B SaaS platform, you do not deploy a separate copy of your code for every client. You deploy a single instance of the application (or a single Monorepo configuration) that dynamically adapts itself based on *who* is logging in.

In our **EV Charging Platform**, we might have two enterprise tenants: "ChargePoint" and "Tesla".
When a user logs in, the Angular application must instantly:
1. Change the primary brand colors (White Labeling).
2. Enable or disable specific features (e.g., Tesla has "Autocharge", ChargePoint does not).
3. Enforce strict permissions based on the user's role *within that specific tenant*.

---

## 3. White-Labeling (Dynamic Theming)

Historically, theming an Angular application meant providing multiple SCSS files and compiling entirely different bundles for each tenant (e.g., `ng build --configuration=tesla`). This is an archaic, unscalable approach if you have 100 tenants.

The modern Enterprise standard is **CSS Custom Properties (Variables)**.

### Step 1: Define Global CSS Variables
In your global `styles.scss`, define the default theme using variables.

```css
:root {
  --primary-color: #0052cc;
  --secondary-color: #172b4d;
  --font-family: 'Inter', sans-serif;
  --logo-url: url('/assets/default-logo.svg');
}
```

### Step 2: Bind Components to Variables
Inside your Angular components, use the variables. View Encapsulation will still work perfectly.

```css
button.primary {
  background-color: var(--primary-color);
  font-family: var(--font-family);
}
```

### Step 3: Inject the Tenant Theme at Runtime
When the user authenticates, the API should return a `TenantConfig` object. We use an Angular `APP_INITIALIZER` or the `AuthService` to dynamically rewrite the CSS variables on the root `document`.

```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  
  applyTenantTheme(config: TenantThemeConfig) {
    const root = document.documentElement;
    
    root.style.setProperty('--primary-color', config.primaryColor);
    root.style.setProperty('--secondary-color', config.secondaryColor);
    
    // Optional: Inject a custom Google Font link dynamically
    this.injectFont(config.fontFamilyUrl);
  }
}
```
Now, one single compiled Angular bundle can adapt to an infinite number of tenant brandings.

---

## 4. Feature Flags

Not all tenants pay for all features. Feature Flags allow you to conditionally render UI elements based on the tenant's subscription tier or configuration.

### The Signal-Based Feature Flag Service
```typescript
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  // A map of feature keys to booleans
  private flags = signal<Record<string, boolean>>({});

  loadFlags(tenantFlags: Record<string, boolean>) {
    this.flags.set(tenantFlags);
  }

  // Returns a computed signal so the UI updates instantly if a flag changes
  hasFeature(featureKey: string) {
    return computed(() => !!this.flags()[featureKey]);
  }
}
```

### Usage in the Template
```typescript
export class DashboardComponent {
  private features = inject(FeatureFlagService);
  
  // Expose the computed signal to the template
  enableAdvancedAnalytics = this.features.hasFeature('ADVANCED_ANALYTICS');
}
```
```html
@if (enableAdvancedAnalytics()) {
  <app-heavy-analytics-chart />
}
```
> **Architect Tip:** Combine Feature Flags with `@defer` (Chapter 15)! If the tenant doesn't have the Advanced Analytics feature, `@defer` ensures they never even download the JavaScript chunk for the chart.

---

## 5. Role-Based Access Control (RBAC)

RBAC controls what a specific user can do within the application (e.g., an Admin can delete chargers, a Viewer can only see them). 

### 1. Route-Level Protection
As discussed in Chapter 10, use Functional Route Guards to prevent users from navigating to unauthorized URLs.

### 2. Component-Level Protection (The `*hasRole` Directive)
It is terrible UX to show a "Delete Charger" button, let the user click it, and then throw a 403 Forbidden error. The button shouldn't exist in the DOM at all.

We can create a Structural Directive that acts exactly like `*ngIf`, but checks permissions.

**`has-role.directive.ts`**
```typescript
import { Directive, Input, TemplateRef, ViewContainerRef, inject, effect } from '@angular/core';
import { AuthService } from '../auth.service';

@Directive({
  selector: '[appHasRole]',
  standalone: true
})
export class HasRoleDirective {
  private templateRef = inject(TemplateRef<any>);
  private viewContainer = inject(ViewContainerRef);
  private authService = inject(AuthService);

  private hasView = false;

  @Input() set appHasRole(requiredRole: string) {
    // Check if the user's role matches
    const isAuthorized = this.authService.userRoles().includes(requiredRole);

    if (isAuthorized && !this.hasView) {
      // Create the element in the DOM
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (!isAuthorized && this.hasView) {
      // Completely remove the element from the DOM
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

**Usage:**
```html
<button *appHasRole="'admin'" class="danger">
  Delete Tenant Network
</button>
```

---

## 6. Common Mistakes

### Beginner Mistakes
* **Hiding UI via CSS:** Using `[style.display]="hasRole ? 'block' : 'none'"` to hide an admin button. The button is still in the DOM! A savvy user can open Chrome DevTools, remove the `display: none`, and click the button. You must use structural directives (or `@if`) to completely remove the element from the DOM, and you must always re-verify permissions on the Backend API.

### Senior Mistakes
* **Hardcoding Tenant Logic:** Writing `if (tenantId === 'tesla') { doSomething() }` inside your components. This breaks the Open-Closed Principle. If you onboard a new tenant tomorrow, you have to rewrite your code. Always rely on generic Feature Flags provided by the backend (e.g., `if (hasFeature('AUTOCHARGE'))`).

---

## 7. Interview Questions

### Senior
**Q: Why are CSS Custom Properties (Variables) vastly superior to SCSS variables for white-labeling a SaaS application?**
> A: SCSS variables are resolved at compile time. If you use SCSS variables for themes, you must compile a separate production bundle for every single tenant, which breaks CI/CD scaling. CSS Custom Properties are resolved by the browser at runtime. This allows you to compile a single Angular bundle and inject the tenant's specific colors into the DOM via JavaScript upon authentication.

### Architect
**Q: How do you handle Route-level Feature Flags without causing visual flashing or redirect loops?**
> A: You should create a `FeatureFlagGuard`. Before the route activates, the guard checks the `FeatureFlagService`. If the tenant lacks the feature, the guard returns a `UrlTree` directing them to a 404 or "Upgrade your plan" page. Crucially, the feature flags must be fetched during the `APP_INITIALIZER` phase or as part of the initial Authentication payload, ensuring they are populated in memory *before* the Angular Router attempts its first navigation.

---

## 8. Summary
In this chapter, we explored the critical requirements of a Multi-Tenant SaaS application. We learned how to White-Label our platform using dynamic runtime CSS variables, implemented a Signal-based Feature Flag engine, and built a secure Structural Directive to handle granular RBAC permissions.

In Chapter 18, we will explore **Testing Strategy**, learning how to properly test these complex enterprise architectures using Unit Tests and modern E2E tools like Playwright.
