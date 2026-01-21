# Astro (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.x
> **Purpose**: Content-focused web framework with Islands Architecture

---

## Philosophy (2025-2026)

Astro is the **content-first framework** that ships zero JavaScript by default, using Islands Architecture for selective hydration.

**Key philosophical shifts:**
- **Zero JS by default** — Ship only what's needed
- **Islands Architecture** — Selective hydration for interactivity
- **Content Collections** — Type-safe content management
- **UI agnostic** — Use React, Vue, Svelte, Solid, or vanilla
- **Server Islands (v5)** — Deferred dynamic content
- **Static + Dynamic** — Unified output modes

---

## TL;DR

- Ship zero JS by default — add islands for interactivity
- Use `client:visible` or `client:idle` instead of `client:load`
- Content Collections for blogs, docs, structured data
- Don't wrap layouts in framework components
- Static lists → hydrate single controller island
- Use `.astro` components for static content
- Framework components only for interactivity
- Server Islands for personalized/dynamic content

---

## Best Practices

### Project Structure

```
my-astro-site/
├── src/
│   ├── components/
│   │   ├── Header.astro         # Static component
│   │   ├── Footer.astro
│   │   ├── Card.astro
│   │   └── react/               # Framework components (islands)
│   │       ├── SearchBar.tsx
│   │       └── CartButton.tsx
│   ├── content/
│   │   ├── config.ts            # Content collection schemas
│   │   ├── blog/
│   │   │   ├── first-post.md
│   │   │   └── second-post.mdx
│   │   └── authors/
│   │       └── john.json
│   ├── layouts/
│   │   ├── BaseLayout.astro
│   │   ├── BlogLayout.astro
│   │   └── DocsLayout.astro
│   ├── pages/
│   │   ├── index.astro
│   │   ├── about.astro
│   │   ├── blog/
│   │   │   ├── index.astro
│   │   │   └── [...slug].astro
│   │   └── api/
│   │       └── search.ts
│   └── styles/
│       └── global.css
├── public/
│   ├── favicon.svg
│   └── images/
├── astro.config.mjs
└── package.json
```

### Astro Component (Static)

```astro
---
// src/components/Card.astro
interface Props {
  title: string;
  description: string;
  href: string;
  image?: string;
}

const { title, description, href, image } = Astro.props;
---

<article class="card">
  {image && <img src={image} alt={title} loading="lazy" />}
  <div class="content">
    <h3>{title}</h3>
    <p>{description}</p>
    <a href={href}>Read more →</a>
  </div>
</article>

<style>
  .card {
    border: 1px solid #eee;
    border-radius: 8px;
    overflow: hidden;
  }
  .content {
    padding: 1rem;
  }
</style>
```

### Layout with Slots

```astro
---
// src/layouts/BaseLayout.astro
interface Props {
  title: string;
  description?: string;
}

const { title, description = 'My Astro Site' } = Astro.props;
---

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content={description} />
    <title>{title}</title>
    <link rel="stylesheet" href="/styles/global.css" />
  </head>
  <body>
    <header>
      <nav>
        <a href="/">Home</a>
        <a href="/blog">Blog</a>
        <a href="/about">About</a>
      </nav>
    </header>

    <main>
      <slot />
    </main>

    <footer>
      <p>&copy; 2025 My Site</p>
    </footer>
  </body>
</html>
```

### Content Collections (Type-Safe)

```typescript
// src/content/config.ts
import { defineCollection, z } from 'astro:content';

const blogCollection = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    author: z.string(),
    tags: z.array(z.string()).default([]),
    image: z.string().optional(),
    draft: z.boolean().default(false),
  }),
});

const authorsCollection = defineCollection({
  type: 'data',
  schema: z.object({
    name: z.string(),
    bio: z.string(),
    avatar: z.string(),
    social: z.object({
      twitter: z.string().optional(),
      github: z.string().optional(),
    }).optional(),
  }),
});

export const collections = {
  blog: blogCollection,
  authors: authorsCollection,
};
```

```markdown
---
# src/content/blog/my-first-post.md
title: "My First Post"
description: "This is my first blog post"
pubDate: 2025-01-15
author: "John Doe"
tags: ["astro", "web"]
---

# Welcome to my blog!

This is the content of my first post...
```

### Querying Content Collections

```astro
---
// src/pages/blog/index.astro
import { getCollection } from 'astro:content';
import BaseLayout from '../../layouts/BaseLayout.astro';
import Card from '../../components/Card.astro';

// Get all non-draft blog posts, sorted by date
const posts = (await getCollection('blog', ({ data }) => !data.draft))
  .sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf());
---

<BaseLayout title="Blog">
  <h1>Blog Posts</h1>

  <div class="posts-grid">
    {posts.map((post) => (
      <Card
        title={post.data.title}
        description={post.data.description}
        href={`/blog/${post.slug}`}
        image={post.data.image}
      />
    ))}
  </div>
</BaseLayout>
```

