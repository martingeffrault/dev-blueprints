# Remix (2025)

> **Last updated**: January 2026
> **Versions covered**: Remix 2.x (React Router v7 framework mode)
> **Purpose**: Full-stack React framework with server-first data loading

---

## Philosophy (2025-2026)

Remix is the **server-first React framework** that embraces web fundamentals — loaders for reads, actions for writes, progressive enhancement, and streaming HTML.

**Key philosophical shifts:**
- **Server-first** — Data loading happens on the server
- **Loaders/Actions model** — Simple read/write pattern
- **Progressive enhancement** — Works without JavaScript
- **35% less JS** — Smaller bundles than alternatives
- **React Router v7 convergence** — Same patterns in both
- **Edge-native** — Deploy to any Fetch-compatible runtime
- **Remix 3 coming** — Full-stack framework package (2026)

---

## TL;DR

- Use `loader` for reading data (GET requests)
- Use `action` for mutations (POST/PUT/DELETE)
- Use `<Form>` component for progressive enhancement
- Keep logic in loaders/actions, not useEffect
- Use error boundaries at route level
- No client-side data fetching for initial load
- Stream long data with `defer`
- Don't call endpoints from other endpoints — call services

---

## Best Practices

### Project Structure

```
my-remix-app/
├── app/
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   └── Card.tsx
│   │   └── features/
│   │       └── UserCard.tsx
│   ├── routes/
│   │   ├── _index.tsx           # Home page (/)
│   │   ├── about.tsx            # /about
│   │   ├── users._index.tsx     # /users
│   │   ├── users.$id.tsx        # /users/:id
│   │   ├── users.$id.edit.tsx   # /users/:id/edit
│   │   ├── api.users.tsx        # /api/users (resource route)
│   │   └── blog/
│   │       ├── _layout.tsx      # Blog layout
│   │       ├── _index.tsx       # /blog
│   │       └── $slug.tsx        # /blog/:slug
│   ├── services/                # Business logic
│   │   ├── user.server.ts
│   │   └── post.server.ts
│   ├── models/                  # Data access
│   │   ├── user.server.ts
│   │   └── post.server.ts
│   ├── utils/
│   │   ├── session.server.ts
│   │   └── db.server.ts
│   ├── root.tsx
│   └── entry.server.tsx
├── public/
├── remix.config.js
└── package.json
```

### Loader (Data Loading)

```typescript
// app/routes/users._index.tsx
import { json } from '@remix-run/node';
import type { LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData } from '@remix-run/react';
import { getUsers } from '~/services/user.server';

export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const page = Number(url.searchParams.get('page') ?? 1);
  const search = url.searchParams.get('search') ?? '';

  const { users, total } = await getUsers({ page, search });

  return json({
    users,
    pagination: {
      page,
      total,
      pages: Math.ceil(total / 10),
    },
  });
}

export default function UsersPage() {
  const { users, pagination } = useLoaderData<typeof loader>();

  return (
    <div>
      <h1>Users ({pagination.total})</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
      <Pagination {...pagination} />
    </div>
  );
}
```

### Action (Mutations)

```typescript
// app/routes/users.new.tsx
import { json, redirect } from '@remix-run/node';
import type { ActionFunctionArgs, LoaderFunctionArgs } from '@remix-run/node';
import { Form, useActionData, useNavigation } from '@remix-run/react';
import { createUser } from '~/services/user.server';
import { requireAuth } from '~/utils/session.server';

export async function loader({ request }: LoaderFunctionArgs) {
  await requireAuth(request);
  return json({});
}

export async function action({ request }: ActionFunctionArgs) {
  await requireAuth(request);

  const formData = await request.formData();
  const name = formData.get('name')?.toString();
  const email = formData.get('email')?.toString();

  // Validation
  const errors: Record<string, string> = {};
  if (!name || name.length < 2) {
    errors.name = 'Name must be at least 2 characters';
  }
  if (!email || !email.includes('@')) {
    errors.email = 'Valid email is required';
  }

  if (Object.keys(errors).length > 0) {
    return json({ errors, values: { name, email } }, { status: 400 });
  }

  const user = await createUser({ name, email });
  return redirect(`/users/${user.id}`);
}

export default function NewUserPage() {
  const actionData = useActionData<typeof action>();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <div>
        <label>
          Name
          <input
            name="name"
            defaultValue={actionData?.values?.name}
          />
        </label>
        {actionData?.errors?.name && (
          <span className="error">{actionData.errors.name}</span>
        )}
      </div>

      <div>
        <label>
          Email
          <input
            name="email"
            type="email"
            defaultValue={actionData?.values?.email}
          />
        </label>
        {actionData?.errors?.email && (
          <span className="error">{actionData.errors.email}</span>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create User'}
      </button>
    </Form>
  );
}
```

