# Solid.js (2025)

> **Last updated**: January 2026
> **Versions covered**: Solid 1.8+, Solid 2.0 preview
> **Purpose**: Fine-grained reactive UI framework with no virtual DOM

---

## Philosophy (2025-2026)

Solid.js is the **fine-grained reactive framework** that compiles components to real DOM operations, delivering near-native performance without a virtual DOM.

**Key philosophical shifts:**
- **Signals are primitives** — Not just state, but reactive foundations
- **No virtual DOM** — Direct DOM updates via compilation
- **Components run once** — Setup only, not re-render
- **Reactivity flows through getters** — Don't break the chain
- **Derive, don't sync** — Compute from signals, don't duplicate
- **90% satisfaction** — State of JS 2025 developer happiness

---

## TL;DR

- Components run once — they're setup functions, not render functions
- Signals are getters — always call them `count()` not `count`
- Don't destructure props — breaks reactivity
- Derive values instead of syncing state
- Use `createEffect` for side effects
- Use `createMemo` for expensive computations
- JSX is real DOM manipulation, not virtual DOM
- `Show`, `For`, `Switch` for control flow (not ternaries in JSX)

---

## Best Practices

### Project Structure

```
my-solid-app/
├── src/
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   └── Card.tsx
│   │   └── features/
│   │       ├── UserCard.tsx
│   │       └── PostList.tsx
│   ├── stores/
│   │   ├── user.ts
│   │   └── posts.ts
│   ├── hooks/                # Custom reactive primitives
│   │   ├── createLocalStorage.ts
│   │   └── createFetch.ts
│   ├── pages/
│   │   ├── Home.tsx
│   │   └── Profile.tsx
│   ├── App.tsx
│   └── index.tsx
├── vite.config.ts
└── package.json
```

### Basic Component with Signals

```tsx
// src/components/Counter.tsx
import { createSignal, createEffect, createMemo } from 'solid-js';

export function Counter() {
  // Signals return [getter, setter]
  const [count, setCount] = createSignal(0);

  // Derived value (automatically tracks dependencies)
  const doubled = createMemo(() => count() * 2);
  const isEven = createMemo(() => count() % 2 === 0);

  // Side effect (runs when dependencies change)
  createEffect(() => {
    console.log('Count changed:', count());
    document.title = `Count: ${count()}`;
  });

  // Event handlers
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(0);

  // Component body runs ONCE — this is setup, not render
  console.log('Counter setup (runs once)');

  return (
    <div class="counter">
      <p>Count: {count()}</p>
      <p>Doubled: {doubled()}</p>
      <p>Is even: {isEven() ? 'Yes' : 'No'}</p>

      <button onClick={decrement}>-</button>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Props (Don't Destructure!)

```tsx
// src/components/UserCard.tsx
import { Component, Show } from 'solid-js';

interface UserCardProps {
  name: string;
  email: string;
  avatar?: string;
  isAdmin?: boolean;
  onEdit?: () => void;
}

// ❌ DON'T destructure props — breaks reactivity
// const UserCard: Component<UserCardProps> = ({ name, email }) => {...}

// ✅ DO — Access props through the props object
export const UserCard: Component<UserCardProps> = (props) => {
  // Props are reactive getters, accessed via props.name

  return (
    <div class="user-card" classList={{ admin: props.isAdmin }}>
      <Show when={props.avatar}>
        <img src={props.avatar} alt={props.name} />
      </Show>

      <h3>{props.name}</h3>
      <p>{props.email}</p>

      <Show when={props.isAdmin}>
        <span class="badge">Admin</span>
      </Show>

      <Show when={props.onEdit}>
        <button onClick={props.onEdit}>Edit</button>
      </Show>
    </div>
  );
};
```

### Using splitProps for Spreading

```tsx
// When you need to split props (e.g., for spreading to DOM elements)
import { splitProps, Component, JSX } from 'solid-js';

interface ButtonProps extends JSX.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
  loading?: boolean;
}

