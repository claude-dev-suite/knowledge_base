# Angular Forms

> Official Documentation: https://angular.dev/guide/forms

## Reactive Forms

Reactive forms provide a model-driven approach to handling form inputs. The form model is defined in the component class.

### Setup

```typescript
// app.config.ts - no special setup needed for standalone
// Just import ReactiveFormsModule in each component that uses forms

import { Component } from '@angular/core';
import { ReactiveFormsModule, FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-login',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <div>
        <label for="email">Email</label>
        <input id="email" formControlName="email" type="email" />
        @if (loginForm.controls.email.hasError('required') && loginForm.controls.email.touched) {
          <span class="error">Email is required</span>
        }
        @if (loginForm.controls.email.hasError('email') && loginForm.controls.email.touched) {
          <span class="error">Invalid email format</span>
        }
      </div>
      <div>
        <label for="password">Password</label>
        <input id="password" formControlName="password" type="password" />
        @if (loginForm.controls.password.hasError('required') && loginForm.controls.password.touched) {
          <span class="error">Password is required</span>
        }
        @if (loginForm.controls.password.hasError('minlength') && loginForm.controls.password.touched) {
          <span class="error">Password must be at least 8 characters</span>
        }
      </div>
      <label>
        <input type="checkbox" formControlName="rememberMe" />
        Remember me
      </label>
      <button type="submit" [disabled]="loginForm.invalid">Login</button>
    </form>
  `
})
export class LoginComponent {
  loginForm = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)]),
    rememberMe: new FormControl(false)
  });

  onSubmit(): void {
    if (this.loginForm.valid) {
      const { email, password, rememberMe } = this.loginForm.value;
      console.log('Login:', email, password, rememberMe);
    }
  }
}
```

---

## Typed Forms

Angular 14+ provides strict typing for reactive forms using `NonNullableFormBuilder`.

### NonNullableFormBuilder

```typescript
import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, NonNullableFormBuilder, Validators } from '@angular/forms';

interface UserFormValue {
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user' | 'editor';
}

@Component({
  selector: 'app-user-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="name" placeholder="Name" />
      <input formControlName="email" type="email" placeholder="Email" />
      <input formControlName="age" type="number" placeholder="Age" />
      <select formControlName="role">
        <option value="admin">Admin</option>
        <option value="user">User</option>
        <option value="editor">Editor</option>
      </select>
      <button type="submit">Submit</button>
      <button type="button" (click)="form.reset()">Reset</button>
    </form>
  `
})
export class UserFormComponent {
  private fb = inject(NonNullableFormBuilder);

  form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
    age: [0, [Validators.required, Validators.min(0), Validators.max(150)]],
    role: ['user' as const, Validators.required]
  });

  onSubmit(): void {
    if (this.form.valid) {
      // Fully typed - TypeScript knows the shape
      const value = this.form.getRawValue();
      // value.name is string, value.age is number, etc.
      console.log(value);
    }
  }
}
```

### Type-Safe Form Controls

```typescript
// With NonNullableFormBuilder, controls never return null after reset
const nameControl = this.fb.control('John');
const value: string = nameControl.value;  // string, not string | null

// For nullable controls, explicitly mark as nullable
const optionalField = new FormControl<string | null>(null);
```

---

## FormGroup with Nested Groups

```typescript
@Component({
  selector: 'app-registration',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <fieldset formGroupName="personalInfo">
        <legend>Personal Information</legend>
        <input formControlName="firstName" placeholder="First Name" />
        <input formControlName="lastName" placeholder="Last Name" />
        <input formControlName="email" type="email" placeholder="Email" />
      </fieldset>

      <fieldset formGroupName="address">
        <legend>Address</legend>
        <input formControlName="street" placeholder="Street" />
        <input formControlName="city" placeholder="City" />
        <input formControlName="state" placeholder="State" />
        <input formControlName="zip" placeholder="ZIP Code" />
      </fieldset>

      <fieldset formGroupName="credentials">
        <legend>Account</legend>
        <input formControlName="username" placeholder="Username" />
        <input formControlName="password" type="password" placeholder="Password" />
        <input formControlName="confirmPassword" type="password" placeholder="Confirm Password" />
      </fieldset>

      <button type="submit" [disabled]="registrationForm.invalid">Register</button>
    </form>
  `
})
export class RegistrationComponent {
  private fb = inject(NonNullableFormBuilder);

