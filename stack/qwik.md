# Qwik (2025)

> **Last updated**: January 2026
> **Versions covered**: Qwik 1.x, 2.x (beta)
> **Purpose**: Resumable framework with zero hydration and instant interactivity

---

## Philosophy (2025-2026)

Qwik is the **resumable framework** that eliminates hydration entirely. It serializes application state on the server and resumes exactly where it left off on the client — with ~1KB of JavaScript.

**Key philosophical shifts:**
- **Resumability over hydration** — No re-execution on client
- **~1KB to interactive** — Minimal JavaScript upfront
- **Lazy loading everything** — Code loads on interaction
- **Signals for reactivity** — Fine-grained updates
- **Progressive enhancement** — Works before JavaScript loads
- **Edge-optimized** — Perfect for edge deployments

---

## TL;DR

- Use `$` suffix for lazy-loaded code (`onClick$`, `component$`)
- Use signals (`useSignal`) instead of global state
- Keep components small for efficient serialization
- Code in `$` functions executes on the client lazily
- Use `routeLoader$` for SSR data fetching
- Use `routeAction$` for form mutations
- Prefetch on hover/scroll with Qwik directives
- Great for e-commerce, content sites, SEO-critical apps

---

## Best Practices

### Project Structure (Qwik City)

```
my-qwik-app/
├── src/
│   ├── components/
│   │   ├── ui/
│   │   │   ├── button.tsx
│   │   │   └── card.tsx
│   │   └── features/
│   │       └── product-card.tsx
│   ├── routes/
│   │   ├── index.tsx           # /
│   │   ├── about/
│   │   │   └── index.tsx       # /about
│   │   ├── products/
│   │   │   ├── index.tsx       # /products
│   │   │   └── [id]/
│   │   │       └── index.tsx   # /products/:id
│   │   └── layout.tsx          # Root layout
│   ├── shared/
│   │   ├── loaders.ts
│   │   └── actions.ts
│   └── entry.ssr.tsx
├── public/
├── qwik.config.ts
└── package.json
```

### Basic Component with Signals

```tsx
// src/components/counter.tsx
import { component$, useSignal, $ } from '@builder.io/qwik';

export const Counter = component$(() => {
  // useSignal for reactive state
  const count = useSignal(0);

  // $ suffix marks code for lazy loading
  const increment = $(() => {
    count.value++;
  });

  const decrement = $(() => {
    count.value--;
  });

  const reset = $(() => {
    count.value = 0;
  });

  // JSX — this serializes and resumes on client
  return (
    <div class="counter">
      <p>Count: {count.value}</p>

      <button onClick$={decrement}>-</button>
      <button onClick$={increment}>+</button>
      <button onClick$={reset}>Reset</button>
    </div>
  );
});
```

### Computed Values with useComputed$

```tsx
import { component$, useSignal, useComputed$ } from '@builder.io/qwik';

export const PriceCalculator = component$(() => {
  const quantity = useSignal(1);
  const pricePerUnit = useSignal(29.99);

  // Computed values (derived from signals)
  const subtotal = useComputed$(() => {
    return quantity.value * pricePerUnit.value;
  });

  const tax = useComputed$(() => {
    return subtotal.value * 0.1;
  });

  const total = useComputed$(() => {
    return subtotal.value + tax.value;
  });

  return (
    <div>
      <input
        type="number"
        value={quantity.value}
        onInput$={(e) => {
          quantity.value = parseInt((e.target as HTMLInputElement).value);
        }}
      />

      <p>Subtotal: ${subtotal.value.toFixed(2)}</p>
      <p>Tax: ${tax.value.toFixed(2)}</p>
      <p>Total: ${total.value.toFixed(2)}</p>
    </div>
  );
});
```

### Store (Complex State)