### Dynamic Blog Post Page

```astro
---
// src/pages/blog/[...slug].astro
import { getCollection, type CollectionEntry } from 'astro:content';
import BlogLayout from '../../layouts/BlogLayout.astro';

export async function getStaticPaths() {
  const posts = await getCollection('blog');
  return posts.map((post) => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

interface Props {
  post: CollectionEntry<'blog'>;
}

const { post } = Astro.props;
const { Content } = await post.render();
---

<BlogLayout title={post.data.title}>
  <article>
    <h1>{post.data.title}</h1>
    <time datetime={post.data.pubDate.toISOString()}>
      {post.data.pubDate.toLocaleDateString()}
    </time>

    <Content />
  </article>
</BlogLayout>
```

### Islands (Interactive Components)

```tsx
// src/components/react/SearchBar.tsx
import { useState, useEffect } from 'react';

interface SearchResult {
  title: string;
  url: string;
}

export default function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isOpen, setIsOpen] = useState(false);

  useEffect(() => {
    if (query.length < 2) {
      setResults([]);
      return;
    }

    const timer = setTimeout(async () => {
      const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
      const data = await response.json();
      setResults(data);
    }, 300);

    return () => clearTimeout(timer);
  }, [query]);

  return (
    <div className="search-container">
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        onFocus={() => setIsOpen(true)}
        placeholder="Search..."
      />
      {isOpen && results.length > 0 && (
        <ul className="search-results">
          {results.map((result, i) => (
            <li key={i}>
              <a href={result.url}>{result.title}</a>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

```astro
---
// Using the island with client directive
import SearchBar from '../components/react/SearchBar';
---

<!-- Only hydrate when visible (best for non-critical) -->
<SearchBar client:visible />

<!-- Hydrate when browser is idle -->
<SearchBar client:idle />

<!-- Hydrate immediately (use sparingly!) -->
<SearchBar client:load />

<!-- Only hydrate on specific media query -->
<nav client:media="(max-width: 768px)">
  <MobileMenu />
</nav>
```

### Server Islands (Astro 5)

```astro
---
// src/components/UserGreeting.astro
// Server Island - renders on demand
const user = await getUser(Astro.cookies.get('session'));
---

<div>
  {user ? (
    <p>Welcome back, {user.name}!</p>
  ) : (
    <a href="/login">Sign in</a>
  )}
</div>
```

```astro
---
// Using Server Island
import UserGreeting from '../components/UserGreeting.astro';
---

<header>
  <nav>...</nav>
  <!-- Deferred server rendering -->
  <UserGreeting server:defer>
    <!-- Fallback while loading -->
    <div slot="fallback">Loading...</div>
  </UserGreeting>
</header>
```

### API Endpoints

```typescript
// src/pages/api/search.ts
import type { APIRoute } from 'astro';
import { getCollection } from 'astro:content';

export const GET: APIRoute = async ({ url }) => {
  const query = url.searchParams.get('q')?.toLowerCase() ?? '';

  if (query.length < 2) {
    return new Response(JSON.stringify([]), {
      headers: { 'Content-Type': 'application/json' },
    });
  }

  const posts = await getCollection('blog');
  const results = posts
    .filter(post =>
      post.data.title.toLowerCase().includes(query) ||
      post.data.description.toLowerCase().includes(query)
    )
    .slice(0, 5)
    .map(post => ({
      title: post.data.title,
      url: `/blog/${post.slug}`,
    }));

  return new Response(JSON.stringify(results), {
    headers: { 'Content-Type': 'application/json' },
  });
};
```

### Configuration

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import tailwind from '@astrojs/tailwind';
import mdx from '@astrojs/mdx';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://mysite.com',
  integrations: [
    react(),
    tailwind(),
    mdx(),
    sitemap(),
  ],
  output: 'hybrid', // static + server routes
  adapter: vercel(), // or cloudflare(), netlify(), etc.
});
```

---

## Anti-Patterns

### ❌ Using client:load Everywhere

**Why it's bad**: Defeats zero-JS benefit.

```astro
<!-- ❌ DON'T — Loads JS immediately -->
<SearchBar client:load />
<Newsletter client:load />
<Comments client:load />

<!-- ✅ DO — Lazy load when appropriate -->
<SearchBar client:idle />        <!-- When browser is idle -->
<Newsletter client:visible />   <!-- When scrolled into view -->
<Comments client:visible />     <!-- When visible -->
```

### ❌ Mapping Arrays to Islands

**Why it's bad**: Creates many hydration points.

```astro
<!-- ❌ DON'T — 100 islands for 100 items -->
{items.map(item => (
  <InteractiveCard client:visible item={item} />
))}

<!-- ✅ DO — Render static, hydrate controller -->
<div class="items-grid">
  {items.map(item => (
    <Card item={item} />  <!-- Static -->
  ))}
</div>
<ItemsSorter client:visible />  <!-- Single island for interactivity -->
```

