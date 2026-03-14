# React Hooks

> Official Documentation: https://react.dev/reference/react/hooks

## useState - State Management

### Basic Usage
```tsx
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<string[]>([]);
```

### Functional Updates
Use when new state depends on previous state:
```tsx
// Correct - uses previous state
setCount(prev => prev + 1);

// Batched updates work correctly
const handleMultipleUpdates = () => {
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  setCount(prev => prev + 1); // Results in +3
};

// Incorrect for dependent updates
setCount(count + 1); // May use stale value in async contexts
```

### Lazy Initialization
For expensive initial computations, pass a function:
```tsx
// Expensive computation runs only once
const [data, setData] = useState(() => {
  return JSON.parse(localStorage.getItem('data') || '{}');
});

// Without lazy init - runs every render (wasteful)
const [data, setData] = useState(JSON.parse(localStorage.getItem('data') || '{}'));
```

### Object State Patterns
```tsx
const [form, setForm] = useState({ name: '', email: '', age: 0 });

// Update single field (immutable pattern)
const updateField = (field: keyof typeof form, value: string | number) => {
  setForm(prev => ({ ...prev, [field]: value }));
};

// Reset state
const resetForm = () => setForm({ name: '', email: '', age: 0 });
```

---

## useEffect - Side Effects

### Dependency Patterns
```tsx
// Runs after every render
useEffect(() => {
  console.log('Rendered');
});

// Runs once on mount
useEffect(() => {
  initializeApp();
}, []);

// Runs when dependencies change
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

### Cleanup Functions
Essential for subscriptions, timers, and event listeners:
```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(setData);

  return () => controller.abort(); // Cleanup on unmount or re-run
}, []);

// Event listeners
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);

// Timers
useEffect(() => {
  const id = setInterval(() => tick(), 1000);
  return () => clearInterval(id);
}, []);
```

### Async Effects Pattern
useEffect callback cannot be async directly:
```tsx
// Correct pattern
useEffect(() => {
  const fetchData = async () => {
    try {
      setLoading(true);
      const response = await fetch(`/api/users/${id}`);
      const data = await response.json();
      setUser(data);
    } catch (error) {
      setError(error);
    } finally {
      setLoading(false);
    }
  };

  fetchData();
}, [id]);

// With abort controller for race conditions
useEffect(() => {
  let cancelled = false;

  const fetchData = async () => {
    const data = await fetch(`/api/users/${id}`).then(r => r.json());
    if (!cancelled) setUser(data);
  };

  fetchData();
  return () => { cancelled = true; };
}, [id]);
```

---

## useContext - Context Consumption

### Creating Context with TypeScript
```tsx
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

