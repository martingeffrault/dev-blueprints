# Hono (2025)

> **Last updated**: January 2026
> **Versions covered**: 4.x
> **Purpose**: Ultrafast web framework built on Web Standards

---

## Philosophy (2025-2026)

Hono is the **ultrafast, lightweight web framework** built on Web Standards. It runs on any JavaScript runtime — Cloudflare Workers, Deno, Bun, Node.js, AWS Lambda, and more.

**Key philosophical shifts:**
- **Web Standards first** — Same Request/Response as browsers
- **Runtime agnostic** — Write once, run anywhere
- **Ultrafast routing** — RegExpRouter with zero linear loops
- **Tiny footprint** — `hono/tiny` under 14kB, zero dependencies
- **First-class TypeScript** — Full type inference
- **Edge-native** — Perfect for Cloudflare, Deno Deploy, etc.

---

## TL;DR

- Don't create "Rails-like controllers" — write handlers inline
- Use middleware composition for shared logic
- Use Zod validator (`@hono/zod-validator`) for type-safe validation
- Use `hono/factory` if you need to extract handlers
- HonoX for full-stack file-based routing
- Same code runs on Workers, Bun, Deno, Node.js
- JSX support for server-side rendering

---

## Best Practices

### Project Structure

```
my-hono-app/
├── src/
│   ├── index.ts           # App entry point
│   ├── routes/
│   │   ├── index.ts       # Route aggregator
│   │   ├── users.ts       # User routes
│   │   ├── posts.ts       # Post routes
│   │   └── auth.ts        # Auth routes
│   ├── middleware/
│   │   ├── auth.ts        # Auth middleware
│   │   └── logger.ts      # Custom logger
│   ├── services/
│   │   ├── user.ts        # User business logic
│   │   └── post.ts        # Post business logic
│   ├── db/
│   │   └── client.ts      # Database client
│   ├── schemas/           # Zod schemas
│   │   ├── user.ts
│   │   └── post.ts
│   └── types/
│       └── index.ts
├── wrangler.toml          # Cloudflare config
├── package.json
└── tsconfig.json
```

### Basic App

```typescript
// src/index.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { logger } from 'hono/logger';
import { prettyJSON } from 'hono/pretty-json';
import { secureHeaders } from 'hono/secure-headers';
import { users } from './routes/users';
import { posts } from './routes/posts';
import { auth } from './routes/auth';

const app = new Hono();

// Global middleware
app.use('*', logger());
app.use('*', secureHeaders());
app.use('*', prettyJSON());
app.use('/api/*', cors());

// Routes
app.route('/api/users', users);
app.route('/api/posts', posts);
app.route('/api/auth', auth);

// Health check
app.get('/health', (c) => c.json({ status: 'ok' }));

// 404 handler
app.notFound((c) => {
  return c.json({ error: 'Not Found' }, 404);
});

// Error handler
app.onError((err, c) => {
  console.error(err);
  return c.json({ error: 'Internal Server Error' }, 500);
});

export default app;
```

### Route File with Handlers

```typescript
// src/routes/users.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { authMiddleware } from '../middleware/auth';
import { getUserById, getUsers, createUser, updateUser, deleteUser } from '../services/user';

const app = new Hono();

// Schemas
const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8),
});

const UpdateUserSchema = z.object({
  name: z.string().min(2).max(100).optional(),
  email: z.string().email().optional(),
});

const QuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().positive().max(100).default(10),
});

// GET /api/users
app.get(
  '/',
  zValidator('query', QuerySchema),
  async (c) => {
    const { page, limit } = c.req.valid('query');
    const { users, total } = await getUsers({ page, limit });

    return c.json({
      data: users,
      pagination: { page, limit, total },
    });
  }
);

// GET /api/users/:id
app.get('/:id', async (c) => {
  const id = c.req.param('id');
  const user = await getUserById(id);

  if (!user) {
    return c.json({ error: 'User not found' }, 404);
  }

  return c.json(user);
});

// POST /api/users
app.post(
  '/',
  zValidator('json', CreateUserSchema),
  async (c) => {
    const data = c.req.valid('json');
    const user = await createUser(data);

    return c.json(user, 201);
  }
);

// PUT /api/users/:id (protected)
app.put(
  '/:id',
  authMiddleware,
  zValidator('json', UpdateUserSchema),
  async (c) => {
    const id = c.req.param('id');
    const data = c.req.valid('json');
    const user = await updateUser(id, data);

    if (!user) {
      return c.json({ error: 'User not found' }, 404);
    }

    return c.json(user);
  }
);

// DELETE /api/users/:id (protected)
app.delete('/:id', authMiddleware, async (c) => {
  const id = c.req.param('id');
  await deleteUser(id);

  return c.json({ success: true });
});

export { app as users };
```

