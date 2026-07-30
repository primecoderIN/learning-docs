# Appendix 5: Forms & Validation Interview Questions

This section covers data entry in Angular, contrasting Template-Driven and Reactive Forms, focusing heavily on Typed Reactive Forms, complex FormArrays, and Custom Validation strategies.

---

## Junior Level Questions

### 1. What is the fundamental difference between Template-Driven Forms and Reactive Forms?
**Answer:**
* **Template-Driven Forms:** Rely heavily on the HTML template using `[(ngModel)]`. They are asynchronous, mutable, and the form model is automatically created by Angular based on the HTML directives. They are easy for simple forms but hard to test.
* **Reactive Forms:** The form model (a tree of `FormControl`, `FormGroup`, etc.) is explicitly defined in the TypeScript class. They are completely synchronous, immutable, strictly typed (in v14+), and heavily rely on RxJS (`valueChanges`). They are the standard for Enterprise applications.

### 2. What are the three core building blocks of Reactive Forms?
**Answer:**
1. **`FormControl`:** Tracks the value and validation status of an individual input field.
2. **`FormGroup`:** Tracks the value and validation status of a group of controls, representing them as an object.
3. **`FormArray`:** Tracks an indexed array of controls (or groups), allowing you to dynamically add or remove controls at runtime.

### 3. How do you disable a Reactive Form control?
**Answer:**
You should not use the `[disabled]` HTML attribute when using Reactive Forms, as Angular will throw a warning in the console. Instead, you disable the control in TypeScript:
```typescript
this.myControl.disable();
this.myControl.enable();

// Or upon creation:
const email = new FormControl({ value: 'test@test.com', disabled: true });
```

---

## Mid-Level Questions

### 4. What is the difference between `form.value` and `form.getRawValue()`?
**Answer:**
This is a critical distinction in Reactive Forms.
* `form.value`: Returns an object containing only the values of the **enabled** controls. If a control is disabled, its key/value pair is completely omitted from the resulting object.
* `form.getRawValue()`: Ignores the disabled state entirely and returns the values of **all** controls in the form group. This is usually what you want when submitting data to an API.

### 5. What are Typed Forms (introduced in Angular v14)?
**Answer:**
Prior to Angular 14, `form.value` returned `any`, meaning the compiler couldn't stop you from assigning a string to a control meant for numbers.
Typed Forms introduced strict generics to the forms API. A `FormGroup` now perfectly infers its shape, and `form.value` returns a strictly typed interface. 

```typescript
const form = new FormGroup({
  age: new FormControl<number>(25, { nonNullable: true })
});

// TypeScript Error: Type 'string' is not assignable to type 'number'
form.patchValue({ age: 'twenty-five' }); 
```

### 6. What is the `NonNullableFormBuilder`?
**Answer:**
By default, if you call `form.reset()`, all controls reset to `null` (because in standard HTML, an empty input is null/empty string). This forces all your strict types to become `string | null`.
The `NonNullableFormBuilder` (or passing `{ nonNullable: true }` to a FormControl) guarantees that the control can never be null. When `reset()` is called, it reverts to its initial default value instead of `null`.

```typescript
private fb = inject(NonNullableFormBuilder);

form = this.fb.group({
  name: ['Default Name'] // Type is strictly `string`, not `string | null`
});
```

### 7. How do you create a dynamic list of inputs where the user can click "Add Row"?
**Answer:**
You must use a `FormArray`. You bind it in the template using `formArrayName`, and iterate over its `.controls` array.

```typescript
users = new FormArray([
  new FormControl('Alice')
]);

addUser() {
  this.users.push(new FormControl(''));
}
```
```html
<div [formGroup]="form">
  <div formArrayName="users">
    <div *ngFor="let user of users.controls; let i=index">
      <input [formControlName]="i">
    </div>
  </div>
</div>
```

---

## Senior Level Questions

