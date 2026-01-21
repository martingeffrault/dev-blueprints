# pnpm (2025-2026)

> **Last updated**: January 2026
> **Versions covered**: 9.x, 10.x
> **Key feature**: Security by default, catalogs, content-addressable storage

---

## Philosophy (2025-2026)

pnpm has evolved into the **recommended package manager for modern JavaScript projects**, especially monorepos. Its **content-addressable storage** and **strict dependency resolution** provide both performance and security advantages over npm and Yarn.

**Key philosophical shifts:**
- **Security by default** - Lifecycle scripts blocked by default (v10+)
- **Content-addressable store** - Hard links save 60-80% disk space
- **Strict node_modules** - No phantom dependencies
- **Workspace-first** - Built-in monorepo support without external tools
- **Catalogs** - Centralized version management (v9.5+)
- **Config dependencies** - Shareable pnpm configuration across projects

### pnpm vs npm vs Yarn (2025)

| Feature | pnpm | npm | Yarn |
|---------|------|-----|------|
| **Disk usage** | 60-80% less | Baseline | Similar to npm |
| **Install speed** | Fastest | Slowest | Fast |
| **Monorepo support** | Built-in | Workspaces | Workspaces |
| **Phantom deps** | Prevented | Allowed | Allowed |
| **Security defaults** | Scripts blocked | Scripts run | Scripts run |
| **Market share (2024)** | ~20% (growing) | ~57% | ~22% |

**Recommendation**: Use pnpm for new projects in 2025, especially monorepos. npm remains viable for simple projects or when pnpm isn't available.

---

## TL;DR

- Use pnpm 10+ for new projects (security by default)
- Always commit `pnpm-lock.yaml` to version control
- Use `pnpm install --frozen-lockfile` in CI
- Use `workspace:*` protocol for internal package references
- Use `catalog:` protocol to centralize dependency versions
- Explicitly allow lifecycle scripts with `onlyBuiltDependencies`
- Run `pnpm store prune` occasionally to clean global store
- Use `pnpm -r` for recursive commands across workspaces

---

## Best Practices

### Installation and Setup

```bash
# Install pnpm (recommended methods)
corepack enable pnpm               # Node.js 16.13+
npm install -g pnpm               # Traditional install
curl -fsSL https://get.pnpm.io/install.sh | sh -  # Standalone

# Verify installation
pnpm --version
```

### Project Setup

```json
// package.json
{
  "name": "my-project",
  "version": "1.0.0",
  "type": "module",
  "packageManager": "pnpm@10.2.0",
  "engines": {
    "node": ">=20.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest"
  }
}
```

### pnpm-workspace.yaml Configuration

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'           # All apps
  - 'packages/*'       # Shared packages
  - 'tools/*'          # Internal tooling
  - '!**/test/**'      # Exclude test directories

# Catalogs - centralized version management (v9.5+)
catalog:
  react: ^19.0.0
  react-dom: ^19.0.0
  typescript: ^5.7.0
  vitest: ^2.1.0
  vite: ^6.0.0

# Multiple named catalogs
catalogs:
  react18:
    react: ^18.3.0
    react-dom: ^18.3.0
  testing:
    vitest: ^2.1.0
    playwright: ^1.49.0
