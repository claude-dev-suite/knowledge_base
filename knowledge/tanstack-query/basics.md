# TanStack Query (React Query)

> Powerful data fetching and caching library for React with automatic background updates and intelligent caching.

## Table of Contents

1. [Setup and Configuration](#setup-and-configuration)
2. [Queries](#queries)
3. [Mutations](#mutations)
4. [Query Invalidation](#query-invalidation)
5. [Optimistic Updates](#optimistic-updates)
6. [Pagination and Infinite Queries](#pagination-and-infinite-queries)
7. [Prefetching](#prefetching)
8. [Error Handling](#error-handling)
9. [Testing](#testing)
10. [Best Practices](#best-practices)

---

## Setup and Configuration

### Basic Setup

```tsx
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,           // 1 minute
            gcTime: 10 * 60 * 1000,         // 10 minutes (formerly cacheTime)
            retry: 3,
            retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
            refetchOnWindowFocus: false,
            refetchOnReconnect: true,
          },
          mutations: {
            retry: 1,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}

// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Query Client Configuration for Production

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Caching
      staleTime: 5 * 60 * 1000,          // Data considered fresh for 5 minutes
      gcTime: 30 * 60 * 1000,            // Garbage collect after 30 minutes

      // Refetching
      refetchOnMount: true,              // Refetch on component mount
      refetchOnWindowFocus: true,        // Refetch when tab regains focus
      refetchOnReconnect: true,          // Refetch on network reconnect

      // Error handling
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),

      // Network mode
      networkMode: 'online',             // 'online' | 'always' | 'offlineFirst'
    },
    mutations: {
      retry: 1,
      networkMode: 'online',
    },
  },
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Global error handling
      if (query.state.data !== undefined) {
        toast.error(`Error updating: ${error.message}`);
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error, _variables, _context, mutation) => {
      if (mutation.options.onError) return;
      toast.error(`Mutation failed: ${error.message}`);
    },
  }),
});
```

---

## Queries

### Basic Query

```tsx
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) {
    throw new Error('Failed to fetch user');
  }
  return response.json();
}

function UserProfile({ userId }: { userId: string }) {
  const {
    data,
    isPending,      // Initial loading
    isLoading,      // No data yet (pending + no data)
    isFetching,     // Any fetch in progress
    isError,
    error,
    isSuccess,
    refetch,
    status,
    fetchStatus,
  } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isPending) return <Skeleton />;
  if (isError) return <Error message={error.message} retry={refetch} />;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}
```

### Query Keys

```tsx
// Simple key
useQuery({ queryKey: ['todos'], queryFn: fetchTodos });

// With variables (changes trigger refetch)
useQuery({ queryKey: ['todo', todoId], queryFn: () => fetchTodo(todoId) });

// With filters
useQuery({
  queryKey: ['todos', { status: 'done', page: 1 }],
  queryFn: () => fetchTodos({ status: 'done', page: 1 }),
});

// Hierarchical (for invalidation)
useQuery({ queryKey: ['users', userId, 'posts'], queryFn: () => fetchUserPosts(userId) });

// Query key factory pattern
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: Filters) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: string) => [...todoKeys.details(), id] as const,
};

// Usage
useQuery({ queryKey: todoKeys.detail(todoId), queryFn: () => fetchTodo(todoId) });
queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
```

### Query Options

```tsx
const { data } = useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,

  // Caching
  staleTime: 5 * 60 * 1000,     // Data fresh for 5 minutes
  gcTime: 30 * 60 * 1000,       // Cache for 30 minutes

  // Refetching
  refetchOnMount: true,
  refetchOnWindowFocus: true,
  refetchOnReconnect: true,
  refetchInterval: 60000,       // Poll every minute
  refetchIntervalInBackground: false,

  // Retries
  retry: 3,
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),

  // Conditional fetching
  enabled: !!userId,            // Only run when userId exists

  // Placeholders and initial data
  placeholderData: previousData,
  placeholderData: (previousData) => previousData,
  initialData: () => getCachedData(),
  initialDataUpdatedAt: Date.now(),

  // Select/transform
  select: (data) => data.filter((user) => user.active),

  // Structure sharing (deep comparison)
  structuralSharing: true,

  // Meta
  meta: { errorMessage: 'Failed to fetch users' },
});
```

### Dependent Queries

```tsx
function UserPosts({ userId }: { userId: string }) {
  // First query
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Dependent query - waits for user data
  const postsQuery = useQuery({
    queryKey: ['posts', userQuery.data?.id],
    queryFn: () => fetchPostsByUser(userQuery.data!.id),
    enabled: !!userQuery.data,  // Only runs when user data exists
  });

  if (userQuery.isPending) return <div>Loading user...</div>;
  if (userQuery.isError) return <div>Error loading user</div>;
  if (postsQuery.isPending) return <div>Loading posts...</div>;

  return <PostList posts={postsQuery.data ?? []} />;
}
```

### Parallel Queries

```tsx
function Dashboard() {
  // Multiple independent queries
  const usersQuery = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const statsQuery = useQuery({ queryKey: ['stats'], queryFn: fetchStats });

  const isLoading = usersQuery.isPending || postsQuery.isPending || statsQuery.isPending;

  if (isLoading) return <Loading />;

  return (
    <div>
      <UserList users={usersQuery.data} />
      <PostList posts={postsQuery.data} />
      <Stats data={statsQuery.data} />
    </div>
  );
}

// Or with useQueries for dynamic queries
function UsersList({ userIds }: { userIds: string[] }) {
  const queries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
    combine: (results) => ({
      data: results.map((result) => result.data),
      pending: results.some((result) => result.isPending),
    }),
  });

  if (queries.pending) return <Loading />;

  return <>{queries.data.map((user) => user && <UserCard key={user.id} user={user} />)}</>;
}
```

---

## Mutations

### Basic Mutation

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface CreateTodoDto {
  title: string;
  completed?: boolean;
}

async function createTodo(data: CreateTodoDto): Promise<Todo> {
  const response = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create todo');
  return response.json();
}

function CreateTodoForm() {
  const queryClient = useQueryClient();
  const [title, setTitle] = useState('');

  const mutation = useMutation({
    mutationFn: createTodo,
    onSuccess: (newTodo) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['todos'] });
      setTitle('');
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate({ title });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Todo title"
      />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create'}
      </button>
      {mutation.isError && <p className="error">{mutation.error.message}</p>}
    </form>
  );
}
```

### Mutation Callbacks

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,

  // Before mutation
  onMutate: async (variables) => {
    // Called before mutationFn
    // Can return context for rollback
    return { previousData: queryClient.getQueryData(['todos']) };
  },

  // On success
  onSuccess: (data, variables, context) => {
    // data: return value from mutationFn
    // variables: variables passed to mutate
    // context: value returned from onMutate
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },

  // On error
  onError: (error, variables, context) => {
    // Rollback on error
    if (context?.previousData) {
      queryClient.setQueryData(['todos'], context.previousData);
    }
    toast.error(error.message);
  },

  // Always called (success or error)
  onSettled: (data, error, variables, context) => {
    // Good place for cleanup
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

// You can also pass callbacks to mutate()
mutation.mutate(data, {
  onSuccess: (data) => {
    // Component-specific success handling
  },
  onError: (error) => {
    // Component-specific error handling
  },
});
```

### Mutation State

```tsx
function TodoItem({ todo }: { todo: Todo }) {
  const updateMutation = useMutation({ mutationFn: updateTodo });
  const deleteMutation = useMutation({ mutationFn: deleteTodo });

  return (
    <div className={updateMutation.isPending ? 'opacity-50' : ''}>
      <span>{todo.title}</span>

      <button
        onClick={() => updateMutation.mutate({ ...todo, completed: !todo.completed })}
        disabled={updateMutation.isPending}
      >
        {todo.completed ? 'Undo' : 'Complete'}
      </button>

      <button
        onClick={() => deleteMutation.mutate(todo.id)}
        disabled={deleteMutation.isPending}
      >
        {deleteMutation.isPending ? 'Deleting...' : 'Delete'}
      </button>

      {updateMutation.isError && <span>Update failed!</span>}
      {deleteMutation.isError && <span>Delete failed!</span>}
    </div>
  );
}
```

---

## Query Invalidation

### Invalidation Patterns

```tsx
const queryClient = useQueryClient();

// Invalidate all queries
queryClient.invalidateQueries();

// Invalidate all queries starting with 'todos'
queryClient.invalidateQueries({ queryKey: ['todos'] });

// Invalidate exact match
queryClient.invalidateQueries({ queryKey: ['todos', 1], exact: true });

// Invalidate with predicate
queryClient.invalidateQueries({
  predicate: (query) =>
    query.queryKey[0] === 'todos' &&
    (query.queryKey[1] as any)?.status === 'pending',
});

// Invalidate and refetch
queryClient.invalidateQueries({
  queryKey: ['todos'],
  refetchType: 'active',  // 'active' | 'inactive' | 'all' | 'none'
});

// Remove queries from cache entirely
queryClient.removeQueries({ queryKey: ['todos'] });

// Reset queries to initial state
queryClient.resetQueries({ queryKey: ['todos'] });
```

### After Mutations

```tsx
const mutation = useMutation({
  mutationFn: createTodo,
  onSuccess: () => {
    // Invalidate all todo queries
    queryClient.invalidateQueries({ queryKey: ['todos'] });

    // Or invalidate specific queries
    queryClient.invalidateQueries({ queryKey: ['todos', 'list'] });

    // Or update cache directly
    queryClient.setQueryData(['todos', 'list'], (old: Todo[]) => [
      ...old,
      newTodo,
    ]);
  },
});
```

---

## Optimistic Updates

### Basic Optimistic Update

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,

  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos', newTodo.id] });

    // Snapshot previous value
    const previousTodo = queryClient.getQueryData(['todos', newTodo.id]);

    // Optimistically update
    queryClient.setQueryData(['todos', newTodo.id], newTodo);

    // Return context with snapshot
    return { previousTodo };
  },

  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos', newTodo.id], context?.previousTodo);
  },

  onSettled: (data, error, variables) => {
    // Always refetch to ensure consistency
    queryClient.invalidateQueries({ queryKey: ['todos', variables.id] });
  },
});
```

### Optimistic Update with List

```tsx
const mutation = useMutation({
  mutationFn: toggleTodo,

  onMutate: async (todoId) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

    queryClient.setQueryData<Todo[]>(['todos'], (old) =>
      old?.map((todo) =>
        todo.id === todoId ? { ...todo, completed: !todo.completed } : todo
      )
    );

    return { previousTodos };
  },

  onError: (err, todoId, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos);
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

### Optimistic Delete

```tsx
const deleteMutation = useMutation({
  mutationFn: deleteTodo,

  onMutate: async (todoId) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

    // Optimistically remove
    queryClient.setQueryData<Todo[]>(['todos'], (old) =>
      old?.filter((todo) => todo.id !== todoId)
    );

    return { previousTodos };
  },

  onError: (err, todoId, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos);
    toast.error('Failed to delete todo');
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

### Optimistic Create

```tsx
const createMutation = useMutation({
  mutationFn: createTodo,

  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

    // Create temporary ID
    const tempTodo: Todo = {
      ...newTodo,
      id: `temp-${Date.now()}`,
      createdAt: new Date().toISOString(),
    };

    // Optimistically add
    queryClient.setQueryData<Todo[]>(['todos'], (old) => [
      tempTodo,
      ...(old ?? []),
    ]);

    return { previousTodos, tempId: tempTodo.id };
  },

  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos);
  },

  onSuccess: (createdTodo, variables, context) => {
    // Replace temp todo with real one
    queryClient.setQueryData<Todo[]>(['todos'], (old) =>
      old?.map((todo) =>
        todo.id === context?.tempId ? createdTodo : todo
      )
    );
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

---

## Pagination and Infinite Queries

### Standard Pagination

```tsx
interface PaginatedResponse<T> {
  data: T[];
  page: number;
  totalPages: number;
  total: number;
}

function PaginatedTodos() {
  const [page, setPage] = useState(1);

  const { data, isPending, isPlaceholderData } = useQuery({
    queryKey: ['todos', { page }],
    queryFn: () => fetchTodos({ page, limit: 10 }),
    placeholderData: keepPreviousData,  // Keep previous data while fetching
  });

  return (
    <div>
      {isPending ? (
        <Loading />
      ) : (
        <>
          <TodoList todos={data.data} />
          <div className="pagination">
            <button
              onClick={() => setPage((p) => Math.max(1, p - 1))}
              disabled={page === 1}
            >
              Previous
            </button>
            <span>
              Page {page} of {data.totalPages}
            </span>
            <button
              onClick={() => setPage((p) => Math.min(data.totalPages, p + 1))}
              disabled={page >= data.totalPages || isPlaceholderData}
            >
              Next
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

### Infinite Query

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

interface InfiniteResponse {
  data: Todo[];
  nextCursor?: string;
}

function InfiniteTodos() {
  const {
    data,
    fetchNextPage,
    fetchPreviousPage,
    hasNextPage,
    hasPreviousPage,
    isFetchingNextPage,
    isFetchingPreviousPage,
    isPending,
    isError,
    error,
  } = useInfiniteQuery({
    queryKey: ['todos', 'infinite'],
    queryFn: ({ pageParam }) => fetchTodos({ cursor: pageParam, limit: 20 }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    getPreviousPageParam: (firstPage) => firstPage.prevCursor,
  });

  if (isPending) return <Loading />;
  if (isError) return <Error error={error} />;

  return (
    <div>
      {data.pages.map((page, i) => (
        <React.Fragment key={i}>
          {page.data.map((todo) => (
            <TodoItem key={todo.id} todo={todo} />
          ))}
        </React.Fragment>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
          ? 'Load More'
          : 'Nothing more to load'}
      </button>
    </div>
  );
}
```

### Infinite Query with Intersection Observer

```tsx
function InfiniteScrollTodos() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({
      queryKey: ['todos', 'infinite'],
      queryFn: ({ pageParam }) => fetchTodos({ cursor: pageParam }),
      initialPageParam: undefined,
      getNextPageParam: (lastPage) => lastPage.nextCursor,
    });

  const loadMoreRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );

    if (loadMoreRef.current) {
      observer.observe(loadMoreRef.current);
    }

    return () => observer.disconnect();
  }, [fetchNextPage, hasNextPage, isFetchingNextPage]);

  return (
    <div>
      {data?.pages.flatMap((page) =>
        page.data.map((todo) => <TodoItem key={todo.id} todo={todo} />)
      )}

      <div ref={loadMoreRef} style={{ height: 20 }}>
        {isFetchingNextPage && <Loading />}
      </div>
    </div>
  );
}
```

---

## Prefetching

### Prefetch on Hover

```tsx
function TodoList() {
  const queryClient = useQueryClient();

  const prefetchTodo = (todoId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['todo', todoId],
      queryFn: () => fetchTodo(todoId),
      staleTime: 5 * 60 * 1000,  // Only prefetch if data is older than 5 minutes
    });
  };

  return (
    <ul>
      {todos.map((todo) => (
        <li
          key={todo.id}
          onMouseEnter={() => prefetchTodo(todo.id)}
        >
          <Link href={`/todos/${todo.id}`}>{todo.title}</Link>
        </li>
      ))}
    </ul>
  );
}
```

### Prefetch on Route Change

```tsx
// Next.js example
import { useRouter } from 'next/router';

function TodoList() {
  const router = useRouter();
  const queryClient = useQueryClient();

  const handleClick = async (todoId: string) => {
    // Prefetch before navigation
    await queryClient.prefetchQuery({
      queryKey: ['todo', todoId],
      queryFn: () => fetchTodo(todoId),
    });

    router.push(`/todos/${todoId}`);
  };

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id} onClick={() => handleClick(todo.id)}>
          {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

### Prefetch in Loader (React Router / Remix)

```tsx
// React Router v6 loader
export async function loader({ params }: LoaderFunctionArgs) {
  const queryClient = getQueryClient();

  await queryClient.prefetchQuery({
    queryKey: ['todo', params.todoId],
    queryFn: () => fetchTodo(params.todoId!),
  });

  return null;
}

// Component uses the cached data
function TodoPage() {
  const { todoId } = useParams();
  const { data } = useQuery({
    queryKey: ['todo', todoId],
    queryFn: () => fetchTodo(todoId!),
  });

  return <TodoDetails todo={data} />;
}
```

---

## Error Handling

### Query Error Handling

```tsx
function TodoList() {
  const {
    data,
    error,
    isError,
    refetch,
    failureCount,
    failureReason,
  } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
  });

  if (isError) {
    return (
      <div className="error">
        <h2>Something went wrong</h2>
        <p>{error.message}</p>
        <p>Failed {failureCount} times</p>
        <button onClick={() => refetch()}>Retry</button>
      </div>
    );
  }

  return <div>{/* render todos */}</div>;
}
```

### Error Boundary Integration

```tsx
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ error, resetErrorBoundary }) => (
            <div className="error">
              <h2>Something went wrong</h2>
              <pre>{error.message}</pre>
              <button onClick={resetErrorBoundary}>Try again</button>
            </div>
          )}
        >
          <TodoList />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}

