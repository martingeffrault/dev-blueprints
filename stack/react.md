# React 19 (2025)

> **Last updated**: January 2026
> **Versions covered**: 19.0, 19.1, 19.2+
> **React Compiler**: 1.0 stable

---

## Philosophy (2025-2026)

React in 2025-2026 has fundamentally shifted to a **server-first paradigm**. React Server Components (RSC) are now the default mental model, with client components being opt-in for interactivity.

**Key philosophical shifts:**
- **Server Components by default** — Client Components are opt-in with `'use client'`
- **No more useEffect for data fetching** — Use async Server Components instead
- **Actions for mutations** — Form handling via Server Actions
- **Automatic memoization** — React Compiler handles useMemo/useCallback/React.memo
- **CSS-in-JS discouraged** — Prefer CSS Modules, Tailwind, or vanilla CSS
- **Streaming & Suspense** — Progressive rendering is the norm

---

## TL;DR

- Use Server Components for data fetching, Client Components only for interactivity
- Stop writing `useMemo`, `useCallback`, `React.memo` — React Compiler does it automatically
- Use `useActionState` for forms, not manual useState + onSubmit
- Use `useOptimistic` for instant UI updates
- Use the `use()` hook to read Promises/Context in render
- Add `'use client'` only at client boundaries, not every file
- Refs are now props — `forwardRef` often unnecessary
- Upgrade to 19.2+ for `<Activity />`, `useEffectEvent`, and security patches

---

## Best Practices

### Server Components First

Prioritize Server Components by default, especially for static or content-heavy sections.

