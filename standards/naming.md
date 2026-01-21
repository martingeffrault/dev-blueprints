# Naming Conventions (2025)

> **Last updated**: January 2025
> **Scope**: Files, variables, functions, CSS, components

---

## Philosophy

Good naming is documentation. If you need a comment to explain what something is, the name is wrong.

---

## Files & Folders

| Type | Convention | Example |
|------|------------|---------|
| **React Components** | PascalCase | `UserProfile.tsx` |
| **Hooks** | camelCase + `use` prefix | `useAuth.ts` |
| **Utilities** | camelCase | `formatDate.ts` |
| **Constants** | camelCase or SCREAMING_SNAKE | `apiEndpoints.ts` or `API_ENDPOINTS.ts` |
| **Types/Interfaces** | PascalCase | `User.types.ts` or `types/user.ts` |
| **Test files** | Same as source + `.test` | `UserProfile.test.tsx` |
| **Folders** | kebab-case | `user-profile/` |

### Component File Structure

```
user-profile/
├── index.ts              # Re-export
├── UserProfile.tsx       # Main component
├── UserProfile.test.tsx  # Tests
├── UserProfile.styles.ts # Styles (if not Tailwind)
└── useUserProfile.ts     # Component-specific hook
```

---

## Variables

| Type | Convention | Example |
|------|------------|---------|
| **Booleans** | `is`, `has`, `can`, `should` prefix | `isLoading`, `hasError`, `canEdit` |
| **Arrays** | Plural nouns | `users`, `items`, `orderIds` |
| **Objects** | Singular nouns | `user`, `config`, `settings` |
| **Functions** | Verb + noun | `getUser`, `createOrder`, `validateEmail` |
| **Event handlers** | `handle` + event | `handleClick`, `handleSubmit` |
| **Callbacks** | `on` + event | `onClick`, `onSubmit` |

### ❌ Anti-Patterns

```typescript
// ❌ DON'T
const data = fetchUsers();        // Too generic
const flag = true;                // What flag?
const arr = [1, 2, 3];           // Meaningless
const handleIt = () => {};        // Handle what?

// ✅ DO
const users = fetchUsers();
const isAuthenticated = true;
const userIds = [1, 2, 3];
const handleFormSubmit = () => {};
```

---

## Functions

| Pattern | Convention | Example |
|---------|------------|---------|
| **Getters** | `get` prefix | `getUser()`, `getOrderTotal()` |
| **Setters** | `set` prefix | `setUser()`, `setTheme()` |
| **Creators** | `create` prefix | `createOrder()`, `createUser()` |
| **Validators** | `validate` or `is` prefix | `validateEmail()`, `isValidEmail()` |
| **Transformers** | `to` or `from` prefix | `toJSON()`, `fromDTO()` |
| **Async** | Should be obvious from context | `fetchUser()`, `loadData()` |

---

## React Components

| Element | Convention | Example |
|---------|------------|---------|
| **Components** | PascalCase | `UserProfile` |
| **Props type** | Component name + `Props` | `UserProfileProps` |
| **Context** | Name + `Context` | `AuthContext` |
| **Provider** | Name + `Provider` | `AuthProvider` |
| **Hooks** | `use` + descriptive name | `useAuth`, `useUserProfile` |

```typescript
// ✅ Pattern
interface UserProfileProps {
  userId: string;
  onEdit?: () => void;
}

export function UserProfile({ userId, onEdit }: UserProfileProps) {
  // ...
}
```

---

## CSS Classes

### Tailwind (Recommended)

Use utility classes directly. For custom components, use descriptive names:

```html
<div className="flex items-center gap-4">
  <Avatar className="size-10" />
</div>
```

### CSS Modules

```css
/* UserProfile.module.css */
.container { }
.header { }
.avatar { }
.name { }
```

### BEM (When needed)

```css
.user-profile { }
.user-profile__header { }
.user-profile__avatar { }
.user-profile--compact { }
```

---

## Constants & Enums

```typescript
// Constants object (preferred)
const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  BAD_REQUEST: 400,
} as const;

// Enum (when you need reverse mapping)
enum OrderStatus {
  Pending = 'pending',
  Confirmed = 'confirmed',
  Shipped = 'shipped',
}
```

---

## API Endpoints

| Resource | GET | POST | PUT/PATCH | DELETE |
|----------|-----|------|-----------|--------|
| Users | `/users` | `/users` | `/users/:id` | `/users/:id` |
| User orders | `/users/:id/orders` | `/users/:id/orders` | `/orders/:id` | `/orders/:id` |

---

## Database

| Element | Convention | Example |
|---------|------------|---------|
| **Tables** | snake_case, plural | `users`, `order_items` |
| **Columns** | snake_case | `created_at`, `user_id` |
| **Primary key** | `id` | `id` |
| **Foreign key** | `<table>_id` | `user_id`, `order_id` |
| **Timestamps** | `created_at`, `updated_at` | |
| **Booleans** | `is_` prefix | `is_active`, `is_deleted` |

---

## Quick Reference

| What | Convention | Example |
|------|------------|---------|
| Component | PascalCase | `UserProfile` |
| Hook | camelCase + use | `useAuth` |
| Variable (bool) | is/has/can prefix | `isLoading` |
| Handler | handle + event | `handleClick` |
| Prop callback | on + event | `onClick` |
| Folder | kebab-case | `user-profile/` |
| File (component) | PascalCase | `UserProfile.tsx` |
| File (util) | camelCase | `formatDate.ts` |
| CSS class | kebab-case or Tailwind | `user-profile` |
| DB table | snake_case plural | `order_items` |
| API route | kebab-case plural | `/api/order-items` |
