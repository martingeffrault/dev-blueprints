# Nuxt 3 (2025)

> **Last updated**: January 2026
> **Versions covered**: 3.x, Nuxt 4 preview
> **Purpose**: Vue.js meta-framework for SSR, SSG, and hybrid rendering

---

## Philosophy (2025-2026)

Nuxt 3 is the **batteries-included Vue framework** built on Nitro server engine and Vite, with automatic imports, file-based routing, and hybrid rendering.

**Key philosophical shifts:**
- **Auto-imports everywhere** — Components, composables, utils
- **Hybrid rendering** — SSR, SSG, ISR, SWR per-route
- **Nitro server engine** — Edge-ready, platform-agnostic
- **TypeScript first** — Full type inference
- **Convention over configuration** — Sensible defaults
- **Nuxt 4 compatibility mode** — Gradual migration path

---

## TL;DR

- Use `<script setup>` in all components
- Prefer composables over plugins
- Use `useFetch`/`useAsyncData` for data fetching
- Don't use server middleware — use explicit utility functions
- Use Pinia for global state
- Use `useNuxtApp()` for runtime context
- File-based routing with layouts
- Don't use `document.title` — use `useHead()`

---

## Best Practices

### Project Structure

```
my-nuxt-app/
├── app/                      # Nuxt 4 structure (future-ready)
│   ├── components/
│   │   ├── global/           # Auto-imported globally
│   │   │   └── AppHeader.vue
│   │   └── features/
│   │       └── UserCard.vue
│   ├── composables/          # Auto-imported composables
│   │   ├── useAuth.ts
│   │   └── useApi.ts
│   ├── layouts/
│   │   ├── default.vue
│   │   └── dashboard.vue
│   ├── pages/
│   │   ├── index.vue
│   │   ├── about.vue
│   │   └── users/
│   │       ├── index.vue
│   │       └── [id].vue
│   ├── middleware/           # Route middleware
│   │   └── auth.ts
│   └── plugins/              # Runtime plugins
│       └── api.ts
├── server/
│   ├── api/                  # API routes
│   │   ├── users/
│   │   │   ├── index.get.ts
│   │   │   ├── index.post.ts
│   │   │   └── [id].get.ts
│   │   └── auth/
│   │       └── login.post.ts
│   ├── utils/                # Server utilities
│   │   └── db.ts
│   └── middleware/           # Server middleware (use sparingly!)
├── public/
├── stores/                   # Pinia stores
│   └── user.ts
├── types/
│   └── index.ts
├── nuxt.config.ts
└── package.json
```

### Data Fetching with useFetch

```vue
<script setup lang="ts">
// Simple data fetching with automatic SSR/caching
const { data: users, pending, error, refresh } = await useFetch('/api/users');

// With options
const { data: posts } = await useFetch('/api/posts', {
  query: { page: 1, limit: 10 },
  pick: ['id', 'title', 'excerpt'],  // Only pick specific fields
  transform: (posts) => posts.map(p => ({
    ...p,
    formattedDate: new Date(p.createdAt).toLocaleDateString(),
  })),
});

// Lazy fetch (doesn't block navigation)
const { data: comments, pending: loadingComments } = useLazyFetch('/api/comments');
</script>

<template>
  <div>
    <div v-if="pending">Loading...</div>
    <div v-else-if="error">{{ error.message }}</div>
    <div v-else>
      <UserCard v-for="user in users" :key="user.id" :user="user" />
    </div>
    <button @click="refresh()">Refresh</button>
  </div>
</template>
```

### useAsyncData for Custom Logic

```vue
<script setup lang="ts">
import type { User } from '~/types';

const route = useRoute();

// useAsyncData for more control
const { data: user, error } = await useAsyncData<User>(
  `user-${route.params.id}`,  // Unique key for caching
  () => $fetch(`/api/users/${route.params.id}`),
  {
    // Re-fetch when params change
    watch: [() => route.params.id],
  }
);

// Parallel data fetching
const [{ data: posts }, { data: comments }] = await Promise.all([
  useFetch('/api/posts'),
  useFetch('/api/comments'),
]);
</script>
```

### Composables

```typescript
// composables/useAuth.ts
export function useAuth() {
  const user = useState<User | null>('user', () => null);
  const isAuthenticated = computed(() => user.value !== null);

  async function login(email: string, password: string) {
    const data = await $fetch('/api/auth/login', {
      method: 'POST',
      body: { email, password },
    });
    user.value = data.user;
    return data;
  }

  async function logout() {
    await $fetch('/api/auth/logout', { method: 'POST' });
    user.value = null;
    navigateTo('/login');
  }

  async function fetchUser() {
    try {
      user.value = await $fetch('/api/auth/me');
    } catch {
      user.value = null;
    }
  }

  return {
    user: readonly(user),
    isAuthenticated,
    login,
    logout,
    fetchUser,
  };
}
```

