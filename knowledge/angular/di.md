# Angular Dependency Injection

> Official Documentation: https://angular.dev/guide/di

## inject() Function

The `inject()` function is the preferred way to inject dependencies in Angular 17+. It can be used in constructors, field initializers, and factory functions.

### Basic Usage

```typescript
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ActivatedRoute, Router } from '@angular/router';

@Component({
  selector: 'app-product-detail',
  template: `
    @if (product(); as product) {
      <h1>{{ product.name }}</h1>
      <p>{{ product.price | currency }}</p>
    }
  `
})
export class ProductDetailComponent {
  // Preferred: inject() in field initializer
  private http = inject(HttpClient);
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private productService = inject(ProductService);
  private destroyRef = inject(DestroyRef);

  product = signal<Product | null>(null);

  constructor() {
    const id = this.route.snapshot.paramMap.get('id')!;
    this.productService.getById(id)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(p => this.product.set(p));
  }
}
```

### inject() vs Constructor Injection

```typescript
// Preferred: inject() function
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private authService = inject(AuthService);

  getProfile(): Observable<User> {
    return this.http.get<User>('/api/profile');
  }
}

// Legacy: Constructor injection (still works, but inject() is preferred)
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private http: HttpClient,
    private authService: AuthService
  ) {}

  getProfile(): Observable<User> {
    return this.http.get<User>('/api/profile');
  }
}
```

### inject() in Functions

```typescript
// inject() works in factory functions and functional guards/resolvers
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }
  return router.createUrlTree(['/login']);
};

export const userResolver: ResolveFn<User> = (route) => {
  const userService = inject(UserService);
  return userService.getById(route.paramMap.get('id')!);
};
```

---

## @Injectable and providedIn

### providedIn Options

```typescript
// Root-level singleton (most common, tree-shakable)
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUser = signal<User | null>(null);

  isAuthenticated(): boolean {
    return this.currentUser() !== null;
  }
}

// Platform-level (shared across all apps on same platform)
@Injectable({ providedIn: 'platform' })
export class AnalyticsService {
  track(event: string, data?: Record<string, unknown>): void {
    // Shared across micro-frontend apps
  }
}

// No providedIn - must be explicitly provided
@Injectable()
export class FormValidationService {
  // Must be listed in component or route providers
}
```

### providedIn Comparison

| Option | Scope | Tree-Shakable | Use Case |
|--------|-------|---------------|----------|
| `'root'` | Application singleton | Yes | Most services (HTTP, auth, state) |
| `'platform'` | Platform singleton | Yes | Shared across multiple apps |
| `'any'` | Per-module instance | Yes | Module-specific instances (rare) |
| Not set | Must be provided explicitly | No | Component-scoped or test services |

---

## Provider Types

### useClass - Provide a Class

```typescript
// Interface / abstract class as token
export abstract class LoggerService {
  abstract log(message: string): void;
  abstract error(message: string, error?: Error): void;
}

// Implementation
@Injectable()
export class ConsoleLoggerService extends LoggerService {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }

  error(message: string, error?: Error): void {
    console.error(`[ERROR] ${message}`, error);
  }
}

// Production implementation
@Injectable()
export class RemoteLoggerService extends LoggerService {
  private http = inject(HttpClient);

  log(message: string): void {
    this.http.post('/api/logs', { level: 'info', message }).subscribe();
  }

  error(message: string, error?: Error): void {
    this.http.post('/api/logs', {
      level: 'error',
      message,
      stack: error?.stack
    }).subscribe();
  }
}

// Register in providers
providers: [
  { provide: LoggerService, useClass: environment.production ? RemoteLoggerService : ConsoleLoggerService }
]
```

### useValue - Provide a Static Value

```typescript
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  apiUrl: string;
  appName: string;
  version: string;
  features: {
    darkMode: boolean;
    notifications: boolean;
  };
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Provide in app.config.ts
providers: [
  {
    provide: APP_CONFIG,
    useValue: {
      apiUrl: 'https://api.example.com',
      appName: 'My App',
      version: '2.1.0',
      features: { darkMode: true, notifications: true }
    } satisfies AppConfig
  }
]

// Inject in components or services
@Component({ /* ... */ })
export class FooterComponent {
  private config = inject(APP_CONFIG);

  version = this.config.version;
  appName = this.config.appName;
}
```

