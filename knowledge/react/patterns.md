# React Patterns & Performance

Advanced patterns, performance optimization, and architectural best practices for React applications.

**Official Documentation:** https://react.dev/learn

---

## Table of Contents

1. [Component Patterns](#component-patterns)
2. [State Management Patterns](#state-management-patterns)
3. [Performance Optimization](#performance-optimization)
4. [Code Splitting & Lazy Loading](#code-splitting--lazy-loading)
5. [Error Handling](#error-handling)
6. [Composition Patterns](#composition-patterns)
7. [Testing Patterns](#testing-patterns)
8. [TypeScript Patterns](#typescript-patterns)

---

## Component Patterns

### Compound Components

Pattern for components that work together sharing implicit state.

```tsx
import { createContext, useContext, useState, ReactNode } from 'react';

// Context for shared state
interface AccordionContextType {
  openItems: Set<string>;
  toggle: (id: string) => void;
}

const AccordionContext = createContext<AccordionContextType | null>(null);

function useAccordionContext() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('Accordion components must be used within Accordion');
  }
  return context;
}

// Parent component
interface AccordionProps {
  children: ReactNode;
  allowMultiple?: boolean;
}

function Accordion({ children, allowMultiple = false }: AccordionProps) {
  const [openItems, setOpenItems] = useState<Set<string>>(new Set());

  const toggle = (id: string) => {
    setOpenItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear();
        next.add(id);
      }
      return next;
    });
  };

  return (
    <AccordionContext.Provider value={{ openItems, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Child components
interface AccordionItemProps {
  id: string;
  children: ReactNode;
}

function AccordionItem({ id, children }: AccordionItemProps) {
  return (
    <div className="accordion-item" data-id={id}>
      {children}
    </div>
  );
}

interface AccordionTriggerProps {
  id: string;
  children: ReactNode;
}

function AccordionTrigger({ id, children }: AccordionTriggerProps) {
  const { openItems, toggle } = useAccordionContext();
  const isOpen = openItems.has(id);

  return (
    <button
      onClick={() => toggle(id)}
      aria-expanded={isOpen}
      className="accordion-trigger"
    >
      {children}
      <span>{isOpen ? '−' : '+'}</span>
    </button>
  );
}

interface AccordionContentProps {
  id: string;
  children: ReactNode;
}

function AccordionContent({ id, children }: AccordionContentProps) {
  const { openItems } = useAccordionContext();
  const isOpen = openItems.has(id);

  if (!isOpen) return null;

  return <div className="accordion-content">{children}</div>;
}

// Attach sub-components
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Usage
function App() {
  return (
    <Accordion allowMultiple>
      <Accordion.Item id="1">
        <Accordion.Trigger id="1">Section 1</Accordion.Trigger>
        <Accordion.Content id="1">Content 1</Accordion.Content>
      </Accordion.Item>
      <Accordion.Item id="2">
        <Accordion.Trigger id="2">Section 2</Accordion.Trigger>
        <Accordion.Content id="2">Content 2</Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

### Render Props

Pattern for sharing code between components using a prop whose value is a function.

```tsx
interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  children: (position: MousePosition) => ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });

  useEffect(() => {
    const handleMouseMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);

  return <>{children(position)}</>;
}

// Usage
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <div>
          Mouse position: {x}, {y}
        </div>
      )}
    </MouseTracker>
  );
}

// Alternative: render prop instead of children
interface DataFetcherProps<T> {
  url: string;
  render: (data: T | null, loading: boolean, error: Error | null) => ReactNode;
}

function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return <>{render(data, loading, error)}</>;
}
```

### Higher-Order Components (HOC)

Pattern for reusing component logic by wrapping components.

```tsx
// Basic HOC
function withLoading<P extends object>(
  WrappedComponent: ComponentType<P>
) {
  return function WithLoadingComponent(
    props: P & { isLoading: boolean }
  ) {
    const { isLoading, ...rest } = props;

    if (isLoading) {
      return <div className="spinner">Loading...</div>;
    }

    return <WrappedComponent {...(rest as P)} />;
  };
}

// HOC with configuration
interface WithAuthOptions {
  redirectTo?: string;
  LoadingComponent?: ComponentType;
}

function withAuth<P extends object>(
  WrappedComponent: ComponentType<P>,
  options: WithAuthOptions = {}
) {
  const {
    redirectTo = '/login',
    LoadingComponent = () => <div>Loading...</div>
  } = options;

  return function WithAuthComponent(props: P) {
    const { user, loading } = useAuth();
    const router = useRouter();

    useEffect(() => {
      if (!loading && !user) {
        router.push(redirectTo);
      }
    }, [user, loading, router]);

    if (loading) return <LoadingComponent />;
    if (!user) return null;

    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard, {
  redirectTo: '/login',
});

// Composing HOCs
const EnhancedComponent = withAuth(withLoading(withTheme(BaseComponent)));

// Better: use custom hook instead of HOC when possible
function useAuthGuard(redirectTo = '/login') {
  const { user, loading } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (!loading && !user) {
      router.push(redirectTo);
    }
  }, [user, loading, router, redirectTo]);

  return { user, loading, isAuthenticated: !!user };
}
```

### Controlled vs Uncontrolled Components

```tsx
// Controlled - parent manages state
interface ControlledInputProps {
  value: string;
  onChange: (value: string) => void;
}

