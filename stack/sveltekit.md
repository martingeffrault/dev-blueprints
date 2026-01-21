# SvelteKit (2025)

> **Last updated**: January 2026
> **Versions covered**: 2.x with Svelte 5
> **Purpose**: Full-stack framework for Svelte with SSR, SSG, and edge deployment

---

## Philosophy (2025-2026)

SvelteKit powered by Svelte 5's runes represents a **mature, full-stack framework** with excellent DX and performance. Remote Functions and async components simplify server communication.

**Key philosophical shifts:**
- **Svelte 5 runes** — Explicit reactivity ($state, $derived, $effect)
- **Remote Functions** — Type-safe server calls without tRPC/GraphQL
- **Form actions** — Progressive enhancement by default
- **Adapter flexibility** — Deploy anywhere (Node, Edge, Static)
- **Built-in observability** — OpenTelemetry support
- **15-30% smaller bundles** — Better tree-shaking with runes

---

## TL;DR

- Use Svelte 5 runes in all new code
- Use `+page.server.ts` for server-only data
- Use `+layout.ts` for shared data (auth, preferences)
- Form actions for mutations (progressive enhancement)
- Keep modals in page files, not layouts
- Deploy to edge for global performance
- Use `load` functions, not `onMount` for data
- Remote Functions for authenticated operations

---

## Best Practices

### Project Structure

```
my-sveltekit-app/
├── src/
│   ├── lib/
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   │   ├── Button.svelte
│   │   │   │   └── Card.svelte
│   │   │   └── features/
│   │   │       └── UserCard.svelte
│   │   ├── server/           # Server-only code
│   │   │   ├── db.ts
│   │   │   └── auth.ts
│   │   ├── stores/           # Svelte stores (or runes)
│   │   │   └── user.svelte.ts
│   │   └── utils/
│   │       └── format.ts
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +layout.ts        # Shared data loading
│   │   ├── +page.svelte
│   │   ├── +page.ts          # Page data loading
│   │   ├── about/
│   │   │   └── +page.svelte
│   │   ├── blog/
│   │   │   ├── +page.svelte
│   │   │   ├── +page.server.ts
│   │   │   └── [slug]/
│   │   │       ├── +page.svelte
│   │   │       └── +page.server.ts
│   │   ├── api/              # API routes
│   │   │   └── users/
│   │   │       └── +server.ts
│   │   └── (auth)/           # Route groups
│   │       ├── login/
│   │       └── register/
│   ├── hooks.server.ts       # Server hooks
│   ├── hooks.client.ts       # Client hooks
│   └── app.html
├── static/
├── svelte.config.js
├── vite.config.ts
└── package.json
```

### Page with Runes (Svelte 5)

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import type { PageData } from './$types';
  import { enhance } from '$app/forms';

  let { data }: { data: PageData } = $props();

  // Local state with runes
  let searchQuery = $state('');
  let sortOrder = $state<'asc' | 'desc'>('desc');

  // Derived values
  let filteredUsers = $derived(
    data.users
      .filter(u => u.name.toLowerCase().includes(searchQuery.toLowerCase()))
      .sort((a, b) => sortOrder === 'asc'
        ? a.name.localeCompare(b.name)
        : b.name.localeCompare(a.name)
      )
  );

  let userCount = $derived(filteredUsers.length);
</script>

<div class="users-page">
  <header>
    <h1>Users ({userCount})</h1>
    <input
      type="search"
      bind:value={searchQuery}
      placeholder="Search users..."
    />
    <select bind:value={sortOrder}>
      <option value="asc">A-Z</option>
      <option value="desc">Z-A</option>
    </select>
  </header>

  <ul>
    {#each filteredUsers as user (user.id)}
      <li>{user.name} - {user.email}</li>
    {/each}
  </ul>
</div>
```

### Server Data Loading

```typescript
// src/routes/blog/+page.server.ts
import { db } from '$lib/server/db';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ url, locals }) => {
  const page = Number(url.searchParams.get('page') ?? 1);
  const limit = 10;

  const [posts, total] = await Promise.all([
    db.post.findMany({
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
      include: { author: true },
    }),
    db.post.count({ where: { published: true } }),
  ]);

  return {
    posts,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
    },
  };
};
```

```typescript
// src/routes/blog/[slug]/+page.server.ts
import { db } from '$lib/server/db';
import { error } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params }) => {
  const post = await db.post.findUnique({
    where: { slug: params.slug },
    include: { author: true, comments: true },
  });

  if (!post) {
    throw error(404, 'Post not found');
  }

  return { post };
};
```

### Layout Data (Global Data)

```typescript
// src/routes/+layout.ts
import type { LayoutLoad } from './$types';

