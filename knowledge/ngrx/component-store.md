# NgRx ComponentStore

> Official Documentation: https://ngrx.io/guide/component-store

Comprehensive reference for NgRx ComponentStore -- a standalone, reactive state management solution for Angular components and services. ComponentStore provides a lightweight alternative to the global NgRx Store for managing local or scoped state without actions, reducers, or effects boilerplate.

---

## Table of Contents

1. [ComponentStore Setup](#componentstore-setup)
2. [State Initialization](#state-initialization)
3. [select() for State Slicing](#select-for-state-slicing)
4. [selectSignal() for Signal-Based Selection](#selectsignal-for-signal-based-selection)
5. [updater() for Synchronous State Changes](#updater-for-synchronous-state-changes)
6. [patchState() for Partial Updates](#patchstate-for-partial-updates)
7. [effect() for Async Operations](#effect-for-async-operations)
8. [tapResponse for Effect Error Handling](#tapresponse-for-effect-error-handling)
9. [Combining Selectors](#combining-selectors)
10. [Lifecycle Hooks](#lifecycle-hooks)
11. [ComponentStore vs Global Store](#componentstore-vs-global-store)
12. [Testing ComponentStore](#testing-componentstore)
13. [Best Practices](#best-practices)
14. [Common Pitfalls](#common-pitfalls)

---

## ComponentStore Setup

### Basic ComponentStore Class

```typescript
// products.store.ts
import { Injectable } from '@angular/core';
import { ComponentStore } from '@ngrx/component-store';

export interface ProductsState {
  products: Product[];
  loading: boolean;
  error: string | null;
  selectedProductId: string | null;
  filter: string;
}

const initialState: ProductsState = {
  products: [],
  loading: false,
  error: null,
  selectedProductId: null,
  filter: '',
};

@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  constructor() {
    super(initialState);
  }
}
```

### Providing in a Component

ComponentStore is typically scoped to a component and its children:

```typescript
// product-list.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { ProductsStore } from './products.store';

@Component({
  selector: 'app-product-list',
  standalone: true,
  providers: [ProductsStore],  // Scoped to this component and children
  template: `
    @if (store.loading()) {
      <app-spinner />
    } @else {
      @for (product of store.filteredProducts(); track product.id) {
        <app-product-card [product]="product" (click)="store.selectProduct(product.id)" />
      }
    }
  `,
})
export class ProductListComponent implements OnInit {
  readonly store = inject(ProductsStore);

  ngOnInit(): void {
    this.store.loadProducts();
  }
}
```

### Providing at Route Level

For state shared across route children:

```typescript
// products.routes.ts
export const PRODUCTS_ROUTES: Routes = [
  {
    path: '',
    providers: [ProductsStore],  // Shared across all child routes
    children: [
      { path: '', component: ProductListComponent },
      { path: ':id', component: ProductDetailComponent },
    ],
  },
];
```

---

## State Initialization

### Constructor Initialization

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  constructor() {
    super(initialState);
  }
}
```

### Lazy Initialization with setState

When initial state depends on async data or parent input:

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  constructor() {
    super(); // No initial state -- lazy init
  }

  // Called from the component after data is available
  initialize(products: Product[]): void {
    this.setState({
      products,
      loading: false,
      error: null,
      selectedProductId: null,
      filter: '',
    });
  }
}
```

### Initialization from Inputs

```typescript
@Component({
  providers: [ProductDetailStore],
})
export class ProductDetailComponent {
  private readonly store = inject(ProductDetailStore);
  private readonly productId = input.required<string>();

  constructor() {
    effect(() => {
      this.store.loadProduct(this.productId());
    });
  }
}
```

---

## select() for State Slicing

### Basic Selectors

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Select a single property (returns Observable)
  readonly products$ = this.select((state) => state.products);
  readonly loading$ = this.select((state) => state.loading);
  readonly error$ = this.select((state) => state.error);
  readonly filter$ = this.select((state) => state.filter);
  readonly selectedProductId$ = this.select((state) => state.selectedProductId);
}
```

### Derived Selectors

Combine multiple selectors to compute derived state:

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Primitive selectors
  readonly products$ = this.select((state) => state.products);
  readonly filter$ = this.select((state) => state.filter);
  readonly selectedProductId$ = this.select((state) => state.selectedProductId);

  // Derived: filtered products
  readonly filteredProducts$ = this.select(
    this.products$,
    this.filter$,
    (products, filter) => {
      if (!filter) return products;
      const lowerFilter = filter.toLowerCase();
      return products.filter(
        (p) =>
          p.name.toLowerCase().includes(lowerFilter) ||
          p.category.toLowerCase().includes(lowerFilter)
      );
    }
  );

  // Derived: currently selected product
  readonly selectedProduct$ = this.select(
    this.products$,
    this.selectedProductId$,
    (products, selectedId) =>
      products.find((p) => p.id === selectedId) ?? null
  );

  // Derived: statistics
  readonly stats$ = this.select(this.products$, (products) => ({
    total: products.length,
    avgPrice:
      products.length > 0
        ? products.reduce((sum, p) => sum + p.price, 0) / products.length
        : 0,
    categories: [...new Set(products.map((p) => p.category))].length,
  }));
}
```

### Select with Debounce

```typescript
readonly filteredProducts$ = this.select(
  this.products$,
  this.filter$,
  (products, filter) => {
    if (!filter) return products;
    return products.filter((p) =>
      p.name.toLowerCase().includes(filter.toLowerCase())
    );
  },
  { debounce: true } // Debounces by one microtask to avoid synchronous emissions
);
```

---

## selectSignal() for Signal-Based Selection

### Basic Signal Selectors (Angular 17+)

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Signal-based selectors -- no need for async pipe in templates
  readonly products = this.selectSignal((state) => state.products);
  readonly loading = this.selectSignal((state) => state.loading);
  readonly error = this.selectSignal((state) => state.error);
  readonly filter = this.selectSignal((state) => state.filter);
  readonly selectedProductId = this.selectSignal((state) => state.selectedProductId);

  // Derived signal selectors
  readonly filteredProducts = this.selectSignal(
    this.products,
    this.filter,
    (products, filter) => {
      if (!filter) return products;
      const lowerFilter = filter.toLowerCase();
      return products.filter(
        (p) =>
          p.name.toLowerCase().includes(lowerFilter) ||
          p.category.toLowerCase().includes(lowerFilter)
      );
    }
  );

  readonly selectedProduct = this.selectSignal(
    this.products,
    this.selectedProductId,
    (products, id) => products.find((p) => p.id === id) ?? null
  );

  readonly productCount = this.selectSignal(
    this.filteredProducts,
    (products) => products.length
  );
}
```

### Using Signals in Templates

```typescript
@Component({
  selector: 'app-product-list',
  standalone: true,
  providers: [ProductsStore],
  template: `
    <input
      [value]="store.filter()"
      (input)="store.updateFilter($event.target.value)"
      placeholder="Search products..."
    />

    <span>{{ store.productCount() }} products found</span>

    @if (store.loading()) {
      <app-spinner />
    } @else if (store.error(); as error) {
      <app-error-message [message]="error" />
    } @else {
      @for (product of store.filteredProducts(); track product.id) {
        <app-product-card
          [product]="product"
          [selected]="product.id === store.selectedProductId()"
          (click)="store.selectProduct(product.id)"
        />
      } @empty {
        <p>No products match your filter.</p>
      }
    }
  `,
})
export class ProductListComponent implements OnInit {
  readonly store = inject(ProductsStore);

  ngOnInit(): void {
    this.store.loadProducts();
  }
}
```

---

## updater() for Synchronous State Changes

### Basic Updaters

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Simple property update
  readonly updateFilter = this.updater((state, filter: string) => ({
    ...state,
    filter,
  }));

  // Select a product
  readonly selectProduct = this.updater((state, productId: string) => ({
    ...state,
    selectedProductId: productId,
  }));

  // Set loading state
  readonly setLoading = this.updater((state, loading: boolean) => ({
    ...state,
    loading,
  }));

  // Add a product to the list
  readonly addProduct = this.updater((state, product: Product) => ({
    ...state,
    products: [...state.products, product],
  }));

  // Remove a product
  readonly removeProduct = this.updater((state, productId: string) => ({
    ...state,
    products: state.products.filter((p) => p.id !== productId),
  }));

  // Update a product in place
  readonly updateProduct = this.updater(
    (state, updated: { id: string; changes: Partial<Product> }) => ({
      ...state,
      products: state.products.map((p) =>
        p.id === updated.id ? { ...p, ...updated.changes } : p
      ),
    })
  );

  // Clear error
  readonly clearError = this.updater((state) => ({
    ...state,
    error: null,
  }));
}
```

### Updater Accepting Observable

Updaters can also accept an Observable source for reactive updates:

```typescript
@Injectable()
export class SearchStore extends ComponentStore<SearchState> {
  readonly updateQuery = this.updater((state, query: string) => ({
    ...state,
    query,
  }));

  // Feed an Observable directly into the updater
  connectSearchInput(searchInput$: Observable<string>): void {
    this.updateQuery(
      searchInput$.pipe(debounceTime(300), distinctUntilChanged())
    );
  }
}
```

---

## patchState() for Partial Updates

`patchState` provides a simpler way to update state when you only need to set known properties:

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Direct partial updates without defining updaters
  setLoading(): void {
    this.patchState({ loading: true, error: null });
  }

  setProducts(products: Product[]): void {
    this.patchState({ products, loading: false });
  }

  setError(error: string): void {
    this.patchState({ loading: false, error });
  }

  updateFilter(filter: string): void {
    this.patchState({ filter });
  }

  selectProduct(selectedProductId: string | null): void {
    this.patchState({ selectedProductId });
  }
}
```

### patchState with Callback

For updates that depend on current state:

```typescript
toggleProductActive(productId: string): void {
  this.patchState((state) => ({
    products: state.products.map((p) =>
      p.id === productId ? { ...p, active: !p.active } : p
    ),
  }));
}

incrementPage(): void {
  this.patchState((state) => ({
    currentPage: state.currentPage + 1,
  }));
}
```

---

## effect() for Async Operations

### Basic HTTP Effect

```typescript
import { tapResponse } from '@ngrx/operators';
import { switchMap, tap } from 'rxjs';

@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  private readonly productsService = inject(ProductsService);

  // Effect triggered by void (no params)
  readonly loadProducts = this.effect<void>((trigger$) =>
    trigger$.pipe(
      tap(() => this.patchState({ loading: true, error: null })),
      switchMap(() =>
        this.productsService.getAll().pipe(
          tapResponse(
            (products) => this.patchState({ products, loading: false }),
            (error: HttpErrorResponse) =>
              this.patchState({ loading: false, error: error.message })
          )
        )
      )
    )
  );

  // Effect with parameter
  readonly loadProductsByCategory = this.effect<string>((category$) =>
    category$.pipe(
      tap(() => this.patchState({ loading: true })),
      switchMap((category) =>
        this.productsService.getByCategory(category).pipe(
          tapResponse(
            (products) => this.patchState({ products, loading: false }),
            (error: HttpErrorResponse) =>
              this.patchState({ loading: false, error: error.message })
          )
        )
      )
    )
  );

  // Effect with complex parameter
  readonly createProduct = this.effect<Omit<Product, 'id'>>((product$) =>
    product$.pipe(
      switchMap((productData) =>
        this.productsService.create(productData).pipe(
          tapResponse(
            (created) =>
              this.patchState((state) => ({
                products: [...state.products, created],
              })),
            (error: HttpErrorResponse) =>
              this.patchState({ error: error.message })
          )
        )
      )
    )
  );

  // Effect for deleting
  readonly deleteProduct = this.effect<string>((id$) =>
    id$.pipe(
      switchMap((id) =>
        this.productsService.delete(id).pipe(
          tapResponse(
            () =>
              this.patchState((state) => ({
                products: state.products.filter((p) => p.id !== id),
              })),
            (error: HttpErrorResponse) =>
              this.patchState({ error: error.message })
          )
        )
      )
    )
  );
}
```

### Effect with Observable Input

Effects can accept Observables directly:

```typescript
@Component({
  providers: [ProductsStore],
})
export class ProductListComponent implements OnInit {
  private readonly store = inject(ProductsStore);
  private readonly route = inject(ActivatedRoute);

  ngOnInit(): void {
    // Pass route param observable directly to effect
    this.store.loadProductsByCategory(
      this.route.params.pipe(map((params) => params['category']))
    );
  }
}
```

---

## tapResponse for Effect Error Handling

`tapResponse` is the recommended way to handle success and error in effects, preventing the effect from completing on error:

```typescript
import { tapResponse } from '@ngrx/operators';

readonly searchProducts = this.effect<string>((query$) =>
  query$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    tap((query) => this.patchState({ loading: true, filter: query })),
    switchMap((query) =>
      this.productsService.search(query).pipe(
        tapResponse({
          next: (products) => this.patchState({ products, loading: false }),
          error: (error: HttpErrorResponse) => {
            console.error('Search failed:', error);
            this.patchState({ loading: false, error: error.message });
          },
          finalize: () => console.log('Search request completed'),
        })
      )
    )
  )
);
```

### tapResponse Overloads

```typescript
// Two-argument form (next, error)
tapResponse(
  (products: Product[]) => this.setProducts(products),
  (error: HttpErrorResponse) => this.setError(error.message)
)

// Object form with optional finalize
tapResponse({
  next: (products: Product[]) => this.setProducts(products),
  error: (error: HttpErrorResponse) => this.setError(error.message),
  finalize: () => this.setLoading(false),
})
```

---

## Combining Selectors

### Composing Multiple Sources

```typescript
@Injectable()
export class DashboardStore extends ComponentStore<DashboardState> {
  private readonly productsStore = inject(ProductsStore);
  private readonly ordersStore = inject(OrdersStore);

  // Combine selectors from different stores
  readonly dashboardSummary$ = this.select(
    this.productsStore.products$,
    this.ordersStore.recentOrders$,
    this.select((state) => state.notifications),
    (products, orders, notifications) => ({
      productCount: products.length,
      pendingOrders: orders.filter((o) => o.status === 'pending').length,
      unreadNotifications: notifications.filter((n) => !n.read).length,
    })
  );
}
```

### ViewModel Pattern

Create a single observable or signal that combines all data a component needs:

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // ViewModel pattern: single observable with all template data
  readonly vm$ = this.select(
    this.filteredProducts$,
    this.loading$,
    this.error$,
    this.selectedProduct$,
    this.stats$,
    (products, loading, error, selectedProduct, stats) => ({
      products,
      loading,
      error,
      selectedProduct,
      stats,
    })
  );

  // Or signal-based ViewModel
  readonly vm = this.selectSignal(
    this.filteredProducts,
    this.loading,
    this.error,
    this.selectedProduct,
    (products, loading, error, selectedProduct) => ({
      products,
      loading,
      error,
      selectedProduct,
    })
  );
}
```

---

## Lifecycle Hooks

### OnStoreInit

Called after the store is instantiated and initialized:

```typescript
import { OnStoreInit } from '@ngrx/component-store';

@Injectable()
export class ProductsStore
  extends ComponentStore<ProductsState>
  implements OnStoreInit
{
  ngrxOnStoreInit(): void {
    console.log('ProductsStore initialized');
    this.loadProducts();
  }
}
```

### OnStateInit

Called after state has been initialized (either via constructor or `setState`):

```typescript
import { OnStateInit } from '@ngrx/component-store';

@Injectable()
export class ProductsStore
  extends ComponentStore<ProductsState>
  implements OnStateInit
{
  constructor() {
    super(); // Lazy init -- no initial state yet
  }

  ngrxOnStateInit(): void {
    // Called after setState() is first called
    console.log('State initialized:', this.get());
    this.loadProducts();
  }
}
```

### Automatic Cleanup

ComponentStore automatically unsubscribes from all effects and selectors when it is destroyed (when the providing component is destroyed). No manual cleanup is needed.

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // This effect is automatically cleaned up when the component is destroyed
  readonly pollProducts = this.effect<void>((trigger$) =>
    trigger$.pipe(
      switchMap(() =>
        interval(30000).pipe(
          switchMap(() => this.productsService.getAll()),
          tapResponse(
            (products) => this.patchState({ products }),
            (error: HttpErrorResponse) => console.error(error)
          )
        )
      )
    )
  );
}
```

---

## ComponentStore vs Global Store

### Decision Guide

| Criteria | ComponentStore | Global Store (NgRx Store) |
|----------|---------------|--------------------------|
| **Scope** | Component/route level | Application-wide |
| **Lifetime** | Tied to component lifecycle | Lives for entire app session |
| **Boilerplate** | Minimal (no actions/reducers) | Actions, reducers, effects, selectors |
| **DevTools** | Not supported | Full Redux DevTools support |
| **Shared state** | Difficult across unrelated components | Easy via global selectors |
| **Testing** | Simpler, mock the store directly | Requires MockStore setup |
| **Time-travel debugging** | Not available | Supported |
| **Use cases** | Forms, modals, wizards, tables, local UI | Auth, user profile, cart, app-wide data |

### When to Use ComponentStore

- **Form state management** -- Multi-step forms, form validation state
- **Table/grid state** -- Sorting, filtering, pagination, row selection
- **Dialog/modal state** -- Local data for modals and overlays
- **Wizard/stepper state** -- Step tracking, intermediate data
- **Search with typeahead** -- Query state and results
- **Component-scoped cache** -- Data that should be re-fetched when navigating away

### When to Use Global Store

- **Authentication state** -- Current user, tokens, permissions
- **Shopping cart** -- Items persisted across routes
- **Application settings** -- Theme, language, preferences
- **Shared entities** -- Data used by multiple features
- **WebSocket/real-time data** -- Shared across components

---

## Testing ComponentStore

### Basic Store Test

```typescript
// products.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { ProductsStore } from './products.store';
import { ProductsService } from '../services/products.service';
import { of, throwError } from 'rxjs';

describe('ProductsStore', () => {
  let store: ProductsStore;
  let productsService: jasmine.SpyObj<ProductsService>;

  beforeEach(() => {
    productsService = jasmine.createSpyObj('ProductsService', [
      'getAll',
      'create',
      'delete',
    ]);

    TestBed.configureTestingModule({
      providers: [
        ProductsStore,
        { provide: ProductsService, useValue: productsService },
      ],
    });

    store = TestBed.inject(ProductsStore);
  });

  it('should initialize with default state', () => {
    expect(store.loading()).toBe(false);
    expect(store.products()).toEqual([]);
    expect(store.error()).toBeNull();
  });

  describe('updaters', () => {
    it('should update the filter', () => {
      store.updateFilter('laptop');
      expect(store.filter()).toBe('laptop');
    });

    it('should select a product', () => {
      store.selectProduct('abc-123');
      expect(store.selectedProductId()).toBe('abc-123');
    });
  });

  describe('effects', () => {
    it('should load products successfully', (done) => {
      const mockProducts = [
        { id: '1', name: 'Product A', price: 29.99, category: 'electronics', stock: 10 },
      ];
      productsService.getAll.and.returnValue(of(mockProducts));

      store.loadProducts();

      // Wait a tick for the effect to complete
      setTimeout(() => {
        expect(store.products()).toEqual(mockProducts);
        expect(store.loading()).toBe(false);
        expect(store.error()).toBeNull();
        done();
      });
    });

    it('should handle load error', (done) => {
      productsService.getAll.and.returnValue(
        throwError(() => ({ message: 'Network error' }))
      );

      store.loadProducts();

      setTimeout(() => {
        expect(store.products()).toEqual([]);
        expect(store.loading()).toBe(false);
        expect(store.error()).toBe('Network error');
        done();
      });
    });
  });

  describe('selectors', () => {
    it('should return filtered products', () => {
      store.patchState({
        products: [
          { id: '1', name: 'Laptop', price: 999, category: 'electronics', stock: 5 },
          { id: '2', name: 'Shirt', price: 29, category: 'clothing', stock: 50 },
          { id: '3', name: 'Laptop Stand', price: 49, category: 'accessories', stock: 20 },
        ],
        filter: 'laptop',
      });

      const filtered = store.filteredProducts();
      expect(filtered.length).toBe(2);
      expect(filtered.map((p) => p.name)).toEqual(['Laptop', 'Laptop Stand']);
    });
  });
});
```

### Testing with Subscriptions

```typescript
describe('observable selectors', () => {
  it('should emit filtered products', (done) => {
    store.patchState({
      products: [
        { id: '1', name: 'Laptop', price: 999, category: 'electronics', stock: 5 },
        { id: '2', name: 'Shirt', price: 29, category: 'clothing', stock: 50 },
      ],
    });

    store.filteredProducts$.subscribe((products) => {
      // Initially empty filter, all products returned
      expect(products.length).toBe(2);
      done();
    });
  });
});
```

---

## Best Practices

### Store Organization

| Guideline | Description |
|-----------|-------------|
| One store per feature/component | Avoid god stores with unrelated state |
| Co-locate store with component | Place store file next to its component |
| Expose signals, hide implementation | Public API is signals/updaters/effects |
| Use `patchState` for simple updates | Only define `updater()` for complex logic |
| Use `tapResponse` in all effects | Prevents silent stream death |
| Delegate HTTP logic to services | Store orchestrates, service communicates |

### Naming Conventions

```typescript
@Injectable()
export class ProductsStore extends ComponentStore<ProductsState> {
  // Selectors: noun or adjective (what the data IS)
  readonly products = this.selectSignal((s) => s.products);
  readonly loading = this.selectSignal((s) => s.loading);
  readonly filteredProducts = this.selectSignal(/* ... */);

  // Updaters: verb describing state change
  readonly updateFilter = this.updater(/* ... */);
  readonly selectProduct = this.updater(/* ... */);
  readonly clearError = this.updater(/* ... */);

  // Effects: verb describing side-effect
  readonly loadProducts = this.effect(/* ... */);
  readonly createProduct = this.effect(/* ... */);
  readonly deleteProduct = this.effect(/* ... */);
}
```

### ViewModel Pattern

```typescript
// GOOD: Single signal for template binding
readonly vm = this.selectSignal(
  this.products,
  this.loading,
  this.error,
  (products, loading, error) => ({ products, loading, error })
);

// Component template:
// @if (store.vm(); as vm) { ... vm.loading ... vm.products ... }
```

---

## Common Pitfalls

### 1. Selecting Before State Is Initialized

```typescript
// WRONG: Selecting from a lazily-initialized store before setState
@Injectable()
export class MyStore extends ComponentStore<MyState> {
  constructor() {
    super(); // No initial state
  }
  readonly items = this.selectSignal((s) => s.items); // Throws at runtime!
}

// CORRECT: Either provide initial state or check initialization
@Injectable()
export class MyStore extends ComponentStore<MyState> {
  constructor() {
    super({ items: [], loading: false }); // Always provide initial state
  }
  readonly items = this.selectSignal((s) => s.items);
}
```

### 2. Forgetting to Provide the Store

```typescript
// WRONG: Injecting without providing -- NullInjectorError
@Component({
  // Missing: providers: [ProductsStore]
})
export class ProductListComponent {
  readonly store = inject(ProductsStore); // Error!
}

// CORRECT: Provide in the component or a parent route
@Component({
  providers: [ProductsStore],
})
export class ProductListComponent {
  readonly store = inject(ProductsStore);
}
```

### 3. Not Using tapResponse in Effects

```typescript
// WRONG: catchError does not re-subscribe the inner stream
readonly loadProducts = this.effect<void>((trigger$) =>
  trigger$.pipe(
    switchMap(() =>
      this.productsService.getAll().pipe(
        tap((products) => this.patchState({ products })),
        catchError((error) => {
          this.patchState({ error: error.message });
          return EMPTY; // Stream completes -- effect dies silently
        })
      )
    )
  )
);

// CORRECT: tapResponse keeps the outer stream alive
readonly loadProducts = this.effect<void>((trigger$) =>
  trigger$.pipe(
    switchMap(() =>
      this.productsService.getAll().pipe(
        tapResponse(
          (products) => this.patchState({ products, loading: false }),
          (error: HttpErrorResponse) =>
            this.patchState({ loading: false, error: error.message })
        )
      )
    )
  )
);
```

### 4. Mutating State Directly

```typescript
// WRONG: Direct mutation
readonly addProduct = this.updater((state, product: Product) => {
  state.products.push(product); // Mutation!
  return state;
});

// CORRECT: Return a new state object
readonly addProduct = this.updater((state, product: Product) => ({
  ...state,
  products: [...state.products, product],
}));
```

### 5. Over-Scoping the Store

```typescript
// PROBLEMATIC: Store provided at root, but only used in one component
@Injectable({ providedIn: 'root' })
export class ProductsStore extends ComponentStore<ProductsState> {
  // State persists forever, even when user navigates away
}

// BETTER: Scope to the component for automatic cleanup
@Injectable() // Not providedIn: 'root'
export class ProductsStore extends ComponentStore<ProductsState> { }

// Provide in component:
@Component({ providers: [ProductsStore] })
export class ProductListComponent { }
```

### 6. Subscribing Manually to Selectors

```typescript
// WRONG: Manual subscription requires manual cleanup
ngOnInit(): void {
  this.store.products$.subscribe(products => {
    this.products = products; // Memory leak if not unsubscribed
  });
}

// CORRECT: Use selectSignal (auto-cleanup, no subscription)
readonly products = this.store.filteredProducts;
// In template: @for (product of products(); track product.id) { ... }

// ALSO CORRECT: Use async pipe (auto-unsubscribes)
// template: @for (product of products$ | async; track product.id) { ... }
```
