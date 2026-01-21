# Supabase (2025)

> **Last updated**: January 2026
> **Versions covered**: Supabase 2.x
> **Purpose**: Open source Firebase alternative with Postgres

---

## Philosophy (2025-2026)

Supabase is the **open source Firebase alternative** built on PostgreSQL, providing database, auth, storage, realtime, and edge functions in one platform.

**Key philosophical shifts:**
- **Postgres at the core** — Full SQL power, not NoSQL
- **Row-Level Security (RLS)** — Security in the database layer
- **Realtime by default** — Listen to database changes
- **Edge Functions** — Serverless Deno functions
- **Auth built-in** — JWT-based authentication
- **Open source** — Self-hostable, no vendor lock-in

---

## TL;DR

- **Always enable RLS** on user-facing tables
- **Never use service_role key** in client code
- Use `supabase.auth.getUser()` on client, not `getSession()`
- Edge Functions for lightweight APIs, not heavy background jobs
- Use CLI for migrations, not dashboard for production
- Enable Point-in-Time Recovery for databases > 4GB
- Real-time only on tables that need it
- Keep secrets in environment variables, never in code

---

## Best Practices

### Project Structure

```
my-supabase-app/
├── supabase/
│   ├── config.toml           # Supabase config
│   ├── migrations/           # Database migrations
│   │   ├── 20240101000000_init.sql
│   │   └── 20240102000000_add_rls.sql
│   ├── seed.sql              # Seed data
│   └── functions/            # Edge Functions
│       ├── hello/
│       │   └── index.ts
│       └── send-email/
│           └── index.ts
├── src/
│   ├── lib/
│   │   └── supabase.ts       # Client initialization
│   └── ...
├── .env.local                # Local env vars
└── package.json
```

### Client Setup

```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';
import type { Database } from './database.types';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey);
```

```typescript
// src/lib/database.types.ts (generated)
// Run: supabase gen types typescript --project-id your-project > database.types.ts

export interface Database {
  public: {
    Tables: {
      users: {
        Row: {
          id: string;
          email: string;
          name: string | null;
          created_at: string;
        };
        Insert: {
          id?: string;
          email: string;
          name?: string | null;
          created_at?: string;
        };
        Update: {
          id?: string;
          email?: string;
          name?: string | null;
          created_at?: string;
        };
      };
      posts: {
        Row: {
          id: string;
          title: string;
          content: string;
          author_id: string;
          published: boolean;
          created_at: string;
        };
        Insert: {
          id?: string;
          title: string;
          content: string;
          author_id: string;
          published?: boolean;
          created_at?: string;
        };
        Update: {
          title?: string;
          content?: string;
          published?: boolean;
        };
      };
    };
  };
}
```

### Authentication

```typescript
// Sign up
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'securepassword123',
  options: {
    data: {
      name: 'John Doe',
    },
  },
});

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'securepassword123',
});

// OAuth sign in
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: 'https://myapp.com/auth/callback',
  },
});

// Get current user (RECOMMENDED)
const { data: { user }, error } = await supabase.auth.getUser();

// Sign out
await supabase.auth.signOut();

// Listen to auth changes
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_IN') {
    console.log('User signed in:', session?.user);
  } else if (event === 'SIGNED_OUT') {
    console.log('User signed out');
  }
});
```

### Row-Level Security (RLS)

```sql
-- migrations/20240101000000_create_tables.sql

-- Create tables
CREATE TABLE public.posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  author_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  published BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Enable RLS
ALTER TABLE public.posts ENABLE ROW LEVEL SECURITY;

-- Policies
-- Anyone can read published posts
CREATE POLICY "Public posts are viewable by everyone"
ON public.posts FOR SELECT
USING (published = true);

-- Authors can read their own posts (including drafts)
CREATE POLICY "Users can view own posts"
ON public.posts FOR SELECT
USING (auth.uid() = author_id);

-- Authors can insert their own posts
CREATE POLICY "Users can create own posts"
ON public.posts FOR INSERT
WITH CHECK (auth.uid() = author_id);

-- Authors can update their own posts
CREATE POLICY "Users can update own posts"
ON public.posts FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);

-- Authors can delete their own posts
CREATE POLICY "Users can delete own posts"
ON public.posts FOR DELETE
USING (auth.uid() = author_id);
```

### Database Queries

```typescript
// Select all published posts
const { data: posts, error } = await supabase
  .from('posts')
  .select('*')
  .eq('published', true)
  .order('created_at', { ascending: false });

// Select with relations
const { data: posts, error } = await supabase
  .from('posts')
  .select(`
    *,
    author:users(id, name, avatar_url),
    comments(id, content, created_at)
  `)
  .eq('published', true);

// Insert
const { data, error } = await supabase
  .from('posts')
  .insert({
    title: 'My Post',
    content: 'Post content...',
    author_id: user.id,
  })
  .select()
  .single();

// Update
const { data, error } = await supabase
  .from('posts')
  .update({ published: true })
  .eq('id', postId)
  .select()
  .single();

// Delete
const { error } = await supabase
  .from('posts')
  .delete()
  .eq('id', postId);

// Pagination
const { data, error, count } = await supabase
  .from('posts')
  .select('*', { count: 'exact' })
  .range(0, 9);  // First 10 items

// Full-text search
const { data, error } = await supabase
  .from('posts')
  .select('*')
  .textSearch('title', 'search query', {
    type: 'websearch',
    config: 'english',
  });
```