### Middleware

```typescript
// src/middleware/auth.ts
import { Context, Next } from 'hono';
import { HTTPException } from 'hono/http-exception';
import { verify } from 'hono/jwt';

export async function authMiddleware(c: Context, next: Next) {
  const authHeader = c.req.header('Authorization');

  if (!authHeader?.startsWith('Bearer ')) {
    throw new HTTPException(401, { message: 'Unauthorized' });
  }

  const token = authHeader.slice(7);

  try {
    const payload = await verify(token, c.env.JWT_SECRET);
    c.set('user', payload);
    await next();
  } catch {
    throw new HTTPException(401, { message: 'Invalid token' });
  }
}

// src/middleware/rateLimit.ts
import { Context, Next } from 'hono';

const requests = new Map<string, { count: number; reset: number }>();

export function rateLimit(limit: number, windowMs: number) {
  return async (c: Context, next: Next) => {
    const ip = c.req.header('CF-Connecting-IP') ?? 'unknown';
    const now = Date.now();
    const record = requests.get(ip);

    if (!record || now > record.reset) {
      requests.set(ip, { count: 1, reset: now + windowMs });
    } else if (record.count >= limit) {
      return c.json({ error: 'Too many requests' }, 429);
    } else {
      record.count++;
    }

    await next();
  };
}
```

### Type-Safe Environment (Cloudflare Workers)

```typescript
// src/index.ts
import { Hono } from 'hono';

type Bindings = {
  DATABASE: D1Database;
  KV_CACHE: KVNamespace;
  JWT_SECRET: string;
};

type Variables = {
  user: { id: string; email: string };
};

const app = new Hono<{ Bindings: Bindings; Variables: Variables }>();

app.get('/users', async (c) => {
  // Type-safe access to bindings
  const db = c.env.DATABASE;
  const cache = c.env.KV_CACHE;

  const cached = await cache.get('users');
  if (cached) {
    return c.json(JSON.parse(cached));
  }

  const { results } = await db
    .prepare('SELECT * FROM users')
    .all();

  await cache.put('users', JSON.stringify(results), { expirationTtl: 3600 });

  return c.json(results);
});

export default app;
```

### JSX/TSX for SSR

```tsx
// src/index.tsx
import { Hono } from 'hono';
import { html } from 'hono/html';

const app = new Hono();

const Layout = ({ children }: { children: any }) => html`
  <!DOCTYPE html>
  <html>
    <head>
      <title>My Hono App</title>
      <script src="https://unpkg.com/htmx.org@1.9.10"></script>
    </head>
    <body>
      ${children}
    </body>
  </html>
`;

const UserList = ({ users }: { users: { id: number; name: string }[] }) => (
  <ul>
    {users.map((user) => (
      <li key={user.id}>{user.name}</li>
    ))}
  </ul>
);

app.get('/', async (c) => {
  const users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
  ];

  return c.html(
    <Layout>
      <h1>Users</h1>
      <UserList users={users} />
    </Layout>
  );
});

export default app;
```

### Factory Pattern (For Extracted Handlers)

```typescript
// src/routes/users.ts
import { Hono } from 'hono';
import { createFactory } from 'hono/factory';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';

const factory = createFactory();

// Extract handlers using factory (preserves types)
const getUser = factory.createHandlers(async (c) => {
  const id = c.req.param('id');
  // ...
  return c.json({ id, name: 'User' });
});

const createUser = factory.createHandlers(
  zValidator('json', z.object({ name: z.string() })),
  async (c) => {
    const data = c.req.valid('json');
    // ...
    return c.json(data, 201);
  }
);

const app = new Hono();
app.get('/:id', ...getUser);
app.post('/', ...createUser);

export { app as users };
```

### Deployment (Cloudflare Workers)

```toml
# wrangler.toml
name = "my-hono-app"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[vars]
ENVIRONMENT = "production"

[[d1_databases]]
binding = "DATABASE"
database_name = "my-database"
database_id = "xxx-xxx-xxx"

[[kv_namespaces]]
binding = "KV_CACHE"
id = "xxx-xxx-xxx"
```

```bash
# Deploy
npx wrangler deploy
```

### Deployment (Bun)

```typescript
// src/index.ts
import app from './app';

export default {
  port: 3000,
  fetch: app.fetch,
};
```

```bash
bun run src/index.ts
```

### Deployment (Node.js)

```typescript
// src/index.ts
import { serve } from '@hono/node-server';
import app from './app';

serve({
  fetch: app.fetch,
  port: 3000,
});
```

---

## Anti-Patterns

### ❌ Rails-Like Controllers

**Why it's bad**: Path parameters can't be inferred without complex generics.