```tsx
import { component$, useStore, $ } from '@builder.io/qwik';

interface Todo {
  id: number;
  text: string;
  done: boolean;
}

interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'done';
}

export const TodoApp = component$(() => {
  // useStore for complex nested state
  const state = useStore<TodoState>({
    todos: [],
    filter: 'all',
  });

  const addTodo = $((text: string) => {
    state.todos.push({
      id: Date.now(),
      text,
      done: false,
    });
  });

  const toggleTodo = $((id: number) => {
    const todo = state.todos.find(t => t.id === id);
    if (todo) {
      todo.done = !todo.done;
    }
  });

  const filteredTodos = () => {
    if (state.filter === 'all') return state.todos;
    return state.todos.filter(t => t.done === (state.filter === 'done'));
  };

  return (
    <div>
      <TodoForm onAdd$={addTodo} />

      <ul>
        {filteredTodos().map((todo) => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.done}
              onChange$={() => toggleTodo(todo.id)}
            />
            <span class={todo.done ? 'done' : ''}>{todo.text}</span>
          </li>
        ))}
      </ul>
    </div>
  );
});
```

### Route Loader (SSR Data)

```tsx
// src/routes/products/index.tsx
import { component$ } from '@builder.io/qwik';
import { routeLoader$ } from '@builder.io/qwik-city';

// Data loads on server, serialized for client
export const useProducts = routeLoader$(async ({ platform }) => {
  // This runs on the server
  const response = await fetch('https://api.example.com/products');
  const products = await response.json();

  return products;
});

export default component$(() => {
  // Access loader data
  const products = useProducts();

  return (
    <div>
      <h1>Products</h1>

      <div class="products-grid">
        {products.value.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
});
```

### Route Action (Form Mutations)

```tsx
// src/routes/products/new/index.tsx
import { component$ } from '@builder.io/qwik';
import { routeAction$, Form, zod$, z } from '@builder.io/qwik-city';

// Server action with validation
export const useCreateProduct = routeAction$(
  async (data, { redirect }) => {
    // This runs on the server
    const response = await fetch('https://api.example.com/products', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    if (!response.ok) {
      return { success: false, error: 'Failed to create product' };
    }

    const product = await response.json();
    throw redirect(302, `/products/${product.id}`);
  },
  // Zod validation
  zod$({
    name: z.string().min(2),
    price: z.coerce.number().positive(),
    description: z.string().optional(),
  })
);

export default component$(() => {
  const createProduct = useCreateProduct();

  return (
    <div>
      <h1>New Product</h1>

      <Form action={createProduct}>
        <div>
          <label>Name</label>
          <input name="name" required />
          {createProduct.value?.fieldErrors?.name && (
            <span class="error">{createProduct.value.fieldErrors.name}</span>
          )}
        </div>

        <div>
          <label>Price</label>
          <input name="price" type="number" step="0.01" required />
        </div>

        <div>
          <label>Description</label>
          <textarea name="description" />
        </div>

        <button type="submit" disabled={createProduct.isRunning}>
          {createProduct.isRunning ? 'Creating...' : 'Create Product'}
        </button>
      </Form>
    </div>
  );
});
```

### Layout with Slot

```tsx
// src/routes/layout.tsx
import { component$, Slot } from '@builder.io/qwik';
import { routeLoader$ } from '@builder.io/qwik-city';

export const useUser = routeLoader$(async ({ cookie }) => {
  const sessionToken = cookie.get('session');
  if (!sessionToken) return null;

  // Fetch user from session
  const response = await fetch('https://api.example.com/auth/me', {
    headers: { Authorization: `Bearer ${sessionToken.value}` },
  });

  if (!response.ok) return null;
  return response.json();
});

export default component$(() => {
  const user = useUser();

  return (
    <div class="app">
      <header>
        <nav>
          <a href="/">Home</a>
          <a href="/products">Products</a>
          {user.value ? (
            <span>Welcome, {user.value.name}</span>
          ) : (
            <a href="/login">Login</a>
          )}
        </nav>
      </header>

      <main>
        <Slot />
      </main>

      <footer>
        <p>© 2025 My App</p>
      </footer>
    </div>
  );
});
```