  registrationForm = this.fb.group({
    personalInfo: this.fb.group({
      firstName: ['', Validators.required],
      lastName: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    }),
    address: this.fb.group({
      street: ['', Validators.required],
      city: ['', Validators.required],
      state: ['', Validators.required],
      zip: ['', [Validators.required, Validators.pattern(/^\d{5}(-\d{4})?$/)]]
    }),
    credentials: this.fb.group({
      username: ['', [Validators.required, Validators.minLength(4)]],
      password: ['', [Validators.required, Validators.minLength(8)]],
      confirmPassword: ['', Validators.required]
    }, { validators: passwordMatchValidator })
  });

  onSubmit(): void {
    if (this.registrationForm.valid) {
      console.log(this.registrationForm.getRawValue());
    } else {
      this.registrationForm.markAllAsTouched();
    }
  }
}
```

---

## Built-in Validators

| Validator | Usage | Description |
|-----------|-------|-------------|
| `Validators.required` | `['', Validators.required]` | Field must have a value |
| `Validators.requiredTrue` | `[false, Validators.requiredTrue]` | Must be `true` (checkboxes) |
| `Validators.email` | `['', Validators.email]` | Must be valid email format |
| `Validators.minLength(n)` | `['', Validators.minLength(3)]` | Minimum string length |
| `Validators.maxLength(n)` | `['', Validators.maxLength(50)]` | Maximum string length |
| `Validators.min(n)` | `[0, Validators.min(1)]` | Minimum number value |
| `Validators.max(n)` | `[100, Validators.max(999)]` | Maximum number value |
| `Validators.pattern(regex)` | `['', Validators.pattern(/^[A-Z]/)]` | Must match regex pattern |

### Composing Validators

```typescript
const form = this.fb.group({
  username: ['', [
    Validators.required,
    Validators.minLength(4),
    Validators.maxLength(20),
    Validators.pattern(/^[a-zA-Z0-9_]+$/)
  ]],
  age: [null, [
    Validators.required,
    Validators.min(13),
    Validators.max(120)
  ]]
});
```

---

## Custom Validators

### Synchronous Validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Factory function pattern (preferred for parameterized validators)
export function forbiddenNameValidator(forbiddenName: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = forbiddenName.test(control.value);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Simple validator function
export function noWhitespaceValidator(control: AbstractControl): ValidationErrors | null {
  if (typeof control.value !== 'string') return null;
  const isWhitespace = control.value.trim().length === 0;
  return isWhitespace ? { whitespace: true } : null;
}

// Strong password validator
export function strongPasswordValidator(control: AbstractControl): ValidationErrors | null {
  const value = control.value as string;
  if (!value) return null;

  const errors: ValidationErrors = {};
  if (!/[A-Z]/.test(value)) errors['missingUppercase'] = true;
  if (!/[a-z]/.test(value)) errors['missingLowercase'] = true;
  if (!/[0-9]/.test(value)) errors['missingNumber'] = true;
  if (!/[!@#$%^&*]/.test(value)) errors['missingSpecial'] = true;

  return Object.keys(errors).length ? errors : null;
}

// Usage
const form = this.fb.group({
  username: ['', [Validators.required, forbiddenNameValidator(/admin/i)]],
  displayName: ['', [Validators.required, noWhitespaceValidator]],
  password: ['', [Validators.required, Validators.minLength(8), strongPasswordValidator]]
});
```

### Async Validator

```typescript
import { AsyncValidatorFn } from '@angular/forms';
import { map, debounceTime, switchMap, first, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

export function uniqueEmailValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl) => {
    if (!control.value) return of(null);

    return of(control.value).pipe(
      debounceTime(300),
      switchMap(email => userService.checkEmailAvailability(email)),
      map(isAvailable => isAvailable ? null : { emailTaken: true }),
      catchError(() => of(null)),  // On error, allow submission
      first()
    );
  };
}

// Usage - async validators are the third argument
const emailControl = new FormControl('', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [uniqueEmailValidator(this.userService)],
  updateOn: 'blur'   // Only validate on blur to reduce API calls
});
```

