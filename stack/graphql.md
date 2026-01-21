# GraphQL (2025)

> **Last updated**: January 2026
> **Versions covered**: GraphQL September 2025 spec, Apollo Client 4.x, Apollo Server 5.x, Yoga 5.x
> **Purpose**: Query language for APIs with type-safe data fetching

---

## Philosophy (2025-2026)

GraphQL is the **declarative query language** that lets clients request exactly the data they need, with strong typing and introspection.

**Key philosophical shifts:**
- **Ask for what you need** — No over-fetching or under-fetching
- **Single endpoint** — Replace REST's multiple endpoints
- **Type system first** — Schema as contract
- **DataLoader pattern** — Solve N+1 queries
- **Persisted queries** — Performance and security
- **Apollo MCP** — GraphQL for AI integrations (2025)

---

## TL;DR

- Name all operations (queries, mutations, fragments)
- Use fragments for component data requirements
- Implement DataLoader for N+1 prevention
- Separate user-specific and general data queries
- Use persisted queries for production
- Pagination with cursor-based connections
- Validate input with custom scalars or directives
- Monitor with Apollo Studio or OpenTelemetry

---

## Best Practices

### Project Structure

```
my-graphql-app/
├── src/
│   ├── schema/
│   │   ├── index.ts           # Schema composition
│   │   ├── typeDefs/
│   │   │   ├── user.graphql
│   │   │   ├── post.graphql
│   │   │   └── common.graphql
│   │   └── resolvers/
│   │       ├── user.ts
│   │       ├── post.ts
│   │       └── scalars.ts
│   ├── dataloaders/
│   │   ├── index.ts
│   │   ├── userLoader.ts
│   │   └── postLoader.ts
│   ├── services/
│   │   ├── user.service.ts
│   │   └── post.service.ts
│   ├── context.ts
│   └── index.ts
├── codegen.ts
└── package.json
```

### Schema Definition

```graphql
# src/schema/typeDefs/common.graphql
scalar DateTime
scalar EmailAddress

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

interface Node {
  id: ID!
}

input PaginationInput {
  first: Int
  after: String
  last: Int
  before: String
}
```

```graphql
# src/schema/typeDefs/user.graphql
type User implements Node {
  id: ID!
  email: EmailAddress!
  name: String!
  avatar: String
  role: UserRole!
  posts(pagination: PaginationInput): PostConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum UserRole {
  USER
  ADMIN
}

type UserEdge {
  node: User!
  cursor: String!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

extend type Query {
  user(id: ID!): User
  me: User
  users(pagination: PaginationInput): UserConnection!
}

extend type Mutation {
  updateProfile(input: UpdateProfileInput!): User!
}

input UpdateProfileInput {
  name: String
  avatar: String
}
```

```graphql
# src/schema/typeDefs/post.graphql
type Post implements Node {
  id: ID!
  title: String!
  slug: String!
  content: String!
  excerpt: String!
  author: User!
  tags: [String!]!
  status: PostStatus!
  commentsCount: Int!
  likesCount: Int!
  publishedAt: DateTime
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

extend type Query {
  post(id: ID): Post
  postBySlug(slug: String!): Post
  posts(
    pagination: PaginationInput
    status: PostStatus
    authorId: ID
    tags: [String!]
  ): PostConnection!
  searchPosts(query: String!, limit: Int): [Post!]!
}

extend type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  publishPost(id: ID!): Post!
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
}

input UpdatePostInput {
  title: String
  content: String
  tags: [String!]
}
```

### Apollo Server Setup

```typescript
// src/index.ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';
import cors from 'cors';
import { typeDefs } from './schema/typeDefs';
import { resolvers } from './schema/resolvers';
import { createContext, Context } from './context';

const app = express();

const server = new ApolloServer<Context>({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production',
  plugins: [
    // Logging plugin
    {
      async requestDidStart() {
        return {
          async didEncounterErrors({ errors }) {
            errors.forEach((error) => {
              console.error('GraphQL Error:', error);
            });
          },
        };
      },
    },
  ],
});

await server.start();

app.use(
  '/graphql',
  cors(),
  express.json(),
  expressMiddleware(server, {
    context: createContext,
  })
);

app.listen(4000, () => {
  console.log('Server running at http://localhost:4000/graphql');
});
```

### Context with DataLoaders

```typescript
// src/context.ts
import { Request } from 'express';
import { createLoaders, Loaders } from './dataloaders';
import { UserService } from './services/user.service';
import { PostService } from './services/post.service';
import { verifyToken } from './auth';

export interface Context {
  user: { id: string; role: string } | null;
  loaders: Loaders;
  services: {
    user: UserService;
    post: PostService;
  };
}

export async function createContext({ req }: { req: Request }): Promise<Context> {
  // Auth
  const token = req.headers.authorization?.replace('Bearer ', '');
  let user = null;

  if (token) {
    try {
      user = await verifyToken(token);
    } catch {
      // Invalid token, continue as unauthenticated
    }
  }

  // Create new loaders per request (important!)
  const loaders = createLoaders();

  return {
    user,
    loaders,
    services: {
      user: new UserService(),
      post: new PostService(),
    },
  };
}
```