### Visibility Task (Lazy Effects)

```tsx
import { component$, useVisibleTask$, useSignal } from '@builder.io/qwik';

export const Analytics = component$(() => {
  const hasTracked = useSignal(false);

  // Only runs when component becomes visible
  useVisibleTask$(({ track }) => {
    // Track dependencies
    track(() => hasTracked.value);

    if (!hasTracked.value) {
      // Send analytics event
      fetch('/api/analytics', {
        method: 'POST',
        body: JSON.stringify({ event: 'page_view' }),
      });
      hasTracked.value = true;
    }
  });

  return <div>Tracking enabled</div>;
});
```

### Resource (Async Data)

```tsx
import { component$, useSignal, useResource$, Resource } from '@builder.io/qwik';

export const SearchResults = component$(() => {
  const query = useSignal('');

  // Resource re-fetches when dependencies change
  const searchResults = useResource$(async ({ track, cleanup }) => {
    const q = track(() => query.value);

    if (q.length < 2) return [];

    const abortController = new AbortController();
    cleanup(() => abortController.abort());

    const response = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
      signal: abortController.signal,
    });

    return response.json();
  });

  return (
    <div>
      <input
        type="search"
        bind:value={query}
        placeholder="Search..."
      />

      <Resource
        value={searchResults}
        onPending={() => <div>Searching...</div>}
        onRejected={(error) => <div>Error: {error.message}</div>}
        onResolved={(results) => (
          <ul>
            {results.map((item) => (
              <li key={item.id}>{item.title}</li>
            ))}
          </ul>
        )}
      />
    </div>
  );
});
```

### Prefetching

```tsx
import { component$ } from '@builder.io/qwik';
import { Link } from '@builder.io/qwik-city';

export const ProductCard = component$(({ product }) => {
  return (
    <div class="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>

      {/* Link with prefetch on hover */}
      <Link
        href={`/products/${product.id}`}
        prefetch="hover"
      >
        View Details
      </Link>
    </div>
  );
});
```

---

## Anti-Patterns

### ❌ Heavy Component Bodies

**Why it's bad**: Large components = more serialization.

```tsx
// ❌ DON'T — Too much in one component
export const HugeComponent = component$(() => {
  // 500 lines of state, logic, JSX
});

// ✅ DO — Small, focused components
export const ProductList = component$(() => {
  const products = useProducts();
  return (
    <div>
      {products.value.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
});
```

### ❌ Using useVisibleTask$ for Everything

**Why it's bad**: useVisibleTask$ downloads and runs code on client.

```tsx
// ❌ DON'T — Client-side data fetching
export const Products = component$(() => {
  const products = useSignal([]);

  useVisibleTask$(async () => {
    const res = await fetch('/api/products');
    products.value = await res.json();
  });

  return <div>{/* ... */}</div>;
});

// ✅ DO — Use routeLoader$ for SSR data
export const useProducts = routeLoader$(async () => {
  const res = await fetch('/api/products');
  return res.json();
});

export default component$(() => {
  const products = useProducts();
  return <div>{/* ... */}</div>;
});
```

### ❌ Forgetting the $ Suffix

**Why it's bad**: Code won't be lazy-loaded.

```tsx
// ❌ DON'T — Missing $
<button onClick={handleClick}>Click</button>

// ✅ DO — Use $ for lazy loading
<button onClick$={handleClick$}>Click</button>
<button onClick$={() => count.value++}>Click</button>
```

### ❌ Blocking the Main Thread

**Why it's bad**: Defeats resumability benefits.