// Custom hook for safe consumption
function useTheme(): ThemeContextType {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

### Provider Pattern
```tsx
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = useCallback(() => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }, []);

  const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage
function App() {
  return (
    <ThemeProvider>
      <Header />
      <Main />
    </ThemeProvider>
  );
}

function Header() {
  const { theme, toggleTheme } = useTheme();
  return <button onClick={toggleTheme}>Current: {theme}</button>;
}
```

---

## useReducer - Complex State Management

### Basic Pattern with TypeScript
```tsx
interface State {
  count: number;
  step: number;
}

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; payload: number }
  | { type: 'reset' };

const initialState: State = { count: 0, step: 1 };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.payload };
    case 'reset':
      return initialState;
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

### Lazy Initialization
```tsx
function init(initialCount: number): State {
  return { count: initialCount, step: 1 };
}

const [state, dispatch] = useReducer(reducer, initialCount, init);
```

### useReducer vs useState
- Use `useState` for simple, independent values
- Use `useReducer` for complex state logic, multiple sub-values, or when next state depends on previous

---

## useCallback - Function Memoization

### Preventing Unnecessary Re-renders
```tsx
function ParentComponent({ id }: { id: string }) {
  // Without useCallback: new function every render
  // const handleClick = () => fetchData(id);

  // With useCallback: same reference unless id changes
  const handleClick = useCallback(() => {
    fetchData(id);
  }, [id]);

  return <MemoizedChild onClick={handleClick} />;
}

const MemoizedChild = memo(function Child({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Fetch</button>;
});
```

### Event Handlers with Parameters
```tsx
const handleItemClick = useCallback((itemId: string) => {
  setSelectedId(itemId);
  onSelect(itemId);
}, [onSelect]);

// Usage in list
{items.map(item => (
  <Item key={item.id} onClick={() => handleItemClick(item.id)} />
))}
```

### When to Use
- Passing callbacks to memoized child components
- Callbacks used as effect dependencies
- Callbacks passed to custom hooks

---

## useMemo - Value Memoization

### Expensive Calculations
```tsx
function DataTable({ items, filter }: Props) {
  const filteredItems = useMemo(() => {
    return items.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    );
  }, [items, filter]);

  const sortedItems = useMemo(() => {
    return [...filteredItems].sort((a, b) => a.name.localeCompare(b.name));
  }, [filteredItems]);

  return <Table data={sortedItems} />;
}
```

### Referential Equality
```tsx
function Component({ data }: { data: Item[] }) {
  // Object recreated every render without useMemo
  const config = useMemo(() => ({
    columns: ['id', 'name', 'status'],
    sortable: true,
  }), []);

  return <DataGrid config={config} data={data} />;
}
```

### useMemo vs useCallback
```tsx
// These are equivalent
const memoizedFn = useCallback(() => doSomething(a, b), [a, b]);
const memoizedFn = useMemo(() => () => doSomething(a, b), [a, b]);
```

---

## useRef - Refs and Mutable Values

### DOM References
```tsx
function TextInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => {
    inputRef.current?.focus();
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

### Mutable Values (No Re-render)
```tsx
function Timer() {
  const intervalRef = useRef<NodeJS.Timeout | null>(null);
  const renderCountRef = useRef(0);

  useEffect(() => {
    renderCountRef.current += 1;
    console.log(`Render count: ${renderCountRef.current}`);
  });

  const startTimer = () => {
    intervalRef.current = setInterval(() => console.log('tick'), 1000);
  };

  const stopTimer = () => {
    if (intervalRef.current) clearInterval(intervalRef.current);
  };

  return (
    <>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
    </>
  );
}
```

### Previous Value Pattern
```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage
const prevCount = usePrevious(count);
```

---

## useLayoutEffect - Synchronous DOM Effects

### When to Use
Runs synchronously after DOM mutations but before browser paint:
```tsx
function Tooltip({ targetRef, content }: Props) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!targetRef.current || !tooltipRef.current) return;

    const targetRect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();

    setPosition({
      top: targetRect.bottom + 8,
      left: targetRect.left + (targetRect.width - tooltipRect.width) / 2,
    });
  }, [targetRef]);

  return (
    <div ref={tooltipRef} style={{ position: 'fixed', ...position }}>
      {content}
    </div>
  );
}
```

### useEffect vs useLayoutEffect
- `useEffect`: Runs asynchronously after paint (default choice)
- `useLayoutEffect`: Runs synchronously before paint (for DOM measurements)

---

## useId - Unique ID Generation

### Accessible Form Elements
```tsx
function FormField({ label }: { label: string }) {
  const id = useId();

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type="text" />
    </div>
  );
}

// Multiple IDs
function PasswordField() {
  const id = useId();

  return (
    <>
      <label htmlFor={`${id}-password`}>Password</label>
      <input id={`${id}-password`} type="password" aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>Must be at least 8 characters</p>
    </>
  );
}
```

---

## useTransition & useDeferredValue - Concurrent Features

### useTransition - Non-Blocking Updates
```tsx
function SearchResults({ query }: { query: string }) {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState<Item[]>([]);

  const handleSearch = (input: string) => {
    // High priority: update input immediately
    setQuery(input);

    // Low priority: defer expensive filtering
    startTransition(() => {
      const filtered = allItems.filter(item =>
        item.name.includes(input)
      );
      setResults(filtered);
    });
  };

  return (
    <div>
      <input onChange={e => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultsList items={results} />
    </div>
  );
}
```

### useDeferredValue - Deferred Rendering
```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  const isStale = query !== deferredQuery;

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Suspense fallback={<Loading />}>
        <div style={{ opacity: isStale ? 0.5 : 1 }}>
          <SearchResults query={deferredQuery} />
        </div>
      </Suspense>
    </div>
  );
}
```

---

## Custom Hooks Patterns

### Data Fetching Hook
```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const json = await response.json();
      setData(json);
    } catch (e) {
      setError(e instanceof Error ? e : new Error('Unknown error'));
    } finally {
      setLoading(false);
    }
  }, [url]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}
```

### Local Storage Hook
```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}
```

### Window Event Hook
```tsx
function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };

    handleResize(); // Set initial size
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

---

## React 19 Features

### use() - Async Resource Reading
```tsx
// Reading promises in render
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise); // Suspends until resolved
  return <h1>{user.name}</h1>;
}

