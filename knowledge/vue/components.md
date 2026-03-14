# Vue 3 Components

> Official Documentation: https://vuejs.org/guide/essentials/component-basics.html

Components are the building blocks of Vue applications. They allow you to encapsulate reusable UI elements with their own logic, template, and styles.

---

## 1. Component Basics (SFC Structure)

Single File Components (SFCs) use the `.vue` extension and contain three optional blocks:

```vue
<script setup lang="ts">
// Logic: imports, reactive state, computed, methods
import { ref, computed } from 'vue';

const count = ref(0);
const doubled = computed(() => count.value * 2);

function increment() {
  count.value++;
}
</script>

<template>
  <!-- Template: HTML with Vue directives -->
  <div class="counter">
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>

<style scoped>
/* Styles: scoped to this component */
.counter {
  padding: 1rem;
  border: 1px solid #ccc;
  border-radius: 8px;
}

button {
  background: #42b883;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  cursor: pointer;
}
</style>
```

### Script Setup vs Options API

```vue
<!-- Composition API with <script setup> (Recommended) -->
<script setup lang="ts">
import { ref } from 'vue';
const message = ref('Hello');
</script>

<!-- Options API (Legacy) -->
<script lang="ts">
import { defineComponent } from 'vue';

export default defineComponent({
  data() {
    return {
      message: 'Hello'
    };
  }
});
</script>
```

### Multiple Script Blocks

```vue
<script lang="ts">
// Regular script for exports that run once per component definition
export interface User {
  id: number;
  name: string;
}
</script>

<script setup lang="ts">
// Setup script for component logic
import { ref } from 'vue';
const user = ref<User | null>(null);
</script>
```

---

## 2. Props Declaration and Validation

### Runtime Declaration

```vue
<script setup lang="ts">
const props = defineProps({
  // Basic type check
  title: String,

  // Multiple possible types
  id: [String, Number],

  // Required prop
  name: {
    type: String,
    required: true
  },

  // With default value
  count: {
    type: Number,
    default: 0
  },

  // Object/Array defaults must use factory function
  items: {
    type: Array,
    default: () => []
  },

  // Custom validator
  status: {
    type: String,
    validator: (value: string) => {
      return ['pending', 'active', 'completed'].includes(value);
    }
  }
});
</script>
```

### TypeScript Declaration (Recommended)

```vue
<script setup lang="ts">
// Type-only declaration
interface Props {
  title: string;
  count?: number;
  items?: string[];
  disabled?: boolean;
  user: {
    id: number;
    name: string;
  };
}

// With defaults using withDefaults
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],
  disabled: false
});

// Accessing props
console.log(props.title);
console.log(props.count);
</script>
```

### Props Best Practices

```vue
<script setup lang="ts">
// DO: Use readonly props
const props = defineProps<{
  modelValue: string;
}>();

// DON'T: Mutate props directly
// props.modelValue = 'new value'; // ERROR!

// DO: Emit events or use local copy
import { ref, watch } from 'vue';

const localValue = ref(props.modelValue);

watch(() => props.modelValue, (newVal) => {
  localValue.value = newVal;
});
</script>
```

---

## 3. Events and Emits

### Basic Emits Declaration

```vue
<script setup lang="ts">
// Runtime declaration
const emit = defineEmits(['submit', 'cancel', 'update']);

// Usage
function handleSubmit() {
  emit('submit');
}

function handleCancel() {
  emit('cancel');
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <button type="submit">Submit</button>
    <button type="button" @click="handleCancel">Cancel</button>
  </form>
</template>
```

### TypeScript Emit Declaration

```vue
<script setup lang="ts">
// Type-based declaration with payload types
const emit = defineEmits<{
  (e: 'update', id: number, value: string): void;
  (e: 'delete', id: number): void;
  (e: 'submit', data: { name: string; email: string }): void;
  (e: 'cancel'): void;
}>();

// Vue 3.3+ shorthand syntax
const emit = defineEmits<{
  update: [id: number, value: string];
  delete: [id: number];
  submit: [data: { name: string; email: string }];
  cancel: [];
}>();

// Usage with type safety
function handleUpdate(id: number) {
  emit('update', id, 'new value');
}
</script>
```

### Event Validation