### Realtime Subscriptions

```typescript
// Subscribe to all changes on a table
const channel = supabase
  .channel('posts-changes')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'posts',
    },
    (payload) => {
      console.log('Change received:', payload);
    }
  )
  .subscribe();

// Subscribe to specific events
const channel = supabase
  .channel('new-posts')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'posts',
      filter: 'published=eq.true',
    },
    (payload) => {
      console.log('New post:', payload.new);
    }
  )
  .subscribe();

// Unsubscribe
supabase.removeChannel(channel);
```

### Storage

```typescript
// Upload file
const { data, error } = await supabase.storage
  .from('avatars')
  .upload(`${userId}/avatar.png`, file, {
    cacheControl: '3600',
    upsert: true,
  });

// Get public URL
const { data } = supabase.storage
  .from('avatars')
  .getPublicUrl(`${userId}/avatar.png`);

// Download file
const { data, error } = await supabase.storage
  .from('avatars')
  .download(`${userId}/avatar.png`);

// Delete file
const { error } = await supabase.storage
  .from('avatars')
  .remove([`${userId}/avatar.png`]);

// Create signed URL (private buckets)
const { data, error } = await supabase.storage
  .from('private-files')
  .createSignedUrl('path/to/file.pdf', 3600);  // 1 hour
```

### Edge Functions

```typescript
// supabase/functions/send-email/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  // Handle CORS preflight
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    // Get auth header
    const authHeader = req.headers.get('Authorization');
    if (!authHeader) {
      throw new Error('No authorization header');
    }

    // Create Supabase client with user's token
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_ANON_KEY')!,
      { global: { headers: { Authorization: authHeader } } }
    );

    // Verify user
    const { data: { user }, error: authError } = await supabase.auth.getUser();
    if (authError || !user) {
      throw new Error('Unauthorized');
    }

    const { to, subject, body } = await req.json();

    // Send email (using your email provider)
    // await sendEmail(to, subject, body);

    return new Response(
      JSON.stringify({ success: true }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

### Server-Side (Node.js)

```typescript
// Use service role key only on server
import { createClient } from '@supabase/supabase-js';

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // Never expose this!
);

// Admin operations (bypasses RLS)
const { data, error } = await supabaseAdmin
  .from('users')
  .select('*');
```

### CLI Workflow

```bash
# Start local development
supabase start

# Create migration
supabase migration new add_posts_table

# Apply migrations
supabase db push

# Generate types
supabase gen types typescript --local > src/lib/database.types.ts

# Deploy Edge Functions
supabase functions deploy send-email

# Link to remote project
supabase link --project-ref your-project-ref

# Push to production
supabase db push --linked
```

---

## Anti-Patterns

### ❌ Using service_role Key in Client Code

**Why it's bad**: Bypasses RLS, grants full database access.

```typescript
// ❌ DON'T — In browser code
const supabase = createClient(url, 'service_role_key_here');

// ✅ DO — Use anon key in client
const supabase = createClient(url, 'anon_key_here');

// ✅ DO — Use service role only on server
// (Next.js API route, Edge Function, etc.)
```

### ❌ Using getSession() for Auth Checks

**Why it's bad**: Session can be stale or tampered.

```typescript
// ❌ DON'T
const { data: { session } } = await supabase.auth.getSession();
const user = session?.user;  // May be stale

// ✅ DO
const { data: { user } } = await supabase.auth.getUser();
// Validates with server
```

### ❌ Disabling RLS on User-Facing Tables

**Why it's bad**: Anyone can read/write all data.

```sql
-- ❌ DON'T
ALTER TABLE posts DISABLE ROW LEVEL SECURITY;

-- ✅ DO — Always enable RLS
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
-- Then create appropriate policies
```

### ❌ Real-time on Everything

**Why it's bad**: Unnecessary overhead and cost.

```typescript
// ❌ DON'T — Subscribe to every table
supabase.channel('all').on('*', callback);

// ✅ DO — Only subscribe to what you need
supabase.channel('chat').on('INSERT', 'messages', callback);
```

### ❌ Making Migrations in Dashboard for Production

**Why it's bad**: No version control, not reproducible.

```bash
# ❌ DON'T — Use dashboard SQL editor for schema changes