```typescript
// ❌ DON'T — Controller class
class UserController {
  static async getUser(c: Context) {
    const id = c.req.param('id');  // Type is string | undefined
    // ...
  }
}
app.get('/users/:id', UserController.getUser);

// ✅ DO — Inline handlers
app.get('/users/:id', async (c) => {
  const id = c.req.param('id');  // Properly typed
  // ...
});

// ✅ OR — Use factory if you must extract
const getUser = factory.createHandlers(async (c) => {
  const id = c.req.param('id');
  // ...
});
app.get('/users/:id', ...getUser);
```

### ❌ Not Using Validator Middleware

**Why it's bad**: Missing validation, no type inference.

```typescript
// ❌ DON'T — Manual parsing
app.post('/users', async (c) => {
  const body = await c.req.json();  // unknown type
  if (!body.name) { /* ... */ }
  // ...
});

// ✅ DO — Zod validator
app.post(
  '/users',
  zValidator('json', CreateUserSchema),
  async (c) => {
    const body = c.req.valid('json');  // Typed!
    // ...
  }
);
```

### ❌ Not Handling Errors Globally

**Why it's bad**: Inconsistent error responses.

```typescript
// ❌ DON'T — Try-catch everywhere
app.get('/users/:id', async (c) => {
  try {
    // ...
  } catch (e) {
    return c.json({ error: 'Error' }, 500);
  }
});

// ✅ DO — Global error handler
app.onError((err, c) => {
  if (err instanceof HTTPException) {
    return c.json({ error: err.message }, err.status);
  }
  console.error(err);
  return c.json({ error: 'Internal Server Error' }, 500);
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **4.11** | Dec 2025 | NotFoundResponse type, middleware type fixes, base URL in hc |
| **4.7** | Aug 2025 | RPC improvements, streaming enhancements |
| **4.0** | 2024 | Improved middleware, better types |
| **HonoX** | 2025 | File-based routing, islands architecture |

### Hono RPC — Type-Safe API Client

```typescript
// server.ts — Export AppType
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';

const app = new Hono()
  .get('/users', async (c) => {
    const users = await getUsers();
    return c.json(users);
  })
  .get('/users/:id', async (c) => {
    const user = await getUserById(c.req.param('id'));
    return c.json(user);
  })
  .post(
    '/users',
    zValidator('json', z.object({ name: z.string(), email: z.string().email() })),
    async (c) => {
      const data = c.req.valid('json');
      const user = await createUser(data);
      return c.json(user, 201);
    }
  );

// Export type for client
export type AppType = typeof app;
export default app;
```

```typescript
// client.ts — Fully typed API calls
import { hc } from 'hono/client';
import type { AppType } from './server';

const client = hc<AppType>('http://localhost:3000');

// Type-safe requests — input and output types inferred!
const users = await client.users.$get();
const user = await client.users[':id'].$get({ param: { id: '123' } });
const newUser = await client.users.$post({
  json: { name: 'John', email: 'john@example.com' }
});

// TypeScript errors if you pass wrong types!
```

**RPC Base URL Types (v4.11+):**
```typescript
// NEW — Type-safe base URL for SWR/TanStack Query keys
import { hc } from 'hono/client';

const client = hc<AppType, 'http://api.example.com'>('http://api.example.com');

// URL is now typed as 'http://api.example.com/users'
const url = client.users.$url();
```

**RPC NotFoundResponse (v4.11+):**
```typescript
// NEW — Properly typed 404 responses
app.get('/users/:id', async (c) => {
  const user = await getUserById(c.req.param('id'));
  if (!user) {
    return c.notFound(); // Type is now inferred on client
  }
  return c.json(user);
});
```

**RPC Performance Tips:**
```typescript
// If your IDE is slow, split routes into multiple files
// and chain them to preserve types:

// routes/users.ts
const users = new Hono()
  .get('/', async (c) => c.json([]))
  .post('/', async (c) => c.json({}));

// routes/posts.ts
const posts = new Hono()
  .get('/', async (c) => c.json([]));

// index.ts — Chain to preserve types
const app = new Hono()
  .route('/users', users)
  .route('/posts', posts);

export type AppType = typeof app;
```

### HonoX — File-Based Meta-Framework

```
# HonoX Project Structure
my-honox-app/
├── app/
│   ├── routes/
│   │   ├── _renderer.tsx    # Root renderer (layout)
│   │   ├── index.tsx        # /
│   │   ├── about.tsx        # /about
│   │   ├── posts/
│   │   │   ├── index.tsx    # /posts
│   │   │   └── [id].tsx     # /posts/:id
│   │   └── api/
│   │       └── users.ts     # /api/users
│   ├── islands/             # Client-side interactive components
│   │   ├── Counter.tsx
│   │   └── SearchBar.tsx
│   └── components/          # Server components (no hydration)
│       └── Header.tsx
├── vite.config.ts
└── package.json
```

**HonoX Renderer (Layout):**
```tsx
// app/routes/_renderer.tsx
import { jsxRenderer } from 'hono/jsx-renderer';