export const load: LayoutLoad = async ({ fetch }) => {
  // Fetch global data - auth state, user preferences
  const response = await fetch('/api/auth/me');

  if (response.ok) {
    const user = await response.json();
    return { user };
  }

  return { user: null };
};
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import type { LayoutData } from './$types';
  import Header from '$lib/components/Header.svelte';
  import Footer from '$lib/components/Footer.svelte';

  let { data, children }: { data: LayoutData; children: any } = $props();
</script>

<div class="app">
  <Header user={data.user} />
  <main>
    {@render children()}
  </main>
  <Footer />
</div>
```

### Form Actions (Progressive Enhancement)

```typescript
// src/routes/blog/new/+page.server.ts
import { db } from '$lib/server/db';
import { fail, redirect } from '@sveltejs/kit';
import type { Actions, PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals }) => {
  if (!locals.user) {
    throw redirect(303, '/login');
  }
  return {};
};

export const actions: Actions = {
  create: async ({ request, locals }) => {
    if (!locals.user) {
      return fail(401, { message: 'Unauthorized' });
    }

    const formData = await request.formData();
    const title = formData.get('title')?.toString();
    const content = formData.get('content')?.toString();
    const slug = formData.get('slug')?.toString();

    // Validation
    const errors: Record<string, string> = {};
    if (!title || title.length < 3) {
      errors.title = 'Title must be at least 3 characters';
    }
    if (!content || content.length < 10) {
      errors.content = 'Content must be at least 10 characters';
    }
    if (!slug || !/^[a-z0-9-]+$/.test(slug)) {
      errors.slug = 'Slug must be lowercase alphanumeric with hyphens';
    }

    if (Object.keys(errors).length > 0) {
      return fail(400, { errors, title, content, slug });
    }

    // Create post
    const post = await db.post.create({
      data: {
        title,
        content,
        slug,
        authorId: locals.user.id,
      },
    });

    throw redirect(303, `/blog/${post.slug}`);
  },
};
```

```svelte
<!-- src/routes/blog/new/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { ActionData } from './$types';

  let { form }: { form: ActionData } = $props();

  let submitting = $state(false);
</script>

<form
  method="POST"
  action="?/create"
  use:enhance={() => {
    submitting = true;
    return async ({ update }) => {
      submitting = false;
      await update();
    };
  }}
>
  <label>
    Title
    <input name="title" value={form?.title ?? ''} />
    {#if form?.errors?.title}
      <span class="error">{form.errors.title}</span>
    {/if}
  </label>

  <label>
    Slug
    <input name="slug" value={form?.slug ?? ''} />
    {#if form?.errors?.slug}
      <span class="error">{form.errors.slug}</span>
    {/if}
  </label>

  <label>
    Content
    <textarea name="content">{form?.content ?? ''}</textarea>
    {#if form?.errors?.content}
      <span class="error">{form.errors.content}</span>
    {/if}
  </label>

  <button type="submit" disabled={submitting}>
    {submitting ? 'Creating...' : 'Create Post'}
  </button>
</form>
```

### Reusable State with Runes

```typescript
// src/lib/stores/counter.svelte.ts
export function createCounter(initial: number = 0) {
  let count = $state(initial);

  return {
    get count() {
      return count;
    },
    increment() {
      count++;
    },
    decrement() {
      count--;
    },
    reset() {
      count = initial;
    },
  };
}

// src/lib/stores/user.svelte.ts
import type { User } from '$lib/types';

class UserStore {
  user = $state<User | null>(null);
  loading = $state(false);

  get isAuthenticated() {
    return this.user !== null;
  }

  async login(email: string, password: string) {
    this.loading = true;
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
        headers: { 'Content-Type': 'application/json' },
      });
      this.user = await response.json();
    } finally {
      this.loading = false;
    }
  }

  logout() {
    this.user = null;
  }
}

export const userStore = new UserStore();
```

### API Routes

```typescript
// src/routes/api/users/+server.ts
import { json } from '@sveltejs/kit';
import { db } from '$lib/server/db';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ url }) => {
  const page = Number(url.searchParams.get('page') ?? 1);
  const limit = Number(url.searchParams.get('limit') ?? 10);

  const users = await db.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
    select: { id: true, name: true, email: true },
  });

  return json(users);
};

export const POST: RequestHandler = async ({ request, locals }) => {
  if (!locals.user) {
    return json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();
  const { name, email } = body;

  const user = await db.user.create({
    data: { name, email },
  });

  return json(user, { status: 201 });
};
```

### Server Hooks (Authentication)

```typescript
// src/hooks.server.ts
import { db } from '$lib/server/db';
import type { Handle } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  // Get session token from cookie
  const sessionToken = event.cookies.get('session');

  if (sessionToken) {
    // Verify and attach user to locals
    const session = await db.session.findUnique({
      where: { token: sessionToken },
      include: { user: true },
    });

    if (session && session.expiresAt > new Date()) {
      event.locals.user = session.user;
    }
  }

  return resolve(event);
};
```

### Error Handling

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/stores';
</script>

<div class="error-page">
  <h1>{$page.status}</h1>
  <p>{$page.error?.message ?? 'Something went wrong'}</p>
  <a href="/">Go home</a>
</div>
```

