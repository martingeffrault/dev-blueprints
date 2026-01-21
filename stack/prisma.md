# Prisma (2025)

> **Last updated**: January 2026
> **Versions covered**: 6.x, 7.x
> **Purpose**: Next-generation ORM for Node.js & TypeScript

---

## Philosophy (2025-2026)

Prisma is evolving into a **faster, TypeScript-native ORM** with the Query Compiler replacing the Rust engine.

**Key philosophical shifts:**
- **Rust → TypeScript** — Query Compiler is now TypeScript/WASM (v6.16+)
- **3.4x faster queries** — Significant performance improvement
- **90% smaller bundle** — From ~14MB to 1.6MB
- **No native binaries** — Simpler deployment, better edge support
- **TypedSQL** — Raw SQL with full type safety
- **Prisma Accelerate** — Edge-ready connection pooling

---

## TL;DR

- Use `include` for eager loading — avoid N+1 queries
- Use singleton pattern for PrismaClient in dev (hot reload)
- Use `migrate` for production, `db push` for prototyping
- Never edit already-deployed migrations
- Use `select` to fetch only needed fields
- TypedSQL for complex raw queries with type safety
- Use Prisma Accelerate for serverless/edge

---

## Best Practices

### Client Setup (Singleton Pattern)

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

export default prisma;
```

### Schema Definition

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  tags      Tag[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([authorId])
  @@index([published])
}

model Profile {
  id     String @id @default(cuid())
  bio    String?
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### Eager Loading (Avoid N+1)

```typescript
// ❌ DON'T — N+1 query problem
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id },
  });
  // This executes N+1 queries!
}

// ✅ DO — Eager loading with include
const usersWithPosts = await prisma.user.findMany({
  include: {
    posts: true,
    profile: true,
  },
});

// ✅ DO — Nested includes
const usersWithDetails = await prisma.user.findMany({
  include: {
    posts: {
      include: {
        tags: true,
      },
      where: {
        published: true,
      },
      orderBy: {
        createdAt: 'desc',
      },
      take: 5,
    },
  },
});
```

### Selective Queries with Select

```typescript
// ✅ Only fetch needed fields (better performance)
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    // posts not included — smaller payload
  },
});

// ✅ Combine select with relations
const posts = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    author: {
      select: {
        name: true,
        email: true,
      },
    },
  },
});
```

### Transactions

```typescript
// ✅ Interactive transactions
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: 'alice@example.com', name: 'Alice' },
  });

  const post = await tx.post.create({
    data: {
      title: 'First Post',
      authorId: user.id,
    },
  });

  return { user, post };
});

// ✅ Batch transactions (faster, atomic)
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { email: 'bob@example.com', name: 'Bob' } }),
  prisma.post.create({ data: { title: 'Another Post', authorId: 'existing-id' } }),
]);
```

### TypedSQL (New in v5.7+)

```typescript
// prisma/sql/getUserPosts.sql
-- @param {String} $1:userId
SELECT
  p.id,
  p.title,
  p.created_at,
  u.name as author_name
FROM posts p
JOIN users u ON p.author_id = u.id
WHERE u.id = $1
ORDER BY p.created_at DESC;

// Usage — fully typed!
import { getUserPosts } from '@prisma/client/sql';

const posts = await prisma.$queryRawTyped(getUserPosts(userId));
// posts is typed based on your SQL query
```

### Soft Deletes with Middleware

```typescript
// Middleware for soft deletes
prisma.$use(async (params, next) => {
  if (params.model === 'User') {
    if (params.action === 'delete') {
      params.action = 'update';
      params.args['data'] = { deletedAt: new Date() };
    }
    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      if (params.args.data !== undefined) {
        params.args.data['deletedAt'] = new Date();
      } else {
        params.args['data'] = { deletedAt: new Date() };
      }
    }
  }
  return next(params);
});
```

### Client Extensions

```typescript
// Extend Prisma Client with custom methods
const prismaWithExtensions = prisma.$extends({
  model: {
    user: {
      async findByEmail(email: string) {
        return prisma.user.findUnique({ where: { email } });
      },
      async softDelete(id: string) {
        return prisma.user.update({
          where: { id },
          data: { deletedAt: new Date() },
        });
      },
    },
  },
  result: {
    user: {
      fullName: {
        needs: { firstName: true, lastName: true },
        compute(user) {
          return `${user.firstName} ${user.lastName}`;
        },
      },
    },
  },
});

// Usage
const user = await prismaWithExtensions.user.findByEmail('alice@example.com');
```

### Migrations Best Practices

```bash
# Development: Create and apply migration
npx prisma migrate dev --name add_user_profile

# Production: Apply pending migrations
npx prisma migrate deploy

# Prototyping only: Push without migration history
npx prisma db push

# View migration status
npx prisma migrate status

# Reset database (dev only!)
npx prisma migrate reset
```

### Relation Load Strategy (v5.7+)

```typescript
// Choose JOIN strategy for better performance
const users = await prisma.user.findMany({
  relationLoadStrategy: 'join', // or 'query'
  include: {
    posts: true,
  },
});
```

---

## Anti-Patterns

### ❌ N+1 Query Problem

**Why it's bad**: Exponentially more database queries.

```typescript
// ❌ DON'T
const users = await prisma.user.findMany();
const usersWithPosts = await Promise.all(
  users.map(async (user) => ({
    ...user,
    posts: await prisma.post.findMany({ where: { authorId: user.id } }),
  }))
);
// 101 queries for 100 users!