// Component with throwOnError
function TodoList() {
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    throwOnError: true,  // Throw to nearest error boundary
  });

  return <div>{/* render todos */}</div>;
}
```

### Global Error Handler

```tsx
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only show toast for background updates
      if (query.state.data !== undefined) {
        toast.error(`Background update failed: ${error.message}`);
      }

      // Log to error tracking service
      errorTracker.capture(error, {
        queryKey: query.queryKey,
      });
    },
  }),
});
```

---

## Testing

### Testing Queries

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render, screen, waitFor } from '@testing-library/react';

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: Infinity,
      },
    },
  });

const wrapper = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={createTestQueryClient()}>
    {children}
  </QueryClientProvider>
);

describe('TodoList', () => {
  it('renders todos', async () => {
    // Mock fetch
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve([{ id: '1', title: 'Test Todo' }]),
    });

    render(<TodoList />, { wrapper });

    expect(screen.getByText('Loading...')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.getByText('Test Todo')).toBeInTheDocument();
    });
  });

  it('handles error', async () => {
    global.fetch = jest.fn().mockRejectedValue(new Error('Network error'));

    render(<TodoList />, { wrapper });

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

### Testing Mutations

```tsx
import { act, renderHook, waitFor } from '@testing-library/react';

describe('useCreateTodo', () => {
  it('creates a todo', async () => {
    const mockTodo = { id: '1', title: 'New Todo' };

    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve(mockTodo),
    });

    const { result } = renderHook(() => useCreateTodo(), { wrapper });

    await act(async () => {
      result.current.mutate({ title: 'New Todo' });
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual(mockTodo);
  });
});
```

---

## Best Practices

### Query Key Conventions

```tsx
// Use factory pattern for query keys
const queryKeys = {
  todos: {
    all: ['todos'] as const,
    lists: () => [...queryKeys.todos.all, 'list'] as const,
    list: (filters: TodoFilters) => [...queryKeys.todos.lists(), filters] as const,
    details: () => [...queryKeys.todos.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.todos.details(), id] as const,
  },
  users: {
    all: ['users'] as const,
    // ...
  },
};
```

### Custom Hooks

```tsx
// hooks/useTodos.ts
export function useTodos(filters?: TodoFilters) {
  return useQuery({
    queryKey: queryKeys.todos.list(filters ?? {}),
    queryFn: () => fetchTodos(filters),
    staleTime: 5 * 60 * 1000,
  });
}

