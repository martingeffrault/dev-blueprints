# Deno 2 (2025)

> **Last updated**: January 2026
> **Versions covered**: 2.x (2.2, 2.3, 2.4)
> **Purpose**: Secure TypeScript/JavaScript runtime with built-in tooling

---

## Philosophy (2025-2026)

Deno 2 is the **secure-by-default runtime** with native TypeScript, built-in tooling, and now full npm compatibility. It's batteries-included with formatting, linting, testing, and OpenTelemetry.

**Key philosophical shifts:**
- **Security first** — Explicit permissions required
- **TypeScript native** — Zero config needed
- **npm compatible** — Works with package.json in v2
- **Built-in tooling** — fmt, lint, test, bench, doc
- **Web standards** — fetch, WebSocket, Promises built-in
- **OpenTelemetry** — Built-in observability (2.2+)
- **Package management** — deno.json or package.json

---

## TL;DR

- Use explicit permissions (`--allow-net`, `--allow-read`)
- Use `deno.json` for configuration (replaces multiple config files)
- Import maps for clean imports
- npm packages work with `npm:` specifier
- Built-in formatter, linter, test runner
- OpenTelemetry built-in for observability
- JSR (JavaScript Registry) for Deno-first packages
- Works with package.json for Node.js migration

---

## Best Practices

### Project Structure

```
my-deno-app/
├── src/
│   ├── main.ts
│   ├── routes/
│   │   ├── users.ts
│   │   └── posts.ts
│   ├── services/
│   │   ├── user.ts
│   │   └── post.ts
│   ├── db/
│   │   └── client.ts
│   └── utils/
│       └── validation.ts
├── tests/
│   ├── user.test.ts
│   └── integration/
│       └── api.test.ts
├── deno.json
├── deno.lock
└── .env
```

### Configuration (deno.json)

```json
{
  "name": "@myorg/my-app",
  "version": "1.0.0",
  "exports": "./src/main.ts",
  "tasks": {
    "dev": "deno run --watch --allow-net --allow-read --allow-env src/main.ts",
    "start": "deno run --allow-net --allow-read --allow-env src/main.ts",
    "test": "deno test --allow-read --allow-env",
    "lint": "deno lint",
    "fmt": "deno fmt",
    "check": "deno check src/main.ts"
  },
  "imports": {
    "@std/": "jsr:@std/",
    "hono": "jsr:@hono/hono@^4",
    "zod": "npm:zod@^3.23",
    "@/": "./src/"
  },
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true
  },
  "lint": {
    "rules": {
      "tags": ["recommended"],
      "include": ["no-unused-vars"]
    }
  },
  "fmt": {
    "lineWidth": 100,
    "indentWidth": 2,
    "singleQuote": true
  }
}
```

### HTTP Server (Deno.serve)

```typescript
// src/main.ts
import { z } from "zod";

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

interface User {
  id: number;
  name: string;
  email: string;
}

const users: User[] = [];
let nextId = 1;

Deno.serve({ port: 3000 }, async (req) => {
  const url = new URL(req.url);

  // GET /users
  if (url.pathname === "/users" && req.method === "GET") {
    return Response.json(users);
  }

  // POST /users
  if (url.pathname === "/users" && req.method === "POST") {
    const body = await req.json();
    const result = UserSchema.safeParse(body);

    if (!result.success) {
      return Response.json(
        { error: result.error.flatten() },
        { status: 400 }
      );
    }

    const user: User = { id: nextId++, ...result.data };
    users.push(user);
    return Response.json(user, { status: 201 });
  }

  // GET /users/:id
  const userMatch = url.pathname.match(/^\/users\/(\d+)$/);
  if (userMatch && req.method === "GET") {
    const user = users.find((u) => u.id === Number(userMatch[1]));
    if (!user) {
      return Response.json({ error: "User not found" }, { status: 404 });
    }
    return Response.json(user);
  }

  // Health check
  if (url.pathname === "/health") {
    return Response.json({ status: "ok" });
  }

  return Response.json({ error: "Not found" }, { status: 404 });
});

console.log("Server running at http://localhost:3000");
```

### With Hono (Recommended Framework)

```typescript
// src/main.ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { logger } from "hono/logger";
import { z } from "zod";
import { zValidator } from "@hono/zod-validator";

const app = new Hono();

// Middleware
app.use("*", logger());
app.use("/api/*", cors());

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
app.get("/api/users", (c) => {
  return c.json(users);
});

app.post(
  "/api/users",
  zValidator("json", CreateUserSchema),
  (c) => {
    const data = c.req.valid("json");
    const user: User = { id: nextId++, ...data };
    users.push(user);
    return c.json(user, 201);
  }
);

app.get("/api/users/:id", (c) => {
  const id = Number(c.req.param("id"));
  const user = users.find((u) => u.id === id);

  if (!user) {
    return c.json({ error: "User not found" }, 404);
  }

  return c.json(user);
});

app.get("/health", (c) => c.json({ status: "ok" }));

Deno.serve({ port: 3000 }, app.fetch);
```

