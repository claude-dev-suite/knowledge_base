# Zustand State Management

> Lightweight, fast state management for React with minimal boilerplate and excellent TypeScript support.

## Table of Contents

1. [Setup and Basic Usage](#setup-and-basic-usage)
2. [Selectors and Performance](#selectors-and-performance)
3. [Async Actions](#async-actions)
4. [Middleware](#middleware)
5. [Store Patterns](#store-patterns)
6. [Testing](#testing)
7. [Best Practices](#best-practices)

---

## Setup and Basic Usage

### Basic Store

```typescript
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setCount: (value: number) => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  setCount: (value) => set({ count: value }),
}));

// Usage in component
function Counter() {
  const { count, increment, decrement, reset } = useCounterStore();

  return (
    <div>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Store with Computed Values

```typescript
interface CartState {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

const useCartStore = create<CartState>((set) => ({
  items: [],

  addItem: (item) => set((state) => {
    const existingItem = state.items.find((i) => i.id === item.id);
    if (existingItem) {
      return {
        items: state.items.map((i) =>
          i.id === item.id
            ? { ...i, quantity: i.quantity + item.quantity }
            : i
        ),
      };
    }
    return { items: [...state.items, item] };
  }),

  removeItem: (id) => set((state) => ({
    items: state.items.filter((item) => item.id !== id),
  })),

  clearCart: () => set({ items: [] }),
}));

// Computed values as selectors
const selectTotalItems = (state: CartState) =>
  state.items.reduce((sum, item) => sum + item.quantity, 0);

const selectTotalPrice = (state: CartState) =>
  state.items.reduce((sum, item) => sum + item.price * item.quantity, 0);

// Usage
function CartSummary() {
  const totalItems = useCartStore(selectTotalItems);
  const totalPrice = useCartStore(selectTotalPrice);

  return (
    <div>
      <p>Items: {totalItems}</p>
      <p>Total: ${totalPrice.toFixed(2)}</p>
    </div>
  );
}
```

---

## Selectors and Performance

### Optimized Selectors

```typescript
// Bad: Returns new object on every render
function BadComponent() {
  const { count, total } = useStore((state) => ({
    count: state.count,
    total: state.total,
  })); // New object every time = re-render
}

// Good: Use shallow comparison
import { shallow } from 'zustand/shallow';

function GoodComponent() {
  const { count, total } = useStore(
    (state) => ({ count: state.count, total: state.total }),
    shallow
  );
}

// Better: Use useShallow hook (Zustand 4.4+)
import { useShallow } from 'zustand/react/shallow';

function BetterComponent() {
  const { count, total } = useStore(
    useShallow((state) => ({ count: state.count, total: state.total }))
  );
}

// Best: Select primitives directly
function BestComponent() {
  const count = useStore((state) => state.count);
  const total = useStore((state) => state.total);
}

// Array selection with shallow
function ItemList() {
  const items = useStore(
    (state) => state.items.filter((item) => item.active),
    shallow
  );
}
```

### Memoized Selectors

```typescript
import { useMemo } from 'react';

function ExpensiveComponent() {
  const items = useStore((state) => state.items);

  // Memoize derived data
  const processedItems = useMemo(
    () => items.map((item) => ({
      ...item,
      displayName: `${item.name} (${item.quantity})`,
    })),
    [items]
  );

  return <ItemList items={processedItems} />;
}

// Or create selector factories
const createFilteredItemsSelector = (status: string) =>
  (state: StoreState) => state.items.filter((item) => item.status === status);

function FilteredList({ status }: { status: string }) {
  const selector = useMemo(
    () => createFilteredItemsSelector(status),
    [status]
  );
  const items = useStore(selector, shallow);

  return <ItemList items={items} />;
}
```

---

## Async Actions

### Basic Async

```typescript
interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
  fetchUser: (id: string) => Promise<void>;
  updateUser: (data: Partial<User>) => Promise<void>;
}

const useUserStore = create<UserState>((set, get) => ({
  user: null,
  loading: false,
  error: null,

  fetchUser: async (id: string) => {
    set({ loading: true, error: null });
    try {
      const response = await fetch(`/api/users/${id}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      const user = await response.json();
      set({ user, loading: false });
    } catch (error) {
      set({
        error: error instanceof Error ? error.message : 'Unknown error',
        loading: false,
      });
    }
  },

  updateUser: async (data: Partial<User>) => {
    const currentUser = get().user;
    if (!currentUser) return;

    set({ loading: true, error: null });
    try {
      const response = await fetch(`/api/users/${currentUser.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      if (!response.ok) throw new Error('Failed to update user');
      const updatedUser = await response.json();
      set({ user: updatedUser, loading: false });
    } catch (error) {
      set({
        error: error instanceof Error ? error.message : 'Unknown error',
        loading: false,
      });
    }
  },
}));
```

### Optimistic Updates

```typescript
interface TodoState {
  todos: Todo[];
  toggleTodo: (id: string) => Promise<void>;
}

const useTodoStore = create<TodoState>((set, get) => ({
  todos: [],

  toggleTodo: async (id: string) => {
    const previousTodos = get().todos;

    // Optimistic update
    set({
      todos: previousTodos.map((todo) =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      ),
    });

    try {
      const todo = previousTodos.find((t) => t.id === id);
      await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed: !todo?.completed }),
      });
    } catch (error) {
      // Rollback on error
      set({ todos: previousTodos });
      console.error('Failed to toggle todo:', error);
    }
  },
}));
```

---

## Middleware

### Persist Middleware

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface SettingsState {
  theme: 'light' | 'dark' | 'system';
  language: string;
  notifications: boolean;
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
  setLanguage: (language: string) => void;
  toggleNotifications: () => void;
}

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: 'system',
      language: 'en',
      notifications: true,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () => set((state) => ({
        notifications: !state.notifications,
      })),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => localStorage),
      // Only persist specific fields
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
        notifications: state.notifications,
      }),
      // Version for migrations
      version: 1,
      migrate: (persistedState, version) => {
        if (version === 0) {
          // Migration from v0 to v1
          return {
            ...persistedState,
            theme: (persistedState as any).darkMode ? 'dark' : 'light',
          };
        }
        return persistedState as SettingsState;
      },
    }
  )
);

// Custom storage (e.g., IndexedDB)
import { get, set, del } from 'idb-keyval';

const idbStorage = {
  getItem: async (name: string) => {
    const value = await get(name);
    return value ?? null;
  },
  setItem: async (name: string, value: string) => {
    await set(name, value);
  },
  removeItem: async (name: string) => {
    await del(name);
  },
};

const useStoreWithIDB = create<State>()(
  persist(
    (set) => ({ /* ... */ }),
    {
      name: 'app-storage',
      storage: createJSONStorage(() => idbStorage),
    }
  )
);
```

### DevTools Middleware

```typescript
import { devtools } from 'zustand/middleware';

const useStore = create<State>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set(
        (state) => ({ count: state.count + 1 }),
        false,  // replace
        'increment'  // action name
      ),
      decrement: () => set(
        (state) => ({ count: state.count - 1 }),
        false,
        'decrement'
      ),
    }),
    {
      name: 'MyStore',
      enabled: process.env.NODE_ENV === 'development',
    }
  )
);
```

### Immer Middleware

```typescript
import { immer } from 'zustand/middleware/immer';

interface TodoState {
  todos: Todo[];
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  removeTodo: (id: string) => void;
  updateTodo: (id: string, text: string) => void;
}

const useTodoStore = create<TodoState>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) => set((state) => {
      state.todos.push({
        id: crypto.randomUUID(),
        text,
        completed: false,
        createdAt: new Date().toISOString(),
      });
    }),

    toggleTodo: (id) => set((state) => {
      const todo = state.todos.find((t) => t.id === id);
      if (todo) {
        todo.completed = !todo.completed;
      }
    }),

    removeTodo: (id) => set((state) => {
      const index = state.todos.findIndex((t) => t.id === id);
      if (index !== -1) {
        state.todos.splice(index, 1);
      }
    }),

    updateTodo: (id, text) => set((state) => {
      const todo = state.todos.find((t) => t.id === id);
      if (todo) {
        todo.text = text;
      }
    }),
  }))
);
```

### Combining Middleware

```typescript
const useStore = create<State>()(
  devtools(
    persist(
      immer((set, get) => ({
        // Store definition with immer syntax
      })),
      {
        name: 'app-storage',
        partialize: (state) => ({ /* fields to persist */ }),
      }
    ),
    { name: 'AppStore' }
  )
);
```

### Custom Middleware

```typescript
import { StateCreator, StoreMutatorIdentifier } from 'zustand';

// Logger middleware
type Logger = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = []
>(
  f: StateCreator<T, Mps, Mcs>,
  name?: string
) => StateCreator<T, Mps, Mcs>;

const logger: Logger = (f, name) => (set, get, store) => {
  const loggedSet: typeof set = (...args) => {
    const previousState = get();
    set(...args);
    const nextState = get();
    console.log(
      `[${name ?? 'store'}]`,
      { previous: previousState, next: nextState }
    );
  };
  return f(loggedSet, get, store);
};

// Usage
const useStore = create<State>()(
  logger(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    'CounterStore'
  )
);
```

---

## Store Patterns

### Slice Pattern

```typescript
// types.ts
interface UserSlice {
  user: User | null;
  setUser: (user: User | null) => void;
  logout: () => void;
}

interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

interface UISlice {
  sidebarOpen: boolean;
  modalOpen: boolean;
  toggleSidebar: () => void;
  openModal: () => void;
  closeModal: () => void;
}

type StoreState = UserSlice & CartSlice & UISlice;

// slices/userSlice.ts
import { StateCreator } from 'zustand';

const createUserSlice: StateCreator<
  StoreState,
  [],
  [],
  UserSlice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
});

// slices/cartSlice.ts
const createCartSlice: StateCreator<
  StoreState,
  [],
  [],
  CartSlice
> = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter((i) => i.id !== id),
  })),
  clearCart: () => set({ items: [] }),
});

// slices/uiSlice.ts
const createUISlice: StateCreator<
  StoreState,
  [],
  [],
  UISlice
> = (set) => ({
  sidebarOpen: false,
  modalOpen: false,
  toggleSidebar: () => set((state) => ({
    sidebarOpen: !state.sidebarOpen,
  })),
  openModal: () => set({ modalOpen: true }),
  closeModal: () => set({ modalOpen: false }),
});

// store.ts
const useStore = create<StoreState>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
  ...createUISlice(...a),
}));

export default useStore;
```

### Factory Pattern

```typescript
// Create store factory for multi-instance stores
function createTodoStore(initialTodos: Todo[] = []) {
  return create<TodoState>()(
    immer((set) => ({
      todos: initialTodos,
      addTodo: (text) => set((state) => {
        state.todos.push({ id: crypto.randomUUID(), text, completed: false });
      }),
      toggleTodo: (id) => set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) todo.completed = !todo.completed;
      }),
    }))
  );
}

// Usage with context for scoped stores
const TodoStoreContext = createContext<ReturnType<typeof createTodoStore> | null>(null);

function TodoProvider({ children, initialTodos }: {
  children: React.ReactNode;
  initialTodos?: Todo[];
}) {
  const storeRef = useRef<ReturnType<typeof createTodoStore>>();
  if (!storeRef.current) {
    storeRef.current = createTodoStore(initialTodos);
  }

  return (
    <TodoStoreContext.Provider value={storeRef.current}>
      {children}
    </TodoStoreContext.Provider>
  );
}

function useTodoStore<T>(selector: (state: TodoState) => T): T {
  const store = useContext(TodoStoreContext);
  if (!store) throw new Error('Missing TodoProvider');
  return useStore(store, selector);
}
```

### Accessing Store Outside React

```typescript
const useStore = create<State>((set, get) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Get current state
const currentCount = useStore.getState().count;

// Set state directly
useStore.setState({ count: 10 });

// Subscribe to changes
const unsubscribe = useStore.subscribe(
  (state, prevState) => {
    console.log('State changed:', { prev: prevState, next: state });
  }
);

// Subscribe to specific slice
const unsubscribeCount = useStore.subscribe(
  (state) => state.count,
  (count, prevCount) => {
    console.log('Count changed:', { prev: prevCount, next: count });
  },
  { equalityFn: Object.is }
);

// In non-React code (e.g., API interceptors)
import { useAuthStore } from './stores/authStore';

api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

---

## Testing

### Testing Stores

```typescript
import { act, renderHook } from '@testing-library/react';
import { useCounterStore } from './counterStore';

describe('useCounterStore', () => {
  // Reset store before each test
  beforeEach(() => {
    useCounterStore.setState({ count: 0 });
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useCounterStore());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('should decrement count', () => {
    const { result } = renderHook(() => useCounterStore());

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(-1);
  });

  it('should reset count', () => {
    const { result } = renderHook(() => useCounterStore());

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(0);
  });
});
```

### Testing Async Actions

```typescript
import { act, renderHook, waitFor } from '@testing-library/react';
import { useUserStore } from './userStore';

// Mock fetch
global.fetch = jest.fn();

describe('useUserStore', () => {
  beforeEach(() => {
    useUserStore.setState({ user: null, loading: false, error: null });
    jest.clearAllMocks();
  });

  it('should fetch user successfully', async () => {
    const mockUser = { id: '1', name: 'John' };
    (fetch as jest.Mock).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser),
    });

    const { result } = renderHook(() => useUserStore());

    await act(async () => {
      await result.current.fetchUser('1');
    });

    expect(result.current.user).toEqual(mockUser);
    expect(result.current.loading).toBe(false);
    expect(result.current.error).toBe(null);
  });

  it('should handle fetch error', async () => {
    (fetch as jest.Mock).mockResolvedValueOnce({
      ok: false,
    });

    const { result } = renderHook(() => useUserStore());

    await act(async () => {
      await result.current.fetchUser('1');
    });

    expect(result.current.user).toBe(null);
    expect(result.current.loading).toBe(false);
    expect(result.current.error).toBe('Failed to fetch user');
  });
});
```

---

## Best Practices

### Do's

1. **Keep stores small and focused** - One store per domain/feature
2. **Use selectors** - Prevent unnecessary re-renders
3. **Use immer for complex updates** - Cleaner mutation syntax
4. **Colocate actions with state** - Keep related logic together
5. **Use TypeScript** - Better DX and type safety
6. **Persist only necessary data** - Use `partialize`

### Don'ts

1. **Don't select entire state** - Always use selectors
2. **Don't create stores inside components** - Use module-level stores
3. **Don't mutate state directly** - Use `set` or immer
4. **Don't store derived data** - Compute on read
5. **Don't put React components in state** - Only serializable data

### Performance Tips

```typescript
// Use atomic selectors
const count = useStore((state) => state.count);
const increment = useStore((state) => state.increment);

// Use subscribeWithSelector for fine-grained subscriptions
import { subscribeWithSelector } from 'zustand/middleware';

const useStore = create<State>()(
  subscribeWithSelector((set) => ({
    count: 0,
    name: 'John',
  }))
);

// Subscribe to specific field
useStore.subscribe(
  (state) => state.count,
  (count) => console.log('Count:', count)
);

// Use temporal for undo/redo
import { temporal } from 'zundo';

const useStore = create<State>()(
  temporal((set) => ({
    // state and actions
  }))
);

const { undo, redo, pastStates, futureStates } = useStore.temporal.getState();
```