### DataLoaders (N+1 Prevention)

```typescript
// src/dataloaders/index.ts
import DataLoader from 'dataloader';
import { User, Post } from '../models';

export interface Loaders {
  userLoader: DataLoader<string, User | null>;
  postsByAuthorLoader: DataLoader<string, Post[]>;
}

export function createLoaders(): Loaders {
  return {
    // Batch user fetching
    userLoader: new DataLoader(async (ids: readonly string[]) => {
      const users = await User.find({ _id: { $in: ids } });
      const userMap = new Map(users.map((u) => [u._id.toString(), u]));
      return ids.map((id) => userMap.get(id) ?? null);
    }),

    // Batch posts by author
    postsByAuthorLoader: new DataLoader(async (authorIds: readonly string[]) => {
      const posts = await Post.find({ 'author._id': { $in: authorIds } });
      const postsByAuthor = new Map<string, Post[]>();

      authorIds.forEach((id) => postsByAuthor.set(id, []));
      posts.forEach((post) => {
        const authorId = post.author._id.toString();
        postsByAuthor.get(authorId)?.push(post);
      });

      return authorIds.map((id) => postsByAuthor.get(id) ?? []);
    }),
  };
}
```

### Resolvers

```typescript
// src/schema/resolvers/post.ts
import { Context } from '../../context';
import { Post } from '../../models';

export const postResolvers = {
  Query: {
    post: async (_: any, { id }: { id: string }, ctx: Context) => {
      return ctx.services.post.findById(id);
    },

    postBySlug: async (_: any, { slug }: { slug: string }, ctx: Context) => {
      return ctx.services.post.findBySlug(slug);
    },

    posts: async (
      _: any,
      { pagination, status, authorId, tags }: any,
      ctx: Context
    ) => {
      const { first = 10, after } = pagination ?? {};
      const result = await ctx.services.post.findMany({
        limit: first + 1,
        cursor: after,
        status,
        authorId,
        tags,
      });

      const hasNextPage = result.length > first;
      const edges = result.slice(0, first).map((post) => ({
        node: post,
        cursor: post._id.toString(),
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: await ctx.services.post.count({ status, authorId, tags }),
      };
    },

    searchPosts: async (
      _: any,
      { query, limit = 10 }: { query: string; limit?: number },
      ctx: Context
    ) => {
      return ctx.services.post.search(query, limit);
    },
  },

  Mutation: {
    createPost: async (_: any, { input }: any, ctx: Context) => {
      if (!ctx.user) throw new Error('Unauthorized');
      return ctx.services.post.create(ctx.user.id, input);
    },

    updatePost: async (_: any, { id, input }: any, ctx: Context) => {
      if (!ctx.user) throw new Error('Unauthorized');
      return ctx.services.post.update(id, ctx.user.id, input);
    },

    deletePost: async (_: any, { id }: { id: string }, ctx: Context) => {
      if (!ctx.user) throw new Error('Unauthorized');
      return ctx.services.post.delete(id, ctx.user.id);
    },

    publishPost: async (_: any, { id }: { id: string }, ctx: Context) => {
      if (!ctx.user) throw new Error('Unauthorized');
      return ctx.services.post.publish(id, ctx.user.id);
    },
  },

  Post: {
    // Use DataLoader to batch author fetching
    author: async (post: Post, _: any, ctx: Context) => {
      return ctx.loaders.userLoader.load(post.author._id.toString());
    },

    excerpt: (post: Post) => {
      return post.content.substring(0, 200) + '...';
    },
  },
};
```

### Client Queries (Apollo Client)

