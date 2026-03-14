# React 18 → Hooks Delta

## Not Available in React 18

- `use()` - Promise/context reading (React 19+)
- `useOptimistic()` - Optimistic updates (React 19+)
- `useFormStatus()` - Form status in children (React 19+)
- `useActionState()` - Form action state (React 19+)

## Syntax Differences

### Reading Promises

```jsx
// React 19
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

// React 18 - Use Suspense + external library
import { useSuspenseQuery } from '@tanstack/react-query';

function Comments() {
  const { data: comments } = useSuspenseQuery({
    queryKey: ['comments'],
    queryFn: fetchComments
  });
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}
```

### Reading Context Conditionally

```jsx
// React 19
function Button({ theme }) {
  if (theme) {
    const context = use(ThemeContext);
    return <button style={{ color: context.color }}>Click</button>;
  }
  return <button>Click</button>;
}

// React 18 - useContext must be at top level
function Button({ theme }) {
  const context = useContext(ThemeContext); // Always called
  if (theme) {
    return <button style={{ color: context.color }}>Click</button>;
  }
  return <button>Click</button>;
}
```

### Optimistic Updates

```jsx
// React 19
function LikeButton({ postId }) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (state, newLike) => [...state, newLike]
  );

  async function handleLike() {
    addOptimisticLike({ id: 'temp', postId });
    await likePost(postId);
  }
}

// React 18 - Manual state management
function LikeButton({ postId }) {
  const [likes, setLikes] = useState([]);
  const [pending, setPending] = useState(false);

  async function handleLike() {
    const tempLike = { id: 'temp', postId };
    setLikes(prev => [...prev, tempLike]);
    setPending(true);

    try {
      const result = await likePost(postId);
      setLikes(prev => prev.map(l =>
        l.id === 'temp' ? result : l
      ));
    } catch {
      setLikes(prev => prev.filter(l => l.id !== 'temp'));
    } finally {
      setPending(false);
    }
  }
}
```

### Form Status

```jsx
// React 19
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}

// React 18 - Pass state from parent
function SubmitButton({ pending }) {
  return <button disabled={pending}>Submit</button>;
}

function Form() {
  const [pending, setPending] = useState(false);

  async function handleSubmit(e) {
    e.preventDefault();
    setPending(true);
    await submitForm();
    setPending(false);
  }

  return (
    <form onSubmit={handleSubmit}>
      <SubmitButton pending={pending} />
    </form>
  );
}
```

## Still Current in React 18

- `useState`, `useEffect`, `useContext` (core hooks)
- `useReducer`, `useCallback`, `useMemo` (performance hooks)
- `useRef`, `useImperativeHandle` (ref hooks)
- `useLayoutEffect`, `useDebugValue` (utility hooks)
- `useSyncExternalStore` (external state)
- `useTransition`, `useDeferredValue` (concurrent features)
- `useId` (unique IDs for accessibility)

## Recommendations for React 18 Users

1. **Data fetching**: Use React Query, SWR, or similar for Suspense support
2. **Optimistic updates**: Implement manually with useState
3. **Form handling**: Use react-hook-form or manual state management
4. **Context**: Always call useContext at component top level

## Migration Path

React 18 → 19 is mostly additive. The new hooks simplify patterns that
required external libraries or manual state management in React 18.
