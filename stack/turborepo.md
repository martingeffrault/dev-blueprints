# Turborepo / Monorepo (2025-2026)

> **Last updated**: January 2026
> **Versions covered**: Turborepo 2.0 - 2.7+
> **Package Managers**: pnpm (recommended), npm, yarn, bun

---

## Philosophy (2025-2026)

Turborepo in 2025-2026 has solidified its position as the **fastest, most developer-friendly monorepo tool** for JavaScript/TypeScript ecosystems. Backed by Vercel, it focuses on one thing: **making your builds as fast as possible**.

**Key philosophical principles:**

- **Speed is the priority** — Content-aware caching prevents duplicate work
- **Zero-config remote caching** — Vercel Remote Cache is free and automatic
- **Incremental adoption** — Drop into existing repos without major refactoring
- **Convention over configuration** — Sensible defaults, minimal turbo.json
- **Package-first architecture** — Each package is its own isolated unit
- **Task orchestration** — Define dependencies between tasks, run in parallel

### Turborepo vs Nx (2025)

| Aspect | Turborepo | Nx |
|--------|-----------|-----|
| **Philosophy** | Fast task runner, minimal config | Full ecosystem with structure |
| **Setup time** | Minutes | Hours for large setups |
| **Config complexity** | ~20 lines typical | 100-200+ lines |
| **Learning curve** | Low | Moderate to high |
| **Remote caching** | Free with Vercel | Nx Cloud (paid tiers) |
| **Best for** | JS/TS projects, speed-focused teams | Large enterprise, polyglot |
| **Architecture help** | Minimal (you manage) | Extensive (generators, graphs) |

**Choose Turborepo if:** Your main problem is slow CI/builds, you want minimal config, and your team can self-manage architecture.

**Choose Nx if:** You need extensive tooling, generators, and structure from day one for a large, complex project.

---

## TL;DR

- Use `apps/` for deployable applications, `packages/` for shared libraries
- Configure task dependencies in `turbo.json` using `dependsOn`
- Use `^` prefix for topological dependencies (dependencies build first)
- Enable remote caching with Vercel (free, zero-config)
- Use `--affected` in CI to only build/test changed packages
- Namespace internal packages: `@repo/ui`, `@repo/config`
- Share configs via internal packages: `@repo/eslint-config`, `@repo/typescript-config`
- Use `turbo dev` for parallel dev servers with the interactive UI
- Never put secrets in environment variables tracked by Turborepo
- Run `turbo devtools` to visualize package and task graphs

---

## Best Practices

### Project Structure (2025)

```
my-monorepo/
├── apps/
│   ├── web/                    # Next.js frontend
│   │   ├── package.json
│   │   └── next.config.js
│   ├── api/                    # Express/Hono backend
│   │   └── package.json
│   └── mobile/                 # React Native app
│       └── package.json
├── packages/
│   ├── ui/                     # Shared UI components
│   │   ├── package.json
│   │   └── src/
│   ├── database/               # Prisma/Drizzle schema + client
│   │   └── package.json
│   ├── types/                  # Shared TypeScript types
│   │   └── package.json
│   ├── utils/                  # Shared utilities
│   │   └── package.json
│   ├── eslint-config/          # Shared ESLint config
│   │   └── package.json
│   └── typescript-config/      # Shared tsconfig bases
│       └── package.json
├── turbo.json                  # Task configuration
├── package.json                # Root workspace config
├── pnpm-workspace.yaml         # pnpm workspace definition
└── .gitignore
```

### Root package.json

```json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "clean": "turbo run clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^2.7.0"
  },
  "packageManager": "pnpm@9.15.0"
}
```

### pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### turbo.json Configuration

```json
{
  "$schema": "https://turborepo.dev/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"],
      "inputs": ["src/**", "package.json", "tsconfig.json"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**", "test/**", "*.test.ts", "*.test.tsx"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

### Understanding dependsOn

```json
{
  "tasks": {
    // ^build = wait for dependencies' build tasks first (topological)
    "build": {
      "dependsOn": ["^build"]
    },

    // test depends on same-package build completing
    "test": {
      "dependsOn": ["build"]
    },

    // lint depends on typecheck in same package
    "lint": {
      "dependsOn": ["typecheck"]
    },

    // Specific package dependency
    "deploy": {
      "dependsOn": ["web#build", "api#build"]
    }
  }
}
```

### Shared TypeScript Config Package

```
packages/typescript-config/
├── package.json
├── base.json
├── nextjs.json
├── react-library.json
└── node.json
```

**package.json:**
```json
{
  "name": "@repo/typescript-config",
  "version": "0.0.0",
  "private": true,
  "files": ["*.json"]
}
```

**base.json:**
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2022",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true
  },
  "exclude": ["node_modules"]
}
```

