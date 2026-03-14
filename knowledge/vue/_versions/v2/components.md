# Vue 2 → Components Delta

## Not Available in Vue 2

- Multiple root elements (fragments)
- `<Teleport>` component
- `<Suspense>` component (experimental in Vue 3)
- `emits` option declaration
- `defineComponent()` with proper inference

## Syntax Differences

### Root Element

```vue
<!-- Vue 3 - Multiple roots allowed -->
<template>
  <header>Header</header>
  <main>Content</main>
  <footer>Footer</footer>
</template>

<!-- Vue 2 - Single root required -->
<template>
  <div>
    <header>Header</header>
    <main>Content</main>
    <footer>Footer</footer>
  </div>
</template>
```

### Async Components

```javascript
// Vue 3
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() =>
  import('./components/MyComponent.vue')
)

const AsyncCompWithOptions = defineAsyncComponent({
  loader: () => import('./MyComponent.vue'),
  loadingComponent: LoadingComponent,
  errorComponent: ErrorComponent,
  delay: 200,
  timeout: 3000
})

// Vue 2
const AsyncComp = () => import('./components/MyComponent.vue')

const AsyncCompWithOptions = () => ({
  component: import('./MyComponent.vue'),
  loading: LoadingComponent,
  error: ErrorComponent,
  delay: 200,
  timeout: 3000
})
```

### Functional Components

```vue
<!-- Vue 3 - No functional attribute needed -->
<script setup>
const props = defineProps(['level'])
</script>

<template>
  <component :is="'h' + level">
    <slot />
  </component>
</template>

<!-- Vue 2 - Functional attribute -->
<template functional>
  <component :is="'h' + props.level">
    <slot />
  </component>
</template>

<script>
export default {
  functional: true,
  props: ['level']
}
</script>
```

### Emits Declaration

```vue
<!-- Vue 3 - Emits declaration required -->
<script setup>
const emit = defineEmits(['submit', 'cancel'])

// Or with validation
const emit = defineEmits({
  submit: (payload) => typeof payload === 'object',
  cancel: null
})
</script>

<!-- Vue 2 - No declaration needed -->
<script>
export default {
  methods: {
    handleSubmit() {
      this.$emit('submit', this.data)
    }
  }
}
</script>
```

### v-model Changes

```vue
<!-- Vue 3 - Child component -->
<script setup>
defineProps(['modelValue'])
defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="modelValue"
    @input="$emit('update:modelValue', $event.target.value)"
  />
</template>

<!-- Vue 2 - Child component -->
<script>
export default {
  props: ['value'],
  methods: {
    onInput(e) {
      this.$emit('input', e.target.value)
    }
  }
}
</script>

<template>
  <input :value="value" @input="onInput" />
</template>
```

### Teleport (Vue 3 Only)

```vue
<!-- Vue 3 -->
<template>
  <button @click="showModal = true">Open Modal</button>

  <Teleport to="body">
    <div v-if="showModal" class="modal">
      <p>Modal content</p>
      <button @click="showModal = false">Close</button>
    </div>
  </Teleport>
</template>

<!-- Vue 2 - Use portal-vue library -->
<template>
  <button @click="showModal = true">Open Modal</button>

  <portal to="modal-target">
    <div v-if="showModal" class="modal">
      <p>Modal content</p>
    </div>
  </portal>
</template>
```

### Attribute Inheritance

```vue
<!-- Vue 3 - $listeners merged into $attrs -->
<script setup>
import { useAttrs } from 'vue'
const attrs = useAttrs()
// attrs includes both attributes AND event listeners
</script>

<!-- Vue 2 - Separate $attrs and $listeners -->
<script>
export default {
  inheritAttrs: false,
  mounted() {
    console.log(this.$attrs)     // Non-prop attributes
    console.log(this.$listeners) // Event listeners (v-on)
  }
}
</script>

<!-- Vue 2 - Forwarding both -->
<template>
  <input v-bind="$attrs" v-on="$listeners" />
</template>

<!-- Vue 3 - $attrs includes listeners -->
<template>
  <input v-bind="$attrs" />
</template>
```

## Still Current in Vue 2

- Props validation
- Slots (default and named)
- Scoped slots
- Dynamic components (`<component :is>`)
- Keep-alive
- Transitions
- v-bind and v-on
- Provide/inject (different syntax)

## Recommendations for Vue 2 Users

1. **Wrap in single root** - Use wrapper div for multiple elements
2. **Use portal-vue** - For teleport-like functionality
3. **Manual $listeners** - Remember to forward both $attrs and $listeners
4. **value/input for v-model** - Default prop/event names