### Environment Variables

```typescript
// Environment variables
const port = Deno.env.get("PORT") ?? "3000";
const dbUrl = Deno.env.get("DATABASE_URL");

// Load .env file (built-in in Deno 2.3+)
// or use @std/dotenv
import { load } from "@std/dotenv";
await load({ export: true });

// Now env vars from .env are available
console.log(Deno.env.get("SECRET_KEY"));
```

### File Operations

```typescript
// Reading files
const text = await Deno.readTextFile("./data.json");
const json = JSON.parse(text);

// Writing files
await Deno.writeTextFile("./output.txt", "Hello, Deno!");
await Deno.writeTextFile(
  "./data.json",
  JSON.stringify({ key: "value" }, null, 2)
);

// File info
const fileInfo = await Deno.stat("./example.txt");
console.log({
  size: fileInfo.size,
  isFile: fileInfo.isFile,
  modified: fileInfo.mtime,
});

// Reading directory
for await (const entry of Deno.readDir("./src")) {
  console.log(entry.name, entry.isDirectory);
}

// Binary files
const bytes = await Deno.readFile("./image.png");
await Deno.writeFile("./copy.png", bytes);
```

### Database (with npm packages)

```typescript
// Using Prisma (npm:)
import { PrismaClient } from "npm:@prisma/client";

const prisma = new PrismaClient();

async function getUsers() {
  return prisma.user.findMany();
}

// Using postgres.js
import postgres from "npm:postgres";

const sql = postgres(Deno.env.get("DATABASE_URL")!);

const users = await sql`SELECT * FROM users`;
```

### Testing

```typescript
// tests/user.test.ts
import { assertEquals, assertThrows } from "@std/assert";
import { describe, it, beforeEach } from "@std/testing/bdd";

describe("User Service", () => {
  beforeEach(() => {
    // Reset state
  });

  it("should create a user", () => {
    const user = createUser({ name: "John", email: "john@example.com" });
    assertEquals(user.name, "John");
    assertEquals(user.email, "john@example.com");
  });

  it("should throw on invalid email", () => {
    assertThrows(
      () => createUser({ name: "John", email: "invalid" }),
      Error,
      "Invalid email"
    );
  });
});

// tests/integration/api.test.ts
Deno.test("GET /health returns ok", async () => {
  const response = await fetch("http://localhost:3000/health");
  const data = await response.json();

  assertEquals(response.status, 200);
  assertEquals(data.status, "ok");
});
```

```bash
# Run tests
deno test

# Run with permissions
deno test --allow-net --allow-read

# Run specific file
deno test tests/user.test.ts

# Watch mode
deno test --watch
```

### Linting (Built-in)

```typescript
// deno.json lint configuration
{
  "lint": {
    "rules": {
      "tags": ["recommended"],
      "include": ["no-unused-vars", "no-explicit-any"],
      "exclude": ["no-empty"]
    }
  }
}

// Deno 2.2+ Plugin system
// deno.json
{
  "lint": {
    "plugins": ["./lint-plugins/my-rule.ts"]
  }
}
```

```bash
# Run linter
deno lint

# Fix issues
deno lint --fix
```

### OpenTelemetry (Built-in 2.2+)

```typescript
// instrumentation.ts (auto-discovered)
// Automatic instrumentation for console.log, Deno.serve, fetch

// Enable with flag:
// deno run --unstable-otel main.ts

// Or in deno.json
{
  "unstable": ["otel"]
}
```

```bash
# Run with OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_SERVICE_NAME=my-app \
deno run --unstable-otel main.ts
```

### Permissions

```bash
# Network access
deno run --allow-net main.ts
deno run --allow-net=example.com,api.example.com main.ts

# File read
deno run --allow-read main.ts
deno run --allow-read=./data,./config main.ts

# File write
deno run --allow-write=./output main.ts

# Environment variables
deno run --allow-env main.ts
deno run --allow-env=DATABASE_URL,SECRET_KEY main.ts

# Run subprocess
deno run --allow-run main.ts
deno run --allow-run=git,npm main.ts

# All permissions (development only!)
deno run -A main.ts
```

### Deploying

