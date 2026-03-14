# Vue 2 → Patterns Delta

## Not Available in Vue 2

- Composables (use Options API mixins instead)
- `<script setup>` patterns
- `defineExpose()` for component refs
- `useSlots()`, `useAttrs()` helpers

## Syntax Differences

### Logic Reuse: Composables vs Mixins

```javascript
// Vue 3 - Composable (recommended)
// composables/useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}

// Usage in component
<script setup>
import { useMouse } from '@/composables/useMouse'
const { x, y } = useMouse()
</script>


// Vue 2 - Mixin
// mixins/mouseMixin.js
export const mouseMixin = {
  data() {
    return {
      mouseX: 0,
      mouseY: 0
    }
  },
  methods: {
    updateMouse(event) {
      this.mouseX = event.pageX
      this.mouseY = event.pageY
    }
  },
  mounted() {
    window.addEventListener('mousemove', this.updateMouse)
  },
  beforeDestroy() {
    window.removeEventListener('mousemove', this.updateMouse)
  }
}

// Usage in component
<script>
import { mouseMixin } from '@/mixins/mouseMixin'

export default {
  mixins: [mouseMixin],
  // mouseX and mouseY available via this
}
</script>
```

### Provide/Inject

```javascript
// Vue 3
// Parent
import { provide, ref } from 'vue'

const theme = ref('dark')
provide('theme', theme)  // Reactive!

// Child
import { inject } from 'vue'
const theme = inject('theme')


// Vue 2
// Parent
export default {
  provide() {
    return {
      theme: this.theme  // NOT reactive by default
    }
  },
  data() {
    return {
      theme: 'dark'
    }
  }
}

// Vue 2 - Making provide reactive
export default {
  provide() {
    return {
      getTheme: () => this.theme  // Pass getter function
    }
  }
}

// Child (Vue 2)
export default {
  inject: ['theme'],
  // or with default
  inject: {
    theme: {
      default: 'light'
    }
  }
}
```

### State Management Pattern

```javascript
// Vue 3 - Pinia (recommended)
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: {
    double: (state) => state.count * 2
  },
  actions: {
    increment() {
      this.count++
    }
  }
})

// Vue 3 - Composition API store
<script setup>
import { useCounterStore } from '@/stores/counter'
const store = useCounterStore()
</script>


// Vue 2 - Vuex
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
  state: {
    count: 0
  },
  getters: {
    double: state => state.count * 2
  },
  mutations: {
    INCREMENT(state) {
      state.count++
    }
  },
  actions: {
    increment({ commit }) {
      commit('INCREMENT')
    }
  }
})

// Vue 2 - Usage with mapHelpers
<script>
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
  computed: {
    ...mapState(['count']),
    ...mapGetters(['double'])
  },
  methods: {
    ...mapActions(['increment'])
  }
}
</script>
```

### Component Communication

```vue
<!-- Vue 3 - Event bus alternative with mitt -->
<script setup>
import { inject } from 'vue'
const emitter = inject('emitter')

emitter.emit('user-logged-in', user)
emitter.on('user-logged-in', handleLogin)
</script>

<!-- Vue 2 - Event bus pattern -->
<script>
// eventBus.js
import Vue from 'vue'
export const EventBus = new Vue()

// Component A - Emit
import { EventBus } from '@/eventBus'
EventBus.$emit('user-logged-in', user)

// Component B - Listen
import { EventBus } from '@/eventBus'
export default {
  mounted() {
    EventBus.$on('user-logged-in', this.handleLogin)
  },
  beforeDestroy() {
    EventBus.$off('user-logged-in', this.handleLogin)
  }
}
</script>
```

### Watchers

```javascript
// Vue 3 - Composition API
import { watch, watchEffect } from 'vue'

// Watch specific source
watch(count, (newVal, oldVal) => {
  console.log(`count changed from ${oldVal} to ${newVal}`)
})

// Watch multiple sources
watch([firstName, lastName], ([newFirst, newLast]) => {
  console.log(`Name: ${newFirst} ${newLast}`)
})

// Immediate watch
watch(source, callback, { immediate: true })

// watchEffect - auto-tracks dependencies
watchEffect(() => {
  console.log(`Count is: ${count.value}`)
})


// Vue 2 - Options API
export default {
  data() {
    return { count: 0, firstName: '', lastName: '' }
  },
  watch: {
    count(newVal, oldVal) {
      console.log(`count changed from ${oldVal} to ${newVal}`)
    },
    // Immediate watcher
    count: {
      handler(newVal) {
        console.log(`count is now ${newVal}`)
      },
      immediate: true
    },
    // Deep watcher
    someObject: {
      handler(newVal) {
        console.log('Object changed')
      },
      deep: true
    }
  }
}
```

## Still Current in Vue 2

- Options API patterns (data, computed, methods)
- Props drilling
- Slots for content distribution
- Scoped slots for render props pattern
- Custom directives
- Render functions

## Recommendations for Vue 2 Users

1. **Mixins** - Use for logic reuse (but be aware of naming conflicts)
2. **Vuex** - Standard state management
3. **Event bus** - Use sparingly for cross-component communication
4. **Scoped slots** - For renderless components pattern
5. **Plan composable migration** - Structure mixins to ease future migration

## Mixin Issues (Why Vue 3 Uses Composables)

```javascript
// Vue 2 - Mixin naming conflicts
const mixinA = {
  data() {
    return { count: 0 }  // Naming conflict!
  }
}

const mixinB = {
  data() {
    return { count: 100 }  // Which count wins?
  }
}

// Vue 3 - Composables are explicit
const { count: countA } = useFeatureA()
const { count: countB } = useFeatureB()
// No conflicts - you control the names
```