### Streaming with defer

```typescript
// app/routes/dashboard.tsx
import { defer } from '@remix-run/node';
import type { LoaderFunctionArgs } from '@remix-run/node';
import { Await, useLoaderData } from '@remix-run/react';
import { Suspense } from 'react';
import { getStats, getRecentActivity, getNotifications } from '~/services/dashboard.server';

export async function loader({ request }: LoaderFunctionArgs) {
  // Critical data - await immediately
  const stats = await getStats();

  // Non-critical data - stream later
  const activityPromise = getRecentActivity();
  const notificationsPromise = getNotifications();

  return defer({
    stats,
    activity: activityPromise,
    notifications: notificationsPromise,
  });
}

export default function Dashboard() {
  const { stats, activity, notifications } = useLoaderData<typeof loader>();

  return (
    <div className="dashboard">
      {/* Critical data renders immediately */}
      <StatsCards stats={stats} />

      {/* Non-critical data streams in */}
      <Suspense fallback={<ActivitySkeleton />}>
        <Await resolve={activity}>
          {(data) => <ActivityFeed activity={data} />}
        </Await>
      </Suspense>

      <Suspense fallback={<NotificationsSkeleton />}>
        <Await resolve={notifications}>
          {(data) => <NotificationsList notifications={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

### Service Layer

```typescript
// app/services/user.server.ts
import { db } from '~/utils/db.server';
import type { User } from '@prisma/client';

interface GetUsersParams {
  page?: number;
  search?: string;
  limit?: number;
}

export async function getUsers({
  page = 1,
  search = '',
  limit = 10,
}: GetUsersParams) {
  const where = search
    ? {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ],
      }
    : {};

  const [users, total] = await Promise.all([
    db.user.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
    }),
    db.user.count({ where }),
  ]);

  return { users, total };
}

export async function getUser(id: string): Promise<User | null> {
  return db.user.findUnique({ where: { id } });
}

export async function createUser(data: { name: string; email: string }): Promise<User> {
  return db.user.create({ data });
}

export async function updateUser(
  id: string,
  data: Partial<{ name: string; email: string }>
): Promise<User> {
  return db.user.update({ where: { id }, data });
}

export async function deleteUser(id: string): Promise<void> {
  await db.user.delete({ where: { id } });
}
```

### Session Management

```typescript
// app/utils/session.server.ts
import { createCookieSessionStorage, redirect } from '@remix-run/node';
import { db } from './db.server';

const sessionStorage = createCookieSessionStorage({
  cookie: {
    name: '__session',
    httpOnly: true,
    maxAge: 60 * 60 * 24 * 7, // 1 week
    path: '/',
    sameSite: 'lax',
    secrets: [process.env.SESSION_SECRET!],
    secure: process.env.NODE_ENV === 'production',
  },
});

export async function getSession(request: Request) {
  return sessionStorage.getSession(request.headers.get('Cookie'));
}

export async function getUserId(request: Request): Promise<string | null> {
  const session = await getSession(request);
  return session.get('userId');
}

export async function requireAuth(request: Request) {
  const userId = await getUserId(request);
  if (!userId) {
    throw redirect('/login');
  }
  return userId;
}

export async function getUser(request: Request) {
  const userId = await getUserId(request);
  if (!userId) return null;
  return db.user.findUnique({ where: { id: userId } });
}

export async function createUserSession(userId: string, redirectTo: string) {
  const session = await sessionStorage.getSession();
  session.set('userId', userId);
  return redirect(redirectTo, {
    headers: {
      'Set-Cookie': await sessionStorage.commitSession(session),
    },
  });
}

export async function logout(request: Request) {
  const session = await getSession(request);
  return redirect('/login', {
    headers: {
      'Set-Cookie': await sessionStorage.destroySession(session),
    },
  });
}
```

### Error Boundaries

```typescript
// app/routes/users.$id.tsx
import { json } from '@remix-run/node';
import type { LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData, useRouteError, isRouteErrorResponse } from '@remix-run/react';
import { getUser } from '~/services/user.server';

