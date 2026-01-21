# Vitest (2025)

> **Last updated**: January 2026
> **Versions covered**: 2.x
> **Integration**: Native Vite compatibility

---

## Philosophy (2025-2026)

Vitest is **THE** testing framework for Vite projects, offering Jest-compatible API with native ESM support and blazing-fast performance.

**Key philosophical shifts:**
- **Vite-native** — Uses Vite's transform pipeline (instant HMR for tests)
- **Jest-compatible** — Easy migration, familiar API
- **TypeScript-first** — No configuration needed
- **Browser mode** — Real browser testing without Puppeteer
- **Workspaces** — First-class monorepo support
- **Vite+ integration** — Part of unified toolchain (2025)

---

## TL;DR

- Vitest shares config with `vite.config.ts` by default
- Use `describe` blocks to organize tests
- Use `vi.mock()` for mocking, `vi.fn()` for spy functions
- Run `vitest` for watch mode, `vitest run` for CI
- Use `--pool=threads` for better performance in large projects
- Use `--no-isolate` for faster tests (if no side effects)
- Browser mode available for component testing

---

## Best Practices

### Basic Configuration

```typescript
// vitest.config.ts (optional - inherits from vite.config.ts)
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,              // Use describe/it/expect without imports
    environment: 'jsdom',       // For React/DOM testing
    setupFiles: ['./tests/setup.ts'],
    include: ['**/*.{test,spec}.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules', 'tests'],
    },
  },
});
```

### Test Setup File

```typescript
// tests/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';

// Cleanup after each test
afterEach(() => {
  cleanup();
});

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation((query) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});
```

### Unit Testing

```typescript
// lib/utils.test.ts
import { describe, it, expect } from 'vitest';
import { formatPrice, validateEmail } from './utils';

describe('formatPrice', () => {
  it('formats USD correctly', () => {
    expect(formatPrice(1000, 'USD')).toBe('$1,000.00');
  });

  it('handles zero', () => {
    expect(formatPrice(0, 'USD')).toBe('$0.00');
  });

  it('handles negative values', () => {
    expect(formatPrice(-50, 'USD')).toBe('-$50.00');
  });
});

describe('validateEmail', () => {
  it.each([
    ['valid@email.com', true],
    ['invalid', false],
    ['no@tld', false],
    ['valid+tag@email.com', true],
  ])('validates %s as %s', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});
```

### Component Testing (React)

```typescript
// components/Button.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders children', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button')).toHaveTextContent('Click me');
  });

  it('calls onClick when clicked', () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click</Button>);

    fireEvent.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when loading', () => {
    render(<Button loading>Submit</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

### Mocking

```typescript
// services/api.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { fetchUser } from './api';

// Mock fetch globally
const mockFetch = vi.fn();
vi.stubGlobal('fetch', mockFetch);

describe('fetchUser', () => {
  beforeEach(() => {
    mockFetch.mockReset();
  });

  it('returns user data', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ id: '1', name: 'Alice' }),
    });

    const user = await fetchUser('1');

    expect(user).toEqual({ id: '1', name: 'Alice' });
    expect(mockFetch).toHaveBeenCalledWith('/api/users/1');
  });

  it('throws on error', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: false,
      status: 404,
    });

    await expect(fetchUser('999')).rejects.toThrow('User not found');
  });
});
```

### Module Mocking

```typescript
// hooks/useAuth.test.ts
import { describe, it, expect, vi } from 'vitest';
import { renderHook } from '@testing-library/react';
import { useAuth } from './useAuth';

// Mock an entire module
vi.mock('@/lib/supabase', () => ({
  supabase: {
    auth: {
      getUser: vi.fn().mockResolvedValue({
        data: { user: { id: '1', email: 'test@example.com' } },
        error: null,
      }),
    },
  },
}));

describe('useAuth', () => {
  it('returns current user', async () => {
    const { result } = renderHook(() => useAuth());

    await vi.waitFor(() => {
      expect(result.current.user).toEqual({
        id: '1',
        email: 'test@example.com',
      });
    });
  });
});
```

### Snapshot Testing

```typescript
// components/Card.test.tsx
import { describe, it, expect } from 'vitest';
import { render } from '@testing-library/react';
import { Card } from './Card';