```typescript
// src/graphql/queries.ts
import { gql } from '@apollo/client';

// Always name operations
export const GET_POSTS = gql`
  query GetPosts($first: Int, $after: String, $status: PostStatus) {
    posts(pagination: { first: $first, after: $after }, status: $status) {
      edges {
        node {
          ...PostCard
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
      totalCount
    }
  }

  ${POST_CARD_FRAGMENT}
`;

// Fragment for component data needs
export const POST_CARD_FRAGMENT = gql`
  fragment PostCard on Post {
    id
    title
    slug
    excerpt
    author {
      id
      name
      avatar
    }
    tags
    publishedAt
    commentsCount
    likesCount
  }
`;

export const GET_POST = gql`
  query GetPost($slug: String!) {
    postBySlug(slug: $slug) {
      id
      title
      content
      author {
        id
        name
        avatar
      }
      tags
      publishedAt
      commentsCount
    }
  }
`;

export const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      slug
    }
  }
`;
```

### React Component with Apollo

```tsx
// src/components/PostList.tsx
import { useQuery } from '@apollo/client';
import { GET_POSTS, PostCardFragment } from '../graphql/queries';

export function PostList() {
  const { data, loading, error, fetchMore } = useQuery(GET_POSTS, {
    variables: { first: 10, status: 'PUBLISHED' },
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  const { edges, pageInfo, totalCount } = data.posts;

  return (
    <div>
      <p>Total: {totalCount} posts</p>

      {edges.map(({ node }: { node: PostCardFragment }) => (
        <PostCard key={node.id} post={node} />
      ))}

      {pageInfo.hasNextPage && (
        <button
          onClick={() =>
            fetchMore({
              variables: { after: pageInfo.endCursor },
            })
          }
        >
          Load More
        </button>
      )}
    </div>
  );
}
```

### Code Generation

```typescript
// codegen.ts
import { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: 'http://localhost:4000/graphql',
  documents: ['src/**/*.{ts,tsx}'],
  generates: {
    './src/__generated__/types.ts': {
      plugins: ['typescript'],
    },
    './src/__generated__/operations.ts': {
      preset: 'client',
      plugins: ['typescript-operations', 'typescript-react-apollo'],
      config: {
        withHooks: true,
      },
    },
  },
};

export default config;
```

---

## Anti-Patterns

### ❌ Anonymous Operations

**Why it's bad**: Hard to debug, no caching benefits.

```graphql
# ❌ DON'T
query {
  posts { id title }
}

# ✅ DO — Name all operations
query GetPosts {
  posts { id title }
}
```

### ❌ N+1 Queries

**Why it's bad**: Database query per item.

```typescript
// ❌ DON'T — Query per post
Post: {
  author: async (post) => {
    return User.findById(post.authorId);  // N+1!
  }
}

// ✅ DO — Use DataLoader
Post: {
  author: async (post, _, ctx) => {
    return ctx.loaders.userLoader.load(post.authorId);
  }
}
```

### ❌ Over-fetching in Queries

**Why it's bad**: Wastes bandwidth, defeats GraphQL purpose.

```graphql
# ❌ DON'T — Fetch everything
query {
  user(id: "1") {
    id name email avatar role
    posts { id title content author { ... } }
    followers { ... }
  }
}

# ✅ DO — Fetch only what's needed
query GetUserProfile($id: ID!) {
  user(id: $id) {
    id
    name
    avatar
  }
}
```

### ❌ No Input Validation

**Why it's bad**: Allows invalid data, security issues.

```graphql
# ❌ DON'T — Accept any string
input CreatePostInput {
  title: String!
  content: String!
}

# ✅ DO — Use custom scalars or validate in resolver
scalar NonEmptyString

input CreatePostInput {
  title: NonEmptyString!  # Or validate in resolver
  content: NonEmptyString!
}
```

---

## 2025-2026 Changelog

| Feature | Date | Description |
|---------|------|-------------|
| **GraphQL September 2025 Spec** | Sep 2025 | @oneOf directive official, Schema Coordinates |
| **Apollo Client 4.0** | Sep 2025 | RxJS, 20-30% smaller bundles, opt-in features |
| **Hive Gateway v2** | Oct 2025 | Open-source Federation router |
| @defer/@stream | Stage 2 | Incremental delivery (experimental) |
| Apollo MCP | 2025 | GraphQL for AI agents |
| Federation 2.9 | 2025 | Apollo Router requires v2 only |

### GraphQL September 2025 Specification

First spec update since October 2021! Key additions:

**@oneOf Directive (Now Official):**
```graphql
# ✅ NEW — Mutually exclusive inputs with @oneOf
input PetInput @oneOf {
  cat: CatInput
  dog: DogInput
  fish: FishInput
}

type Mutation {
  createPet(input: PetInput!): Pet!
}
```
```graphql
# Usage — Exactly ONE field must be provided
mutation {
  createPet(input: { cat: { name: "Whiskers" } }) {
    id
  }
}
```

**Schema Coordinates:**
```graphql
# NEW — Stable identifiers for tooling
# Query.user = the user field on Query type
# User.name = the name field on User type

# Used for: codegen, linting, PR comments, schema registries
```

**@defer/@stream (Stage 2 — Experimental):**
```graphql
# EXPERIMENTAL — Incremental delivery
query GetPost($id: ID!) {
  post(id: $id) {
    title
    content
    # Defer non-critical data
    comments @defer {
      id
      text
      author { name }
    }
  }
}
```
```typescript
// Yoga/Apollo — Enable experimental support
import { createYoga } from 'graphql-yoga';

const yoga = createYoga({
  schema,
  // Yoga supports @defer/@stream natively
});
```

### Apollo Client 4.0 Breaking Changes (September 2025)

**Import Path Changes (Required):**
```typescript
// ❌ OLD — React hooks from main entry
import { useQuery, useMutation } from '@apollo/client';

// ✅ NEW — React hooks from /react
import { useQuery, useMutation } from '@apollo/client/react';

// Core client imports remain the same
import { ApolloClient, InMemoryCache } from '@apollo/client';
```

**Link Creator Functions → Classes:**
```typescript
// ❌ OLD — Function creators
import { createHttpLink, createErrorLink } from '@apollo/client';
const httpLink = createHttpLink({ uri: '/graphql' });

// ✅ NEW — Classes
import { HttpLink } from '@apollo/client';
const httpLink = new HttpLink({ uri: '/graphql' });
```

**RxJS Now Required:**
```bash
# RxJS is now a peer dependency
npm install rxjs
```
```typescript
// Access RxJS operators for advanced use cases
import { from } from 'rxjs';
import { retry, catchError } from 'rxjs/operators';

// Apollo Client observables are now RxJS Observables
const observable = client.watchQuery({ query: GET_POSTS });
```

**ServerError Changes:**
```typescript
// ❌ OLD — result property (parsed JSON)
catch (error) {
  if (error instanceof ServerError) {
    console.log(error.result); // Parsed JSON
  }
}

// ✅ NEW — bodyText property (raw string)
catch (error) {
  if (error instanceof ServerError) {
    console.log(error.bodyText); // Raw string body
    const parsed = JSON.parse(error.bodyText); // Parse manually if needed
  }
}
```

**Opt-in Features (Smaller Bundles):**
```typescript
// ❌ OLD — Everything bundled by default
import { ApolloClient, InMemoryCache } from '@apollo/client';

// ✅ NEW — Explicitly import features you need
import { ApolloClient } from '@apollo/client/core';
import { InMemoryCache } from '@apollo/client/cache';
import { HttpLink } from '@apollo/client/link/http';
import { LocalState } from '@apollo/client/local-state'; // Only if using @client

// HTTP Link is NOT bundled by default anymore!
const client = new ApolloClient({
  cache: new InMemoryCache(),
  link: new HttpLink({ uri: '/graphql' }), // Must import explicitly
});
```

**Namespace Types:**
```typescript
// ✅ NEW — Types via namespaces
import { useQuery } from '@apollo/client/react';

// Options and Result types are attached to the hook
const options: useQuery.Options<GetPostsQuery, GetPostsQueryVariables> = {
  variables: { first: 10 },
};

function MyComponent() {
  const result: useQuery.Result<GetPostsQuery> = useQuery(GET_POSTS, options);
}
```

**Migration Codemod:**
```bash
# Automate 90% of migration
npx @apollo/client-codemod-migrate-3-to-4 src
```

### Hive Gateway v2 (Open-Source Apollo Router Alternative)

```typescript
// Hive Gateway — Open-source Federation router
import { createGateway } from '@graphql-hive/gateway';

const gateway = createGateway({
  supergraph: './supergraph.graphql',
  // Or fetch from Hive
  supergraph: {
    type: 'hive',
    endpoint: process.env.HIVE_CDN_ENDPOINT,
    key: process.env.HIVE_CDN_KEY,
  },
});

// Deploy anywhere: Node.js, Bun, Deno, Workers, etc.
```

### Apollo Federation v2 Only (Router v1.60+)

```yaml
# Apollo Router v1.60+ no longer supports Federation v1
# Migrate subgraphs to Federation v2

# ❌ OLD — Federation v1
extend type Query {
  product(id: ID!): Product
}

# ✅ NEW — Federation v2
extend schema @link(url: "https://specs.apollo.dev/federation/v2.0")

type Query {
  product(id: ID!): Product @shareable
}
```

---

## Quick Reference

| Concept | Purpose |
|---------|---------|
| Query | Read data |
| Mutation | Write data |
| Subscription | Real-time updates |
| Fragment | Reusable field selections |
| Directive | Modify execution |
| Scalar | Custom types |

| Apollo Client Hook | Usage |
|--------------------|-------|
| `useQuery` | Fetch data |
| `useLazyQuery` | Fetch on demand |
| `useMutation` | Modify data |
| `useSubscription` | Real-time data |
| `useFragment` | Read fragment from cache |

---

## Resources

- [GraphQL Official Documentation](https://graphql.org/)
- [Apollo Server Documentation](https://www.apollographql.com/docs/apollo-server)
- [Apollo Client Documentation](https://www.apollographql.com/docs/react)
- [Query Best Practices](https://www.apollographql.com/docs/react/data/operation-best-practices)
- [DataLoader](https://github.com/graphql/dataloader)
- [GraphQL Code Generator](https://the-guild.dev/graphql/codegen)
