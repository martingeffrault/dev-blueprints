# Drizzle ORM (2025)

> **Last updated**: January 2026
> **Versions covered**: 0.36+, 1.0 beta
> **Purpose**: SQL-first TypeScript ORM with zero dependencies

---

## Philosophy (2025-2026)

Drizzle is the **SQL-first ORM** that feels like writing SQL but with full TypeScript safety.

**Key philosophical shifts:**
- **If you know SQL, you know Drizzle** — Minimal abstraction
- **Zero dependencies** — 7.4kb minified+gzipped
- **Zero code generation** — Types auto-sync with schema
- **Serverless-first** — Instant cold starts, no binary engines
- **1 query = 1 SQL statement** — Predictable performance
- **Schema as code** — TypeScript schema definitions

---

## TL;DR

- Define schema in TypeScript, mirrors SQL `CREATE TABLE`
- Use `$inferSelect` and `$inferInsert` for type extraction
- Always use `where` clauses with delete operations
- Use `drizzle-kit` for migrations
- No code generation needed — types infer automatically
- Perfect for serverless (Vercel, Cloudflare Workers, etc.)
- Use relational queries API for complex joins

---

## Best Practices

### Schema Definition

```typescript
// db/schema.ts
import {
  pgTable,
  text,
  timestamp,
  uuid,
  boolean,
  integer,
  index,
  uniqueIndex,
} from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  role: text('role', { enum: ['user', 'admin', 'moderator'] }).default('user'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => [
  uniqueIndex('email_idx').on(table.email),
]);

export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => [
  index('author_idx').on(table.authorId),
  index('published_idx').on(table.published),
]);

export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  content: text('content').notNull(),
  postId: uuid('post_id').notNull().references(() => posts.id),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Define relations (for relational query API)
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
  comments: many(comments),
}));

export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
  comments: many(comments),
}));

export const commentsRelations = relations(comments, ({ one }) => ({
  post: one(posts, {
    fields: [comments.postId],
    references: [posts.id],
  }),
  author: one(users, {
    fields: [comments.authorId],
    references: [users.id],
  }),
}));
```

### Type Inference

```typescript
// Type inference from schema
import { users, posts } from './schema';

// Select type (includes all columns, for query results)
type User = typeof users.$inferSelect;

// Insert type (excludes auto-generated fields)
type NewUser = typeof users.$inferInsert;

// Example usage
const newUser: NewUser = {
  email: 'alice@example.com',
  name: 'Alice',
  // id, createdAt, updatedAt auto-generated
};
```

### Database Client Setup

```typescript
// db/index.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export const db = drizzle(pool, { schema });

// For serverless (Neon, Vercel Postgres, etc.)
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

### Basic CRUD Operations

```typescript
import { db } from './db';
import { users, posts } from './db/schema';
import { eq, and, or, gt, desc, asc, like } from 'drizzle-orm';

// CREATE
const newUser = await db.insert(users).values({
  email: 'alice@example.com',
  name: 'Alice',
}).returning();

// INSERT multiple
await db.insert(users).values([
  { email: 'bob@example.com', name: 'Bob' },
  { email: 'charlie@example.com', name: 'Charlie' },
]);

// READ - Select all
const allUsers = await db.select().from(users);

// READ - Select specific columns
const userEmails = await db.select({
  id: users.id,
  email: users.email,
}).from(users);

// READ - With where clause
const admins = await db.select()
  .from(users)
  .where(eq(users.role, 'admin'));

// READ - Complex conditions
const results = await db.select()
  .from(users)
  .where(and(
    eq(users.role, 'user'),
    gt(users.createdAt, new Date('2024-01-01')),
    or(
      like(users.email, '%@gmail.com'),
      like(users.email, '%@company.com')
    )
  ))
  .orderBy(desc(users.createdAt))
  .limit(10)
  .offset(0);

// UPDATE
await db.update(users)
  .set({ name: 'Alice Updated' })
  .where(eq(users.id, userId));

// DELETE (always use where!)
await db.delete(users)
  .where(eq(users.id, userId));