export const Button: Component<ButtonProps> = (props) => {
  // Split custom props from native props
  const [local, buttonProps] = splitProps(props, ['variant', 'loading', 'children']);

  return (
    <button
      {...buttonProps}
      class={`btn btn-${local.variant ?? 'primary'}`}
      disabled={local.loading || props.disabled}
    >
      <Show when={local.loading} fallback={local.children}>
        Loading...
      </Show>
    </button>
  );
};
```

### Control Flow Components

```tsx
import { Show, For, Switch, Match, Index } from 'solid-js';

function TodoList() {
  const [todos, setTodos] = createSignal([
    { id: 1, text: 'Learn Solid', done: false },
    { id: 2, text: 'Build app', done: false },
  ]);

  const [filter, setFilter] = createSignal<'all' | 'active' | 'done'>('all');

  const filteredTodos = createMemo(() => {
    const f = filter();
    if (f === 'all') return todos();
    return todos().filter(t => t.done === (f === 'done'));
  });

  return (
    <div>
      {/* Conditional rendering */}
      <Show
        when={todos().length > 0}
        fallback={<p>No todos yet!</p>}
      >
        {/* List rendering with keyed items */}
        <ul>
          <For each={filteredTodos()}>
            {(todo, index) => (
              <li>
                <span>{index() + 1}. {todo.text}</span>
                <input
                  type="checkbox"
                  checked={todo.done}
                  onChange={() => toggleTodo(todo.id)}
                />
              </li>
            )}
          </For>
        </ul>
      </Show>

      {/* Switch/Match for multiple conditions */}
      <Switch fallback={<span>Showing all</span>}>
        <Match when={filter() === 'active'}>
          <span>Showing active only</span>
        </Match>
        <Match when={filter() === 'done'}>
          <span>Showing completed only</span>
        </Match>
      </Switch>
    </div>
  );
}
```

### Stores (Complex State)

```tsx
// src/stores/todos.ts
import { createStore, produce } from 'solid-js/store';

interface Todo {
  id: number;
  text: string;
  done: boolean;
}

interface TodoStore {
  todos: Todo[];
  filter: 'all' | 'active' | 'done';
}

export function createTodoStore() {
  const [state, setState] = createStore<TodoStore>({
    todos: [],
    filter: 'all',
  });

  const addTodo = (text: string) => {
    setState('todos', todos => [...todos, {
      id: Date.now(),
      text,
      done: false,
    }]);
  };

  const toggleTodo = (id: number) => {
    // Use produce for nested updates
    setState(
      produce((s) => {
        const todo = s.todos.find(t => t.id === id);
        if (todo) todo.done = !todo.done;
      })
    );
  };

  const removeTodo = (id: number) => {
    setState('todos', todos => todos.filter(t => t.id !== id));
  };

  const setFilter = (filter: TodoStore['filter']) => {
    setState('filter', filter);
  };

  const filteredTodos = () => {
    if (state.filter === 'all') return state.todos;
    return state.todos.filter(t => t.done === (state.filter === 'done'));
  };

  return {
    state,
    addTodo,
    toggleTodo,
    removeTodo,
    setFilter,
    filteredTodos,
  };
}
```

### Custom Reactive Primitives

```tsx
// src/hooks/createLocalStorage.ts
import { createSignal, createEffect, onCleanup } from 'solid-js';

export function createLocalStorage<T>(key: string, initialValue: T) {
  // Read from localStorage or use initial
  const stored = localStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [value, setValue] = createSignal<T>(initial);

  // Persist to localStorage on change
  createEffect(() => {
    localStorage.setItem(key, JSON.stringify(value()));
  });

  return [value, setValue] as const;
}

// src/hooks/createFetch.ts
import { createSignal, createResource, onCleanup } from 'solid-js';