**nextjs.json:**
```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "ES2022"],
    "jsx": "preserve",
    "plugins": [{ "name": "next" }]
  }
}
```

**App tsconfig.json:**
```json
{
  "extends": "@repo/typescript-config/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### Shared ESLint Config Package (ESLint v9 Flat Config)

```
packages/eslint-config/
├── package.json
├── base.js
├── next.js
├── react.js
└── node.js
```

**package.json:**
```json
{
  "name": "@repo/eslint-config",
  "version": "0.0.0",
  "private": true,
  "exports": {
    "./base": "./base.js",
    "./next": "./next.js",
    "./react": "./react.js",
    "./node": "./node.js"
  },
  "dependencies": {
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-react": "^7.35.0",
    "eslint-plugin-react-hooks": "^5.0.0"
  },
  "peerDependencies": {
    "eslint": "^9.0.0",
    "typescript": "^5.0.0"
  }
}
```

**base.js:**
```js
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import prettierConfig from "eslint-config-prettier";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  prettierConfig,
  {
    rules: {
      "@typescript-eslint/no-unused-vars": [
        "error",
        { argsIgnorePattern: "^_" }
      ],
      "@typescript-eslint/no-explicit-any": "warn"
    }
  },
  {
    ignores: ["dist/**", "node_modules/**", ".next/**"]
  }
);
```

**App eslint.config.js:**
```js
import baseConfig from "@repo/eslint-config/next";

export default [
  ...baseConfig,
  {
    // App-specific overrides
  }
];
```

### Shared UI Package

```
packages/ui/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts
    ├── button.tsx
    └── card.tsx
```

**package.json:**
```json
{
  "name": "@repo/ui",
  "version": "0.0.0",
  "private": true,
  "exports": {
    ".": "./src/index.ts",
    "./button": "./src/button.tsx",
    "./card": "./src/card.tsx"
  },
  "scripts": {
    "build": "tsc",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^19.0.0"
  },
  "devDependencies": {
    "@repo/eslint-config": "workspace:*",
    "@repo/typescript-config": "workspace:*",
    "typescript": "^5.7.0"
  }
}
```

**Using workspace protocol (pnpm):**
```json
{
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/types": "workspace:*"
  }
}
```

### Remote Caching Setup

**Vercel (Zero-config):**
```bash
# Link to Vercel (free)
npx turbo login
npx turbo link

# Or set environment variables in CI
TURBO_TOKEN=your_token
TURBO_TEAM=your_team
```

**Self-hosted (optional):**
```json
// turbo.json
{
  "remoteCache": {
    "signature": true
  }
}
```

### CI/CD Optimization (GitHub Actions)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2  # Needed for --affected

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile

      # Use --affected to only run tasks for changed packages
      - name: Build
        run: pnpm turbo build --affected

      - name: Lint
        run: pnpm turbo lint --affected

      - name: Test
        run: pnpm turbo test --affected

      - name: Typecheck
        run: pnpm turbo typecheck --affected
```

### Using --affected for Faster CI

```bash
# Only build packages affected by changes since main
turbo build --affected

# In PR context, automatically compares to base branch
# Uses GITHUB_BASE_REF in GitHub Actions

# Combine with filtering
turbo build --affected --filter="./apps/*"
```

### Package-Specific Configuration (turbo.json in packages)

```json
// apps/web/turbo.json
{
  "extends": ["//"],
  "tasks": {
    "build": {
      "outputs": [".next/**", "!.next/cache/**"],
      "env": ["NEXT_PUBLIC_API_URL"]
    }
  }
}
```

### Environment Variables

```json
// turbo.json
{
  "globalEnv": ["CI", "NODE_ENV"],
  "tasks": {
    "build": {
      "env": ["DATABASE_URL", "API_KEY"],
      "passThroughEnv": ["npm_config_registry"]
    }
  }
}
```

### Visualizing Your Monorepo (Turborepo 2.7+)