export default jsxRenderer(({ children, title }) => (
  <html>
    <head>
      <title>{title ?? 'My HonoX App'}</title>
      <script type="module" src="/app/client.ts"></script>
    </head>
    <body>
      <header>
        <nav>
          <a href="/">Home</a>
          <a href="/about">About</a>
        </nav>
      </header>
      <main>{children}</main>
    </body>
  </html>
));
```

**HonoX Page:**
```tsx
// app/routes/index.tsx
import { createRoute } from 'honox/factory';
import Counter from '../islands/Counter';

export default createRoute((c) => {
  return c.render(
    <div>
      <h1>Welcome to HonoX</h1>
      <p>This is server-rendered HTML</p>

      {/* This island will be hydrated on the client */}
      <Counter initial={0} />
    </div>,
    { title: 'Home' }
  );
});
```

**HonoX Islands (Client-Side Hydration):**
```tsx
// app/islands/Counter.tsx
import { useState } from 'hono/jsx';

// Only this component's JS is sent to the browser
export default function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
      <button onClick={() => setCount(count - 1)}>-</button>
    </div>
  );
}
```

**HonoX API Route:**
```typescript
// app/routes/api/users.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';

const app = new Hono()
  .get('/', async (c) => {
    const users = await getUsers();
    return c.json(users);
  })
  .post(
    '/',
    zValidator('json', z.object({ name: z.string() })),
    async (c) => {
      const data = c.req.valid('json');
      return c.json(await createUser(data), 201);
    }
  );

export default app;
```

### Streaming Responses

```typescript
// Server-Sent Events (SSE)
import { Hono } from 'hono';
import { streamSSE } from 'hono/streaming';

const app = new Hono();

app.get('/sse', async (c) => {
  return streamSSE(c, async (stream) => {
    let id = 0;
    while (true) {
      const data = await getLatestData();
      await stream.writeSSE({
        data: JSON.stringify(data),
        event: 'update',
        id: String(id++),
      });
      await stream.sleep(1000);
    }
  });
});

// Streaming text
app.get('/stream', async (c) => {
  return c.streamText(async (stream) => {
    for (const chunk of generateChunks()) {
      await stream.write(chunk);
      await stream.sleep(100);
    }
  });
});
```

**Note**: SSE is not yet supported in the RPC client. Use EventSource directly for SSE endpoints:
```typescript
// Client-side SSE (use EventSource, not hc)
const eventSource = new EventSource('/sse');
eventSource.addEventListener('update', (event) => {
  const data = JSON.parse(event.data);
  console.log('Update:', data);
});
```

---

## Quick Reference

| Feature | Usage |
|---------|-------|
| Create app | `new Hono()` |
| Route | `app.get('/path', handler)` |
| Params | `c.req.param('id')` |
| Query | `c.req.query('page')` |
| Body (JSON) | `c.req.json()` |
| Body (validated) | `c.req.valid('json')` |
| Headers | `c.req.header('Authorization')` |
| Response JSON | `c.json({ data })` |
| Response HTML | `c.html(<Component />)` |
| Set variable | `c.set('user', user)` |
| Get variable | `c.get('user')` |
| Environment | `c.env.DATABASE` |

| Middleware | Import |
|------------|--------|
| CORS | `import { cors } from 'hono/cors'` |
| Logger | `import { logger } from 'hono/logger'` |
| JWT | `import { jwt } from 'hono/jwt'` |
| Bearer Auth | `import { bearerAuth } from 'hono/bearer-auth'` |
| Secure Headers | `import { secureHeaders } from 'hono/secure-headers'` |
| Cache | `import { cache } from 'hono/cache'` |

| Runtime | Adapter |
|---------|---------|
| Cloudflare Workers | Default export |
| Bun | `export default { fetch: app.fetch }` |
| Deno | `Deno.serve(app.fetch)` |
| Node.js | `@hono/node-server` |
| AWS Lambda | `@hono/aws-lambda` |

---

## Resources

- [Official Hono Documentation](https://hono.dev/docs/)
- [Hono Best Practices](https://hono.dev/docs/guides/best-practices)
- [Hono GitHub](https://github.com/honojs/hono)
- [HonoX Meta-Framework](https://github.com/honojs/honox)
- [Build Production-Ready Apps with Hono](https://www.freecodecamp.org/news/build-production-ready-web-apps-with-hono/)
- [Hono vs Express 2025](https://khmercoder.com/@stoic/articles/25847997)
