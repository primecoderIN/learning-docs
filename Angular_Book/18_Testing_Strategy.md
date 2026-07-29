# Chapter 18: Testing Strategy

## 1. Learning Objectives
By the end of this chapter, you will be able to:
* Differentiate between Unit, Component/Integration, and End-to-End (E2E) testing.
* Write robust isolated Unit Tests for Services, Pipes, and Signal Stores.
* Use `TestBed` to write Component tests that interact with the DOM.
* Mock dependencies effectively using `HttpTestingController` and DI overrides.
* Understand the paradigm shift from Protractor to modern E2E tools like Playwright and Cypress.

---

## 2. Introduction: The Testing Pyramid

Testing in enterprise software is not optional. A lack of tests in a massive Angular Monorepo guarantees that deploying a fix to the Billing module will secretly break the Admin module.

To maintain velocity safely, we rely on the **Testing Pyramid**:
1. **Unit Tests (The Base - 70%):** Fast, isolated tests targeting pure logic (Services, Pipes, Utility functions, NgRx Reducers).
2. **Component Tests (The Middle - 20%):** Tests that instantiate the Component class *and* its HTML template to verify DOM interactions.
3. **E2E Tests (The Peak - 10%):** Slow, comprehensive tests that spin up a real browser and click through the application exactly as a user would.

Angular provides the `TestBed` API for Component testing, and ships with **Jasmine/Karma** (or increasingly, **Jest/Vitest**) for Unit testing.

---

## 3. Isolated Unit Testing

Isolated tests do not use Angular's `TestBed`. They simply instantiate a TypeScript class using `new` and test its methods. They run in milliseconds.

### Testing a Pure Pipe
Because a pure Pipe has no external dependencies, it is incredibly easy to test.

**`kwh.pipe.spec.ts`**
```typescript
import { KwhPipe } from './kwh.pipe';

describe('KwhPipe', () => {
  let pipe: KwhPipe;

  beforeEach(() => {
    pipe = new KwhPipe();
  });

  it('should transform 5.1234 to "5.12 kWh"', () => {
    expect(pipe.transform(5.1234)).toBe('5.12 kWh');
  });

  it('should handle null values gracefully', () => {
    expect(pipe.transform(null)).toBe('0.00 kWh');
  });
});
```

---

## 4. Component Testing with `TestBed`

When you need to test if clicking a button actually updates the DOM, you must use `TestBed`. It creates a dynamically compiled Angular module environment specifically for your test.

### Case Study: Testing the Charger Badge

Let's test the `ChargerBadgeComponent` we built in Chapter 3. Since it is a Standalone Component, testing is significantly cleaner than in the legacy `NgModule` era.

**`charger-badge.component.spec.ts`**
```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ChargerBadgeComponent } from './charger-badge.component';

describe('ChargerBadgeComponent', () => {
  let component: ChargerBadgeComponent;
  let fixture: ComponentFixture<ChargerBadgeComponent>;

  beforeEach(async () => {
    // 1. Configure the Testing Module
    await TestBed.configureTestingModule({
      imports: [ChargerBadgeComponent] // Standalone components go in imports!
    }).compileComponents();

    // 2. Instantiate the Component and its DOM
    fixture = TestBed.createComponent(ChargerBadgeComponent);
    component = fixture.componentInstance;
  });

  it('should render "🟢 Ready" and apply correct CSS class when AVAILABLE', () => {
    // Arrange: Set the input (Modern Signal Input via componentRef)
    fixture.componentRef.setInput('status', 'AVAILABLE');
    
    // Act: Tell Angular to run change detection
    fixture.detectChanges();

    // Assert: Query the DOM
    const badgeElement: HTMLElement = fixture.nativeElement.querySelector('.badge');
    
    expect(badgeElement.textContent).toContain('Ready');
    expect(badgeElement.classList.contains('status-available')).toBeTrue();
  });
});
```

---

## 5. Mocking Dependencies (HTTP)

When testing a component or service that makes API calls, **you must never hit the real API**. Real APIs are slow, flakey, and mutate real data. You must mock the `HttpClient`.

Angular provides the `HttpTestingController` specifically for this.

**`charger.service.spec.ts`**
```typescript
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { ChargerService } from './charger.service';

describe('ChargerService', () => {
  let service: ChargerService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ChargerService,
        provideHttpClient(),
        provideHttpClientTesting() // Intercepts all HTTP requests!
      ]
    });

    service = TestBed.inject(ChargerService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Ensure no unexpected requests were made
    httpMock.verify(); 
  });

  it('should fetch a list of chargers', () => {
    const mockData = [{ id: '1', name: 'Tesla V3' }];

    // 1. Trigger the method
    service.getAll().subscribe(chargers => {
      expect(chargers.length).toBe(1);
      expect(chargers[0].name).toBe('Tesla V3');
    });

    // 2. Expect an HTTP GET request to a specific URL
    const req = httpMock.expectOne('/api/v1/chargers');
    expect(req.request.method).toBe('GET');

    // 3. Resolve the request with our mock data
    req.flush(mockData);
  });
});
```