```bash
# Launch interactive devtools in browser
turbo devtools

# View package graph
turbo ls --graph

# View task graph for build
turbo build --graph

# Output as JSON
turbo build --dry-run=json
```

---

## Anti-Patterns

### Avoid Nested Workspace Patterns

**Why it's bad**: Package managers cannot reliably resolve deeply nested workspaces.

```yaml
# pnpm-workspace.yaml

# DON'T - Nested patterns cause resolution issues
packages:
  - "apps/**"
  - "packages/**"

# DO - Single level of nesting
packages:
  - "apps/*"
  - "packages/*"
```

### Avoid Missing Lockfile

**Why it's bad**: Turborepo uses the lockfile to understand package relationships.

```bash
# DON'T - Run turbo without lockfile
rm pnpm-lock.yaml
turbo build  # Unpredictable behavior!

# DO - Always commit and maintain lockfile
git add pnpm-lock.yaml
turbo build
```

### Avoid Caching Dev Tasks

**Why it's bad**: Dev servers need to run fresh every time.

```json
// turbo.json

// DON'T - Caching dev causes stale servers
{
  "tasks": {
    "dev": {
      "outputs": ["dist/**"]  // This caches dev output!
    }
  }
}

// DO - Disable caching for dev
{
  "tasks": {
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

### Avoid Incorrect dependsOn Direction

**Why it's bad**: Missing `^` causes tasks to run out of order.

```json
// DON'T - Missing ^ means "same package only"
{
  "tasks": {
    "build": {
      "dependsOn": ["build"]  // Circular reference!
    }
  }
}

// DO - Use ^ for dependency packages
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"]  // Dependencies first
    }
  }
}
```

### Avoid Forgetting Outputs

**Why it's bad**: Cache misses occur when outputs aren't declared.

```json
// DON'T - Missing outputs means no caching benefit
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"]
      // outputs not specified = nothing cached
    }
  }
}

// DO - Declare all outputs
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    }
  }
}
```

### Avoid Secrets in Tracked Environment Variables

**Why it's bad**: Secrets affect cache keys and may leak into logs.

```json
// turbo.json

// DON'T - Track secrets as env vars
{
  "tasks": {
    "build": {
      "env": ["API_SECRET", "DATABASE_PASSWORD"]
    }
  }
}

// DO - Use passThroughEnv for secrets (doesn't affect cache key)
{
  "tasks": {
    "build": {
      "env": ["NODE_ENV", "NEXT_PUBLIC_API_URL"],
      "passThroughEnv": ["API_SECRET", "DATABASE_PASSWORD"]
    }
  }
}
```

### Avoid Wide Glob Patterns in tsconfig

**Why it's bad**: Degrades linting and type-checking performance.

```json
// tsconfig.json

// DON'T - Recursive glob is slow
{
  "include": ["**/*.ts", "**/*.tsx"]
}

// DO - Specific paths
{
  "include": ["src/**/*.ts", "src/**/*.tsx"]
}
```

### Avoid Duplicating Dependencies

**Why it's bad**: Version mismatches and bloated node_modules.

```json
// packages/ui/package.json

// DON'T - Duplicate common deps in each package
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}

// DO - Use peerDependencies for shared deps
{
  "peerDependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "react": "^19.0.0"
  }
}
```

### Avoid Running All Tasks in CI

**Why it's bad**: Wastes resources rebuilding unchanged packages.

```yaml
# .github/workflows/ci.yml

# DON'T - Rebuild everything
- run: turbo build

# DO - Only affected packages
- run: turbo build --affected
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.0 | Jun 2024 | New terminal UI (TUI), strict env mode default, Rust rewrite complete |
| 2.1 | Aug 2024 | Affected package detection (`--affected`), UI polish, repository exploration |
| 2.2 | Oct 2024 | Performance improvements, better error messages |
| 2.3 | Nov 2024 | Extended package configurations, improved caching |
| 2.4 | Dec 2024 | Bug fixes, stability improvements |
| 2.5 | Jan 2025 | Better Bun support, performance optimizations |
| 2.6 | Feb 2025 | Bun lockfile v1 support, Yarn catalogs compatibility |
| 2.7 | Mar 2025 | **Devtools** (`turbo devtools`), extended `extends` key, Biome integration |
| 2.7.5 | Jan 2026 | `errorsOnlyShowHash` flag, new docs subcommand, OOM-killed task handling |

