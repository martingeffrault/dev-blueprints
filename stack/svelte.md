# Svelte 5 (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.x
> **Purpose**: Compile-time reactive framework with minimal runtime

---

## Philosophy (2025-2026)

Svelte 5 introduces **runes** — explicit, universal reactivity that works everywhere.

**Key philosophical shifts:**
- **Runes = explicit reactivity** — `$state`, `$derived`, `$effect`
- **Universal reactivity** — Works in `.svelte` and `.svelte.js` files
- **No virtual DOM** — Compile-time optimization
- **15-30% smaller bundles** — Better tree-shaking
- **Backwards compatible** — Svelte 4 code still works
- **Snippets replace slots** — More powerful composition

---

## TL;DR

- Use `$state` for reactive state (replaces `let` reactivity)
- Use `$derived` for computed values
- Use `$effect` for side effects
- Use `$props` for component props (replaces `export let`)
- Runes are top-level only
- Stores still work but runes are preferred
- Migrate gradually — Svelte 4 syntax still supported

---

## Best Practices

### Basic Component with Runes

```svelte
<!-- Counter.svelte -->
<script lang="ts">
  // Reactive state with $state
  let count = $state(0);

  // Derived value with $derived
  let doubled = $derived(count * 2);
  let isEven = $derived(count % 2 === 0);

  // Functions
  function increment() {
    count++;
  }

  function decrement() {
    count--;
  }

  function reset() {
    count = 0;
  }
</script>

<div class="counter">
  <p>Count: {count}</p>
  <p>Doubled: {doubled}</p>
  <p>Is even: {isEven ? 'Yes' : 'No'}</p>

  <button onclick={decrement}>-</button>
  <button onclick={increment}>+</button>
  <button onclick={reset}>Reset</button>
</div>
```

### Props with $props

```svelte
<!-- UserCard.svelte -->
<script lang="ts">
  interface Props {
    name: string;
    email: string;
    avatar?: string;
    isAdmin?: boolean;
  }

  // Props with defaults
  let { name, email, avatar = '/default-avatar.png', isAdmin = false }: Props = $props();
</script>

<div class="user-card" class:admin={isAdmin}>
  <img src={avatar} alt={name} />
  <h3>{name}</h3>
  <p>{email}</p>
  {#if isAdmin}
    <span class="badge">Admin</span>
  {/if}
</div>

<style>
  .admin {
    border: 2px solid gold;
  }
</style>
```

### Bindable Props with $bindable

```svelte
<!-- SearchInput.svelte -->
<script lang="ts">
  interface Props {
    value: string;
    placeholder?: string;
  }

  let { value = $bindable(''), placeholder = 'Search...' }: Props = $props();
</script>

<input
  type="text"
  bind:value
  {placeholder}
/>

<!-- Usage with two-way binding -->
<!-- <SearchInput bind:value={searchQuery} /> -->
```

### Effects with $effect

```svelte
<script lang="ts">
  let count = $state(0);
  let title = $state('My App');

  // Effect runs when dependencies change
  $effect(() => {
    document.title = `${title} - Count: ${count}`;
  });

  // Effect with cleanup
  $effect(() => {
    const interval = setInterval(() => {
      console.log('Count is:', count);
    }, 1000);

    // Cleanup function
    return () => {
      clearInterval(interval);
    };
  });

  // Pre-effect (runs before DOM update)
  $effect.pre(() => {
    console.log('About to update, count is:', count);
  });
</script>
```

### Complex State (Objects/Arrays)