---

## 6. End-to-End (E2E) Testing: The Modern Era

For years, Angular shipped with **Protractor** as the default E2E testing framework. It was notoriously flakey, slow, and relied on the deprecated WebDriver API. The Angular team officially deprecated Protractor.

Today, enterprise teams use one of two tools: **Cypress** or **Playwright**.

### Why Playwright?
Playwright (by Microsoft) is rapidly becoming the industry standard.
* **Speed:** It runs tests in parallel across multiple CPU cores natively.
* **Auto-Waiting:** It automatically waits for elements to be actionable (visible, enabled) before clicking them, eliminating the dreaded `setTimeout` hacks required in older tools.
* **Multiple Browsers:** It tests Chromium, WebKit (Safari), and Firefox simultaneously.

### Example Playwright Test (EV Login Flow)
E2E tests do not know that Angular exists. They interact with the browser exactly like a human driver.

**`login.spec.ts`**
```typescript
import { test, expect } from '@playwright/test';

test('User can log in and view the dashboard', async ({ page }) => {
  // Navigate to the app
  await page.goto('http://localhost:4200/auth/login');

  // Fill out the form
  await page.getByLabel('Email Address').fill('admin@ev-platform.com');
  await page.getByLabel('Password').fill('SecurePassword123');
  
  // Click submit
  await page.getByRole('button', { name: 'Log In' }).click();

  // Assert the URL changed to the dashboard
  await expect(page).toHaveURL(/.*dashboard/);

  // Assert the success toast appeared
  await expect(page.locator('.toast-success')).toHaveText('Welcome back, Admin!');
});
```

---

## 7. Common Mistakes

### Beginner Mistakes
* **Failing to call `fixture.detectChanges()`:** In Component tests, you set a property and query the DOM, but the DOM hasn't changed! Angular `TestBed` does not trigger change detection automatically when you modify a property. You must call `fixture.detectChanges()` manually.
* **Not verifying HttpMock:** Forgetting to call `httpMock.verify()` in `afterEach`. This hides bugs where a service accidentally fired a duplicate HTTP request that the test didn't account for.

### Architect Mistakes
* **Over-mocking in Component Tests:** Mocking every single child component in a Dashboard test (Shallow Testing). If you mock everything, you are testing nothing but implementation details. If `DashboardComponent` renders a `ChartComponent`, let the real `ChartComponent` render, but mock the global Data Service that feeds it.
* **E2E Test Bloat (The Ice Cream Cone Anti-Pattern):** Writing 500 E2E tests and only 10 Unit tests. E2E tests take minutes to run; Unit tests take milliseconds. If an E2E test fails, it's hard to know exactly which line of code broke. Only use E2E tests for "Happy Paths" (e.g., Login, Checkout, Submit Form). Use Unit tests for every edge case.

---

## 8. Interview Questions

### Intermediate
**Q: When would you use Isolated Unit Testing versus `TestBed` Component Testing?**
> A: Isolated Unit Testing involves instantiating a class using the `new` keyword, bypassing Angular entirely. It is significantly faster and should be used for pure logic: Pipes, utility functions, state Reducers, and Services (if you mock their dependencies manually). `TestBed` should only be used when you need to test the interaction between the TypeScript class and the HTML template (the DOM), or when a service relies heavily on Angular's Dependency Injection system (like Interceptors).

### Architect
**Q: Explain how to properly test a Smart (Container) Component versus a Dumb (Presentational) Component.**
> A: A Dumb component (e.g., a Button) should be tested by manipulating its Inputs and verifying the HTML output, and simulating clicks to ensure it emits the correct Outputs. 
> 
> A Smart component (e.g., a Dashboard) should be tested by mocking its injected Services. You provide a mock `ChargerService` that returns a hardcoded RxJS Observable. You then run change detection and verify that the Smart component correctly delegates that mocked data down to its child components. You never test the actual API logic in the Smart component's test.

---

## 9. Summary
In this chapter, we conquered Angular Testing. We explored the Testing Pyramid, wrote isolated tests for pure logic, used `TestBed` and `fixture.componentRef.setInput()` to test DOM interactions, mocked HTTP requests with `HttpTestingController`, and replaced the deprecated Protractor with Playwright for modern E2E testing.

In Chapter 19, we will dive into **Security Best Practices**, learning how Angular protects applications from cross-site scripting (XSS) and cross-site request forgery (CSRF).
