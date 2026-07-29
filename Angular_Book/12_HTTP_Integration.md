# Chapter 12: HTTP & API Integration

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Configure the modern `provideHttpClient` in a Standalone application.
* Write robust API services utilizing RxJS observables.
* Implement Functional HTTP Interceptors for Authentication and Global Error Handling.
* Understand the HTTP caching paradigm and retry logic for flakey network connections.

---

## 2. Introduction: The Angular HttpClient

Angular does not use the native browser `fetch` API. It uses its own highly specialized `HttpClient`. 

Why? Because `HttpClient`:
* Always returns **Observables**, enabling RxJS features like cancellation (aborting XHR requests instantly) and retry logic.
* Automatically parses JSON responses.
* Supports robust request/response interception for injecting JWT tokens or catching 401 Unauthorized errors globally.
* Can deeply integrate with Angular's Server-Side Rendering (SSR) transfer state.

---

## 3. Configuring the HttpClient (Modern Standalone API)

Historically, you imported `HttpClientModule` into your `AppModule`. In modern Angular, you use the `provideHttpClient` function in your `app.config.ts`.

**`app.config.ts`**
```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      // Uses the native fetch API under the hood (improves SSR performance)
      withFetch(), 
      // Registers our functional interceptors
      withInterceptors([authInterceptor, errorInterceptor])
    )
  ]
};
```

---

## 4. Writing an API Service

An API service should be an `@Injectable` singleton. It is responsible for making HTTP requests, catching errors, and mapping raw backend DTOs into frontend domain models if necessary.

**`charger.service.ts`**
```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, catchError, retry, throwError } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ChargerService {
  private http = inject(HttpClient);
  private apiUrl = '/api/v1/chargers'; // Ideally injected via Environment Token

  // Returns an Observable of Chargers
  getAll(): Observable<Charger[]> {
    return this.http.get<Charger[]>(this.apiUrl).pipe(
      // Automatically retry the request up to 2 times if it fails
      retry(2),
      catchError(this.handleError)
    );
  }

  // Create a new charger
  create(charger: Partial<Charger>): Observable<Charger> {
    return this.http.post<Charger>(this.apiUrl, charger).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: any) {
    console.error('API Error:', error);
    // Transform the error into a user-friendly message
    return throwError(() => new Error('Failed to communicate with the Charger API.'));
  }
}
```

---

## 5. Functional Interceptors

Interceptors sit between your `HttpClient` and the network. They intercept every outgoing HTTP Request and every incoming HTTP Response.

In Angular v15+, class-based interceptors were deprecated in favor of **Functional Interceptors**, which are significantly easier to write and use the `inject()` function.

### Case Study: The JWT Authentication Interceptor

In our EV Platform, almost every API request requires a Bearer token in the `Authorization` header. We should not manually add this to every request in our Services. An Interceptor can do it globally.

**`auth.interceptor.ts`**
```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  // If we have a token, clone the request and add the Authorization header
  if (token) {
    const clonedRequest = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    
    // Pass the cloned, modified request to the next handler
    return next(clonedRequest);
  }

  // If no token, pass the original request unchanged
  return next(req);
};
```

### Case Study: The Global Error Interceptor

What if a user's token expires, and the API starts returning `401 Unauthorized`? We need an interceptor to catch this globally, log the user out, and redirect them to the login page.

**`error.interceptor.ts`**
```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { AuthService } from '../auth.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      
      if (error.status === 401) {
        // Token expired or invalid
        authService.logout();
        router.navigate(['/auth/login']);
      } 
      else if (error.status === 403) {
        // RBAC denial
        router.navigate(['/unauthorized']);
      }

      // Re-throw the error so the local service can still catch it if it wants to
      return throwError(() => error);
    })
  );
};
```

---

## 6. HTTP Caching Strategies

Enterprise applications often fetch data that rarely changes (e.g., a list of Countries, Categories, or Charger Models). Re-fetching this data every time the user navigates between routes wastes bandwidth and slows down the UI.

We can cache these HTTP requests using RxJS `shareReplay`.

**`reference-data.service.ts`**
```typescript
@Injectable({ providedIn: 'root' })
export class ReferenceDataService {
  private http = inject(HttpClient);
  
  // Create an observable property, not a function!
  private categoriesCache$: Observable<Category[]> | null = null;

  getCategories(): Observable<Category[]> {
    // If the cache exists, return it instantly. No HTTP call made.
    if (this.categoriesCache$) {
      return this.categoriesCache$;
    }

    // Otherwise, make the HTTP call, and cache the result
    this.categoriesCache$ = this.http.get<Category[]>('/api/categories').pipe(
      // Cache the last 1 emitted value. 
      // Keep the cache alive even if all subscribers unsubscribe.
      shareReplay(1) 
    );

    return this.categoriesCache$;
  }
  
  clearCache() {
    this.categoriesCache$ = null;
  }
}
```

---

## 7. Common Mistakes

### Beginner Mistakes
* **Forgetting to Subscribe:** An `HttpClient.get()` does absolutely nothing until you call `.subscribe()` (or pipe it into an `AsyncPipe` or SignalStore effect). Observables are lazy.
* **Mutating the HTTP Request:** You cannot mutate the `HttpRequest` object inside an interceptor (e.g., `req.headers.set(...)`). The request object is strictly immutable. You must use `req.clone()` to create a new modified request.

### Architect Mistakes
* **Leaking Secrets:** Hardcoding API keys or base URLs into the Service. Always use an `InjectionToken` or Angular's `environment.ts` files to inject URLs dynamically, allowing CI/CD to swap them per environment (Dev, Staging, Prod).

---

## 8. Interview Questions

### Intermediate
**Q: How do you cancel an ongoing HTTP request in Angular?**
> A: Because `HttpClient` returns an Observable, you simply unsubscribe from it. If you are using an `AsyncPipe`, destroying the component automatically unsubscribes, triggering the browser to instantly abort the underlying XHR/fetch request. If managing it manually, you call `subscription.unsubscribe()`.

### Senior
**Q: Explain the order of execution when you register multiple HttpInterceptors.**
> A: Interceptors run in the exact order they are provided in the `withInterceptors([A, B, C])` array. 
> * For the outgoing Request, the order is `A -> B -> C -> Network`.
> * For the incoming Response, the observable stream flows backward: `Network -> C -> B -> A`.

---

## 9. Summary
In this chapter, we explored the Angular `HttpClient`. We learned how to build robust, retry-capable API services, and how to use RxJS `shareReplay` to cache static data. We also implemented modern Functional Interceptors to handle JWT Authentication and global error routing securely.

This completes **Part III**. In Chapter 13, we will enter the most advanced section of the book, **Part IV: Performance & Internals**, starting with an incredibly deep dive into the Change Detection mechanism.
