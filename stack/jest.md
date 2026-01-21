# Jest (2025-2026)

> **Last updated**: January 2026
> **Versions covered**: 29.x, 30.x
> **Integration**: React, TypeScript 5.4+, Node.js 18+

---

## Philosophy (2025-2026)

Jest remains the **most widely adopted** JavaScript testing framework, especially for React applications, React Native, and legacy projects. With Jest 30 (released June 2025), the framework delivers significant performance improvements, better ESM support, and modern TypeScript integration.

**Key philosophical shifts:**
- **Performance-first** — 37% faster tests, 77% lower memory in large codebases
- **Modern JavaScript** — Native `.mts` and `.cts` support, improved ESM handling
- **TypeScript 5.4+** — First-class TypeScript support with stricter requirements
- **User-centric testing** — Pair with React Testing Library for behavior-focused tests
- **Vitest alternative** — For Vite projects, consider Vitest for better integration

---

## TL;DR

- Use `jest.config.ts` with TypeScript for type-safe configuration
- Follow AAA pattern: Arrange, Act, Assert
- Use `jest.fn()` for mocks, `jest.spyOn()` for tracking without replacing
- Prefer `@testing-library/react` with user-centric queries (getByRole, getByText)
- Use `async/await` with proper assertions for async tests
- Keep snapshots small; use inline snapshots when possible
- Set coverage thresholds in CI (aim for 80%+ on critical paths)
- Restore mocks with `jest.restoreAllMocks()` in `afterEach`
- Use `userEvent` over `fireEvent` for realistic user interactions

---

## Best Practices

### Configuration (jest.config.ts)

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  // Use ts-jest for TypeScript support
  preset: 'ts-jest',

  // Use jsdom for React/DOM testing, 'node' for backend
  testEnvironment: 'jsdom',

  // Setup files run before each test file
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],

  // Test file patterns
  testMatch: [
    '**/__tests__/**/*.{ts,tsx}',
    '**/*.{spec,test}.{ts,tsx}',
  ],

  // Module path aliases (match tsconfig paths)
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    // Handle CSS/asset imports
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy',
    '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },

  // Coverage configuration
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/types/**',
    '!src/**/*.stories.{ts,tsx}',
  ],

  // Coverage thresholds - enforce minimum coverage
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Higher thresholds for critical paths
    './src/utils/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
  },

  // Transform configuration
  transform: {
    '^.+\\.tsx?$': ['ts-jest', {
      tsconfig: 'tsconfig.json',
    }],
  },

  // Clear mocks between tests
  clearMocks: true,

  // Restore mocks between tests
  restoreMocks: true,
};

export default config;
```

### Setup File

```typescript
// jest.setup.ts
import '@testing-library/jest-dom';

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// Mock IntersectionObserver
class MockIntersectionObserver {
  observe = jest.fn();
  unobserve = jest.fn();
  disconnect = jest.fn();
}

Object.defineProperty(window, 'IntersectionObserver', {
  writable: true,
  value: MockIntersectionObserver,
});

// Global cleanup
afterEach(() => {
  jest.clearAllMocks();
});
```

### Test Structure and Naming

```typescript
// components/Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

