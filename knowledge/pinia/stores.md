# Pinia Store Reference

Comprehensive documentation for Pinia state management.

**Official Documentation:** https://pinia.vuejs.org/core-concepts/

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Defining Stores](#defining-stores)
3. [State](#state)
4. [Getters](#getters)
5. [Actions](#actions)
6. [Using Stores](#using-stores)
7. [Plugins](#plugins)
8. [Testing](#testing)

---

## Getting Started

### Installation

```bash
npm install pinia
```

### Setup

```typescript
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const pinia = createPinia()
const app = createApp(App)

app.use(pinia)
app.mount('#app')
```

---

## Defining Stores

### Option Store (Options API Style)

```typescript
// stores/counter.ts
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  // State
  state: () => ({
    count: 0,
    name: 'Counter',
    items: [] as string[],
  }),

  // Getters (computed)
  getters: {
    doubleCount: (state) => state.count * 2,
    doubleCountPlusOne(): number {
      return this.doubleCount + 1
    },
  },

  // Actions (methods)
  actions: {
    increment() {
      this.count++
    },
    async fetchItems() {
      this.items = await api.fetchItems()
    },
  },
})
```

### Setup Store (Composition API Style)

```typescript
// stores/counter.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0)
  const name = ref('Counter')
  const items = ref<string[]>([])

  // Getters
  const doubleCount = computed(() => count.value * 2)
  const doubleCountPlusOne = computed(() => doubleCount.value + 1)

  // Actions
  function increment() {
    count.value++
  }

  async function fetchItems() {
    items.value = await api.fetchItems()
  }

  return {
    // State
    count,
    name,
    items,
    // Getters
    doubleCount,
    doubleCountPlusOne,
    // Actions
    increment,
    fetchItems,
  }
})
```

---

## State

### Defining State

```typescript
// Option Store
state: () => ({
  // Primitives
  count: 0,
  name: 'John',
  isActive: true,

  // Arrays
  items: [] as Item[],

  // Objects
  user: null as User | null,
  settings: {
    theme: 'dark',
    language: 'en',
  },
})

// Setup Store
const count = ref(0)
const name = ref('John')
const items = ref<Item[]>([])
const user = ref<User | null>(null)
```

### Accessing State

```typescript
const store = useCounterStore()

// Direct access (reactive)
console.log(store.count)
console.log(store.name)

// In template
// {{ store.count }}
```

### Modifying State

```typescript
const store = useCounterStore()

// Direct mutation
store.count++
store.name = 'Jane'

// Multiple mutations with $patch
store.$patch({
  count: store.count + 1,
  name: 'Jane',
})

// Function patch (for arrays, complex logic)
store.$patch((state) => {
  state.count++
  state.items.push({ id: 1 })
  state.items = state.items.filter(item => item.active)
})
```

### Replacing State

```typescript
// Replace entire state
store.$state = {
  count: 0,
  name: 'Reset',
  items: [],
}
```

### Resetting State

```typescript
// Reset to initial state
store.$reset()

// Note: $reset() only works with Option Stores
// For Setup Stores, create a reset action:
function $reset() {
  count.value = 0
  name.value = 'Counter'
  items.value = []
}
```

### State Subscription

```typescript
// Watch state changes
store.$subscribe((mutation, state) => {
  console.log('Mutation type:', mutation.type) // 'direct' | 'patch object' | 'patch function'
  console.log('Store ID:', mutation.storeId)
  console.log('New state:', state)

  // Persist to localStorage
  localStorage.setItem('counter', JSON.stringify(state))
})

// Detached subscription (survives component unmount)
store.$subscribe(callback, { detached: true })
```

---

## Getters

### Basic Getters

```typescript
// Option Store
getters: {
  // Arrow function with state
  doubleCount: (state) => state.count * 2,

  // Regular function for accessing other getters via `this`
  doubleCountPlusOne(): number {
    return this.doubleCount + 1
  },
}

// Setup Store
const doubleCount = computed(() => count.value * 2)
const doubleCountPlusOne = computed(() => doubleCount.value + 1)
```

### Getters with Arguments

```typescript
// Option Store
getters: {
  getUserById: (state) => {
    return (userId: number) => state.users.find(u => u.id === userId)
  },
}

// Setup Store
const getUserById = computed(() => {
  return (userId: number) => users.value.find(u => u.id === userId)
})

// Usage
const user = store.getUserById(1)
```

### Accessing Other Stores' Getters

```typescript
import { useOtherStore } from './other'

// Option Store
getters: {
  combined(): string {
    const otherStore = useOtherStore()
    return this.name + otherStore.name
  },
}

// Setup Store
const combined = computed(() => {
  const otherStore = useOtherStore()
  return name.value + otherStore.name
})
```

---

## Actions

### Basic Actions

```typescript
// Option Store
actions: {
  increment() {
    this.count++
  },

  incrementBy(amount: number) {
    this.count += amount
  },

  reset() {
    this.count = 0
    this.name = 'Counter'
  },
}

// Setup Store
function increment() {
  count.value++
}

function incrementBy(amount: number) {
  count.value += amount
}
```

### Async Actions

```typescript
// Option Store
actions: {
  async fetchUser(userId: number) {
    try {
      this.loading = true
      this.user = await api.fetchUser(userId)
    } catch (error) {
      this.error = error
    } finally {
      this.loading = false
    }
  },

  async fetchUserOrders() {
    // Wait for user to be loaded
    if (!this.user) {
      await this.fetchUser(1)
    }
    this.orders = await api.fetchOrders(this.user.id)
  },
}

// Setup Store
async function fetchUser(userId: number) {
  try {
    loading.value = true
    user.value = await api.fetchUser(userId)
  } catch (err) {
    error.value = err
  } finally {
    loading.value = false
  }
}
```

### Accessing Other Stores

```typescript
import { useAuthStore } from './auth'

// Option Store
actions: {
  async fetchUserData() {
    const authStore = useAuthStore()
    if (!authStore.isAuthenticated) {
      throw new Error('Not authenticated')
    }
    this.userData = await api.fetchUserData(authStore.token)
  },
}

// Setup Store
async function fetchUserData() {
  const authStore = useAuthStore()
  if (!authStore.isAuthenticated) {
    throw new Error('Not authenticated')
  }
  userData.value = await api.fetchUserData(authStore.token)
}
```

### Action Subscription

```typescript
// Subscribe to action calls
const unsubscribe = store.$onAction(
  ({
    name,      // Action name
    store,     // Store instance
    args,      // Arguments passed to action
    after,     // Hook after action resolves
    onError,   // Hook on action error
  }) => {
    console.log(`Action "${name}" called with args:`, args)

    after((result) => {
      console.log(`Action "${name}" finished with result:`, result)
    })

    onError((error) => {
      console.error(`Action "${name}" failed:`, error)
    })
  }
)

// Unsubscribe
unsubscribe()

// Detached subscription
store.$onAction(callback, true)
```

---

## Using Stores

### In Components

```vue
<script setup lang="ts">
import { useCounterStore } from '@/stores/counter'
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// Direct access (not reactive for primitives when destructured)
// const { count } = store // ❌ Not reactive

// Use storeToRefs for reactive destructuring
const { count, name } = storeToRefs(store)

// Actions can be destructured directly
const { increment, fetchItems } = store
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Name: {{ name }}</p>
    <p>Double: {{ store.doubleCount }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>
```

### storeToRefs

```typescript
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// ✅ Correct: refs are reactive
const { count, name, items } = storeToRefs(store)

// ✅ Correct: getters as computed refs
const { doubleCount } = storeToRefs(store)

// ✅ Actions can be destructured directly (not reactive)
const { increment, fetchItems } = store
```

### In Options API

```vue
<script>
import { mapStores, mapState, mapActions } from 'pinia'
import { useCounterStore } from '@/stores/counter'

export default {
  computed: {
    // Access store as this.counterStore
    ...mapStores(useCounterStore),

    // Map state properties
    ...mapState(useCounterStore, ['count', 'name']),

    // Map with aliases
    ...mapState(useCounterStore, {
      myCount: 'count',
      double: 'doubleCount',
    }),
  },

  methods: {
    ...mapActions(useCounterStore, ['increment']),
  },
}
</script>
```

### Outside Components

```typescript
// In router guards, plugins, etc.
import { useCounterStore } from '@/stores/counter'
import { createPinia, setActivePinia } from 'pinia'

// Ensure pinia is created
const pinia = createPinia()
setActivePinia(pinia)

// Now you can use stores
const store = useCounterStore()

// Or pass pinia instance
const store = useCounterStore(pinia)
```

---

## Plugins

### Creating Plugins

```typescript
import { PiniaPluginContext } from 'pinia'

export function myPiniaPlugin(context: PiniaPluginContext) {
  const { store, app, pinia, options } = context

  // Add property to every store
  store.hello = 'world'

  // Add method to every store
  store.greet = () => console.log('Hello!')

  // Watch state changes
  store.$subscribe((mutation) => {
    console.log('State changed in', mutation.storeId)
  })

  // Return properties to add to store
  return {
    secret: 'my secret',
  }
}

// Register plugin
const pinia = createPinia()
pinia.use(myPiniaPlugin)
```

### Persistence Plugin

```typescript
import { PiniaPluginContext } from 'pinia'

export function persistPlugin({ store }: PiniaPluginContext) {
  // Load persisted state
  const persisted = localStorage.getItem(store.$id)
  if (persisted) {
    store.$patch(JSON.parse(persisted))
  }

  // Save state on change
  store.$subscribe((mutation, state) => {
    localStorage.setItem(store.$id, JSON.stringify(state))
  })
}

// Or use pinia-plugin-persistedstate
// npm install pinia-plugin-persistedstate
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)

// In store definition
export const useStore = defineStore('store', {
  state: () => ({ count: 0 }),
  persist: true, // Enable persistence
})

// With options
export const useStore = defineStore('store', {
  state: () => ({ count: 0, temp: '' }),
  persist: {
    key: 'my-store',
    storage: sessionStorage,
    paths: ['count'], // Only persist count
  },
})
```

### Typing Plugins

```typescript
import 'pinia'

declare module 'pinia' {
  export interface PiniaCustomProperties {
    hello: string
    greet: () => void
  }

  export interface PiniaCustomStateProperties<S> {
    secret: string
  }

  export interface DefineStoreOptionsBase<S, Store> {
    debounce?: Partial<Record<keyof StoreActions<Store>, number>>
  }
}
```

---

## Testing

### Setup

```typescript
import { setActivePinia, createPinia } from 'pinia'
import { useCounterStore } from '@/stores/counter'
import { beforeEach, describe, it, expect, vi } from 'vitest'

describe('Counter Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('initializes with count of 0', () => {
    const store = useCounterStore()
    expect(store.count).toBe(0)
  })

  it('increments count', () => {
    const store = useCounterStore()
    store.increment()
    expect(store.count).toBe(1)
  })

  it('computes doubleCount', () => {
    const store = useCounterStore()
    store.count = 5
    expect(store.doubleCount).toBe(10)
  })
})
```

### Testing Actions

```typescript
import { vi } from 'vitest'

it('fetches user data', async () => {
  const store = useUserStore()

  // Mock API
  vi.spyOn(api, 'fetchUser').mockResolvedValue({ id: 1, name: 'John' })

  await store.fetchUser(1)

  expect(store.user).toEqual({ id: 1, name: 'John' })
  expect(api.fetchUser).toHaveBeenCalledWith(1)
})

it('handles fetch error', async () => {
  const store = useUserStore()

  vi.spyOn(api, 'fetchUser').mockRejectedValue(new Error('Failed'))

  await store.fetchUser(1)

  expect(store.error).toBe('Failed')
  expect(store.user).toBeNull()
})
```

### Testing with Initial State

```typescript
import { createTestingPinia } from '@pinia/testing'

it('renders with initial state', () => {
  const wrapper = mount(Counter, {
    global: {
      plugins: [
        createTestingPinia({
          initialState: {
            counter: { count: 10 },
          },
        }),
      ],
    },
  })

  expect(wrapper.text()).toContain('10')
})
```

### Mocking Stores

```typescript
import { createTestingPinia } from '@pinia/testing'

it('mocks actions', () => {
  const wrapper = mount(Counter, {
    global: {
      plugins: [createTestingPinia()],
    },
  })

  const store = useCounterStore()

  // Actions are mocked by default
  wrapper.find('button').trigger('click')
  expect(store.increment).toHaveBeenCalledTimes(1)
})

// Stub actions to run real implementation
const pinia = createTestingPinia({
  stubActions: false,
})
```

---

## Best Practices

### Store Organization

```
stores/
├── index.ts         # Export all stores
├── counter.ts       # Counter store
├── user.ts          # User store
├── cart.ts          # Cart store
└── types.ts         # Shared types
```

### Naming Conventions

```typescript
// Store file: user.ts
// Store name: 'user'
// Composable: useUserStore

export const useUserStore = defineStore('user', {
  // ...
})
```

### Composition

```typescript
// Composing multiple stores
export function useCheckout() {
  const cartStore = useCartStore()
  const userStore = useUserStore()
  const paymentStore = usePaymentStore()

  async function checkout() {
    if (!userStore.isAuthenticated) {
      throw new Error('Please login')
    }

    const order = await paymentStore.processPayment(cartStore.items)
    cartStore.clear()

    return order
  }

  return {
    checkout,
    isReady: computed(() =>
      userStore.isAuthenticated && cartStore.items.length > 0
    ),
  }
}
```

---

## Quick Reference

| API | Description |
|-----|-------------|
| `defineStore(id, options)` | Create option store |
| `defineStore(id, setup)` | Create setup store |
| `storeToRefs(store)` | Extract reactive refs |
| `store.$state` | Access full state |
| `store.$patch(obj)` | Batch update |
| `store.$reset()` | Reset to initial |
| `store.$subscribe(fn)` | Watch state changes |
| `store.$onAction(fn)` | Watch actions |
| `mapState()` | Options API helper |
| `mapActions()` | Options API helper |
