# Bun (2025)

> **Last updated**: January 2026
> **Versions covered**: 1.3+
> **Purpose**: All-in-one JavaScript runtime, bundler, package manager, and test runner

---

## Philosophy (2025-2026)

Bun is the **all-in-one JavaScript toolkit** built for speed — 4× HTTP throughput vs Node.js, native TypeScript, and built-in tooling that replaces your entire fragmented toolchain.

**Key philosophical shifts:**
- **All-in-one** — Runtime + package manager + bundler + test runner
- **4× faster HTTP** — JavaScriptCore engine vs V8
- **Native TypeScript** — Zero transpilation needed
- **Node.js compatible** — Drop-in replacement for most cases
- **Instant starts** — Fast cold starts for serverless
- **Hot reload** — Built-in with `--hot` flag (1.3+)
- **npm compatible** — 99%+ packages work in 2025

---

## TL;DR

- Use `bun run` for scripts — faster than npm/yarn
- Use `bun install` — 10× faster than npm
- Use `bun test` — Jest-compatible, no config
- Use `bun build` — bundler included
- Hot reload with `--hot` (replaces nodemon)
- Native TypeScript — no tsconfig for simple projects
- Works with most npm packages in 2025
- Consider for new projects, not critical production (yet)

---

## Best Practices

### Project Setup

```bash
# Create new project
bun init

# Install dependencies (10× faster than npm)
bun install

# Add a package
bun add express zod

# Add dev dependency
bun add -d typescript @types/express

# Run scripts
bun run dev
bun run build
bun run test
```

### Package.json

```json
{
  "name": "my-bun-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "bun --hot src/index.ts",
    "start": "bun src/index.ts",
    "build": "bun build src/index.ts --outdir=dist --target=bun",
    "test": "bun test",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "^5.4.0"
  }
}
```

### HTTP Server (Bun.serve)

```typescript
// src/index.ts
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

const users: { id: number; name: string; email: string }[] = [];
let nextId = 1;

const server = Bun.serve({
  port: 3000,
  async fetch(req) {
    const url = new URL(req.url);

    // GET /users
    if (url.pathname === '/users' && req.method === 'GET') {
      return Response.json(users);
    }

    // POST /users
    if (url.pathname === '/users' && req.method === 'POST') {
      const body = await req.json();
      const result = UserSchema.safeParse(body);

      if (!result.success) {
        return Response.json(
          { error: result.error.flatten() },
          { status: 400 }
        );
      }

      const user = { id: nextId++, ...result.data };
      users.push(user);
      return Response.json(user, { status: 201 });
    }

    // GET /users/:id
    const userMatch = url.pathname.match(/^\/users\/(\d+)$/);
    if (userMatch && req.method === 'GET') {
      const user = users.find(u => u.id === Number(userMatch[1]));
      if (!user) {
        return Response.json({ error: 'User not found' }, { status: 404 });
      }
      return Response.json(user);
    }

    // Health check
    if (url.pathname === '/health') {
      return Response.json({ status: 'ok' });
    }

    return Response.json({ error: 'Not found' }, { status: 404 });
  },
});

console.log(`Server running at http://localhost:${server.port}`);
```

### With Hono (Recommended Framework)

```typescript
// src/index.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { logger } from 'hono/logger';
import { z } from 'zod';
import { zValidator } from '@hono/zod-validator';

const app = new Hono();

// Middleware
app.use('*', logger());
app.use('/api/*', cors());

// Types
interface User {
  id: number;
  name: string;
  email: string;
}

const users: User[] = [];
let nextId = 1;

// Schemas
const CreateUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

// Routes
app.get('/api/users', (c) => {
  return c.json(users);
});

app.post(
  '/api/users',
  zValidator('json', CreateUserSchema),
  (c) => {
    const data = c.req.valid('json');
    const user: User = { id: nextId++, ...data };
    users.push(user);
    return c.json(user, 201);
  }
);

app.get('/api/users/:id', (c) => {
  const id = Number(c.req.param('id'));
  const user = users.find(u => u.id === id);

  if (!user) {
    return c.json({ error: 'User not found' }, 404);
  }

  return c.json(user);
});

app.get('/health', (c) => c.json({ status: 'ok' }));

