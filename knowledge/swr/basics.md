# SWR Basics

SWR (stale-while-revalidate) is a React data fetching library.

## Installation

```bash
npm install swr
```

## Basic Usage

```typescript
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

function Profile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher);

  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  return <div>{data.name}</div>;
}
```

## Global Configuration

```typescript
import { SWRConfig } from 'swr';

function App() {
  return (
    <SWRConfig value={{
      fetcher,
      revalidateOnFocus: true,
      dedupingInterval: 2000,
    }}>
      <MyApp />
    </SWRConfig>
  );
}
```

## Return Values

```typescript
const {
  data,           // Data from fetcher
  error,          // Error from fetcher
  isLoading,      // Initial loading (no data yet)
  isValidating,   // Revalidating (may have stale data)
  mutate,         // Function to update cache
} = useSWR(key, fetcher, options);
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| revalidateOnFocus | true | Revalidate on window focus |
| revalidateOnReconnect | true | Revalidate on reconnect |
| refreshInterval | 0 | Polling interval (ms) |
| dedupingInterval | 2000 | Dedup window (ms) |
| errorRetryCount | 3 | Retry count on error |
| suspense | false | Enable Suspense |

## Conditional Fetching

```typescript
// Only fetch when id exists
const { data } = useSWR(id ? `/api/users/${id}` : null, fetcher);
```

## Mutations

```typescript
import useSWRMutation from 'swr/mutation';

async function createUser(url: string, { arg }: { arg: CreateUserDto }) {
  return fetch(url, { method: 'POST', body: JSON.stringify(arg) }).then(r => r.json());
}

function CreateUser() {
  const { trigger, isMutating } = useSWRMutation('/api/users', createUser);

  return <button onClick={() => trigger({ name: 'John' })}>Create</button>;
}
```

## Optimistic Updates

```typescript
const { data, mutate } = useSWR('/api/todos', fetcher);

const addTodo = async (text: string) => {
  // Optimistic update
  mutate([...data, { text }], { revalidate: false });

  await fetch('/api/todos', { method: 'POST', body: JSON.stringify({ text }) });
  mutate(); // Revalidate
};
```

## Infinite Loading

```typescript
import useSWRInfinite from 'swr/infinite';

const getKey = (page: number) => `/api/users?page=${page + 1}`;

const { data, size, setSize } = useSWRInfinite(getKey, fetcher);
const users = data?.flat() ?? [];
```

## Comparison with TanStack Query

| Feature | SWR | TanStack Query |
|---------|-----|----------------|
| Bundle size | ~4KB | ~12KB |
| API | Simpler | More features |
| DevTools | Basic | Advanced |
| Mutations | useSWRMutation | useMutation |
| Caching | Simpler | More control |