describe('Card', () => {
  it('matches snapshot', () => {
    const { container } = render(
      <Card title="Test" description="Description" />
    );
    expect(container).toMatchSnapshot();
  });
});
```

---

## Anti-Patterns

### ❌ Testing Implementation Details

**Why it's bad**: Tests break when implementation changes, even if behavior is correct.

```typescript
// ❌ DON'T — Testing internal state
it('sets loading state', () => {
  const { result } = renderHook(() => useData());
  expect(result.current.internalLoadingState).toBe(true);
});

// ✅ DO — Test behavior
it('shows loading indicator while fetching', () => {
  render(<DataComponent />);
  expect(screen.getByRole('progressbar')).toBeVisible();
});
```

### ❌ Not Cleaning Up Between Tests

**Why it's bad**: State leaks between tests cause flaky results.

```typescript
// ❌ DON'T — Shared mutable state
let users = [];

it('adds user', () => {
  users.push({ id: '1' });
  expect(users).toHaveLength(1);
});

it('has no users', () => {
  expect(users).toHaveLength(0); // Fails!
});

// ✅ DO — Fresh state per test
describe('users', () => {
  let users: User[];

  beforeEach(() => {
    users = [];
  });

  it('adds user', () => {
    users.push({ id: '1' });
    expect(users).toHaveLength(1);
  });

  it('has no users', () => {
    expect(users).toHaveLength(0); // Passes
  });
});
```

### ❌ Using Arbitrary Timeouts

**Why it's bad**: Slow, flaky tests.

```typescript
// ❌ DON'T — Arbitrary timeout
it('loads data', async () => {
  render(<DataComponent />);
  await new Promise(r => setTimeout(r, 1000)); // Bad!
  expect(screen.getByText('Data')).toBeVisible();
});

// ✅ DO — Wait for elements
it('loads data', async () => {
  render(<DataComponent />);
  await screen.findByText('Data'); // Auto-waits
});
```

### ❌ Not Using `vi.fn()` for Callbacks

**Why it's bad**: Can't verify calls or arguments.

```typescript
// ❌ DON'T
let called = false;
const onClick = () => { called = true; };

// ✅ DO
const onClick = vi.fn();
// Can now use:
expect(onClick).toHaveBeenCalled();
expect(onClick).toHaveBeenCalledWith('arg');
expect(onClick).toHaveBeenCalledTimes(1);
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.0 | 2024 | Stable browser mode, improved workspaces |
| 2.x | 2025 | Performance improvements, better error messages |
| Vite+ | Oct 2025 | Vitest becomes part of unified Vite+ toolchain |

---

## Quick Reference

| Task | Command |
|------|---------|
| Run in watch mode | `vitest` |
| Single run (CI) | `vitest run` |
| With coverage | `vitest --coverage` |
| Run specific file | `vitest path/to/file.test.ts` |
| Run matching pattern | `vitest -t "pattern"` |
| Update snapshots | `vitest -u` |
| UI mode | `vitest --ui` |

| Mock Function | Usage |
|---------------|-------|
| Spy function | `vi.fn()` |
| Mock module | `vi.mock('module')` |
| Mock global | `vi.stubGlobal('name', value)` |
| Fake timers | `vi.useFakeTimers()` |
| Restore mocks | `vi.restoreAllMocks()` |

---

## Performance Tips

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    // Use threads pool for large projects
    pool: 'threads',

    // Disable isolation if tests don't have side effects
    isolate: false,

    // Shard tests across CI machines
    // Run with: vitest --shard=1/3
  },
});
```

---

## Resources

- [Official Vitest Documentation](https://vitest.dev/)
- [Vitest Configuration](https://vitest.dev/config/)
- [Testing Library](https://testing-library.com/)
- [Vitest Best Practices](https://www.projectrules.ai/rules/vitest)
- [Component Testing Guide](https://vitest.dev/guide/browser/component-testing)