### 8. Explain how to write a Custom Synchronous Validator.
**Answer:**
A custom validator is just a function that takes an `AbstractControl` and returns an object containing validation errors if it fails, or `null` if it passes.

```typescript
export function noWhitespaceValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const isWhitespace = (control.value || '').trim().length === 0;
    const isValid = !isWhitespace;
    
    return isValid ? null : { whitespace: 'Value cannot be empty' };
  };
}
```

### 9. Explain how to write an Asynchronous Validator (e.g., checking if an email is taken in a database).
**Answer:**
An Async Validator returns an `Observable<ValidationErrors | null>`. It must be passed as the *third* argument when creating a FormControl.

```typescript
export function emailTakenValidator(api: ApiService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return api.checkEmail(control.value).pipe(
      map(isTaken => (isTaken ? { emailTaken: true } : null)),
      catchError(() => of(null)) // Must catch errors so the stream doesn't die!
    );
  };
}

// Usage: new FormControl('', [SyncValidators], [AsyncValidators]);
```

### 10. Why is `updateOn: 'blur'` critical for performance when using Async Validators?
**Answer:**
By default, a `FormControl` triggers validation on every single keystroke (`updateOn: 'change'`). If you attach an Async Validator that makes a database query, the user typing "hello@test.com" will trigger 14 separate HTTP requests to your backend, overwhelming the database.
You solve this either by adding RxJS `timer(500)` logic inside the validator to debounce it, or much simpler, by setting `{ updateOn: 'blur' }` when configuring the FormControl, meaning the HTTP request only fires when the user clicks out of the input box.

### 11. How do you implement Cross-Field Validation (e.g., Password and Confirm Password must match)?
**Answer:**
You cannot apply the validator to the individual `FormControl` because it doesn't have access to its sibling controls. You must apply the validator to the parent `FormGroup`.

```typescript
export const matchPasswordValidator: ValidatorFn = (control: AbstractControl) => {
  const password = control.get('password');
  const confirm = control.get('confirmPassword');

  if (password && confirm && password.value !== confirm.value) {
    // We attach the error to the confirm control manually
    confirm.setErrors({ mismatch: true });
    return { mismatch: true }; 
  }
  return null;
};

// Applied here:
const form = new FormGroup({ password: [], confirmPassword: [] }, { validators: matchPasswordValidator });
```

---

## Architect Level Questions

### 12. In a massive enterprise form (e.g., a 10-step wizard with 200 fields), how do you prevent the `valueChanges` observable from causing massive UI stuttering?
**Answer:**
When a massive form tree uses `valueChanges`, changing a single deeply nested child control triggers the `valueChanges` of its parent `FormGroup`, all the way up to the root `FormGroup`. If you have heavy RxJS logic subscribed to the root `form.valueChanges`, it will execute on every single keystroke.
To solve this:
1. Use `{ emitEvent: false }` when calling `patchValue` if you don't want to trigger the observers.
2. Pipe the root `valueChanges` through `debounceTime(300)` and `distinctUntilChanged()` so processing only happens when the user pauses.
3. Break the massive `FormGroup` down. The 10 steps should be 10 isolated `FormGroups` stored in an NgRx Store, rather than one monolithic 200-field `FormGroup` in memory.

### 13. You are building a reusable, complex Custom Form Component (like a multi-select dropdown with an API search). How do you seamlessly integrate it with a parent component's `FormGroup`?
**Answer:**
You must implement the `ControlValueAccessor` interface and provide it via the `NG_VALUE_ACCESSOR` token. 
This tells Angular: *"Treat this complex Angular Component exactly like a native HTML `<input>`."*
When the parent component uses `<app-multi-select formControlName="tags">`, Angular automatically calls `writeValue()` on the custom component to pass data in, and the custom component calls `registerOnChange()` to pass data back out to the parent's `FormGroup`. This abstracts all the internal complexity of the custom control away from the parent component.
