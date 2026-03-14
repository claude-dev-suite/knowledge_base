# NgRx Store

> Official Documentation: https://ngrx.io/guide/store

Comprehensive reference for NgRx Store -- the reactive state management library for Angular applications built on RxJS. NgRx Store provides a single, immutable data store for your entire application state, following the Redux pattern of actions, reducers, and selectors.

---

## Table of Contents

1. [Store Setup](#store-setup)
2. [Actions with createActionGroup](#actions-with-createactiongroup)
3. [Reducers with createFeature](#reducers-with-createfeature)
4. [Selectors and Selector Composition](#selectors-and-selector-composition)
5. [Signal-Based Selection](#signal-based-selection)
6. [Store Injection and Dispatching](#store-injection-and-dispatching)
7. [Feature State Registration](#feature-state-registration)
8. [Immutable State Patterns](#immutable-state-patterns)
9. [Entity Patterns with Store](#entity-patterns-with-store)
10. [Meta-Reducers](#meta-reducers)
11. [Store DevTools](#store-devtools)
12. [Best Practices](#best-practices)
13. [Common Pitfalls](#common-pitfalls)

---

## Store Setup

### Application-Level Provider

Configure the root store in your application bootstrap or `app.config.ts`:

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { provideStoreDevtools } from '@ngrx/store-devtools';
import { isDevMode } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideEffects(),
    provideStoreDevtools({
      maxAge: 25,
      logOnly: !isDevMode(),
      autoPause: true,
      connectInZone: true,
    }),
  ],
};
```

### Bootstrap with Store

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

### Root State with Initial Reducers

When you have top-level state that should be available immediately:

```typescript
// app.config.ts
import { provideStore } from '@ngrx/store';
import { routerReducer } from '@ngrx/router-store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({
      router: routerReducer,
    }),
  ],
};
```

---

## Actions with createActionGroup

### Basic Action Group

`createActionGroup()` is the recommended way to define related actions as a group:

```typescript
// products.actions.ts
import { createActionGroup, emptyProps, props } from '@ngrx/store';
import { Product } from '../models/product.model';

export const ProductsPageActions = createActionGroup({
  source: 'Products Page',
  events: {
    'Opened': emptyProps(),
    'Search Changed': props<{ query: string }>(),
    'Sort Changed': props<{ sortBy: string; sortOrder: 'asc' | 'desc' }>(),
    'Page Changed': props<{ pageIndex: number; pageSize: number }>(),
    'Add Product': props<{ product: Omit<Product, 'id'> }>(),
    'Update Product': props<{ id: string; changes: Partial<Product> }>(),
    'Delete Product': props<{ id: string }>(),
  },
});

export const ProductsApiActions = createActionGroup({
  source: 'Products API',
  events: {
    'Products Loaded Successfully': props<{ products: Product[] }>(),
    'Products Loaded Failure': props<{ error: string }>(),
    'Product Created Successfully': props<{ product: Product }>(),
    'Product Created Failure': props<{ error: string }>(),
    'Product Updated Successfully': props<{ product: Product }>(),
    'Product Deleted Successfully': props<{ id: string }>(),
  },
});
```

### Dispatching Actions from Components

```typescript
import { ProductsPageActions } from './state/products.actions';

// Dispatching is straightforward:
this.store.dispatch(ProductsPageActions.opened());
this.store.dispatch(ProductsPageActions.searchChanged({ query: 'laptop' }));
this.store.dispatch(ProductsPageActions.deleteProduct({ id: '123' }));
```

### Single Actions (Less Common)

For standalone actions not part of a group:

```typescript
import { createAction, props } from '@ngrx/store';

export const appInitialized = createAction('[App] Initialized');
export const userLoggedOut = createAction(
  '[Auth] User Logged Out',
  props<{ reason: string }>()
);
```

---

## Reducers with createFeature

### createFeature (Recommended)

`createFeature()` automatically generates a feature reducer and selectors:

```typescript
// products.reducer.ts
import { createFeature, createReducer, on } from '@ngrx/store';
import { ProductsPageActions, ProductsApiActions } from './products.actions';
import { Product } from '../models/product.model';

export interface ProductsState {
  products: Product[];
  loading: boolean;
  error: string | null;
  query: string;
  selectedProductId: string | null;
  sortBy: string;
  sortOrder: 'asc' | 'desc';
}

const initialState: ProductsState = {
  products: [],
  loading: false,
  error: null,
  query: '',
  selectedProductId: null,
  sortBy: 'name',
  sortOrder: 'asc',
};

export const productsFeature = createFeature({
  name: 'products',
  reducer: createReducer(
    initialState,

    // Page actions
    on(ProductsPageActions.opened, (state) => ({
      ...state,
      loading: true,
      error: null,
    })),

    on(ProductsPageActions.searchChanged, (state, { query }) => ({
      ...state,
      query,
    })),

    on(ProductsPageActions.sortChanged, (state, { sortBy, sortOrder }) => ({
      ...state,
      sortBy,
      sortOrder,
    })),

    // API success actions
    on(ProductsApiActions.productsLoadedSuccessfully, (state, { products }) => ({
      ...state,
      products,
      loading: false,
      error: null,
    })),

    on(ProductsApiActions.productCreatedSuccessfully, (state, { product }) => ({
      ...state,
      products: [...state.products, product],
    })),

    on(ProductsApiActions.productUpdatedSuccessfully, (state, { product }) => ({
      ...state,
      products: state.products.map(p =>
        p.id === product.id ? product : p
      ),
    })),

    on(ProductsApiActions.productDeletedSuccessfully, (state, { id }) => ({
      ...state,
      products: state.products.filter(p => p.id !== id),
    })),

    // API failure actions
    on(ProductsApiActions.productsLoadedFailure, (state, { error }) => ({
      ...state,
      loading: false,
      error,
    })),
  ),
});

// createFeature auto-generates these selectors:
// productsFeature.selectProductsState  -> entire feature state
// productsFeature.selectProducts       -> state.products
// productsFeature.selectLoading        -> state.loading
// productsFeature.selectError          -> state.error
// productsFeature.selectQuery          -> state.query
// productsFeature.selectSelectedProductId -> state.selectedProductId
// productsFeature.selectSortBy         -> state.sortBy
// productsFeature.selectSortOrder      -> state.sortOrder

export const {
  selectProducts,
  selectLoading,
  selectError,
  selectQuery,
  selectSelectedProductId,
} = productsFeature;
```

### createReducer Standalone

When you need a reducer without the full `createFeature` wrapper:

```typescript
import { createReducer, on } from '@ngrx/store';

export interface CounterState {
  count: number;
}

const initialState: CounterState = { count: 0 };

export const counterReducer = createReducer(
  initialState,
  on(CounterActions.increment, (state) => ({ ...state, count: state.count + 1 })),
  on(CounterActions.decrement, (state) => ({ ...state, count: state.count - 1 })),
  on(CounterActions.reset, () => initialState),
  on(CounterActions.set, (state, { value }) => ({ ...state, count: value })),
);
```

---

## Selectors and Selector Composition

### createSelector

Compose complex derived state from simple selectors:

```typescript
// products.selectors.ts
import { createSelector } from '@ngrx/store';
import { productsFeature } from './products.reducer';

// Use auto-generated selectors from createFeature as building blocks
const { selectProducts, selectQuery, selectSortBy, selectSortOrder, selectSelectedProductId } = productsFeature;

// Derived selector: filtered products
export const selectFilteredProducts = createSelector(
  selectProducts,
  selectQuery,
  (products, query) => {
    if (!query) return products;
    const lowerQuery = query.toLowerCase();
    return products.filter(p =>
      p.name.toLowerCase().includes(lowerQuery) ||
      p.description.toLowerCase().includes(lowerQuery)
    );
  }
);

// Derived selector: sorted and filtered products
export const selectSortedFilteredProducts = createSelector(
  selectFilteredProducts,
  selectSortBy,
  selectSortOrder,
  (products, sortBy, sortOrder) => {
    return [...products].sort((a, b) => {
      const aVal = a[sortBy as keyof typeof a] ?? '';
      const bVal = b[sortBy as keyof typeof b] ?? '';
      const comparison = String(aVal).localeCompare(String(bVal));
      return sortOrder === 'asc' ? comparison : -comparison;
    });
  }
);

// Derived selector: currently selected product
export const selectSelectedProduct = createSelector(
  selectProducts,
  selectSelectedProductId,
  (products, selectedId) => products.find(p => p.id === selectedId) ?? null
);

// Derived selector: product statistics
export const selectProductStats = createSelector(
  selectProducts,
  (products) => ({
    total: products.length,
    averagePrice: products.length > 0
      ? products.reduce((sum, p) => sum + p.price, 0) / products.length
      : 0,
    outOfStock: products.filter(p => p.stock === 0).length,
  })
);
```

### Selector with Props (Parameterized)

Use factory selectors for parameterized selection:

```typescript
// Factory selector pattern
export const selectProductById = (id: string) =>
  createSelector(selectProducts, (products) =>
    products.find(p => p.id === id) ?? null
  );

// Usage in component
const product = this.store.selectSignal(selectProductById('abc-123'));
```

### Combining Selectors Across Features

```typescript
import { selectCurrentUser } from '../auth/auth.selectors';
import { selectProducts } from '../products/products.reducer';

export const selectUserProducts = createSelector(
  selectCurrentUser,
  selectProducts,
  (user, products) => {
    if (!user) return [];
    return products.filter(p => p.ownerId === user.id);
  }
);
```

---

## Signal-Based Selection

### selectSignal (Angular 17+)

Prefer `selectSignal()` in modern Angular components for fine-grained reactivity:

```typescript
// product-list.component.ts
import { Component, inject } from '@angular/core';
import { Store } from '@ngrx/store';
import { selectSortedFilteredProducts, selectProductStats } from './state/products.selectors';
import { productsFeature } from './state/products.reducer';
import { ProductsPageActions } from './state/products.actions';

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    <div class="stats">
      <span>Total: {{ stats().total }}</span>
      <span>Avg Price: {{ stats().averagePrice | currency }}</span>
    </div>

    @if (loading()) {
      <app-spinner />
    } @else {
      @for (product of products(); track product.id) {
        <app-product-card
          [product]="product"
          (delete)="onDelete(product.id)"
        />
      } @empty {
        <p>No products found.</p>
      }
    }
  `,
})
export class ProductListComponent {
  private readonly store = inject(Store);

  // Signal-based selectors - auto-update reactively
  readonly products = this.store.selectSignal(selectSortedFilteredProducts);
  readonly loading = this.store.selectSignal(productsFeature.selectLoading);
  readonly error = this.store.selectSignal(productsFeature.selectError);
  readonly stats = this.store.selectSignal(selectProductStats);

  onDelete(id: string): void {
    this.store.dispatch(ProductsPageActions.deleteProduct({ id }));
  }
}
```

### Observable-Based Selection (Legacy)

For older codebases or when working with RxJS pipes:

```typescript
import { AsyncPipe } from '@angular/common';

@Component({
  imports: [AsyncPipe],
  template: `
    @for (product of products$ | async; track product.id) {
      <app-product-card [product]="product" />
    }
  `,
})
export class ProductListComponent {
  private readonly store = inject(Store);
  readonly products$ = this.store.select(selectSortedFilteredProducts);
}
```

---

## Store Injection and Dispatching

### Modern Injection Pattern

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';

@Component({ /* ... */ })
export class DashboardComponent implements OnInit {
  private readonly store = inject(Store);

  readonly metrics = this.store.selectSignal(selectDashboardMetrics);
  readonly isLoading = this.store.selectSignal(selectLoading);

  ngOnInit(): void {
    this.store.dispatch(DashboardPageActions.opened());
  }

  refresh(): void {
    this.store.dispatch(DashboardPageActions.refreshClicked());
  }
}
```

### Dispatching Multiple Actions

```typescript
// Dispatch multiple actions in sequence
onCheckout(cart: CartItem[]): void {
  this.store.dispatch(CartActions.checkoutStarted({ items: cart }));
  this.store.dispatch(AnalyticsActions.checkoutTracked({ itemCount: cart.length }));
}
```

---

## Feature State Registration

### Lazy-Loaded Feature State

Register feature state when a route module loads:

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
    children: [
      {
        path: '',
        component: ProductListComponent,
      },
      {
        path: ':id',
        component: ProductDetailComponent,
      },
    ],
  },
];
```

### Eager Feature State

For features needed at application startup:

```typescript
// app.config.ts
import { provideStore, provideState } from '@ngrx/store';
import { authFeature } from './auth/state/auth.reducer';
import { AuthEffects } from './auth/state/auth.effects';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideState(authFeature),
    provideEffects(AuthEffects),
  ],
};
```

---

## Immutable State Patterns

### Spread Operator for Object Updates

```typescript
on(UserActions.updateProfile, (state, { name, email }) => ({
  ...state,
  user: {
    ...state.user,
    name,
    email,
    updatedAt: new Date().toISOString(),
  },
}))
```

### Array Operations (Immutable)

| Operation | Immutable Pattern |
|-----------|-------------------|
| Add item | `[...state.items, newItem]` |
| Prepend item | `[newItem, ...state.items]` |
| Remove item | `state.items.filter(i => i.id !== id)` |
| Update item | `state.items.map(i => i.id === id ? { ...i, ...changes } : i)` |
| Replace all | `newItems` (direct replacement) |
| Sort | `[...state.items].sort(compareFn)` |

```typescript
// Add to array
on(TodoActions.addTodo, (state, { todo }) => ({
  ...state,
  todos: [...state.todos, todo],
})),

// Remove from array
on(TodoActions.removeTodo, (state, { id }) => ({
  ...state,
  todos: state.todos.filter(t => t.id !== id),
})),

// Update item in array
on(TodoActions.toggleTodo, (state, { id }) => ({
  ...state,
  todos: state.todos.map(t =>
    t.id === id ? { ...t, completed: !t.completed } : t
  ),
})),

// Nested update
on(OrderActions.updateLineItem, (state, { orderId, lineItemId, quantity }) => ({
  ...state,
  orders: state.orders.map(order =>
    order.id === orderId
      ? {
          ...order,
          lineItems: order.lineItems.map(item =>
            item.id === lineItemId ? { ...item, quantity } : item
          ),
        }
      : order
  ),
})),
```

---

## Entity Patterns with Store

### Using NgRx Entity Adapter in Reducers

Combine `@ngrx/entity` with `createFeature`:

```typescript
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';
import { createFeature, createReducer, on } from '@ngrx/store';
import { Product } from '../models/product.model';

export interface ProductsState extends EntityState<Product> {
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

export const productsAdapter: EntityAdapter<Product> = createEntityAdapter<Product>({
  selectId: (product) => product.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

const initialState: ProductsState = productsAdapter.getInitialState({
  loading: false,
  error: null,
  selectedId: null,
});

export const productsFeature = createFeature({
  name: 'products',
  reducer: createReducer(
    initialState,
    on(ProductsApiActions.productsLoadedSuccessfully, (state, { products }) =>
      productsAdapter.setAll(products, { ...state, loading: false })
    ),
    on(ProductsApiActions.productCreatedSuccessfully, (state, { product }) =>
      productsAdapter.addOne(product, state)
    ),
    on(ProductsApiActions.productUpdatedSuccessfully, (state, { product }) =>
      productsAdapter.upsertOne(product, state)
    ),
    on(ProductsApiActions.productDeletedSuccessfully, (state, { id }) =>
      productsAdapter.removeOne(id, state)
    ),
  ),
  extraSelectors: ({ selectProductsState }) => ({
    ...productsAdapter.getSelectors(selectProductsState),
  }),
});

// Now you have: productsFeature.selectAll, selectEntities, selectIds, selectTotal
```

---

## Meta-Reducers

Meta-reducers are higher-order reducers that process actions before normal reducers.

### Logger Meta-Reducer

```typescript
// meta-reducers.ts
import { ActionReducer, MetaReducer } from '@ngrx/store';
import { isDevMode } from '@angular/core';

export function logger(reducer: ActionReducer<any>): ActionReducer<any> {
  return (state, action) => {
    const nextState = reducer(state, action);
    console.groupCollapsed(`[NgRx] ${action.type}`);
    console.log('Previous State:', state);
    console.log('Action:', action);
    console.log('Next State:', nextState);
    console.groupEnd();
    return nextState;
  };
}

export const metaReducers: MetaReducer[] = isDevMode() ? [logger] : [];
```

### Hydration Meta-Reducer (localStorage Persistence)

```typescript
export function hydrationMetaReducer(
  reducer: ActionReducer<any>
): ActionReducer<any> {
  return (state, action) => {
    if (action.type === '@ngrx/store/init') {
      const storedState = localStorage.getItem('appState');
      if (storedState) {
        try {
          return JSON.parse(storedState);
        } catch {
          localStorage.removeItem('appState');
        }
      }
    }
    const nextState = reducer(state, action);
    localStorage.setItem('appState', JSON.stringify(nextState));
    return nextState;
  };
}
```

### Registering Meta-Reducers

```typescript
// app.config.ts
import { provideStore } from '@ngrx/store';
import { metaReducers } from './state/meta-reducers';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({}, { metaReducers }),
  ],
};
```

---

## Store DevTools

### Setup

```typescript
import { provideStoreDevtools } from '@ngrx/store-devtools';
import { isDevMode } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStoreDevtools({
      maxAge: 50,               // Number of actions to retain
      logOnly: !isDevMode(),    // Restrict to log-only in production
      autoPause: true,          // Pause when devtools not open
      connectInZone: true,      // Connect inside Angular zone for change detection
      trace: isDevMode(),       // Include stack trace for each action
      traceLimit: 75,           // Max stack frames to capture
    }),
  ],
};
```

### DevTools Features

| Feature | Description |
|---------|-------------|
| **Action Log** | View all dispatched actions in order |
| **State Diff** | See what changed in state after each action |
| **Time Travel** | Jump to any previous state snapshot |
| **Action Replay** | Replay actions from a starting point |
| **State Export/Import** | Save and restore state snapshots |
| **Skip/Jump** | Skip individual actions or jump to a state |

---

## Best Practices

### State Shape Design

| Guideline | Reason |
|-----------|--------|
| Keep state flat and normalized | Avoid deeply nested structures that are hard to update immutably |
| Use entity adapter for collections | Built-in CRUD operations and selectors |
| Store only serializable data | Dates as ISO strings, not `Date` objects |
| Separate UI state from domain state | Loading flags, selected IDs in dedicated slices |
| Derive computed values via selectors | Never store values that can be computed from other state |

### Action Hygiene

```typescript
// GOOD: Descriptive source and event names
export const ProductsPageActions = createActionGroup({
  source: 'Products Page',
  events: {
    'Opened': emptyProps(),
    'Filter Changed': props<{ filter: ProductFilter }>(),
  },
});

// BAD: Generic or imperative action names
export const setProducts = createAction('SET_PRODUCTS', props<{ data: any }>());
```

### Selector Memoization

```typescript
// GOOD: Small, composable selectors that benefit from memoization
export const selectActiveProducts = createSelector(
  selectProducts,
  (products) => products.filter(p => p.active)
);

export const selectActiveProductCount = createSelector(
  selectActiveProducts,
  (products) => products.length
);

// BAD: One giant selector doing everything
export const selectEverything = createSelector(
  selectProductsState,
  (state) => ({
    filtered: state.products.filter(p => p.active && p.name.includes(state.query)),
    count: state.products.length,
    averagePrice: state.products.reduce((s, p) => s + p.price, 0) / state.products.length,
  })
);
```

### File Organization

```
feature/
  state/
    feature.actions.ts      # createActionGroup definitions
    feature.reducer.ts       # createFeature with state and reducer
    feature.selectors.ts     # Composed selectors
    feature.effects.ts       # Side effects
    feature.model.ts         # Interfaces
  components/
    feature-list.component.ts
    feature-detail.component.ts
  feature.routes.ts          # Route config with provideState/provideEffects
```

---

## Common Pitfalls

### 1. Mutating State Directly

```typescript
// WRONG: Mutates existing array
on(TodoActions.addTodo, (state, { todo }) => {
  state.todos.push(todo); // Mutation - will not trigger change detection
  return state;
})

// CORRECT: Return a new object
on(TodoActions.addTodo, (state, { todo }) => ({
  ...state,
  todos: [...state.todos, todo],
}))
```

### 2. Selecting in Effects Instead of Components

```typescript
// WRONG: Selecting state in an effect to push to component
someEffect$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SomeAction),
    withLatestFrom(this.store.select(selectSomething)),
    map(([action, data]) => SomeActions.setData({ data }))  // Unnecessary round-trip
  )
);

// CORRECT: Select directly in the component
readonly data = this.store.selectSignal(selectSomething);
```

### 3. Storing Non-Serializable Values

```typescript
// WRONG: Date objects, class instances, functions
interface BadState {
  createdAt: Date;             // Not serializable
  callback: () => void;        // Not serializable
  instance: MyService;         // Not serializable
}

// CORRECT: Serializable values only
interface GoodState {
  createdAt: string;           // ISO string
  callbackId: string;          // Reference by ID
  serviceResult: ServiceData;  // Plain data
}
```

### 4. Over-Dispatching Actions

```typescript
// WRONG: Dispatching in rapid succession (e.g., in a loop)
items.forEach(item => {
  this.store.dispatch(ItemActions.addItem({ item }));
});

// CORRECT: Batch into a single action
this.store.dispatch(ItemActions.addItems({ items }));
```

### 5. Fat Actions vs Fat Reducers

```typescript
// WRONG: Performing logic in the component before dispatching
const filteredProducts = products.filter(p => p.active);
const sortedProducts = filteredProducts.sort((a, b) => a.name.localeCompare(b.name));
this.store.dispatch(ProductActions.setProducts({ products: sortedProducts }));

// CORRECT: Dispatch raw data, let selectors derive
this.store.dispatch(ProductsApiActions.productsLoadedSuccessfully({ products }));
// Filtering and sorting handled by selectSortedFilteredProducts selector
```

### 6. Forgetting to Unsubscribe (Observable Pattern)

```typescript
// WRONG: Memory leak with manual subscribe
ngOnInit() {
  this.store.select(selectProducts).subscribe(products => {
    this.products = products;
  });
}

// CORRECT: Use selectSignal (no subscription needed)
readonly products = this.store.selectSignal(selectProducts);

// ALSO CORRECT: Use async pipe in template
// products$ = this.store.select(selectProducts);
// template: {{ products$ | async }}
```

### 7. Not Providing Feature State for Lazy Routes

```typescript
// WRONG: Feature selectors return undefined if state not registered
// (forgetting provideState in route config)

// CORRECT: Always register with the route
export const FEATURE_ROUTES: Routes = [
  {
    path: '',
    providers: [
      provideState(productsFeature),   // Required!
      provideEffects(ProductsEffects), // Required!
    ],
    component: ProductShellComponent,
  },
];
```
