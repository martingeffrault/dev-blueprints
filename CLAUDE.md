# Dev Blueprints - AI Context Guide

This repository contains cutting-edge development best practices, anti-patterns, and conventions for 2025-2026. It's designed to be added to any project to guide AI assistants toward modern, clean code.

---

## Purpose

When this repo is added as context to an AI coding assistant (Claude, Cursor, Copilot), it should:

1. **Follow current best practices** — Not outdated patterns from tutorials
2. **Avoid anti-patterns** — Know what NOT to do
3. **Use cutting-edge features** — Leverage 2025 updates
4. **Apply consistent conventions** — Naming, structure, style

---

## Repository Structure

```
dev-blueprints/
├── CLAUDE.md              # This file - AI instructions
├── README.md              # Public documentation
├── stack/                 # Per-technology guides
│   ├── react.md           # React 19+ best practices
│   ├── nextjs.md          # Next.js 15+ App Router
│   ├── typescript.md      # TypeScript 5.x strict mode
│   └── ...
├── standards/             # Cross-stack conventions
│   ├── naming.md          # Naming conventions
│   ├── git.md             # Git workflow
│   └── ...
└── security/              # Security patterns
    ├── web.md             # OWASP, XSS, CSRF
    └── ...
```

---

## How to Use This Context

### When coding with a specific technology

Reference the relevant stack file:
```
"Follow the patterns in dev-blueprints/stack/react.md"
"Check dev-blueprints/stack/nextjs.md for App Router patterns"
```

### When setting up a project

Reference standards:
```
"Apply naming conventions from dev-blueprints/standards/naming.md"
"Use git workflow from dev-blueprints/standards/git.md"
```

### When reviewing code

Check anti-patterns:
```
"Verify this doesn't use anti-patterns listed in dev-blueprints/stack/react.md"
```

---

## File Format

Each stack file follows this structure:

```markdown
# [Technology] (2025)

> Last updated: [date]
> Versions: [covered versions]

## Philosophy (2025)
Current direction and mindset of the technology.

## TL;DR
Quick reference of the most important rules.

## Best Practices
Detailed patterns to follow.

## Anti-Patterns
What to AVOID with explanations.

## 2025 Changelog
Recent updates, new features, breaking changes.

## Quick Reference
Tables for fast lookup.
```

---

## Maintenance Rules

1. **Date everything** — Each file has "Last updated" date
2. **Version specificity** — State which versions are covered
3. **Changelog section** — Track 2025-2026 updates
4. **Anti-patterns are critical** — Always explain WHY something is bad
5. **Code examples** — Show DO and DON'T side by side
6. **Sources** — Link to official docs when possible

---

## When to Update

- New major version released → Update philosophy + changelog
- New pattern emerges → Add to best practices
- Pattern becomes outdated → Move to anti-patterns
- Security issue discovered → Update security/ files

---

## Priority Order for AI

When patterns conflict, follow this priority:

1. **Security** — Never compromise
2. **Type safety** — TypeScript strict mode
3. **Performance** — Core Web Vitals
4. **Maintainability** — Clean code principles
5. **DX** — Developer experience
