# Vue 3 (2025)

> **Last updated**: January 2026
> **Versions covered**: 3.4+
> **Purpose**: Progressive JavaScript framework for building UIs

---

## Philosophy (2025-2026)

Vue 3 with the **Composition API** is the standard for modern Vue development, offering better TypeScript support and code organization.

**Key philosophical shifts:**
- **Composition API default** — `<script setup>` is the standard
- **Logic by feature** — Group related code together
- **Composables for reuse** — Extract and share logic
- **TypeScript-first** — Full type inference
- **Pinia for state** — Official state management
- **Vapor mode coming** — No virtual DOM option

---

## TL;DR

- Use `<script setup>` for all new components
- Use `ref` for primitives, `reactive` for objects
- Extract reusable logic into composables
- Use Pinia for global state, not Vuex
- Use `provide`/`inject` for deep component communication
- Avoid prop drilling and event tunneling
- Don't use `v-if` and `v-for` on same element
- Use TypeScript for better DX

---

## Best Practices

### Component with script setup

```vue
<!-- components/UserProfile.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useUserStore } from '@/stores/user';
import type { User } from '@/types';

// Props with TypeScript
const props = defineProps<{
  userId: string;
  showBio?: boolean;
}>();

// Emits with TypeScript
const emit = defineEmits<{
  (e: 'update', user: User): void;
  (e: 'delete', id: string): void;
}>();

// Reactive state
const user = ref<User | null>(null);
const loading = ref(true);
const error = ref<string | null>(null);

// Computed
const displayName = computed(() => {
  if (!user.value) return '';
  return `${user.value.firstName} ${user.value.lastName}`;
});

// Store
const userStore = useUserStore();

// Lifecycle
onMounted(async () => {
  try {
    user.value = await userStore.fetchUser(props.userId);
  } catch (e) {
    error.value = 'Failed to load user';
  } finally {
    loading.value = false;
  }
});

// Methods
function handleUpdate() {
  if (user.value) {
    emit('update', user.value);
  }
}
</script>

<template>
  <div class="user-profile">
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <template v-else-if="user">
      <h2>{{ displayName }}</h2>
      <p v-if="showBio">{{ user.bio }}</p>
      <button @click="handleUpdate">Update</button>
    </template>
  </div>
</template>
```

### Composables (Reusable Logic)

```typescript
// composables/useApi.ts
import { ref, type Ref } from 'vue';

interface UseApiReturn<T> {
  data: Ref<T | null>;
  loading: Ref<boolean>;
  error: Ref<string | null>;
  execute: () => Promise<void>;
}

export function useApi<T>(fetcher: () => Promise<T>): UseApiReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>;
  const loading = ref(false);
  const error = ref<string | null>(null);

  async function execute() {
    loading.value = true;
    error.value = null;
    try {
      data.value = await fetcher();
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'An error occurred';
    } finally {
      loading.value = false;
    }
  }

  return { data, loading, error, execute };
}

// Usage in component
const { data: users, loading, error, execute } = useApi(() =>
  fetch('/api/users').then(r => r.json())
);

onMounted(execute);
```

```typescript
// composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue';

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const stored = localStorage.getItem(key);
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>;

  watch(
    data,
    (newValue) => {
      localStorage.setItem(key, JSON.stringify(newValue));
    },
    { deep: true }
  );

  return data;
}
```

```typescript
// composables/useDebounce.ts
import { ref, watch, type Ref } from 'vue';

export function useDebounce<T>(value: Ref<T>, delay: number = 300): Ref<T> {
  const debouncedValue = ref(value.value) as Ref<T>;

  let timeout: ReturnType<typeof setTimeout>;

  watch(value, (newValue) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => {
      debouncedValue.value = newValue;
    }, delay);
  });

  return debouncedValue;
}
```

### Pinia Store

```typescript
// stores/user.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import type { User } from '@/types';

export const useUserStore = defineStore('user', () => {
  // State
  const currentUser = ref<User | null>(null);
  const users = ref<User[]>([]);
  const loading = ref(false);

  // Getters
  const isAuthenticated = computed(() => currentUser.value !== null);
  const userCount = computed(() => users.value.length);

  // Actions
  async function fetchUser(id: string): Promise<User> {
    loading.value = true;
    try {
      const response = await fetch(`/api/users/${id}`);
      const user = await response.json();
      return user;
    } finally {
      loading.value = false;
    }
  }

  async function login(email: string, password: string) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    currentUser.value = await response.json();
  }

  function logout() {
    currentUser.value = null;
  }

  return {
    // State
    currentUser,
    users,
    loading,
    // Getters
    isAuthenticated,
    userCount,
    // Actions
    fetchUser,
    login,
    logout,
  };
});
```

### Provide/Inject (Deep Props)

```typescript
// Parent component
<script setup lang="ts">
import { provide, ref } from 'vue';

const theme = ref('light');
const toggleTheme = () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light';
};

// Provide to all descendants
provide('theme', theme);
provide('toggleTheme', toggleTheme);
</script>

// Deep child component (no prop drilling!)
<script setup lang="ts">
import { inject, type Ref } from 'vue';

const theme = inject<Ref<string>>('theme');
const toggleTheme = inject<() => void>('toggleTheme');
</script>

<template>
  <div :class="theme">
    <button @click="toggleTheme">Toggle Theme</button>
  </div>
</template>
```

### Form Handling

