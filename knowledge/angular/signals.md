# Angular Signals

> Official Documentation: https://angular.dev/guide/signals

## signal() - Creating Writable Signals

A signal is a reactive value wrapper that notifies consumers when it changes. Writable signals are created with the `signal()` function.

### Basic Usage

```typescript
import { signal } from '@angular/core';

// Primitive signals
const count = signal(0);
const name = signal('Angular');
const isVisible = signal(true);

// Reading a signal value (call it like a function)
console.log(count());    // 0
console.log(name());     // 'Angular'

// Setting a new value
count.set(5);
name.set('Angular 19');

// Updating based on current value
count.update(current => current + 1);
```

### Typed Signals

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  roles: string[];
}

// Explicit type with initial value
const user = signal<User | null>(null);

// Type inferred from initial value
const items = signal<string[]>([]);
const config = signal({ theme: 'dark', lang: 'en' });

// Setting complex values (must provide full object)
user.set({ id: 1, name: 'Alice', email: 'alice@example.com', roles: ['admin'] });

// Updating nested values immutably
user.update(current => {
  if (!current) return current;
  return { ...current, name: 'Bob' };
});
```

### Signals in Components

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count() }}</span>
      <button (click)="increment()">+</button>
      <button (click)="reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  count = signal(0);

  increment(): void {
    this.count.update(c => c + 1);
  }

  decrement(): void {
    this.count.update(c => c - 1);
  }

  reset(): void {
    this.count.set(0);
  }
}
```

---

## computed() - Derived Signals

Computed signals automatically recalculate when their dependencies change. They are read-only.

```typescript
import { signal, computed } from '@angular/core';

const price = signal(100);
const quantity = signal(3);
const taxRate = signal(0.08);

// Derived values - automatically update when dependencies change
const subtotal = computed(() => price() * quantity());
const tax = computed(() => subtotal() * taxRate());
const total = computed(() => subtotal() + tax());

console.log(total()); // 324

price.set(200);
console.log(total()); // 648 (automatically recalculated)
```

### Computed with Complex Logic

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  inStock: boolean;
}

@Component({
  selector: 'app-product-list',
  template: `
    <input (input)="searchTerm.set(getInputValue($event))" placeholder="Search..." />
    <select (change)="selectedCategory.set(getSelectValue($event))">
      <option value="">All Categories</option>
      @for (cat of categories(); track cat) {
        <option [value]="cat">{{ cat }}</option>
      }
    </select>

    <p>Showing {{ filteredProducts().length }} of {{ products().length }} products</p>

    @for (product of filteredProducts(); track product.id) {
      <app-product-card [product]="product" />
    } @empty {
      <p>No products match your criteria.</p>
    }
  `
})
export class ProductListComponent {
  products = signal<Product[]>([]);
  searchTerm = signal('');
  selectedCategory = signal('');

  categories = computed(() => {
    const cats = new Set(this.products().map(p => p.category));
    return Array.from(cats).sort();
  });

  filteredProducts = computed(() => {
    let result = this.products();
    const search = this.searchTerm().toLowerCase();
    const category = this.selectedCategory();

    if (search) {
      result = result.filter(p => p.name.toLowerCase().includes(search));
    }
    if (category) {
      result = result.filter(p => p.category === category);
    }
    return result.filter(p => p.inStock);
  });

  totalValue = computed(() =>
    this.filteredProducts().reduce((sum, p) => sum + p.price, 0)
  );

  getInputValue(event: Event): string {
    return (event.target as HTMLInputElement).value;
  }

  getSelectValue(event: Event): string {
    return (event.target as HTMLSelectElement).value;
  }
}
```

---

## effect() - Side Effects

Effects run whenever their signal dependencies change. They are primarily used for logging, synchronization with external systems, and other side effects.

```typescript
import { effect, signal, Component, inject, DestroyRef } from '@angular/core';

@Component({
  selector: 'app-settings',
  template: `
    <label>
      Theme:
      <select (change)="theme.set(getSelectValue($event))">
        <option value="light">Light</option>
        <option value="dark">Dark</option>
      </select>
    </label>
    <label>
      Language:
      <select (change)="language.set(getSelectValue($event))">
        <option value="en">English</option>
        <option value="es">Spanish</option>
      </select>
    </label>
  `
})
export class SettingsComponent {
  theme = signal('light');
  language = signal('en');