### Cross-Field Validator

```typescript
export const passwordMatchValidator: ValidatorFn = (group: AbstractControl): ValidationErrors | null => {
  const password = group.get('password')?.value;
  const confirmPassword = group.get('confirmPassword')?.value;

  if (!password || !confirmPassword) return null;

  return password === confirmPassword ? null : { passwordMismatch: true };
};

// Date range validator
export const dateRangeValidator: ValidatorFn = (group: AbstractControl): ValidationErrors | null => {
  const start = group.get('startDate')?.value;
  const end = group.get('endDate')?.value;

  if (!start || !end) return null;

  return new Date(start) < new Date(end) ? null : { invalidDateRange: true };
};

// Usage
const form = this.fb.group({
  password: ['', Validators.required],
  confirmPassword: ['', Validators.required]
}, { validators: passwordMatchValidator });
```

---

## Dynamic Forms with FormArray

```typescript
import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, NonNullableFormBuilder, FormArray, Validators } from '@angular/forms';

@Component({
  selector: 'app-invoice',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="invoiceForm" (ngSubmit)="onSubmit()">
      <input formControlName="clientName" placeholder="Client Name" />

      <div formArrayName="lineItems">
        <h3>Line Items</h3>
        @for (item of lineItems.controls; track $index; let i = $index) {
          <div class="line-item" [formGroupName]="i">
            <input formControlName="description" placeholder="Description" />
            <input formControlName="quantity" type="number" placeholder="Qty" />
            <input formControlName="unitPrice" type="number" placeholder="Unit Price" />
            <span class="total">{{ getLineTotal(i) | currency }}</span>
            <button type="button" (click)="removeItem(i)">Remove</button>
          </div>
        }
      </div>

      <button type="button" (click)="addItem()">Add Line Item</button>

      <div class="totals">
        <p>Subtotal: {{ subtotal | currency }}</p>
        <p>Tax (10%): {{ tax | currency }}</p>
        <p><strong>Total: {{ total | currency }}</strong></p>
      </div>

      <button type="submit" [disabled]="invoiceForm.invalid">Create Invoice</button>
    </form>
  `
})
export class InvoiceComponent {
  private fb = inject(NonNullableFormBuilder);

  invoiceForm = this.fb.group({
    clientName: ['', Validators.required],
    lineItems: this.fb.array([this.createLineItem()])
  });

  get lineItems(): FormArray {
    return this.invoiceForm.controls.lineItems;
  }

  get subtotal(): number {
    return this.lineItems.controls.reduce((sum, control, i) => {
      return sum + this.getLineTotal(i);
    }, 0);
  }

  get tax(): number {
    return this.subtotal * 0.1;
  }

  get total(): number {
    return this.subtotal + this.tax;
  }