function ControlledInput({ value, onChange }: ControlledInputProps) {
  return (
    <input
      value={value}
      onChange={e => onChange(e.target.value)}
    />
  );
}

// Uncontrolled - component manages its own state
interface UncontrolledInputProps {
  defaultValue?: string;
  onBlur?: (value: string) => void;
}

function UncontrolledInput({ defaultValue = '', onBlur }: UncontrolledInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <input
      ref={inputRef}
      defaultValue={defaultValue}
      onBlur={() => onBlur?.(inputRef.current?.value ?? '')}
    />
  );
}

// Hybrid - supports both controlled and uncontrolled
interface InputProps {
  value?: string;
  defaultValue?: string;
  onChange?: (value: string) => void;
}

function Input({ value, defaultValue, onChange }: InputProps) {
  const [internalValue, setInternalValue] = useState(defaultValue ?? '');
  const isControlled = value !== undefined;
  const inputValue = isControlled ? value : internalValue;

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    if (!isControlled) {
      setInternalValue(newValue);
    }
    onChange?.(newValue);
  };

  return <input value={inputValue} onChange={handleChange} />;
}
```

---

## State Management Patterns

### State Colocation

Keep state as close as possible to where it's used.

```tsx
// Bad - state lifted too high
function App() {
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedItem, setSelectedItem] = useState(null);

  return (
    <div>
      <Header searchQuery={searchQuery} setSearchQuery={setSearchQuery} />
      <Sidebar />
      <Main selectedItem={selectedItem} setSelectedItem={setSelectedItem} />
      <Footer />
    </div>
  );
}

// Good - state colocated
function App() {
  return (
    <div>
      <Header />
      <Sidebar />
      <Main />
      <Footer />
    </div>
  );
}

function Header() {
  const [searchQuery, setSearchQuery] = useState(''); // Local state
  return <SearchBar value={searchQuery} onChange={setSearchQuery} />;
}

function Main() {
  const [selectedItem, setSelectedItem] = useState(null); // Local state
  return <ItemList selectedItem={selectedItem} onSelect={setSelectedItem} />;
}
```

### State Lifting

Lift state up when siblings need to share it.

```tsx
// State needs to be shared between SearchInput and SearchResults
function SearchPage() {
  const [query, setQuery] = useState('');
  const [filters, setFilters] = useState<Filters>({});

  return (
    <div>
      <SearchInput value={query} onChange={setQuery} />
      <FilterPanel filters={filters} onChange={setFilters} />
      <SearchResults query={query} filters={filters} />
    </div>
  );
}
```

### Reducer Pattern for Complex State

```tsx
interface FormState {
  values: Record<string, string>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
  isValid: boolean;
}

type FormAction =
  | { type: 'SET_VALUE'; field: string; value: string }
  | { type: 'SET_ERROR'; field: string; error: string }
  | { type: 'SET_TOUCHED'; field: string }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR'; errors: Record<string, string> }
  | { type: 'RESET' };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_VALUE':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: '' },
      };

    case 'SET_ERROR':
      return {
        ...state,
        errors: { ...state.errors, [action.field]: action.error },
        isValid: false,
      };

    case 'SET_TOUCHED':
      return {
        ...state,
        touched: { ...state.touched, [action.field]: true },
      };

    case 'SUBMIT_START':
      return { ...state, isSubmitting: true };

    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false };

    case 'SUBMIT_ERROR':
      return {
        ...state,
        isSubmitting: false,
        errors: action.errors,
        isValid: false,
      };

    case 'RESET':
      return initialState;

    default:
      return state;
  }
}

