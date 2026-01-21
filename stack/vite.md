# Vite 6/7 (2025)

> **Last updated**: January 2026
> **Versions covered**: 6.x, 7.0+
> **Future**: Vite+, Rolldown (Rust bundler)

---

## Philosophy (2025-2026)

Vite 6 focuses on the **Environment API** for better SSR/client separation, improved build targets, and continues its mission of being the fastest development experience.

**Key philosophical shifts:**
- **Environment API** — Framework authors get closer-to-production dev experience
- **Vite+** — Unified toolchain announced (testing, linting, formatting built-in)
- **Rolldown coming** — Rust-based bundler for even faster builds
- **Modern defaults** — Target `baseline-widely-available` (Chrome 107+)
- **CSS-first Tailwind** — First-class v4 integration
- **No legacy support** — Node.js 18+ required

---

## TL;DR

- Use `vite.config.ts` (TypeScript config is standard)
- Set `base` option when deploying to subdirectories
- Use path aliases via `resolve.alias` for cleaner imports
- Use `.env.development` and `.env.production` for environment-specific config
- Vite 6 dropped Node.js 21 support — use 18, 20, or 22+
- Default build target is now `baseline-widely-available`
- Sass legacy API removed — use modern API only

---

## Best Practices

### Basic Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],

  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@lib': path.resolve(__dirname, './src/lib'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
    },
  },

  server: {
    port: 3000,
    open: true,
  },

  build: {
    target: 'esnext',
    sourcemap: true,
  },
});
```

### Environment Variables

```bash
# .env.development
VITE_API_URL=http://localhost:8000
VITE_DEBUG=true

# .env.production
VITE_API_URL=https://api.example.com
VITE_DEBUG=false
```

```typescript
// Access in code (must be prefixed with VITE_)
const apiUrl = import.meta.env.VITE_API_URL;
const isDev = import.meta.env.DEV;
const isProd = import.meta.env.PROD;
```

### Dynamic Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig(({ command, mode }) => {
  const isDev = command === 'serve';
  const isProd = mode === 'production';

  return {
    base: isProd ? '/my-app/' : '/',
    define: {
      __DEV__: isDev,
    },
    build: {
      sourcemap: isDev,
    },
  };
});
```

### Project Structure

```
my-app/
├── public/              # Static assets (served as-is)
│   └── favicon.ico
├── src/
│   ├── components/      # React components
│   ├── hooks/           # Custom hooks
│   ├── lib/             # Utilities
│   ├── styles/          # Global styles
│   ├── App.tsx
│   └── main.tsx
├── tests/               # Test files
├── .env.development     # Dev environment
├── .env.production      # Prod environment
├── index.html           # Entry HTML
├── vite.config.ts       # Vite config
├── tsconfig.json
└── package.json
```

### Optimizing Dependencies

```typescript
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: ['lodash-es', 'axios'], // Pre-bundle these
    exclude: ['my-local-package'],   // Don't pre-bundle
  },

  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash-es', 'date-fns'],
        },
      },
    },
  },
});
```

### CSS Configuration

```typescript
// vite.config.ts
export default defineConfig({
  css: {
    modules: {
      localsConvention: 'camelCase',
    },
    preprocessorOptions: {
      scss: {
        // Modern API only in Vite 6
        additionalData: `@use "@/styles/variables" as *;`,
      },
    },
  },
});
```

### Proxy Configuration

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
});
```

---

## Anti-Patterns

### ❌ Not Setting `base` for Subdirectory Deployment

**Why it's bad**: Assets won't load correctly.

```typescript
// ❌ DON'T — Deploying to /app/ without base
export default defineConfig({
  // Missing base option
});

// ✅ DO
export default defineConfig({
  base: '/app/',
});
```

### ❌ Using Environment Variables Without VITE_ Prefix

**Why it's bad**: Variables won't be exposed to client code.

```bash
# ❌ DON'T
API_KEY=secret  # Not accessible in client

# ✅ DO
VITE_API_KEY=secret  # Accessible via import.meta.env.VITE_API_KEY
```

### ❌ Importing Large Libraries Without Chunking

**Why it's bad**: Creates huge bundles, slow initial load.

```typescript
// ❌ DON'T — All in one bundle
import _ from 'lodash';

// ✅ DO — Tree-shakeable imports
import { debounce, throttle } from 'lodash-es';

// ✅ OR — Dynamic imports
const { Chart } = await import('chart.js');
```

### ❌ Using Sass Legacy API

**Why it's bad**: Removed in Vite 6.

```typescript
// ❌ DON'T — Legacy API (removed)
css: {
  preprocessorOptions: {
    scss: {
      api: 'legacy', // Error in Vite 6
    },
  },
}

// ✅ DO — Modern API (default)
css: {
  preprocessorOptions: {
    scss: {
      additionalData: `@use "variables" as *;`,
    },
  },
}
```

### ❌ Hardcoding Paths

**Why it's bad**: Breaks on deployment.

```typescript
// ❌ DON'T
<img src="/images/logo.png" />  // Breaks with base path