```

### Relational Queries (v2 API)

```typescript
// Fetch user with all posts
const userWithPosts = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    posts: true,
  },
});

// Nested relations with filtering
const userWithPublishedPosts = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    posts: {
      where: eq(posts.published, true),
      orderBy: desc(posts.createdAt),
      limit: 5,
      with: {
        comments: {
          limit: 10,
          with: {
            author: true,
          },
        },
      },
    },
  },
});

// Find many with relations
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: {
      columns: {
        id: true,
        title: true,
      },
    },
  },
  limit: 20,
});
```

### Joins (SQL-style)

```typescript
import { eq } from 'drizzle-orm';

// Inner join
const postsWithAuthors = await db
  .select({
    postId: posts.id,
    postTitle: posts.title,
    authorName: users.name,
    authorEmail: users.email,
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id));

// Left join
const usersWithPosts = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId));

// Multiple joins
const commentsWithDetails = await db
  .select({
    comment: comments.content,
    postTitle: posts.title,
    authorName: users.name,
  })
  .from(comments)
  .innerJoin(posts, eq(comments.postId, posts.id))
  .innerJoin(users, eq(comments.authorId, users.id));
```

### Transactions

```typescript
// Transaction with automatic rollback on error
const result = await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({
    email: 'new@example.com',
    name: 'New User',
  }).returning();

  const [post] = await tx.insert(posts).values({
    title: 'First Post',
    authorId: user.id,
  }).returning();

  return { user, post };
});

// Savepoints
await db.transaction(async (tx) => {
  await tx.insert(users).values({ ... });

  // Nested savepoint
  await tx.transaction(async (tx2) => {
    await tx2.insert(posts).values({ ... });
    // Can rollback to this point
  });
});
```

### Migrations with Drizzle Kit

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

```bash
# Generate migration
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# Push schema directly (dev only)
npx drizzle-kit push

# Open Drizzle Studio
npx drizzle-kit studio

# Introspect existing database
npx drizzle-kit introspect
```

### Prepared Statements

```typescript
import { sql } from 'drizzle-orm';

// Prepared statement for better performance
const getUserByEmail = db
  .select()
  .from(users)
  .where(eq(users.email, sql.placeholder('email')))
  .prepare('get_user_by_email');

// Execute with parameters
const user = await getUserByEmail.execute({ email: 'alice@example.com' });
```

---

## Anti-Patterns

### ❌ Forgetting WHERE on DELETE

**Why it's bad**: Deletes all records in table.

```typescript
// ❌ DON'T — Catastrophic!
await db.delete(users);  // Deletes ALL users!

// ✅ DO — Always use where
await db.delete(users).where(eq(users.id, userId));
```

### ❌ Not Using Indexes

**Why it's bad**: Slow queries on large tables.

```typescript
// ❌ DON'T — No indexes
export const posts = pgTable('posts', {
  authorId: uuid('author_id').notNull(),
});

// ✅ DO — Add indexes for frequently queried columns
export const posts = pgTable('posts', {
  authorId: uuid('author_id').notNull(),
}, (table) => [
  index('author_idx').on(table.authorId),
]);
```

### ❌ Selecting All Columns When Not Needed

**Why it's bad**: Increased data transfer and memory usage.

```typescript
// ❌ DON'T
const users = await db.select().from(users);

// ✅ DO — Select only needed columns
const users = await db.select({
  id: users.id,
  name: users.name,
}).from(users);
```

### ❌ Not Defining Relations

**Why it's bad**: Can't use relational queries API.

```typescript
// ❌ DON'T — Missing relations
export const posts = pgTable('posts', { ... });
// No relations defined

// ✅ DO — Define relations for complex queries
export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}));
```

### ❌ Modifying Migration History

**Why it's bad**: Causes inconsistent database states.

```bash
# ❌ DON'T — Edit or delete migration files after applying