// Custom hook using reducer
function useForm(initialValues: Record<string, string>) {
  const initialState: FormState = {
    values: initialValues,
    errors: {},
    touched: {},
    isSubmitting: false,
    isValid: true,
  };

  const [state, dispatch] = useReducer(formReducer, initialState);

  const setValue = (field: string, value: string) => {
    dispatch({ type: 'SET_VALUE', field, value });
  };

  const setTouched = (field: string) => {
    dispatch({ type: 'SET_TOUCHED', field });
  };

  const handleSubmit = async (onSubmit: (values: Record<string, string>) => Promise<void>) => {
    dispatch({ type: 'SUBMIT_START' });
    try {
      await onSubmit(state.values);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (error) {
      dispatch({ type: 'SUBMIT_ERROR', errors: { form: 'Submission failed' } });
    }
  };

  return { ...state, setValue, setTouched, handleSubmit };
}
```

### Context Splitting

Split context to prevent unnecessary re-renders.

```tsx
// Bad - single context causes all consumers to re-render
const AppContext = createContext<{
  user: User | null;
  theme: Theme;
  notifications: Notification[];
  setUser: (user: User) => void;
  setTheme: (theme: Theme) => void;
  addNotification: (n: Notification) => void;
} | null>(null);

// Good - split into separate contexts
const UserContext = createContext<{
  user: User | null;
  setUser: (user: User) => void;
} | null>(null);

const ThemeContext = createContext<{
  theme: Theme;
  setTheme: (theme: Theme) => void;
} | null>(null);

const NotificationContext = createContext<{
  notifications: Notification[];
  addNotification: (n: Notification) => void;
} | null>(null);

// Further optimization: separate state and dispatch contexts
const UserStateContext = createContext<User | null>(null);
const UserDispatchContext = createContext<((user: User) => void) | null>(null);

function UserProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  return (
    <UserStateContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
}

// Consumers only re-render when their specific context changes
function UserProfile() {
  const user = useContext(UserStateContext); // Only re-renders when user changes
  return <div>{user?.name}</div>;
}

function LoginButton() {
  const setUser = useContext(UserDispatchContext); // Never re-renders from user changes
  return <button onClick={() => setUser({ id: '1', name: 'John' })}>Login</button>;
}
```

---

## Performance Optimization

### React.memo for Expensive Components

```tsx
interface ExpensiveListProps {
  items: Item[];
  onItemClick: (id: string) => void;
}

// Memoize to prevent re-renders when props haven't changed
const ExpensiveList = memo(function ExpensiveList({
  items,
  onItemClick,
}: ExpensiveListProps) {
  console.log('ExpensiveList rendered');

  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
});

// With custom comparison
const DeepCompareList = memo(
  function DeepCompareList({ items }: { items: Item[] }) {
    return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
  },
  (prevProps, nextProps) => {
    // Return true if props are equal (should NOT re-render)
    return JSON.stringify(prevProps.items) === JSON.stringify(nextProps.items);
  }
);

// Parent component must use useCallback for handlers
function Parent() {
  const [items, setItems] = useState<Item[]>([]);
  const [count, setCount] = useState(0);

  // Without useCallback, new function reference every render
  // breaks memoization of ExpensiveList
  const handleItemClick = useCallback((id: string) => {
    console.log('Clicked:', id);
  }, []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      {/* ExpensiveList won't re-render when count changes */}
      <ExpensiveList items={items} onItemClick={handleItemClick} />
    </div>
  );
}
```

### Virtualization for Long Lists

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

interface VirtualListProps {
  items: Item[];
  itemHeight: number;
}

function VirtualList({ items, itemHeight }: VirtualListProps) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => itemHeight,
    overscan: 5, // Render 5 extra items above/below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '400px', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Debouncing and Throttling

```tsx
// Custom debounce hook
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Custom debounced callback hook
function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number
): T {
  const timeoutRef = useRef<NodeJS.Timeout>();

  return useCallback(
    ((...args) => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
      timeoutRef.current = setTimeout(() => {
        callback(...args);
      }, delay);
    }) as T,
    [callback, delay]
  );
}

// Usage in search
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      searchAPI(debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}