```vue
<script setup lang="ts">
const emit = defineEmits({
  // Validation function
  submit: (payload: { email: string }) => {
    if (payload.email && payload.email.includes('@')) {
      return true;
    }
    console.warn('Invalid email');
    return false;
  },
  // No validation
  cancel: null
});
</script>
```

### Parent Listening to Events

```vue
<!-- Parent.vue -->
<template>
  <ChildComponent
    @submit="handleSubmit"
    @update="handleUpdate"
    @cancel="handleCancel"
  />
</template>

<script setup lang="ts">
function handleSubmit(data: { name: string; email: string }) {
  console.log('Submitted:', data);
}

function handleUpdate(id: number, value: string) {
  console.log('Updated:', id, value);
}

function handleCancel() {
  console.log('Cancelled');
}
</script>
```

---

## 4. v-model on Components

### Basic v-model (Vue 3.4+ with defineModel)

```vue
<!-- Parent.vue -->
<template>
  <CustomInput v-model="searchText" />
</template>

<script setup lang="ts">
import { ref } from 'vue';
const searchText = ref('');
</script>

<!-- CustomInput.vue -->
<script setup lang="ts">
const model = defineModel<string>();
</script>

<template>
  <input
    :value="model"
    @input="model = ($event.target as HTMLInputElement).value"
  />
</template>
```

### defineModel Options

```vue
<script setup lang="ts">
// With options
const model = defineModel<string>({
  required: true,
  default: ''
});

// With local transformations
const model = defineModel<string>({
  get(value) {
    return value?.toUpperCase();
  },
  set(value) {
    return value?.toLowerCase();
  }
});
</script>
```

### Named v-model

```vue
<!-- Parent.vue -->
<template>
  <UserForm
    v-model:firstName="user.firstName"
    v-model:lastName="user.lastName"
    v-model:email="user.email"
  />
</template>

<!-- UserForm.vue -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName');
const lastName = defineModel<string>('lastName');
const email = defineModel<string>('email');
</script>

<template>
  <input v-model="firstName" placeholder="First name" />
  <input v-model="lastName" placeholder="Last name" />
  <input v-model="email" type="email" placeholder="Email" />
</template>
```

### v-model Modifiers

```vue
<!-- Parent.vue -->
<template>
  <CustomInput v-model.capitalize="text" />
</template>

<!-- CustomInput.vue -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string, 'capitalize'>({
  set(value) {
    if (modifiers.capitalize && value) {
      return value.charAt(0).toUpperCase() + value.slice(1);
    }
    return value;
  }
});
</script>

<template>
  <input
    :value="model"
    @input="model = ($event.target as HTMLInputElement).value"
  />
</template>
```

### Legacy v-model (Pre Vue 3.4)

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string;
}>();

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void;
}>();
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
  />
</template>
```

---

## 5. Slots (Default, Named, Scoped)

### Default Slot

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot>
      <!-- Fallback content if no slot provided -->
      <p>Default card content</p>
    </slot>
  </div>
</template>

<!-- Parent.vue -->
<template>
  <Card>
    <p>This replaces the default slot</p>
  </Card>
</template>
```

### Named Slots

```vue
<!-- BaseLayout.vue -->
<template>
  <div class="layout">
    <header>
      <slot name="header">Default Header</slot>
    </header>

    <main>
      <slot>Default Content</slot>
    </main>

    <aside>
      <slot name="sidebar">Default Sidebar</slot>
    </aside>

    <footer>
      <slot name="footer">Default Footer</slot>
    </footer>
  </div>
</template>

<!-- Parent.vue -->
<template>
  <BaseLayout>
    <template #header>
      <h1>My Application</h1>
    </template>

    <template #default>
      <p>Main content goes here</p>
    </template>

    <template #sidebar>
      <nav>Navigation links</nav>
    </template>

    <template #footer>
      <p>Copyright 2024</p>
    </template>
  </BaseLayout>
</template>
```

### Scoped Slots

