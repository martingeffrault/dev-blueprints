# Zustand (2025)

> **Last updated**: January 2026
> **Versions covered**: 4.x, 5.x
> **Purpose**: Lightweight state management for React

---

## Philosophy (2025-2026)

Zustand is the **simplest way to manage global state** in React, offering a minimal API with maximum flexibility.

**Key philosophical shifts:**
- **No providers** — Direct store access via hooks
- **No boilerplate** — Create store in 5 lines
- **Selectors always** — Prevent unnecessary re-renders
- **Slices pattern** — Split large stores into modules
- **Middleware** — persist, devtools, immer built-in
- **Client-only with RSC** — Don't use Zustand in Server Components

---

## TL;DR

- Always use selectors: `useStore((state) => state.value)`
- Separate actions from state in store structure
- Create custom hooks for each selector
- Use slices pattern for large stores
- Use `persist` middleware for localStorage
- Use `devtools` middleware for debugging
- Don't use Zustand with React Server Components

---

## Best Practices

### Basic Store

```typescript
// stores/counter.ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));
```

### Using Selectors (Always!)

```typescript
// ❌ DON'T — Subscribes to entire store
function Counter() {
  const store = useCounterStore(); // Re-renders on ANY change
  return <div>{store.count}</div>;
}

// ✅ DO — Use selectors
function Counter() {
  const count = useCounterStore((state) => state.count);
  return <div>{count}</div>;
}

// ✅ DO — Selector for actions (stable reference)
function Controls() {
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);
  return (
    <>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </>
  );
}
```

### Custom Hooks Pattern

```typescript
// stores/user.ts
import { create } from 'zustand';

interface UserState {
  user: User | null;
  isAuthenticated: boolean;
  setUser: (user: User | null) => void;
  logout: () => void;
}

const useUserStore = create<UserState>((set) => ({
  user: null,
  isAuthenticated: false,
  setUser: (user) => set({ user, isAuthenticated: !!user }),
  logout: () => set({ user: null, isAuthenticated: false }),
}));

// Custom hooks for clean API
export const useUser = () => useUserStore((state) => state.user);
export const useIsAuthenticated = () => useUserStore((state) => state.isAuthenticated);
export const useUserActions = () => useUserStore((state) => ({
  setUser: state.setUser,
  logout: state.logout,
}));
```

### Separating State from Actions

```typescript
// stores/cart.ts
import { create } from 'zustand';

interface CartItem {
  id: string;
  name: string;
  quantity: number;
  price: number;
}

interface CartState {
  items: CartItem[];
  total: number;
}

interface CartActions {
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
}

const useCartStore = create<CartState & CartActions>((set, get) => ({
  // State
  items: [],
  total: 0,

  // Actions
  addItem: (item) => set((state) => {
    const existing = state.items.find((i) => i.id === item.id);
    const items = existing
      ? state.items.map((i) =>
          i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
        )
      : [...state.items, { ...item, quantity: 1 }];
    return {
      items,
      total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
    };
  }),

  removeItem: (id) => set((state) => {
    const items = state.items.filter((i) => i.id !== id);
    return {
      items,
      total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
    };
  }),

  updateQuantity: (id, quantity) => set((state) => {
    const items = state.items.map((i) =>
      i.id === id ? { ...i, quantity } : i
    );
    return {
      items,
      total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
    };
  }),

  clearCart: () => set({ items: [], total: 0 }),
}));

// Selectors
export const useCartItems = () => useCartStore((state) => state.items);
export const useCartTotal = () => useCartStore((state) => state.total);
export const useCartActions = () => useCartStore((state) => ({
  addItem: state.addItem,
  removeItem: state.removeItem,
  updateQuantity: state.updateQuantity,
  clearCart: state.clearCart,
}));
```