export default {
  port: 3000,
  fetch: app.fetch,
};
```

### File Operations

```typescript
// Reading files
const file = Bun.file('data.json');
const content = await file.text();
const json = await file.json();

// Writing files
await Bun.write('output.txt', 'Hello, Bun!');
await Bun.write('data.json', JSON.stringify({ key: 'value' }));

// File metadata
const file = Bun.file('example.txt');
console.log({
  size: file.size,
  type: file.type,
  lastModified: file.lastModified,
});

// Streaming large files
const file = Bun.file('large.csv');
const stream = file.stream();
for await (const chunk of stream) {
  // Process chunk
}
```

### Environment Variables

```typescript
// bunfig.toml (optional)
// [env]
// NODE_ENV = "development"

// Access env vars
const port = Bun.env.PORT ?? 3000;
const dbUrl = Bun.env.DATABASE_URL;

// .env files are auto-loaded
// .env
// DATABASE_URL=postgres://localhost/mydb
// SECRET_KEY=supersecret
```

### SQLite (Built-in)

```typescript
import { Database } from 'bun:sqlite';

const db = new Database('mydb.sqlite');

// Create table
db.run(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);

// Prepared statements (recommended)
const insertUser = db.prepare(
  'INSERT INTO users (name, email) VALUES ($name, $email)'
);

const getUser = db.prepare('SELECT * FROM users WHERE id = ?');
const getAllUsers = db.prepare('SELECT * FROM users');

// Usage
insertUser.run({ $name: 'John', $email: 'john@example.com' });

const user = getUser.get(1);
const users = getAllUsers.all();

// Transactions
const insertMany = db.transaction((users: { name: string; email: string }[]) => {
  for (const user of users) {
    insertUser.run({ $name: user.name, $email: user.email });
  }
});

insertMany([
  { name: 'Alice', email: 'alice@example.com' },
  { name: 'Bob', email: 'bob@example.com' },
]);
```

### Testing

```typescript
// src/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Cannot divide by zero');
  return a / b;
}
```

```typescript
// src/math.test.ts
import { describe, expect, test, beforeEach, mock } from 'bun:test';
import { add, divide } from './math';

describe('math', () => {
  test('add', () => {
    expect(add(1, 2)).toBe(3);
    expect(add(-1, 1)).toBe(0);
  });

  test('divide', () => {
    expect(divide(10, 2)).toBe(5);
  });

  test('divide by zero throws', () => {
    expect(() => divide(1, 0)).toThrow('Cannot divide by zero');
  });
});

// HTTP testing
import app from './index';

describe('API', () => {
  test('GET /health', async () => {
    const response = await app.fetch(
      new Request('http://localhost/health')
    );
    expect(response.status).toBe(200);

    const data = await response.json();
    expect(data.status).toBe('ok');
  });

  test('POST /api/users', async () => {
    const response = await app.fetch(
      new Request('http://localhost/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: 'Test', email: 'test@example.com' }),
      })
    );

    expect(response.status).toBe(201);
    const user = await response.json();
    expect(user.name).toBe('Test');
  });
});
```

```bash
# Run tests
bun test

# Watch mode
bun test --watch

# Coverage
bun test --coverage
```

### Bundling

```bash
# Build for Bun runtime
bun build src/index.ts --outdir=dist --target=bun

# Build for browser
bun build src/client.ts --outdir=public --target=browser --minify

# Build for Node.js
bun build src/index.ts --outdir=dist --target=node
```

```typescript
// Complex build with plugins
await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './dist',
  target: 'bun',
  minify: true,
  sourcemap: 'external',
  splitting: true,
  plugins: [
    // Custom plugins
  ],
});
```

### WebSocket Server

```typescript
const server = Bun.serve({
  port: 3000,
  fetch(req, server) {
    // Upgrade to WebSocket
    if (server.upgrade(req)) {
      return; // Upgraded
    }
    return new Response('Upgrade failed', { status: 500 });
  },
  websocket: {
    open(ws) {
      console.log('Client connected');
      ws.subscribe('chat');
    },
    message(ws, message) {
      // Broadcast to all subscribers
      ws.publish('chat', message);
    },
    close(ws) {
      console.log('Client disconnected');
    },
  },
});
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM oven/bun:1.3 AS base
WORKDIR /app