```

### Monorepo Structure

```
my-monorepo/
├── apps/
│   ├── web/
│   │   └── package.json
│   ├── api/
│   │   └── package.json
│   └── mobile/
│       └── package.json
├── packages/
│   ├── ui/
│   │   └── package.json
│   ├── utils/
│   │   └── package.json
│   └── config/
│       └── package.json
├── tools/
│   └── eslint-config/
│       └── package.json
├── package.json              # Root package.json
├── pnpm-workspace.yaml       # Workspace config
└── pnpm-lock.yaml            # Single lockfile
```

### Workspace Protocol for Internal Dependencies

```json
// apps/web/package.json
{
  "name": "@myorg/web",
  "dependencies": {
    "@myorg/ui": "workspace:*",
    "@myorg/utils": "workspace:^"
  }
}
```

```json
// packages/ui/package.json
{
  "name": "@myorg/ui",
  "version": "1.0.0",
  "dependencies": {
    "react": "catalog:",
    "react-dom": "catalog:"
  },
  "peerDependencies": {
    "react": "catalog:"
  }
}
```

### Catalog Protocol Usage

```json
// package.json using catalog
{
  "dependencies": {
    "react": "catalog:",
    "react-dom": "catalog:",
    "typescript": "catalog:"
  },
  "devDependencies": {
    "vitest": "catalog:testing",
    "playwright": "catalog:testing"
  }
}
```

### Lockfile Management

```bash
# CI/CD - fail if lockfile needs update
pnpm install --frozen-lockfile

# Update lockfile only (no node_modules changes)
pnpm install --lockfile-only

# Fix broken lockfile entries
pnpm install --fix-lockfile

# Force reinstall (recreate lockfile)
pnpm install --force
```

### Filtering Commands in Workspaces

```bash
# Run command in specific package
pnpm --filter @myorg/web dev

# Run in all packages
pnpm -r build

# Run in packages matching pattern
pnpm --filter "./apps/*" build

# Run in changed packages (git-based)
pnpm --filter "[HEAD^1]" test

# Run in package and its dependencies
pnpm --filter @myorg/web... build

# Run in package and its dependents
pnpm --filter ...@myorg/ui build

# Exclude packages
pnpm --filter "!@myorg/docs" build
```

### Security Configuration (v10+)

```json
// package.json - allow specific packages to run scripts
{
  "pnpm": {
    "onlyBuiltDependencies": [
      "esbuild",
      "prisma",
      "@prisma/client",
      "sharp",
      "bcrypt",
      "sqlite3"
    ]
  }
}
```

```yaml
# pnpm-workspace.yaml (v10.26+)
allowBuilds:
  esbuild: true
  prisma: true
  sharp: true
  "*": false  # Block all others by default

# Block exotic protocols in transitive deps
blockExoticSubdeps: true
```

### Dependency Overrides and Patches

```json
// package.json
{
  "pnpm": {
    "overrides": {
      "lodash": "^4.17.21",
      "foo@2": "npm:foo@3",
      "bar": "$bar",
      "unwanted-dep": "-"
    },
    "patchedDependencies": {
      "some-package@1.0.0": "patches/some-package@1.0.0.patch"
    }
  }
}
```

```bash
# Create a patch for a dependency
pnpm patch some-package@1.0.0

# Edit the package in the temporary directory
# Then commit the patch
pnpm patch-commit /path/to/temp/some-package
```

### Scripts and Lifecycle Hooks

```json
// package.json
{
  "scripts": {
    "preinstall": "node scripts/check-env.js",
    "prepare": "husky",
    "build": "tsc && vite build",
    "prebuild": "rimraf dist",
    "postbuild": "node scripts/post-build.js"
  }
}
```

```ini
# .npmrc - enable pre/post scripts for custom commands
enable-pre-post-scripts=true
```

### Security Auditing

```bash
# Check for vulnerabilities
pnpm audit

# Auto-fix vulnerabilities
pnpm audit --fix

# Production dependencies only
pnpm audit --prod

# Output as JSON
pnpm audit --json

# Set severity threshold
pnpm audit --audit-level=high
```

### Publishing Packages

```bash
# Publish single package
pnpm publish --access public

# Publish all changed packages in workspace
pnpm -r publish

# Dry run
pnpm publish --dry-run

# Publish with specific tag
pnpm publish --tag beta
```

```json
// package.json for publishable package
{
  "name": "@myorg/utils",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "files": ["dist"],
  "publishConfig": {
    "access": "public",
    "directory": "dist"
  }
}
```

### Using Changesets for Monorepo Publishing

```bash
# Install changesets
pnpm add -Dw @changesets/cli

