# React 19 Reference

## New Hooks

### useActionState
```tsx
const [state, action, isPending] = useActionState(
  async (prevState, formData) => {
    const result = await submitForm(formData);
    return result;
  },
  initialState
);

<form action={action}>
  <input name="email" />
  <button disabled={isPending}>Submit</button>
  {state.error && <p>{state.error}</p>}
</form>
```

### useFormStatus
```tsx
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Submitting...' : 'Submit'}</button>;
}
```

### useOptimistic
```tsx
const [optimisticItems, addOptimistic] = useOptimistic(
  items,
  (state, newItem) => [...state, { ...newItem, pending: true }]
);

async function addItem(formData) {
  addOptimistic({ id: Date.now(), name: formData.get('name') });
  await saveItem(formData);
}
```

### use() Hook
```tsx
// Read promises
function UserProfile({ userPromise }) {
  const user = use(userPromise); // Suspends until resolved
  return <div>{user.name}</div>;
}

// Conditional context
function ThemeButton() {
  if (shouldUseTheme) {
    const theme = use(ThemeContext);
    return <button style={{ color: theme.primary }}>Click</button>;
  }
  return <button>Click</button>;
}
```

## Actions

### Form Actions
```tsx
async function createUser(formData: FormData) {
  'use server';
  const name = formData.get('name');
  await db.users.create({ name });
  revalidatePath('/users');
}

<form action={createUser}>
  <input name="name" required />
  <button type="submit">Create</button>
</form>
```

### startTransition with async
```tsx
const [isPending, startTransition] = useTransition();

startTransition(async () => {
  await updateDatabase(data);
  setItems(newItems);
});
```

## Server Components

```tsx
// Server Component (default in App Router)
async function UserList() {
  const users = await db.users.findMany();
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

// Client Component
'use client';
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

## ref as Prop

```tsx
// React 19: ref is a regular prop
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}

// Usage
<Input ref={inputRef} placeholder="Enter text" />
```

## Document Metadata

```tsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={post.url} />
      <h1>{post.title}</h1>
    </article>
  );
}
```

## Resource Preloading

```tsx
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom';

function App() {
  prefetchDNS('https://api.example.com');
  preconnect('https://api.example.com');
  preload('/styles/main.css', { as: 'style' });
  preinit('/scripts/analytics.js', { as: 'script' });

  return <div>...</div>;
}
```

## React Compiler (Experimental)

```js
// babel.config.js
module.exports = {
  plugins: ['babel-plugin-react-compiler'],
};
```

Automatic memoization - no more manual useMemo/useCallback needed.

## Key Changes

| Feature | React 18 | React 19 |
|---------|----------|----------|
| Form handling | Manual state | useActionState |
| Optimistic UI | Manual | useOptimistic |
| Promise reading | useEffect | use() |
| forwardRef | Required | ref as prop |
| Context | useContext only | use(Context) |
