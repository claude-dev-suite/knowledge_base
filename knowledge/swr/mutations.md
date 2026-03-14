# SWR Mutations

Advanced patterns for data mutations with SWR.

## useSWRMutation

```typescript
import useSWRMutation from 'swr/mutation';

async function createUser(url: string, { arg }: { arg: CreateUserDto }) {
  return fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(arg),
  }).then(res => res.json());
}

function CreateUser() {
  const { trigger, isMutating, error, data, reset } = useSWRMutation(
    '/api/users',
    createUser
  );

  const handleSubmit = async (userData: CreateUserDto) => {
    try {
      const result = await trigger(userData);
      console.log('Created:', result);
    } catch (e) {
      console.error('Failed:', e);
    }
  };

  return (
    <button onClick={() => handleSubmit({ name: 'John' })} disabled={isMutating}>
      {isMutating ? 'Creating...' : 'Create User'}
    </button>
  );
}
```

## Optimistic Updates

```typescript
function TodoList() {
  const { data, mutate } = useSWR<Todo[]>('/api/todos', fetcher);

  const addTodo = async (text: string) => {
    const newTodo = { id: Date.now(), text, completed: false };

    // Optimistic update - immediately show the new item
    await mutate(
      async (currentData) => {
        // This runs after the optimistic update
        const response = await fetch('/api/todos', {
          method: 'POST',
          body: JSON.stringify({ text }),
        });
        const created = await response.json();
        return [...(currentData || []).filter(t => t.id !== newTodo.id), created];
      },
      {
        optimisticData: [...(data || []), newTodo],
        rollbackOnError: true,
        revalidate: false,
      }
    );
  };

  const deleteTodo = async (id: number) => {
    await mutate(
      async (currentData) => {
        await fetch(`/api/todos/${id}`, { method: 'DELETE' });
        return currentData?.filter(t => t.id !== id);
      },
      {
        optimisticData: data?.filter(t => t.id !== id),
        rollbackOnError: true,
      }
    );
  };

  const toggleTodo = async (id: number) => {
    const todo = data?.find(t => t.id === id);
    if (!todo) return;

    await mutate(
      async (currentData) => {
        const response = await fetch(`/api/todos/${id}`, {
          method: 'PATCH',
          body: JSON.stringify({ completed: !todo.completed }),
        });
        const updated = await response.json();
        return currentData?.map(t => t.id === id ? updated : t);
      },
      {
        optimisticData: data?.map(t =>
          t.id === id ? { ...t, completed: !t.completed } : t
        ),
        rollbackOnError: true,
      }
    );
  };
}
```

## Bound Mutate vs Global Mutate

```typescript
import { mutate } from 'swr';

// Global mutate - affects any component using this key
mutate('/api/users');

// With data
mutate('/api/users', newData);

// With options
mutate('/api/users', newData, { revalidate: false });

// Bound mutate (from useSWR) - same key, component-specific
function UserProfile() {
  const { data, mutate: boundMutate } = useSWR('/api/user', fetcher);

  // These are equivalent:
  boundMutate(newData);
  mutate('/api/user', newData);
}
```

## Mutate Multiple Keys

```typescript
import { mutate } from 'swr';

// Mutate by key prefix (using filter function)
mutate(
  key => typeof key === 'string' && key.startsWith('/api/users'),
  undefined,
  { revalidate: true }
);

// Mutate specific keys
const updateUserEverywhere = async (userId: string, data: UserUpdate) => {
  await fetch(`/api/users/${userId}`, {
    method: 'PUT',
    body: JSON.stringify(data),
  });

  // Revalidate all related caches
  mutate(`/api/users/${userId}`);
  mutate('/api/users');
  mutate(key => typeof key === 'string' && key.includes(`user=${userId}`));
};
```

## Mutation with Revalidation

```typescript
function UserSettings() {
  const { data: user, mutate } = useSWR('/api/user', fetcher);

  const updateSettings = async (settings: UserSettings) => {
    // Option 1: Revalidate after mutation
    await fetch('/api/user/settings', {
      method: 'PUT',
      body: JSON.stringify(settings),
    });
    mutate(); // Triggers revalidation

    // Option 2: Update locally, then revalidate in background
    mutate(
      { ...user, settings },
      { revalidate: true }
    );

    // Option 3: Update locally, skip revalidation
    mutate(
      { ...user, settings },
      { revalidate: false }
    );
  };
}
```

## Error Handling in Mutations

```typescript
function CreatePost() {
  const { trigger, error, reset } = useSWRMutation('/api/posts', createPost);

  const handleSubmit = async (data: PostData) => {
    try {
      await trigger(data, {
        onSuccess: (data) => {
          toast.success('Post created!');
          router.push(`/posts/${data.id}`);
        },
        onError: (error) => {
          toast.error(error.message);
        },
      });
    } catch (e) {
      // Error is also available via the error return value
    }
  };

  return (
    <div>
      {error && (
        <div className="error">
          {error.message}
          <button onClick={reset}>Dismiss</button>
        </div>
      )}
      <PostForm onSubmit={handleSubmit} />
    </div>
  );
}
```

## Conditional Mutation

```typescript
function EditableField({ id, field }: { id: string; field: string }) {
  const [isEditing, setIsEditing] = useState(false);
  const { data } = useSWR(`/api/items/${id}`, fetcher);

  const { trigger, isMutating } = useSWRMutation(
    isEditing ? `/api/items/${id}` : null, // Only create mutation when editing
    updateItem
  );

  const handleSave = async (value: string) => {
    await trigger({ [field]: value });
    setIsEditing(false);
  };
}
```

## Mutation with Dependent Data

```typescript
function OrderForm({ cartId }: { cartId: string }) {
  const { data: cart } = useSWR(`/api/carts/${cartId}`, fetcher);

  const { trigger: createOrder } = useSWRMutation(
    '/api/orders',
    async (url, { arg }: { arg: OrderData }) => {
      const response = await fetch(url, {
        method: 'POST',
        body: JSON.stringify(arg),
      });
      return response.json();
    }
  );

  const handleCheckout = async () => {
    if (!cart) return;

    const order = await createOrder({
      items: cart.items,
      total: cart.total,
    });

    // Clear cart after successful order
    mutate(`/api/carts/${cartId}`, null);

    return order;
  };
}
```

## Mutation Options Reference

| Option | Type | Description |
|--------|------|-------------|
| `optimisticData` | `Data` | Data to display before mutation completes |
| `revalidate` | `boolean` | Revalidate after mutation (default: true) |
| `rollbackOnError` | `boolean` | Rollback optimistic data on error |
| `populateCache` | `boolean/function` | Whether to update cache with result |
| `throwOnError` | `boolean` | Throw error instead of returning it |

