# Vue 3 Patterns & Best Practices

Advanced patterns, architectural best practices, and performance optimization for Vue 3 applications.

**Official Documentation:** https://vuejs.org/guide/best-practices/

---

## Table of Contents

1. [Component Patterns](#component-patterns)
2. [State Management Patterns](#state-management-patterns)
3. [Composable Patterns](#composable-patterns)
4. [Performance Optimization](#performance-optimization)
5. [Form Handling](#form-handling)
6. [Async Patterns](#async-patterns)
7. [Testing Patterns](#testing-patterns)
8. [Project Structure](#project-structure)

---

## Component Patterns

### Renderless Components

Components that provide logic without rendering any UI.

```vue
<!-- components/renderless/MouseTracker.vue -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

const x = ref(0);
const y = ref(0);

function update(event: MouseEvent) {
  x.value = event.pageX;
  y.value = event.pageY;
}

onMounted(() => window.addEventListener('mousemove', update));
onUnmounted(() => window.removeEventListener('mousemove', update));

defineExpose({ x, y });
</script>

<template>
  <slot :x="x" :y="y" />
</template>
```

```vue
<!-- Usage -->
<template>
  <MouseTracker v-slot="{ x, y }">
    <div>Mouse: {{ x }}, {{ y }}</div>
  </MouseTracker>
</template>
```

### Compound Components

Components that work together with shared implicit state.

```vue
<!-- components/Tabs/TabsContext.ts -->
<script lang="ts">
import { InjectionKey, Ref } from 'vue';

export interface TabsContext {
  activeTab: Ref<string>;
  setActiveTab: (id: string) => void;
  registerTab: (id: string) => void;
  unregisterTab: (id: string) => void;
}

export const TabsKey: InjectionKey<TabsContext> = Symbol('Tabs');
</script>
```

```vue
<!-- components/Tabs/Tabs.vue -->
<script setup lang="ts">
import { ref, provide, readonly } from 'vue';
import { TabsKey, type TabsContext } from './TabsContext';

const props = defineProps<{
  defaultTab?: string;
}>();

const emit = defineEmits<{
  change: [tab: string];
}>();

const activeTab = ref(props.defaultTab || '');
const tabs = ref<string[]>([]);

function setActiveTab(id: string) {
  activeTab.value = id;
  emit('change', id);
}

function registerTab(id: string) {
  if (!tabs.value.includes(id)) {
    tabs.value.push(id);
    if (!activeTab.value) {
      activeTab.value = id;
    }
  }
}

function unregisterTab(id: string) {
  const index = tabs.value.indexOf(id);
  if (index > -1) {
    tabs.value.splice(index, 1);
  }
}

provide<TabsContext>(TabsKey, {
  activeTab: readonly(activeTab),
  setActiveTab,
  registerTab,
  unregisterTab,
});
</script>

<template>
  <div class="tabs">
    <slot />
  </div>
</template>
```

```vue
<!-- components/Tabs/Tab.vue -->
<script setup lang="ts">
import { inject, onMounted, onUnmounted, computed } from 'vue';
import { TabsKey } from './TabsContext';

const props = defineProps<{
  id: string;
  label: string;
}>();

const context = inject(TabsKey);
if (!context) throw new Error('Tab must be used within Tabs');

const isActive = computed(() => context.activeTab.value === props.id);

onMounted(() => context.registerTab(props.id));
onUnmounted(() => context.unregisterTab(props.id));
</script>

<template>
  <div v-if="isActive" class="tab-content">
    <slot />
  </div>
</template>
```

```vue
<!-- components/Tabs/TabList.vue -->
<script setup lang="ts">
import { inject } from 'vue';
import { TabsKey } from './TabsContext';

const context = inject(TabsKey);
if (!context) throw new Error('TabList must be used within Tabs');
</script>

<template>
  <div class="tab-list" role="tablist">
    <slot :activeTab="context.activeTab.value" :setActiveTab="context.setActiveTab" />
  </div>
</template>
```

```vue
<!-- Usage -->
<template>
  <Tabs default-tab="profile" @change="handleTabChange">
    <TabList v-slot="{ activeTab, setActiveTab }">
      <button
        v-for="tab in ['profile', 'settings', 'billing']"
        :key="tab"
        :class="{ active: activeTab === tab }"
        @click="setActiveTab(tab)"
      >
        {{ tab }}
      </button>
    </TabList>

    <Tab id="profile" label="Profile">
      <ProfileContent />
    </Tab>
    <Tab id="settings" label="Settings">
      <SettingsContent />
    </Tab>
    <Tab id="billing" label="Billing">
      <BillingContent />
    </Tab>
  </Tabs>
</template>
```

### Controlled vs Uncontrolled Components

```vue
<!-- Hybrid component supporting both modes -->
<script setup lang="ts">
import { ref, computed, watch } from 'vue';

interface Props {
  modelValue?: string;
  defaultValue?: string;
}

const props = withDefaults(defineProps<Props>(), {
  defaultValue: '',
});

const emit = defineEmits<{
  'update:modelValue': [value: string];
  change: [value: string];
}>();

// Internal state for uncontrolled mode
const internalValue = ref(props.defaultValue);

// Determine if controlled
const isControlled = computed(() => props.modelValue !== undefined);

// Computed value that works in both modes
const value = computed({
  get: () => (isControlled.value ? props.modelValue! : internalValue.value),
  set: (newValue: string) => {
    if (isControlled.value) {
      emit('update:modelValue', newValue);
    } else {
      internalValue.value = newValue;
    }
    emit('change', newValue);
  },
});
</script>

<template>
  <input v-model="value" type="text" />
</template>
```

### Higher-Order Components (HOC)

```typescript
// hoc/withLoading.ts
import { defineComponent, h, PropType } from 'vue';

export function withLoading<T extends { new (): any }>(
  WrappedComponent: T,
  loadingComponent?: any
) {
  return defineComponent({
    name: `WithLoading${WrappedComponent.name || 'Component'}`,
    props: {
      loading: {
        type: Boolean,
        default: false,
      },
    },
    setup(props, { slots, attrs }) {
      return () => {
        if (props.loading) {
          return loadingComponent
            ? h(loadingComponent)
            : h('div', { class: 'loading' }, 'Loading...');
        }
        return h(WrappedComponent, attrs, slots);
      };
    },
  });
}

// Usage
import UserList from './UserList.vue';
import Spinner from './Spinner.vue';

const UserListWithLoading = withLoading(UserList, Spinner);
```

### Polymorphic Components

```vue
<!-- components/Box.vue -->
<script setup lang="ts">
import { computed, type Component } from 'vue';

interface Props {
  as?: string | Component;
}

const props = withDefaults(defineProps<Props>(), {
  as: 'div',
});

const component = computed(() => props.as);
</script>

<template>
  <component :is="component" v-bind="$attrs">
    <slot />
  </component>
</template>
```

```vue
<!-- Usage -->
<template>
  <Box as="section" class="content">Section content</Box>
  <Box as="article">Article content</Box>
  <Box :as="RouterLink" to="/about">Link content</Box>
</template>
```

---

## State Management Patterns

### Module Pattern with Composables

```typescript
// stores/useUserStore.ts
import { ref, computed, readonly } from 'vue';

// Module-level state (singleton)
const user = ref<User | null>(null);
const loading = ref(false);
const error = ref<Error | null>(null);

export function useUserStore() {
  const isAuthenticated = computed(() => !!user.value);
  const displayName = computed(() =>
    user.value ? `${user.value.firstName} ${user.value.lastName}` : 'Guest'
  );

  async function login(credentials: LoginCredentials) {
    loading.value = true;
    error.value = null;
    try {
      user.value = await authService.login(credentials);
    } catch (e) {
      error.value = e as Error;
      throw e;
    } finally {
      loading.value = false;
    }
  }

  async function logout() {
    await authService.logout();
    user.value = null;
  }

  function $reset() {
    user.value = null;
    loading.value = false;
    error.value = null;
  }

  return {
    // State (readonly to prevent direct mutations)
    user: readonly(user),
    loading: readonly(loading),
    error: readonly(error),
    // Getters
    isAuthenticated,
    displayName,
    // Actions
    login,
    logout,
    $reset,
  };
}
```

### Factory Pattern for Instance Stores

```typescript
// stores/createTodoStore.ts
import { ref, computed } from 'vue';

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

export function createTodoStore() {
  const todos = ref<Todo[]>([]);
  const filter = ref<'all' | 'active' | 'completed'>('all');

  const filteredTodos = computed(() => {
    switch (filter.value) {
      case 'active':
        return todos.value.filter(t => !t.completed);
      case 'completed':
        return todos.value.filter(t => t.completed);
      default:
        return todos.value;
    }
  });

  const remaining = computed(() =>
    todos.value.filter(t => !t.completed).length
  );

  function addTodo(text: string) {
    todos.value.push({
      id: crypto.randomUUID(),
      text,
      completed: false,
    });
  }

  function toggleTodo(id: string) {
    const todo = todos.value.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  }

  function removeTodo(id: string) {
    const index = todos.value.findIndex(t => t.id === id);
    if (index > -1) {
      todos.value.splice(index, 1);
    }
  }

  return {
    todos,
    filter,
    filteredTodos,
    remaining,
    addTodo,
    toggleTodo,
    removeTodo,
  };
}

// Usage - each call creates a new instance
const listA = createTodoStore();
const listB = createTodoStore();
```

### Optimistic Updates Pattern

```typescript
// composables/useOptimistic.ts
import { ref, type Ref } from 'vue';

export function useOptimistic<T>(
  serverState: Ref<T>,
  updateFn: (optimistic: T, current: T) => T
) {
  const optimisticState = ref<T | null>(null) as Ref<T | null>;

  const state = computed(() =>
    optimisticState.value !== null
      ? updateFn(optimisticState.value, serverState.value)
      : serverState.value
  );

  function setOptimistic(value: T) {
    optimisticState.value = value;
  }

  function clearOptimistic() {
    optimisticState.value = null;
  }

  return {
    state,
    setOptimistic,
    clearOptimistic,
  };
}

// Usage
const { state: todos, setOptimistic, clearOptimistic } = useOptimistic(
  serverTodos,
  (optimistic, current) => [...current, optimistic]
);

async function addTodo(newTodo: Todo) {
  setOptimistic(newTodo);
  try {
    await api.addTodo(newTodo);
    // Server state will update via query invalidation
  } catch (error) {
    clearOptimistic();
    throw error;
  }
}
```

---

## Composable Patterns

### Composable with Cleanup

```typescript
// composables/useEventListener.ts
import { onMounted, onUnmounted, type MaybeRef, unref } from 'vue';

export function useEventListener<K extends keyof WindowEventMap>(
  target: MaybeRef<EventTarget | null | undefined>,
  event: K,
  handler: (event: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions
) {
  const cleanup = () => {
    const el = unref(target);
    el?.removeEventListener(event, handler as EventListener, options);
  };

  onMounted(() => {
    const el = unref(target);
    el?.addEventListener(event, handler as EventListener, options);
  });

  onUnmounted(cleanup);

  return cleanup;
}
```

### Composable with Async State

```typescript
// composables/useAsyncState.ts
import { ref, shallowRef, type Ref } from 'vue';

interface UseAsyncStateOptions<T> {
  immediate?: boolean;
  initialState?: T;
  onError?: (error: Error) => void;
}

interface UseAsyncStateReturn<T, P extends any[]> {
  state: Ref<T>;
  isLoading: Ref<boolean>;
  error: Ref<Error | null>;
  execute: (...args: P) => Promise<T>;
}

export function useAsyncState<T, P extends any[] = []>(
  asyncFn: (...args: P) => Promise<T>,
  options: UseAsyncStateOptions<T> = {}
): UseAsyncStateReturn<T, P> {
  const {
    immediate = false,
    initialState = null as T,
    onError,
  } = options;

  const state = shallowRef<T>(initialState);
  const isLoading = ref(false);
  const error = ref<Error | null>(null);

  async function execute(...args: P): Promise<T> {
    isLoading.value = true;
    error.value = null;

    try {
      const result = await asyncFn(...args);
      state.value = result;
      return result;
    } catch (e) {
      error.value = e as Error;
      onError?.(e as Error);
      throw e;
    } finally {
      isLoading.value = false;
    }
  }

  if (immediate) {
    execute(...([] as unknown as P));
  }

  return {
    state,
    isLoading,
    error,
    execute,
  };
}

// Usage
const { state: users, isLoading, execute: fetchUsers } = useAsyncState(
  () => api.getUsers(),
  { immediate: true }
);
```

### Composable with Dependency Injection

```typescript
// composables/useInject.ts
import { inject, type InjectionKey } from 'vue';

export function useInjectStrict<T>(key: InjectionKey<T>, fallback?: T): T {
  const resolved = inject(key, fallback);

  if (resolved === undefined) {
    throw new Error(`Could not resolve injection key: ${String(key)}`);
  }

  return resolved;
}

// Usage
const user = useInjectStrict(UserKey); // Throws if not provided
```

### Composable Composition

```typescript
// composables/usePagination.ts
import { ref, computed, watch, type Ref } from 'vue';

interface UsePaginationOptions {
  page?: number;
  pageSize?: number;
  total?: Ref<number>;
}

export function usePagination(options: UsePaginationOptions = {}) {
  const page = ref(options.page ?? 1);
  const pageSize = ref(options.pageSize ?? 10);
  const total = options.total ?? ref(0);

  const totalPages = computed(() =>
    Math.ceil(total.value / pageSize.value)
  );

  const hasNext = computed(() => page.value < totalPages.value);
  const hasPrev = computed(() => page.value > 1);

  const offset = computed(() => (page.value - 1) * pageSize.value);

  function next() {
    if (hasNext.value) page.value++;
  }

  function prev() {
    if (hasPrev.value) page.value--;
  }

  function goTo(pageNum: number) {
    page.value = Math.max(1, Math.min(pageNum, totalPages.value));
  }

  // Reset to page 1 when pageSize changes
  watch(pageSize, () => {
    page.value = 1;
  });

  return {
    page,
    pageSize,
    totalPages,
    hasNext,
    hasPrev,
    offset,
    next,
    prev,
    goTo,
  };
}

// composables/usePaginatedFetch.ts - Composing composables
export function usePaginatedFetch<T>(url: string) {
  const items = ref<T[]>([]);
  const total = ref(0);

  const pagination = usePagination({ total });
  const { state, isLoading, execute } = useAsyncState(async () => {
    const response = await fetch(
      `${url}?page=${pagination.page.value}&pageSize=${pagination.pageSize.value}`
    );
    const data = await response.json();
    items.value = data.items;
    total.value = data.total;
    return data;
  });

  // Refetch when pagination changes
  watch([pagination.page, pagination.pageSize], () => {
    execute();
  });

  return {
    items,
    isLoading,
    ...pagination,
    refetch: execute,
  };
}
```

---

## Performance Optimization

### Computed vs Methods

```vue
<script setup lang="ts">
import { ref, computed } from 'vue';

const items = ref<Item[]>([]);

// Computed: cached, only re-evaluates when dependencies change
const sortedItems = computed(() => {
  console.log('Computing sorted items');
  return [...items.value].sort((a, b) => a.name.localeCompare(b.name));
});

// Method: called every render, no caching
function getSortedItems() {
  console.log('Getting sorted items');
  return [...items.value].sort((a, b) => a.name.localeCompare(b.name));
}
</script>

<template>
  <!-- sortedItems called once, cached -->
  <div>{{ sortedItems.length }}</div>
  <div>{{ sortedItems[0] }}</div>

  <!-- getSortedItems() called twice, computed each time -->
  <div>{{ getSortedItems().length }}</div>
  <div>{{ getSortedItems()[0] }}</div>
</template>
```

### v-once and v-memo

```vue
<template>
  <!-- Rendered once, never updates -->
  <div v-once>
    {{ expensiveComputation() }}
  </div>

  <!-- Only re-renders when dependencies change -->
  <div v-for="item in items" :key="item.id" v-memo="[item.id, item.selected]">
    <ExpensiveComponent :item="item" />
  </div>
</template>
```

### Lazy Loading Components

```vue
<script setup lang="ts">
import { defineAsyncComponent, ref } from 'vue';

// Basic async component
const AsyncModal = defineAsyncComponent(() =>
  import('./components/Modal.vue')
);

// With loading and error states
const AsyncChart = defineAsyncComponent({
  loader: () => import('./components/Chart.vue'),
  loadingComponent: () => import('./components/ChartSkeleton.vue'),
  errorComponent: () => import('./components/ChartError.vue'),
  delay: 200, // Show loading after 200ms
  timeout: 10000, // Timeout after 10s
});

const showModal = ref(false);
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <!-- Only loads when showModal is true -->
  <AsyncModal v-if="showModal" @close="showModal = false" />

  <Suspense>
    <AsyncChart :data="chartData" />
    <template #fallback>
      <ChartSkeleton />
    </template>
  </Suspense>
</template>
```

### Virtual Scrolling

```vue
<!-- Using @vueuse/core -->
<script setup lang="ts">
import { ref } from 'vue';
import { useVirtualList } from '@vueuse/core';

const items = ref(Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`,
})));

const { list, containerProps, wrapperProps } = useVirtualList(items, {
  itemHeight: 50,
  overscan: 10,
});
</script>

<template>
  <div v-bind="containerProps" class="virtual-list-container">
    <div v-bind="wrapperProps">
      <div
        v-for="{ data, index } in list"
        :key="data.id"
        class="virtual-list-item"
      >
        {{ data.name }}
      </div>
    </div>
  </div>
</template>

<style scoped>
.virtual-list-container {
  height: 400px;
  overflow-y: auto;
}

.virtual-list-item {
  height: 50px;
}
</style>
```

### Debouncing and Throttling

```typescript
// composables/useDebouncedRef.ts
import { ref, customRef, type Ref } from 'vue';

export function useDebouncedRef<T>(value: T, delay = 300): Ref<T> {
  let timeout: ReturnType<typeof setTimeout>;

  return customRef((track, trigger) => ({
    get() {
      track();
      return value;
    },
    set(newValue: T) {
      clearTimeout(timeout);
      timeout = setTimeout(() => {
        value = newValue;
        trigger();
      }, delay);
    },
  }));
}

// Usage
const searchQuery = useDebouncedRef('', 500);
// Updates debounced by 500ms
```

```vue
<script setup lang="ts">
import { ref, watch } from 'vue';
import { useDebounceFn, useThrottleFn } from '@vueuse/core';

const searchInput = ref('');
const scrollPosition = ref(0);

// Debounced search
const debouncedSearch = useDebounceFn((query: string) => {
  api.search(query);
}, 500);

watch(searchInput, (query) => {
  debouncedSearch(query);
});

// Throttled scroll handler
const handleScroll = useThrottleFn(() => {
  scrollPosition.value = window.scrollY;
}, 100);
</script>
```

### Selective Reactivity with shallowRef

```typescript
import { shallowRef, triggerRef } from 'vue';

// For large arrays/objects where you only need top-level reactivity
const largeDataset = shallowRef<DataItem[]>([]);

// Setting new array triggers update
largeDataset.value = newData;

// Mutating items doesn't trigger update (better performance)
largeDataset.value[0].name = 'Updated'; // No re-render

// Manually trigger when needed
triggerRef(largeDataset);
```

---

## Form Handling

### Form Validation Pattern

```typescript
// composables/useForm.ts
import { ref, reactive, computed } from 'vue';

type ValidationRule<T> = (value: T) => string | true;

interface FieldConfig<T> {
  initialValue: T;
  rules?: ValidationRule<T>[];
}

interface FormConfig {
  [key: string]: FieldConfig<any>;
}

export function useForm<T extends FormConfig>(config: T) {
  type FormValues = { [K in keyof T]: T[K]['initialValue'] };
  type FormErrors = { [K in keyof T]: string };

  const values = reactive<FormValues>(
    Object.fromEntries(
      Object.entries(config).map(([key, field]) => [key, field.initialValue])
    ) as FormValues
  );

  const errors = reactive<FormErrors>(
    Object.fromEntries(
      Object.keys(config).map(key => [key, ''])
    ) as FormErrors
  );

  const touched = reactive<{ [K in keyof T]: boolean }>(
    Object.fromEntries(
      Object.keys(config).map(key => [key, false])
    ) as { [K in keyof T]: boolean }
  );

  const isValid = computed(() =>
    Object.values(errors).every(error => error === '')
  );

  const isDirty = computed(() =>
    Object.keys(config).some(
      key => values[key as keyof T] !== config[key as keyof T].initialValue
    )
  );

  function validate(field?: keyof T): boolean {
    const fieldsToValidate = field ? [field] : Object.keys(config);

    let valid = true;

    for (const key of fieldsToValidate) {
      const fieldConfig = config[key as keyof T];
      const value = values[key as keyof T];
      errors[key as keyof T] = '';

      for (const rule of fieldConfig.rules || []) {
        const result = rule(value);
        if (result !== true) {
          errors[key as keyof T] = result;
          valid = false;
          break;
        }
      }
    }

    return valid;
  }

  function handleBlur(field: keyof T) {
    touched[field] = true;
    validate(field);
  }

  function reset() {
    for (const key of Object.keys(config)) {
      values[key as keyof T] = config[key as keyof T].initialValue;
      errors[key as keyof T] = '';
      touched[key as keyof T] = false;
    }
  }

  return {
    values,
    errors,
    touched,
    isValid,
    isDirty,
    validate,
    handleBlur,
    reset,
  };
}

// Validation rules
export const required = (msg = 'Required'): ValidationRule<any> =>
  (value) => (value !== '' && value !== null && value !== undefined) || msg;

export const email = (msg = 'Invalid email'): ValidationRule<string> =>
  (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) || msg;

export const minLength = (min: number, msg?: string): ValidationRule<string> =>
  (value) => value.length >= min || msg || `Minimum ${min} characters`;
```

```vue
<!-- Usage -->
<script setup lang="ts">
import { useForm, required, email, minLength } from '@/composables/useForm';

const { values, errors, touched, isValid, validate, handleBlur, reset } = useForm({
  email: {
    initialValue: '',
    rules: [required(), email()],
  },
  password: {
    initialValue: '',
    rules: [required(), minLength(8, 'Password must be at least 8 characters')],
  },
});

async function handleSubmit() {
  if (validate()) {
    await api.login(values);
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <div>
      <input
        v-model="values.email"
        type="email"
        @blur="handleBlur('email')"
      />
      <span v-if="touched.email && errors.email" class="error">
        {{ errors.email }}
      </span>
    </div>

    <div>
      <input
        v-model="values.password"
        type="password"
        @blur="handleBlur('password')"
      />
      <span v-if="touched.password && errors.password" class="error">
        {{ errors.password }}
      </span>
    </div>

    <button type="submit" :disabled="!isValid">Login</button>
  </form>
</template>
```

---

## Async Patterns

### Suspense with Async Setup

```vue
<!-- AsyncUserProfile.vue -->
<script setup lang="ts">
// Component with async setup - works with Suspense
const props = defineProps<{ userId: string }>();

// Top-level await
const user = await fetchUser(props.userId);
const posts = await fetchUserPosts(props.userId);
</script>

<template>
  <div class="profile">
    <h1>{{ user.name }}</h1>
    <PostList :posts="posts" />
  </div>
</template>
```

```vue
<!-- Parent component -->
<template>
  <Suspense>
    <AsyncUserProfile :user-id="userId" />

    <template #fallback>
      <ProfileSkeleton />
    </template>
  </Suspense>
</template>
```

### Error Handling with onErrorCaptured

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue';

const error = ref<Error | null>(null);

onErrorCaptured((err, instance, info) => {
  error.value = err;
  console.error('Captured error:', err);
  console.log('In component:', instance);
  console.log('Error info:', info);

  // Return false to stop propagation
  return false;
});

function retry() {
  error.value = null;
}
</script>

<template>
  <div v-if="error" class="error-boundary">
    <h2>Something went wrong</h2>
    <p>{{ error.message }}</p>
    <button @click="retry">Try Again</button>
  </div>

  <slot v-else />
</template>
```

### Cancelable Requests Pattern

```typescript
// composables/useCancelableRequest.ts
import { ref, onUnmounted } from 'vue';

export function useCancelableRequest<T>(
  requestFn: (signal: AbortSignal) => Promise<T>
) {
  const data = ref<T | null>(null);
  const loading = ref(false);
  const error = ref<Error | null>(null);
  let controller: AbortController | null = null;

  async function execute() {
    // Cancel previous request
    cancel();

    controller = new AbortController();
    loading.value = true;
    error.value = null;

    try {
      data.value = await requestFn(controller.signal);
    } catch (e) {
      if ((e as Error).name !== 'AbortError') {
        error.value = e as Error;
      }
    } finally {
      loading.value = false;
    }
  }

  function cancel() {
    controller?.abort();
    controller = null;
  }

  onUnmounted(cancel);

  return {
    data,
    loading,
    error,
    execute,
    cancel,
  };
}

// Usage
const { data, loading, execute } = useCancelableRequest((signal) =>
  fetch('/api/data', { signal }).then(r => r.json())
);
```

---

## Testing Patterns

### Component Testing with Vue Test Utils

```typescript
// components/__tests__/Counter.spec.ts
import { mount } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import Counter from '../Counter.vue';

describe('Counter', () => {
  it('renders initial count', () => {
    const wrapper = mount(Counter, {
      props: { initialCount: 5 },
    });

    expect(wrapper.text()).toContain('5');
  });

  it('increments count on button click', async () => {
    const wrapper = mount(Counter);

    await wrapper.find('[data-testid="increment"]').trigger('click');

    expect(wrapper.text()).toContain('1');
  });

  it('emits update event', async () => {
    const wrapper = mount(Counter);

    await wrapper.find('[data-testid="increment"]').trigger('click');

    expect(wrapper.emitted('update')).toHaveLength(1);
    expect(wrapper.emitted('update')![0]).toEqual([1]);
  });
});
```

### Testing Composables

```typescript
// composables/__tests__/useCounter.spec.ts
import { describe, it, expect } from 'vitest';
import { useCounter } from '../useCounter';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { count } = useCounter();
    expect(count.value).toBe(0);
  });

  it('initializes with custom value', () => {
    const { count } = useCounter(10);
    expect(count.value).toBe(10);
  });

  it('increments count', () => {
    const { count, increment } = useCounter();

    increment();

    expect(count.value).toBe(1);
  });

  it('computes doubled value', () => {
    const { count, doubled, increment } = useCounter(5);

    expect(doubled.value).toBe(10);

    increment();
    expect(doubled.value).toBe(12);
  });
});
```

### Testing with Provide/Inject

```typescript
import { mount } from '@vue/test-utils';
import { describe, it, expect } from 'vitest';
import ChildComponent from '../ChildComponent.vue';
import { ThemeKey } from '../keys';
import { ref } from 'vue';

describe('ChildComponent with injection', () => {
  it('uses injected theme', () => {
    const wrapper = mount(ChildComponent, {
      global: {
        provide: {
          [ThemeKey as symbol]: ref('dark'),
        },
      },
    });

    expect(wrapper.classes()).toContain('theme-dark');
  });
});
```

---

## Project Structure

### Feature-Based Structure

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.vue
│   │   │   └── RegisterForm.vue
│   │   ├── composables/
│   │   │   └── useAuth.ts
│   │   ├── services/
│   │   │   └── authService.ts
│   │   ├── stores/
│   │   │   └── useAuthStore.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── index.ts           # Public API
│   │
│   ├── products/
│   │   ├── components/
│   │   ├── composables/
│   │   ├── services/
│   │   └── index.ts
│   │
│   └── orders/
│       └── ...
│
├── shared/
│   ├── components/            # Shared UI components
│   │   ├── Button.vue
│   │   ├── Modal.vue
│   │   └── index.ts
│   ├── composables/           # Shared composables
│   │   ├── useFetch.ts
│   │   └── useLocalStorage.ts
│   ├── utils/                 # Utility functions
│   └── types/                 # Shared types
│
├── layouts/
│   ├── DefaultLayout.vue
│   └── AuthLayout.vue
│
├── pages/                     # Route pages
│   ├── index.vue
│   ├── login.vue
│   └── products/
│       ├── index.vue
│       └── [id].vue
│
├── plugins/                   # Vue plugins
├── router/
├── App.vue
└── main.ts
```

### Barrel Exports

```typescript
// features/auth/index.ts
export { default as LoginForm } from './components/LoginForm.vue';
export { default as RegisterForm } from './components/RegisterForm.vue';
export { useAuth } from './composables/useAuth';
export { useAuthStore } from './stores/useAuthStore';
export type { User, LoginCredentials } from './types';

// Usage in other features
import { useAuth, LoginForm } from '@/features/auth';
```

---

## Quick Reference

### Component Communication

| Method | Use Case |
|--------|----------|
| Props | Parent → Child data |
| Emits | Child → Parent events |
| v-model | Two-way binding |
| Provide/Inject | Deep component tree |
| Composables | Shared stateful logic |
| Event Bus | Sibling communication (use sparingly) |

### Performance Checklist

- [ ] Use `computed` for derived state
- [ ] Use `shallowRef` for large objects
- [ ] Use `v-once` for static content
- [ ] Use `v-memo` for expensive list items
- [ ] Lazy load heavy components
- [ ] Virtual scroll for long lists
- [ ] Debounce/throttle frequent updates
- [ ] Split code by routes

### Composable Naming Conventions

| Pattern | Example |
|---------|---------|
| State management | `useUserStore`, `useCartStore` |
| Side effects | `useEventListener`, `useInterval` |
| Data fetching | `useFetch`, `useQuery` |
| Browser APIs | `useLocalStorage`, `useGeolocation` |
| UI state | `useModal`, `useToast` |