  createLineItem() {
    return this.fb.group({
      description: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]],
      unitPrice: [0, [Validators.required, Validators.min(0)]]
    });
  }

  addItem(): void {
    this.lineItems.push(this.createLineItem());
  }

  removeItem(index: number): void {
    if (this.lineItems.length > 1) {
      this.lineItems.removeAt(index);
    }
  }

  getLineTotal(index: number): number {
    const item = this.lineItems.at(index);
    const qty = item.get('quantity')?.value ?? 0;
    const price = item.get('unitPrice')?.value ?? 0;
    return qty * price;
  }

  onSubmit(): void {
    if (this.invoiceForm.valid) {
      console.log(this.invoiceForm.getRawValue());
    }
  }
}
```

---

## Template-Driven Forms

Template-driven forms use directives like `ngModel` to bind form controls in the template.

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

interface ContactForm {
  name: string;
  email: string;
  subject: string;
  message: string;
}

@Component({
  selector: 'app-contact',
  imports: [FormsModule],
  template: `
    <form #contactForm="ngForm" (ngSubmit)="onSubmit(contactForm)">
      <div>
        <label for="name">Name</label>
        <input id="name" name="name" [(ngModel)]="form.name"
               required minlength="2" #name="ngModel" />
        @if (name.invalid && name.touched) {
          @if (name.errors?.['required']) {
            <span class="error">Name is required</span>
          }
          @if (name.errors?.['minlength']) {
            <span class="error">Name must be at least 2 characters</span>
          }
        }
      </div>

      <div>
        <label for="email">Email</label>
        <input id="email" name="email" [(ngModel)]="form.email"
               required email #email="ngModel" type="email" />
        @if (email.invalid && email.touched) {
          <span class="error">Valid email is required</span>
        }
      </div>

      <div>
        <label for="subject">Subject</label>
        <select id="subject" name="subject" [(ngModel)]="form.subject" required>
          <option value="">Select a subject</option>
          <option value="general">General Inquiry</option>
          <option value="support">Technical Support</option>
          <option value="billing">Billing</option>
        </select>
      </div>

      <div>
        <label for="message">Message</label>
        <textarea id="message" name="message" [(ngModel)]="form.message"
                  required minlength="10" #message="ngModel"></textarea>
        @if (message.invalid && message.touched) {
          <span class="error">Message must be at least 10 characters</span>
        }
      </div>

      <button type="submit" [disabled]="contactForm.invalid">Send</button>
    </form>
  `
})
export class ContactComponent {
  form: ContactForm = {
    name: '',
    email: '',
    subject: '',
    message: ''
  };

  onSubmit(form: NgForm): void {
    if (form.valid) {
      console.log('Contact form:', this.form);
      form.resetForm();
    }
  }
}
```

---

## Custom Form Controls (ControlValueAccessor)

Create custom form controls that work with both reactive and template-driven forms.

```typescript
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  template: `
    <div class="stars" [class.disabled]="disabled()">
      @for (star of stars; track star) {
        <button
          type="button"
          (click)="rate(star)"
          (mouseenter)="hovered.set(star)"
          (mouseleave)="hovered.set(0)"
          [class.filled]="star <= (hovered() || value())"
          [disabled]="disabled()"
          [attr.aria-label]="star + ' star' + (star > 1 ? 's' : '')"
        >
          ★
        </button>
      }
      <span class="label">{{ value() }} / 5</span>
    </div>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true
    }
  ]
})
export class StarRatingComponent implements ControlValueAccessor {
  readonly stars = [1, 2, 3, 4, 5];
  value = signal(0);
  hovered = signal(0);
  disabled = signal(false);

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  // Called by Angular to write a value to the control
  writeValue(value: number): void {
    this.value.set(value ?? 0);
  }

  // Register callback for value changes
  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  // Register callback for touch events
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  // Called when the control is disabled/enabled
  setDisabledState(isDisabled: boolean): void {
    this.disabled.set(isDisabled);
  }

  rate(star: number): void {
    if (this.disabled()) return;
    this.value.set(star);
    this.onChange(star);
    this.onTouched();
  }
}
```

Usage with reactive forms:

```typescript
@Component({
  imports: [ReactiveFormsModule, StarRatingComponent],
  template: `
    <form [formGroup]="reviewForm">
      <app-star-rating formControlName="rating" />
      <textarea formControlName="comment" placeholder="Your review..."></textarea>
    </form>
  `
})
export class ReviewComponent {
  private fb = inject(NonNullableFormBuilder);

  reviewForm = this.fb.group({
    rating: [0, [Validators.required, Validators.min(1)]],
    comment: ['', [Validators.required, Validators.minLength(10)]]
  });
}
```

Usage with template-driven forms:

```html
<app-star-rating [(ngModel)]="userRating" name="rating" required />
```

---

## Form State

| Property | Type | Description |
|----------|------|-------------|
| `valid` | `boolean` | All validators pass |
| `invalid` | `boolean` | At least one validator fails |
| `pending` | `boolean` | Async validators are running |
| `disabled` | `boolean` | Control is disabled |
| `enabled` | `boolean` | Control is enabled |
| `pristine` | `boolean` | Value has not been changed by user |
| `dirty` | `boolean` | Value has been changed by user |
| `touched` | `boolean` | Control has been focused and blurred |
| `untouched` | `boolean` | Control has never been blurred |
| `errors` | `ValidationErrors \| null` | Current validation errors |
| `value` | `T` | Current control value |

### Error Display Helper

```typescript
@Component({
  selector: 'app-form-field',
  template: `
    <div class="form-field" [class.has-error]="showError()">
      <label [for]="fieldId">{{ label }}</label>
      <ng-content />
      @if (showError()) {
        @for (message of errorMessages(); track message) {
          <span class="error">{{ message }}</span>
        }
      }
    </div>
  `
})
export class FormFieldComponent {
  control = input.required<AbstractControl>();
  label = input.required<string>();
  fieldId = input('');

