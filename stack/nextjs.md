# Next.js 15+ (2025)

> **Last updated**: January 2026
> **Versions covered**: 15.0 - 15.5, preview of 16.x
> **Bundler**: Turbopack (stable in 15.5+, default in 16)

---

## Philosophy (2025-2026)

Next.js in 2025-2026 has embraced **explicit control over caching** and **server-first rendering** with React Server Components as the foundation.

**Key philosophical shifts:**
- **Uncached by default** — Fetch, Route Handlers, and Router Cache no longer cache automatically
- **Server Components first** — `'use client'` is opt-in for interactivity
- **Turbopack as default** — 2-5x faster builds, stable in production
- **Partial Prerendering (PPR)** — Static shells with dynamic holes via Suspense
- **Server Actions** — Replace API routes for mutations
- **`use cache`** — Explicit, fine-grained caching control

---

## TL;DR

- Fetch is **NOT cached by default** — use `cache: 'force-cache'` or `'use cache'` explicitly
- Keep components as Server Components unless they need interactivity
- Use `loading.tsx` for streaming/Suspense fallbacks
- Use Server Actions for mutations, not API routes
- Turbopack is now stable — use `--turbopack` for 2-5x faster builds
- Use `revalidateTag()` and `revalidatePath()` for on-demand cache invalidation
- Upgrade to 15.5+ for Node.js middleware (not just Edge)
- PPR lets you mix static + dynamic in the same page

---

## Best Practices

### Project Structure (2025)

```
src/
├── app/                    # App Router
│   ├── (marketing)/        # Route group (no URL impact)
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── dashboard/
│   │   ├── page.tsx
│   │   ├── loading.tsx     # Suspense fallback
│   │   ├── error.tsx       # Error boundary
│   │   └── layout.tsx
│   ├── api/                # Route Handlers (only when needed)
│   │   └── webhooks/
│   │       └── route.ts
│   ├── layout.tsx          # Root layout
│   └── page.tsx            # Home
├── components/
│   ├── ui/                 # Generic UI (Button, Card)
│   └── features/           # Feature-specific
├── lib/                    # Utilities, db clients
├── actions/                # Server Actions
└── types/                  # TypeScript definitions
```

### Data Fetching (Server Components)

```tsx
// ✅ Direct data fetching in Server Components
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.products.findUnique({
    where: { id: params.id }
  });

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} />
    </div>
  );
}
```

### Explicit Caching (Next.js 15+)

```tsx
// ❌ NOT cached by default in Next.js 15
const data = await fetch('https://api.example.com/data');

// ✅ Explicitly cache
const data = await fetch('https://api.example.com/data', {
  cache: 'force-cache'
});

// ✅ ISR - Revalidate every 60 seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
});

// ✅ Cache with tags for on-demand invalidation
const data = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] }
});

// In a Server Action:
import { revalidateTag } from 'next/cache';
revalidateTag('products');
```

### Route Segment Config

```tsx
// app/products/page.tsx

// Force dynamic rendering (SSR)
export const dynamic = 'force-dynamic';

// Force static rendering (SSG)
export const dynamic = 'force-static';

// ISR - Revalidate every 60 seconds
export const revalidate = 60;

// Opt entire segment into caching
export const fetchCache = 'default-cache';
```

### Server Actions for Mutations

```tsx
// actions/cart.ts
'use server';

import { revalidatePath } from 'next/cache';

export async function addToCart(productId: string) {
  const session = await getSession();

  await db.cartItems.create({
    data: {
      userId: session.userId,
      productId,
      quantity: 1
    }
  });

  revalidatePath('/cart');
}

// Component usage
import { addToCart } from '@/actions/cart';

function AddToCartButton({ productId }: { productId: string }) {
  return (
    <form action={addToCart.bind(null, productId)}>
      <button type="submit">Add to Cart</button>
    </form>
  );
}
```

### Streaming with loading.tsx

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}

// Or use Suspense directly
async function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection /> {/* Streams in when ready */}
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <ChartSection />
      </Suspense>
    </div>
  );
}
```

### Client Components (Islands)

```tsx
// components/features/SearchBar.tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';

export function SearchBar() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const router = useRouter();

  function handleSearch(e: React.FormEvent) {
    e.preventDefault();
    startTransition(() => {
      router.push(`/search?q=${encodeURIComponent(query)}`);
    });
  }

  return (
    <form onSubmit={handleSearch}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <button disabled={isPending}>
        {isPending ? 'Searching...' : 'Search'}
      </button>
    </form>
  );
}
```

### Parallel Routes & Intercepting Routes

```
app/
├── @modal/              # Parallel route slot
│   └── (.)photo/[id]/   # Intercepts /photo/[id]
│       └── page.tsx     # Modal view
├── photo/[id]/
│   └── page.tsx         # Full page view
└── layout.tsx           # Renders {children} and {modal}
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  modal
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}
```

---

## Anti-Patterns

### ❌ Overusing 'use client'

**Why it's bad**: Ships unnecessary JavaScript, defeats RSC benefits.

```tsx
// ❌ DON'T — Entire page as client component
'use client';
export default function ProductPage({ product }) {
  return (
    <div>
      <Header />           {/* Static, doesn't need JS */}
      <ProductDetails />   {/* Static */}
      <Reviews />          {/* Static */}
      <Footer />           {/* Static */}
    </div>
  );
}

