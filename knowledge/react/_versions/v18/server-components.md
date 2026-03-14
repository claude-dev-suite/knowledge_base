# React 18 → Server Components Delta

## Not Available in React 18 (without Next.js 13+)

- Native React Server Components - Requires framework support
- `'use server'` directive - Server Actions (React 19+)
- `useActionState()` - Form action state (React 19+)
- `useOptimistic()` - Optimistic updates (React 19+)
- `useFormStatus()` - Form pending state (React 19+)

## Framework Requirements

React 18 requires **Next.js 13+ App Router** or similar framework for RSC support.
React 19 includes RSC primitives natively.

## Syntax Differences

### Server Actions

```tsx
// React 19 - Native Server Actions
async function submitForm(formData: FormData) {
  'use server';
  await db.user.create({ data: { name: formData.get('name') } });
}

function Form() {
  return (
    <form action={submitForm}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  );
}

// React 18 - API Route Pattern
'use client';
function Form() {
  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const formData = new FormData(e.target as HTMLFormElement);
    await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify({ name: formData.get('name') }),
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Form State Handling

```tsx
// React 19 - useActionState
'use client';
import { useActionState } from 'react';
import { submitAction } from './actions';

function Form() {
  const [state, formAction, isPending] = useActionState(submitAction, null);

  return (
    <form action={formAction}>
      <input name="email" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}

// React 18 - Manual state management
'use client';
import { useState } from 'react';

function Form() {
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setIsPending(true);
    setError(null);

    try {
      const formData = new FormData(e.target as HTMLFormElement);
      const res = await fetch('/api/submit', {
        method: 'POST',
        body: formData,
      });
      if (!res.ok) throw new Error('Failed');
    } catch (err) {
      setError('Submission failed');
    } finally {
      setIsPending(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

### Optimistic Updates

```tsx
// React 19 - useOptimistic
'use client';
import { useOptimistic } from 'react';
import { addItem } from './actions';

function ItemList({ items }: { items: Item[] }) {
  const [optimisticItems, addOptimisticItem] = useOptimistic(
    items,
    (state, newItem: Item) => [...state, newItem]
  );

  async function handleAdd(formData: FormData) {
    const newItem = { id: 'temp', name: formData.get('name') as string };
    addOptimisticItem(newItem);
    await addItem(formData);
  }

  return (
    <>
      <ul>
        {optimisticItems.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
      <form action={handleAdd}>
        <input name="name" />
        <button>Add</button>
      </form>
    </>
  );
}

// React 18 - Manual optimistic state
'use client';
import { useState } from 'react';

function ItemList({ initialItems }: { initialItems: Item[] }) {
  const [items, setItems] = useState(initialItems);
  const [pendingItems, setPendingItems] = useState<Item[]>([]);

  async function handleAdd(e: React.FormEvent) {
    e.preventDefault();
    const formData = new FormData(e.target as HTMLFormElement);
    const tempItem = { id: `temp-${Date.now()}`, name: formData.get('name') as string };

    // Optimistic add
    setPendingItems(prev => [...prev, tempItem]);

    try {
      const res = await fetch('/api/items', {
        method: 'POST',
        body: JSON.stringify({ name: tempItem.name }),
      });
      const newItem = await res.json();
      setItems(prev => [...prev, newItem]);
    } catch {
      // Rollback on error
    } finally {
      setPendingItems(prev => prev.filter(i => i.id !== tempItem.id));
    }
  }

  const allItems = [...items, ...pendingItems];

  return (
    <>
      <ul>
        {allItems.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
      <form onSubmit={handleAdd}>
        <input name="name" />
        <button>Add</button>
      </form>
    </>
  );
}
```

## Still Current in React 18 (with Next.js 13+)

- Server Components (via Next.js App Router)
- `'use client'` directive
- Streaming with Suspense
- Async Server Components
- Data fetching patterns (fetch in Server Components)

## Recommendations for React 18 Users

1. **Use Next.js 13+ App Router** for RSC support
2. **API Routes** for mutations instead of Server Actions
3. **Manual state management** for form handling and optimistic updates
4. **TanStack Query** for client-side data fetching with optimistic updates
5. **Consider upgrading to React 19** for native Server Actions support

## Migration Path

When upgrading from React 18 to React 19:
1. Replace API routes with Server Actions
2. Replace manual form state with `useActionState`
3. Replace manual optimistic state with `useOptimistic`
4. Replace `useFormStatus` prop drilling with the hook

