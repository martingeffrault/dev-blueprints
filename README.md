# Dev Blueprints

Cutting-edge development best practices, anti-patterns, and conventions for 2025-2026. Designed to guide AI coding assistants toward modern, clean code.

---

## What This Is

A comprehensive reference for modern web development covering:

- **Stack guides** — Per-technology best practices and anti-patterns
- **Standards** — Naming conventions, git workflows, project structure
- **Security** — Authentication, web security, secrets management

Each file includes:
- Current philosophy of the technology (2025)
- Best practices with code examples
- Anti-patterns to avoid (with explanations)
- 2025 changelog of updates and breaking changes

---

## Who It's For

- **Developers** wanting to stay current with best practices
- **Teams** needing consistent coding standards
- **AI-assisted development** — Add to your project for better AI suggestions

---

## Quick Start

### Clone for AI Context

```bash
git clone https://github.com/martingeffrault/dev-blueprints.git
```

Add to your project's AI context (Cursor, Claude, Copilot) for best-practice-aware code generation.

### Browse Online

- [Stack Guides](stack/) — React, Next.js, TypeScript, etc.
- [Standards](standards/) — Naming, Git, API conventions
- [Security](security/) — Web security, auth, secrets

---

## Stack Coverage

| Technology | File | Versions |
|------------|------|----------|
| React | [stack/react.md](stack/react.md) | 19+ |
| Next.js | [stack/nextjs.md](stack/nextjs.md) | 15+ (App Router) |
| TypeScript | [stack/typescript.md](stack/typescript.md) | 5.x |
| Tailwind CSS | [stack/tailwind.md](stack/tailwind.md) | 4.x |
| Node.js | [stack/node.md](stack/node.md) | 22+ |
| PostgreSQL | [stack/postgres.md](stack/postgres.md) | 16+ |
| Prisma | [stack/prisma.md](stack/prisma.md) | 6.x |
| Drizzle | [stack/drizzle.md](stack/drizzle.md) | Latest |
| TanStack Query | [stack/tanstack-query.md](stack/tanstack-query.md) | 5.x |
| Zustand | [stack/zustand.md](stack/zustand.md) | 5.x |
| Zod | [stack/zod.md](stack/zod.md) | 3.x |
| tRPC | [stack/trpc.md](stack/trpc.md) | 11.x |
| Vite | [stack/vite.md](stack/vite.md) | 6.x |
| Vitest | [stack/vitest.md](stack/vitest.md) | 2.x |
| Playwright | [stack/playwright.md](stack/playwright.md) | Latest |

---

## Standards Coverage

| Standard | File | Scope |
|----------|------|-------|
| Naming Conventions | [standards/naming.md](standards/naming.md) | Files, variables, CSS, components |
| Git Workflow | [standards/git.md](standards/git.md) | Branches, commits, PRs, tags |
| API Design | [standards/api.md](standards/api.md) | REST, GraphQL, versioning |
| Database | [standards/database.md](standards/database.md) | Tables, columns, migrations |
| Project Structure | [standards/project-structure.md](standards/project-structure.md) | Folder organization |
| Documentation | [standards/documentation.md](standards/documentation.md) | Comments, READMEs, JSDoc |

---

## Security Coverage

| Topic | File | Scope |
|-------|------|-------|
| Web Security | [security/web.md](security/web.md) | OWASP, XSS, CSRF, headers |
| Authentication | [security/auth.md](security/auth.md) | JWT, sessions, OAuth |
| Secrets Management | [security/secrets.md](security/secrets.md) | Env vars, vaults, .env |

---

## File Format

Each technology file follows a consistent structure:

```
# [Technology] (2025)

## Philosophy (2025)        <- Current direction of the tech
## TL;DR                    <- Quick reference
## Best Practices           <- What to do
## Anti-Patterns            <- What NOT to do
## 2025 Changelog           <- Recent updates
## Quick Reference          <- Tables for fast lookup
```

---

## Contributing

Found outdated info? Have a better pattern? Contributions welcome.

- Open an issue for suggestions
- Submit a PR for corrections
- Keep the format consistent

---

## License

MIT License. Use freely.

---

Stay current. Write clean code.