# ✅ DO — Create new migrations to fix issues
npx drizzle-kit generate
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 0.30 | 2024 | Relational queries v2 preview |
| 0.33 | 2024 | Improved PostgreSQL support |
| 0.44 | Late 2024 | Last stable v0 release |
| **1.0-beta.1** | Jan 2025 | RQBv2 stable, schema in relations |
| **1.0-beta.2** | Feb 2025 | **MSSQL support**, native Bun/Deno, tsx loader |

### Drizzle 1.0 Breaking Changes

**RQBv2 Syntax Change:**
```typescript
// v1 (deprecated → moved to db._query)
import { relations } from 'drizzle-orm';
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

// v2 (new in 1.0) — schema IN the relations definition
import { defineRelations } from 'drizzle-orm';
export const relations = defineRelations(schema, (r) => ({
  users: r.users({
    posts: r.many.posts(),
  }),
}));
```

**drizzle-kit Changes:**
- Migrated from `esbuild-register` to `tsx` loader
- Native Bun and Deno support
- `drizzle-kit pull` now generates `relations.ts` in new syntax

### Relational Queries v2 Features

```typescript
// v1 (deprecated)
const result = await db.query.users.findMany({
  with: { posts: true },
});

// v2 (current)
const result = await db.query.users.findMany({
  with: {
    posts: {
      where: eq(posts.published, true),
      orderBy: desc(posts.createdAt),
    },
  },
});
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Insert | `db.insert(table).values({ ... })` |
| Select all | `db.select().from(table)` |
| Select columns | `db.select({ id: table.id }).from(table)` |
| Where | `db.select().from(table).where(eq(table.id, id))` |
| Update | `db.update(table).set({ ... }).where(eq(...))` |
| Delete | `db.delete(table).where(eq(...))` |
| Order by | `.orderBy(desc(table.createdAt))` |
| Limit | `.limit(10).offset(0)` |
| Join | `.innerJoin(table2, eq(table.id, table2.fk))` |
| Transaction | `db.transaction(async (tx) => { ... })` |
| Relational query | `db.query.table.findMany({ with: {...} })` |

| Operator | Usage |
|----------|-------|
| Equals | `eq(column, value)` |
| Not equals | `ne(column, value)` |
| Greater than | `gt(column, value)` |
| Less than | `lt(column, value)` |
| Greater/equal | `gte(column, value)` |
| Less/equal | `lte(column, value)` |
| Like | `like(column, '%pattern%')` |
| In | `inArray(column, [1, 2, 3])` |
| Is null | `isNull(column)` |
| And | `and(cond1, cond2)` |
| Or | `or(cond1, cond2)` |

| CLI Command | Purpose |
|-------------|---------|
| `drizzle-kit generate` | Generate migration |
| `drizzle-kit migrate` | Apply migrations |
| `drizzle-kit push` | Push schema (dev) |
| `drizzle-kit studio` | Visual DB editor |
| `drizzle-kit introspect` | Generate schema from DB |

---

## Drizzle vs Prisma

| Aspect | Drizzle | Prisma |
|--------|---------|--------|
| Bundle size | ~7.4kb | ~1.6MB (v7) |
| Cold start | Instant | Faster in v7 |
| Query style | SQL-like | Object-oriented |
| Code generation | None | Required |
| Learning curve | SQL knowledge required | Lower (abstracted) |
| Serverless | Excellent | Good (v7 improved) |
| Type safety | Excellent | Excellent |

---

## Resources

- [Official Drizzle Documentation](https://orm.drizzle.team/)
- [Drizzle ORM GitHub](https://github.com/drizzle-team/drizzle-orm)
- [Ultimate Guide 2025](https://dev.to/sameer_saleem/the-ultimate-guide-to-drizzle-orm-postgresql-2025-edition-22b)
- [3 Biggest Mistakes with Drizzle](https://medium.com/@lior_amsalem/3-biggest-mistakes-with-drizzle-orm-1327e2531aff)
- [Drizzle vs Prisma Comparison](https://www.bytebase.com/blog/drizzle-vs-prisma/)
- [Better Stack Drizzle Guide](https://betterstack.com/community/guides/scaling-nodejs/drizzle-orm/)
