# Zustand 4 → Basics Delta

## Not Available in Zustand 4

- `useShallow` built-in helper
- Improved TypeScript inference
- Some deprecated APIs

## Syntax Differences

### Shallow Comparison

```typescript
// Zustand 5 - Built-in useShallow
import { useShallow } from 'zustand/react/shallow';

const { bears, fish } = useStore(
  useShallow((state) => ({ bears: state.bears, fish: state.fish }))
);


// Zustand 4 - Import from shallow
import shallow from 'zustand/shallow';

const { bears, fish } = useStore(
  (state) => ({ bears: state.bears, fish: state.fish }),
  shallow
);
```

### Store Creation

```typescript
// Zustand 5 - Same API, better types
import { create } from 'zustand';

interface BearState {
  bears: number;
  increase: () => void;
}

const useStore = create<BearState>()((set) => ({
  bears: 0,
  increase: () => set((state) => ({ bears: state.bears + 1 })),
}));


// Zustand 4 - Same pattern
import create from 'zustand';  // Default export in v4

interface BearState {
  bears: number;
  increase: () => void;
}

const useStore = create<BearState>((set) => ({
  bears: 0,
  increase: () => set((state) => ({ bears: state.bears + 1 })),
}));
```

### Import Changes

```typescript
// Zustand 5 - Named export
import { create } from 'zustand';
import { createWithEqualityFn } from 'zustand/traditional';  // If needed

// Zustand 4 - Default export
import create from 'zustand';
import createVanilla from 'zustand/vanilla';
```

### createWithEqualityFn (Removed in v5)

```typescript
// Zustand 4 - createWithEqualityFn available
import { createWithEqualityFn } from 'zustand/traditional';

const useStore = createWithEqualityFn<State>()(
  (set) => ({...}),
  Object.is  // Custom equality function
);


// Zustand 5 - Use create with useShallow instead
import { create } from 'zustand';
import { useShallow } from 'zustand/react/shallow';

const useStore = create<State>()((set) => ({...}));

// In component, use useShallow for shallow comparison
const state = useStore(useShallow((s) => ({ a: s.a, b: s.b })));
```

### Middleware

```typescript
// Both Zustand 4 and 5 - Middleware works the same
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useStore = create<State>()(
  devtools(
    persist(
      (set) => ({
        bears: 0,
        increase: () => set((state) => ({ bears: state.bears + 1 })),
      }),
      { name: 'bear-storage' }
    )
  )
);
```

## Still Current in Zustand 4

- Basic store creation
- set() and get() functions
- Subscriptions
- All middleware (devtools, persist, immer, etc.)
- Vanilla stores
- React integration

## Selector Optimization

```typescript
// Both versions - Selector to avoid re-renders
const bears = useStore((state) => state.bears);

// Both versions - Multiple values (causes re-render on any change)
const { bears, fish } = useStore((state) => ({ bears: state.bears, fish: state.fish }));

// Zustand 5 - Shallow comparison
import { useShallow } from 'zustand/react/shallow';
const { bears, fish } = useStore(useShallow((state) => ({ bears: state.bears, fish: state.fish })));

// Zustand 4 - Shallow comparison
import shallow from 'zustand/shallow';
const { bears, fish } = useStore(
  (state) => ({ bears: state.bears, fish: state.fish }),
  shallow
);
```

## Recommendations for Zustand 4 Users

1. **Use selectors** to minimize re-renders
2. **Import shallow** for object selections
3. **Named exports** when upgrading to v5
4. **Replace createWithEqualityFn** with create + useShallow

## Migration Path

1. Update import: `import { create } from 'zustand'`
2. Replace shallow import: `useShallow` from `zustand/react/shallow`
3. Remove createWithEqualityFn usage
4. Test selector behavior