export async function loader({ params }: LoaderFunctionArgs) {
  const user = await getUser(params.id!);

  if (!user) {
    throw json({ message: 'User not found' }, { status: 404 });
  }

  return json({ user });
}

export default function UserPage() {
  const { user } = useLoaderData<typeof loader>();
  return <UserProfile user={user} />;
}

// Route-specific error boundary
export function ErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    return (
      <div className="error">
        <h1>{error.status}</h1>
        <p>{error.data.message}</p>
      </div>
    );
  }

  return (
    <div className="error">
      <h1>Something went wrong</h1>
      <p>{error instanceof Error ? error.message : 'Unknown error'}</p>
    </div>
  );
}
```

### Resource Routes (API)

```typescript
// app/routes/api.users.tsx
import { json } from '@remix-run/node';
import type { LoaderFunctionArgs, ActionFunctionArgs } from '@remix-run/node';
import { getUsers, createUser } from '~/services/user.server';

export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url);
  const search = url.searchParams.get('search') ?? '';

  const { users } = await getUsers({ search });
  return json(users);
}

export async function action({ request }: ActionFunctionArgs) {
  if (request.method !== 'POST') {
    return json({ error: 'Method not allowed' }, { status: 405 });
  }

  const body = await request.json();
  const user = await createUser(body);
  return json(user, { status: 201 });
}
```

### Root Layout

```typescript
// app/root.tsx
import {
  Links,
  LiveReload,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
} from '@remix-run/react';
import type { LinksFunction, LoaderFunctionArgs } from '@remix-run/node';
import { json } from '@remix-run/node';
import { getUser } from '~/utils/session.server';

import styles from '~/styles/global.css?url';

export const links: LinksFunction = () => [
  { rel: 'stylesheet', href: styles },
];

export async function loader({ request }: LoaderFunctionArgs) {
  const user = await getUser(request);
  return json({ user });
}

export default function App() {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet />
        <ScrollRestoration />
        <Scripts />
        <LiveReload />
      </body>
    </html>
  );
}
```

---

## Anti-Patterns

### ❌ Client-Side Data Fetching on Load

**Why it's bad**: Misses SSR, causes loading waterfalls.

```typescript
// ❌ DON'T — useEffect for initial data
export default function UsersPage() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch('/api/users').then(r => r.json()).then(setUsers);
  }, []);

  return <UserList users={users} />;
}

// ✅ DO — Loader
export async function loader() {
  const users = await getUsers();
  return json({ users });
}

export default function UsersPage() {
  const { users } = useLoaderData<typeof loader>();
  return <UserList users={users} />;
}
```

### ❌ Endpoint Calling Another Endpoint

**Why it's bad**: Extra serialization, latency, authentication overhead.

```typescript
// ❌ DON'T
export async function loader() {
  const response = await fetch('/api/users');  // HTTP call to self!
  return json({ users: await response.json() });
}

// ✅ DO — Call service directly
export async function loader() {
  const { users } = await getUsers();
  return json({ users });
}
```

### ❌ Forgetting Progressive Enhancement

**Why it's bad**: Breaks without JavaScript.

```typescript
// ❌ DON'T — onClick with fetch
export default function DeleteButton({ userId }: { userId: string }) {
  const handleDelete = async () => {
    await fetch(`/api/users/${userId}`, { method: 'DELETE' });
  };
  return <button onClick={handleDelete}>Delete</button>;
}

// ✅ DO — Form with action
export default function DeleteButton({ userId }: { userId: string }) {
  return (
    <Form method="post" action={`/users/${userId}/delete`}>
      <button type="submit">Delete</button>
    </Form>
  );
}
```

### ❌ Fat Route Modules

**Why it's bad**: Hard to test, hard to reuse.

```typescript
// ❌ DON'T — All logic in route
export async function loader() {
  // 50 lines of database queries
  // 30 lines of data transformation
  // 20 lines of permission checks
}

// ✅ DO — Extract to services
export async function loader({ request }: LoaderFunctionArgs) {
  const user = await requireAuth(request);
  const data = await getDashboardData(user.id);
  return json(data);
}
```

### ❌ Redux for All State

**Why it's bad**: Server-side data doesn't need client state management.

```typescript
// ❌ DON'T — Redux for server data
const users = useSelector(state => state.users);