### ❌ Wrapping Layouts in Framework Components

**Why it's bad**: Forces entire layout to hydrate.

```astro
<!-- ❌ DON'T -->
<ReactLayout client:load>
  <slot />
</ReactLayout>

<!-- ✅ DO — Use Astro layouts, islands for interactive parts -->
<BaseLayout>
  <Header />  <!-- Astro component -->
  <CartButton client:idle />  <!-- Small interactive island -->
  <slot />
</BaseLayout>
```

### ❌ Using Framework Components for Static Content

**Why it's bad**: Unnecessary JS overhead.

```astro
<!-- ❌ DON'T — React component for static content -->
<ReactCard client:load title="Hello" />

<!-- ✅ DO — Astro component (zero JS) -->
<Card title="Hello" />
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 4.0 | Dec 2023 | Dev toolbar, i18n routing |
| **5.0** | **Dec 2024** | **Server Islands**, Content Layer API, unified output |
| 5.15 | Oct 2025 | View Transitions baseline, stability |
| **6.0 alpha** | **Dec 2025** | Early preview available |

### Astro 5.0 Breaking Changes

**Content Collections Config Renamed:**
```typescript
// ❌ OLD — src/content/config.ts (deprecated)
// ✅ NEW — src/content/config.ts → src/content.config.ts

// The file is now at the root level, not inside content/
```

**Explicit Collection Definition Required:**
```typescript
// ❌ OLD — Collections auto-generated for all folders in src/content/
// ✅ NEW — Collections MUST be defined in content.config.ts

// If you have src/content/blog/ but don't define it,
// it will NOT be a collection anymore
```

**Content Layer API Changes:**
```typescript
// src/content.config.ts (NEW location!)
import { defineCollection, z } from 'astro:content';

// The loader option replaces "magic" file discovery
const blog = defineCollection({
  type: 'content', // or 'data'
  schema: z.object({
    title: z.string(),
    date: z.date(),
  }),
});

export const collections = { blog };
```

**Import Changes:**
```typescript
// ❌ OLD — z from astro:content deprecated
import { z } from 'astro:content';

// ✅ NEW — Use astro/zod
import { z } from 'astro/zod';
```

### Server Islands Enhancements

```astro
---
// Server Islands can now set headers (customize cache lifetime)
---
<UserProfile server:defer>
  <div slot="fallback">Loading profile...</div>
</UserProfile>
```

Features added since 5.0:
- Set headers inside islands (customize cache per-island)
- Works with automatic page compression
- Props automatically encrypted for privacy

### View Transitions Baseline (Oct 2025)

View Transitions are now **Baseline Newly Available** — supported in all major browsers!

```astro
---
import { ViewTransitions } from 'astro:transitions';
---
<head>
  <ViewTransitions />
</head>

<!-- Persist state across navigation -->
<Counter client:load transition:persist />

<!-- Keep existing props on navigation -->
<Island transition:persist transition:persist-props />
```

### Astro 6 Alpha (December 2025)

Early alpha of Astro v6 is available for testing:
```bash
npm install astro@next
```

Key changes being tested:
- Further Content Layer improvements
- Enhanced build performance
- New integration APIs

### Astro 5.0 Highlights

- **Server Islands**: Deferred rendering for dynamic content
- **Unified output modes**: Static and hybrid merged
- **Content Layer API**: Explicit content configuration
- **Type-safe env variables**: `astro:env` module
- **Simplified prerendering**: Per-page control

---

## Quick Reference

| Directive | When to Use |
|-----------|-------------|
| `client:load` | Critical interactivity (use sparingly) |
| `client:idle` | Non-critical, load when browser idle |
| `client:visible` | Below the fold, load when visible |
| `client:media` | Specific viewport only |
| `client:only` | Never pre-render, client-only |

| Content Type | Use |
|--------------|-----|
| `.astro` | Static pages, layouts, components |
| `.md` / `.mdx` | Blog posts, documentation |
| `.ts` (pages/api) | API endpoints |
| `.tsx` / `.vue` | Interactive islands |

| Command | Purpose |
|---------|---------|
| `astro dev` | Start dev server |
| `astro build` | Build for production |
| `astro preview` | Preview production build |
| `astro add` | Add integrations |
| `astro check` | TypeScript checking |

---

## Resources

- [Official Astro Documentation](https://docs.astro.build/)
- [Astro Islands Architecture](https://docs.astro.build/en/concepts/islands/)
- [Astro 5.0 Release](https://astro.build/blog/astro-5/)
- [Content Collections Guide](https://docs.astro.build/en/guides/content-collections/)
- [Islands Architecture Explained](https://strapi.io/blog/astro-islands-architecture-explained-complete-guide)
- [Sharing State with Islands](https://frontendatscale.com/blog/islands-architecture-state/)
