# Svelte 4 → Runes Delta

## Not Available in Svelte 4

- `$state()` - Use `let` declarations instead
- `$derived()` - Use `$:` reactive declarations
- `$effect()` - Use `$:` for side effects
- `$props()` - Use `export let` for props
- `$bindable()` - Use `export let` with bind:
- `$inspect()` - Use console.log or Svelte DevTools
- `$host()` - Not available

## Syntax Differences

### Reactive State

```svelte
<!-- Svelte 5 -->
<script>
  let count = $state(0);
</script>

<!-- Svelte 4 -->
<script>
  let count = 0;  // Automatically reactive at top level
</script>
```

### Computed Values

```svelte
<!-- Svelte 5 -->
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>

<!-- Svelte 4 -->
<script>
  let count = 0;
  $: doubled = count * 2;
</script>
```

### Side Effects

```svelte
<!-- Svelte 5 -->
<script>
  let count = $state(0);
  $effect(() => {
    console.log(count);
    return () => cleanup();
  });
</script>

<!-- Svelte 4 -->
<script>
  import { onDestroy } from 'svelte';

  let count = 0;
  $: console.log(count);

  // Cleanup requires onDestroy
  onDestroy(() => cleanup());
</script>
```

### Props

```svelte
<!-- Svelte 5 -->
<script>
  let { name, age = 0 } = $props();
</script>

<!-- Svelte 4 -->
<script>
  export let name;
  export let age = 0;
</script>
```

### Bindable Props

```svelte
<!-- Svelte 5 -->
<script>
  let { value = $bindable() } = $props();
</script>

<!-- Svelte 4 -->
<script>
  export let value;  // Automatically bindable
</script>
```

## Still Current in Svelte 4

- `$:` reactive declarations (replaced by runes in v5)
- `export let` for props (replaced by `$props()` in v5)
- Slots (replaced by snippets in v5, but still supported)
- Stores with `$` prefix auto-subscription
- All lifecycle functions (onMount, onDestroy, etc.)

## Recommendations for Svelte 4 Users

1. **Reactive declarations** with `$:` work well for most cases
2. **export let** is the standard way to declare props
3. **Stores** are still the best way to share state between components
4. Consider upgrading to Svelte 5 for new projects