  constructor() {
    // Effect runs when theme or language changes
    effect(() => {
      const currentTheme = this.theme();
      const currentLang = this.language();
      document.documentElement.setAttribute('data-theme', currentTheme);
      localStorage.setItem('settings', JSON.stringify({
        theme: currentTheme,
        language: currentLang
      }));
    });
  }

  getSelectValue(event: Event): string {
    return (event.target as HTMLSelectElement).value;
  }
}
```

### Effect Cleanup

```typescript
@Component({ /* ... */ })
export class WebSocketComponent {
  endpoint = signal('wss://api.example.com/ws');

  constructor() {
    effect((onCleanup) => {
      const url = this.endpoint();
      const ws = new WebSocket(url);

      ws.onmessage = (event) => {
        console.log('Message:', event.data);
      };

      // Cleanup runs before the effect re-executes or on destroy
      onCleanup(() => {
        ws.close();
      });
    });
  }
}
```

### Effect Options

```typescript
// allowSignalWrites: permit signal writes inside effects
effect(() => {
  const items = this.rawItems();
  this.sortedItems.set(items.sort((a, b) => a.name.localeCompare(b.name)));
}, { allowSignalWrites: true });
```

---

## Signal-Based Inputs and Outputs

### input() and input.required()

```typescript
import { Component, input, computed } from '@angular/core';

@Component({
  selector: 'app-user-badge',
  template: `
    <span class="badge" [class]="badgeClass()">
      {{ initials() }}
    </span>
    <span class="name">{{ displayName() }}</span>
  `
})
export class UserBadgeComponent {
  name = input.required<string>();
  role = input<'admin' | 'user' | 'guest'>('user');
  showRole = input(false);

  // Computed values derived from inputs (since inputs are signals)
  initials = computed(() => {
    return this.name()
      .split(' ')
      .map(part => part[0])
      .join('')
      .toUpperCase();
  });

  displayName = computed(() => {
    const base = this.name();
    return this.showRole() ? `${base} (${this.role()})` : base;
  });

  badgeClass = computed(() => `badge-${this.role()}`);
}
```

### output()

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-file-upload',
  template: `
    <input type="file" (change)="onFileSelected($event)" [accept]="accept()" />
    @if (uploading()) {
      <progress [value]="progress()" max="100"></progress>
    }
  `
})
export class FileUploadComponent {
  accept = input('.pdf,.doc,.docx');

  fileSelected = output<File>();
  uploadComplete = output<{ url: string; size: number }>();
  uploadError = output<string>();

  uploading = signal(false);
  progress = signal(0);

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    if (file) {
      this.fileSelected.emit(file);
      this.upload(file);
    }
  }

  private async upload(file: File): Promise<void> {
    this.uploading.set(true);
    try {
      // Upload logic...
      this.uploadComplete.emit({ url: 'https://...', size: file.size });
    } catch (e) {
      this.uploadError.emit('Upload failed');
    } finally {
      this.uploading.set(false);
    }
  }
}
```

---

## model() - Two-Way Binding

```typescript
import { Component, model, signal, computed } from '@angular/core';

@Component({
  selector: 'app-color-picker',
  template: `
    <div class="preview" [style.background-color]="color()"></div>
    <input type="color" [value]="color()" (input)="onColorInput($event)" />
    <input type="text" [value]="color()" (input)="onColorInput($event)" />
  `
})
export class ColorPickerComponent {
  color = model('#3b82f6');   // Creates a two-way bindable signal

  onColorInput(event: Event): void {
    this.color.set((event.target as HTMLInputElement).value);
  }
}

// Parent usage with two-way binding
@Component({
  template: `
    <app-color-picker [(color)]="selectedColor" />
    <p>Selected: {{ selectedColor }}</p>
  `,
  imports: [ColorPickerComponent]
})
export class ParentComponent {
  selectedColor = '#3b82f6';
}
```

---

## linkedSignal() - Derived Writable Signals

`linkedSignal()` creates a writable signal whose value is derived from other signals but can also be set manually. When source signals change, the linked signal recomputes.