  private errorMap: Record<string, string | ((err: any) => string)> = {
    required: 'This field is required',
    email: 'Invalid email address',
    minlength: (err) => `Minimum ${err.requiredLength} characters`,
    maxlength: (err) => `Maximum ${err.requiredLength} characters`,
    min: (err) => `Minimum value is ${err.min}`,
    max: (err) => `Maximum value is ${err.max}`,
    pattern: 'Invalid format'
  };

  showError = computed(() => {
    const ctrl = this.control();
    return ctrl.invalid && (ctrl.dirty || ctrl.touched);
  });

  errorMessages = computed(() => {
    const errors = this.control().errors;
    if (!errors) return [];
    return Object.keys(errors).map(key => {
      const mapper = this.errorMap[key];
      if (typeof mapper === 'function') return mapper(errors[key]);
      return mapper ?? key;
    });
  });
}
```

---

## Form Submission Patterns

### Handling Submission State

```typescript
@Component({
  selector: 'app-create-user',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="name" placeholder="Name" />
      <input formControlName="email" type="email" placeholder="Email" />

      @if (serverError()) {
        <div class="alert alert-error">{{ serverError() }}</div>
      }

      <button type="submit" [disabled]="form.invalid || submitting()">
        @if (submitting()) {
          <span class="spinner"></span> Saving...
        } @else {
          Create User
        }
      </button>
    </form>
  `
})
export class CreateUserComponent {
  private fb = inject(NonNullableFormBuilder);
  private userService = inject(UserService);
  private router = inject(Router);

  submitting = signal(false);
  serverError = signal<string | null>(null);

  form = this.fb.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });

  onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitting.set(true);
    this.serverError.set(null);

    this.userService.create(this.form.getRawValue()).pipe(
      finalize(() => this.submitting.set(false))
    ).subscribe({
      next: (user) => {
        this.router.navigate(['/users', user.id]);
      },
      error: (err) => {
        if (err.status === 422) {
          // Apply server-side validation errors to form controls
          const errors = err.error.errors;
          Object.keys(errors).forEach(field => {
            this.form.get(field)?.setErrors({ serverError: errors[field] });
          });
        } else {
          this.serverError.set('An unexpected error occurred. Please try again.');
        }
      }
    });
  }
}
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use reactive forms | Prefer reactive forms for complex forms with validation |
| Use typed forms | Use `NonNullableFormBuilder` for type safety |
| Use `updateOn: 'blur'` for async validators | Reduces unnecessary API calls |
| Mark all as touched on submit | Show all errors when user submits invalid form |
| Create reusable validators | Extract validators into separate files |
| Use `ControlValueAccessor` | Create custom form controls that integrate with Angular forms |
| Disable submit during async operations | Prevent double submissions |
| Reset form properly | Use `form.reset()` or `form.resetForm()` for template-driven |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Missing `ReactiveFormsModule` import | `formGroup`/`formControlName` do not work | Import `ReactiveFormsModule` in standalone component |
| Using `value` instead of `getRawValue()` | Disabled controls excluded from `value` | Use `getRawValue()` to include disabled controls |
| Forgetting `name` attribute with `ngModel` | Template-driven form errors | Always add `name` attribute with `ngModel` |
| Mutating form value directly | Form state becomes inconsistent | Use `setValue()`, `patchValue()`, or control methods |
| Not handling async validator pending state | Submit while validation is running | Check `form.pending` before submit |
| Cross-field validator on wrong level | Validator never runs | Apply cross-field validators on the parent `FormGroup` |
| FormArray tracking by index | Incorrect rendering after add/remove | Use `track $index` or a unique identifier |
| Not unsubscribing from `valueChanges` | Memory leaks | Use `takeUntilDestroyed()` |
