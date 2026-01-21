# tRPC (2025)

> **Last updated**: January 2026
> **Versions covered**: 10.x, 11.x
> **Purpose**: End-to-end typesafe APIs for TypeScript

---

## Philosophy (2025-2026)

tRPC enables **end-to-end type safety** without schemas, code generation, or runtime overhead.

**Key philosophical shifts:**
- **Zero code generation** — Types inferred directly from router
- **TypeScript as the contract** — No separate API schema
- **Move fast, break nothing** — Catch errors at compile time
- **React Query integration** — First-class hooks support
- **Server Components ready** — Direct server-side calls
- **Streaming support** — Async generators for real-time data

---

## TL;DR

- Export `AppRouter` type from server for client type inference
- Use Zod for input validation (runtime + type safety)
- Use monorepo or shared package for type sharing
- `useQuery` for reads, `useMutation` for writes
- Use `"use client"` directive for React hooks
- Server Components can call procedures directly
- Procedures = type-safe endpoints

---

## Best Practices

### Server Setup

```typescript
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import { z } from 'zod';
import superjson from 'superjson';

// Context type
export type Context = {
  user: { id: string; email: string } | null;
  db: PrismaClient;
};

const t = initTRPC.context<Context>().create({
  transformer: superjson, // For Date, Map, Set serialization
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError: error.cause instanceof z.ZodError
          ? error.cause.flatten()
          : null,
      },
    };
  },
});

export const router = t.router;
export const publicProcedure = t.procedure;

// Protected procedure middleware
const isAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({
    ctx: {
      user: ctx.user, // Now non-nullable
    },
  });
});

export const protectedProcedure = t.procedure.use(isAuthed);
```

### Router Definition

```typescript
// server/routers/user.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';

export const userRouter = router({
  // Public procedure
  getById: publicProcedure
    .input(z.object({ id: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      const user = await ctx.db.user.findUnique({
        where: { id: input.id },
      });
      if (!user) {
        throw new TRPCError({ code: 'NOT_FOUND' });
      }
      return user;
    }),

  // Protected procedure
  updateProfile: protectedProcedure
    .input(z.object({
      name: z.string().min(2).max(100),
      bio: z.string().max(500).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.user.update({
        where: { id: ctx.user.id },
        data: input,
      });
    }),

  // List with pagination
  list: publicProcedure
    .input(z.object({
      limit: z.number().min(1).max(100).default(10),
      cursor: z.string().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const items = await ctx.db.user.findMany({
        take: input.limit + 1,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });

      let nextCursor: string | undefined;
      if (items.length > input.limit) {
        const nextItem = items.pop();
        nextCursor = nextItem?.id;
      }

      return { items, nextCursor };
    }),
});
```

### Root Router

```typescript
// server/routers/_app.ts
import { router } from '../trpc';
import { userRouter } from './user';
import { postRouter } from './post';

export const appRouter = router({
  user: userRouter,
  post: postRouter,
});

// Export type for client
export type AppRouter = typeof appRouter;
```

### Client Setup (React)

```typescript
// lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/routers/_app';

export const trpc = createTRPCReact<AppRouter>();

// Provider setup
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { useState } from 'react';
import { trpc } from '@/lib/trpc';
import superjson from 'superjson';

export function TRPCProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: '/api/trpc',
          transformer: superjson,
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

### Using Queries

```typescript
// components/UserProfile.tsx
'use client';

import { trpc } from '@/lib/trpc';

export function UserProfile({ userId }: { userId: string }) {
  // Query with full type inference
  const { data: user, isLoading, error } = trpc.user.getById.useQuery(
    { id: userId },
    {
      enabled: !!userId,
      staleTime: 60 * 1000,
    }
  );

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### Using Mutations

```typescript
// components/EditProfile.tsx
'use client';

import { trpc } from '@/lib/trpc';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const updateProfileSchema = z.object({
  name: z.string().min(2).max(100),
  bio: z.string().max(500).optional(),
});

type UpdateProfileInput = z.infer<typeof updateProfileSchema>;

export function EditProfile() {
  const utils = trpc.useUtils();

  const { register, handleSubmit, formState: { errors } } = useForm<UpdateProfileInput>({
    resolver: zodResolver(updateProfileSchema),
  });

  const updateProfile = trpc.user.updateProfile.useMutation({
    onSuccess: () => {
      // Invalidate queries to refetch
      utils.user.getById.invalidate();
    },
    onError: (error) => {
      console.error('Failed to update:', error.message);
    },
  });

  const onSubmit = (data: UpdateProfileInput) => {
    updateProfile.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} placeholder="Name" />
      {errors.name && <span>{errors.name.message}</span>}

      <textarea {...register('bio')} placeholder="Bio" />

      <button type="submit" disabled={updateProfile.isPending}>
        {updateProfile.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### Infinite Queries

```typescript
// components/UserList.tsx
'use client';

import { trpc } from '@/lib/trpc';

export function UserList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = trpc.user.list.useInfiniteQuery(
    { limit: 10 },
    {
      getNextPageParam: (lastPage) => lastPage.nextCursor,
    }
  );

  return (
    <div>
      {data?.pages.map((page) =>
        page.items.map((user) => (
          <UserCard key={user.id} user={user} />
        ))
      )}

      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

### Server Components (Direct Call)

```typescript
// app/users/[id]/page.tsx
import { appRouter } from '@/server/routers/_app';
import { createContext } from '@/server/context';

export default async function UserPage({ params }: { params: { id: string } }) {
  // Create server-side caller
  const ctx = await createContext();
  const caller = appRouter.createCaller(ctx);

  // Direct procedure call (no HTTP)
  const user = await caller.user.getById({ id: params.id });

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### Optimistic Updates

```typescript
const utils = trpc.useUtils();

const likeMutation = trpc.post.like.useMutation({
  onMutate: async ({ postId }) => {
    // Cancel outgoing queries
    await utils.post.getById.cancel({ id: postId });

    // Snapshot previous value
    const previousPost = utils.post.getById.getData({ id: postId });

    // Optimistically update
    utils.post.getById.setData({ id: postId }, (old) =>
      old ? { ...old, likes: old.likes + 1 } : old
    );

    return { previousPost };
  },
  onError: (err, { postId }, context) => {
    // Rollback on error
    if (context?.previousPost) {
      utils.post.getById.setData({ id: postId }, context.previousPost);
    }
  },
  onSettled: (_, __, { postId }) => {
    // Refetch after mutation
    utils.post.getById.invalidate({ id: postId });
  },
});
```

### Next.js API Route Handler

```typescript
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/routers/_app';
import { createContext } from '@/server/context';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext,
  });