// Throttle hook for scroll events
function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastUpdated = useRef(Date.now());

  useEffect(() => {
    const now = Date.now();
    if (now - lastUpdated.current >= interval) {
      lastUpdated.current = now;
      setThrottledValue(value);
    } else {
      const timer = setTimeout(() => {
        lastUpdated.current = Date.now();
        setThrottledValue(value);
      }, interval - (now - lastUpdated.current));

      return () => clearTimeout(timer);
    }
  }, [value, interval]);

  return throttledValue;
}
```

### Optimizing Re-renders with Keys

```tsx
// Bad - index as key causes unnecessary re-renders and bugs
{items.map((item, index) => (
  <Item key={index} data={item} /> // Don't do this!
))}

// Good - stable unique identifier
{items.map(item => (
  <Item key={item.id} data={item} />
))}

// Force re-mount with key change (useful for resetting state)
function EditForm({ recordId }: { recordId: string }) {
  // Form state resets when recordId changes
  return <Form key={recordId} recordId={recordId} />;
}

// Avoid expensive reconciliation with unique key
function AnimatedList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        // Unique key ensures smooth animations during reorder
        <motion.li key={item.id} layout>
          {item.name}
        </motion.li>
      ))}
    </ul>
  );
}
```

### Avoiding Prop Drilling with Context

```tsx
// Before - prop drilling
function App() {
  const [user, setUser] = useState<User | null>(null);

  return (
    <Layout user={user}>
      <Header user={user} />
      <Main user={user} setUser={setUser}>
        <Sidebar user={user} />
        <Content user={user}>
          <Profile user={user} setUser={setUser} />
        </Content>
      </Main>
    </Layout>
  );
}

// After - context
const UserContext = createContext<{
  user: User | null;
  setUser: (user: User | null) => void;
} | null>(null);

function UserProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const value = useMemo(() => ({ user, setUser }), [user]);

  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

function useUser() {
  const context = useContext(UserContext);
  if (!context) throw new Error('useUser must be within UserProvider');
  return context;
}

function App() {
  return (
    <UserProvider>
      <Layout>
        <Header />
        <Main>
          <Sidebar />
          <Content>
            <Profile /> {/* Gets user from context */}
          </Content>
        </Main>
      </Layout>
    </UserProvider>
  );
}
```

---

## Code Splitting & Lazy Loading

### Route-Based Code Splitting

```tsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

// Named exports require different syntax
const Profile = lazy(() =>
  import('./pages/Profile').then(module => ({ default: module.Profile }))
);

// Loading fallback component
function PageLoader() {
  return (
    <div className="page-loader">
      <Spinner />
      <p>Loading...</p>
    </div>
  );
}

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### Component-Level Code Splitting

```tsx
// Heavy component loaded on demand
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      <button onClick={() => setShowEditor(true)}>Show Editor</button>

      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart data={chartData} />
        </Suspense>
      )}

      {showEditor && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichTextEditor />
        </Suspense>
      )}
    </div>
  );
}
```

### Preloading Components

```tsx
// Preload on hover/focus
const Settings = lazy(() => import('./pages/Settings'));

// Preload function
const preloadSettings = () => {
  import('./pages/Settings');
};

function Navigation() {
  return (
    <nav>
      <Link
        to="/settings"
        onMouseEnter={preloadSettings}
        onFocus={preloadSettings}
      >
        Settings
      </Link>
    </nav>
  );
}

// Preload after initial render
useEffect(() => {
  // Preload likely next pages after app loads
  const timer = setTimeout(() => {
    import('./pages/Dashboard');
    import('./pages/Profile');
  }, 2000);

  return () => clearTimeout(timer);
}, []);
```

### Dynamic Imports for Libraries

```tsx
function DocumentEditor() {
  const [editor, setEditor] = useState<typeof import('heavy-editor') | null>(null);

  useEffect(() => {
    // Load heavy library on mount
    import('heavy-editor').then(setEditor);
  }, []);

  if (!editor) {
    return <EditorSkeleton />;
  }

  return <editor.Editor />;
}

// With error handling
async function loadPdfLibrary() {
  try {
    const pdfLib = await import('pdf-lib');
    return pdfLib;
  } catch (error) {
    console.error('Failed to load PDF library:', error);
    throw error;
  }
}
```

---

## Error Handling

### Error Boundaries