```vue
<!-- DataList.vue -->
<script setup lang="ts">
interface Item {
  id: number;
  name: string;
  status: 'active' | 'inactive';
}

defineProps<{
  items: Item[];
}>();
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="item.id">
      <slot
        name="item"
        :item="item"
        :index="index"
        :isFirst="index === 0"
        :isLast="index === items.length - 1"
      >
        <!-- Default rendering -->
        {{ item.name }}
      </slot>
    </li>
  </ul>
</template>

<!-- Parent.vue -->
<template>
  <DataList :items="items">
    <template #item="{ item, index, isFirst, isLast }">
      <div :class="{ first: isFirst, last: isLast }">
        <span>{{ index + 1 }}.</span>
        <strong>{{ item.name }}</strong>
        <span :class="item.status">{{ item.status }}</span>
      </div>
    </template>
  </DataList>
</template>
```

### TypeScript Slot Definitions

```vue
<script setup lang="ts">
defineSlots<{
  default(): any;
  header(): any;
  item(props: { item: Item; index: number }): any;
  footer(props: { total: number }): any;
}>();
</script>
```

### Conditional Slots

```vue
<script setup lang="ts">
import { useSlots } from 'vue';

const slots = useSlots();
const hasHeader = computed(() => !!slots.header);
</script>

<template>
  <div class="card">
    <header v-if="hasHeader">
      <slot name="header" />
    </header>
    <main>
      <slot />
    </main>
  </div>
</template>
```

---

## 6. Provide/Inject

### Basic Provide/Inject

```vue
<!-- Ancestor.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue';

const theme = ref('dark');
const user = ref({ name: 'John', id: 1 });

provide('theme', theme);
provide('user', user);
</script>

<!-- Descendant.vue (any level deep) -->
<script setup lang="ts">
import { inject } from 'vue';

const theme = inject('theme');
const user = inject('user');
</script>
```

### Typed Provide/Inject with InjectionKey

```typescript
// keys.ts
import type { InjectionKey, Ref } from 'vue';

export interface User {
  id: number;
  name: string;
  email: string;
}

export const userKey: InjectionKey<Ref<User>> = Symbol('user');
export const themeKey: InjectionKey<Ref<'light' | 'dark'>> = Symbol('theme');
```

```vue
<!-- Provider.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue';
import { userKey, themeKey, type User } from './keys';

const user = ref<User>({ id: 1, name: 'John', email: 'john@example.com' });
const theme = ref<'light' | 'dark'>('dark');

provide(userKey, user);
provide(themeKey, theme);
</script>

<!-- Consumer.vue -->
<script setup lang="ts">
import { inject } from 'vue';
import { userKey, themeKey } from './keys';

// Fully typed!
const user = inject(userKey);
const theme = inject(themeKey);

// With default value
const theme = inject(themeKey, ref('light'));

// Throw error if not provided
const user = inject(userKey);
if (!user) {
  throw new Error('User must be provided');
}
</script>
```

### Provide/Inject with Reactivity

```vue
<!-- Provider.vue -->
<script setup lang="ts">
import { provide, ref, readonly } from 'vue';

const count = ref(0);

function increment() {
  count.value++;
}

// Provide readonly to prevent direct mutation
provide('count', readonly(count));
provide('increment', increment);
</script>

<!-- Consumer.vue -->
<script setup lang="ts">
import { inject } from 'vue';

const count = inject<Readonly<Ref<number>>>('count');
const increment = inject<() => void>('increment');
</script>

<template>
  <p>Count: {{ count }}</p>
  <button @click="increment">Increment</button>
</template>
```

### App-Level Provide

```typescript
// main.ts
import { createApp } from 'vue';
import App from './App.vue';

const app = createApp(App);

app.provide('appName', 'My Application');
app.provide('version', '1.0.0');

app.mount('#app');
```

---

## 7. Dynamic Components

### Basic Dynamic Component

```vue
<script setup lang="ts">
import { ref, shallowRef } from 'vue';
import TabHome from './TabHome.vue';
import TabProfile from './TabProfile.vue';
import TabSettings from './TabSettings.vue';

const tabs = {
  home: TabHome,
  profile: TabProfile,
  settings: TabSettings
};

type TabName = keyof typeof tabs;
const currentTab = ref<TabName>('home');

// Use shallowRef for component references to avoid deep reactivity
const currentComponent = shallowRef(tabs.home);

function switchTab(tab: TabName) {
  currentTab.value = tab;
  currentComponent.value = tabs[tab];
}
</script>

<template>
  <div class="tabs">
    <button
      v-for="(_, name) in tabs"
      :key="name"
      :class="{ active: currentTab === name }"
      @click="switchTab(name as TabName)"
    >
      {{ name }}
    </button>
  </div>

  <component :is="currentComponent" />
</template>
```