### useFactory - Provide with Factory Function

```typescript
export const API_CLIENT = new InjectionToken<ApiClient>('ApiClient');

// Factory function with dependencies
providers: [
  {
    provide: API_CLIENT,
    useFactory: () => {
      const http = inject(HttpClient);
      const config = inject(APP_CONFIG);
      const authService = inject(AuthService);

      return new ApiClient({
        baseUrl: config.apiUrl,
        http,
        getToken: () => authService.getAccessToken()
      });
    }
  }
]
```

```typescript
// Factory for environment-dependent services
export const STORAGE_SERVICE = new InjectionToken<StorageService>('StorageService');

providers: [
  {
    provide: STORAGE_SERVICE,
    useFactory: () => {
      const platformId = inject(PLATFORM_ID);
      if (isPlatformBrowser(platformId)) {
        return new LocalStorageService();
      }
      return new InMemoryStorageService(); // SSR fallback
    }
  }
]
```

### useExisting - Alias a Provider

```typescript
// Create an alias to an existing provider
@Injectable({ providedIn: 'root' })
export class ExtendedUserService extends UserService {
  getFullProfile(): Observable<FullProfile> {
    // Extended functionality
    return this.http.get<FullProfile>('/api/profile/full');
  }
}

providers: [
  ExtendedUserService,
  { provide: UserService, useExisting: ExtendedUserService }
  // Both UserService and ExtendedUserService resolve to the same instance
]
```

---

## InjectionToken

Use `InjectionToken` for non-class dependencies and configuration values.

```typescript
import { InjectionToken } from '@angular/core';

// Simple value token
export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');
export const MAX_RETRIES = new InjectionToken<number>('MAX_RETRIES');
export const IS_PRODUCTION = new InjectionToken<boolean>('IS_PRODUCTION');

// Token with default factory (tree-shakable)
export const WINDOW = new InjectionToken<Window>('Window', {
  providedIn: 'root',
  factory: () => window
});

export const LOCAL_STORAGE = new InjectionToken<Storage>('LocalStorage', {
  providedIn: 'root',
  factory: () => {
    if (isPlatformBrowser(inject(PLATFORM_ID))) {
      return localStorage;
    }
    // Return a no-op storage for SSR
    return {
      length: 0,
      clear: () => {},
      getItem: () => null,
      key: () => null,
      removeItem: () => {},
      setItem: () => {}
    } as Storage;
  }
});

// Usage
@Injectable({ providedIn: 'root' })
export class StorageService {
  private storage = inject(LOCAL_STORAGE);

  get<T>(key: string): T | null {
    const item = this.storage.getItem(key);
    return item ? JSON.parse(item) : null;
  }

  set<T>(key: string, value: T): void {
    this.storage.setItem(key, JSON.stringify(value));
  }
}
```

### Complex Token Types

```typescript
// Function token
export const DATE_FORMATTER = new InjectionToken<(date: Date) => string>('DateFormatter');

providers: [
  {
    provide: DATE_FORMATTER,
    useValue: (date: Date) => date.toISOString().split('T')[0]
  }
]

// Interface token
export interface FeatureFlags {
  enableChat: boolean;
  enableAnalytics: boolean;
  maxUploadSizeMb: number;
}

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('FeatureFlags');
```

---

## Hierarchical Injector System

Angular has a hierarchy of injectors: Platform > Root > Component.

### Component-Level Providers

```typescript
// Each instance of this component gets its own FormStateService
@Component({
  selector: 'app-edit-form',
  providers: [FormStateService],  // New instance per component
  template: `
    <form [formGroup]="form">
      <!-- form fields -->
    </form>
  `
})
export class EditFormComponent {
  private formState = inject(FormStateService);
}
```