### Rust Rewrite (Complete)

Turborepo 2.0 completed a full rewrite from Go to Rust, providing:
- Significant performance improvements
- Better memory management
- Resolved many edge case bugs from the Go implementation
- Foundation for future optimizations

### Firebase App Hosting Support (January 2026)

Firebase App Hosting now officially supports Turborepo alongside Nx:
- Optimized builds for Next.js apps in monorepos
- Task scheduling and parallelization for faster deployments
- Seamless integration with Firebase deployment pipeline

### Key 2.0 Breaking Changes

| Change | Migration |
|--------|-----------|
| `$VAR` syntax removed in dependsOn | Use `env` key instead |
| `globalDotEnv`/`dotEnv` removed | Include .env in `inputs` |
| `--ignore` removed | Use `--filter` |
| Strict env mode default | Declare all env vars in `env` |
| Root is implicit dependency | No action needed |

### Key 2.7 Features (2025)

- **Visual Devtools**: Run `turbo devtools` for interactive graph visualization
- **Extended `extends`**: Package turbo.json can extend from any package, not just root
- **Biome noUndeclaredEnvVars**: Lint rule catches undeclared environment variables
- **Yarn 4.10 catalogs**: Lockfile parser supports new catalog syntax

---

## Quick Reference

### turbo.json Keys

| Key | Description | Example |
|-----|-------------|---------|
| `tasks` | Define task configurations | `{ "build": { ... } }` |
| `dependsOn` | Tasks that must complete first | `["^build", "lint"]` |
| `outputs` | Files to cache | `["dist/**"]` |
| `inputs` | Files that affect cache | `["src/**"]` |
| `cache` | Enable/disable caching | `false` for dev |
| `persistent` | Long-running task | `true` for dev servers |
| `env` | Env vars affecting cache key | `["NODE_ENV"]` |
| `passThroughEnv` | Env vars not affecting cache | `["API_SECRET"]` |
| `globalEnv` | Global env vars for all tasks | `["CI"]` |

### Common Commands

| Command | Description |
|---------|-------------|
| `turbo build` | Run build in all packages |
| `turbo dev` | Run dev servers (parallel) |
| `turbo build --filter=web` | Build specific package |
| `turbo build --filter=./apps/*` | Build all apps |
| `turbo build --affected` | Build only changed packages |
| `turbo build --dry-run` | Show what would run |
| `turbo build --graph` | Output task graph |
| `turbo devtools` | Launch visual devtools |
| `turbo ls` | List packages |
| `turbo login` | Authenticate with Vercel |
| `turbo link` | Link to Vercel Remote Cache |

### Filter Syntax

| Pattern | Description |
|---------|-------------|
| `--filter=web` | Specific package |
| `--filter=@repo/ui` | By package name |
| `--filter=./apps/*` | Glob path |
| `--filter=web...` | Package and dependents |
| `--filter=...web` | Package and dependencies |
| `--filter=web^...` | Only dependents (not web) |
| `--filter=[HEAD^1]` | Changed since commit |
| `--filter={./packages/*}[main...]` | Changed packages since main |

### Environment Variable Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `env` | Part of cache key | Feature flags, API URLs |
| `passThroughEnv` | Not in cache key | Secrets, tokens |
| `globalEnv` | Applies to all tasks | CI, NODE_ENV |

---

## Decision Tree

```
Are you setting up a new monorepo?
├── Yes → Use create-turbo: npx create-turbo@latest
└── No → Adding to existing repo?
    ├── Yes → Install turbo, create turbo.json, define tasks
    └── No → Just need faster CI?
        └── Enable remote caching + use --affected
```

---

## Resources

- [Official Turborepo Documentation](https://turborepo.dev/docs)
- [Turborepo Blog](https://turborepo.dev/blog)
- [Vercel Remote Cache](https://vercel.com/docs/monorepos/remote-caching)
- [Turborepo GitHub](https://github.com/vercel/turborepo)
- [Structuring a Repository](https://turborepo.dev/docs/crafting-your-repository/structuring-a-repository)
- [Configuring Tasks](https://turborepo.dev/docs/crafting-your-repository/configuring-tasks)
- [Constructing CI](https://turborepo.dev/docs/crafting-your-repository/constructing-ci)