export function createFetch<T>(url: () => string) {
  const [data, { refetch, mutate }] = createResource(url, async (url) => {
    const response = await fetch(url);
    if (!response.ok) throw new Error('Fetch failed');
    return response.json() as Promise<T>;
  });

  return {
    data,
    loading: () => data.loading,
    error: () => data.error,
    refetch,
    mutate,
  };
}
```

### Resources (Async Data)

```tsx
import { createResource, Suspense, ErrorBoundary } from 'solid-js';

interface User {
  id: number;
  name: string;
  email: string;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('User not found');
  return response.json();
}

function UserProfile(props: { userId: string }) {
  // createResource for async data
  const [user, { refetch, mutate }] = createResource(
    () => props.userId,  // Source signal
    fetchUser            // Fetcher
  );

  return (
    <div>
      <Show when={!user.loading} fallback={<p>Loading...</p>}>
        <Show when={user()} fallback={<p>User not found</p>}>
          {(u) => (
            <div>
              <h2>{u().name}</h2>
              <p>{u().email}</p>
              <button onClick={() => refetch()}>Refresh</button>
            </div>
          )}
        </Show>
      </Show>
    </div>
  );
}

// With Suspense boundary
function App() {
  return (
    <ErrorBoundary fallback={(err) => <div>Error: {err.message}</div>}>
      <Suspense fallback={<div>Loading...</div>}>
        <UserProfile userId="1" />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Context

```tsx
// src/context/ThemeContext.tsx
import { createContext, useContext, ParentComponent } from 'solid-js';
import { createStore } from 'solid-js/store';

type Theme = 'light' | 'dark';

interface ThemeContextValue {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue>();

export const ThemeProvider: ParentComponent = (props) => {
  const [state, setState] = createStore({ theme: 'light' as Theme });

  const toggleTheme = () => {
    setState('theme', t => t === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme: state.theme, toggleTheme }}>
      {props.children}
    </ThemeContext.Provider>
  );
};

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
}
```

---

## Anti-Patterns

### ❌ Destructuring Props

**Why it's bad**: Props are getters — destructuring reads them once.

```tsx
// ❌ DON'T — Breaks reactivity
const Component = ({ name, count }) => {
  return <p>{name}: {count}</p>;  // Never updates!
};

// ✅ DO — Access via props object
const Component = (props) => {
  return <p>{props.name}: {props.count}</p>;  // Reactive!
};

// ✅ OR — Use splitProps if needed
const [local, others] = splitProps(props, ['name', 'count']);
```

### ❌ Not Calling Signals

**Why it's bad**: You're passing the signal function, not the value.

```tsx
// ❌ DON'T — Passing signal function
const [count, setCount] = createSignal(0);
<ChildComponent count={count} />  // ChildComponent gets a function!

// ✅ DO — Call the signal
<ChildComponent count={count()} />  // ChildComponent gets the value
```

### ❌ Reading Signals in Component Body

**Why it's bad**: Component body runs once, won't update.

```tsx
// ❌ DON'T — Read in component body
function Component() {
  const [count, setCount] = createSignal(0);
  const message = `Count is ${count()}`;  // Captured once!
  return <p>{message}</p>;  // Never updates!
}

// ✅ DO — Read in JSX or reactive contexts
function Component() {
  const [count, setCount] = createSignal(0);
  return <p>Count is {count()}</p>;  // Updates!
}

// ✅ OR — Use createMemo
function Component() {
  const [count, setCount] = createSignal(0);
  const message = createMemo(() => `Count is ${count()}`);
  return <p>{message()}</p>;  // Updates!
}
```

### ❌ Syncing Instead of Deriving

**Why it's bad**: Double source of truth, easy to get out of sync.

```tsx
// ❌ DON'T — Sync state
const [items, setItems] = createSignal([]);
const [count, setCount] = createSignal(0);

createEffect(() => {
  setCount(items().length);  // Syncing!
});

// ✅ DO — Derive
const [items, setItems] = createSignal([]);
const count = createMemo(() => items().length);  // Derived!
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **1.9** | Dec 2024 | Unified async/streaming SSR, onCleanup on server |
| 1.8 | 2024 | Improved TypeScript, better errors |
| **SolidStart 1.0** | May 2024 | createAsync, cache, action APIs |
| **2.0** | In Development | Signals 2.0, simplified reactivity |

### Solid 1.9 — Unified Async SSR

**Key Change**: Async and streaming rendering are now unified:
```tsx
// Before 1.9: Async and streaming were separate code paths
// After 1.9: Same code path, just different flushing strategy

// Error Boundaries now server-render in async mode
// (Previously they were client-only)
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>

// onCleanup now runs on server if branch changes
createEffect(() => {
  onCleanup(() => {
    // Now runs on server too!
  });
});
```

### SolidStart 1.0 Breaking Changes

**New Data APIs (Breaking):**
```tsx
// ❌ OLD — createRouteData, createServerData$
import { createRouteData, createServerData$ } from 'solid-start';

// ✅ NEW — createAsync, cache, action
import { createAsync, cache, action } from '@solidjs/router';

// Cache function (replaces createRouteData)
const getUser = cache(async (id: string) => {
  'use server';
  return db.user.findUnique({ where: { id } });
}, 'user');

// Use in component
function UserProfile(props: { id: string }) {
  const user = createAsync(() => getUser(props.id));
  return <Show when={user()}>{(u) => <h1>{u().name}</h1>}</Show>;
}

// Action (replaces createServerAction$)
const updateUser = action(async (formData: FormData) => {
  'use server';
  const name = formData.get('name');
  // ...
});
```

**Router Independence:**
```tsx
// SolidStart no longer requires @solidjs/router
// Any router can be used with file-system routing

// @solidjs/router and @solidjs/meta are optional
```

**Server Functions in Components (v1.1.0+):**
```tsx
// ❌ BREAKING in v1.1.0 — No inline server functions in components
function MyComponent() {
  const getData = async () => {
    'use server';
    return db.getData(); // Error!
  };
}

// ✅ DO — Define server functions outside components
const getData = cache(async () => {
  'use server';
  return db.getData();
}, 'data');

function MyComponent() {
  const data = createAsync(() => getData());
}
```

### Solid 2.0 Preview

Development is underway with focus on:
- **Signals 2.0** — Simpler reactive model
- **Props improvements** — Better destructuring, spread, binding
- **Smaller bundles** — New compiler optimizations
- **Enhanced dev tools**

The team is releasing experiments via `@solidjs/signals`:
```tsx
// Experimental: May change before 2.0
import { createSignal } from '@solidjs/signals';
```

---

## Quick Reference

| Primitive | Purpose |
|-----------|---------|
| `createSignal` | Reactive state |
| `createMemo` | Derived/computed values |
| `createEffect` | Side effects |
| `createResource` | Async data |
| `createStore` | Nested reactive state |
| `createContext` | Dependency injection |

| Control Flow | Usage |
|--------------|-------|
| `<Show when={...}>` | Conditional rendering |
| `<For each={...}>` | List rendering (keyed) |
| `<Index each={...}>` | List rendering (by index) |
| `<Switch>/<Match>` | Multiple conditions |
| `<ErrorBoundary>` | Error handling |
| `<Suspense>` | Async loading |

| Props Pattern | Usage |
|---------------|-------|
| `props.name` | Access prop |
| `splitProps(props, [...])` | Split props |
| `mergeProps(defaults, props)` | Merge with defaults |

---

## Resources

- [Official Solid.js Documentation](https://docs.solidjs.com/)
- [Solid.js Best Practices](https://www.brenelz.com/posts/solid-js-best-practices/)
- [Signals Documentation](https://docs.solidjs.com/concepts/signals)
- [SolidJS for React Developers](https://marmelab.com/blog/2025/05/28/solidjs-for-react-developper.html)
- [The Road to Solid 2.0](https://github.com/solidjs/solid/discussions/2425)
- [Solid.js in 2025](https://www.javascriptdoctor.blog/2025/05/solidjs-in-2025-build-high-performance.html)