```svelte
<script lang="ts">
  interface Todo {
    id: number;
    text: string;
    completed: boolean;
  }

  // Objects and arrays are deeply reactive by default
  let todos = $state<Todo[]>([
    { id: 1, text: 'Learn Svelte 5', completed: false },
    { id: 2, text: 'Build something', completed: false },
  ]);

  let newTodoText = $state('');

  // Derived values
  let completedCount = $derived(todos.filter(t => t.completed).length);
  let remainingCount = $derived(todos.length - completedCount);

  function addTodo() {
    if (!newTodoText.trim()) return;

    todos.push({
      id: Date.now(),
      text: newTodoText,
      completed: false,
    });
    newTodoText = '';
  }

  function toggleTodo(id: number) {
    const todo = todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  }

  function removeTodo(id: number) {
    const index = todos.findIndex(t => t.id === id);
    if (index !== -1) {
      todos.splice(index, 1);
    }
  }
</script>

<div class="todo-app">
  <form onsubmit={(e) => { e.preventDefault(); addTodo(); }}>
    <input bind:value={newTodoText} placeholder="Add todo..." />
    <button type="submit">Add</button>
  </form>

  <ul>
    {#each todos as todo (todo.id)}
      <li class:completed={todo.completed}>
        <input
          type="checkbox"
          checked={todo.completed}
          onchange={() => toggleTodo(todo.id)}
        />
        <span>{todo.text}</span>
        <button onclick={() => removeTodo(todo.id)}>×</button>
      </li>
    {/each}
  </ul>

  <p>{completedCount} completed, {remainingCount} remaining</p>
</div>
```

### Reusable Logic in .svelte.js Files

```typescript
// lib/counter.svelte.ts
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

// Usage in component
<script lang="ts">
  import { createCounter } from '$lib/counter.svelte';

  const counter = createCounter(10);
</script>

<button onclick={counter.increment}>{counter.count}</button>
```

```typescript
// lib/fetch.svelte.ts
export function useFetch<T>(url: string) {
  let data = $state<T | null>(null);
  let loading = $state(true);
  let error = $state<Error | null>(null);

  $effect(() => {
    loading = true;
    error = null;

    fetch(url)
      .then(r => r.json())
      .then(json => {
        data = json;
      })
      .catch(e => {
        error = e;
      })
      .finally(() => {
        loading = false;
      });
  });

  return {
    get data() { return data; },
    get loading() { return loading; },
    get error() { return error; },
  };
}
```

### Snippets (Replace Slots)

```svelte
<!-- Card.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    title: string;
    header?: Snippet;
    children: Snippet;
    footer?: Snippet;
  }

  let { title, header, children, footer }: Props = $props();
</script>

<div class="card">
  {#if header}
    <header>{@render header()}</header>
  {:else}
    <header><h2>{title}</h2></header>
  {/if}

  <main>
    {@render children()}
  </main>

  {#if footer}
    <footer>{@render footer()}</footer>
  {/if}
</div>

<!-- Usage -->
<Card title="My Card">
  <p>Card content here</p>

  {#snippet footer()}
    <button>Action</button>
  {/snippet}
</Card>
```

### Event Handlers (Svelte 5 Style)

```svelte
<script lang="ts">
  // Callback props instead of createEventDispatcher
  interface Props {
    onsubmit?: (data: FormData) => void;
    oncancel?: () => void;
  }

  let { onsubmit, oncancel }: Props = $props();

  function handleSubmit(event: SubmitEvent) {
    event.preventDefault();
    const formData = new FormData(event.target as HTMLFormElement);
    onsubmit?.({ name: formData.get('name') as string });
  }
</script>

<form onsubmit={handleSubmit}>
  <input name="name" />
  <button type="submit">Submit</button>
  <button type="button" onclick={oncancel}>Cancel</button>
</form>

<!-- Parent usage -->
<!-- <MyForm onsubmit={(data) => console.log(data)} oncancel={() => modal = false} /> -->
```

### Lifecycle with $effect

```svelte
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  // Traditional lifecycle (still works)
  onMount(() => {
    console.log('Component mounted');
    return () => console.log('Component unmounted');
  });

  // OR use $effect for mount behavior
  $effect(() => {
    console.log('Component mounted (via effect)');

    return () => {
      console.log('Cleanup on unmount');
    };
  });
</script>
```

---

## Anti-Patterns

### ❌ Mixing Svelte 4 and 5 Syntax

**Why it's bad**: Confusing, inconsistent behavior.

```svelte
<!-- ❌ DON'T — Mix old and new -->
<script>
  export let name;          // Svelte 4 props
  let count = $state(0);    // Svelte 5 runes
</script>

<!-- ✅ DO — Consistent style -->
<script>
  let { name } = $props();  // Svelte 5 props
  let count = $state(0);    // Svelte 5 state
</script>
```

