# TanStack Query v4 → Basics Delta

## Not Available in v4

- `useSuspenseQuery` hook (v5+)
- Simplified optimistic updates
- Streaming SSR support
- Built-in eslint plugin

## Breaking Changes from v4 to v5

### Status Naming

```typescript
// TanStack Query v5
const { data, isPending, isLoading } = useQuery({...})
// isPending: No data yet, query is fetching
// isLoading: isPending AND isFetching (first load)

// TanStack Query v4
const { data, isLoading, isInitialLoading } = useQuery({...})
// isLoading: Query is fetching (any time)
// isInitialLoading: No data AND fetching
```

### Status Mapping

| v4 | v5 |
|----|-----|
| `status: 'loading'` | `status: 'pending'` |
| `isLoading` | `isPending` |
| `isInitialLoading` | `isLoading` |

### Callbacks Removed from useQuery

```typescript
// TanStack Query v4 - Callbacks in useQuery
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  onSuccess: (data) => {
    console.log('Success:', data)
  },
  onError: (error) => {
    console.log('Error:', error)
  },
  onSettled: (data, error) => {
    console.log('Settled')
  }
})

// TanStack Query v5 - Use useEffect instead
const { data, error, isSuccess, isError } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

useEffect(() => {
  if (isSuccess) {
    console.log('Success:', data)
  }
}, [isSuccess, data])

useEffect(() => {
  if (isError) {
    console.log('Error:', error)
  }
}, [isError, error])
```

### cacheTime → gcTime

```typescript
// TanStack Query v4
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      cacheTime: 1000 * 60 * 5 // 5 minutes
    }
  }
})

// TanStack Query v5
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 5 // 5 minutes (gc = garbage collection)
    }
  }
})
```

### keepPreviousData → placeholderData

```typescript
// TanStack Query v4
const { data } = useQuery({
  queryKey: ['users', page],
  queryFn: () => fetchUsers(page),
  keepPreviousData: true
})

// TanStack Query v5
import { keepPreviousData } from '@tanstack/react-query'

const { data } = useQuery({
  queryKey: ['users', page],
  queryFn: () => fetchUsers(page),
  placeholderData: keepPreviousData
})
```

### Mandatory queryFn

```typescript
// TanStack Query v4 - queryFn could be undefined
const { data } = useQuery({
  queryKey: ['user'],
  // No queryFn - would use default from queryClient
})

// TanStack Query v5 - queryFn required
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser  // Required!
})
```

### Suspense Queries

```typescript
// TanStack Query v5 - Dedicated hook
import { useSuspenseQuery } from '@tanstack/react-query'

function UserProfile() {
  const { data } = useSuspenseQuery({
    queryKey: ['user'],
    queryFn: fetchUser
  })
  // data is always defined here
  return <div>{data.name}</div>
}

// TanStack Query v4 - suspense option
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  suspense: true  // data could still be undefined in types
})
```

### Optimistic Updates

```typescript
// TanStack Query v5 - Simplified
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })
    const previousTodos = queryClient.getQueryData(['todos'])
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo])
    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})

// TanStack Query v4 - Same pattern, but callbacks work differently
```

## Still Current in v4

- useQuery, useMutation, useQueryClient hooks
- Query keys as arrays
- Query invalidation
- Infinite queries
- Prefetching
- Query cancellation
- Retry logic
- Stale/fresh concepts

## Recommendations for v4 Users

1. **Rename status checks** when upgrading:
   - `isLoading` → `isPending`
   - `isInitialLoading` → `isLoading`
2. **Move callbacks to useEffect**
3. **Rename cacheTime to gcTime**
4. **Use placeholderData with keepPreviousData function**
5. **Add queryFn to all queries**