// ✅ DO — Server Component with client islands
export default async function ProductPage({ params }) {
  const product = await getProduct(params.id);

  return (
    <div>
      <Header />
      <ProductDetails product={product} />
      <AddToCartButton productId={product.id} /> {/* Client island */}
      <Reviews productId={product.id} />
      <Footer />
    </div>
  );
}
```

### ❌ Using API Routes for Internal Mutations

**Why it's bad**: Server Actions are simpler, type-safe, and reduce roundtrips.

```tsx
// ❌ DON'T — Unnecessary API route
// app/api/likes/route.ts
export async function POST(req: Request) {
  const { postId } = await req.json();
  await db.likes.create({ data: { postId } });
  return Response.json({ success: true });
}

// Component
async function handleLike() {
  await fetch('/api/likes', {
    method: 'POST',
    body: JSON.stringify({ postId })
  });
}

// ✅ DO — Server Action
'use server';
export async function likePost(postId: string) {
  await db.likes.create({ data: { postId } });
  revalidatePath('/posts');
}

// Component just calls the action directly
<form action={likePost.bind(null, postId)}>
  <button>Like</button>
</form>
```

### ❌ Assuming Fetch is Cached

**Why it's bad**: Next.js 15 changed defaults — fetch is uncached now.

```tsx
// ❌ DON'T — Assumes caching like Next.js 14
async function getData() {
  const res = await fetch('https://api.example.com/data');
  // This makes a new request every time!
  return res.json();
}

// ✅ DO — Explicit caching
async function getData() {
  const res = await fetch('https://api.example.com/data', {
    cache: 'force-cache',
    next: { tags: ['data'] }
  });
  return res.json();
}
```

### ❌ Not Using loading.tsx for UX

**Why it's bad**: Users see blank screens during data fetching.

```tsx
// ❌ DON'T — No loading state
// app/dashboard/page.tsx
async function DashboardPage() {
  const data = await slowDataFetch(); // 3 second wait
  return <Dashboard data={data} />;
}

// ✅ DO — Add loading.tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}
```

### ❌ Prop Drilling Through Layouts

**Why it's bad**: Layouts can't pass props to pages. Use shared data fetching patterns.

```tsx
// ❌ DON'T — Can't do this
// app/dashboard/layout.tsx
export default async function Layout({ children }) {
  const user = await getUser();
  return <div>{children(user)}</div>; // Not possible
}

// ✅ DO — Fetch in parallel or use React Context
// Option 1: Parallel fetching
// app/dashboard/page.tsx
async function DashboardPage() {
  const user = await getUser(); // Deduped if called elsewhere
  return <Dashboard user={user} />;
}

// Option 2: Context for client components
// app/providers.tsx
'use client';
export function UserProvider({ user, children }) {
  return <UserContext.Provider value={user}>{children}</UserContext.Provider>;
}
```

### ❌ Mixing Server and Client State

**Why it's bad**: Creates hydration mismatches and confusing data flow.

```tsx
// ❌ DON'T — Server data modified client-side without sync
'use client';
function ProductList({ initialProducts }) {
  const [products, setProducts] = useState(initialProducts);
  // Client state diverges from server...
}

// ✅ DO — Use TanStack Query for client mutations
'use client';
function ProductList({ initialProducts }) {
  const { data: products } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    initialData: initialProducts
  });
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 15.0 | Oct 2024 | React 19 support, caching defaults changed (uncached by default), Turbopack Dev stable |
| 15.1 | Dec 2024 | Bug fixes, stability improvements |
| 15.2 | Feb 2025 | Experimental Node.js middleware |
| 15.3 | Apr 2025 | Turbopack adoption surge (50%+ dev sessions) |
| 15.4 | Jun 2025 | Performance improvements, better error messages |
| 15.5 | Aug 2025 | **Turbopack builds beta**, Node.js middleware stable, TypeScript config improvements |
| 16.0 | Nov 2025 | Turbopack default, Cache Components replace PPR experimental flag |
| 16.1 | Dec 2025 | Refinements, File System caching stable |

### Key 15.5 Features

- **Turbopack Production Builds (Beta)**: 2-5x faster compilation
- **Node.js Middleware**: No longer limited to Edge Runtime
- **`next lint` deprecation warning**: Migrating to standalone ESLint

### Key Changes from 14 → 15

| Feature | Next.js 14 | Next.js 15 |
|---------|------------|------------|
| Fetch caching | `force-cache` default | `no-store` default |
| Route Handler caching | Cached | Uncached |
| Router Cache (pages) | Cached | Uncached |
| Turbopack | Experimental | Stable |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Static page | Default behavior or `dynamic = 'force-static'` |
| SSR (dynamic) | `dynamic = 'force-dynamic'` |
| ISR | `revalidate = 60` (seconds) |
| Cache fetch | `cache: 'force-cache'` or `next: { revalidate: 60 }` |
| On-demand revalidate | `revalidateTag('tag')` or `revalidatePath('/path')` |
| Mutations | Server Actions (`'use server'`) |
| Loading states | `loading.tsx` or `<Suspense>` |
| Error handling | `error.tsx` (client) or `global-error.tsx` |
| Middleware | `middleware.ts` at root (Node.js or Edge) |
| Fast builds | `--turbopack` flag |

---

## Rendering Decision Tree

```
Is the data user-specific or request-time dependent?
├── Yes → dynamic = 'force-dynamic' (SSR)
└── No → Is the data updated frequently?
    ├── Yes → revalidate = 60 (ISR)
    └── No → dynamic = 'force-static' (SSG)
```

---

## Resources

- [Official Next.js Documentation](https://nextjs.org/docs)
- [Next.js 15 Release Notes](https://nextjs.org/blog/next-15)
- [Next.js 15.5 Release Notes](https://nextjs.org/blog/next-15-5)
- [Caching and Revalidating Guide](https://nextjs.org/docs/app/getting-started/caching-and-revalidating)
- [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Production Checklist](https://nextjs.org/docs/app/guides/production-checklist)