### Dynamic Component with Props

```vue
<script setup lang="ts">
import { ref, computed } from 'vue';

const components = {
  text: defineAsyncComponent(() => import('./TextInput.vue')),
  number: defineAsyncComponent(() => import('./NumberInput.vue')),
  select: defineAsyncComponent(() => import('./SelectInput.vue'))
};

const fieldType = ref<keyof typeof components>('text');
const fieldValue = ref('');

const currentComponent = computed(() => components[fieldType.value]);
</script>

<template>
  <component
    :is="currentComponent"
    v-model="fieldValue"
    :label="'Enter value'"
    @change="handleChange"
  />
</template>
```

---

## 8. Async Components

### Basic Async Component

```typescript
import { defineAsyncComponent } from 'vue';

// Simple lazy loading
const AsyncModal = defineAsyncComponent(() =>
  import('./components/Modal.vue')
);

// With loading and error handling
const AsyncDashboard = defineAsyncComponent({
  loader: () => import('./components/Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,        // Delay before showing loading component (ms)
  timeout: 10000,    // Timeout before showing error component (ms)
  suspensible: false // Opt-out of Suspense control
});
```

### Async Component with Error Handling

```vue
<script setup lang="ts">
import { defineAsyncComponent, h } from 'vue';
import LoadingSpinner from './LoadingSpinner.vue';

const AsyncHeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: {
    props: ['error'],
    setup(props) {
      return () => h('div', { class: 'error' }, [
        h('p', 'Failed to load component'),
        h('pre', props.error?.message)
      ]);
    }
  },
  delay: 100,
  timeout: 30000,
  onError(error, retry, fail, attempts) {
    if (attempts <= 3) {
      // Retry up to 3 times
      retry();
    } else {
      fail();
    }
  }
});
</script>

<template>
  <AsyncHeavyChart :data="chartData" />
</template>
```

### Route-Level Code Splitting

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('../views/Home.vue')
    },
    {
      path: '/dashboard',
      component: () => import('../views/Dashboard.vue')
    },
    {
      path: '/admin',
      component: () => import('../views/Admin.vue'),
      // Named chunk for debugging
      // component: () => import(/* webpackChunkName: "admin" */ '../views/Admin.vue')
    }
  ]
});
```

---

## 9. Attribute Fallthrough

### Automatic Attribute Inheritance

```vue
<!-- MyButton.vue -->
<template>
  <!-- Attributes like class, id, @click automatically fall through -->
  <button class="my-button">
    <slot />
  </button>
</template>

<!-- Parent.vue -->
<template>
  <!-- class="large" and @click will be applied to <button> -->
  <MyButton class="large" @click="handleClick">
    Click Me
  </MyButton>
</template>
```

### Disabling Attribute Inheritance

```vue
<script setup lang="ts">
// Disable automatic inheritance
defineOptions({
  inheritAttrs: false
});
</script>

<script lang="ts">
// Alternative in separate script block
export default {
  inheritAttrs: false
};
</script>
```

### Accessing Fallthrough Attributes

```vue
<script setup lang="ts">
import { useAttrs } from 'vue';

defineOptions({
  inheritAttrs: false
});

const attrs = useAttrs();
</script>

<template>
  <div class="wrapper">
    <!-- Apply attrs to specific element -->
    <input v-bind="attrs" />
    <span class="icon">...</span>
  </div>
</template>
```

### Multi-Root Components

```vue
<script setup lang="ts">
import { useAttrs } from 'vue';

defineOptions({
  inheritAttrs: false
});

const attrs = useAttrs();
</script>

<template>
  <!-- Multiple root elements - must manually bind attrs -->
  <header>Header</header>
  <main v-bind="attrs">Main Content</main>
  <footer>Footer</footer>
</template>
```

---

## 10. Teleport

### Basic Teleport

```vue
<script setup lang="ts">
import { ref } from 'vue';

