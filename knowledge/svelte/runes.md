# Svelte 5 Runes

> Official Documentation: https://svelte.dev/docs/svelte/what-are-runes

Runes are Svelte 5's new reactivity primitives that replace the `$:` reactive declarations from Svelte 4.

## $state - Reactive State

Creates reactive state that triggers updates when changed.

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'John', age: 30 });

  function increment() {
    count++;  // Triggers reactivity
  }

  function birthday() {
    user.age++;  // Deep reactivity works
  }
</script>

<button onclick={increment}>Count: {count}</button>
<p>{user.name} is {user.age} years old</p>
```

### $state.raw - Non-deep Reactivity

For objects where you don't need deep reactivity:

```svelte
<script>
  let data = $state.raw({ items: [] });

  // Must reassign entire object to trigger update
  data = { items: [...data.items, newItem] };
</script>
```

### $state.snapshot - Get Plain Object

```svelte
<script>
  let user = $state({ name: 'John' });

  function logUser() {
    // Get plain object (no proxy)
    console.log($state.snapshot(user));
  }
</script>
```

---

## $derived - Computed Values

Creates a value that automatically updates when its dependencies change.

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
  let quadrupled = $derived(doubled * 2);

  // Complex derivations
  let items = $state([1, 2, 3]);
  let total = $derived(items.reduce((sum, n) => sum + n, 0));
</script>

<p>Count: {count}, Doubled: {doubled}, Quadrupled: {quadrupled}</p>
```

### $derived.by - Complex Derivations

For derivations that need more than an expression:

```svelte
<script>
  let items = $state([]);

  let stats = $derived.by(() => {
    if (items.length === 0) return { avg: 0, max: 0 };

    const sum = items.reduce((a, b) => a + b, 0);
    return {
      avg: sum / items.length,
      max: Math.max(...items)
    };
  });
</script>
```

---

## $effect - Side Effects

Runs code when dependencies change. Replaces `$:` for side effects.

```svelte
<script>
  let count = $state(0);

  $effect(() => {
    console.log(`Count changed to ${count}`);

    // Cleanup function (optional)
    return () => {
      console.log('Cleaning up...');
    };
  });
</script>
```

### $effect.pre - Run Before DOM Update

```svelte
<script>
  let div;
  let messages = $state([]);

  $effect.pre(() => {
    // Runs before DOM updates
    // Useful for scroll position preservation
    if (div) {
      const shouldScroll = div.scrollTop + div.clientHeight >= div.scrollHeight;
      if (shouldScroll) {
        // Will scroll after update
      }
    }
  });
</script>
```

### $effect.tracking - Check if Tracking

```svelte
<script>
  $effect(() => {
    console.log('Inside effect:', $effect.tracking()); // true
  });

  console.log('Outside effect:', $effect.tracking()); // false
</script>
```

### $effect.root - Untracked Effect

```svelte
<script>
  const cleanup = $effect.root(() => {
    $effect(() => {
      // This effect runs outside component lifecycle
    });

    return () => {
      // Manual cleanup
    };
  });

  onDestroy(cleanup);
</script>
```

---

## $props - Component Props

Declares component props. Replaces `export let`.

```svelte
<script>
  // Basic props
  let { name, age = 0 } = $props();

  // With TypeScript
  interface Props {
    name: string;
    age?: number;
    onUpdate?: (value: number) => void;
  }

  let { name, age = 0, onUpdate }: Props = $props();
</script>
```

### $bindable - Two-way Binding Props

```svelte
<!-- Child.svelte -->
<script>
  let { value = $bindable() } = $props();
</script>

<input bind:value />

<!-- Parent.svelte -->
<script>
  let text = $state('');
</script>

<Child bind:value={text} />
```

### Rest Props

```svelte
<script>
  let { class: className, ...rest } = $props();
</script>

<div class={className} {...rest}>
  <slot />
</div>
```

---

## $inspect - Debug Reactivity

Development-only tool for debugging reactive values.

```svelte
<script>
  let count = $state(0);
  let user = $state({ name: 'John' });

  // Logs when values change
  $inspect(count);
  $inspect(user);

  // Custom logging
  $inspect(count).with(console.trace);
</script>
```

---

## $host - Custom Element Host

Access the host element in custom elements.

```svelte
<svelte:options customElement="my-element" />

<script>
  $host().dispatchEvent(new CustomEvent('greeting'));
</script>
```

---

## Migration from Svelte 4

| Svelte 4 | Svelte 5 |
|----------|----------|
| `let count = 0` | `let count = $state(0)` |
| `$: doubled = count * 2` | `let doubled = $derived(count * 2)` |
| `$: console.log(count)` | `$effect(() => console.log(count))` |
| `export let name` | `let { name } = $props()` |
| `<slot />` | `{@render children()}` with snippets |

---

## Best Practices

1. **Use $state for mutable data** - Not for constants
2. **Prefer $derived over $effect** - For computed values
3. **Keep effects focused** - One responsibility per effect
4. **Use TypeScript** - Better inference with props
5. **Don't overuse $state.raw** - Deep reactivity is usually fine