### Persist Middleware

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface SettingsState {
  theme: 'light' | 'dark';
  language: string;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
}

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: 'settings-storage', // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
      }), // Only persist these fields
    }
  )
);
```

### DevTools Middleware

```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export const useStore = create<StoreState>()(
  devtools(
    (set) => ({
      // ... state and actions
    }),
    {
      name: 'MyStore', // Name in Redux DevTools
      enabled: process.env.NODE_ENV === 'development',
    }
  )
);
```

### Combining Middleware

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

export const useStore = create<StoreState>()(
  devtools(
    persist(
      immer((set) => ({
        items: [],
        addItem: (item) => set((state) => {
          state.items.push(item); // Immer allows mutations
        }),
      })),
      { name: 'store' }
    ),
    { name: 'Store' }
  )
);
```

### Slices Pattern (Large Stores)

```typescript
// stores/slices/userSlice.ts
import { StateCreator } from 'zustand';

export interface UserSlice {
  user: User | null;
  setUser: (user: User | null) => void;
}

export const createUserSlice: StateCreator<UserSlice> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
});

// stores/slices/cartSlice.ts
export interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
}

export const createCartSlice: StateCreator<CartSlice> = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
});

// stores/index.ts
import { create } from 'zustand';
import { createUserSlice, UserSlice } from './slices/userSlice';
import { createCartSlice, CartSlice } from './slices/cartSlice';

type StoreState = UserSlice & CartSlice;

export const useStore = create<StoreState>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}));
```

---

## Anti-Patterns

### ❌ Subscribing to Entire Store

**Why it's bad**: Component re-renders on any state change.

```typescript
// ❌ DON'T
const { count, user, cart } = useStore();
// Re-renders when ANY of these change

// ✅ DO
const count = useStore((state) => state.count);
// Only re-renders when count changes
```

### ❌ Not Using Stable Action References

**Why it's bad**: Creates new function references.

```typescript
// ❌ DON'T
function Component() {
  const increment = () => useStore.getState().increment();
}

// ✅ DO
function Component() {
  const increment = useStore((state) => state.increment);
}
```

### ❌ Using Zustand with Server Components

**Why it's bad**: Server Components don't have client-side state.

```typescript
// ❌ DON'T — In Server Component
async function ServerComponent() {
  const count = useCounterStore((s) => s.count); // Error!
}

// ✅ DO — Client Components only
'use client';
function ClientComponent() {
  const count = useCounterStore((s) => s.count);
}
```

### ❌ Mega-Store Anti-Pattern

**Why it's bad**: Tight coupling, hard to test.

```typescript
// ❌ DON'T — One massive store
const useStore = create({
  user: null,
  cart: [],
  settings: {},
  notifications: [],
  modal: null,
  // ... 50 more properties
});

// ✅ DO — Multiple focused stores
const useUserStore = create({ ... });
const useCartStore = create({ ... });
const useSettingsStore = create({ ... });
const useUIStore = create({ ... });
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Create store | `create<State>((set) => ({ ... }))` |
| Read state | `useStore((state) => state.value)` |
| Update state | `set({ value: newValue })` |
| Update based on prev | `set((state) => ({ count: state.count + 1 }))` |
| Get state outside React | `useStore.getState()` |
| Set state outside React | `useStore.setState({ value })` |
| Subscribe outside React | `useStore.subscribe((state) => ...)` |
| Persist | `persist(fn, { name: 'key' })` |
| DevTools | `devtools(fn, { name: 'Store' })` |
| Immer | `immer(fn)` |

---

## When to Use Zustand vs Alternatives

| Use Case | Solution |
|----------|----------|
| Simple global state | Zustand |
| Complex global state | Zustand or Redux Toolkit |
| Server data | TanStack Query (not Zustand) |
| Form state | React Hook Form (not Zustand) |
| URL state | URL params (not Zustand) |
| Local component state | useState (not Zustand) |
| Simple shared state | React Context |
| Atomic state | Jotai |
| State machines | XState |

---

## Resources

- [Official Zustand Documentation](https://github.com/pmndrs/zustand)
- [TkDodo - Working with Zustand](https://tkdodo.eu/blog/working-with-zustand)
- [React State Management 2025](https://www.developerway.com/posts/react-state-management-2025)
- [Zustand vs Redux](https://makersden.io/blog/react-state-management-in-2025)