# Initialize
pnpm changeset init

# Create a changeset
pnpm changeset

# Version packages
pnpm changeset version

# Publish
pnpm changeset publish
```

### Store Management

```bash
# View store path
pnpm store path

# Remove orphaned packages
pnpm store prune

# Verify store integrity
pnpm store status

# Add package to store without installing
pnpm store add react@19.0.0
```

### .npmrc Configuration

```ini
# .npmrc
# Use frozen lockfile in CI
frozen-lockfile=true

# Hoist patterns for specific tools
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*

# Strict peer dependencies
strict-peer-dependencies=true

# Auto-install peers
auto-install-peers=true

# Node linker mode
node-linker=hoisted

# Use shared workspace lockfile (default)
shared-workspace-lockfile=true

# Store directory
store-dir=~/.pnpm-store

# Enable pre/post scripts
enable-pre-post-scripts=true

# Registry
registry=https://registry.npmjs.org/
```

---

## Anti-Patterns

### Do Not Run install Without Lockfile in CI

**Why it's bad**: Non-deterministic builds, potential security issues.

```bash
# DON'T - allows lockfile updates
pnpm install

# DO - fail if lockfile needs update
pnpm install --frozen-lockfile
```

### Do Not Use Relative Paths for Workspace Dependencies

**Why it's bad**: Breaks portability and publishing.

```json
// DON'T
{
  "dependencies": {
    "@myorg/utils": "../packages/utils"
  }
}

// DO
{
  "dependencies": {
    "@myorg/utils": "workspace:*"
  }
}
```

### Do Not Ignore the Lockfile

**Why it's bad**: Different builds get different dependency versions.

```gitignore
# DON'T
pnpm-lock.yaml

# DO - always commit lockfile
# (no entry for pnpm-lock.yaml in .gitignore)
```

### Do Not Use * for External Dependencies

**Why it's bad**: Unpredictable updates, security risks.

```json
// DON'T
{
  "dependencies": {
    "lodash": "*",
    "axios": "latest"
  }
}

// DO
{
  "dependencies": {
    "lodash": "^4.17.21",
    "axios": "^1.7.0"
  }
}
```

### Do Not Blindly Allow All Build Scripts

**Why it's bad**: Supply chain attack vector.

```json
// DON'T - allows all scripts
{
  "pnpm": {
    "onlyBuiltDependencies": ["*"]
  }
}

// DO - explicit allowlist
{
  "pnpm": {
    "onlyBuiltDependencies": [
      "esbuild",
      "prisma"
    ]
  }
}
```

### Do Not Mix Package Managers

**Why it's bad**: Different lockfiles, inconsistent installs.

```
# DON'T
project/
  package-lock.json   # npm
  pnpm-lock.yaml      # pnpm
  yarn.lock           # yarn

# DO - use only one
project/
  pnpm-lock.yaml
```

### Do Not Access Phantom Dependencies

**Why it's bad**: Works locally but fails in production.

```typescript
// DON'T - lodash is a transitive dependency
import { debounce } from 'lodash'; // Not in your package.json!

// DO - explicitly add to dependencies
// First: pnpm add lodash
import { debounce } from 'lodash';
```

### Do Not Use Separate Lockfiles Per Package

**Why it's bad**: Massive performance hit, slower installs.

```yaml
# DON'T
# .npmrc
shared-workspace-lockfile=false

# DO - use default (shared lockfile)
# .npmrc
shared-workspace-lockfile=true
```

### Do Not Prune Store Too Frequently

**Why it's bad**: Removes packages you might need again soon.

```bash
# DON'T - in every CI run
pnpm store prune