// ✅ DO
const usersWithPosts = await prisma.user.findMany({
  include: { posts: true },
});
// 1 query with JOIN
```

### ❌ Not Using Singleton in Development

**Why it's bad**: Connection pool exhaustion during hot reload.

```typescript
// ❌ DON'T
const prisma = new PrismaClient(); // Creates new client on every hot reload

// ✅ DO — Use singleton pattern (see setup above)
import prisma from '@/lib/prisma';
```

### ❌ Editing Deployed Migrations

**Why it's bad**: Causes inconsistent database states.

```bash
# ❌ DON'T — Edit existing migration files after deployment

# ✅ DO — Create a new migration to fix issues
npx prisma migrate dev --name fix_user_column
```

### ❌ Using db push in Production

**Why it's bad**: No migration history, can cause data loss.

```bash
# ❌ DON'T
npx prisma db push  # In production

# ✅ DO
npx prisma migrate deploy  # In production
```

### ❌ Fetching All Fields When Not Needed

**Why it's bad**: Wastes bandwidth and memory.

```typescript
// ❌ DON'T
const users = await prisma.user.findMany(); // Fetches everything

// ✅ DO
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
  },
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 5.7 | 2024 | TypedSQL preview, `relationLoadStrategy` |
| 6.0 | Nov 2024 | Query Compiler (TypeScript), TypeScript 5.1+ required |
| 6.2 | Dec 2024 | **Omit GA** (exclude fields from results) |
| 6.6+ | 2025 | Query Compiler production-ready, edge improvements |
| 7.0 | 2025 | Rust-free, 3.4x faster, 90% smaller bundle |

### Prisma 6 Breaking Changes

**Buffer → Uint8Array:**
```typescript
// Before (v5) — Buffer for Bytes fields
const data: Buffer = record.binaryField;

// After (v6) — Uint8Array
const data: Uint8Array = record.binaryField;
```

**Omit Feature (GA in 6.2):**
```typescript
// Exclude sensitive fields from results
const user = await prisma.user.findUnique({
  where: { id: userId },
  omit: { password: true, apiKey: true },
});
// user has no password or apiKey fields
```

**M2M Relations (PostgreSQL):**
- Implicit M2M relation tables now use **primary key** instead of unique index
- Affects replica identity behavior

### Prisma 6.18+ Configuration

**prisma.config.ts (New):**
```typescript
// prisma.config.ts — New config file format
import { defineConfig } from '@prisma/config';

export default defineConfig({
  // Fields previously in schema.prisma now available here
  earlyAccess: ['driverAdapters'],
  // Future-proofing for Prisma 7
});
```

**Driver Adapters Required (v6+):**
```typescript
// Rust engine removed — must install driver adapter manually
// For PostgreSQL:
import { PrismaClient } from '@prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const adapter = new PrismaPg(pool);

const prisma = new PrismaClient({ adapter });
```

### Prisma 7 Architecture

- **No Rust binaries** — Pure TypeScript/WASM
- **Bundle size**: 14MB → 1.6MB
- **Query speed**: Up to 3.4x faster
- **Deployment**: No native dependencies to manage
- **Edge support**: Better serverless compatibility
- **Driver adapters**: Required for all databases

---

## Quick Reference

| Task | Solution |
|------|----------|
| Create record | `prisma.user.create({ data: {...} })` |
| Find many | `prisma.user.findMany({ where: {...} })` |
| Find unique | `prisma.user.findUnique({ where: { id } })` |
| Update | `prisma.user.update({ where: { id }, data: {...} })` |
| Delete | `prisma.user.delete({ where: { id } })` |
| Upsert | `prisma.user.upsert({ where, create, update })` |
| Count | `prisma.user.count({ where: {...} })` |
| Include relations | `include: { posts: true }` |
| Select fields | `select: { id: true, name: true }` |
| Order by | `orderBy: { createdAt: 'desc' }` |
| Pagination | `skip: 10, take: 20` |
| Transaction | `prisma.$transaction([...])` |
| Raw query | `prisma.$queryRaw` |
| TypedSQL | `prisma.$queryRawTyped(sqlFile(...))` |

| CLI Command | Purpose |
|-------------|---------|
| `prisma generate` | Generate Prisma Client |
| `prisma db push` | Push schema to DB (dev) |
| `prisma migrate dev` | Create migration (dev) |
| `prisma migrate deploy` | Apply migrations (prod) |
| `prisma studio` | Visual database editor |
| `prisma format` | Format schema file |

---

## Resources

- [Official Prisma Documentation](https://www.prisma.io/docs)
- [Prisma 6 Release Blog](https://www.prisma.io/blog/prisma-6-better-performance-more-flexibility-and-type-safe-sql)
- [Prisma 7 Announcement](https://www.prisma.io/blog/announcing-prisma-orm-7-0-0)
- [Architecture Shift Blog](https://www.prisma.io/blog/from-rust-to-typescript-a-new-chapter-for-prisma-orm)
- [Prisma Deep-Dive Handbook 2025](https://dev.to/mihir_bhadak/prisma-deep-dive-handbook-2025-from-zero-to-expert-1761)
- [N+1 Query Problem Guide](https://www.furkanbaytekin.dev/blogs/software/n1-query-problem-fixing-it-with-sql-and-prisma-orm)
