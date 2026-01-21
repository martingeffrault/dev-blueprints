# Project Structure (2025)

> **Last updated**: January 2025
> **Scope**: Folder organization for modern web projects

---

## Philosophy

Structure should be intuitive. New developers should find files without asking. Colocation over separation by type.

---

## Next.js App Router

```
project/
├── src/
│   ├── app/                    # Routes
│   │   ├── (auth)/            # Route groups
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── layout.tsx
│   │   │   └── loading.tsx
│   │   ├── api/               # API routes
│   │   ├── layout.tsx         # Root layout
│   │   └── page.tsx           # Home page
│   │
│   ├── components/            # Shared components
│   │   ├── ui/               # Primitive UI
│   │   │   ├── Button.tsx
│   │   │   └── Input.tsx
│   │   └── features/         # Feature components
│   │       ├── auth/
│   │       └── dashboard/
│   │
│   ├── lib/                   # Utilities
│   │   ├── utils.ts
│   │   ├── db.ts
│   │   └── auth.ts
│   │
│   ├── hooks/                 # Global hooks
│   │   └── useMediaQuery.ts
│   │
│   └── types/                 # TypeScript types
│       └── index.ts
│
├── public/                    # Static assets
├── tests/                     # E2E tests
└── package.json
```

---

## Feature-Based (Alternative)

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── actions/
│   │   └── types.ts
│   └── orders/
│       ├── components/
│       ├── hooks/
│       └── types.ts
├── shared/
│   ├── components/
│   ├── hooks/
│   └── utils/
└── app/
```

---

## Key Principles

### Colocation

Keep related files together:

```
components/
└── UserProfile/
    ├── index.ts
    ├── UserProfile.tsx
    ├── UserProfile.test.tsx
    ├── useUserProfile.ts
    └── UserProfile.module.css
```

### Barrel Exports

```typescript
// components/index.ts
export { Button } from './Button';
export { Input } from './Input';
```

### Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"]
    }
  }
}
```

---

## Quick Reference

| Folder | Contains |
|--------|----------|
| `app/` | Routes and layouts |
| `components/` | React components |
| `lib/` | Utilities and configs |
| `hooks/` | Custom React hooks |
| `types/` | TypeScript types |
| `public/` | Static files |
| `tests/` | Test files |