# DO - occasionally, during maintenance
pnpm store prune
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 9.0 | Apr 2024 | Node 18+ required, improved workspace support |
| 9.5 | Jul 2024 | **Catalogs** introduced for centralized versions |
| 9.12 | Dec 2024 | `catalog:` support in `pnpm add` |
| 10.0 | Jan 2025 | **Security by default** - lifecycle scripts blocked |
| 10.7 | Apr 2025 | `allowUnusedPatches` setting added |
| 10.26 | Dec 2025 | `allowBuilds` setting, `blockExoticSubdeps` |

### Major v10 Breaking Changes

1. **Lifecycle scripts blocked** - Dependencies can't run install scripts by default
2. **Git dependencies** - `prepare` scripts blocked unless explicitly allowed
3. **Explicit permissions** - Use `onlyBuiltDependencies` or `allowBuilds`
4. **Config dependencies** - New way to share pnpm config across projects

### Notable npm Supply Chain Attacks (2025)

| Date | Attack | Impact |
|------|--------|--------|
| Sep 2025 | Chalk/debug compromise | 2.6B weekly downloads affected |
| Sep 2025 | Shai-Hulud worm | 500+ packages compromised |
| Nov 2025 | SHA1-Hulud wave | Second iteration of attack |

**pnpm v10's default script blocking would have prevented these attacks.**

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Install dependencies | `pnpm install` |
| Install (CI mode) | `pnpm install --frozen-lockfile` |
| Add dependency | `pnpm add <pkg>` |
| Add dev dependency | `pnpm add -D <pkg>` |
| Add to root (workspace) | `pnpm add -Dw <pkg>` |
| Remove dependency | `pnpm remove <pkg>` |
| Update dependency | `pnpm update <pkg>` |
| Update all | `pnpm update` |
| Run script | `pnpm run <script>` |
| Run script (shorthand) | `pnpm <script>` |

### Workspace Commands

| Task | Command |
|------|---------|
| Run in all packages | `pnpm -r <cmd>` |
| Run in filtered packages | `pnpm --filter <pattern> <cmd>` |
| Add to specific package | `pnpm --filter <pkg> add <dep>` |
| List packages | `pnpm list -r` |
| Why is package installed | `pnpm why <pkg>` |

### Filter Patterns

| Pattern | Description |
|---------|-------------|
| `@scope/name` | Exact package name |
| `./apps/*` | Directory glob |
| `[HEAD^1]` | Changed since commit |
| `...@scope/name` | Package and dependents |
| `@scope/name...` | Package and dependencies |
| `!@scope/name` | Exclude package |

### Security Commands

| Task | Command |
|------|---------|
| Security audit | `pnpm audit` |
| Auto-fix vulnerabilities | `pnpm audit --fix` |
| Check outdated | `pnpm outdated` |
| Update interactively | `pnpm update -i` |

### Store Commands

| Task | Command |
|------|---------|
| View store path | `pnpm store path` |
| Prune orphaned packages | `pnpm store prune` |
| Check store status | `pnpm store status` |

---

## CI/CD Configuration

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 10

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build
        run: pnpm build

      - name: Test
        run: pnpm test
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - install
  - build
  - test

variables:
  PNPM_HOME: /root/.local/share/pnpm

install:
  image: node:22
  stage: install
  before_script:
    - corepack enable pnpm
  script:
    - pnpm install --frozen-lockfile
  cache:
    key: pnpm-$CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
      - .pnpm-store/
```

---

## Resources

- [Official pnpm Documentation](https://pnpm.io/)
- [pnpm Workspaces Guide](https://pnpm.io/workspaces)
- [pnpm Catalogs](https://pnpm.io/catalogs)
- [pnpm Supply Chain Security](https://pnpm.io/supply-chain-security)
- [pnpm 2025 Blog Post](https://pnpm.io/blog/2025/12/29/pnpm-in-2025)
- [Monorepo Guide with pnpm + Changesets](https://jsdev.space/complete-monorepo-guide/)
- [pnpm Benchmarks](https://pnpm.io/benchmarks)