export { handler as GET, handler as POST };
```

---

## Anti-Patterns

### ❌ Forgetting "use client" Directive

**Why it's bad**: tRPC hooks only work in client components.

```typescript
// ❌ DON'T — Missing directive
function UserProfile() {
  const { data } = trpc.user.getById.useQuery({ id }); // Error!
}

// ✅ DO
'use client';

function UserProfile() {
  const { data } = trpc.user.getById.useQuery({ id }); // Works
}
```

### ❌ Skipping Input Validation

**Why it's bad**: No runtime validation, vulnerable to invalid data.

```typescript
// ❌ DON'T
createUser: publicProcedure
  .mutation(async ({ input }) => {
    // input is `unknown`, no validation!
  }),

// ✅ DO
createUser: publicProcedure
  .input(z.object({
    email: z.string().email(),
    name: z.string().min(2),
  }))
  .mutation(async ({ input }) => {
    // input is typed and validated
  }),
```

### ❌ Not Using Monorepo for Type Sharing

**Why it's bad**: Types won't be inferred, returns `any`.

```typescript
// ❌ DON'T — Separate repos without shared types
const { data } = trpc.user.getById.useQuery({ id });
// data is `any` if AppRouter type isn't shared

// ✅ DO — Monorepo or shared package
// Import AppRouter type in client setup
import type { AppRouter } from '@server/routers/_app';
export const trpc = createTRPCReact<AppRouter>();
```

### ❌ Mixing Data Fetching Patterns

**Why it's bad**: Inconsistent architecture, harder to maintain.

```typescript
// ❌ DON'T — Mix tRPC with raw fetch
const tRPCData = trpc.user.getById.useQuery({ id });
const rawData = await fetch('/api/other-endpoint'); // Inconsistent

// ✅ DO — Use tRPC consistently
const userData = trpc.user.getById.useQuery({ id });
const otherData = trpc.other.getData.useQuery(); // Consistent
```

### ❌ Missing Provider Wrapper

**Why it's bad**: Hooks throw errors.

```typescript
// ❌ DON'T — Forget provider
function App() {
  return <UserProfile />; // Hooks fail!
}

// ✅ DO
function App() {
  return (
    <TRPCProvider>
      <UserProfile />
    </TRPCProvider>
  );
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 10.x | 2024 | Stable, React Query v5 support |
| 11.x | 2025 | Improved streaming, better error handling |
| 11.x | 2025 | Enhanced Server Components support |

### Error Handling Improvements (v11)

```typescript
// Better typed error handling
const mutation = trpc.user.create.useMutation({
  onError: (error) => {
    // Access Zod validation errors
    if (error.data?.zodError) {
      const fieldErrors = error.data.zodError.fieldErrors;
      // Handle field-specific errors
    }
  },
});
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Query | `trpc.router.procedure.useQuery({ input })` |
| Mutation | `trpc.router.procedure.useMutation()` |
| Infinite query | `trpc.router.procedure.useInfiniteQuery()` |
| Invalidate | `utils.router.procedure.invalidate()` |
| Set cache | `utils.router.procedure.setData()` |
| Prefetch | `utils.router.procedure.prefetch()` |
| Server call | `appRouter.createCaller(ctx).router.procedure()` |

| Procedure Type | Use Case |
|----------------|----------|
| `publicProcedure` | No auth required |
| `protectedProcedure` | Auth required |
| `.query()` | Read operations |
| `.mutation()` | Write operations |
| `.subscription()` | Real-time updates |

| Error Code | HTTP Status | Meaning |
|------------|-------------|---------|
| `UNAUTHORIZED` | 401 | Not authenticated |
| `FORBIDDEN` | 403 | Not authorized |
| `NOT_FOUND` | 404 | Resource not found |
| `BAD_REQUEST` | 400 | Invalid input |
| `INTERNAL_SERVER_ERROR` | 500 | Server error |

---

## Resources

- [Official tRPC Documentation](https://trpc.io/)
- [tRPC Blog](https://trpc.io/blog)
- [tRPC + Next.js Guide](https://learnwebcraft.com/learn/nextjs/trpc-nextjs-type-safe-apis)
- [Better Stack tRPC Guide](https://betterstack.com/community/guides/scaling-nodejs/trpc-explained/)
- [Full-Stack TypeScript with tRPC](https://www.robinwieruch.de/react-trpc/)
- [Type Safety That Scales](https://medium.com/@Nexumo_/type-safety-that-scales-trpc-typescript-d17bd6f97d65)
