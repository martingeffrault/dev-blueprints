# GitHub Actions (2025)

> **Last updated**: January 2026
> **Versions covered**: GitHub Actions 2024-2025
> **Purpose**: CI/CD automation with security best practices

---

## Philosophy (2025-2026)

GitHub Actions is the **most widely used CI/CD platform** (51% adoption per CNCF 2024). In 2025, security is paramount — the tj-actions/changed-files compromise affected 23,000+ repositories. Every workflow is an attack surface.

**Key principles:**
- **Security by default** — Pin actions to SHAs, use OIDC
- **Least privilege** — Minimal GITHUB_TOKEN permissions
- **Reusable workflows** — DRY across repositories
- **Fast feedback** — Parallel jobs, caching
- **Immutable builds** — Reproducible artifacts

---

## TL;DR

- Pin third-party actions to full commit SHAs
- Use OIDC for cloud authentication (no long-lived secrets)
- Set minimal `permissions` for GITHUB_TOKEN
- Never trust user input in `${{ }}` expressions
- Use reusable workflows for consistency
- Enable Dependabot for action updates
- Cache dependencies aggressively
- Use matrix builds for multiple versions

---

## Best Practices

### Secure Workflow Template

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Restrict default permissions
permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci
      - run: npm test -- --coverage

      - uses: codecov/codecov-action@e28ff129e5465c2c0dcc6f003fc735cb6ae0c673 # v5.0.7
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - uses: actions/upload-artifact@6f51ac03b9356f520e9adb1b1b7802705f340c2b # v4.5.0
        with:
          name: build
          path: dist/
          retention-days: 7
```

### OIDC Authentication (AWS)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Required for OIDC

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      # OIDC authentication - no long-lived credentials!
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      - run: aws s3 sync dist/ s3://my-bucket/
```

### Matrix Builds

```yaml
name: Test Matrix

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        exclude:
          - os: macos-latest
            node: 18
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - run: npm ci
      - run: npm test
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '22'
      environment:
        required: true
        type: string
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'

      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

      - run: npm run build

      - uses: actions/upload-artifact@6f51ac03b9356f520e9adb1b1b7802705f340c2b # v4.5.0
        with:
          name: build-${{ inputs.environment }}
          path: dist/
```

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [develop]

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      environment: staging
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Caching Strategies

```yaml
name: Build with Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      # Node.js with built-in caching
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: 'npm'

      # Custom cache for build outputs
      - uses: actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a # v4.1.2
        with:
          path: |
            .next/cache
            node_modules/.cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-
            ${{ runner.os }}-nextjs-

      - run: npm ci
      - run: npm run build
```

### Docker Build & Push

```yaml
name: Docker

on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: docker/setup-buildx-action@c47758b77c9736f4b2ef4073d4d51994fabfe349 # v3.7.1

      - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@8e5442c4ef9f78752691e2d8f8d19755c6f78e81 # v5.5.1
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Release Workflow

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 0

      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
        with:
          node-version: 22
          cache: 'npm'
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
      - run: npm run build
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - uses: softprops/action-gh-release@c062e08bd532815e2082a85e87e3ef29c3e6d191 # v2.0.8
        with:
          generate_release_notes: true
          files: |
            dist/*.js
            dist/*.d.ts
```

### Security Scanning

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Weekly

permissions:
  contents: read
  security-events: write

jobs:
  codeql:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: github/codeql-action/init@662472033e021d55d94146f66f6058822b0b39fd # v3.27.0
        with:
          languages: javascript-typescript

      - uses: github/codeql-action/autobuild@662472033e021d55d94146f66f6058822b0b39fd # v3.27.0

      - uses: github/codeql-action/analyze@662472033e021d55d94146f66f6058822b0b39fd # v3.27.0

  dependency-review:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

      - uses: actions/dependency-review-action@4081bf99e2866ebe428f5a0e7c52ddd3a2370167 # v4.4.0
        with:
          fail-on-severity: high
```

### Composite Actions

```yaml
# .github/actions/setup-project/action.yml
name: Setup Project
description: Setup Node.js and install dependencies

inputs:
  node-version:
    description: Node.js version
    required: false
    default: '22'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - run: npm ci
      shell: bash

    - run: npm run build --if-present
      shell: bash
```

```yaml
# Usage in workflow
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: ./.github/actions/setup-project
      - run: npm test
```

### Environment Protection

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: echo "Deploying to staging"

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "Deploying to production"
```

### Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    groups:
      development:
        patterns:
          - '@types/*'
          - 'eslint*'
          - 'prettier*'
          - 'typescript'
        update-types:
          - minor
          - patch

  # GitHub Actions
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      actions:
        patterns:
          - '*'
```

---

## Anti-Patterns

### ❌ Using @latest or @main

```yaml
# ❌ DON'T — Mutable reference, can be compromised
- uses: some-org/some-action@main
- uses: some-org/some-action@v1

# ✅ DO — Pin to full commit SHA
- uses: some-org/some-action@abc123def456... # v1.2.3
```

### ❌ Excessive Permissions

```yaml
# ❌ DON'T — Overly permissive
permissions: write-all

# ✅ DO — Minimal permissions per job
permissions:
  contents: read

jobs:
  release:
    permissions:
      contents: write  # Only for release job
```

### ❌ Trusting User Input

```yaml
# ❌ DON'T — Code injection vulnerability
- run: echo "PR title: ${{ github.event.pull_request.title }}"

# ✅ DO — Use environment variable
- run: echo "PR title: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### ❌ Long-Lived Secrets

```yaml
# ❌ DON'T — Stored AWS credentials
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

# ✅ DO — OIDC authentication
- uses: aws-actions/configure-aws-credentials@...
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1
```

### ❌ No Caching

```yaml
# ❌ DON'T — Downloads every time
- run: npm install

# ✅ DO — Cache dependencies
- uses: actions/setup-node@...
  with:
    node-version: 22
    cache: 'npm'
- run: npm ci
```

---

## Quick Reference

| Event | Trigger |
|-------|---------|
| `push` | Direct push to branch |
| `pull_request` | PR opened/updated |
| `workflow_dispatch` | Manual trigger |
| `schedule` | Cron schedule |
| `release` | Release created |
| `workflow_call` | Reusable workflow |

| Context | Purpose |
|---------|---------|
| `github.sha` | Commit SHA |
| `github.ref` | Branch/tag ref |
| `github.actor` | User who triggered |
| `github.event_name` | Trigger event |
| `secrets.GITHUB_TOKEN` | Auto-generated token |
| `runner.os` | Runner OS |

| Permission | Purpose |
|------------|---------|
| `contents: read` | Checkout code |
| `contents: write` | Push, release |
| `packages: write` | Push to GHCR |
| `id-token: write` | OIDC auth |
| `security-events: write` | CodeQL results |

---

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Security Hardening Guide](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GitHub Actions Security Best Practices](https://www.stepsecurity.io/blog/github-actions-security-best-practices)
- [Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [OIDC with Cloud Providers](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