# ✅ DO — Use CLI migrations
supabase migration new add_users_table
# Edit migration file
supabase db push
```

---

## 2025-2026 Changelog

| Feature | Date | Description |
|---------|------|-------------|
| **New API Keys** | Jun 2025 | `sb_publishable_*` replaces anon key, `sb_secret_*` replaces service_role |
| **OAuth 2.1 Server** | Nov 2025 | Public beta for OAuth 2.1 authorization server |
| **Vector Buckets** | Oct 2025 | Cold storage for embeddings (S3 Vectors, alpha) |
| **Analytics Buckets** | Oct 2025 | Iceberg/S3 Tables for analytical workloads (alpha) |
| **Realtime Settings** | 2025 | Dashboard config, channel restrictions |
| Branching | 2024+ | Database branches for dev |

### New API Keys (June 2025)

```typescript
// ❌ OLD — anon key and service_role key
const supabase = createClient(url, 'eyJ...');  // anon key

// ✅ NEW — Publishable and secret keys
// sb_publishable_* — Client-safe (replaces anon key)
// sb_secret_* — Server-only (replaces service_role key)

// Client-side
const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_PUBLISHABLE_KEY  // sb_publishable_...
);

// Server-side (create multiple secret keys with different scopes)
const supabaseAdmin = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SECRET_KEY  // sb_secret_...
);
```

**JWT Signing Keys:**
```typescript
// ❌ OLD — Symmetric secret for JWT signing
// ✅ NEW — Dedicated JWT Signing Keys (more secure)
// Configure in Dashboard → Auth → JWT
```

### OAuth 2.1 Server (November 2025)

```typescript
// NEW — Supabase as OAuth 2.1 authorization server
// Your app can issue access tokens to third-party apps

// Configure in Dashboard → Auth → OAuth 2.1
// Supports:
// - Authorization Code + PKCE
// - Client credentials
// - Token introspection
// - Token revocation
```

### Security Notification Templates

```typescript
// NEW — Expanded security notifications for:
// - Password changes
// - Email changes
// - Phone changes
// - Identity linking (OAuth)
// - MFA status changes

// Configure templates in Dashboard → Auth → Email Templates
```

### Vector Buckets (Alpha)

```typescript
// NEW — Cold storage for embeddings with query engine
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(url, key);

// Store embeddings in Vector Bucket
const { data, error } = await supabase.storage
  .from('embeddings')  // Vector bucket
  .upload('doc1', embeddingData);

// Query with similarity search
const { data: results } = await supabase.rpc('search_embeddings', {
  query_embedding: queryVector,
  match_count: 10,
});
```

### Realtime Settings

```typescript
// NEW — Configure Realtime in Dashboard
// Settings available:
// - Max connections per client
// - Message rate limits
// - Channel restrictions (public/private)

// Channel restrictions:
// - Public: Anyone can connect
// - Private: Requires authentication
// - Presence: Real-time user tracking
```

### Edge Functions Updates

```typescript
// NEW — Deploy Node.js apps as Edge Functions
// supabase/functions/my-node-app/index.ts
import express from 'npm:express@4';

const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Node.js on Edge!' });
});

export default app;
```

```bash
# NEW — Download Edge Functions without Docker
supabase functions download my-function

# Bulk edit secrets
supabase secrets set KEY1=value1 KEY2=value2
```

### Integrations Section

```sql
-- NEW — Cron Jobs (pg_cron extension)
-- Configure in Dashboard → Integrations → Cron
SELECT cron.schedule(
  'cleanup-old-sessions',
  '0 0 * * *',  -- Daily at midnight
  $$DELETE FROM sessions WHERE created_at < NOW() - INTERVAL '30 days'$$
);

-- NEW — Queues (pg_mq extension)
-- Configure in Dashboard → Integrations → Queues
SELECT mq.send('email-queue', '{"to": "user@example.com", "subject": "Welcome"}');
```

---

## Quick Reference

| Operation | Code |
|-----------|------|
| Select | `supabase.from('table').select('*')` |
| Insert | `supabase.from('table').insert({...})` |
| Update | `supabase.from('table').update({...}).eq('id', id)` |
| Delete | `supabase.from('table').delete().eq('id', id)` |
| Filter | `.eq()`, `.gt()`, `.lt()`, `.like()`, `.in()` |
| Order | `.order('column', { ascending: false })` |
| Limit | `.limit(10)` |
| Paginate | `.range(0, 9)` |

| Auth Method | Usage |
|-------------|-------|
| Email/Password | `signUp()`, `signInWithPassword()` |
| OAuth | `signInWithOAuth({ provider: 'google' })` |
| Magic Link | `signInWithOtp({ email })` |
| Phone OTP | `signInWithOtp({ phone })` |
| Get User | `auth.getUser()` |
| Sign Out | `auth.signOut()` |

---

## Resources

- [Official Supabase Documentation](https://supabase.com/docs)
- [Supabase Best Practices](https://www.leanware.co/insights/supabase-best-practices)
- [2025 Security Guide](https://github.com/orgs/supabase/discussions/38690)
- [RLS Performance Guide](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv)
- [Production Checklist](https://supabase.com/docs/guides/deployment/going-into-prod)
- [CLI Best Practices](https://bix-tech.com/supabase-cli-best-practices-how-to-boost-security-and-control-in-your-development-workflow/)
