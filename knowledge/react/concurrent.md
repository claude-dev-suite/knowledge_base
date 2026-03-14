# React Concurrent Features Reference

## useTransition

Mark state updates as non-urgent (can be interrupted):

```tsx
function TabContainer() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab: string) {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <div className={isPending ? 'opacity-50' : ''}>
      <nav>
        {['home', 'about', 'contact'].map(t => (
          <button key={t} onClick={() => selectTab(t)}>{t}</button>
        ))}
      </nav>
      {tab === 'home' && <Home />}
      {tab === 'about' && <About />}
      {tab === 'contact' && <Contact />}
    </div>
  );
}
```

## Search with Transition

```tsx
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value); // Immediate: high priority

    startTransition(() => {
      const filtered = expensiveFilter(allItems, value);
      setResults(filtered); // Deferred: can be interrupted
    });
  };

  return (
    <div>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ul className={isPending ? 'opacity-50' : ''}>
        {results.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </div>
  );
}
```

## useDeferredValue

Defer updating a value to keep UI responsive:

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <div className={isStale ? 'opacity-50' : ''}>
        <SearchResults query={deferredQuery} />
      </div>
    </div>
  );
}

// MUST be memoized to benefit from deferred value
const SearchResults = memo(function SearchResults({ query }: { query: string }) {
  const results = useMemo(() => filterItems(query), [query]);
  return <ul>{results.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
});
```

## startTransition (without hook)

For use outside components:

```tsx
import { startTransition } from 'react';

// In event handler
function handleClick() {
  startTransition(() => {
    setPage('/heavy-page');
  });
}

// In effect
useEffect(() => {
  startTransition(() => {
    setData(processData(rawData));
  });
}, [rawData]);

// In async function
async function loadData() {
  const data = await fetchData();
  startTransition(() => {
    setData(data);
  });
}
```

## With Suspense

```tsx
function App() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  return (
    <div>
      <nav>
        <button onClick={() => startTransition(() => setTab('posts'))}>
          Posts
        </button>
      </nav>

      {/* Without transition: shows fallback immediately */}
      {/* With transition: keeps showing current tab while loading */}
      <Suspense fallback={<TabSkeleton />}>
        <div className={isPending ? 'opacity-50' : ''}>
          {tab === 'posts' && <Posts />}
        </div>
      </Suspense>
    </div>
  );
}
```

## Priority-Based Updates

```tsx
function Dashboard() {
  const [filter, setFilter] = useState('all');
  const [data, setData] = useState<Data[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleFilterChange = (newFilter: string) => {
    setFilter(newFilter); // High priority: UI feedback

    startTransition(() => {
      const filtered = processLargeDataset(rawData, newFilter);
      setData(filtered); // Low priority: data processing
    });
  };

  return (
    <div>
      <FilterButtons selected={filter} onChange={handleFilterChange} />
      {isPending && <LoadingOverlay />}
      <DataGrid data={data} />
    </div>
  );
}
```

## Async Transitions (React 19)

```tsx
function CommentForm({ postId }) {
  const [comments, setComments] = useState<Comment[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = async (formData: FormData) => {
    const text = formData.get('comment') as string;

    startTransition(async () => {
      const newComment = await submitComment(postId, text);
      setComments(prev => [...prev, newComment]);
    });
  };

  return (
    <form action={handleSubmit}>
      <textarea name="comment" required />
      <button disabled={isPending}>
        {isPending ? 'Posting...' : 'Post'}
      </button>
    </form>
  );
}
```

## useTransition vs useDeferredValue

| Feature | useTransition | useDeferredValue |
|---------|---------------|------------------|
| Purpose | Wrap state updates | Defer a value |
| Usage | You control the update | You receive a value |
| Returns | `[isPending, startTransition]` | Deferred value |
| Use case | Button clicks, form submits | Props from parent |

```tsx
// useTransition: You control when to update
function Parent() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    startTransition(() => setQuery(e.target.value));
  };
}

// useDeferredValue: You receive a value and defer it
function Child({ query }) {
  const deferredQuery = useDeferredValue(query);
  return <ExpensiveList filter={deferredQuery} />;
}
```

## When to Use

| Use | When |
|-----|------|
| useTransition | User-initiated heavy updates |
| useDeferredValue | Received props for expensive renders |
| Neither | Fast operations (don't need it) |

## Common Pitfalls

| Issue | Cause | Solution |
|-------|-------|----------|
| No improvement | Component not memoized | Add `memo()` |
| Still blocking | Synchronous heavy work | Move to Web Worker |
| isPending always false | Update is fast | Remove transition |