// Group related tests with describe
describe('Button', () => {
  // Use descriptive test names that explain expected behavior
  it('renders children text correctly', () => {
    render(<Button>Click me</Button>);

    expect(screen.getByRole('button')).toHaveTextContent('Click me');
  });

  it('calls onClick handler when clicked', async () => {
    const user = userEvent.setup();
    const handleClick = jest.fn();

    render(<Button onClick={handleClick}>Click</Button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when loading prop is true', () => {
    render(<Button loading>Submit</Button>);

    expect(screen.getByRole('button')).toBeDisabled();
  });

  // Group edge cases
  describe('edge cases', () => {
    it('prevents click when disabled', async () => {
      const user = userEvent.setup();
      const handleClick = jest.fn();

      render(<Button onClick={handleClick} disabled>Click</Button>);

      await user.click(screen.getByRole('button'));

      expect(handleClick).not.toHaveBeenCalled();
    });
  });
});
```

### Parameterized Tests

```typescript
// utils/validation.test.ts
import { validateEmail, validatePassword } from './validation';

describe('validateEmail', () => {
  it.each([
    ['valid@email.com', true],
    ['user+tag@domain.co.uk', true],
    ['invalid', false],
    ['no@tld', false],
    ['@nodomain.com', false],
    ['spaces in@email.com', false],
  ])('validates "%s" as %s', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});

describe('validatePassword', () => {
  it.each`
    password          | valid    | reason
    ${'Abc123!@#'}    | ${true}  | ${'meets all requirements'}
    ${'short'}        | ${false} | ${'too short'}
    ${'nouppercase1!'} | ${false} | ${'missing uppercase'}
    ${'NOLOWERCASE1!'} | ${false} | ${'missing lowercase'}
  `('returns $valid for "$password" ($reason)', ({ password, valid }) => {
    expect(validatePassword(password).isValid).toBe(valid);
  });
});
```

### Mocking Patterns

#### Function Mocking with jest.fn()

```typescript
// services/userService.test.ts
import { createUser, notifyUser } from './userService';

describe('createUser', () => {
  it('notifies user after creation', async () => {
    const mockNotify = jest.fn().mockResolvedValue({ sent: true });

    await createUser({ name: 'Alice', email: 'alice@example.com' }, mockNotify);

    expect(mockNotify).toHaveBeenCalledWith({
      email: 'alice@example.com',
      message: expect.stringContaining('Welcome'),
    });
    expect(mockNotify).toHaveBeenCalledTimes(1);
  });
});
```

#### Module Mocking with jest.mock()

```typescript
// components/UserProfile.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import { fetchUser } from '@/api/users';

// Mock the entire module
jest.mock('@/api/users');

// Type the mock for TypeScript
const mockFetchUser = fetchUser as jest.MockedFunction<typeof fetchUser>;

describe('UserProfile', () => {
  beforeEach(() => {
    mockFetchUser.mockReset();
  });

  it('displays user data after loading', async () => {
    mockFetchUser.mockResolvedValue({
      id: '1',
      name: 'Alice Johnson',
      email: 'alice@example.com',
    });

    render(<UserProfile userId="1" />);

    // Loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();

    // Wait for data
    await waitFor(() => {
      expect(screen.getByText('Alice Johnson')).toBeInTheDocument();
    });

    expect(mockFetchUser).toHaveBeenCalledWith('1');
  });

  it('displays error message on failure', async () => {
    mockFetchUser.mockRejectedValue(new Error('Network error'));

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

#### Spying with jest.spyOn()

```typescript
// utils/analytics.test.ts
import * as analytics from './analytics';

describe('analytics', () => {
  afterEach(() => {
    jest.restoreAllMocks();
  });

  it('tracks page view with correct data', () => {
    // Spy on the method without replacing it
    const trackSpy = jest.spyOn(analytics, 'track');

    analytics.trackPageView('/home');

    expect(trackSpy).toHaveBeenCalledWith('page_view', {
      path: '/home',
      timestamp: expect.any(Number),
    });
  });

  it('can mock implementation when needed', () => {
    const trackSpy = jest.spyOn(analytics, 'track')
      .mockImplementation(() => {}); // Prevent actual tracking

    analytics.trackPageView('/dashboard');

    expect(trackSpy).toHaveBeenCalled();
  });
});
```

#### Partial Module Mocking

```typescript
// Only mock specific exports
jest.mock('@/lib/api', () => ({
  ...jest.requireActual('@/lib/api'),
  fetchData: jest.fn(), // Only mock fetchData
}));
```

### Testing React Components

```typescript
// components/TodoList.test.tsx
import { render, screen, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TodoList } from './TodoList';

describe('TodoList', () => {
  const defaultTodos = [
    { id: '1', text: 'Learn Jest', completed: false },
    { id: '2', text: 'Write tests', completed: true },
  ];

  it('renders all todo items', () => {
    render(<TodoList todos={defaultTodos} />);

    expect(screen.getByText('Learn Jest')).toBeInTheDocument();
    expect(screen.getByText('Write tests')).toBeInTheDocument();
  });

  it('shows completed items with strikethrough', () => {
    render(<TodoList todos={defaultTodos} />);

    const completedItem = screen.getByText('Write tests');
    expect(completedItem).toHaveClass('line-through');
  });

  it('adds new todo when form is submitted', async () => {
    const user = userEvent.setup();
    const onAdd = jest.fn();

    render(<TodoList todos={defaultTodos} onAdd={onAdd} />);

    const input = screen.getByPlaceholderText(/add todo/i);
    await user.type(input, 'New task');
    await user.click(screen.getByRole('button', { name: /add/i }));

    expect(onAdd).toHaveBeenCalledWith('New task');
  });

  it('filters todos by completion status', async () => {
    const user = userEvent.setup();
    render(<TodoList todos={defaultTodos} />);

    // Click filter button
    await user.click(screen.getByRole('button', { name: /completed/i }));

    // Only completed items visible
    expect(screen.queryByText('Learn Jest')).not.toBeInTheDocument();
    expect(screen.getByText('Write tests')).toBeInTheDocument();
  });
});
```

#### Testing Hooks

```typescript
// hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter());

    expect(result.current.count).toBe(0);
  });

  it('initializes with custom value', () => {
    const { result } = renderHook(() => useCounter(10));

    expect(result.current.count).toBe(10);
  });

  it('increments count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('decrements count', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });
});
```

### Testing Async Code

```typescript
// services/api.test.ts
import { fetchUserById, createUser } from './api';

describe('API Service', () => {
  describe('fetchUserById', () => {
    it('returns user data on success', async () => {
      const user = await fetchUserById('123');

      expect(user).toEqual({
        id: '123',
        name: expect.any(String),
        email: expect.stringContaining('@'),
      });
    });

    it('throws error for non-existent user', async () => {
      await expect(fetchUserById('invalid')).rejects.toThrow('User not found');
    });

    // Using resolves/rejects matchers
    it('resolves with user data', async () => {
      await expect(fetchUserById('123')).resolves.toHaveProperty('id', '123');
    });
  });

  describe('createUser', () => {
    it('creates user and returns with id', async () => {
      const newUser = await createUser({
        name: 'Bob',
        email: 'bob@example.com',
      });

      expect(newUser.id).toBeDefined();
      expect(newUser.name).toBe('Bob');
    });
  });
});
```

#### Testing with waitFor

```typescript
// components/AsyncComponent.test.tsx
import { render, screen, waitFor, waitForElementToBeRemoved } from '@testing-library/react';
import { DataLoader } from './DataLoader';

describe('DataLoader', () => {
  it('shows loading then data', async () => {
    render(<DataLoader />);

    // Verify loading state
    expect(screen.getByRole('progressbar')).toBeInTheDocument();

    // Wait for loading to disappear
    await waitForElementToBeRemoved(() => screen.queryByRole('progressbar'));

    // Data should be visible
    expect(screen.getByText(/loaded data/i)).toBeInTheDocument();
  });

  it('waits for specific condition', async () => {
    render(<DataLoader />);

    await waitFor(() => {
      expect(screen.getByTestId('data-count')).toHaveTextContent('10');
    }, {
      timeout: 3000,
      interval: 100,
    });
  });
});
```

#### Testing with Fake Timers

```typescript
// utils/debounce.test.ts
import { debounce } from './debounce';

describe('debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('delays function execution', () => {
    const fn = jest.fn();
    const debouncedFn = debounce(fn, 500);

    debouncedFn();
    expect(fn).not.toHaveBeenCalled();

    jest.advanceTimersByTime(499);
    expect(fn).not.toHaveBeenCalled();

    jest.advanceTimersByTime(1);
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it('resets timer on subsequent calls', () => {
    const fn = jest.fn();
    const debouncedFn = debounce(fn, 500);

    debouncedFn();
    jest.advanceTimersByTime(400);

    debouncedFn(); // Reset timer
    jest.advanceTimersByTime(400);
    expect(fn).not.toHaveBeenCalled();

    jest.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledTimes(1);
  });
});
```

### Snapshot Testing

```typescript
// components/Card.test.tsx
import { render } from '@testing-library/react';
import { Card } from './Card';

describe('Card', () => {
  // Use inline snapshots for small, focused tests
  it('renders correctly', () => {
    const { container } = render(
      <Card title="Test Title" description="Test description" />
    );

    expect(container.firstChild).toMatchInlineSnapshot(`
      <div
        class="card"
      >
        <h2
          class="card-title"
        >
          Test Title
        </h2>
        <p
          class="card-description"
        >
          Test description
        </p>
      </div>
    `);
  });

  // Use property matchers for dynamic values
  it('renders with dynamic content', () => {
    const { container } = render(
      <Card
        title="Dynamic"
        createdAt={new Date()}
        id={crypto.randomUUID()}
      />
    );

    expect(container.firstChild).toMatchSnapshot({
      'data-id': expect.any(String),
      'data-created': expect.any(String),
    });
  });
});
```

### Code Coverage

```typescript
// jest.config.ts - Coverage settings
const config: Config = {
  // Enable coverage collection
  collectCoverage: true,

  // Specify which files to collect coverage from
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.stories.{ts,tsx}',
    '!src/types/**',
    '!src/index.ts',
  ],

  // Coverage output directory
  coverageDirectory: 'coverage',

  // Coverage reporters
  coverageReporters: ['text', 'text-summary', 'lcov', 'html'],

  // Enforce thresholds
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Critical business logic requires higher coverage
    './src/services/payment/': {
      branches: 95,
      functions: 95,
      lines: 95,
      statements: 95,
    },
  },
};
```

---

## Anti-Patterns

### Test Implementation Details

**Why it's bad**: Tests break when implementation changes, even if behavior is correct.

```typescript
// DON'T - Testing internal state
it('sets loading state', () => {
  const { result } = renderHook(() => useData());
  expect(result.current._internalLoadingState).toBe(true);
});

// DO - Test observable behavior
it('shows loading indicator while fetching', () => {
  render(<DataComponent />);
  expect(screen.getByRole('progressbar')).toBeInTheDocument();
});
```

### Using fireEvent Instead of userEvent

**Why it's bad**: `fireEvent` dispatches DOM events, while `userEvent` simulates real user interactions.

```typescript
// DON'T - fireEvent doesn't simulate real user behavior
import { fireEvent } from '@testing-library/react';

it('handles input', () => {
  render(<Form />);
  fireEvent.change(screen.getByRole('textbox'), { target: { value: 'test' } });
});

// DO - userEvent simulates actual user typing
import userEvent from '@testing-library/user-event';

it('handles input', async () => {
  const user = userEvent.setup();
  render(<Form />);
  await user.type(screen.getByRole('textbox'), 'test');
});
```

### Arbitrary Timeouts

**Why it's bad**: Slow, flaky tests that fail intermittently.

```typescript
// DON'T - Arbitrary timeout
it('loads data', async () => {
  render(<DataComponent />);
  await new Promise(r => setTimeout(r, 2000)); // Bad!
  expect(screen.getByText('Data')).toBeVisible();
});

// DO - Wait for elements
it('loads data', async () => {
  render(<DataComponent />);
  expect(await screen.findByText('Data')).toBeVisible();
});

// DO - Use waitFor for conditions
it('loads data', async () => {
  render(<DataComponent />);
  await waitFor(() => {
    expect(screen.getByText('Data')).toBeVisible();
  });
});
```

### Not Cleaning Up Mocks

**Why it's bad**: State leaks between tests cause flaky results.

```typescript
// DON'T - Mocks persist between tests
const mockFetch = jest.fn();
jest.spyOn(global, 'fetch').mockImplementation(mockFetch);

it('first test', async () => {
  mockFetch.mockResolvedValue({ json: () => ({ data: 'test' }) });
  // ...
});

it('second test', async () => {
  // mockFetch still has previous mock implementation!
});

// DO - Clean up in afterEach or use restoreMocks config
afterEach(() => {
  jest.restoreAllMocks();
});

// Or in jest.config.ts
const config = {
  restoreMocks: true,
  clearMocks: true,
};
```

### Large Snapshots

**Why it's bad**: Hard to review, easy to approve without checking, break frequently.

```typescript
// DON'T - Snapshot entire page
it('renders page', () => {
  const { container } = render(<EntirePage />);
  expect(container).toMatchSnapshot(); // 500+ lines!
});

// DO - Snapshot specific, focused elements
it('renders header correctly', () => {
  render(<EntirePage />);
  expect(screen.getByRole('banner')).toMatchInlineSnapshot(`
    <header class="header">
      <h1>Title</h1>
    </header>
  `);
});
```

### Testing Third-Party Libraries

**Why it's bad**: You're testing their code, not yours.

```typescript
// DON'T - Testing axios works
it('axios makes requests', async () => {
  const response = await axios.get('/api/users');
  expect(response.status).toBe(200);
});

// DO - Test YOUR code that uses axios
jest.mock('axios');
const mockAxios = axios as jest.Mocked<typeof axios>;

it('fetchUsers returns formatted data', async () => {
  mockAxios.get.mockResolvedValue({ data: [{ id: 1, name: 'Alice' }] });

  const users = await fetchUsers();

  expect(users).toEqual([{ id: 1, name: 'Alice', displayName: 'Alice' }]);
});
```

### Not Asserting Async Operations

**Why it's bad**: Tests pass without actually verifying anything.

```typescript
// DON'T - Missing await/return causes false positives
it('fetches data', () => {
  fetchData().then(data => {
    expect(data).toBe('test'); // Never runs!
  });
});

// DO - Always await or return promises
it('fetches data', async () => {
  const data = await fetchData();
  expect(data).toBe('test');
});

// DO - Use expect.assertions for extra safety
it('fetches data', async () => {
  expect.assertions(1);
  const data = await fetchData();
  expect(data).toBe('test');
});
```

### Querying by Test IDs When Better Options Exist

**Why it's bad**: Doesn't test accessibility, users don't see test IDs.

```typescript
// DON'T - Test ID when role available
render(<button data-testid="submit-btn">Submit</button>);
screen.getByTestId('submit-btn');

// DO - Use accessible queries
screen.getByRole('button', { name: /submit/i });

// DO - Test ID only when no better option
// (e.g., for a generic container with no semantic meaning)
screen.getByTestId('chart-container');
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 29.0 | Aug 2022 | Node 14.15+ required, jsdom 20, new snapshot format |
| 29.5 | Apr 2023 | `jest.mocked()` improvements, better TypeScript support |
| 29.7 | Sep 2023 | Performance improvements, bug fixes |
| 30.0 | Jun 2025 | **Major release**: Node 18+ required, TypeScript 5.4+, 37% faster, 77% lower memory |
| 30.x | 2025-2026 | `expect.arrayOf`, `advanceTimersToNextFrame()`, Error cause in snapshots |

### Jest 30 Highlights (June 2025)

**Performance Improvements:**
- 37% faster test execution in large codebases
- 77% lower memory usage
- New `unrs-resolver` for faster module resolution

**Breaking Changes:**
- Node 14, 16, 19, 21 support dropped (Node 18+ required)
- TypeScript 5.4+ required
- jsdom upgraded from 21 to 26
- `jest.genMockFromModule()` removed (use `jest.createMockFromModule()`)
- `--testPathPattern` renamed to `--testPathPatterns`

**New Features:**
```typescript
// New expect.arrayOf matcher
expect([1, 2, 3]).toEqual(expect.arrayOf(expect.any(Number)));

// Advance timers to next animation frame
jest.advanceTimersToNextFrame();

// Error cause now included in snapshots
const error = new Error('Failed', { cause: new Error('Network') });
expect(error).toMatchSnapshot(); // Includes cause

// Async timer advancement
await jest.advanceTimersByTimeAsync(1000);
```

**Migration from Jest 29:**
```typescript
// Update jest.config.ts
const config: Config = {
  // jsdom is no longer bundled - install separately
  testEnvironment: 'jest-environment-jsdom',

  // Update test path pattern flag in CLI
  // Old: --testPathPattern=utils
  // New: --testPathPatterns=utils
};
```

---

## Quick Reference

### CLI Commands

| Command | Description |
|---------|-------------|
| `jest` | Run all tests |
| `jest --watch` | Watch mode |
| `jest --coverage` | Generate coverage report |
| `jest path/to/file` | Run specific file |
| `jest -t "pattern"` | Run tests matching pattern |
| `jest --updateSnapshot` | Update snapshots |
| `jest --runInBand` | Run serially (for debugging) |
| `jest --detectOpenHandles` | Debug hanging tests |

### Mock Functions

| Method | Usage |
|--------|-------|
| `jest.fn()` | Create mock function |
| `jest.mock('module')` | Mock entire module |
| `jest.spyOn(obj, 'method')` | Spy on method |
| `jest.mocked(fn)` | TypeScript type helper |
| `mockFn.mockReturnValue(val)` | Set return value |
| `mockFn.mockResolvedValue(val)` | Set async return value |
| `mockFn.mockImplementation(fn)` | Custom implementation |
| `mockFn.mockReset()` | Reset mock state |
| `jest.restoreAllMocks()` | Restore all spies |

### Timer Mocks

| Method | Usage |
|--------|-------|
| `jest.useFakeTimers()` | Enable fake timers |
| `jest.useRealTimers()` | Restore real timers |
| `jest.advanceTimersByTime(ms)` | Advance time |
| `jest.advanceTimersByTimeAsync(ms)` | Advance time (async) |
| `jest.advanceTimersToNextFrame()` | Advance to next frame |
| `jest.runAllTimers()` | Run all timers |
| `jest.runOnlyPendingTimers()` | Run pending timers |

### Testing Library Queries

| Priority | Query | When to Use |
|----------|-------|-------------|
| 1 | `getByRole` | Interactive elements |
| 2 | `getByLabelText` | Form fields |
| 3 | `getByPlaceholderText` | Inputs without labels |
| 4 | `getByText` | Non-interactive text |
| 5 | `getByDisplayValue` | Current input value |
| 6 | `getByAltText` | Images |
| 7 | `getByTitle` | Title attribute |
| Last | `getByTestId` | No semantic alternative |

### Common Matchers

| Matcher | Usage |
|---------|-------|
| `toBe(value)` | Strict equality |
| `toEqual(value)` | Deep equality |
| `toMatchObject(obj)` | Partial object match |
| `toHaveLength(n)` | Array/string length |
| `toContain(item)` | Array contains |
| `toThrow(error)` | Function throws |
| `toHaveBeenCalled()` | Mock was called |
| `toHaveBeenCalledWith(args)` | Mock called with args |
| `toMatchSnapshot()` | Snapshot test |
| `toMatchInlineSnapshot()` | Inline snapshot |

---

## Resources

- [Official Jest Documentation](https://jestjs.io/)
- [Jest 30 Release Blog](https://jestjs.io/blog/2025/06/04/jest-30)
- [Upgrading to Jest 30](https://jestjs.io/docs/upgrading-to-jest30)
- [Testing Library Documentation](https://testing-library.com/)
- [Kent C. Dodds - Common RTL Mistakes](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [JavaScript Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)
- [Jest Mocking Best Practices - Microsoft](https://devblogs.microsoft.com/ise/jest-mocking-best-practices/)