// Reading context conditionally
function Theme({ showTheme }: { showTheme: boolean }) {
  if (showTheme) {
    const theme = use(ThemeContext);
    return <div>Theme: {theme}</div>;
  }
  return null;
}
```

### useOptimistic - Optimistic Updates
```tsx
function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: Todo) => [...state, newTodo]
  );

  async function addTodo(formData: FormData) {
    const newTodo = { id: Date.now(), text: formData.get('text') as string };
    addOptimistic(newTodo); // Update UI immediately
    await saveTodo(newTodo); // Server request
  }

  return (
    <form action={addTodo}>
      <input name="text" />
      <button type="submit">Add</button>
      <ul>
        {optimisticTodos.map(todo => <li key={todo.id}>{todo.text}</li>)}
      </ul>
    </form>
  );
}
```

### useFormStatus - Form Submission State
```tsx
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function ContactForm() {
  return (
    <form action={submitForm}>
      <input name="email" type="email" />
      <SubmitButton /> {/* Must be inside <form> */}
    </form>
  );
}
```

### useActionState - Server Action State
```tsx
interface FormState {
  message: string;
  errors?: { email?: string };
}

async function submitAction(prevState: FormState, formData: FormData): Promise<FormState> {
  const email = formData.get('email') as string;

  if (!email.includes('@')) {
    return { message: '', errors: { email: 'Invalid email' } };
  }

  await saveEmail(email);
  return { message: 'Success!' };
}

function Newsletter() {
  const [state, formAction, isPending] = useActionState(submitAction, { message: '' });

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      {state.errors?.email && <span>{state.errors.email}</span>}
      <button disabled={isPending}>
        {isPending ? 'Subscribing...' : 'Subscribe'}
      </button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

---

## Rules of Hooks

### 1. Only Call Hooks at the Top Level
```tsx
// WRONG - conditional hook
function Component({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // Error!
  }
}

// CORRECT - hook always called
function Component({ isLoggedIn }) {
  const [user, setUser] = useState(null);

  if (!isLoggedIn) return <Login />;
  return <Profile user={user} />;
}
```

### 2. Only Call Hooks from React Functions
```tsx
// WRONG - regular function
function getUser() {
  const [user] = useState(null); // Error!
  return user;
}

// CORRECT - custom hook (starts with "use")
function useUser() {
  const [user, setUser] = useState(null);
  return user;
}
```

### 3. Custom Hooks Must Start with "use"
```tsx
// WRONG
function fetchData() { ... }

// CORRECT
function useFetchData() { ... }
```

---

## Best Practices and Common Pitfalls

### Stale Closures
```tsx
// WRONG - stale closure
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // Always uses initial count (0)
    }, 1000);
    return () => clearInterval(id);
  }, []); // Missing count dependency
}

// CORRECT - functional update
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1); // Uses current value
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

### Object Dependencies
```tsx
// WRONG - new object every render triggers effect
useEffect(() => {
  fetchData(options);
}, [{ page: 1, limit: 10 }]); // Always different reference

// CORRECT - primitive values or useMemo
const options = useMemo(() => ({ page: 1, limit: 10 }), []);
useEffect(() => {
  fetchData(options);
}, [options]);
```

### Infinite Loops
```tsx
// WRONG - effect updates its own dependency
useEffect(() => {
  setItems([...items, newItem]); // Triggers infinite loop
}, [items]);

// CORRECT - functional update
useEffect(() => {
  setItems(prev => [...prev, newItem]);
}, [newItem]);
```

### Missing Cleanup
```tsx
// WRONG - memory leak
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
}, []);

// CORRECT - cleanup on unmount
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### Over-Memoization
```tsx
// UNNECESSARY - primitive values are cheap to compare
const doubled = useMemo(() => count * 2, [count]);

// NECESSARY - expensive computation
const sorted = useMemo(() =>
  items.slice().sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

### State Updates After Unmount
```tsx
// WRONG - may update after unmount
useEffect(() => {
  fetchData().then(data => setData(data));
}, []);

// CORRECT - check if mounted
useEffect(() => {
  let mounted = true;
  fetchData().then(data => {
    if (mounted) setData(data);
  });
  return () => { mounted = false; };
}, []);
```

### Derived State Anti-Pattern
```tsx
// WRONG - redundant state
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter(i => i.active));
}, [items]);

// CORRECT - derive during render
const [items, setItems] = useState([]);
const filteredItems = items.filter(i => i.active);

// Or use useMemo for expensive operations
const filteredItems = useMemo(() =>
  items.filter(i => i.active),
  [items]
);
```