```tsx
import { Component, ReactNode } from 'react';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary
      fallback={<ErrorPage />}
      onError={(error, info) => {
        // Log to error tracking service
        logErrorToService(error, info);
      }}
    >
      <Router>
        <Routes />
      </Router>
    </ErrorBoundary>
  );
}

// Granular error boundaries
function Dashboard() {
  return (
    <div className="dashboard">
      <ErrorBoundary fallback={<WidgetError />}>
        <ChartWidget />
      </ErrorBoundary>
      <ErrorBoundary fallback={<WidgetError />}>
        <StatsWidget />
      </ErrorBoundary>
    </div>
  );
}
```

### Hook-Based Error Handling

```tsx
// Custom hook for async error handling
function useAsyncError() {
  const [_, setError] = useState();

  return useCallback((error: Error) => {
    setError(() => {
      throw error; // This will be caught by error boundary
    });
  }, []);
}

// Usage
function DataComponent() {
  const throwError = useAsyncError();

  useEffect(() => {
    fetchData()
      .catch(error => {
        // Throw to nearest error boundary
        throwError(error);
      });
  }, [throwError]);

  return <div>Data</div>;
}
```

### Error Recovery Patterns

```tsx
interface RetryableErrorBoundaryProps {
  children: ReactNode;
  maxRetries?: number;
}

function RetryableErrorBoundary({
  children,
  maxRetries = 3,
}: RetryableErrorBoundaryProps) {
  const [retryCount, setRetryCount] = useState(0);
  const [key, setKey] = useState(0);

  const handleRetry = () => {
    if (retryCount < maxRetries) {
      setRetryCount(c => c + 1);
      setKey(k => k + 1); // Force remount
    }
  };

  return (
    <ErrorBoundary
      key={key}
      fallback={
        <div>
          <p>Error occurred</p>
          {retryCount < maxRetries ? (
            <button onClick={handleRetry}>
              Retry ({maxRetries - retryCount} attempts left)
            </button>
          ) : (
            <p>Max retries exceeded</p>
          )}
        </div>
      }
    >
      {children}
    </ErrorBoundary>
  );
}
```

---

## Composition Patterns

### Slots Pattern

```tsx
interface CardProps {
  children: ReactNode;
  header?: ReactNode;
  footer?: ReactNode;
}

function Card({ children, header, footer }: CardProps) {
  return (
    <div className="card">
      {header && <div className="card-header">{header}</div>}
      <div className="card-body">{children}</div>
      {footer && <div className="card-footer">{footer}</div>}
    </div>
  );
}

// Usage
function ProductCard({ product }: { product: Product }) {
  return (
    <Card
      header={<h2>{product.name}</h2>}
      footer={<button>Add to Cart</button>}
    >
      <img src={product.image} alt={product.name} />
      <p>{product.description}</p>
      <span>${product.price}</span>
    </Card>
  );
}
```

### Function as Children (FaCC)

```tsx
interface ToggleProps {
  children: (props: { on: boolean; toggle: () => void }) => ReactNode;
}

function Toggle({ children }: ToggleProps) {
  const [on, setOn] = useState(false);
  const toggle = useCallback(() => setOn(v => !v), []);

  return <>{children({ on, toggle })}</>;
}

// Usage
function App() {
  return (
    <Toggle>
      {({ on, toggle }) => (
        <div>
          <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>
          {on && <div>Content visible when ON</div>}
        </div>
      )}
    </Toggle>
  );
}
```

### Polymorphic Components

```tsx
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
  children: ReactNode;
} & Omit<React.ComponentPropsWithoutRef<E>, 'as' | 'children'>;

function Box<E extends React.ElementType = 'div'>({
  as,
  children,
  ...props
}: PolymorphicProps<E>) {
  const Component = as || 'div';
  return <Component {...props}>{children}</Component>;
}

// Usage
function App() {
  return (
    <>
      <Box>Default div</Box>
      <Box as="section">Section element</Box>
      <Box as="button" onClick={() => alert('clicked')}>
        Button element
      </Box>
      <Box as={Link} to="/about">
        React Router Link
      </Box>
    </>
  );
}
```

---

## Testing Patterns

### Component Testing with Testing Library

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Test user interactions
describe('LoginForm', () => {
  it('submits form with valid data', async () => {
    const handleSubmit = jest.fn();
    const user = userEvent.setup();

    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /login/i }));

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });

  it('shows validation errors', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.click(screen.getByRole('button', { name: /login/i }));

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(await screen.findByText(/password is required/i)).toBeInTheDocument();
  });
});

// Test async behavior
describe('UserProfile', () => {
  it('loads and displays user data', async () => {
    // Mock fetch
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ name: 'John Doe', email: 'john@example.com' }),
    });

    render(<UserProfile userId="1" />);

    // Check loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();

    // Wait for data
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
});
```

### Custom Hook Testing

```tsx
import { renderHook, act, waitFor } from '@testing-library/react';

