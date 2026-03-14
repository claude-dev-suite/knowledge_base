# React Performance Reference

## React.memo

```tsx
// Only re-renders when props change
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  return <div>{/* expensive render */}</div>;
});

// Custom comparison
const MemoizedList = memo(
  function List({ items }) { return <ul>...</ul>; },
  (prevProps, nextProps) => prevProps.items.length === nextProps.items.length
);
```

## useMemo

```tsx
function ProductList({ products, filter }) {
  // Only recalculates when products or filter change
  const filteredProducts = useMemo(() => {
    return products.filter(p => p.category === filter);
  }, [products, filter]);

  return <ul>{filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

## useCallback

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // Stable reference for child components
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []);

  return <MemoizedChild onClick={handleClick} />;
}
```

## Virtualization

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              height: virtualRow.size,
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

## Code Splitting

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));

// Route-based splitting
<Route
  path="/dashboard"
  element={
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  }
/>

// Preload on hover
const DashboardImport = () => import('./Dashboard');
<Link onMouseEnter={DashboardImport}>Dashboard</Link>
```

## useTransition

```tsx
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value); // High priority

    startTransition(() => {
      setResults(filterItems(value)); // Low priority, can be interrupted
    });
  };

  return (
    <div className={isPending ? 'opacity-50' : ''}>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      <ResultsList results={results} />
    </div>
  );
}
```

## useDeferredValue

```tsx
function Search({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div className={isStale ? 'opacity-50' : ''}>
      <SearchResults query={deferredQuery} />
    </div>
  );
}
```

## Avoiding Re-renders

```tsx
// ❌ Creates new object every render
<Child style={{ color: 'red' }} />

// ✅ Stable reference
const style = useMemo(() => ({ color: 'red' }), []);
<Child style={style} />

// ❌ Creates new function every render
<Child onClick={() => handleClick(id)} />

// ✅ Stable callback
const onClick = useCallback(() => handleClick(id), [id]);
<Child onClick={onClick} />

// ✅ Or use data attribute
<Child onClick={handleClick} data-id={id} />
```

## State Colocation

```tsx
// ❌ State too high - entire app re-renders
function App() {
  const [inputValue, setInputValue] = useState('');
  return (
    <Layout>
      <Input value={inputValue} onChange={setInputValue} />
      <ExpensiveComponent />
    </Layout>
  );
}

// ✅ State colocated - only Input re-renders
function App() {
  return (
    <Layout>
      <InputWithState />
      <ExpensiveComponent />
    </Layout>
  );
}

function InputWithState() {
  const [inputValue, setInputValue] = useState('');
  return <Input value={inputValue} onChange={setInputValue} />;
}
```

## React DevTools Profiler

1. Open React DevTools → Profiler tab
2. Click Record
3. Perform action
4. Stop recording
5. Analyze flame graph

Look for:
- Components rendering unnecessarily
- Long render times
- Cascading re-renders

## Key Performance Tips

| Do | Don't |
|----|-------|
| Memoize expensive calculations | Memoize everything |
| Split state by update frequency | Put all state together |
| Use keys properly in lists | Use index as key |
| Virtualize long lists | Render 1000+ items |
| Lazy load routes | Bundle everything |
| Colocate state | Lift state too high |

## Measuring Performance

```tsx
import { Profiler } from 'react';

function onRenderCallback(
  id, phase, actualDuration, baseDuration, startTime, commitTime
) {
  console.log(`${id} ${phase}: ${actualDuration}ms`);
}

<Profiler id="Navigation" onRender={onRenderCallback}>
  <Navigation />
</Profiler>
```
