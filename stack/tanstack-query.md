# TanStack Query v5 (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.x
> **Requirement**: React 18+

---

## Philosophy (2025-2026)

TanStack Query v5 is the **standard for server state management** in React, providing automatic caching, background refetching, and seamless SSR integration.

**Key philosophical shifts:**
- **Suspense-first** — `useSuspenseQuery` is now the recommended approach
- **Unified API** — Single object parameter for all hooks
- **Renamed statuses** — `loading` → `pending`, `cacheTime` → `gcTime`
- **20% smaller** — Better tree-shaking
- **Server-side defaults** — No retries on server (0 instead of 3)
- **Streaming SSR** — Native support for React Server Components

---

## TL;DR

- Use `useSuspenseQuery` for data fetching (with React 18 Suspense)
- Pass single object to hooks: `useQuery({ queryKey, queryFn })`
- `isPending` replaces `isLoading` for initial load state
- `gcTime` replaces `cacheTime`
- Use `queryKey` as array: `['users', userId]`
- Invalidate with `queryClient.invalidateQueries({ queryKey: ['users'] })`
- Use `useMutation` for create/update/delete operations

---

## Best Practices

### Setup

```typescript
// lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,        // 1 minute
      gcTime: 5 * 60 * 1000,       // 5 minutes (was cacheTime)
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

// app/providers.tsx
'use client';
import { QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { queryClient } from '@/lib/query-client';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### Basic Query

```typescript
// hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';

export function useUser(userId: string) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: async () => {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      return response.json() as Promise<User>;
    },
    enabled: !!userId, // Only fetch if userId exists
  });
}

// Usage in component
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isPending, error } = useUser(userId);

  if (isPending) return <Skeleton />;
  if (error) return <Error message={error.message} />;

  return <div>{user.name}</div>;
}
```

### Suspense Query (Recommended)

```typescript
// hooks/useUser.ts
import { useSuspenseQuery } from '@tanstack/react-query';

export function useUser(userId: string) {
  return useSuspenseQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });
}

// Usage with Suspense boundary
function UserPage({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserProfile userId={userId} />
    </Suspense>
  );
}

function UserProfile({ userId }: { userId: string }) {
  // No need for isPending/error checks with Suspense
  const { data: user } = useUser(userId);
  return <div>{user.name}</div>;
}
```

### Mutations

```typescript
// hooks/useUpdateUser.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: { userId: string; name: string }) => {
      const response = await fetch(`/api/users/${data.userId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: data.name }),
      });
      if (!response.ok) throw new Error('Failed to update');
      return response.json();
    },
    onSuccess: (data, variables) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['users', variables.userId] });
    },
  });
}

// Usage
function EditUser({ userId }: { userId: string }) {
  const { mutate, isPending } = useUpdateUser();

  const handleSubmit = (name: string) => {
    mutate({ userId, name });
  };

  return (
    <form onSubmit={(e) => { e.preventDefault(); handleSubmit(name); }}>
      <input name="name" />
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### Optimistic Updates

```typescript
export function useLikePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: string) => likePostApi(postId),

    onMutate: async (postId) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['posts', postId] });

      // Snapshot previous value
      const previousPost = queryClient.getQueryData<Post>(['posts', postId]);

      // Optimistically update
      queryClient.setQueryData<Post>(['posts', postId], (old) =>
        old ? { ...old, likes: old.likes + 1 } : old
      );

      return { previousPost };
    },

    onError: (err, postId, context) => {
      // Rollback on error
      if (context?.previousPost) {
        queryClient.setQueryData(['posts', postId], context.previousPost);
      }
    },

    onSettled: (data, error, postId) => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: ['posts', postId] });
    },
  });
}
```

### Dependent Queries

```typescript
function UserPosts({ userId }: { userId: string }) {
  // First query
  const { data: user } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });

  // Dependent query - only runs when user is available
  const { data: posts } = useQuery({
    queryKey: ['posts', { authorId: user?.id }],
    queryFn: () => fetchPostsByAuthor(user!.id),
    enabled: !!user?.id, // Only fetch when user is loaded
  });

  return <PostList posts={posts ?? []} />;
}
```

### Infinite Queries

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

export function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) => fetchPosts({ cursor: pageParam, limit: 20 }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}

// Usage
function PostFeed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfinitePosts();

  return (
    <div>
      {data?.pages.map((page) =>
        page.posts.map((post) => <Post key={post.id} post={post} />)
      )}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

### Query Key Factory

```typescript
// lib/query-keys.ts
export const queryKeys = {
  users: {
    all: ['users'] as const,
    detail: (id: string) => ['users', id] as const,
    posts: (id: string) => ['users', id, 'posts'] as const,
  },
  posts: {
    all: ['posts'] as const,
    detail: (id: string) => ['posts', id] as const,
    infinite: ['posts', 'infinite'] as const,
  },
};