describe('useCounter', () => {
  it('increments counter', () => {
    const { result } = renderHook(() => useCounter(0));

    expect(result.current.count).toBe(0);

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });
});

describe('useFetch', () => {
  it('fetches data successfully', async () => {
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ data: 'test' }),
    });

    const { result } = renderHook(() => useFetch('/api/data'));

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.data).toEqual({ data: 'test' });
    expect(result.current.error).toBeNull();
  });
});
```

---

## TypeScript Patterns

### Generic Components

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
  keyExtractor: (item: T) => string;
  emptyMessage?: string;
}

function List<T>({
  items,
  renderItem,
  keyExtractor,
  emptyMessage = 'No items',
}: ListProps<T>) {
  if (items.length === 0) {
    return <div className="empty">{emptyMessage}</div>;
  }

  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
interface User {
  id: string;
  name: string;
}

function UserList({ users }: { users: User[] }) {
  return (
    <List
      items={users}
      keyExtractor={user => user.id}
      renderItem={user => <span>{user.name}</span>}
    />
  );
}
```

### Discriminated Unions for State

```tsx
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useAsync<T>() {
  const [state, setState] = useState<AsyncState<T>>({ status: 'idle' });

  const execute = useCallback(async (promise: Promise<T>) => {
    setState({ status: 'loading' });
    try {
      const data = await promise;
      setState({ status: 'success', data });
    } catch (error) {
      setState({ status: 'error', error: error as Error });
    }
  }, []);

  return { ...state, execute };
}

// Usage with exhaustive type checking
function DataDisplay<T>({ state }: { state: AsyncState<T> }) {
  switch (state.status) {
    case 'idle':
      return <div>Ready to load</div>;
    case 'loading':
      return <Spinner />;
    case 'success':
      return <div>{JSON.stringify(state.data)}</div>;
    case 'error':
      return <div>Error: {state.error.message}</div>;
    default:
      // TypeScript will error if we forget a case
      const _exhaustive: never = state;
      return null;
  }
}
```

### Strict Event Handlers

```tsx
// Type-safe event handlers
type InputChangeHandler = ChangeEventHandler<HTMLInputElement>;
type ButtonClickHandler = MouseEventHandler<HTMLButtonElement>;

interface FormFieldProps {
  value: string;
  onChange: InputChangeHandler;
  onBlur?: FocusEventHandler<HTMLInputElement>;
}

function FormField({ value, onChange, onBlur }: FormFieldProps) {
  return <input value={value} onChange={onChange} onBlur={onBlur} />;
}

// Generic event handler type
type EventHandler<E extends SyntheticEvent> = (event: E) => void;

// Form event with typed target
function handleSubmit(e: FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const formData = new FormData(e.currentTarget);
  // ...
}
```

### Props with Children Type

```tsx
// Explicit children type
interface ContainerProps {
  children: ReactNode;
  className?: string;
}

// Function children
interface RenderProps<T> {
  children: (data: T) => ReactNode;
}

// Required single child
interface WrapperProps {
  children: ReactElement;
}

// Array of specific elements
interface TabsProps {
  children: ReactElement<TabProps>[];
}

// PropsWithChildren utility
type ButtonProps = PropsWithChildren<{
  variant: 'primary' | 'secondary';
  onClick: () => void;
}>;
```

---

## Quick Reference

### Performance Checklist

- [ ] Use `React.memo` for expensive pure components
- [ ] Use `useCallback` for functions passed to memoized children
- [ ] Use `useMemo` for expensive calculations
- [ ] Avoid inline object/array creation in JSX props
- [ ] Use proper keys (not array indices)
- [ ] Implement virtualization for long lists
- [ ] Code split routes and heavy components
- [ ] Debounce/throttle frequent updates
- [ ] Avoid unnecessary state (derive when possible)
- [ ] Split context to prevent unnecessary re-renders

### Common Anti-Patterns to Avoid

| Anti-Pattern | Better Approach |
|--------------|-----------------|
| Index as key | Use stable unique ID |
| Inline handlers with deps | useCallback |
| Object in dep array | useMemo or primitives |
| State for derived data | Calculate during render |
| Prop drilling | Context or composition |
| Giant useEffect | Split into multiple |
| Sync state from props | Controlled/uncontrolled |
