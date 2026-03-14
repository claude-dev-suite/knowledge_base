# NgRx Entity

> Official Documentation: https://ngrx.io/guide/entity

Comprehensive reference for NgRx Entity -- an adapter library for managing collections of records within NgRx Store. Entity provides a standardized, high-performance state shape for collections and pre-built reducer operations for CRUD, eliminating boilerplate and enforcing consistent patterns.

---

## Table of Contents

1. [Entity Adapter Setup](#entity-adapter-setup)
2. [EntityState Interface](#entitystate-interface)
3. [CRUD Operations](#crud-operations)
4. [Sorted vs Unsorted Collections](#sorted-vs-unsorted-collections)
5. [Built-in Selectors](#built-in-selectors)
6. [Integration with createFeature](#integration-with-createfeature)
7. [Entity Map Operations](#entity-map-operations)
8. [Normalized State Patterns](#normalized-state-patterns)
9. [Multiple Entity Collections](#multiple-entity-collections)
10. [Best Practices](#best-practices)
11. [Common Pitfalls](#common-pitfalls)

---

## Entity Adapter Setup

### Creating an Entity Adapter

```typescript
// product.model.ts
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  createdAt: string;
}
```

```typescript
// products.reducer.ts
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';
import { Product } from '../models/product.model';

// Basic adapter - uses 'id' property by default
export const productsAdapter: EntityAdapter<Product> = createEntityAdapter<Product>();
```

### Custom selectId

When your entity does not use `id` as the identifier:

```typescript
export interface OrderItem {
  sku: string;
  name: string;
  quantity: number;
  unitPrice: number;
}

export const orderItemsAdapter = createEntityAdapter<OrderItem>({
  selectId: (item) => item.sku,
});
```

### Custom sortComparer

Maintain entities in a sorted order within the store:

```typescript
export const productsAdapter = createEntityAdapter<Product>({
  selectId: (product) => product.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

// Sort by multiple fields
export const tasksAdapter = createEntityAdapter<Task>({
  sortComparer: (a, b) => {
    // First by priority (high to low)
    const priorityOrder = { high: 0, medium: 1, low: 2 };
    const priorityDiff = priorityOrder[a.priority] - priorityOrder[b.priority];
    if (priorityDiff !== 0) return priorityDiff;
    // Then by creation date (newest first)
    return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime();
  },
});
```

---

## EntityState Interface

The `EntityState<T>` interface provides a normalized collection shape:

```typescript
interface EntityState<T> {
  ids: string[] | number[];  // Ordered array of entity IDs
  entities: Dictionary<T>;   // Lookup table of entities by ID
}
```

### Extended State

Add custom properties alongside the entity collection:

```typescript
export interface ProductsState extends EntityState<Product> {
  loading: boolean;
  error: string | null;
  selectedProductId: string | null;
  filter: ProductFilter;
  pagination: {
    pageIndex: number;
    pageSize: number;
    totalCount: number;
  };
}

// Initialize with getInitialState
const initialState: ProductsState = productsAdapter.getInitialState({
  loading: false,
  error: null,
  selectedProductId: null,
  filter: { category: null, minPrice: 0, maxPrice: Infinity },
  pagination: { pageIndex: 0, pageSize: 20, totalCount: 0 },
});
```

---

## CRUD Operations

### Complete Operations Reference

| Method | Description | Input |
|--------|-------------|-------|
| `addOne` | Add one entity | `entity, state` |
| `addMany` | Add multiple entities | `entities[], state` |
| `setOne` | Add or replace one entity | `entity, state` |
| `setMany` | Add or replace multiple entities | `entities[], state` |
| `setAll` | Replace entire collection | `entities[], state` |
| `removeOne` | Remove one entity by ID | `id, state` |
| `removeMany` | Remove multiple entities by IDs or predicate | `ids[] \| predicate, state` |
| `removeAll` | Remove all entities | `state` |
| `updateOne` | Update one entity (partial) | `Update<T>, state` |
| `updateMany` | Update multiple entities (partial) | `Update<T>[], state` |
| `upsertOne` | Add if new, update if exists | `entity, state` |
| `upsertMany` | Add if new, update if exists (multiple) | `entities[], state` |
| `mapOne` | Map a single entity with a function | `EntityMapOne<T>, state` |
| `map` | Map all entities with a function | `EntityMap<T>, state` |

### Add Operations

```typescript
import { createFeature, createReducer, on } from '@ngrx/store';

export const productsFeature = createFeature({
  name: 'products',
  reducer: createReducer(
    initialState,

    // addOne: Add a single entity to the collection
    on(ProductsApiActions.productCreatedSuccessfully, (state, { product }) =>
      productsAdapter.addOne(product, state)
    ),

    // addMany: Add multiple entities, skipping duplicates
    on(ProductsApiActions.productsLoadedSuccessfully, (state, { products }) =>
      productsAdapter.addMany(products, { ...state, loading: false })
    ),

    // setAll: Replace entire collection
    on(ProductsApiActions.allProductsRefreshed, (state, { products }) =>
      productsAdapter.setAll(products, { ...state, loading: false })
    ),
  ),
});
```

### Update Operations

```typescript
// updateOne: Partial update by ID
on(ProductsApiActions.productUpdatedSuccessfully, (state, { product }) =>
  productsAdapter.updateOne(
    { id: product.id, changes: product },
    state
  )
),

// updateMany: Partial update for multiple entities
on(ProductsActions.markManyOutOfStock, (state, { ids }) =>
  productsAdapter.updateMany(
    ids.map(id => ({ id, changes: { stock: 0 } })),
    state
  )
),

// setOne: Replace (or insert) a single entity entirely
on(ProductsApiActions.productFetched, (state, { product }) =>
  productsAdapter.setOne(product, state)
),

// setMany: Replace (or insert) multiple entities entirely
on(ProductsApiActions.batchProductsFetched, (state, { products }) =>
  productsAdapter.setMany(products, state)
),
```

### Upsert Operations

Upsert adds the entity if it does not exist, or merges with the existing entity:

```typescript
// upsertOne: Insert or merge a single entity
on(WebSocketActions.productUpdated, (state, { product }) =>
  productsAdapter.upsertOne(product, state)
),

// upsertMany: Insert or merge multiple entities
on(ProductsApiActions.productsSearched, (state, { products }) =>
  productsAdapter.upsertMany(products, { ...state, loading: false })
),
```

### Remove Operations

```typescript
// removeOne: Remove by ID
on(ProductsApiActions.productDeletedSuccessfully, (state, { id }) =>
  productsAdapter.removeOne(id, state)
),

// removeMany: Remove by array of IDs
on(ProductsActions.bulkDelete, (state, { ids }) =>
  productsAdapter.removeMany(ids, state)
),

// removeMany: Remove by predicate
on(ProductsActions.removeOutOfStock, (state) =>
  productsAdapter.removeMany(
    (product) => product.stock === 0,
    state
  )
),

// removeAll: Clear the entire collection
on(ProductsActions.clearAll, (state) =>
  productsAdapter.removeAll(state)
),
```

---

## Sorted vs Unsorted Collections

### Unsorted (Default Insertion Order)

When no `sortComparer` is provided, entities maintain insertion order:

```typescript
const adapter = createEntityAdapter<Product>();
// ids: ['3', '1', '2'] - order depends on when entities were added
```

### Sorted

When a `sortComparer` is provided, the `ids` array is automatically re-sorted after every operation:

```typescript
const adapter = createEntityAdapter<Product>({
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});
// ids: ['1', '2', '3'] - always sorted alphabetically by name
```

### Performance Considerations

| Aspect | Unsorted | Sorted |
|--------|----------|--------|
| `addOne` | O(1) | O(n) binary search + insert |
| `addMany` | O(k) | O(k * n) |
| `setAll` | O(n) | O(n log n) |
| `removeOne` | O(n) splice | O(n) splice |
| Retrieval order | Insertion order | Sorted by comparer |
| Use when | Order not important or handled in selectors | Always need sorted display |

For most applications, prefer **unsorted** adapters and sort in selectors. This avoids re-sorting the entire collection on every write.

---

## Built-in Selectors

### Adapter Selector Methods

`EntityAdapter` provides a `getSelectors()` method that creates four pre-built selectors:

```typescript
// Using with createFeature's extraSelectors
export const productsFeature = createFeature({
  name: 'products',
  reducer: createReducer(initialState, /* ... */),
  extraSelectors: ({ selectProductsState }) => ({
    ...productsAdapter.getSelectors(selectProductsState),
  }),
});

// Auto-generated selectors:
// productsFeature.selectAll       -> Product[]    (all entities as array)
// productsFeature.selectEntities  -> Dictionary<Product> (entity lookup)
// productsFeature.selectIds       -> string[]     (ordered ID array)
// productsFeature.selectTotal     -> number       (count of entities)
```

### Standalone Selector Usage

```typescript
const { selectAll, selectEntities, selectIds, selectTotal } =
  productsAdapter.getSelectors();
```

### Composing with Entity Selectors

```typescript
import { createSelector } from '@ngrx/store';

// Select feature state
const selectProductsState = productsFeature.selectProductsState;

// Use adapter selectors as building blocks
const { selectAll, selectEntities, selectTotal } =
  productsAdapter.getSelectors(selectProductsState);

// Derived: filtered products
export const selectProductsByCategory = (category: string) =>
  createSelector(selectAll, (products) =>
    products.filter(p => p.category === category)
  );

// Derived: products with low stock
export const selectLowStockProducts = createSelector(
  selectAll,
  (products) => products.filter(p => p.stock < 10)
);

// Derived: selected product by ID from state
export const selectCurrentProduct = createSelector(
  selectEntities,
  productsFeature.selectSelectedProductId,
  (entities, selectedId) => (selectedId ? entities[selectedId] ?? null : null)
);

// Derived: product stats
export const selectProductStats = createSelector(
  selectAll,
  selectTotal,
  (products, total) => ({
    total,
    totalValue: products.reduce((sum, p) => sum + p.price * p.stock, 0),
    categories: [...new Set(products.map(p => p.category))],
    outOfStock: products.filter(p => p.stock === 0).length,
  })
);
```

---

## Integration with createFeature

### Full Feature Example

```typescript
// tasks.reducer.ts
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';
import { createFeature, createReducer, on } from '@ngrx/store';
import { Task, TaskStatus } from '../models/task.model';
import { TasksPageActions, TasksApiActions } from './tasks.actions';

export const tasksAdapter: EntityAdapter<Task> = createEntityAdapter<Task>({
  selectId: (task) => task.id,
  sortComparer: (a, b) => {
    const priorityOrder: Record<string, number> = { high: 0, medium: 1, low: 2 };
    return (priorityOrder[a.priority] ?? 3) - (priorityOrder[b.priority] ?? 3);
  },
});

export interface TasksState extends EntityState<Task> {
  loading: boolean;
  error: string | null;
  statusFilter: TaskStatus | 'all';
}

const initialState: TasksState = tasksAdapter.getInitialState({
  loading: false,
  error: null,
  statusFilter: 'all',
});

export const tasksFeature = createFeature({
  name: 'tasks',
  reducer: createReducer(
    initialState,

    on(TasksPageActions.opened, (state) => ({
      ...state,
      loading: true,
    })),

    on(TasksPageActions.statusFilterChanged, (state, { status }) => ({
      ...state,
      statusFilter: status,
    })),

    on(TasksApiActions.tasksLoadedSuccessfully, (state, { tasks }) =>
      tasksAdapter.setAll(tasks, { ...state, loading: false, error: null })
    ),

    on(TasksApiActions.tasksLoadedFailure, (state, { error }) => ({
      ...state,
      loading: false,
      error,
    })),

    on(TasksApiActions.taskCreatedSuccessfully, (state, { task }) =>
      tasksAdapter.addOne(task, state)
    ),

    on(TasksApiActions.taskUpdatedSuccessfully, (state, { task }) =>
      tasksAdapter.upsertOne(task, state)
    ),

    on(TasksApiActions.taskDeletedSuccessfully, (state, { id }) =>
      tasksAdapter.removeOne(id, state)
    ),

    on(TasksPageActions.bulkStatusChanged, (state, { ids, status }) =>
      tasksAdapter.updateMany(
        ids.map(id => ({ id, changes: { status } })),
        state
      )
    ),
  ),

  extraSelectors: ({ selectTasksState, selectStatusFilter }) => {
    const entitySelectors = tasksAdapter.getSelectors(selectTasksState);

    const selectFilteredTasks = createSelector(
      entitySelectors.selectAll,
      selectStatusFilter,
      (tasks, filter) =>
        filter === 'all' ? tasks : tasks.filter(t => t.status === filter)
    );

    const selectTaskCountsByStatus = createSelector(
      entitySelectors.selectAll,
      (tasks) => ({
        total: tasks.length,
        todo: tasks.filter(t => t.status === 'todo').length,
        inProgress: tasks.filter(t => t.status === 'in-progress').length,
        done: tasks.filter(t => t.status === 'done').length,
      })
    );

    return {
      ...entitySelectors,
      selectFilteredTasks,
      selectTaskCountsByStatus,
    };
  },
});
```

### Using in Components

```typescript
@Component({
  selector: 'app-task-board',
  standalone: true,
  template: `
    <div class="task-counts">
      <span>Total: {{ counts().total }}</span>
      <span>Todo: {{ counts().todo }}</span>
      <span>In Progress: {{ counts().inProgress }}</span>
      <span>Done: {{ counts().done }}</span>
    </div>

    @for (task of tasks(); track task.id) {
      <app-task-card [task]="task" />
    }
  `,
})
export class TaskBoardComponent {
  private readonly store = inject(Store);

  readonly tasks = this.store.selectSignal(tasksFeature.selectFilteredTasks);
  readonly counts = this.store.selectSignal(tasksFeature.selectTaskCountsByStatus);
  readonly loading = this.store.selectSignal(tasksFeature.selectLoading);
}
```

---

## Entity Map Operations

### mapOne -- Transform a Single Entity

```typescript
on(TasksActions.toggleTaskCompletion, (state, { id }) =>
  tasksAdapter.mapOne(
    {
      id,
      map: (task) => ({
        ...task,
        status: task.status === 'done' ? 'todo' : 'done',
        completedAt: task.status === 'done' ? null : new Date().toISOString(),
      }),
    },
    state
  )
),
```

### map -- Transform All Entities

```typescript
on(TasksActions.resetAllTasks, (state) =>
  tasksAdapter.map(
    (task) => ({ ...task, status: 'todo' as const, completedAt: null }),
    state
  )
),
```

---

## Normalized State Patterns

### Normalizing Nested API Responses

When your API returns nested data, normalize it before storing:

```typescript
// API Response
interface OrderResponse {
  id: string;
  customer: Customer;
  items: OrderItem[];
  total: number;
}

// Normalized state: separate adapters for each entity type
export const ordersAdapter = createEntityAdapter<Order>();
export const customersAdapter = createEntityAdapter<Customer>();
export const orderItemsAdapter = createEntityAdapter<OrderItem>({
  selectId: (item) => item.sku,
});

// In effect or service: normalize before dispatching
readonly loadOrder$ = createEffect(() =>
  this.actions$.pipe(
    ofType(OrderPageActions.opened),
    switchMap(({ orderId }) =>
      this.orderService.getById(orderId).pipe(
        map((response) => {
          const { customer, items, ...order } = response;
          return OrderApiActions.orderLoadedSuccessfully({
            order: { ...order, customerId: customer.id, itemSkus: items.map(i => i.sku) },
            customer,
            items,
          });
        }),
        catchError((error) =>
          of(OrderApiActions.orderLoadedFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### Cross-Feature Selectors with Entities

```typescript
export const selectOrderWithDetails = (orderId: string) =>
  createSelector(
    ordersFeature.selectEntities,
    customersFeature.selectEntities,
    orderItemsFeature.selectAll,
    (orders, customers, allItems) => {
      const order = orders[orderId];
      if (!order) return null;
      return {
        ...order,
        customer: customers[order.customerId] ?? null,
        items: allItems.filter(item => order.itemSkus.includes(item.sku)),
      };
    }
  );
```

---

## Multiple Entity Collections

### Managing Several Collections in One Feature

```typescript
export interface NotificationsState {
  alerts: EntityState<Alert>;
  messages: EntityState<Message>;
  unreadCount: number;
}

export const alertsAdapter = createEntityAdapter<Alert>({
  sortComparer: (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
});

export const messagesAdapter = createEntityAdapter<Message>({
  sortComparer: (a, b) => new Date(b.sentAt).getTime() - new Date(a.sentAt).getTime(),
});

const initialState: NotificationsState = {
  alerts: alertsAdapter.getInitialState(),
  messages: messagesAdapter.getInitialState(),
  unreadCount: 0,
};

export const notificationsFeature = createFeature({
  name: 'notifications',
  reducer: createReducer(
    initialState,

    on(NotificationsApiActions.alertsLoaded, (state, { alerts }) => ({
      ...state,
      alerts: alertsAdapter.setAll(alerts, state.alerts),
    })),

    on(NotificationsApiActions.messageReceived, (state, { message }) => ({
      ...state,
      messages: messagesAdapter.addOne(message, state.messages),
      unreadCount: state.unreadCount + 1,
    })),

    on(NotificationsActions.alertDismissed, (state, { id }) => ({
      ...state,
      alerts: alertsAdapter.removeOne(id, state.alerts),
    })),

    on(NotificationsActions.messageRead, (state, { id }) => ({
      ...state,
      messages: messagesAdapter.updateOne(
        { id, changes: { read: true } },
        state.messages
      ),
      unreadCount: Math.max(0, state.unreadCount - 1),
    })),
  ),
});
```

---

## Best Practices

### Adapter Usage Guidelines

| Guideline | Description |
|-----------|-------------|
| Use adapters for all collections | Avoid manual array/object management |
| Prefer unsorted adapters | Sort in selectors for flexibility |
| Use `upsertOne`/`upsertMany` for live data | WebSocket updates, polling |
| Use `setAll` for full refreshes | Page load, cache invalidation |
| Use `updateOne` for partial updates | Form saves, toggle operations |
| Extend `EntityState` for custom properties | Loading, error, filters |

### State Normalization

- Flatten nested API responses into separate entity collections
- Use foreign keys (IDs) to reference related entities
- Compose data back together in selectors, not in reducers
- One adapter per entity type

### Selector Composition with Entities

```typescript
// GOOD: Small, composable selectors
const entitySelectors = productsAdapter.getSelectors(selectProductsState);

export const selectActiveProducts = createSelector(
  entitySelectors.selectAll,
  (products) => products.filter(p => p.active)
);

export const selectActiveProductCount = createSelector(
  selectActiveProducts,
  (products) => products.length
);

// BAD: Accessing entity internals directly
export const selectProductById = createSelector(
  selectProductsState,
  (state, id: string) => state.entities[id] // Breaks memoization with props
);
```

---

## Common Pitfalls

### 1. Forgetting getInitialState Custom Properties

```typescript
// WRONG: Missing custom state in getInitialState
const initialState: ProductsState = productsAdapter.getInitialState();
// TypeScript error: missing loading, error, selectedProductId

// CORRECT: Provide all custom properties
const initialState: ProductsState = productsAdapter.getInitialState({
  loading: false,
  error: null,
  selectedProductId: null,
});
```

### 2. Mutating Entity State Properties

```typescript
// WRONG: Mutating the state object directly
on(SomeAction, (state, { id }) => {
  state.selectedProductId = id;  // Mutation!
  return state;
})

// CORRECT: Spread to create a new object
on(SomeAction, (state, { id }) => ({
  ...state,
  selectedProductId: id,
}))
```

### 3. Using setAll When addMany Is Intended

```typescript
// WRONG: setAll replaces everything -- previous data is lost
on(ProductsApiActions.nextPageLoaded, (state, { products }) =>
  productsAdapter.setAll(products, state)  // Wipes out page 1 data!
)

// CORRECT: addMany appends without removing existing
on(ProductsApiActions.nextPageLoaded, (state, { products }) =>
  productsAdapter.addMany(products, { ...state, loading: false })
)

// Or upsertMany if duplicates are possible
on(ProductsApiActions.nextPageLoaded, (state, { products }) =>
  productsAdapter.upsertMany(products, { ...state, loading: false })
)
```

### 4. Not Using selectId for Non-Standard IDs

```typescript
// WRONG: Adapter looks for 'id' property which does not exist
interface Employee {
  empCode: string;
  name: string;
}
const adapter = createEntityAdapter<Employee>(); // Runtime error!

// CORRECT: Specify the selectId function
const adapter = createEntityAdapter<Employee>({
  selectId: (employee) => employee.empCode,
});
```

### 5. Sorted Adapter Performance with Large Collections

```typescript
// PROBLEMATIC: Sorting 10,000+ entities on every add/update
const adapter = createEntityAdapter<LogEntry>({
  sortComparer: (a, b) => b.timestamp.localeCompare(a.timestamp),
});
// Every addOne triggers a full re-sort of the ids array

// BETTER: Use unsorted adapter + sort in selector (only runs when selected)
const adapter = createEntityAdapter<LogEntry>();

export const selectRecentLogs = createSelector(
  entitySelectors.selectAll,
  (logs) => [...logs].sort((a, b) => b.timestamp.localeCompare(a.timestamp)).slice(0, 100)
);
```