### viewProviders vs providers

```typescript
@Component({
  selector: 'app-tabs',
  // providers: visible to component and ALL descendants (including content projection)
  providers: [TabGroupService],

  // viewProviders: visible to component and view children only (NOT content projection)
  viewProviders: [TabGroupService],

  template: `
    <div class="tabs">
      <ng-content />  <!-- Projected content -->
    </div>
  `
})
export class TabsComponent {}
```

| Scope | `providers` | `viewProviders` |
|-------|-------------|-----------------|
| Component itself | Yes | Yes |
| View children | Yes | Yes |
| Content children (ng-content) | Yes | No |

### Service Scope Examples

```typescript
// Application-wide singleton
@Injectable({ providedIn: 'root' })
export class AuthService { }

// Route-level instance (shared among child routes)
export const adminRoutes: Routes = [
  {
    path: '',
    providers: [AdminStateService],  // Shared within admin routes
    children: [
      { path: '', component: AdminDashboardComponent },
      { path: 'users', component: AdminUsersComponent }
    ]
  }
];

// Component-level instance (new per component)
@Component({
  providers: [FormValidationService]  // Each form gets its own instance
})
export class UserFormComponent { }
```

---

## Multi-Providers

Multi-providers allow multiple values to be registered under the same token.

```typescript
export const VALIDATOR = new InjectionToken<Validator[]>('Validators');

export interface Validator {
  validate(data: unknown): ValidationError[];
}

@Injectable()
export class RequiredFieldsValidator implements Validator {
  validate(data: unknown): ValidationError[] {
    // Check required fields
    return [];
  }
}

@Injectable()
export class BusinessRulesValidator implements Validator {
  validate(data: unknown): ValidationError[] {
    // Check business rules
    return [];
  }
}

// Register multiple values under one token
providers: [
  { provide: VALIDATOR, useClass: RequiredFieldsValidator, multi: true },
  { provide: VALIDATOR, useClass: BusinessRulesValidator, multi: true }
]

// Inject as array
@Injectable()
export class ValidationService {
  private validators = inject(VALIDATOR); // Validator[]

  validateAll(data: unknown): ValidationError[] {
    return this.validators.flatMap(v => v.validate(data));
  }
}
```

### Common Multi-Provider Patterns

```typescript
// APP_INITIALIZER - run functions on app startup
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: () => {
        const authService = inject(AuthService);
        return () => authService.initializeFromStorage();
      },
      multi: true
    },
    {
      provide: APP_INITIALIZER,
      useFactory: () => {
        const configService = inject(ConfigService);
        return () => configService.loadRemoteConfig();
      },
      multi: true
    }
  ]
};

// HTTP_INTERCEPTORS (class-based)
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
{ provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }
```

---

## Optional Dependencies

```typescript
@Injectable({ providedIn: 'root' })
export class NotificationService {
  // inject() with optional flag
  private analytics = inject(AnalyticsService, { optional: true });

  send(notification: Notification): void {
    // Display notification...

    // Only track if analytics is available
    this.analytics?.track('notification_shown', {
      type: notification.type
    });
  }
}
```

### inject() Options

| Option | Type | Description |
|--------|------|-------------|
| `optional` | `boolean` | Returns `null` if provider not found (default: throws error) |
| `self` | `boolean` | Only look in the component's own injector |
| `skipSelf` | `boolean` | Skip the component's injector, start from parent |
| `host` | `boolean` | Stop at host component injector |

```typescript
@Component({ /* ... */ })
export class ChildComponent {
  // Only check parent injectors (skip self)
  private parentService = inject(ParentService, { skipSelf: true });

  // Only check own injector
  private localService = inject(LocalService, { self: true, optional: true });

  // Stop at host component boundary
  private hostService = inject(HostService, { host: true, optional: true });
}
```

---

## forwardRef

Use `forwardRef` when you have circular dependencies or need to reference a class before it is defined.