export function useTodo(id: string) {
  return useQuery({
    queryKey: queryKeys.todos.detail(id),
    queryFn: () => fetchTodo(id),
    enabled: !!id,
  });
}

export function useCreateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.todos.lists() });
    },
  });
}
```

### Performance Tips

```tsx
// 1. Use select to transform data
const { data: completedTodos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (todos) => todos.filter((t) => t.completed),
});

// 2. Use staleTime appropriately
// Data that rarely changes: longer staleTime
// Real-time data: shorter staleTime or refetchInterval

// 3. Prefetch data for better UX
// On hover, before navigation, in loaders

// 4. Use placeholderData for instant feedback
const { data } = useQuery({
  queryKey: ['todos', page],
  queryFn: () => fetchTodos({ page }),
  placeholderData: keepPreviousData,
});

// 5. Batch invalidations
queryClient.invalidateQueries({
  queryKey: ['todos'],
  refetchType: 'active',  // Only refetch currently rendered queries
});
```

### Common Pitfalls

```tsx
// DON'T: Create new query function reference on every render
function Bad() {
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: async () => {  // New function every render!
      return fetch('/api/todos').then((r) => r.json());
    },
  });
}

// DO: Define query function outside or use useCallback
const fetchTodos = async () => {
  return fetch('/api/todos').then((r) => r.json());
};

function Good() {
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,  // Stable reference
  });
}

// DON'T: Forget to include dependencies in query key
function Bad({ filter }: { filter: string }) {
  const { data } = useQuery({
    queryKey: ['todos'],  // Missing filter!
    queryFn: () => fetchTodos({ filter }),
  });
}

// DO: Include all dependencies in query key
function Good({ filter }: { filter: string }) {
  const { data } = useQuery({
    queryKey: ['todos', { filter }],  // Filter included
    queryFn: () => fetchTodos({ filter }),
  });
}
```