```tsx
// ❌ DON'T — Heavy sync operation
const Component = component$(() => {
  const data = heavyCalculation();  // Blocks!
  return <div>{data}</div>;
});

// ✅ DO — Use resource or compute lazily
const Component = component$(() => {
  const data = useResource$(async () => heavyCalculation());
  return <Resource value={data} onResolved={d => <div>{d}</div>} />;
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 1.0 | 2023 | Stable release |
| 1.x | 2024-2025 | Improved DX, better tooling |
| **2.0 beta** | **2025** | New npm scope `@qwik.dev/*`, lighter HTML, faster task scheduler |

### Qwik 2.0 Beta Key Changes

**New NPM Package Scope (Breaking):**
```typescript
// ❌ OLD — Qwik 1.x imports
import { component$, useSignal } from '@builder.io/qwik';
import { routeLoader$ } from '@builder.io/qwik-city';

// ✅ NEW — Qwik 2.x imports
import { component$, useSignal } from '@qwik.dev/core';
import { routeLoader$ } from '@qwik.dev/router';
```

**HTML Nesting Enforcement (Breaking):**
```tsx
// ❌ Qwik 2 enforces valid HTML nesting
<p>
  <div>Invalid!</div>  // Error: <p> cannot contain <div>
</p>

// ✅ Valid HTML structure
<div>
  <div>Valid nesting</div>
</div>
```

**Lighter HTML Output:**
```html
<!-- Qwik 2.0 produces much lighter HTML -->
<!-- - Removed comment nodes from serialization -->
<!-- - Loader serialization disabled by default -->
<!-- - Virtual nodes moved to end of HTML stream -->
<!-- Result: Faster first paint, smaller page weight -->
```

**Improved Resumability:**
```typescript
// Qwik 2.0 optimizations:
// - More efficient encoding for virtual nodes
// - Lazier materialization — only creates nodes needed for user input
// - Non-human readable data moved to end of HTML
// - Faster UI rendering, then framework data arrives
```

**Performance Targets:**
```
- Average page weight 2025: 2.5MB
- Qwik 2.0 target: Sub-10KB JavaScript payload
- Framework data no longer blocks first paint
- Task scheduler rewritten for faster response times
```

**Community Governance:**
```bash
# Qwik is now a true community project
# - Discord renamed to "Qwik"
# - GitHub repo moved to QwikDev organization
# - NPM packages published under @qwik.dev scope
# - New governance model for contributions
```

**Backward Compatibility:**
Qwik 2.0 aims for NO breaking API changes (besides imports). The internal rewrite focuses on performance without changing how you write components.

---

## Quick Reference

| Primitive | Purpose |
|-----------|---------|
| `useSignal` | Reactive primitive value |
| `useStore` | Reactive object/array |
| `useComputed$` | Derived values |
| `useResource$` | Async data fetching |
| `useTask$` | Server-side effects |
| `useVisibleTask$` | Client-side effects |

| Qwik City | Purpose |
|-----------|---------|
| `routeLoader$` | SSR data loading |
| `routeAction$` | Form mutations |
| `Form` | Progressive form component |
| `Link` | Client-side navigation |
| `Slot` | Component children |

| Event Handler | Usage |
|---------------|-------|
| `onClick$` | Click handler (lazy) |
| `onInput$` | Input handler (lazy) |
| `onSubmit$` | Form submit (lazy) |
| `preventdefault:submit` | Prevent default |

---

## When to Use Qwik

| Great For | Not Ideal For |
|-----------|---------------|
| E-commerce | Real-time apps |
| Content sites | Highly interactive dashboards |
| SEO-critical pages | Offline-first PWAs |
| Landing pages | Apps with heavy client state |
| Edge deployments | Games, canvas apps |

---

## Resources

- [Official Qwik Documentation](https://qwik.dev/docs/)
- [Resumability Explained](https://qwik.dev/docs/concepts/resumable/)
- [Qwik in 2025](https://www.learn-qwik.com/blog/qwik-2025/)
- [Why Resumability is the Future](https://medium.com/@vioscott/why-resumability-is-the-future-of-frontend-in-2025-f631ffb6be7f)
- [Think Qwik](https://qwik.dev/docs/concepts/think-qwik/)
- [Qwik vs Angular vs Solid 2025](https://metadesignsolutions.com/angular-vs-qwik-vs-solidjs-in-2025-the-speed-dx-comparison-resumability-ssr-hydration-techniques/)