```typescript
import { signal, linkedSignal, computed, Component } from '@angular/core';

@Component({
  selector: 'app-shipping',
  template: `
    <h3>Country: {{ country() }}</h3>
    <select (change)="country.set(getSelectValue($event))">
      <option value="US">United States</option>
      <option value="CA">Canada</option>
      <option value="UK">United Kingdom</option>
    </select>

    <h3>Shipping Method: {{ shippingMethod() }}</h3>
    <select (change)="shippingMethod.set(getSelectValue($event))">
      @for (method of availableMethods(); track method) {
        <option [value]="method">{{ method }}</option>
      }
    </select>
  `
})
export class ShippingComponent {
  country = signal('US');

  availableMethods = computed(() => {
    switch (this.country()) {
      case 'US': return ['Standard', 'Express', 'Overnight'];
      case 'CA': return ['Standard', 'Express'];
      case 'UK': return ['Standard', 'Royal Mail'];
      default: return ['Standard'];
    }
  });

  // Resets to first available method when country changes
  // But user can still manually select a different method
  shippingMethod = linkedSignal(() => this.availableMethods()[0]);

  getSelectValue(event: Event): string {
    return (event.target as HTMLSelectElement).value;
  }
}
```

### linkedSignal with Previous Value

```typescript
const page = signal(1);
const pageSize = signal(10);

// When pageSize changes, compute a new page to maintain roughly the same position
const currentPage = linkedSignal({
  source: pageSize,
  computation: (newSize, previous) => {
    if (!previous) return 1;
    const firstItemIndex = (previous.value - 1) * previous.source;
    return Math.floor(firstItemIndex / newSize) + 1;
  }
});
```

---

## resource() - Async Data Fetching

The `resource()` API manages async data fetching with built-in loading, error, and value states.

```typescript
import { Component, signal, resource, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

@Component({
  selector: 'app-user-detail',
  template: `
    @switch (userResource.status()) {
      @case ('loading') {
        <app-skeleton />
      }
      @case ('error') {
        <app-error [message]="userResource.error()?.message" />
      }
      @case ('resolved') {
        @if (userResource.value(); as user) {
          <h2>{{ user.name }}</h2>
          <p>{{ user.email }}</p>
        }
      }
    }
    <button (click)="userResource.reload()">Refresh</button>
  `
})
export class UserDetailComponent {
  private http = inject(HttpClient);

  userId = signal(1);

  userResource = resource({
    request: () => ({ id: this.userId() }),
    loader: async ({ request, abortSignal }) => {
      const response = await firstValueFrom(
        this.http.get<User>(`/api/users/${request.id}`)
      );
      return response;
    }
  });
}
```

### rxResource for Observable-Based Loading

```typescript
import { rxResource } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class ProductDetailComponent {
  private productService = inject(ProductService);

  productId = signal(1);

  productResource = rxResource({
    request: () => ({ id: this.productId() }),
    loader: ({ request }) => {
      return this.productService.getById(request.id);
    }
  });
}
```

---

## RxJS Interop

### toSignal() - Observable to Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-clock',
  template: `<p>Elapsed: {{ elapsed() }}s</p>`
})
export class ClockComponent {
  // Convert observable to signal
  elapsed = toSignal(interval(1000), { initialValue: 0 });
}
```

```typescript
@Component({ /* ... */ })
export class UserListComponent {
  private userService = inject(UserService);
  private route = inject(ActivatedRoute);

  // Route params as signal
  userId = toSignal(
    this.route.params.pipe(map(p => p['id'])),
    { initialValue: '' }
  );

  // Service data as signal
  users = toSignal(this.userService.getAll(), { initialValue: [] });

  // With requireSync for synchronous observables (e.g., BehaviorSubject)
  // currentUser = toSignal(this.authService.currentUser$, { requireSync: true });
}
```

### toObservable() - Signal to Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';
import { switchMap, debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `
    <input (input)="searchTerm.set(getInputValue($event))" placeholder="Search..." />
    @for (result of results(); track result.id) {
      <div>{{ result.title }}</div>
    }
  `
})
export class SearchComponent {
  private searchService = inject(SearchService);

  searchTerm = signal('');
  results = signal<SearchResult[]>([]);

  // Convert signal to observable for RxJS operators
  private searchTerm$ = toObservable(this.searchTerm);

  constructor() {
    this.searchTerm$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => term.length > 2
        ? this.searchService.search(term)
        : of([])
      ),
      takeUntilDestroyed()
    ).subscribe(results => this.results.set(results));
  }

  getInputValue(event: Event): string {
    return (event.target as HTMLInputElement).value;
  }
}
```

---

## Signal Equality Functions

```typescript
// Custom equality to avoid unnecessary recalculations
const position = signal(
  { x: 0, y: 0 },
  { equal: (a, b) => a.x === b.x && a.y === b.y }
);

position.set({ x: 0, y: 0 }); // No notification (equal to current value)
position.set({ x: 1, y: 0 }); // Notifies consumers (different value)
```