# Install dependencies
FROM base AS deps
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

# Build
FROM deps AS build
COPY . .
RUN bun run build

# Production
FROM base AS runner
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./

USER bun
EXPOSE 3000
CMD ["bun", "run", "dist/index.js"]
```

---

## Anti-Patterns

### ❌ Using Bun for Critical Production Without Testing

**Why it's bad**: Ecosystem maturity still developing.

```typescript
// ❌ DON'T — Rush to production without validation
// Deploy directly to production banking system

// ✅ DO — Test thoroughly, have fallback plan
// 1. Run comprehensive tests
// 2. Test with actual npm dependencies
// 3. Have Node.js fallback ready
// 4. Start with non-critical services
```

### ❌ Assuming All npm Packages Work

**Why it's bad**: ~99% work, but edge cases exist.

```typescript
// ❌ DON'T — Assume native modules work
import someNativeAddon from 'native-addon';  // May not work

// ✅ DO — Test packages, check Bun compatibility
// Check: https://bun.sh/docs/runtime/nodejs-apis
```

### ❌ Not Leveraging Built-in Features

**Why it's bad**: Adding dependencies for built-in features.

```typescript
// ❌ DON'T — Use external packages for built-in features
import sqlite3 from 'sqlite3';  // External
import { jest } from '@jest/globals';  // External test runner

// ✅ DO — Use Bun built-ins
import { Database } from 'bun:sqlite';
import { test, expect } from 'bun:test';
```

### ❌ Using nodemon with Bun

**Why it's bad**: Bun has built-in hot reload.

```json
// ❌ DON'T
{
  "scripts": {
    "dev": "nodemon --exec bun src/index.ts"
  }
}

// ✅ DO
{
  "scripts": {
    "dev": "bun --hot src/index.ts"
  }
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 1.0 | Sep 2023 | Production ready |
| 1.1 | 2024 | Windows support, improved compatibility |
| 1.2 | 2024 | S3 client, improved bundler |
| 1.3 | Oct 2025 | Hot reload, shell, improved testing |

### Bun 1.3 Highlights

- **`--hot` flag**: Built-in hot reload (replaces nodemon)
- **Shell API**: `Bun.shell` for scripting
- **Improved compatibility**: 99%+ npm packages work
- **Better memory management**: Reduced footprint
- **Faster module resolution**: Improved startup times

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `bun init` | Create new project |
| `bun install` | Install dependencies |
| `bun add <pkg>` | Add package |
| `bun remove <pkg>` | Remove package |
| `bun run <script>` | Run package.json script |
| `bun <file.ts>` | Run TypeScript file |
| `bun --hot <file>` | Run with hot reload |
| `bun test` | Run tests |
| `bun build` | Bundle for production |
| `bun upgrade` | Upgrade Bun itself |

| Built-in | Usage |
|----------|-------|
| HTTP Server | `Bun.serve({ fetch })` |
| File I/O | `Bun.file()`, `Bun.write()` |
| SQLite | `import { Database } from 'bun:sqlite'` |
| Testing | `import { test } from 'bun:test'` |
| Hashing | `Bun.hash()`, `Bun.password.hash()` |
| WebSocket | `Bun.serve({ websocket })` |
| Shell | `Bun.$\`command\`` |

| Node.js | Bun Equivalent |
|---------|----------------|
| `npm install` | `bun install` |
| `npm run dev` | `bun run dev` |
| `npx` | `bunx` |
| `node file.js` | `bun file.ts` |
| nodemon | `bun --hot` |
| Jest | `bun test` |
| Webpack/esbuild | `bun build` |

---

## Resources

- [Official Bun Documentation](https://bun.sh/docs)
- [Bun vs Node.js 2025](https://strapi.io/blog/bun-vs-nodejs-performance-comparison-guide)
- [Bun Critical Evaluation 2025](https://angelo-lima.fr/en/bun-2025-critical-evaluation-javascript-runtime-alternative/)
- [Bun 1.3 Release](https://bun.sh/blog/bun-v1.3)
- [Is Bun Ready for Production?](https://devtechinsights.com/bun-vs-nodejs-production-2025/)
- [Bun for Web Development](https://www.itpathsolutions.com/bun-js-for-web-development)
