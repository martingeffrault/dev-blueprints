# Node.js 22+ (2025)

> **Last updated**: January 2026
> **Versions covered**: 22 LTS, 23, 24 LTS
> **Key feature**: Native TypeScript execution

---

## Philosophy (2025-2026)

Node.js in 2025-2026 has embraced **native TypeScript support**, **ES modules as default**, and **reduced dependency on external packages** with built-in alternatives.

**Key philosophical shifts:**
- **TypeScript native** — Run `.ts` files directly with `--experimental-strip-types`
- **ES Modules default** — CommonJS is legacy
- **Built-in test runner** — No need for Jest/Mocha for simple tests
- **Built-in watch mode** — No need for nodemon
- **Fetch API stable** — No need for node-fetch
- **WebStreams API** — Aligned with browser standards
- **Permission model** — Fine-grained security controls

---

## TL;DR

- Use Node.js 22 LTS or 24 LTS for production
- Run TypeScript directly: `node --experimental-strip-types app.ts`
- Use ES modules (`"type": "module"` in package.json)
- Use built-in `node:test` instead of Jest for simple projects
- Use built-in `--watch` instead of nodemon
- Use native `fetch()` instead of axios/node-fetch
- Use `--env-file` instead of dotenv for simple cases

---

## Best Practices

### Project Setup (2025)

```json
// package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "engines": {
    "node": ">=22.0.0"
  },
  "scripts": {
    "dev": "node --watch --env-file=.env src/index.ts",
    "start": "node src/index.ts",
    "test": "node --test",
    "test:watch": "node --test --watch"
  }
}
```

### Native TypeScript Execution

```bash
# Node.js 22.6+ (experimental flag)
node --experimental-strip-types app.ts

# Node.js 22.18+ / 23+ (flag optional in some configs)
node app.ts

# With env file
node --experimental-strip-types --env-file=.env app.ts
```

```typescript
// app.ts — runs directly!
interface User {
  id: string;
  name: string;
}

const user: User = {
  id: '1',
  name: 'Alice'
};

console.log(user.name);
```

### ES Modules

```typescript
// ✅ ES Modules (recommended)
import { readFile } from 'node:fs/promises';
import { join } from 'node:path';

const data = await readFile(join(process.cwd(), 'data.json'), 'utf-8');
```

```typescript
// package.json imports for cleaner paths
// package.json
{
  "imports": {
    "#lib/*": "./src/lib/*.js",
    "#utils/*": "./src/utils/*.js"
  }
}

// Usage
import { db } from '#lib/database';
import { logger } from '#utils/logger';
```

### Built-in Test Runner

```typescript
// tests/user.test.ts
import { describe, it, before, after, mock } from 'node:test';
import assert from 'node:assert/strict';
import { createUser, getUser } from '../src/user.js';

describe('User module', () => {
  before(() => {
    // Setup
  });

  after(() => {
    // Cleanup
  });

  it('should create a user', async () => {
    const user = await createUser({ name: 'Alice' });
    assert.equal(user.name, 'Alice');
    assert.ok(user.id);
  });

  it('should mock dependencies', async () => {
    const mockFetch = mock.fn(() =>
      Promise.resolve({ json: () => Promise.resolve({ id: '1' }) })
    );

    // Use mock in test
    const result = await getUser('1', mockFetch);
    assert.equal(mockFetch.mock.callCount(), 1);
  });
});
```

```bash
# Run tests
node --test

# Run with coverage
node --test --experimental-test-coverage

# Watch mode
node --test --watch

# Glob patterns
node --test "tests/**/*.test.ts"
```

### Built-in Watch Mode

```bash
# Development with auto-restart
node --watch src/index.ts

# Watch specific paths
node --watch-path=./src --watch-path=./config src/index.ts

# Preserve output between restarts
node --watch --watch-preserve-output src/index.ts
```

### Native Fetch API

```typescript
// ✅ No need for node-fetch or axios
const response = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${process.env.API_KEY}`
  },
  body: JSON.stringify({ name: 'Alice' })
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const user = await response.json();
```

### Environment Variables

```bash
# Built-in env file support (no dotenv needed)
node --env-file=.env src/index.ts

# Multiple env files (later ones override)
node --env-file=.env --env-file=.env.local src/index.ts
```

```typescript
// Type-safe env access
const config = {
  port: parseInt(process.env.PORT ?? '3000', 10),
  dbUrl: process.env.DATABASE_URL!,
  nodeEnv: process.env.NODE_ENV ?? 'development'
};
```

### Permission Model

```bash
# Restrict file system access
node --experimental-permission --allow-fs-read=/app/data app.ts

# Restrict network access
node --experimental-permission --allow-net=api.example.com app.ts

# Full restriction (explicit permissions required)
node --experimental-permission app.ts
```

### Async Local Storage (Context)

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';

interface RequestContext {
  requestId: string;
  userId?: string;
}

const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

// Middleware sets context
function requestMiddleware(req, res, next) {
  const context: RequestContext = {
    requestId: crypto.randomUUID(),
    userId: req.headers['x-user-id']
  };

  asyncLocalStorage.run(context, () => next());
}

// Access anywhere in call stack
function logMessage(message: string) {
  const context = asyncLocalStorage.getStore();
  console.log(`[${context?.requestId}] ${message}`);
}
```

### HTTP/2 and HTTP/3

