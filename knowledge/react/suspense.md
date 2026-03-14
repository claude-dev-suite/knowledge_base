# React Suspense Reference

## Basic Usage

```tsx
import { Suspense } from 'react';

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <AsyncComponent />
    </Suspense>
  );
}
```

## Code Splitting with lazy()

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

## Named Exports

```tsx
const MyComponent = lazy(() =>
  import('./components').then(module => ({ default: module.MyComponent }))
);
```

## Data Fetching (React 19)

```tsx
// With use() hook
function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

function App() {
  const userPromise = fetchUser(userId);

  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

## Nested Suspense

```tsx
function Dashboard() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <Header />
      <Suspense fallback={<ChartSkeleton />}>
        <Charts />
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <DataTable />
      </Suspense>
    </Suspense>
  );
}
```

## With Error Boundary

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <ErrorBoundary fallback={<ErrorPage />}>
      <Suspense fallback={<Loading />}>
        <AsyncContent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## SuspenseList (Experimental)

```tsx
import { SuspenseList, Suspense } from 'react';

function Feed() {
  return (
    <SuspenseList revealOrder="forwards" tail="collapsed">
      <Suspense fallback={<PostSkeleton />}>
        <Post id={1} />
      </Suspense>
      <Suspense fallback={<PostSkeleton />}>
        <Post id={2} />
      </Suspense>
      <Suspense fallback={<PostSkeleton />}>
        <Post id={3} />
      </Suspense>
    </SuspenseList>
  );
}
```

Options:
- `revealOrder`: "forwards" | "backwards" | "together"
- `tail`: "collapsed" | "hidden"

## Streaming SSR (Next.js)

```tsx
// app/page.tsx
async function Page() {
  return (
    <div>
      <Header />
      <Suspense fallback={<ContentSkeleton />}>
        <SlowContent />
      </Suspense>
      <Footer />
    </div>
  );
}
```

## Preloading on Hover

```tsx
const DashboardImport = () => import('./Dashboard');
const Dashboard = lazy(DashboardImport);

function NavLink() {
  return (
    <Link
      to="/dashboard"
      onMouseEnter={DashboardImport}
    >
      Dashboard
    </Link>
  );
}
```

## With Transitions

```tsx
function TabContainer() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  return (
    <div className={isPending ? 'opacity-50' : ''}>
      <Suspense fallback={<TabSkeleton />}>
        {tab === 'home' && <Home />}
        {tab === 'posts' && <Posts />}
      </Suspense>
    </div>
  );
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Place Suspense close to async content | Wrap entire app in single Suspense |
| Use meaningful fallbacks | Show generic spinners |
| Combine with Error Boundaries | Ignore error states |
| Preload on user intent | Wait for navigation |
| Use nested boundaries | Single boundary for all |