// Usage
useQuery({
  queryKey: queryKeys.users.detail(userId),
  queryFn: () => fetchUser(userId),
});

// Invalidate all user queries
queryClient.invalidateQueries({ queryKey: queryKeys.users.all });
```

---

## Anti-Patterns

### ❌ Using Overloaded Function Signatures

**Why it's bad**: Removed in v5 for consistency.

```typescript
// ❌ DON'T — v4 overloads (removed in v5)
useQuery(['users'], fetchUsers);
useQuery(['users'], fetchUsers, { staleTime: 5000 });

// ✅ DO — Single object (v5)
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  staleTime: 5000,
});
```

### ❌ Using `isLoading` for Initial Load

**Why it's bad**: Renamed in v5.

```typescript
// ❌ DON'T — Old naming
const { isLoading } = useQuery(...);
if (isLoading) return <Spinner />; // Shows on refetch too!

// ✅ DO — Use isPending for initial load
const { isPending, isFetching } = useQuery(...);
if (isPending) return <Spinner />; // Only initial load
// isFetching = true during any fetch (including background)
```

### ❌ Fetching in useEffect

**Why it's bad**: Duplicates TanStack Query's functionality.

```typescript
// ❌ DON'T
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
}

// ✅ DO
function UserProfile({ userId }) {
  const { data: user } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => fetchUser(userId),
  });
}
```

### ❌ Not Using Query Keys Properly

**Why it's bad**: Cache won't update correctly.

```typescript
// ❌ DON'T — Static key ignores params
useQuery({
  queryKey: ['user'], // Same key for all users!
  queryFn: () => fetchUser(userId),
});

// ✅ DO — Include dependencies in key
useQuery({
  queryKey: ['users', userId],
  queryFn: () => fetchUser(userId),
});
```

### ❌ Mutating Query Data Directly

**Why it's bad**: Bypasses cache, causes inconsistencies.

```typescript
// ❌ DON'T
const { data } = useQuery(...);
data.name = 'New Name'; // Direct mutation!

// ✅ DO — Use setQueryData or invalidate
queryClient.setQueryData(['users', userId], (old) => ({
  ...old,
  name: 'New Name',
}));
```

---

## v4 to v5 Migration

| v4 | v5 |
|----|-----|
| `isLoading` (initial) | `isPending` |
| `isLoading` (any fetch) | `isPending && isFetching` |
| `cacheTime` | `gcTime` |
| `keepPreviousData` | `placeholderData: keepPreviousData` |
| `useQuery(key, fn, options)` | `useQuery({ queryKey, queryFn, ...options })` |
| `useMutation(fn, options)` | `useMutation({ mutationFn, ...options })` |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Fetch data | `useQuery({ queryKey, queryFn })` |
| Fetch with Suspense | `useSuspenseQuery({ queryKey, queryFn })` |
| Mutate data | `useMutation({ mutationFn })` |
| Invalidate cache | `queryClient.invalidateQueries({ queryKey })` |
| Prefetch | `queryClient.prefetchQuery({ queryKey, queryFn })` |
| Set cache directly | `queryClient.setQueryData(queryKey, data)` |
| Infinite scroll | `useInfiniteQuery({ queryKey, queryFn, getNextPageParam })` |
| Cancel query | `queryClient.cancelQueries({ queryKey })` |

---

## Resources

- [Official TanStack Query Documentation](https://tanstack.com/query/latest)
- [Migration Guide v4 to v5](https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5)
- [TkDodo's Blog](https://tkdodo.eu/blog/practical-react-query)
- [React Query Devtools](https://tanstack.com/query/latest/docs/framework/react/devtools)