```typescript
import { createSecureServer } from 'node:http2';
import { readFileSync } from 'node:fs';

const server = createSecureServer({
  key: readFileSync('server-key.pem'),
  cert: readFileSync('server-cert.pem')
});

server.on('stream', (stream, headers) => {
  stream.respond({
    ':status': 200,
    'content-type': 'text/html'
  });
  stream.end('<h1>Hello HTTP/2!</h1>');
});

server.listen(443);
```

---

## Anti-Patterns

### ❌ Using CommonJS in New Projects

**Why it's bad**: ES Modules are the standard, better tooling support.

```javascript
// ❌ DON'T — CommonJS
const fs = require('fs');
const { join } = require('path');
module.exports = { myFunction };

// ✅ DO — ES Modules
import fs from 'node:fs/promises';
import { join } from 'node:path';
export { myFunction };
```

### ❌ Using External Packages for Built-ins

**Why it's bad**: Unnecessary dependencies, security surface, update burden.

```bash
# ❌ DON'T
npm install node-fetch dotenv nodemon uuid jest

# ✅ DO — Use built-ins
# fetch() — built-in
# --env-file — built-in
# --watch — built-in
# crypto.randomUUID() — built-in
# node:test — built-in
```

### ❌ Blocking the Event Loop

**Why it's bad**: Blocks all concurrent operations.

```typescript
// ❌ DON'T — Sync operations block
import { readFileSync } from 'node:fs';
const data = readFileSync('large-file.json', 'utf-8'); // Blocks!

// ✅ DO — Async operations
import { readFile } from 'node:fs/promises';
const data = await readFile('large-file.json', 'utf-8');
```

### ❌ Not Handling Errors Properly

**Why it's bad**: Unhandled rejections crash Node.js.

```typescript
// ❌ DON'T — Unhandled promise rejection
async function fetchData() {
  const res = await fetch(url); // Could throw!
  return res.json();
}

// ✅ DO — Handle errors
async function fetchData() {
  try {
    const res = await fetch(url);
    if (!res.ok) {
      throw new Error(`HTTP ${res.status}`);
    }
    return res.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error; // Re-throw or handle appropriately
  }
}
```

### ❌ Using `var` or Untyped Code

**Why it's bad**: Type safety prevents bugs.

```typescript
// ❌ DON'T
var users = [];
function getUser(id) {
  return users.find(u => u.id == id);
}

// ✅ DO
interface User {
  id: string;
  name: string;
}

const users: User[] = [];

function getUser(id: string): User | undefined {
  return users.find(u => u.id === id);
}
```

### ❌ Hardcoding Configuration

**Why it's bad**: Prevents deployment flexibility.

```typescript
// ❌ DON'T
const db = new Database('localhost:5432');
const port = 3000;

// ✅ DO
const config = {
  dbHost: process.env.DB_HOST ?? 'localhost',
  dbPort: parseInt(process.env.DB_PORT ?? '5432', 10),
  port: parseInt(process.env.PORT ?? '3000', 10)
};
```

### ❌ Not Using Proper Signals for Shutdown

**Why it's bad**: Abrupt termination loses data, breaks connections.

```typescript
// ✅ DO — Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');

  // Stop accepting new connections
  server.close();

  // Close database connections
  await db.close();

  // Exit
  process.exit(0);
});

process.on('SIGINT', async () => {
  // Same handling for Ctrl+C
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 22.0 | Apr 2024 | V8 12.4, ESM `require()` behind flag, WebSocket client |
| 22.6 | Jul 2024 | `--experimental-strip-types` (TypeScript) |
| 22.18 | Jul 2025 | Native TypeScript support unflagged |
| 23.0 | Oct 2024 | ESM `require()` default, stabilized `--run` flag |
| 23.6 | Jan 2025 | TypeScript direct execution improvements |
| 24.0 | Oct 2025 | LTS release, reduced external dependencies |

### Version Strategy

| Type | Version | Use Case |
|------|---------|----------|
| LTS (Active) | 24.x | Production (recommended) |
| LTS (Maintenance) | 22.x | Production (stable) |
| Current | 25.x | Testing new features |

---

## Quick Reference

| Task | Solution |
|------|----------|
| Run TypeScript | `node --experimental-strip-types app.ts` |
| Watch mode | `node --watch app.ts` |
| Env file | `node --env-file=.env app.ts` |
| Run tests | `node --test` |
| Test coverage | `node --test --experimental-test-coverage` |
| HTTP requests | Native `fetch()` |
| Generate UUID | `crypto.randomUUID()` |
| Hash data | `crypto.createHash('sha256').update(data).digest('hex')` |
| Graceful shutdown | Handle `SIGTERM`, `SIGINT` |

---

## Security Best Practices

1. **Keep Node.js updated** — Use latest LTS
2. **Use permission model** — `--experimental-permission`
3. **Validate all input** — Use Zod or similar
4. **Avoid eval/new Function** — Code injection risk
5. **Use parameterized queries** — SQL injection prevention
6. **Set security headers** — Helmet.js or manual
7. **Rate limit APIs** — Prevent abuse
8. **Audit dependencies** — `npm audit`

---

## Resources

- [Official Node.js Documentation](https://nodejs.org/docs/latest/api/)
- [Node.js 22 Release Notes](https://nodejs.org/en/blog/release/v22.0.0)
- [Node.js 23 Release Notes](https://nodejs.org/en/blog/release/v23.0.0)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [Modern Node.js Patterns 2025](https://kashw1n.com/blog/nodejs-2025/)
