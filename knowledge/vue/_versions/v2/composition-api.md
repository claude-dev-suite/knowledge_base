# Vue 2 → Composition API Delta

## Not Available in Vue 2 (Native)

- `<script setup>` syntax - Vue 3+ only
- `ref()`, `reactive()`, `computed()` - Requires @vue/composition-api plugin
- `watch()`, `watchEffect()` - Requires plugin
- `defineProps()`, `defineEmits()`, `defineExpose()` - Vue 3+ only
- `Teleport` component - Vue 3+ only
- `Suspense` component - Vue 3+ only
- Multiple root elements - Vue 3+ only

## Vue 2 with Composition API Plugin

Vue 2.7+ has built-in Composition API. For earlier versions, use `@vue/composition-api`:

```bash
npm install @vue/composition-api
```

```javascript
// main.js
import Vue from 'vue'
import VueCompositionAPI from '@vue/composition-api'

Vue.use(VueCompositionAPI)
```

## Syntax Differences

### Component Definition

```vue
<!-- Vue 3 - script setup -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<!-- Vue 2 - Options API (standard) -->
<script>
export default {
  data() {
    return {
      count: 0
    }
  },
  computed: {
    doubled() {
      return this.count * 2
    }
  }
}
</script>

<!-- Vue 2.7+ or with plugin - setup() function -->
<script>
import { ref, computed } from '@vue/composition-api'
// or from 'vue' in Vue 2.7+

export default {
  setup() {
    const count = ref(0)
    const doubled = computed(() => count.value * 2)

    return { count, doubled }
  }
}
</script>
```

### Props Definition

```vue
<!-- Vue 3 -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  count?: number
}>()
</script>

<!-- Vue 2 -->
<script>
export default {
  props: {
    title: {
      type: String,
      required: true
    },
    count: {
      type: Number,
      default: 0
    }
  }
}
</script>
```

### Emits

```vue
<!-- Vue 3 -->
<script setup>
const emit = defineEmits(['update', 'delete'])
emit('update', newValue)
</script>

<!-- Vue 2 -->
<script>
export default {
  methods: {
    handleUpdate(value) {
      this.$emit('update', value)
    }
  }
}
</script>
```

### v-model

```vue
<!-- Vue 3 - multiple v-model -->
<UserForm v-model:name="name" v-model:email="email" />

<!-- Vue 2 - single v-model + .sync -->
<UserForm v-model="name" :email.sync="email" />

<!-- Vue 3 child -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<!-- Vue 2 child -->
<script>
export default {
  props: ['value'],  // v-model uses 'value' by default
  methods: {
    updateValue(val) {
      this.$emit('input', val)  // v-model listens to 'input'
    }
  }
}
</script>
```

### Lifecycle Hooks

```vue
<!-- Vue 3 -->
<script setup>
import { onMounted, onUnmounted, onBeforeMount } from 'vue'

onBeforeMount(() => console.log('before mount'))
onMounted(() => console.log('mounted'))
onUnmounted(() => console.log('unmounted'))
</script>

<!-- Vue 2 - Options API -->
<script>
export default {
  beforeMount() {
    console.log('before mount')
  },
  mounted() {
    console.log('mounted')
  },
  beforeDestroy() {  // Note: "destroyed" not "unmounted"
    console.log('before destroy')
  },
  destroyed() {
    console.log('destroyed')
  }
}
</script>
```

### App Creation

```javascript
// Vue 3
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.use(router)
app.use(store)
app.mount('#app')

// Vue 2
import Vue from 'vue'
import App from './App.vue'

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```

### Global Components

```javascript
// Vue 3
app.component('MyComponent', MyComponent)

// Vue 2
Vue.component('MyComponent', MyComponent)
```

### Filters (Removed in Vue 3)

```vue
<!-- Vue 2 - Filters supported -->
<template>
  <p>{{ price | currency }}</p>
</template>

<script>
export default {
  filters: {
    currency(value) {
      return '$' + value.toFixed(2)
    }
  }
}
</script>

<!-- Vue 3 - Use computed or methods instead -->
<script setup>
import { computed } from 'vue'

const props = defineProps(['price'])
const formattedPrice = computed(() => '$' + props.price.toFixed(2))
</script>
```

## Still Current in Vue 2

- Options API (data, computed, methods, watch)
- Template syntax (v-if, v-for, v-bind, v-on)
- Component props and events
- Slots (named and scoped)
- Mixins (discouraged in Vue 3, use composables)
- Vuex for state management
- Vue Router

## Recommendations for Vue 2 Users

1. **Use Vue 2.7** for built-in Composition API support
2. **Options API** remains fully supported and valid
3. **Consider Pinia** over Vuex for new stores (works with Vue 2.7+)
4. **Plan migration** to Vue 3 for new projects
5. **Use @vue/composition-api** if stuck on Vue 2.6

## Migration Path

1. Upgrade to Vue 2.7 first (has Composition API built-in)
2. Migrate to Vue 3 compatible syntax gradually
3. Replace filters with computed properties
4. Update v-model usage (value/input → modelValue/update:modelValue)
5. Replace .sync with v-model:prop
6. Update lifecycle hook names (destroyed → unmounted)
7. Migrate from new Vue() to createApp()