const showModal = ref(false);
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <!-- Render modal at body level, outside component hierarchy -->
  <Teleport to="body">
    <div v-if="showModal" class="modal-overlay">
      <div class="modal">
        <h2>Modal Title</h2>
        <p>Modal content here</p>
        <button @click="showModal = false">Close</button>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  max-width: 500px;
}
</style>
```

### Teleport to Custom Container

```html
<!-- index.html -->
<body>
  <div id="app"></div>
  <div id="modals"></div>
  <div id="tooltips"></div>
</body>
```

```vue
<template>
  <Teleport to="#modals">
    <Modal v-if="showModal" />
  </Teleport>

  <Teleport to="#tooltips">
    <Tooltip v-if="showTooltip" :content="tooltipContent" />
  </Teleport>
</template>
```

### Conditional Teleport

```vue
<script setup lang="ts">
import { ref } from 'vue';

const isMobile = ref(window.innerWidth < 768);
</script>

<template>
  <!-- Disable teleport on mobile -->
  <Teleport to="body" :disabled="isMobile">
    <div class="sidebar">
      Sidebar content
    </div>
  </Teleport>
</template>
```

### Multiple Teleports to Same Target

```vue
<!-- Components teleport to same target in order -->
<Teleport to="#notifications">
  <Notification message="First" />
</Teleport>

<Teleport to="#notifications">
  <Notification message="Second" />
</Teleport>

<!-- Result in #notifications: First, then Second -->
```

---

## 11. KeepAlive

### Basic KeepAlive

```vue
<script setup lang="ts">
import { ref, shallowRef } from 'vue';
import TabA from './TabA.vue';
import TabB from './TabB.vue';

const currentTab = shallowRef(TabA);
</script>

<template>
  <button @click="currentTab = TabA">Tab A</button>
  <button @click="currentTab = TabB">Tab B</button>

  <!-- Component state is preserved when switching -->
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

### Include/Exclude Components

```vue
<template>
  <!-- Only keep these components alive -->
  <KeepAlive include="TabA,TabB">
    <component :is="currentTab" />
  </KeepAlive>

  <!-- Exclude these from being cached -->
  <KeepAlive exclude="HeavyComponent">
    <component :is="currentTab" />
  </KeepAlive>

  <!-- Using regex -->
  <KeepAlive :include="/Tab[AB]/">
    <component :is="currentTab" />
  </KeepAlive>

  <!-- Using array -->
  <KeepAlive :include="['TabA', 'TabB']">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

### Max Cached Instances

```vue
<template>
  <!-- LRU cache - oldest accessed component removed when limit exceeded -->
  <KeepAlive :max="5">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

### Lifecycle Hooks for Cached Components

```vue
<script setup lang="ts">
import { onActivated, onDeactivated } from 'vue';

// Called when component becomes active (shown)
onActivated(() => {
  console.log('Component activated');
  // Refresh data, start timers, etc.
});

// Called when component is cached (hidden)
onDeactivated(() => {
  console.log('Component deactivated');
  // Stop timers, pause videos, etc.
});
</script>
```

### KeepAlive with Router

```vue
<!-- App.vue -->
<template>
  <router-view v-slot="{ Component }">
    <KeepAlive :include="cachedViews">
      <component :is="Component" />
    </KeepAlive>
  </router-view>
</template>

<script setup lang="ts">
import { ref } from 'vue';

// Define which route components to cache
const cachedViews = ref(['HomeView', 'DashboardView']);
</script>
```

---

## 12. Expose (Component Public Interface)

### Basic Expose

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue';

const count = ref(0);
const internalState = ref('private');

function increment() {
  count.value++;
}

function reset() {
  count.value = 0;
}

// Only expose specific properties/methods
defineExpose({
  count,
  increment,
  reset
});
</script>

<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue';
import ChildComponent from './ChildComponent.vue';

const childRef = ref<InstanceType<typeof ChildComponent> | null>(null);

function handleClick() {
  childRef.value?.increment();
  console.log(childRef.value?.count);
  // childRef.value?.internalState // Error: not exposed
}
</script>

<template>
  <ChildComponent ref="childRef" />
  <button @click="handleClick">Increment Child</button>
</template>
```

### Typed Expose

```vue
<script setup lang="ts">
import { ref } from 'vue';

const inputRef = ref<HTMLInputElement>();

function focus() {
  inputRef.value?.focus();
}