// ✅ DO — Use import
import logo from '@/assets/logo.png';
<img src={logo} />
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **6.0** | Nov 2024 | Environment API, ssr boolean → environment, Sass legacy removed |
| **6.x** | 2025 | Environment API RC phase |
| **7.0** | Dec 2025 | ESM-only dist, Rust preview server |
| Vite+ | 2025 | Announced — unified toolchain |
| Future | 2026 | Rolldown integration (Rust bundler) |

### Vite 6.0 — Environment API (Game Changer)

**What it is**: The most significant major release since Vite 2. Framework authors can now create multiple environments (client, SSR, workerd, etc.) that closely match production.

**Why it matters**: Edge deployment (Cloudflare Workers, Vercel Edge) is now as convenient as Node.js development.

```typescript
// vite.config.ts — Multiple environments
import { defineConfig } from 'vite';

export default defineConfig({
  environments: {
    client: {
      // Standard browser environment (default)
    },
    ssr: {
      // Node.js SSR environment
      optimizeDeps: {
        include: ['some-ssr-dep'],
      },
    },
    workerd: {
      // Cloudflare Workers environment
      webCompatible: true,
      resolve: {
        conditions: ['workerd'],
      },
    },
  },
});
```

**Plugin Hook Changes (Breaking):**
```typescript
// ❌ OLD — ssr boolean in last options parameter
export default function myPlugin() {
  return {
    name: 'my-plugin',
    resolveId(id, importer, options) {
      if (options.ssr) {
        // Handle SSR
      }
    },
  };
}

// ✅ NEW — Use this.environment
export default function myPlugin() {
  return {
    name: 'my-plugin',
    resolveId(id, importer, options) {
      if (this.environment.name === 'ssr') {
        // Handle SSR
      }
      // Or check environment mode:
      if (this.environment.mode === 'ssr') {
        // Handle any SSR-like environment
      }
    },
  };
}
```

**API Status**:
- Environment API is in **RC phase** — stable between major releases
- Some specific APIs still marked experimental
- Backward compatible for SPAs and existing SSR setups

### Vite 6.0 Breaking Changes

**Build Target Renamed:**
```typescript
// ❌ OLD — 'modules' target
export default defineConfig({
  build: {
    target: 'modules', // Error in Vite 6
  },
});

// ✅ NEW — 'baseline-widely-available'
export default defineConfig({
  build: {
    target: 'baseline-widely-available', // Chrome 107+, Edge 107+, Firefox 104+, Safari 16+
  },
});
```

**Sass Legacy API Removed:**
```typescript
// ❌ OLD — Legacy API option
css: {
  preprocessorOptions: {
    scss: {
      api: 'legacy', // Removed in Vite 6!
    },
  },
}

// ✅ NEW — Modern API is the only option
css: {
  preprocessorOptions: {
    scss: {
      // api option removed — modern API only
      additionalData: `@use "variables" as *;`,
    },
  },
}
```

**CSS Output File Name in Library Mode:**
```typescript
// ❌ OLD — CSS output was always style.css
// ✅ NEW — CSS uses name from package.json or build.lib.fileName
export default defineConfig({
  build: {
    lib: {
      fileName: 'my-lib', // CSS output: my-lib.css (not style.css)
    },
  },
});
```

**Node.js Support:**
- Node.js 18, 20, 22+ supported
- Node.js 21 dropped

### Vite 7.0 Highlights

**ESM-only Distribution:**
```bash
# Vite 7 distributed as ESM only
# CJS still works via Node.js 20.19+ require(esm) support
# Existing plugins work without modification
```

**Baseline Support:**
```typescript
// NEW — Interop 2025 browser support
export default defineConfig({
  build: {
    target: 'baseline', // Baseline Widely Available features
  },
});
```

**Rust Integration:**
- `vite preview` uses Rust-based server
- Foundation for Rolldown bundler integration

### Vite+ (Announced 2025)

Vite+ is a superset of Vite with additional commands:
- Built-in testing (Vitest integration)
- Built-in linting
- Built-in formatting
- Library bundling
- Monorepo support
- Built-in caching

---

## Quick Reference

| Task | Solution |
|------|----------|
| Dev server | `vite` |
| Build | `vite build` |
| Preview build | `vite preview` |
| Env vars | `import.meta.env.VITE_*` |
| Path alias | `resolve.alias` in config |
| Proxy API | `server.proxy` in config |
| Manual chunks | `build.rollupOptions.output.manualChunks` |
| CSS modules | `*.module.css` files |
| Static assets | `/public` folder |
| Profiling | `vite --profile` |

---

## Performance Tips

1. **Use `optimizeDeps.include`** — Pre-bundle known dependencies
2. **Enable sourcemaps only in dev** — Faster production builds
3. **Split vendor chunks** — Better caching
4. **Use dynamic imports** — Code splitting for routes
5. **Profile slow builds** — `vite --profile` + Speedscope

---

## Resources

- [Official Vite Documentation](https://vite.dev/)
- [Vite Configuration Reference](https://vite.dev/config/)
- [Migration from v5](https://vite.dev/guide/migration)
- [ViteConf 2025 Recap](https://voidzero.dev/posts/whats-new-viteconf-2025)
- [Breaking Changes](https://vite.dev/changes/)