```typescript
// composables/useApi.ts
export function useApi() {
  const config = useRuntimeConfig();

  async function get<T>(url: string): Promise<T> {
    return $fetch(url, {
      baseURL: config.public.apiBase,
    });
  }

  async function post<T>(url: string, body: unknown): Promise<T> {
    return $fetch(url, {
      method: 'POST',
      baseURL: config.public.apiBase,
      body,
    });
  }

  return { get, post };
}
```

### Server API Routes

```typescript
// server/api/users/index.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event);
  const { page = 1, limit = 10 } = query;

  // Use explicit utility functions instead of server middleware
  const db = useDatabase();

  const users = await db.user.findMany({
    skip: (Number(page) - 1) * Number(limit),
    take: Number(limit),
  });

  return users;
});
```

```typescript
// server/api/users/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id');

  if (!id) {
    throw createError({
      statusCode: 400,
      statusMessage: 'User ID is required',
    });
  }

  const db = useDatabase();
  const user = await db.user.findUnique({ where: { id } });

  if (!user) {
    throw createError({
      statusCode: 404,
      statusMessage: 'User not found',
    });
  }

  return user;
});
```

```typescript
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event);
  const { email, password } = body;

  // Validate
  if (!email || !password) {
    throw createError({
      statusCode: 400,
      statusMessage: 'Email and password are required',
    });
  }

  // Authenticate
  const user = await authenticateUser(email, password);

  if (!user) {
    throw createError({
      statusCode: 401,
      statusMessage: 'Invalid credentials',
    });
  }

  // Set session cookie
  setCookie(event, 'session', createSessionToken(user.id), {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7, // 1 week
  });

  return { user };
});
```

### Server Utilities (Not Middleware)

```typescript
// server/utils/db.ts
import { PrismaClient } from '@prisma/client';

let prisma: PrismaClient;

export function useDatabase() {
  if (!prisma) {
    prisma = new PrismaClient();
  }
  return prisma;
}

// server/utils/auth.ts
export function useAuth(event: H3Event) {
  const sessionToken = getCookie(event, 'session');

  if (!sessionToken) {
    return null;
  }

  return verifySession(sessionToken);
}

// Usage in API routes - explicit call instead of middleware
// server/api/protected/data.get.ts
export default defineEventHandler(async (event) => {
  const user = useAuth(event);

  if (!user) {
    throw createError({ statusCode: 401 });
  }

  // Protected logic here
  return { data: 'protected data' };
});
```

### Route Middleware

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated.value && to.path !== '/login') {
    return navigateTo('/login');
  }
});

// middleware/admin.ts
export default defineNuxtRouteMiddleware((to) => {
  const { user } = useAuth();

  if (user.value?.role !== 'admin') {
    return navigateTo('/');
  }
});
```

```vue
<!-- pages/admin/index.vue -->
<script setup>
definePageMeta({
  middleware: ['auth', 'admin'],
});
</script>
```

### SEO and Meta Tags

```vue
<script setup lang="ts">
// Use useHead for meta tags - NOT document.title
useHead({
  title: 'My Page Title',
  meta: [
    { name: 'description', content: 'Page description' },
    { property: 'og:title', content: 'My Page Title' },
  ],
});

// Or use useSeoMeta for typed SEO
useSeoMeta({
  title: 'My Page Title',
  description: 'Page description',
  ogTitle: 'My Page Title',
  ogDescription: 'Page description',
  ogImage: '/og-image.png',
  twitterCard: 'summary_large_image',
});
</script>
```

### Pinia Store

```typescript
// stores/user.ts
import { defineStore } from 'pinia';

interface User {
  id: string;
  name: string;
  email: string;
}

export const useUserStore = defineStore('user', () => {
  const user = ref<User | null>(null);
  const isAuthenticated = computed(() => user.value !== null);

  async function fetchUser() {
    try {
      user.value = await $fetch('/api/auth/me');
    } catch {
      user.value = null;
    }
  }

  function setUser(newUser: User | null) {
    user.value = newUser;
  }

  return {
    user,
    isAuthenticated,
    fetchUser,
    setUser,
  };
});
```

### Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // Enable Nuxt 4 compatibility
  future: {
    compatibilityVersion: 4,
  },

  // App configuration
  app: {
    head: {
      title: 'My Nuxt App',
      meta: [
        { charset: 'utf-8' },
        { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      ],
    },
  },

  // Runtime config
  runtimeConfig: {
    secretKey: process.env.SECRET_KEY,
    public: {
      apiBase: process.env.API_BASE || '/api',
    },
  },

  // Modules
  modules: [
    '@pinia/nuxt',
    '@nuxtjs/tailwindcss',
    '@vueuse/nuxt',
  ],

  // SSR configuration
  ssr: true,

  // Route rules for hybrid rendering
  routeRules: {
    '/': { prerender: true },
    '/blog/**': { swr: 3600 },  // Stale-while-revalidate: 1 hour
    '/admin/**': { ssr: false }, // Client-only
    '/api/**': { cors: true },
  },

  // Nitro configuration
  nitro: {
    preset: 'vercel', // or 'cloudflare', 'netlify', etc.
  },
});
```