```dockerfile
# Dockerfile
FROM denoland/deno:2.3.0

WORKDIR /app

# Cache dependencies
COPY deno.json deno.lock ./
RUN deno cache --lock=deno.lock src/main.ts

# Copy source
COPY . .

# Compile (optional, for faster startup)
RUN deno compile --allow-net --allow-read --allow-env -o app src/main.ts

USER deno
EXPOSE 3000

CMD ["./app"]
# Or: CMD ["deno", "run", "--allow-net", "--allow-read", "--allow-env", "src/main.ts"]
```

### deno compile (Single Binary)

```bash
# Compile to single executable
deno compile --allow-net --allow-read -o myapp src/main.ts

# Cross-compile
deno compile --target x86_64-unknown-linux-gnu -o myapp-linux src/main.ts
deno compile --target x86_64-apple-darwin -o myapp-macos src/main.ts
deno compile --target x86_64-pc-windows-msvc -o myapp.exe src/main.ts

# Include files
deno compile --include data.json --allow-read -o myapp src/main.ts
```

---

## Anti-Patterns

### ❌ Using -A in Production

**Why it's bad**: Defeats Deno's security model.

```bash
# ❌ DON'T
deno run -A src/main.ts

# ✅ DO — Explicit permissions
deno run --allow-net --allow-read=./data --allow-env=DATABASE_URL src/main.ts
```

### ❌ Scattered URL Imports

**Why it's bad**: Version management nightmare.

```typescript
// ❌ DON'T — URLs everywhere
import { serve } from "https://deno.land/std@0.220.0/http/server.ts";
import { z } from "https://deno.land/x/zod@v3.22.0/mod.ts";

// ✅ DO — Use import maps in deno.json
// deno.json: { "imports": { "@std/http": "jsr:@std/http@^1" } }
import { serve } from "@std/http";
```

### ❌ Ignoring deno.lock

**Why it's bad**: Inconsistent dependencies across environments.

```bash
# ❌ DON'T — Skip lockfile
deno run --no-lock main.ts

# ✅ DO — Use lockfile
deno cache --lock=deno.lock main.ts
deno run --lock=deno.lock main.ts
```

### ❌ Not Using deno check

**Why it's bad**: Type errors slip through.

```bash
# ❌ DON'T — Skip type checking
deno run main.ts  # May have type errors

# ✅ DO — Check types first
deno check main.ts && deno run main.ts

# Or in CI
deno check src/main.ts
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.0 | Oct 2024 | npm compatibility, package.json support |
| 2.1 | Dec 2024 | Long-term support, improved stability |
| 2.2 | Feb 2025 | OpenTelemetry, lint plugins, new lint rules |
| 2.3 | May 2025 | Local npm packages, improved deno compile |
| 2.4 | Aug 2025 | deno bundle returns, enhanced tooling |

### Deno 2 Key Features

- **npm Compatibility**: `npm:` specifier works seamlessly
- **package.json Support**: Drop-in Node.js migration
- **JSR**: JavaScript Registry for Deno-first packages
- **OpenTelemetry**: Built-in observability
- **Lint Plugins**: Extensible linting (2.2+)
- **Local npm Packages**: Use workspace packages (2.3+)

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `deno run file.ts` | Run TypeScript file |
| `deno run --watch` | Run with auto-reload |
| `deno check` | Type check |
| `deno fmt` | Format code |
| `deno lint` | Lint code |
| `deno test` | Run tests |
| `deno bench` | Run benchmarks |
| `deno compile` | Create binary |
| `deno install` | Install script as command |
| `deno task <name>` | Run task from deno.json |

| Permission | Flag |
|------------|------|
| Network | `--allow-net` |
| Read files | `--allow-read` |
| Write files | `--allow-write` |
| Environment | `--allow-env` |
| Run subprocess | `--allow-run` |
| FFI | `--allow-ffi` |
| All (dev only) | `-A` |

| Import Specifier | Source |
|------------------|--------|
| `jsr:@std/...` | JavaScript Registry |
| `npm:package` | npm package |
| `https://...` | URL (avoid) |
| `./local.ts` | Local file |

---

## Resources

- [Official Deno Documentation](https://deno.com/manual)
- [Deno 2 Announcement](https://deno.com/blog/v2.0)
- [Deno 2.2 Release](https://blog.bagwanpankaj.com/deno/what-is-new-in-deno-2-2)
- [Deno 2.3 Release](https://deno.com/blog/v2.3)
- [JSR - JavaScript Registry](https://jsr.io/)
- [Deno in 2025 Use Cases](https://medium.com/@asierr/deno-in-2025-the-best-use-cases-and-when-you-should-still-choose-node-abdfad1a3fb5)
- [Deno vs Node.js 2025](https://devsdata.com/deno-vs-node-which-one-is-better-in-2025/)