```typescript
// Array equality by content
const tags = signal<string[]>([], {
  equal: (a, b) =>
    a.length === b.length && a.every((val, i) => val === b[i])
});
```

---

## untracked() - Reading Without Tracking

```typescript
import { untracked, effect, signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');
const logCount = signal(0);

effect(() => {
  // Only tracks firstName. Changes to lastName or logCount
  // will NOT cause this effect to re-run.
  const first = firstName();
  const last = untracked(() => lastName());
  const count = untracked(() => logCount());

  console.log(`Effect #${count}: ${first} ${last}`);
  untracked(() => logCount.update(c => c + 1));
});
```

---

## Signal Patterns

### Form State with Signals

```typescript
@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [value]="email()" (input)="email.set(getInputValue($event))" />
      @if (emailError()) {
        <span class="error">{{ emailError() }}</span>
      }
      <input type="password" [value]="password()" (input)="password.set(getInputValue($event))" />
      <button [disabled]="!isValid() || submitting()">
        {{ submitting() ? 'Logging in...' : 'Login' }}
      </button>
    </form>
  `
})
export class LoginComponent {
  email = signal('');
  password = signal('');
  submitting = signal(false);

  emailError = computed(() => {
    const val = this.email();
    if (!val) return 'Email is required';
    if (!val.includes('@')) return 'Invalid email format';
    return null;
  });

  isValid = computed(() =>
    !this.emailError() && this.password().length >= 8
  );

  getInputValue(event: Event): string {
    return (event.target as HTMLInputElement).value;
  }

  async onSubmit(): Promise<void> {
    if (!this.isValid()) return;
    this.submitting.set(true);
    // ...
  }
}
```

### Signal-Based State Management

```typescript
interface AppState {
  user: User | null;
  notifications: Notification[];
  preferences: Preferences;
}

@Injectable({ providedIn: 'root' })
export class AppStore {
  // Private writable signals
  private _user = signal<User | null>(null);
  private _notifications = signal<Notification[]>([]);
  private _preferences = signal<Preferences>(DEFAULT_PREFERENCES);

  // Public read-only signals
  readonly user = this._user.asReadonly();
  readonly notifications = this._notifications.asReadonly();
  readonly preferences = this._preferences.asReadonly();

  // Derived state
  readonly isLoggedIn = computed(() => this._user() !== null);
  readonly unreadCount = computed(() =>
    this._notifications().filter(n => !n.read).length
  );

  setUser(user: User | null): void {
    this._user.set(user);
  }

  addNotification(notification: Notification): void {
    this._notifications.update(list => [...list, notification]);
  }

  markAsRead(id: string): void {
    this._notifications.update(list =>
      list.map(n => n.id === id ? { ...n, read: true } : n)
    );
  }

  updatePreferences(partial: Partial<Preferences>): void {
    this._preferences.update(prefs => ({ ...prefs, ...partial }));
  }
}
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Prefer signals over BehaviorSubject | Signals are simpler and integrate better with templates |
| Use `computed()` for derived state | Avoid recomputing in templates; use `computed()` |
| Avoid heavy logic in `effect()` | Effects should perform side effects, not compute state |
| Use `untracked()` for ancillary reads | Prevent unwanted reactive dependencies |
| Expose `asReadonly()` from services | Prevent external mutation of store state |
| Use `input()` over `@Input()` | Signal inputs enable derived computed values |
| Combine with OnPush | Signals + OnPush = optimal change detection |
| Use `takeUntilDestroyed()` for RxJS | Automatic cleanup when mixing signals and observables |
| Use `linkedSignal()` for resettable state | When state depends on other signals but needs manual override |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Calling signal in constructor before set | Returns initial/default value | Use `computed()` or `effect()` for reactive reads |
| Writing to signals inside `computed()` | Runtime error | Use `effect()` with `allowSignalWrites` for side effects |
| Missing `initialValue` in `toSignal()` | Signal type includes `undefined` | Provide `initialValue` or handle `undefined` |
| Not calling signal in template | Displays function reference `[Function]` | Always use `signal()` with parentheses in templates |
| Creating effects outside injection context | Runtime error | Create effects in constructor or with `runInInjectionContext` |
| Forgetting `onCleanup` in effects | Resource leaks | Use `onCleanup` callback for subscriptions, timers, connections |
| Deep object mutation | Signal does not notify | Always create new object references with `update()` |
| Overusing `effect()` | Hard to debug reactive chains | Prefer `computed()` for derived state; `effect()` only for true side effects |