### ❌ Runes Outside Top Level

**Why it's bad**: Runes only work at the top level.

```svelte
<script>
  // ❌ DON'T
  function createState() {
    let value = $state(0);  // Error! Not top-level
    return value;
  }

  // ✅ DO — Top-level only
  let value = $state(0);

  // ✅ OR — Use .svelte.ts files for reusable state
</script>
```

### ❌ Over-reactive Updates

**Why it's bad**: Unnecessary re-renders.

```svelte
<script>
  // ❌ DON'T — Object reference changes trigger updates
  let user = $state({ name: 'Alice', age: 30 });

  $effect(() => {
    // This runs even if only age changed
    console.log('User name:', user.name);
  });

  // ✅ DO — Extract specific dependencies
  let userName = $derived(user.name);

  $effect(() => {
    // Only runs when name actually changes
    console.log('User name:', userName);
  });
</script>
```

### ❌ Direct State Assignment in $derived

**Why it's bad**: $derived should be pure.

```svelte
<script>
  let count = $state(0);

  // ❌ DON'T — Side effects in derived
  let doubled = $derived(() => {
    console.log('Computing...'); // Side effect!
    return count * 2;
  });

  // ✅ DO — Keep derived pure
  let doubled = $derived(count * 2);

  // Use $effect for side effects
  $effect(() => {
    console.log('Count changed:', count);
  });
</script>
```

### ❌ Multiple Event Listeners on Same Event

**Why it's bad**: Svelte 5 removed this anti-pattern.

```svelte
<!-- ❌ DON'T (Svelte 4 anti-pattern) -->
<button on:click={handler1} on:click={handler2}>

<!-- ✅ DO — Single handler that calls both -->
<button onclick={() => { handler1(); handler2(); }}>
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 5.0 | Oct 2024 | Runes, Snippets, Event attributes |
| 5.x | 2025 | Stability improvements |
| SvelteKit 2.x | 2025 | Full Svelte 5 integration |

### Migration from Svelte 4

| Svelte 4 | Svelte 5 |
|----------|----------|
| `let x = 0` (reactive) | `let x = $state(0)` |
| `$: doubled = x * 2` | `let doubled = $derived(x * 2)` |
| `$: console.log(x)` | `$effect(() => console.log(x))` |
| `export let prop` | `let { prop } = $props()` |
| `createEventDispatcher` | Callback props |
| `<slot>` | `{@render children()}` |
| `on:click` | `onclick` |

---

## Quick Reference

| Rune | Purpose |
|------|---------|
| `$state(value)` | Create reactive state |
| `$derived(expr)` | Computed value |
| `$derived.by(fn)` | Complex derived with function |
| `$effect(fn)` | Side effect |
| `$effect.pre(fn)` | Effect before DOM update |
| `$props()` | Declare component props |
| `$bindable()` | Two-way bindable prop |
| `$inspect(value)` | Debug logging |

| Pattern | Syntax |
|---------|--------|
| Reactive array | `let items = $state([])` |
| Reactive object | `let obj = $state({})` |
| Readonly derived | `let x = $derived(...)` (can't reassign) |
| Effect cleanup | `$effect(() => { return cleanup; })` |
| Snippet | `{#snippet name()}...{/snippet}` |
| Render snippet | `{@render name()}` |

---

## Resources

- [Official Svelte 5 Documentation](https://svelte.dev/docs)
- [Runes Introduction](https://svelte.dev/blog/runes)
- [Svelte 5 Migration Guide](https://svelte.dev/docs/svelte/v5-migration-guide)
- [SvelteKit 2025 Best Practices](https://zxce3.net/posts/sveltekit-2025-modern-development-trends-and-best-practices/)
- [Runes Real-World Patterns](https://www.captaincodeman.com/svelte-5-runes-real-world-patterns-and-gotchas)
- [Global State with Runes](https://mainmatter.com/blog/2025/03/11/global-state-in-svelte-5/)