// ✅ DO — Loader data + Form for mutations
const { users } = useLoaderData<typeof loader>();
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.9 | 2024 | Single Fetch preview (`unstable_singleFetch`) |
| 2.13 | 2024 | Single Fetch stabilized (`v3_singleFetch`) |
| **2.x → RRv7** | **2024-2025** | **Remix becomes React Router v7 framework mode** |
| 3.0 | 2026 (planned) | Full-stack `remix` package |

### Remix → React Router v7 Migration

"What we planned to release as Remix v3 is now going to be released as React Router v7."

**Migration Overview:**
- If all Remix v2 future flags enabled, migration is mostly dependency updates
- Codemod available for automatic migration

**Package Changes:**
```bash
# Before (Remix)
@remix-run/node
@remix-run/react
@remix-run/dev

# After (React Router v7)
react-router  # Unified package
@react-router/node  # Server adapters
```

**Import Changes:**
```typescript
// Before (Remix)
import { json, redirect, LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData, Form } from '@remix-run/react';

// After (React Router v7)
import { useLoaderData, Form, redirect } from 'react-router';
import type { Route } from './+types/route-name';
```

### Single Fetch (Default in React Router v7)

**Single Fetch** replaces multiple parallel loader calls with a single request:

```typescript
// Enable in Remix v2 (required before RRv7 migration)
// remix.config.js or vite.config.ts
export default {
  future: {
    v3_singleFetch: true,  // Stabilized flag
  },
};
```

**Breaking Changes with Single Fetch:**
```typescript
// ❌ json() and defer() are DEPRECATED in React Router v7
import { json, defer } from '@remix-run/node';

export async function loader() {
  return json({ users });  // ❌ Deprecated
}

// ✅ Return naked objects instead
export async function loader() {
  return { users };  // ✅ Direct return
}

// ✅ Streaming with promises (replaces defer)
export async function loader() {
  return {
    critical: await getCriticalData(),
    streamed: getSlowData(),  // Promise streams automatically
  };
}
```

**Required entry.server.tsx Changes:**
```typescript
// Must use streaming renderer (not renderToString)
import { renderToPipeableStream } from 'react-dom/server';

// Client-side must wrap hydrateRoot in startTransition
import { startTransition } from 'react';
import { hydrateRoot } from 'react-dom/client';

startTransition(() => {
  hydrateRoot(document, <App />);
});
```

### Future Flags to Enable Before Migration

```typescript
// remix.config.js
export default {
  future: {
    v3_fetcherPersist: true,
    v3_relativeSplatPath: true,
    v3_throwAbortReason: true,
    v3_singleFetch: true,
    v3_lazyRouteDiscovery: true,
    v3_routeConfig: true,
  },
};
```

---

## Quick Reference

| Concept | Usage |
|---------|-------|
| Data loading | `loader` function (server) |
| Mutations | `action` function (server) |
| Access loader data | `useLoaderData<typeof loader>()` |
| Access action data | `useActionData<typeof action>()` |
| Form state | `useNavigation().state` |
| Params | `useParams()` |
| Search params | `useSearchParams()` |
| Navigate | `useNavigate()` or `redirect()` |
| Stream data | `defer()` + `<Await>` |

| File Convention | Route |
|-----------------|-------|
| `_index.tsx` | `/` (index route) |
| `about.tsx` | `/about` |
| `users._index.tsx` | `/users` |
| `users.$id.tsx` | `/users/:id` |
| `users.$id.edit.tsx` | `/users/:id/edit` |
| `api.users.tsx` | `/api/users` (resource route) |

---

## Resources

- [Official Remix Documentation](https://remix.run/docs)
- [Remix vs Next.js 2025](https://strapi.io/blog/next-js-vs-remix-2025-developer-framework-comparison-guide)
- [Remix Best Practices](https://codilime.com/blog/project-in-remix-best-practices/)
- [Remix Error Handling](https://betterstack.com/community/guides/scaling-nodejs/error-handling-remix/)
- [React Router v7](https://reactrouter.com/)
- [Remix Jam 2025 Recap](https://remix.run/blog/remix-jam-2025-recap)
