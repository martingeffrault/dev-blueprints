# Documentation (2025)

> **Last updated**: January 2025
> **Scope**: Comments, READMEs, JSDoc/TSDoc

---

## Philosophy

Code should be self-documenting. Comments explain "why", not "what". Documentation exists for things code can't express.

---

## When to Comment

### ✅ DO Comment

- Complex business logic
- Non-obvious decisions
- Workarounds for bugs/limitations
- TODO/FIXME with context
- Public API documentation

### ❌ DON'T Comment

- What the code does (it's obvious)
- Every function
- Outdated information

---

## JSDoc/TSDoc

### Functions

```typescript
/**
 * Calculates the total price including tax.
 *
 * @param items - Cart items with prices
 * @param taxRate - Tax rate as decimal (e.g., 0.2 for 20%)
 * @returns Total price including tax
 *
 * @example
 * const total = calculateTotal([{ price: 100 }], 0.2);
 * // Returns 120
 */
function calculateTotal(items: CartItem[], taxRate: number): number {
  // ...
}
```

### Components

```typescript
/**
 * User profile card with avatar and basic info.
 *
 * @example
 * <UserCard user={user} onEdit={handleEdit} />
 */
interface UserCardProps {
  /** The user to display */
  user: User;
  /** Called when edit button is clicked */
  onEdit?: () => void;
}
```

---

## README Structure

```markdown
# Project Name

Brief description.

## Getting Started

Prerequisites and installation.

## Development

How to run locally.

## Testing

How to run tests.

## Deployment

How to deploy.

## Architecture

High-level overview (link to docs).
```

---

## TODO/FIXME

```typescript
// TODO(username): Implement caching - ticket PROJ-123
// FIXME: Workaround for Safari bug, remove after v17
// HACK: Temporary fix until API is updated
```

---

## Quick Reference

| Pattern | When |
|---------|------|
| JSDoc `@param` | Function parameters |
| JSDoc `@returns` | Return value |
| JSDoc `@example` | Usage example |
| JSDoc `@deprecated` | Will be removed |
| `// TODO` | Future improvement |
| `// FIXME` | Known issue |
| `// HACK` | Temporary workaround |