### Configuration

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

/** @type {import('@sveltejs/kit').Config} */
const config = {
  preprocess: vitePreprocess(),

  kit: {
    adapter: adapter(),

    // CSP configuration
    csp: {
      mode: 'auto',
      directives: {
        'script-src': ['self'],
        'style-src': ['self', 'unsafe-inline'],
      },
    },

    // Prerender configuration
    prerender: {
      handleHttpError: 'warn',
      handleMissingId: 'warn',
    },
  },
};

export default config;
```

---

## Anti-Patterns

### ❌ Modal State in Layouts

**Why it's bad**: Modals survive page changes, state management becomes tricky.

```svelte
<!-- ❌ DON'T — Modal in layout -->
<!-- src/routes/+layout.svelte -->
<script>
  let showModal = $state(false);
</script>

<slot />
{#if showModal}
  <Modal />
{/if}

<!-- ✅ DO — Modal in each page that needs it -->
<!-- src/routes/users/+page.svelte -->
<script>
  let showModal = $state(false);
</script>

<button onclick={() => showModal = true}>Open</button>
{#if showModal}
  <Modal onclose={() => showModal = false} />
{/if}
```

### ❌ Data Fetching in onMount

**Why it's bad**: Misses SSR benefits, causes loading states.

```svelte
<!-- ❌ DON'T -->
<script>
  import { onMount } from 'svelte';

  let users = $state([]);

  onMount(async () => {
    const res = await fetch('/api/users');
    users = await res.json();
  });
</script>

<!-- ✅ DO — Use load function -->
<!-- +page.server.ts -->
export const load = async () => {
  const users = await db.user.findMany();
  return { users };
};

<!-- +page.svelte -->
<script>
  let { data } = $props();
  // data.users is available immediately, SSR'd
</script>
```

### ❌ Mixing Svelte 4 and 5 Patterns

**Why it's bad**: Inconsistent, confusing, misses optimization.

```svelte
<!-- ❌ DON'T -->
<script>
  export let name;           // Svelte 4
  let count = $state(0);     // Svelte 5
</script>

<!-- ✅ DO — Consistent Svelte 5 -->
<script>
  let { name } = $props();
  let count = $state(0);
</script>
```

### ❌ Remote Functions for Public APIs

**Why it's bad**: Remote Functions are for internal, authenticated operations.

```typescript
// ❌ DON'T — Public API as Remote Function
// Keep REST endpoints for public APIs, webhooks, third-party integrations

// ✅ DO — Remote Functions for internal app logic
// Use +server.ts for public APIs
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.0 | Dec 2023 | Svelte 5 support prep |
| 2.x | 2024-2025 | Svelte 5 runes, shallow routing |
| 2025 | Ongoing | Remote Functions, async components |

### SvelteKit 2025 Features

- **Remote Functions**: Type-safe server calls without tRPC
- **Async Components**: Direct await in component scripts
- **OpenTelemetry**: Built-in observability via `instrumentation.server.ts`
- **Service Workers**: Built-in offline support
- **15-30% smaller bundles**: Better tree-shaking with Svelte 5

---

## Quick Reference

| File | Purpose |
|------|---------|
| `+page.svelte` | Page component |
| `+page.ts` | Universal data loading |
| `+page.server.ts` | Server-only data loading |
| `+layout.svelte` | Layout wrapper |
| `+layout.ts` | Layout data (shared) |
| `+layout.server.ts` | Server layout data |
| `+server.ts` | API endpoint |
| `+error.svelte` | Error boundary |

| Pattern | Syntax |
|---------|--------|
| Props (Svelte 5) | `let { prop } = $props()` |
| State | `let count = $state(0)` |
| Derived | `let double = $derived(count * 2)` |
| Effect | `$effect(() => {...})` |
| Snippet | `{#snippet name()}...{/snippet}` |
| Render | `{@render children()}` |

---

## Resources

- [Official SvelteKit Documentation](https://svelte.dev/docs/kit)
- [SvelteKit 2025 Modern Development](https://zxce3.net/posts/sveltekit-2025-modern-development-trends-and-best-practices/)
- [SvelteKit Complete Guide 2025](https://criztec.com/sveltekit-complete-developers-guide-2025)
- [Svelte 5 Runes](https://svelte.dev/blog/runes)
- [SvelteKit Performance](https://svelte.dev/docs/kit/performance)
- [Svelte and SvelteKit Updates 2025](https://blog.openreplay.com/svelte-sveltekit-updates-summer-2025-recap/)