```vue
<script setup lang="ts">
import { ref, computed } from 'vue';

interface FormData {
  email: string;
  password: string;
  remember: boolean;
}

const form = ref<FormData>({
  email: '',
  password: '',
  remember: false,
});

const errors = ref<Partial<FormData>>({});

const isValid = computed(() => {
  return form.value.email.includes('@') && form.value.password.length >= 8;
});

function validate() {
  errors.value = {};
  if (!form.value.email.includes('@')) {
    errors.value.email = 'Invalid email';
  }
  if (form.value.password.length < 8) {
    errors.value.password = 'Password must be at least 8 characters';
  }
  return Object.keys(errors.value).length === 0;
}

async function handleSubmit() {
  if (!validate()) return;

  // Submit form
  console.log('Submitting:', form.value);
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <div>
      <input v-model="form.email" type="email" placeholder="Email" />
      <span v-if="errors.email" class="error">{{ errors.email }}</span>
    </div>

    <div>
      <input v-model="form.password" type="password" placeholder="Password" />
      <span v-if="errors.password" class="error">{{ errors.password }}</span>
    </div>

    <label>
      <input v-model="form.remember" type="checkbox" />
      Remember me
    </label>

    <button type="submit" :disabled="!isValid">Login</button>
  </form>
</template>
```

### Watchers

```typescript
import { ref, watch, watchEffect } from 'vue';

const searchQuery = ref('');
const results = ref([]);

// Watch specific value
watch(searchQuery, async (newQuery, oldQuery) => {
  if (newQuery.length > 2) {
    results.value = await searchApi(newQuery);
  }
});

// Watch with options
watch(
  searchQuery,
  async (newQuery) => {
    results.value = await searchApi(newQuery);
  },
  { immediate: true, debounce: 300 }
);

// Watch multiple sources
watch(
  [searchQuery, sortOrder],
  ([query, sort]) => {
    fetchResults(query, sort);
  }
);

// watchEffect - auto-tracks dependencies
watchEffect(async () => {
  // Automatically re-runs when searchQuery.value changes
  if (searchQuery.value) {
    results.value = await searchApi(searchQuery.value);
  }
});
```

---

## Anti-Patterns

### ❌ Using v-if and v-for Together

**Why it's bad**: Performance issues, extra computations.

```vue
<!-- ❌ DON'T -->
<li v-for="user in users" v-if="user.isActive" :key="user.id">
  {{ user.name }}
</li>

<!-- ✅ DO — Use computed -->
<script setup>
const activeUsers = computed(() => users.value.filter(u => u.isActive));
</script>

<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

### ❌ Mutating Props

**Why it's bad**: Breaks one-way data flow.

```vue
<!-- ❌ DON'T -->
<script setup>
const props = defineProps<{ count: number }>();
props.count++; // Error! Props are readonly
</script>

<!-- ✅ DO — Emit event to parent -->
<script setup>
const props = defineProps<{ count: number }>();
const emit = defineEmits<{ (e: 'update:count', value: number): void }>();

function increment() {
  emit('update:count', props.count + 1);
}
</script>
```

### ❌ Prop Drilling

**Why it's bad**: Complex, hard to maintain.

```vue
<!-- ❌ DON'T — Pass through multiple levels -->
<Grandparent :theme="theme">
  <Parent :theme="theme">
    <Child :theme="theme" />  <!-- 3 levels deep! -->
  </Parent>
</Grandparent>

<!-- ✅ DO — Use provide/inject -->
<script setup>
provide('theme', theme);
</script>

<!-- In deeply nested child -->
<script setup>
const theme = inject('theme');
</script>
```

### ❌ Side Effects in Computed

**Why it's bad**: Computed should be pure.

```typescript
// ❌ DON'T
const fullName = computed(() => {
  console.log('Computing...'); // Side effect!
  sendAnalytics();             // Side effect!
  return `${firstName.value} ${lastName.value}`;
});

// ✅ DO — Keep computed pure
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`;
});

// Use watch for side effects
watch(fullName, (newName) => {
  sendAnalytics(newName);
});
```

### ❌ Overusing $refs

**Why it's bad**: Bypasses Vue's reactivity.

```vue
<!-- ❌ DON'T — Use refs for data flow -->
<script setup>
const childRef = ref();
function getData() {
  return childRef.value.someData; // Direct access
}
</script>

<!-- ✅ DO — Use props and emits -->
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 3.3 | 2023 | Generic components, defineSlots |
| 3.4 | Dec 2023 | defineModel, v-bind shorthand |
| 3.5 | 2024 | Reactive props destructure, useTemplateRef |
| Vapor | Coming | No virtual DOM mode |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Reactive primitive | `const count = ref(0)` |
| Reactive object | `const state = reactive({...})` |
| Computed | `const double = computed(() => count.value * 2)` |
| Watch | `watch(source, callback)` |
| Props | `defineProps<{ name: string }>()` |
| Emits | `defineEmits<{ (e: 'click'): void }>()` |
| Lifecycle | `onMounted`, `onUnmounted`, etc. |
| Template ref | `const el = ref<HTMLElement>()` |
| Provide | `provide('key', value)` |
| Inject | `const value = inject('key')` |

| ref vs reactive | Use Case |
|-----------------|----------|
| `ref` | Primitives (string, number, boolean) |
| `ref` | Values that may be reassigned |
| `reactive` | Objects that won't be reassigned |
| `reactive` | Deep reactivity needed |

---

## Resources

- [Official Vue 3 Documentation](https://vuejs.org/)
- [Composition API FAQ](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Vue 3 Best Practices](https://medium.com/@ignatovich.dm/vue-3-best-practices-cb0a6e281ef4)
- [Pinia Documentation](https://pinia.vuejs.org/)
- [Why Composition API in 2025](https://www.zignuts.com/blog/vue-composition-api-benefits-2025)
- [Vue Patterns and Anti-Patterns](https://www.binarcode.com/blog/3-anti-patterns-to-avoid-in-vuejs)