---

## Anti-Patterns

### ❌ Using Server Middleware for Everything

**Why it's bad**: Hidden complexity, runs on every request, inflates bundle.

```typescript
// ❌ DON'T — Server middleware runs globally
// server/middleware/auth.ts
export default defineEventHandler((event) => {
  const user = verifyToken(getCookie(event, 'token'));
  event.context.user = user;
});

// ✅ DO — Explicit utility calls in routes
// server/utils/auth.ts
export function useAuth(event: H3Event) {
  return verifyToken(getCookie(event, 'token'));
}

// server/api/protected.get.ts
export default defineEventHandler((event) => {
  const user = useAuth(event);  // Explicit call
  if (!user) throw createError({ statusCode: 401 });
  return { data: 'protected' };
});
```

### ❌ Using document.title Instead of useHead

**Why it's bad**: Doesn't work on navigation, breaks SSR.

```vue
<!-- ❌ DON'T -->
<script setup>
onMounted(() => {
  document.title = 'My Page';  // Won't work on SSR or navigation
});
</script>

<!-- ✅ DO -->
<script setup>
useHead({ title: 'My Page' });
</script>
```

### ❌ Overusing Vuex/Pinia for Trivial State

**Why it's bad**: Unnecessary overhead and complexity.

```vue
<!-- ❌ DON'T — Store for local UI state -->
<script setup>
const store = useDropdownStore();  // Overkill!
</script>

<!-- ✅ DO — Local state for UI -->
<script setup>
const isOpen = ref(false);
</script>
```

### ❌ Not Using SSR-Safe Code

**Why it's bad**: Hydration mismatches, SEO issues.

```vue
<!-- ❌ DON'T — Client-only APIs without checks -->
<script setup>
const width = window.innerWidth;  // Error on SSR!
</script>

<!-- ✅ DO — Check for client -->
<script setup>
const width = ref(0);

onMounted(() => {
  width.value = window.innerWidth;
});
// Or use VueUse
const { width } = useWindowSize();
</script>
```

### ❌ Plugins When Composables Suffice

**Why it's bad**: Plugins add global overhead.

```typescript
// ❌ DON'T — Plugin for simple utility
// plugins/formatDate.ts
export default defineNuxtPlugin(() => {
  return {
    provide: {
      formatDate: (date: Date) => date.toLocaleDateString(),
    },
  };
});

// ✅ DO — Composable
// composables/useFormat.ts
export function useFormat() {
  const formatDate = (date: Date) => date.toLocaleDateString();
  return { formatDate };
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 3.8 | 2024 | Nuxt 4 compatibility mode |
| 3.9 | 2024 | Improved HMR, faster dev |
| 3.10+ | 2025 | Route groups, better caching |
| 4.0 | 2025 | New app/ directory structure |

### Nuxt 4 Changes

- New `app/` directory structure (pages, components in app/)
- Improved TypeScript support
- Better error handling
- Simplified configuration

---

## Quick Reference

| Feature | Usage |
|---------|-------|
| Data fetching | `useFetch('/api/...')` |
| Lazy fetch | `useLazyFetch('/api/...')` |
| Custom async | `useAsyncData('key', () => ...)` |
| Global state | `useState('key', () => initial)` |
| Navigation | `navigateTo('/path')` |
| Route params | `useRoute().params` |
| Head/SEO | `useHead({...})` / `useSeoMeta({...})` |
| Runtime config | `useRuntimeConfig()` |
| Cookies | `useCookie('name')` |

| Rendering Mode | Route Rule |
|----------------|------------|
| SSR (default) | (no rule needed) |
| Static | `{ prerender: true }` |
| SWR | `{ swr: 3600 }` |
| ISR | `{ isr: 3600 }` |
| Client-only | `{ ssr: false }` |

---

## Resources

- [Official Nuxt Documentation](https://nuxt.com/docs)
- [Mastering Nuxt](https://masteringnuxt.com/)
- [Nuxt Performance Best Practices](https://nuxt.com/docs/guide/best-practices/performance)
- [Server Middleware Anti-Pattern](https://masteringnuxt.com/blog/server-middleware-is-an-anti-pattern-in-nuxt)
- [Nuxt 4 Migration](https://nuxt.com/docs/getting-started/upgrade)
- [Common State Management Mistakes](https://moldstud.com/articles/p-common-state-management-mistakes-in-nuxtjs-you-didnt-know-you-were-making)
