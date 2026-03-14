# React Context Reference

## Creating Context

```tsx
import { createContext, useContext, useState } from 'react';

interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | null>(null);

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = () => setTheme(t => t === 'light' ? 'dark' : 'light');

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
}
```

## Usage

```tsx
function App() {
  return (
    <ThemeProvider>
      <Header />
      <Main />
    </ThemeProvider>
  );
}

function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();
  return <button onClick={toggleTheme}>Current: {theme}</button>;
}
```

## React 19: use() for Context

```tsx
import { use } from 'react';

function ThemeButton() {
  // Can be used conditionally!
  if (someCondition) {
    const theme = use(ThemeContext);
    return <button style={{ background: theme.primary }}>Click</button>;
  }
  return <button>Click</button>;
}
```

## Splitting State and Dispatch

```tsx
const StateContext = createContext<State | null>(null);
const DispatchContext = createContext<Dispatch | null>(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
}

// Components only re-render when their specific context changes
function DisplayCount() {
  const state = useContext(StateContext);
  return <span>{state.count}</span>;
}

function IncrementButton() {
  const dispatch = useContext(DispatchContext);
  return <button onClick={() => dispatch({ type: 'increment' })}>+</button>;
}
```

## Composing Providers

```tsx
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// Compose helper
function composeProviders(...providers: React.FC<{ children: React.ReactNode }>[]) {
  return ({ children }: { children: React.ReactNode }) =>
    providers.reduceRight((acc, Provider) => <Provider>{acc}</Provider>, children);
}

const Providers = composeProviders(AuthProvider, ThemeProvider, QueryProvider);
```

## Performance: Memoizing Value

```tsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  // Memoize to prevent unnecessary re-renders
  const value = useMemo(() => ({
    theme,
    toggleTheme: () => setTheme(t => t === 'light' ? 'dark' : 'light')
  }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## Context with Reducer

```tsx
interface State { count: number; }
type Action = { type: 'increment' } | { type: 'decrement' };

const CounterContext = createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
} | null>(null);

function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
  }
}

function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });
  return (
    <CounterContext.Provider value={{ state, dispatch }}>
      {children}
    </CounterContext.Provider>
  );
}
```

## Default Value Pattern

```tsx
const defaultValue: ThemeContextType = {
  theme: 'light',
  toggleTheme: () => console.warn('No ThemeProvider'),
};

const ThemeContext = createContext<ThemeContextType>(defaultValue);

// No null check needed
function useTheme() {
  return useContext(ThemeContext);
}
```

## When to Use Context

| Use Context | Use Props |
|-------------|-----------|
| Theme, locale, auth | Component-specific data |
| Deeply nested access | 1-2 levels deep |
| Many components need it | Few components need it |
| Rarely changes | Frequently changes |

## Common Patterns

```tsx
// Auth Context
const AuthContext = createContext<{
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
} | null>(null);

// Toast Context
const ToastContext = createContext<{
  toasts: Toast[];
  addToast: (message: string, type: ToastType) => void;
  removeToast: (id: string) => void;
} | null>(null);

// Modal Context
const ModalContext = createContext<{
  isOpen: boolean;
  content: React.ReactNode;
  openModal: (content: React.ReactNode) => void;
  closeModal: () => void;
} | null>(null);
```