```tsx
// ✅ Server Component (default) — no directive needed
async function ProductList() {
  const products = await db.products.findMany(); // Direct DB access

  return (
    <ul>
      {products.map(p => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

### Client Boundaries Strategy

Create client boundaries only where interactivity is needed. All children of a `'use client'` component become client components.

```tsx
// ✅ Small, focused client island
'use client';
function AddToCartButton({ productId }: { productId: string }) {
  const [pending, startTransition] = useTransition();

  return (
    <button
      onClick={() => startTransition(() => addToCart(productId))}
      disabled={pending}
    >
      {pending ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}

// ✅ Server Component wraps client island
async function ProductCard({ id }: { id: string }) {
  const product = await getProduct(id);

  return (
    <div>
      <h2>{product.name}</h2>
      <p>{product.description}</p>
      <AddToCartButton productId={id} /> {/* Client island */}
    </div>
  );
}
```

### Forms with useActionState (React 19+)

```tsx
'use client';
import { useActionState } from 'react';
import { submitForm } from './actions';

function ContactForm() {
  const [state, formAction, isPending] = useActionState(submitForm, null);

  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Sending...' : 'Submit'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

### Optimistic Updates with useOptimistic

```tsx
'use client';
import { useOptimistic } from 'react';

function LikeButton({ initialLikes, postId }: Props) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    initialLikes,
    (state, newLike: number) => state + newLike
  );

  async function handleLike() {
    addOptimisticLike(1); // Instant UI update
    await likePost(postId); // Server action
  }

  return <button onClick={handleLike}>❤️ {optimisticLikes}</button>;
}
```

### The use() Hook

Read Promises or Context directly in render with automatic Suspense.

```tsx
import { use } from 'react';

function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise); // Suspends until resolved

  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}
```

### State Management Strategy (2025)

| State Type | Solution |
|------------|----------|
| Local UI state | `useState`, `useReducer` |
| Form state | `useActionState`, `useFormStatus` |
| Server data (initial) | Server Components |
| Server data (mutations/cache) | TanStack Query, SWR |
| Complex global state | Zustand, Jotai |
| Simple shared state | React Context |

---

## Anti-Patterns

### ❌ useEffect for Data Fetching

**Why it's bad**: Creates waterfalls, duplicate requests, and race conditions.

```tsx
// ❌ DON'T — Old pattern
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]);

  if (!user) return <Loading />;
  return <div>{user.name}</div>;
}

// ✅ DO — Server Component
async function UserProfile({ userId }) {
  const user = await getUser(userId);
  return <div>{user.name}</div>;
}
```

### ❌ Manual Memoization Everywhere

**Why it's bad**: React Compiler handles this automatically. Manual memoization adds noise.

```tsx
// ❌ DON'T — Unnecessary with React Compiler
const MemoizedComponent = React.memo(function MyComponent({ data }) {
  const processed = useMemo(() => expensiveCalc(data), [data]);
  const handler = useCallback(() => doSomething(data), [data]);
  return <Child data={processed} onClick={handler} />;
});

// ✅ DO — Let React Compiler optimize
function MyComponent({ data }) {
  const processed = expensiveCalc(data);
  const handler = () => doSomething(data);
  return <Child data={processed} onClick={handler} />;
}
```

**Exception**: Still use manual memoization for:
- External library APIs requiring stable references
- Class components
- Imperative refs/animation handles

### ❌ Giant Client Components

**Why it's bad**: Ships unnecessary JS to client, defeats RSC benefits.

```tsx
// ❌ DON'T — Everything is client
'use client';
function ProductPage() {
  // All this ships to client even though it's static
  return (
    <div>
      <Header />
      <ProductDetails />
      <Reviews />
      <RelatedProducts />
      <Footer />
    </div>
  );
}

// ✅ DO — Server Component with client islands
function ProductPage() {
  return (
    <div>
      <Header />                          {/* Server */}
      <ProductDetails />                   {/* Server */}
      <Reviews />                          {/* Server */}
      <AddToCartButton />                  {/* Client island */}
      <RelatedProducts />                  {/* Server */}
      <Footer />                           {/* Server */}
    </div>
  );
}
```

### ❌ Array Index as Key

**Why it's bad**: Causes bugs when list items are reordered/deleted.

```tsx
// ❌ DON'T
{items.map((item, index) => (
  <Item key={index} data={item} />
))}

// ✅ DO
{items.map(item => (
  <Item key={item.id} data={item} />
))}
```

### ❌ State Not in React State

**Why it's bad**: Variables are redeclared every render, React can't track changes.

```tsx
// ❌ DON'T
function Counter() {
  let count = 0; // Reset every render!
  return <button onClick={() => count++}>{count}</button>;
}

// ✅ DO
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### ❌ Prop Drilling Through Many Layers

**Why it's bad**: Hard to maintain, creates coupling.

```tsx
// ❌ DON'T
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <UserMenu user={user} />
    </Sidebar>
  </Layout>
</App>

// ✅ DO — Use Context or Zustand
const UserContext = createContext(null);

function App() {
  return (
    <UserContext value={user}>
      <Layout>
        <Sidebar>
          <UserMenu /> {/* Uses use(UserContext) */}
        </Sidebar>
      </Layout>
    </UserContext>
  );
}
```

### ❌ Async in useEffect Without Cleanup

**Why it's bad**: Memory leaks, race conditions, updates on unmounted components.

```tsx
// ❌ DON'T
useEffect(() => {
  fetchData().then(setData);
}, []);

// ✅ DO — If you must use useEffect
useEffect(() => {
  let cancelled = false;
  fetchData().then(data => {
    if (!cancelled) setData(data);
  });
  return () => { cancelled = true; };
}, []);

// ✅ BETTER — Use Server Component or TanStack Query
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 19.0 | Dec 2024 | RSC stable, Actions, `useActionState`, `useOptimistic`, `use()`, ref as prop |
| 19.0.1 | Jan 2025 | **Security fix** for RSC RCE vulnerability |
| 19.1 | Mar 2025 | Owner Stack debugging, refinements |
| 19.1.2 | Jun 2025 | Security patches |
| 19.2 | Oct 2025 | `<Activity />`, `useEffectEvent`, `cacheSignal`, Performance Tracks |
| 19.2.1+ | Dec 2025 | Additional security fixes |
| Compiler 1.0 | 2025 | Stable automatic memoization, 20-30% render time reduction |

### React 19.2 Highlights

**`<Activity />`** — Pre-render hidden UI without impacting performance:
```tsx
<Activity mode="hidden">
  <ExpensiveComponent /> {/* Pre-rendered but hidden */}
</Activity>
```

**`useEffectEvent`** — Extract non-reactive logic from Effects:
```tsx
function Chat({ roomId }) {
  const onMessage = useEffectEvent((msg) => {
    showNotification(msg, theme); // theme changes don't reconnect
  });

  useEffect(() => {
    const conn = connect(roomId, onMessage);
    return () => conn.disconnect();
  }, [roomId]); // Only roomId triggers reconnect
}
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Data fetching | Server Components (async) |
| Form handling | `useActionState` + Server Action |
| Optimistic UI | `useOptimistic` |
| Read Promise in render | `use(promise)` |
| Loading states | `<Suspense fallback={...}>` |
| Transitions | `useTransition`, `startTransition` |
| Refs | Pass as prop (no `forwardRef` needed) |
| Memoization | React Compiler (automatic) |
| Complex state | Zustand / Jotai |
| Server cache | TanStack Query |

---

## Resources

- [Official React Documentation](https://react.dev)
- [React 19.2 Release Notes](https://react.dev/blog/2025/10/01/react-19-2)
- [React Compiler Documentation](https://react.dev/learn/react-compiler)
- [Server Components Guide](https://react.dev/reference/rsc/server-components)
- [Josh W. Comeau - Making Sense of RSC](https://www.joshwcomeau.com/react/server-components/)
- [React Design Patterns 2025](https://www.telerik.com/blogs/react-design-patterns-best-practices)
