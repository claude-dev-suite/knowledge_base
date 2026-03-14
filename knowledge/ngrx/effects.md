# NgRx Effects

> Official Documentation: https://ngrx.io/guide/effects

Comprehensive reference for NgRx Effects -- the side-effect management library for Angular applications. Effects handle asynchronous operations like HTTP requests, WebSocket connections, timers, and other interactions outside the store, keeping components and reducers pure.

---

## Table of Contents

1. [Effects Setup](#effects-setup)
2. [Creating Functional Effects](#creating-functional-effects)
3. [ofType Operator](#oftype-operator)
4. [RxJS Operator Patterns](#rxjs-operator-patterns)
5. [Error Handling](#error-handling)
6. [Non-Dispatching Effects](#non-dispatching-effects)
7. [Effect Lifecycle](#effect-lifecycle)
8. [Conditional Effects](#conditional-effects)
9. [Router Effects](#router-effects)
10. [Accessing State in Effects](#accessing-state-in-effects)
11. [Testing Effects](#testing-effects)
12. [Best Practices](#best-practices)
13. [Common Pitfalls](#common-pitfalls)

---

## Effects Setup

### Provider Registration

Register effects at the application level or within lazy-loaded routes:

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { AuthEffects } from './auth/state/auth.effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideEffects(AuthEffects),
  ],
};
```

### Lazy-Loaded Route Effects

```typescript
// products.routes.ts
import { Routes } from '@angular/router';
import { provideState } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { productsFeature } from './state/products.reducer';
import { ProductsEffects } from './state/products.effects';

export const PRODUCTS_ROUTES: Routes = [
  {
    path: '',
    providers: [
      provideState(productsFeature),
      provideEffects(ProductsEffects),
    ],
    component: ProductShellComponent,
  },
];
```

---

## Creating Functional Effects

### Basic HTTP Effect

The recommended pattern uses `createEffect()` with functional injection:

```typescript
// products.effects.ts
import { inject, Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { ProductsApiActions, ProductsPageActions } from './products.actions';
import { ProductsService } from '../services/products.service';
import { catchError, exhaustMap, map, of } from 'rxjs';

@Injectable()
export class ProductsEffects {
  private readonly actions$ = inject(Actions);
  private readonly productsService = inject(ProductsService);

  readonly loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsPageActions.opened),
      exhaustMap(() =>
        this.productsService.getAll().pipe(
          map((products) =>
            ProductsApiActions.productsLoadedSuccessfully({ products })
          ),
          catchError((error) =>
            of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
          )
        )
      )
    )
  );

  readonly createProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsPageActions.addProduct),
      concatMap(({ product }) =>
        this.productsService.create(product).pipe(
          map((created) =>
            ProductsApiActions.productCreatedSuccessfully({ product: created })
          ),
          catchError((error) =>
            of(ProductsApiActions.productCreatedFailure({ error: error.message }))
          )
        )
      )
    )
  );

  readonly deleteProduct$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsPageActions.deleteProduct),
      mergeMap(({ id }) =>
        this.productsService.delete(id).pipe(
          map(() => ProductsApiActions.productDeletedSuccessfully({ id })),
          catchError((error) =>
            of(ProductsApiActions.productDeletedFailure({ error: error.message }))
          )
        )
      )
    )
  );
}
```

### Fully Functional Effects (No Class)

NgRx 17+ supports fully functional effects without a class wrapper:

```typescript
// products.effects.ts
import { inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { ProductsService } from '../services/products.service';
import { ProductsPageActions, ProductsApiActions } from './products.actions';
import { exhaustMap, map, catchError, of } from 'rxjs';

export const loadProducts = createEffect(
  (actions$ = inject(Actions), productsService = inject(ProductsService)) =>
    actions$.pipe(
      ofType(ProductsPageActions.opened),
      exhaustMap(() =>
        productsService.getAll().pipe(
          map((products) =>
            ProductsApiActions.productsLoadedSuccessfully({ products })
          ),
          catchError((error) =>
            of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
          )
        )
      )
    ),
  { functional: true }
);

export const createProduct = createEffect(
  (actions$ = inject(Actions), productsService = inject(ProductsService)) =>
    actions$.pipe(
      ofType(ProductsPageActions.addProduct),
      concatMap(({ product }) =>
        productsService.create(product).pipe(
          map((created) =>
            ProductsApiActions.productCreatedSuccessfully({ product: created })
          ),
          catchError((error) =>
            of(ProductsApiActions.productCreatedFailure({ error: error.message }))
          )
        )
      )
    ),
  { functional: true }
);
```

Register functional effects with `provideEffects`:

```typescript
import * as productsEffects from './state/products.effects';

// In route or app config:
provideEffects(productsEffects);
```

---

## ofType Operator

### Single Action

```typescript
this.actions$.pipe(
  ofType(ProductsPageActions.opened),
  // ...
)
```

### Multiple Actions

```typescript
this.actions$.pipe(
  ofType(
    ProductsPageActions.opened,
    ProductsPageActions.refreshClicked,
    ProductsPageActions.filterChanged,
  ),
  // All three actions will trigger this effect
  exhaustMap(() => this.productsService.getAll().pipe(/* ... */))
)
```

---

## RxJS Operator Patterns

Choosing the right flattening operator is critical for correct behavior:

| Operator | Behavior | Use When |
|----------|----------|----------|
| `switchMap` | Cancels previous inner observable | Fetching data (search, autocomplete) |
| `concatMap` | Queues and processes in order | Order matters (create, update) |
| `mergeMap` | Runs all in parallel | Independent operations (analytics, logging) |
| `exhaustMap` | Ignores while previous is active | Preventing duplicate submissions (login, page load) |

### switchMap -- Search / Autocomplete

```typescript
readonly search$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.searchChanged),
    debounceTime(300),
    distinctUntilChanged((prev, curr) => prev.query === curr.query),
    switchMap(({ query }) =>
      this.productsService.search(query).pipe(
        map((results) => ProductsApiActions.searchResultsLoaded({ results })),
        catchError((error) =>
          of(ProductsApiActions.searchFailed({ error: error.message }))
        )
      )
    )
  )
);
```

### concatMap -- Sequential Mutations

```typescript
readonly updateProduct$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.updateProduct),
    concatMap(({ id, changes }) =>
      this.productsService.update(id, changes).pipe(
        map((product) =>
          ProductsApiActions.productUpdatedSuccessfully({ product })
        ),
        catchError((error) =>
          of(ProductsApiActions.productUpdatedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### mergeMap -- Parallel Independent Operations

```typescript
readonly trackAnalytics$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.deleteProduct),
    mergeMap(({ id }) =>
      this.analyticsService.track('product_deleted', { productId: id }).pipe(
        map(() => AnalyticsActions.eventTracked({ event: 'product_deleted' })),
        catchError(() => EMPTY) // Silently ignore analytics failures
      )
    )
  )
);
```

### exhaustMap -- Prevent Duplicate Submissions

```typescript
readonly login$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.loginSubmitted),
    exhaustMap(({ credentials }) =>
      this.authService.login(credentials).pipe(
        map((user) => AuthApiActions.loginSuccess({ user })),
        catchError((error) =>
          of(AuthApiActions.loginFailure({ error: error.message }))
        )
      )
    )
  )
);
```

---

## Error Handling

### catchError Inside the Inner Observable

Always place `catchError` inside the inner observable (switchMap/concatMap/etc.) to prevent the effect from dying:

```typescript
// CORRECT: catchError inside the inner pipe
readonly loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.opened),
    exhaustMap(() =>
      this.productsService.getAll().pipe(
        map((products) =>
          ProductsApiActions.productsLoadedSuccessfully({ products })
        ),
        catchError((error) =>
          of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### Retry with Backoff

```typescript
import { retry, timer } from 'rxjs';

readonly loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.opened),
    exhaustMap(() =>
      this.productsService.getAll().pipe(
        retry({
          count: 3,
          delay: (error, retryCount) => {
            console.warn(`Retry attempt ${retryCount}:`, error.message);
            return timer(retryCount * 1000); // 1s, 2s, 3s
          },
        }),
        map((products) =>
          ProductsApiActions.productsLoadedSuccessfully({ products })
        ),
        catchError((error) =>
          of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### Global Error Handling for Effects

```typescript
// error-handler.effects.ts
@Injectable()
export class ErrorHandlerEffects {
  private readonly actions$ = inject(Actions);
  private readonly snackBar = inject(MatSnackBar);

  readonly showErrorNotification$ = createEffect(() =>
    this.actions$.pipe(
      // Match any action ending with "Failure"
      filter((action) => /Failure$/.test(action.type)),
      tap((action: any) => {
        this.snackBar.open(
          action.error ?? 'An unexpected error occurred',
          'Dismiss',
          { duration: 5000, panelClass: 'error-snackbar' }
        );
      })
    ),
    { dispatch: false }
  );
}
```

---

## Non-Dispatching Effects

For side effects that should not dispatch actions (e.g., logging, navigation, notifications):

```typescript
@Injectable()
export class NavigationEffects {
  private readonly actions$ = inject(Actions);
  private readonly router = inject(Router);

  readonly navigateToProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsApiActions.productCreatedSuccessfully),
      tap(({ product }) => {
        this.router.navigate(['/products', product.id]);
      })
    ),
    { dispatch: false }
  );

  readonly logActions$ = createEffect(() =>
    this.actions$.pipe(
      tap((action) => console.log('[Action]', action.type))
    ),
    { dispatch: false }
  );
}
```

### Saving to localStorage

```typescript
readonly persistState$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SettingsActions.themeChanged, SettingsActions.languageChanged),
    withLatestFrom(this.store.select(selectSettings)),
    tap(([, settings]) => {
      localStorage.setItem('app-settings', JSON.stringify(settings));
    })
  ),
  { dispatch: false }
);
```

---

## Effect Lifecycle

### OnInitEffects

Run an action when effects are registered:

```typescript
import { OnInitEffects } from '@ngrx/effects';
import { Action } from '@ngrx/store';

@Injectable()
export class AppEffects implements OnInitEffects {
  ngrxOnInitEffects(): Action {
    return AppActions.appInitialized();
  }
}
```

### OnRunEffects

Control the lifecycle of effect subscriptions:

```typescript
import { OnRunEffects, EffectNotification } from '@ngrx/effects';
import { Observable, exhaustMap, takeUntil } from 'rxjs';

@Injectable()
export class SessionEffects implements OnRunEffects {
  private readonly actions$ = inject(Actions);

  readonly heartbeat$ = createEffect(() =>
    this.actions$.pipe(
      ofType(SessionActions.started),
      switchMap(() =>
        interval(30000).pipe(
          map(() => SessionActions.heartbeatSent()),
          takeUntil(this.actions$.pipe(ofType(SessionActions.ended)))
        )
      )
    )
  );

  ngrxOnRunEffects(
    resolvedEffects$: Observable<EffectNotification>
  ): Observable<EffectNotification> {
    return this.actions$.pipe(
      ofType(AuthApiActions.loginSuccess),
      exhaustMap(() =>
        resolvedEffects$.pipe(
          takeUntil(this.actions$.pipe(ofType(AuthActions.logoutConfirmed)))
        )
      )
    );
  }
}
```

---

## Conditional Effects

### Using withLatestFrom for State-Dependent Effects

```typescript
readonly loadProductsIfNeeded$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.opened),
    withLatestFrom(this.store.select(selectProducts)),
    filter(([, products]) => products.length === 0),
    exhaustMap(() =>
      this.productsService.getAll().pipe(
        map((products) =>
          ProductsApiActions.productsLoadedSuccessfully({ products })
        ),
        catchError((error) =>
          of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### Using concatLatestFrom (Preferred Over withLatestFrom)

`concatLatestFrom` is lazily evaluated and only subscribes to the selector when the source emits:

```typescript
import { concatLatestFrom } from '@ngrx/operators';

readonly refreshProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.refreshClicked),
    concatLatestFrom(() => this.store.select(selectQuery)),
    switchMap(([, query]) =>
      this.productsService.search(query).pipe(
        map((products) =>
          ProductsApiActions.productsLoadedSuccessfully({ products })
        ),
        catchError((error) =>
          of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

---

## Router Effects

### Navigation After Action

```typescript
import { Router } from '@angular/router';

@Injectable()
export class AuthEffects {
  private readonly actions$ = inject(Actions);
  private readonly router = inject(Router);

  readonly redirectAfterLogin$ = createEffect(() =>
    this.actions$.pipe(
      ofType(AuthApiActions.loginSuccess),
      tap(() => this.router.navigate(['/dashboard']))
    ),
    { dispatch: false }
  );

  readonly redirectAfterLogout$ = createEffect(() =>
    this.actions$.pipe(
      ofType(AuthActions.logoutConfirmed),
      tap(() => this.router.navigate(['/login']))
    ),
    { dispatch: false }
  );
}
```

### Listening to Router Actions

```typescript
import { routerNavigatedAction } from '@ngrx/router-store';

readonly loadOnNavigation$ = createEffect(() =>
  this.actions$.pipe(
    ofType(routerNavigatedAction),
    filter(({ payload }) =>
      payload.routerState.url.startsWith('/products')
    ),
    map(() => ProductsPageActions.opened())
  )
);
```

---

## Accessing State in Effects

### concatLatestFrom Pattern

```typescript
import { concatLatestFrom } from '@ngrx/operators';

readonly updateProduct$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.saveClicked),
    concatLatestFrom(() => [
      this.store.select(selectSelectedProductId),
      this.store.select(selectFormChanges),
    ]),
    filter(([, id]) => id !== null),
    concatMap(([, id, changes]) =>
      this.productsService.update(id!, changes).pipe(
        map((product) =>
          ProductsApiActions.productUpdatedSuccessfully({ product })
        ),
        catchError((error) =>
          of(ProductsApiActions.productUpdatedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

---

## Testing Effects

### Basic Effect Test with TestBed

```typescript
// products.effects.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { provideMockStore } from '@ngrx/store/testing';
import { Observable, of, throwError } from 'rxjs';
import { hot, cold } from 'jasmine-marbles';
import { ProductsEffects } from './products.effects';
import { ProductsService } from '../services/products.service';
import { ProductsPageActions, ProductsApiActions } from './products.actions';

describe('ProductsEffects', () => {
  let effects: ProductsEffects;
  let actions$: Observable<any>;
  let productsService: jasmine.SpyObj<ProductsService>;

  beforeEach(() => {
    productsService = jasmine.createSpyObj('ProductsService', ['getAll', 'create', 'delete']);

    TestBed.configureTestingModule({
      providers: [
        ProductsEffects,
        provideMockActions(() => actions$),
        provideMockStore(),
        { provide: ProductsService, useValue: productsService },
      ],
    });

    effects = TestBed.inject(ProductsEffects);
  });

  describe('loadProducts$', () => {
    it('should return productsLoadedSuccessfully on success', () => {
      const products = [{ id: '1', name: 'Product A', price: 29.99 }];
      const action = ProductsPageActions.opened();
      const outcome = ProductsApiActions.productsLoadedSuccessfully({ products });

      actions$ = hot('-a', { a: action });
      productsService.getAll.and.returnValue(cold('-b|', { b: products }));

      expect(effects.loadProducts$).toBeObservable(
        hot('--b', { b: outcome })
      );
    });

    it('should return productsLoadedFailure on error', () => {
      const action = ProductsPageActions.opened();
      const error = new Error('Network error');
      const outcome = ProductsApiActions.productsLoadedFailure({
        error: 'Network error',
      });

      actions$ = hot('-a', { a: action });
      productsService.getAll.and.returnValue(cold('-#|', {}, error));

      expect(effects.loadProducts$).toBeObservable(
        hot('--b', { b: outcome })
      );
    });
  });
});
```

### Testing Non-Dispatching Effects

```typescript
describe('navigateToProducts$', () => {
  let router: jasmine.SpyObj<Router>;

  beforeEach(() => {
    router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
  });

  it('should navigate to the created product', (done) => {
    const product = { id: '123', name: 'New Product', price: 19.99 };
    const action = ProductsApiActions.productCreatedSuccessfully({ product });
    actions$ = of(action);

    effects.navigateToProducts$.subscribe(() => {
      expect(router.navigate).toHaveBeenCalledWith(['/products', '123']);
      done();
    });
  });
});
```

---

## Best Practices

### Effect Organization

| Guideline | Description |
|-----------|-------------|
| One effect per side effect | Each effect handles a single concern |
| Group related effects in one class | All product-related effects in `ProductsEffects` |
| Use descriptive names | `loadProducts$`, not `effect1$` |
| Keep effects thin | Delegate business logic to services |
| Prefer functional effects | Use `{ functional: true }` for simpler, testable effects |

### Operator Selection Guide

| Scenario | Operator | Why |
|----------|----------|-----|
| GET request / data fetch | `switchMap` | Cancel stale requests |
| POST / create resource | `concatMap` | Preserve order |
| DELETE / fire-and-forget | `mergeMap` | Run in parallel |
| Login / form submit | `exhaustMap` | Prevent double-submit |
| Search / autocomplete | `switchMap` + `debounceTime` | Cancel outdated queries |

### Error Handling Rules

1. Always catch errors inside the inner observable
2. Return an observable (not throw) from `catchError`
3. Dispatch failure actions so reducers can update error state
4. Consider retry with exponential backoff for transient failures
5. Use a global error effect for consistent error notification

---

## Common Pitfalls

### 1. catchError Outside the Inner Observable

```typescript
// WRONG: Effect stream dies on first error
readonly loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.opened),
    exhaustMap(() => this.productsService.getAll()),
    map((products) => ProductsApiActions.productsLoadedSuccessfully({ products })),
    catchError((error) =>
      of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
    ) // After this error, the effect never fires again!
  )
);

// CORRECT: catchError inside inner pipe
readonly loadProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.opened),
    exhaustMap(() =>
      this.productsService.getAll().pipe(
        map((products) =>
          ProductsApiActions.productsLoadedSuccessfully({ products })
        ),
        catchError((error) =>
          of(ProductsApiActions.productsLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### 2. Using switchMap for Mutations

```typescript
// WRONG: switchMap cancels previous create if user clicks quickly
readonly createProduct$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.addProduct),
    switchMap(({ product }) => // Previous request gets cancelled!
      this.productsService.create(product).pipe(/* ... */)
    )
  )
);

// CORRECT: concatMap queues and processes in order
readonly createProduct$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ProductsPageActions.addProduct),
    concatMap(({ product }) =>
      this.productsService.create(product).pipe(/* ... */)
    )
  )
);
```

### 3. Forgetting to Register Effects

```typescript
// WRONG: Effects class exists but is never registered
// No provideEffects(ProductsEffects) anywhere

// CORRECT: Register in route or app config
provideEffects(ProductsEffects)
```

### 4. Infinite Effect Loops

```typescript
// WRONG: Effect listens to action it dispatches
readonly infiniteLoop$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SomeActions.doSomething),
    map(() => SomeActions.doSomething()) // Dispatches the same action!
  )
);

// CORRECT: Dispatch a different action or use { dispatch: false }
readonly doSomething$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SomeActions.doSomething),
    map(() => SomeActions.somethingDone())
  )
);
```

### 5. withLatestFrom Eager Subscription

```typescript
// PROBLEMATIC: withLatestFrom subscribes eagerly, may get stale initial value
this.actions$.pipe(
  ofType(SomeAction),
  withLatestFrom(this.store.select(selectSomething)),
  // ...
)

// PREFERRED: concatLatestFrom subscribes lazily
this.actions$.pipe(
  ofType(SomeAction),
  concatLatestFrom(() => this.store.select(selectSomething)),
  // ...
)
```

### 6. Heavy Logic in Effects

```typescript
// WRONG: Business logic in effect
readonly processOrder$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrderActions.submit),
    switchMap(({ order }) => {
      const total = order.items.reduce((s, i) => s + i.price * i.qty, 0);
      const tax = total * 0.08;
      const discount = order.coupon ? total * 0.1 : 0;
      // ... more calculations
      return this.orderService.create({ ...order, total, tax, discount });
    })
  )
);

// CORRECT: Delegate logic to a service
readonly processOrder$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrderActions.submit),
    switchMap(({ order }) =>
      this.orderService.processAndCreate(order).pipe(/* ... */)
    )
  )
);
```
