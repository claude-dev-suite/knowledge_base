# Vue 3 Composition API

> Official Documentation: https://vuejs.org/guide/extras/composition-api-faq.html

The Composition API is a set of APIs that allows you to author Vue components using imported functions instead of declaring options. It provides better logic reuse, more flexible code organization, and improved TypeScript support.

---

## Table of Contents

1. [Setup Function and Script Setup](#setup-function-and-script-setup)
2. [ref() and reactive()](#ref-and-reactive)
3. [computed()](#computed)
4. [watch() and watchEffect()](#watch-and-watcheffect)
5. [Lifecycle Hooks](#lifecycle-hooks)
6. [provide() and inject()](#provide-and-inject)
7. [Template Refs](#template-refs)
8. [Composables](#composables)
9. [toRef(), toRefs(), unref()](#toref-torefs-unref)
10. [Props and Emits](#props-and-emits)
11. [TypeScript Integration](#typescript-integration)
12. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Setup Function and Script Setup

### Traditional setup() Function

The `setup()` function is the entry point for using the Composition API in a component:

```typescript
import { ref, reactive, computed } from 'vue';

export default {
  props: {
    title: String,
  },
  setup(props, context) {
    // props - reactive props object
    // context.attrs - non-prop attributes
    // context.slots - slots
    // context.emit - emit function
    // context.expose - expose public properties

    const count = ref(0);

    function increment() {
      count.value++;
    }

    // Return values exposed to template
    return {
      count,
      increment,
    };
  },
};
```

### Script Setup Syntax (Recommended)

`<script setup>` is a compile-time syntactic sugar that is more concise and performant:

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';

// Top-level bindings are automatically exposed to template
const count = ref(0);
const message = ref('Hello');

// Functions are automatically available
function increment() {
  count.value++;
}

// Computed properties work the same way
const doubleCount = computed(() => count.value * 2);

// Lifecycle hooks
onMounted(() => {
  console.log('Component is mounted');
});
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <p>Count: {{ count }} (Double: {{ doubleCount }})</p>
    <button @click="increment">Increment</button>
  </div>
</template>
```

### Exposing Public Properties

By default, `<script setup>` components are closed. Use `defineExpose` to expose properties:

```vue
<script setup lang="ts">
import { ref } from 'vue';

const count = ref(0);
const publicMethod = () => console.log('Called from parent');

// Explicitly expose to parent components
defineExpose({
  count,
  publicMethod,
});
</script>
```

---

## ref() and reactive()

### ref() - For Primitives and Single Values

`ref()` creates a reactive reference that wraps any value:

```typescript
import { ref } from 'vue';

// Primitive values
const count = ref(0);
const name = ref('John');
const isActive = ref(true);

// Access and modify with .value
console.log(count.value); // 0
count.value++;
console.log(count.value); // 1

// ref can also hold objects (they become deeply reactive)
const user = ref({ name: 'John', age: 30 });
user.value.age = 31; // Reactive update

// ref with arrays
const items = ref<string[]>([]);
items.value.push('item'); // Reactive
items.value = ['new', 'array']; // Also reactive
```

### reactive() - For Objects and Collections

`reactive()` creates a deeply reactive proxy of an object:

```typescript
import { reactive } from 'vue';

// Object state
const state = reactive({
  count: 0,
  user: {
    name: 'John',
    profile: {
      email: 'john@example.com',
    },
  },
  items: [] as string[],
});

// Direct property access (no .value needed)
console.log(state.count); // 0
state.count++;
state.user.name = 'Jane';
state.items.push('item');

// Nested objects are also reactive
state.user.profile.email = 'jane@example.com';
```

### Key Differences Between ref() and reactive()

| Feature | ref() | reactive() |
|---------|-------|------------|
| Value types | Any (primitives, objects) | Objects only |
| Access | `.value` required in JS | Direct access |
| Reassignment | Can reassign entire value | Cannot reassign (loses reactivity) |
| Destructuring | Maintains reactivity | Loses reactivity |
| Template usage | Auto-unwrapped | Direct access |

```typescript
import { ref, reactive } from 'vue';

// ref can be reassigned
const data = ref({ count: 0 });
data.value = { count: 10 }; // Works fine

// reactive cannot be reassigned
let state = reactive({ count: 0 });
state = reactive({ count: 10 }); // Loses reactivity binding!

// Destructuring differences
const refObj = ref({ a: 1, b: 2 });
const { a, b } = refObj.value; // a and b are NOT reactive

const reactiveObj = reactive({ a: 1, b: 2 });
const { a: ra, b: rb } = reactiveObj; // ra and rb are NOT reactive
```

### shallowRef() and shallowReactive()

For performance optimization when deep reactivity is not needed:

```typescript
import { shallowRef, shallowReactive, triggerRef } from 'vue';

// Only .value assignment is reactive
const shallow = shallowRef({ nested: { count: 0 } });
shallow.value.nested.count++; // NOT reactive
shallow.value = { nested: { count: 1 } }; // Reactive

// Manually trigger updates
triggerRef(shallow);

// Only root-level properties are reactive
const shallowState = shallowReactive({
  count: 0,
  nested: { value: 1 },
});
shallowState.count++; // Reactive
shallowState.nested.value++; // NOT reactive
```

---

## computed()

### Readonly Computed (Getter Only)

```typescript
import { ref, computed } from 'vue';

const firstName = ref('John');
const lastName = ref('Doe');

// Computed property with getter
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`;
});

console.log(fullName.value); // "John Doe"
// fullName.value = 'Jane Doe'; // Error: computed is readonly
```

### Writable Computed (Getter and Setter)

```typescript
import { ref, computed } from 'vue';

const firstName = ref('John');
const lastName = ref('Doe');

const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`;
  },
  set(newValue: string) {
    const parts = newValue.split(' ');
    firstName.value = parts[0] || '';
    lastName.value = parts[1] || '';
  },
});

console.log(fullName.value); // "John Doe"
fullName.value = 'Jane Smith';
console.log(firstName.value); // "Jane"
console.log(lastName.value); // "Smith"
```

### Computed with TypeScript

```typescript
import { ref, computed, ComputedRef } from 'vue';

interface User {
  firstName: string;
  lastName: string;
}

const user = ref<User>({ firstName: 'John', lastName: 'Doe' });

// Explicit return type
const displayName: ComputedRef<string> = computed(() => {
  return `${user.value.firstName} ${user.value.lastName}`;
});

// Computed with complex types
const userInfo = computed<{ name: string; initials: string }>(() => ({
  name: `${user.value.firstName} ${user.value.lastName}`,
  initials: `${user.value.firstName[0]}${user.value.lastName[0]}`,
}));
```

### Computed Best Practices

```typescript
import { ref, computed } from 'vue';

const items = ref([1, 2, 3, 4, 5]);
const multiplier = ref(2);

// GOOD: Pure computation, no side effects
const doubled = computed(() => items.value.map((n) => n * multiplier.value));

// BAD: Side effects in computed
const badComputed = computed(() => {
  console.log('Computing...'); // Side effect!
  return items.value.length;
});

// GOOD: Expensive computations are cached
const expensiveResult = computed(() => {
  // Only re-runs when dependencies change
  return items.value.reduce((sum, n) => sum + Math.pow(n, 3), 0);
});
```

---

## watch() and watchEffect()

### watch() - Explicit Dependency Tracking

```typescript
import { ref, reactive, watch } from 'vue';

const count = ref(0);
const user = reactive({ name: 'John', age: 30 });

// Watch a single ref
watch(count, (newValue, oldValue) => {
  console.log(`Count: ${oldValue} -> ${newValue}`);
});

// Watch a getter function
watch(
  () => user.name,
  (newName, oldName) => {
    console.log(`Name: ${oldName} -> ${newName}`);
  }
);

// Watch multiple sources
watch([count, () => user.name], ([newCount, newName], [oldCount, oldName]) => {
  console.log(`Count: ${oldCount} -> ${newCount}`);
  console.log(`Name: ${oldName} -> ${newName}`);
});

// Watch entire reactive object (requires deep option)
watch(
  () => user,
  (newUser) => {
    console.log('User changed:', newUser);
  },
  { deep: true }
);
```

### watch() Options

```typescript
import { ref, watch } from 'vue';

const searchQuery = ref('');

// immediate: Run callback immediately on creation
watch(
  searchQuery,
  (newQuery) => {
    console.log('Query:', newQuery);
  },
  { immediate: true }
);

// deep: Watch nested properties
const state = ref({ nested: { count: 0 } });
watch(
  state,
  (newState) => {
    console.log('Deep change:', newState.nested.count);
  },
  { deep: true }
);

// flush: Control callback timing
watch(
  searchQuery,
  () => {
    // Access updated DOM
  },
  { flush: 'post' } // 'pre' | 'post' | 'sync'
);

// once: Run only once (Vue 3.4+)
watch(
  searchQuery,
  (newQuery) => {
    console.log('First change:', newQuery);
  },
  { once: true }
);
```

### watchEffect() - Automatic Dependency Tracking

```typescript
import { ref, watchEffect } from 'vue';

const count = ref(0);
const name = ref('John');

// Automatically tracks all reactive dependencies used inside
const stop = watchEffect(() => {
  console.log(`Count: ${count.value}, Name: ${name.value}`);
});

// Runs immediately, then re-runs when count or name changes
count.value++; // Triggers watchEffect
name.value = 'Jane'; // Triggers watchEffect

// Stop watching
stop();
```

### watchEffect() with Cleanup

```typescript
import { ref, watchEffect } from 'vue';

const id = ref(1);

watchEffect((onCleanup) => {
  const controller = new AbortController();

  fetch(`/api/user/${id.value}`, { signal: controller.signal })
    .then((res) => res.json())
    .then((data) => console.log(data));

  // Cleanup function runs before next execution or on unmount
  onCleanup(() => {
    controller.abort();
  });
});
```

### watchPostEffect() and watchSyncEffect()

```typescript
import { ref, watchPostEffect, watchSyncEffect } from 'vue';

const count = ref(0);

// Equivalent to watch with { flush: 'post' }
watchPostEffect(() => {
  // Runs after DOM updates
  console.log('DOM updated, count:', count.value);
});

// Equivalent to watch with { flush: 'sync' }
watchSyncEffect(() => {
  // Runs synchronously (use sparingly)
  console.log('Sync update, count:', count.value);
});
```

---

## Lifecycle Hooks

### Available Lifecycle Hooks

```typescript
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onActivated,
  onDeactivated,
  onErrorCaptured,
  onRenderTracked,
  onRenderTriggered,
} from 'vue';

// Before component DOM is mounted
onBeforeMount(() => {
  console.log('Before mount - DOM not yet available');
});

// After component DOM is mounted
onMounted(() => {
  console.log('Mounted - DOM is available');
  // Safe to access DOM elements, start timers, fetch data
});

// Before component re-renders due to reactive state change
onBeforeUpdate(() => {
  console.log('Before update - about to re-render');
});

// After component has re-rendered
onUpdated(() => {
  console.log('Updated - DOM has been updated');
  // Be careful: don't mutate state here (infinite loop risk)
});

// Before component is unmounted
onBeforeUnmount(() => {
  console.log('Before unmount - still functional');
  // Good place to start cleanup
});

// After component is unmounted
onUnmounted(() => {
  console.log('Unmounted - fully destroyed');
  // Clean up: remove listeners, cancel timers, close connections
});
```

### Keep-Alive Specific Hooks

```typescript
import { onActivated, onDeactivated } from 'vue';

// Called when component is activated from keep-alive cache
onActivated(() => {
  console.log('Component activated');
  // Resume activities, refresh data
});

// Called when component is deactivated into keep-alive cache
onDeactivated(() => {
  console.log('Component deactivated');
  // Pause activities, save state
});
```

### Error Handling and Debug Hooks

```typescript
import { onErrorCaptured, onRenderTracked, onRenderTriggered } from 'vue';

// Capture errors from descendant components
onErrorCaptured((error, instance, info) => {
  console.error('Error captured:', error);
  console.log('Component:', instance);
  console.log('Info:', info);
  return false; // Prevent error from propagating
});

// Debug: track reactive dependencies (dev only)
onRenderTracked((event) => {
  console.log('Dependency tracked:', event);
});

// Debug: identify what triggered re-render (dev only)
onRenderTriggered((event) => {
  console.log('Re-render triggered by:', event);
});
```

### Lifecycle Hook Patterns

```typescript
import { ref, onMounted, onUnmounted } from 'vue';

// Pattern: Setup and cleanup
export function useEventListener(
  target: EventTarget,
  event: string,
  callback: EventListener
) {
  onMounted(() => target.addEventListener(event, callback));
  onUnmounted(() => target.removeEventListener(event, callback));
}

// Pattern: Async data fetching
const data = ref(null);
const loading = ref(true);
const error = ref(null);

onMounted(async () => {
  try {
    const response = await fetch('/api/data');
    data.value = await response.json();
  } catch (e) {
    error.value = e;
  } finally {
    loading.value = false;
  }
});
```

---

## provide() and inject()

### Basic Usage

```typescript
// Parent component
import { provide, ref } from 'vue';

const theme = ref('dark');
const user = ref({ name: 'John', role: 'admin' });

provide('theme', theme);
provide('user', user);
```

```typescript
// Child/Descendant component (any depth)
import { inject } from 'vue';

// With default value
const theme = inject('theme', 'light');

// Without default (may be undefined)
const user = inject('user');
```

### Type-Safe Provide/Inject with Symbols

```typescript
// keys.ts - Define injection keys
import type { InjectionKey, Ref } from 'vue';

export interface User {
  name: string;
  role: string;
}

export const ThemeKey: InjectionKey<Ref<string>> = Symbol('theme');
export const UserKey: InjectionKey<Ref<User>> = Symbol('user');
```

```typescript
// Parent component
import { provide, ref } from 'vue';
import { ThemeKey, UserKey } from './keys';

const theme = ref('dark');
const user = ref({ name: 'John', role: 'admin' });

provide(ThemeKey, theme);
provide(UserKey, user);
```

```typescript
// Child component
import { inject } from 'vue';
import { ThemeKey, UserKey } from './keys';

// Fully typed!
const theme = inject(ThemeKey); // Ref<string> | undefined
const user = inject(UserKey); // Ref<User> | undefined

// With required assertion
const requiredTheme = inject(ThemeKey)!;
```

### Readonly Provide Pattern

```typescript
import { provide, inject, readonly, ref } from 'vue';

// Parent - provide readonly to prevent mutations from children
const count = ref(0);
const increment = () => count.value++;

provide('count', readonly(count));
provide('increment', increment);

// Child
const count = inject<Readonly<Ref<number>>>('count');
const increment = inject<() => void>('increment');
```

### App-Level Provide

```typescript
// main.ts
import { createApp } from 'vue';
import App from './App.vue';

const app = createApp(App);

// Available to all components
app.provide('globalConfig', {
  apiUrl: 'https://api.example.com',
  version: '1.0.0',
});

app.mount('#app');
```

---

## Template Refs

### Basic Template Refs

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';

// The ref name must match template ref attribute
const inputRef = ref<HTMLInputElement | null>(null);
const divRef = ref<HTMLDivElement | null>(null);

onMounted(() => {
  // Access DOM element after mount
  inputRef.value?.focus();
  console.log(divRef.value?.offsetHeight);
});
</script>

<template>
  <input ref="inputRef" type="text" />
  <div ref="divRef">Content</div>
</template>
```

### Component Refs

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import ChildComponent from './ChildComponent.vue';

// Type the ref with the component's exposed interface
const childRef = ref<InstanceType<typeof ChildComponent> | null>(null);

onMounted(() => {
  // Access exposed methods/properties
  childRef.value?.someMethod();
  console.log(childRef.value?.someProperty);
});
</script>

<template>
  <ChildComponent ref="childRef" />
</template>
```

### Refs in v-for

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';

const items = ref(['a', 'b', 'c']);
const itemRefs = ref<HTMLLIElement[]>([]);

onMounted(() => {
  console.log(itemRefs.value); // Array of li elements
});
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item" ref="itemRefs">
      {{ item }}
    </li>
  </ul>
</template>
```

### Function Refs

```vue
<script setup lang="ts">
import { ref } from 'vue';

const dynamicRefs = ref<Map<string, HTMLElement>>(new Map());

const setItemRef = (el: HTMLElement | null, key: string) => {
  if (el) {
    dynamicRefs.value.set(key, el);
  } else {
    dynamicRefs.value.delete(key);
  }
};
</script>

<template>
  <div v-for="item in items" :key="item.id" :ref="(el) => setItemRef(el, item.id)">
    {{ item.name }}
  </div>
</template>
```

---

## Composables

Composables are functions that leverage Vue's Composition API to encapsulate and reuse stateful logic.

### Basic Composable Pattern

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue';

export function useCounter(initialValue = 0) {
  const count = ref(initialValue);

  const doubled = computed(() => count.value * 2);

  function increment() {
    count.value++;
  }

  function decrement() {
    count.value--;
  }

  function reset() {
    count.value = initialValue;
  }

  return {
    count,
    doubled,
    increment,
    decrement,
    reset,
  };
}
```

### Composable with Lifecycle Hooks

```typescript
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue';

export function useMouse() {
  const x = ref(0);
  const y = ref(0);

  function update(event: MouseEvent) {
    x.value = event.pageX;
    y.value = event.pageY;
  }

  onMounted(() => {
    window.addEventListener('mousemove', update);
  });

  onUnmounted(() => {
    window.removeEventListener('mousemove', update);
  });

  return { x, y };
}
```

### Async Composable Pattern

```typescript
// composables/useFetch.ts
import { ref, watchEffect, toValue, type MaybeRefOrGetter } from 'vue';

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null);
  const error = ref<Error | null>(null);
  const loading = ref(false);

  async function fetchData() {
    loading.value = true;
    error.value = null;

    try {
      const response = await fetch(toValue(url));
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      data.value = await response.json();
    } catch (e) {
      error.value = e as Error;
    } finally {
      loading.value = false;
    }
  }

  watchEffect(() => {
    fetchData();
  });

  return { data, error, loading, refetch: fetchData };
}

// Usage
const { data, loading, error } = useFetch<User[]>('/api/users');
```

### Composable with Options

```typescript
// composables/useStorage.ts
import { ref, watch } from 'vue';

interface UseStorageOptions {
  storage?: Storage;
  serializer?: {
    read: (value: string) => unknown;
    write: (value: unknown) => string;
  };
}

export function useStorage<T>(
  key: string,
  defaultValue: T,
  options: UseStorageOptions = {}
) {
  const { storage = localStorage, serializer = JSON } = options;

  const storedValue = storage.getItem(key);
  const data = ref<T>(
    storedValue ? (serializer.read(storedValue) as T) : defaultValue
  );

  watch(
    data,
    (newValue) => {
      if (newValue === null || newValue === undefined) {
        storage.removeItem(key);
      } else {
        storage.setItem(key, serializer.write(newValue));
      }
    },
    { deep: true }
  );

  return data;
}
```

### Composing Composables

```typescript
// composables/useUserProfile.ts
import { computed } from 'vue';
import { useFetch } from './useFetch';
import { useStorage } from './useStorage';

export function useUserProfile(userId: string) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${userId}`);
  const preferences = useStorage(`user-prefs-${userId}`, { theme: 'light' });

  const displayName = computed(() => {
    if (!user.value) return 'Guest';
    return `${user.value.firstName} ${user.value.lastName}`;
  });

  return {
    user,
    loading,
    error,
    preferences,
    displayName,
  };
}
```

---

## toRef(), toRefs(), unref()

### toRef() - Create Ref from Reactive Property

```typescript
import { reactive, toRef } from 'vue';

const state = reactive({
  count: 0,
  name: 'John',
});

// Create a ref that syncs with source property
const countRef = toRef(state, 'count');

countRef.value++; // Updates state.count
state.count++; // Updates countRef.value

// Useful for passing reactive properties to composables
function useFeature(count: Ref<number>) {
  // ...
}
useFeature(toRef(state, 'count'));
```

### toRefs() - Convert Reactive Object to Refs

```typescript
import { reactive, toRefs } from 'vue';

const state = reactive({
  count: 0,
  name: 'John',
  items: ['a', 'b'],
});

// Convert all properties to refs
const { count, name, items } = toRefs(state);

// Now destructured values maintain reactivity
count.value++; // Updates state.count
name.value = 'Jane'; // Updates state.name

// Common pattern: return from composables
function useUser() {
  const user = reactive({
    name: 'John',
    email: 'john@example.com',
  });

  return {
    ...toRefs(user), // Allows destructuring while keeping reactivity
  };
}

const { name, email } = useUser(); // Both are refs
```

### unref() - Unwrap Refs

```typescript
import { ref, unref } from 'vue';

const count = ref(10);
const plainValue = 20;

// unref returns .value if ref, otherwise returns as-is
console.log(unref(count)); // 10
console.log(unref(plainValue)); // 20

// Useful in functions that accept both refs and plain values
function double(value: number | Ref<number>): number {
  return unref(value) * 2;
}

double(count); // 20
double(plainValue); // 40
```

### isRef(), isReactive(), isProxy()

```typescript
import { ref, reactive, isRef, isReactive, isProxy } from 'vue';

const refValue = ref(0);
const reactiveValue = reactive({ count: 0 });

isRef(refValue); // true
isRef(reactiveValue); // false

isReactive(reactiveValue); // true
isReactive(refValue); // false

isProxy(reactiveValue); // true
isProxy(refValue.value); // false (primitive)
```

---

## Props and Emits

### Props with TypeScript

```vue
<script setup lang="ts">
// Runtime declaration
const props = defineProps({
  title: {
    type: String,
    required: true,
  },
  count: {
    type: Number,
    default: 0,
  },
  items: {
    type: Array as PropType<string[]>,
    default: () => [],
  },
});

// Type-based declaration (recommended)
interface Props {
  title: string;
  count?: number;
  items?: string[];
  user?: {
    name: string;
    email: string;
  };
}

const props = defineProps<Props>();

// With defaults
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],
  user: () => ({ name: 'Guest', email: '' }),
});

// Access props
console.log(props.title);
console.log(props.count);
</script>
```

### Emits with TypeScript

```vue
<script setup lang="ts">
// Runtime declaration
const emit = defineEmits(['update', 'delete', 'submit']);

// Type-based declaration (recommended)
const emit = defineEmits<{
  (e: 'update', value: string): void;
  (e: 'delete', id: number): void;
  (e: 'submit', data: { name: string; email: string }): void;
}>();

// Vue 3.3+ shorthand syntax
const emit = defineEmits<{
  update: [value: string];
  delete: [id: number];
  submit: [data: { name: string; email: string }];
}>();

// Using emit
function handleUpdate() {
  emit('update', 'new value');
}

function handleSubmit() {
  emit('submit', { name: 'John', email: 'john@example.com' });
}
</script>
```

### v-model Support

```vue
<script setup lang="ts">
// Single v-model
const props = defineProps<{
  modelValue: string;
}>();

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void;
}>();

// Multiple v-models
const props = defineProps<{
  modelValue: string;
  title: string;
}>();

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void;
  (e: 'update:title', value: string): void;
}>();

// Helper for two-way binding
import { computed } from 'vue';

const localValue = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value),
});
</script>

<template>
  <input v-model="localValue" />
</template>
```

### defineModel() (Vue 3.4+)

```vue
<script setup lang="ts">
// Simplified v-model handling
const modelValue = defineModel<string>();
const title = defineModel<string>('title');

// With options
const count = defineModel<number>('count', { default: 0 });

// Direct usage - automatically syncs with parent
modelValue.value = 'new value'; // Emits update:modelValue
</script>

<template>
  <input v-model="modelValue" />
</template>
```

---

## TypeScript Integration

### Component Type Annotations

```vue
<script setup lang="ts">
import { ref, reactive, computed, type Ref, type ComputedRef } from 'vue';

// Explicit ref typing
const count: Ref<number> = ref(0);
const user: Ref<User | null> = ref(null);

// Generic type argument (preferred)
const items = ref<string[]>([]);
const data = ref<Record<string, unknown>>({});

// Reactive typing
interface State {
  count: number;
  users: User[];
  config: Config | null;
}

const state: State = reactive({
  count: 0,
  users: [],
  config: null,
});

// Computed typing
const doubleCount: ComputedRef<number> = computed(() => count.value * 2);
</script>
```

### Generic Components

```vue
<script setup lang="ts" generic="T extends { id: string | number }">
import { ref } from 'vue';

const props = defineProps<{
  items: T[];
  selected?: T;
}>();

const emit = defineEmits<{
  (e: 'select', item: T): void;
  (e: 'delete', item: T): void;
}>();

const activeItem = ref<T | null>(null);
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id" @click="emit('select', item)">
      <slot :item="item" />
    </li>
  </ul>
</template>
```

### Typing Composables

```typescript
// composables/useApi.ts
import { ref, type Ref } from 'vue';

interface UseApiReturn<T> {
  data: Ref<T | null>;
  error: Ref<Error | null>;
  loading: Ref<boolean>;
  execute: () => Promise<void>;
}

export function useApi<T>(
  fetcher: () => Promise<T>
): UseApiReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>;
  const error = ref<Error | null>(null);
  const loading = ref(false);

  async function execute() {
    loading.value = true;
    error.value = null;
    try {
      data.value = await fetcher();
    } catch (e) {
      error.value = e as Error;
    } finally {
      loading.value = false;
    }
  }

  return { data, error, loading, execute };
}
```

### Utility Types

```typescript
import type {
  PropType,
  ComponentPublicInstance,
  ExtractPropTypes,
  ExtractPublicPropTypes,
} from 'vue';

// PropType for complex prop types
const props = {
  callback: {
    type: Function as PropType<(value: string) => void>,
    required: true,
  },
  options: {
    type: Object as PropType<{ key: string; value: number }[]>,
    default: () => [],
  },
};

// Extract prop types
type Props = ExtractPropTypes<typeof props>;
type PublicProps = ExtractPublicPropTypes<typeof props>;
```

---

## Best Practices and Common Pitfalls

### Best Practices

```typescript
// 1. Prefer ref() for primitives, reactive() for objects
const count = ref(0); // Good
const state = reactive({ count: 0, name: '' }); // Good

// 2. Use computed for derived state
const fullName = computed(() => `${first.value} ${last.value}`);

// 3. Always clean up side effects
onMounted(() => {
  const interval = setInterval(tick, 1000);
  onUnmounted(() => clearInterval(interval));
});

// 4. Use watchEffect cleanup parameter
watchEffect((onCleanup) => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal });
  onCleanup(() => controller.abort());
});

// 5. Extract reusable logic into composables
// Instead of duplicating logic, create composables

// 6. Type your composable return values
export function useFeature(): {
  value: Ref<number>;
  update: (n: number) => void;
} {
  // ...
}

// 7. Use provide/inject for deep prop drilling
provide(ThemeKey, theme); // Instead of passing through many levels
```

### Common Pitfalls

```typescript
// PITFALL 1: Losing reactivity with destructuring
const state = reactive({ count: 0 });
const { count } = state; // count is NOT reactive!
// FIX: Use toRefs
const { count } = toRefs(state); // count is reactive

// PITFALL 2: Replacing reactive object
let state = reactive({ count: 0 });
state = reactive({ count: 1 }); // Loses reactivity binding!
// FIX: Mutate properties instead
state.count = 1;
// Or use ref for the whole object
const state = ref({ count: 0 });
state.value = { count: 1 }; // Works

// PITFALL 3: Forgetting .value in JS (not template)
const count = ref(0);
console.log(count); // RefImpl object, not 0
console.log(count.value); // 0

// PITFALL 4: Async in setup without await
// BAD
onMounted(async () => {
  data.value = await fetchData(); // Component might unmount before this
});
// BETTER: Handle cancellation
onMounted(async () => {
  const controller = new AbortController();
  onUnmounted(() => controller.abort());
  try {
    data.value = await fetchData({ signal: controller.signal });
  } catch (e) {
    if (!controller.signal.aborted) throw e;
  }
});

// PITFALL 5: Mutating props
const props = defineProps<{ count: number }>();
props.count++; // Error! Props are readonly
// FIX: Emit event to parent
emit('update:count', props.count + 1);

// PITFALL 6: Side effects in computed
const bad = computed(() => {
  apiCall(); // Don't do this!
  return value.value;
});
// FIX: Use watch or watchEffect for side effects

// PITFALL 7: Not handling template ref lifecycle
const el = ref<HTMLElement | null>(null);
el.value.focus(); // Error if called before mount!
// FIX: Check in onMounted or use optional chaining
onMounted(() => el.value?.focus());

// PITFALL 8: Watch not triggering for nested changes
const obj = ref({ nested: { value: 1 } });
watch(obj, () => {}); // Won't trigger on obj.value.nested.value change
// FIX: Use deep option
watch(obj, () => {}, { deep: true });
```

### Performance Tips

```typescript
// 1. Use shallowRef for large objects that replace entirely
const largeList = shallowRef<Item[]>([]);
largeList.value = newList; // Only this triggers reactivity

// 2. Use computed for expensive calculations (automatic caching)
const sorted = computed(() => [...items.value].sort());

// 3. Avoid watching large objects deeply when not needed
watch(() => state.specificProperty, callback); // Better than deep watch

// 4. Use v-once for static content in templates
// <div v-once>{{ staticContent }}</div>

// 5. Lazy initialization for expensive values
const expensive = ref<ExpensiveObject | null>(null);
onMounted(() => {
  expensive.value = createExpensiveObject();
});
```

---

## Quick Reference

| API | Purpose | Usage |
|-----|---------|-------|
| `ref()` | Reactive primitive/object | `const x = ref(0)` |
| `reactive()` | Reactive object | `const obj = reactive({})` |
| `computed()` | Derived state | `computed(() => x.value * 2)` |
| `watch()` | Explicit watchers | `watch(source, callback)` |
| `watchEffect()` | Auto-tracking effect | `watchEffect(() => {})` |
| `toRef()` | Property to ref | `toRef(obj, 'key')` |
| `toRefs()` | Object to refs | `toRefs(reactiveObj)` |
| `unref()` | Unwrap ref | `unref(maybeRef)` |
| `provide()` | Provide to descendants | `provide(key, value)` |
| `inject()` | Inject from ancestor | `inject(key, default)` |