function blur() {
  inputRef.value?.blur();
}

function getValue(): string {
  return inputRef.value?.value ?? '';
}

// Type the exposed interface
defineExpose({
  focus,
  blur,
  getValue,
  el: inputRef
});
</script>

<template>
  <input ref="inputRef" type="text" />
</template>
```

---

## 13. Best Practices and Common Pitfalls

### Component Naming

```typescript
// Use PascalCase for component names
// Good
import UserProfile from './UserProfile.vue';
import BaseButton from './BaseButton.vue';

// Avoid single-word names (except App.vue)
// Bad: Button.vue, Input.vue (can conflict with HTML)
// Good: BaseButton.vue, AppInput.vue
```

### Props Best Practices

```vue
<script setup lang="ts">
// DO: Use detailed prop definitions
interface Props {
  items: Item[];
  loading?: boolean;
  errorMessage?: string;
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  errorMessage: ''
});

// DON'T: Mutate props
// props.items.push(newItem); // BAD!

// DO: Emit events or copy
const localItems = ref([...props.items]);
const emit = defineEmits<{
  (e: 'update:items', items: Item[]): void;
}>();

function addItem(item: Item) {
  emit('update:items', [...props.items, item]);
}
</script>
```

### Avoiding Common Pitfalls

```vue
<script setup lang="ts">
import { ref, reactive, watch, computed } from 'vue';

// PITFALL 1: Losing reactivity by destructuring
const props = defineProps<{ user: User }>();
// Bad: const { name } = props.user; // loses reactivity
// Good: use props.user.name directly or toRefs

// PITFALL 2: Using reactive() for primitives
// Bad: const count = reactive(0);
// Good:
const count = ref(0);

// PITFALL 3: Forgetting .value with refs in script
const message = ref('Hello');
// Bad: console.log(message);
// Good:
console.log(message.value);

// PITFALL 4: Mutating computed values
const doubled = computed(() => count.value * 2);
// Bad: doubled.value = 10; // Error!

// PITFALL 5: Async in setup without proper handling
// Bad: Direct await without Suspense
// const data = await fetchData();

// Good: Use lifecycle hooks or composables
import { onMounted } from 'vue';

const data = ref<Data | null>(null);
onMounted(async () => {
  data.value = await fetchData();
});
</script>
```

### Performance Best Practices

```vue
<script setup lang="ts">
import { shallowRef, markRaw, computed } from 'vue';

// Use shallowRef for large objects that don't need deep reactivity
const largeData = shallowRef<LargeObject[]>([]);

// Use markRaw for objects that should never be reactive
const staticConfig = markRaw({
  apiUrl: 'https://api.example.com',
  timeout: 5000
});

// Avoid expensive computations without memoization
// Bad: Computed in template
// <div>{{ items.filter(...).map(...) }}</div>

// Good: Use computed property
const processedItems = computed(() =>
  items.value.filter(...).map(...)
);

// Use v-once for static content
// <div v-once>{{ staticContent }}</div>

// Use v-memo for expensive list rendering
// <div v-for="item in list" :key="item.id" v-memo="[item.id, item.selected]">
</script>
```

### Component Organization

```
components/
  base/           # Reusable base components
    BaseButton.vue
    BaseInput.vue
    BaseModal.vue
  features/       # Feature-specific components
    user/
      UserProfile.vue
      UserSettings.vue
    dashboard/
      DashboardChart.vue
      DashboardStats.vue
  layout/         # Layout components
    TheHeader.vue
    TheSidebar.vue
    TheFooter.vue
```

### Composables for Reusable Logic

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
    reset
  };
}
```

```vue
<!-- Usage in component -->
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter';

const { count, doubled, increment, decrement, reset } = useCounter(10);
</script>
```

---

## Quick Reference

| Feature | Syntax |
|---------|--------|
| Props | `defineProps<Props>()` |
| Emits | `defineEmits<Emits>()` |
| v-model | `defineModel<T>()` |
| Slots | `defineSlots<Slots>()` |
| Expose | `defineExpose({ ... })` |
| Options | `defineOptions({ inheritAttrs: false })` |
| Provide | `provide(key, value)` |
| Inject | `inject(key, defaultValue?)` |
| Attrs | `useAttrs()` |
| Slots access | `useSlots()` |