```typescript
import { forwardRef, Inject, Injectable } from '@angular/core';

// Circular dependency: ServiceA depends on ServiceB and vice versa
@Injectable({ providedIn: 'root' })
export class ServiceA {
  // Use forwardRef because ServiceB is defined after ServiceA
  private serviceB = inject(forwardRef(() => ServiceB));

  methodA(): string {
    return 'A -> ' + this.serviceB.methodB();
  }
}

@Injectable({ providedIn: 'root' })
export class ServiceB {
  private serviceA = inject(ServiceA);

  methodB(): string {
    return 'B';
  }
}
```

```typescript
// Common use: ControlValueAccessor self-reference
@Component({
  selector: 'app-custom-input',
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => CustomInputComponent),
      multi: true
    }
  ]
})
export class CustomInputComponent implements ControlValueAccessor {
  // ...
}
```

---

## Injection Context Rules

`inject()` can only be called in specific contexts:

```typescript
// VALID contexts for inject():

// 1. Constructor
@Component({ /* ... */ })
export class MyComponent {
  constructor() {
    const service = inject(MyService); // Valid
  }
}

// 2. Field initializer
@Component({ /* ... */ })
export class MyComponent {
  private service = inject(MyService); // Valid
}

// 3. Factory function in providers
providers: [
  {
    provide: TOKEN,
    useFactory: () => {
      const dep = inject(SomeDep); // Valid
      return new Something(dep);
    }
  }
]

// 4. Functional guards, resolvers, interceptors
export const myGuard: CanActivateFn = () => {
  const auth = inject(AuthService); // Valid
  return auth.isAuthenticated();
};

// 5. Inside effect(), computed() in constructor context
@Component({ /* ... */ })
export class MyComponent {
  private service = inject(MyService);

  constructor() {
    effect(() => {
      // inject() is NOT valid here (but dependencies injected above are accessible)
      this.service.doSomething(); // Use already-injected references instead
    });
  }
}

// INVALID contexts:
// - setTimeout/setInterval callbacks
// - Promise .then() handlers
// - Event handlers
// - ngOnInit and other lifecycle hooks (inject in field initializer instead)

// Use runInInjectionContext for dynamic injection
@Injectable({ providedIn: 'root' })
export class PluginLoader {
  private injector = inject(EnvironmentInjector);

  loadPlugin(): void {
    runInInjectionContext(this.injector, () => {
      const service = inject(SomeService); // Valid inside runInInjectionContext
    });
  }
}
```

---

## Best Practices

| Practice | Recommendation |
|----------|---------------|
| Use `inject()` over constructor | Cleaner, works in field initializers, enables better tree-shaking |
| Use `providedIn: 'root'` | Default for most services; tree-shakable singleton |
| Use `InjectionToken` for non-class deps | Type-safe, descriptive, supports default factories |
| Keep services focused | Single responsibility; one service per concern |
| Use component-level providers for state | Each component instance gets its own service |
| Use `optional: true` for optional deps | Prevents runtime errors when a provider is missing |
| Prefer interfaces with `InjectionToken` | Enables swapping implementations easily |
| Use `APP_INITIALIZER` for startup logic | Runs before app renders; supports async operations |

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| `inject()` outside injection context | `RuntimeError: inject() must be called from an injection context` | Call `inject()` in constructor, field initializer, or factory |
| Forgetting `providedIn: 'root'` | Service not found / `NullInjectorError` | Add `providedIn: 'root'` or list in providers array |
| Circular dependency | Stack overflow or `Cannot instantiate cyclic dependency` | Use `forwardRef()` or restructure to break the cycle |
| Multiple instances when expecting singleton | Different injector levels create separate instances | Ensure `providedIn: 'root'` or provide at correct level |
| Using `multi: true` without reading as array | Only gets last registration | Always inject multi-providers as array type |
| Forgetting `@Injectable()` decorator | DI metadata missing, provider not injectable | Always add `@Injectable()` to services |
| Component providers leaking | Service outlives component | Component-level providers are destroyed with the component |
| Token collision | Different providers sharing the same class token | Use `InjectionToken` for unique identification |
