# Cypress (2025)

> **Last updated**: January 2026
> **Versions covered**: Cypress 14+, 15
> **Purpose**: E2E and component testing with real browser execution

---

## Philosophy (2025-2026)

Cypress is the **developer-friendly testing framework** — tests run in a real browser with time-travel debugging. In 2025, the focus is on **component testing** alongside E2E, and **containerized CI/CD** pipelines.

**Key principles:**
- **Test as user would** — Interact with visible elements
- **Data attributes for selection** — Resilient to CSS/JS changes
- **No arbitrary waits** — Cypress handles async automatically
- **Isolated tests** — Each test independent
- **Component + E2E** — Both types for full coverage

---

## TL;DR

- Use `data-cy` attributes for element selection
- Never use `cy.wait(ms)` — use assertions instead
- Reset state before each test (Cypress does this automatically)
- Use `cy.intercept()` to mock API responses
- Programmatic login, not UI login for every test
- Component tests for isolation, E2E for integration
- Use Page Object Model sparingly (prefer App Actions)
- Run tests in CI with Docker for consistency

---

## Best Practices

### Project Structure

```
cypress/
├── e2e/                        # E2E tests
│   ├── auth/
│   │   ├── login.cy.ts
│   │   └── register.cy.ts
│   ├── users/
│   │   ├── user-list.cy.ts
│   │   └── user-profile.cy.ts
│   └── checkout/
│       └── checkout-flow.cy.ts
├── component/                  # Component tests (co-locate with components)
├── fixtures/                   # Test data
│   ├── users.json
│   └── products.json
├── support/
│   ├── commands.ts             # Custom commands
│   ├── component.ts            # Component test setup
│   ├── e2e.ts                  # E2E test setup
│   └── index.d.ts              # Type definitions
├── downloads/
├── screenshots/
└── videos/
```

### Cypress Configuration

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    screenshotOnRunFailure: true,
    defaultCommandTimeout: 10000,
    requestTimeout: 10000,
    responseTimeout: 30000,
    retries: {
      runMode: 2,
      openMode: 0,
    },
    setupNodeEvents(on, config) {
      // implement node event listeners here
    },
  },
  component: {
    devServer: {
      framework: 'react',
      bundler: 'vite',
    },
    viewportWidth: 500,
    viewportHeight: 500,
  },
  env: {
    apiUrl: 'http://localhost:3001/api',
  },
});
```

### Element Selection with data-cy

```tsx
// Component with test attributes
export function LoginForm() {
  return (
    <form data-cy="login-form">
      <input
        data-cy="email-input"
        type="email"
        placeholder="Email"
      />
      <input
        data-cy="password-input"
        type="password"
        placeholder="Password"
      />
      <button data-cy="submit-button" type="submit">
        Login
      </button>
      <p data-cy="error-message" className="error" />
    </form>
  );
}
```

```typescript
// cypress/e2e/auth/login.cy.ts
describe('Login', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('should login with valid credentials', () => {
    // Use data-cy attributes - resilient to CSS/structure changes
    cy.get('[data-cy="email-input"]').type('user@example.com');
    cy.get('[data-cy="password-input"]').type('password123');
    cy.get('[data-cy="submit-button"]').click();

    // Assert redirect
    cy.url().should('eq', Cypress.config().baseUrl + '/dashboard');
    cy.get('[data-cy="welcome-message"]').should('contain', 'Welcome');
  });

  it('should show error for invalid credentials', () => {
    cy.get('[data-cy="email-input"]').type('wrong@example.com');
    cy.get('[data-cy="password-input"]').type('wrongpassword');
    cy.get('[data-cy="submit-button"]').click();

    cy.get('[data-cy="error-message"]')
      .should('be.visible')
      .and('contain', 'Invalid credentials');
  });
});
```

### API Mocking with cy.intercept()

```typescript
// cypress/e2e/users/user-list.cy.ts
describe('User List', () => {
  beforeEach(() => {
    // Mock API response
    cy.intercept('GET', '/api/users', {
      fixture: 'users.json',
    }).as('getUsers');

    // Login programmatically (not through UI)
    cy.loginByApi('admin@example.com', 'password123');

    cy.visit('/users');
    cy.wait('@getUsers');
  });

  it('should display list of users', () => {
    cy.get('[data-cy="user-card"]').should('have.length', 3);
  });

  it('should filter users by search term', () => {
    cy.get('[data-cy="search-input"]').type('John');

    cy.get('[data-cy="user-card"]')
      .should('have.length', 1)
      .first()
      .should('contain', 'John Doe');
  });

  it('should handle API error gracefully', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 500,
      body: { error: 'Server error' },
    }).as('getUsersError');

    cy.reload();
    cy.wait('@getUsersError');

    cy.get('[data-cy="error-message"]')
      .should('be.visible')
      .and('contain', 'Failed to load users');

    cy.get('[data-cy="retry-button"]').should('be.visible');
  });
});
```

### Custom Commands

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      loginByApi(email: string, password: string): Chainable<void>;
      loginByUI(email: string, password: string): Chainable<void>;
      getByCy(selector: string): Chainable<JQuery<HTMLElement>>;
    }
  }
}

// Programmatic login - faster, use for most tests
Cypress.Commands.add('loginByApi', (email: string, password: string) => {
  cy.request({
    method: 'POST',
    url: `${Cypress.env('apiUrl')}/auth/login`,
    body: { email, password },
  }).then((response) => {
    window.localStorage.setItem('token', response.body.token);
  });
});

// UI login - use only for testing login flow itself
Cypress.Commands.add('loginByUI', (email: string, password: string) => {
  cy.visit('/login');
  cy.get('[data-cy="email-input"]').type(email);
  cy.get('[data-cy="password-input"]').type(password);
  cy.get('[data-cy="submit-button"]').click();
  cy.url().should('not.include', '/login');
});

// Helper for data-cy selection
Cypress.Commands.add('getByCy', (selector: string) => {
  return cy.get(`[data-cy="${selector}"]`);
});

export {};
```

### Component Testing

```typescript
// src/components/Button/Button.cy.tsx
import { Button } from './Button';

describe('Button Component', () => {
  it('should render with text', () => {
    cy.mount(<Button>Click me</Button>);
    cy.get('button').should('contain', 'Click me');
  });

  it('should call onClick when clicked', () => {
    const onClick = cy.stub().as('onClick');
    cy.mount(<Button onClick={onClick}>Click me</Button>);

    cy.get('button').click();
    cy.get('@onClick').should('have.been.calledOnce');
  });

  it('should be disabled when disabled prop is true', () => {
    cy.mount(<Button disabled>Disabled</Button>);
    cy.get('button').should('be.disabled');
  });

  it('should show loading spinner when loading', () => {
    cy.mount(<Button loading>Submit</Button>);
    cy.get('[data-cy="loading-spinner"]').should('be.visible');
    cy.get('button').should('be.disabled');
  });

  it('should apply variant styles', () => {
    cy.mount(<Button variant="primary">Primary</Button>);
    cy.get('button').should('have.class', 'btn-primary');

    cy.mount(<Button variant="secondary">Secondary</Button>);
    cy.get('button').should('have.class', 'btn-secondary');
  });
});
```

```typescript
// src/components/UserCard/UserCard.cy.tsx
import { UserCard } from './UserCard';

const mockUser = {
  id: '1',
  name: 'John Doe',
  email: 'john@example.com',
  avatar: 'https://example.com/avatar.jpg',
  role: 'admin' as const,
};

describe('UserCard Component', () => {
  it('should display user information', () => {
    cy.mount(<UserCard user={mockUser} />);

    cy.get('[data-cy="user-name"]').should('contain', 'John Doe');
    cy.get('[data-cy="user-email"]').should('contain', 'john@example.com');
    cy.get('[data-cy="user-avatar"]').should('have.attr', 'src', mockUser.avatar);
  });

  it('should show admin badge for admin users', () => {
    cy.mount(<UserCard user={mockUser} />);
    cy.get('[data-cy="admin-badge"]').should('be.visible');
  });

  it('should not show admin badge for regular users', () => {
    const regularUser = { ...mockUser, role: 'user' as const };
    cy.mount(<UserCard user={regularUser} />);
    cy.get('[data-cy="admin-badge"]').should('not.exist');
  });

  it('should emit select event when clicked', () => {
    const onSelect = cy.stub().as('onSelect');
    cy.mount(<UserCard user={mockUser} onSelect={onSelect} />);

    cy.get('[data-cy="user-card"]').click();
    cy.get('@onSelect').should('have.been.calledWith', mockUser);
  });
});
```

### E2E Test Flow

```typescript
// cypress/e2e/checkout/checkout-flow.cy.ts
describe('Checkout Flow', () => {
  beforeEach(() => {
    // Setup: Login and seed cart
    cy.loginByApi('customer@example.com', 'password123');

    cy.intercept('GET', '/api/cart', {
      fixture: 'cart-with-items.json',
    }).as('getCart');

    cy.intercept('POST', '/api/checkout', {
      statusCode: 200,
      body: { orderId: 'order-123', status: 'confirmed' },
    }).as('submitOrder');
  });

  it('should complete checkout successfully', () => {
    // Step 1: View cart
    cy.visit('/cart');
    cy.wait('@getCart');
    cy.get('[data-cy="cart-item"]').should('have.length.at.least', 1);
    cy.get('[data-cy="checkout-button"]').click();

    // Step 2: Shipping info
    cy.url().should('include', '/checkout/shipping');
    cy.get('[data-cy="address-input"]').type('123 Main St');
    cy.get('[data-cy="city-input"]').type('New York');
    cy.get('[data-cy="zip-input"]').type('10001');
    cy.get('[data-cy="continue-button"]').click();

    // Step 3: Payment
    cy.url().should('include', '/checkout/payment');
    cy.get('[data-cy="card-number"]').type('4242424242424242');
    cy.get('[data-cy="card-expiry"]').type('1225');
    cy.get('[data-cy="card-cvc"]').type('123');
    cy.get('[data-cy="place-order-button"]').click();

    // Step 4: Confirmation
    cy.wait('@submitOrder');
    cy.url().should('include', '/order/confirmation');
    cy.get('[data-cy="order-number"]').should('contain', 'order-123');
    cy.get('[data-cy="success-message"]').should('be.visible');
  });

  it('should show validation errors for invalid card', () => {
    cy.visit('/checkout/payment');

    cy.get('[data-cy="card-number"]').type('1234');
    cy.get('[data-cy="place-order-button"]').click();

    cy.get('[data-cy="card-error"]')
      .should('be.visible')
      .and('contain', 'Invalid card number');
  });
});
```

### Fixtures

```json
// cypress/fixtures/users.json
[
  {
    "id": "1",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "admin"
  },
  {
    "id": "2",
    "name": "Jane Smith",
    "email": "jane@example.com",
    "role": "user"
  },
  {
    "id": "3",
    "name": "Bob Wilson",
    "email": "bob@example.com",
    "role": "user"
  }
]
```

### Docker Configuration

```dockerfile
# Dockerfile.cypress
FROM cypress/included:14

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# Run tests
CMD ["npm", "run", "test:e2e:ci"]
```

```yaml
# docker-compose.cypress.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=test
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3000/health']
      interval: 5s
      timeout: 3s
      retries: 30

  cypress:
    image: cypress/included:14
    depends_on:
      app:
        condition: service_healthy
    environment:
      - CYPRESS_baseUrl=http://app:3000
    working_dir: /e2e
    volumes:
      - ./:/e2e
      - /e2e/node_modules
    command: npx cypress run
```

### GitHub Actions Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  cypress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Cypress run
        uses: cypress-io/github-action@v6
        with:
          start: npm run start
          wait-on: 'http://localhost:3000'
          wait-on-timeout: 120
          browser: chrome
          record: true
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots
          retention-days: 7

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cypress-videos
          path: cypress/videos
          retention-days: 7
```

### Test Support Setup

```typescript
// cypress/support/e2e.ts
import './commands';

// Reset state before each test
beforeEach(() => {
  // Clear localStorage
  cy.clearLocalStorage();

  // Clear cookies
  cy.clearCookies();

  // Reset any intercepted routes
  cy.intercept('/api/**', (req) => {
    // Let requests pass through by default
    req.continue();
  });
});

// Global error handling
Cypress.on('uncaught:exception', (err) => {
  // Ignore specific errors
  if (err.message.includes('ResizeObserver')) {
    return false;
  }
  return true;
});
```

---

## Anti-Patterns

### ❌ Arbitrary Waits

```typescript
// ❌ DON'T — Flaky and slow
cy.get('[data-cy="submit"]').click();
cy.wait(3000);  // Arbitrary wait
cy.get('[data-cy="success"]');

// ✅ DO — Wait for specific condition
cy.get('[data-cy="submit"]').click();
cy.get('[data-cy="success"]').should('be.visible');
```

### ❌ Selecting by CSS Classes

```typescript
// ❌ DON'T — Brittle, changes with styling
cy.get('.btn-primary.large');
cy.get('#submit-button');

// ✅ DO — Use data-cy attributes
cy.get('[data-cy="submit-button"]');
```

### ❌ UI Login for Every Test

```typescript
// ❌ DON'T — Slow, tests login flow repeatedly
beforeEach(() => {
  cy.visit('/login');
  cy.get('[data-cy="email"]').type('user@example.com');
  cy.get('[data-cy="password"]').type('password');
  cy.get('[data-cy="submit"]').click();
});

// ✅ DO — Programmatic login
beforeEach(() => {
  cy.loginByApi('user@example.com', 'password');
});
```

### ❌ Chaining After Actions

```typescript
// ❌ DON'T — Actions don't yield subject
cy.get('[data-cy="input"]')
  .type('hello')
  .should('have.value', 'hello');  // Works but not recommended

cy.get('[data-cy="button"]')
  .click()
  .get('[data-cy="modal"]');  // DON'T chain after click

// ✅ DO — Separate query from assertion
cy.get('[data-cy="input"]').type('hello');
cy.get('[data-cy="input"]').should('have.value', 'hello');

cy.get('[data-cy="button"]').click();
cy.get('[data-cy="modal"]').should('be.visible');
```

### ❌ Test Interdependence

```typescript
// ❌ DON'T — Tests depend on each other
it('should create user', () => {
  // Creates user that next test depends on
});

it('should edit user', () => {
  // Fails if previous test didn't run
});

// ✅ DO — Independent tests with setup
beforeEach(() => {
  cy.request('POST', '/api/test/reset');
  cy.request('POST', '/api/test/seed');
});

it('should create user', () => { /* ... */ });
it('should edit user', () => { /* ... */ });
```

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `cy.visit(url)` | Navigate to URL |
| `cy.get(selector)` | Query element |
| `cy.contains(text)` | Find by text |
| `cy.intercept()` | Mock/spy network |
| `cy.request()` | Direct HTTP request |
| `cy.wait('@alias')` | Wait for intercept |
| `cy.fixture()` | Load fixture data |

| Assertion | Purpose |
|-----------|---------|
| `.should('be.visible')` | Element visible |
| `.should('exist')` | Element in DOM |
| `.should('have.length', n)` | Collection size |
| `.should('contain', text)` | Contains text |
| `.should('have.value', v)` | Input value |
| `.should('be.disabled')` | Element disabled |

| Command | Description |
|---------|-------------|
| `npx cypress open` | Open test runner |
| `npx cypress run` | Run headlessly |
| `npx cypress run --spec "path"` | Run specific spec |
| `npx cypress run --browser chrome` | Specific browser |
| `npx cypress run --record` | Record to Dashboard |

---

## 2025 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 13.x | 2024 | Stable baseline |
| **14.0** | **Jan 2025** | **JIT default**, Node 20+, React 18+, Angular 17+, Vue 2 removed |
| 14.x | 2025 | Angular 21, zoneless CT support |
| **15.0** | **2025** | Further breaking changes (see GitHub) |

### Cypress 14 Breaking Changes

**Platform Requirements:**
- Node.js 16 and 21 **removed** — use 18, 20, or 22+
- Bundled Node.js upgraded to **20.18.1**
- Linux requires glibc ≥2.28 (Ubuntu 20+, RHEL 8+)

**Framework Support Removed:**
```typescript
// ❌ REMOVED — No longer bundled
- Vue 2 → Use @cypress/vue2 package instead
- Svelte 3/4 → Use @cypress/svelte@2.x instead
- React ≤17 → React 18+ required
- Next.js ≤13 → Next.js 14+ required
- Angular <17.2.0 → Angular 17.2+ required
```

**New Framework Support:**
```typescript
// ✅ ADDED in Cypress 14+
- React 19
- Angular 19, 21 (with zoneless support)
- Next.js 15
- Svelte 5
- Vite 6
```

**JIT Compilation Default:**
```typescript
// cypress.config.ts
export default defineConfig({
  component: {
    // JIT compilation now DEFAULT (was experimental)
    // Only compiles resources related to current spec
    // Note: No benefit with Vite (already fast)
    justInTimeCompile: true, // Now default, deprecated option
  },
});
```

**Angular Zoneless Support (Cypress 14.3+):**
```typescript
// Angular 21+ zoneless component testing
import { mount } from 'cypress/angular-zoneless';

cy.mount(MyComponent);
```

**Chrome 137+ Warning:**
Chrome 137+ no longer supports `--load-extension` in branded Chrome. Use Electron, Chrome for Testing, or Chromium for `@cypress/puppeteer`.

---

## Resources

- [Cypress Documentation](https://docs.cypress.io/)
- [Cypress Best Practices](https://docs.cypress.io/guides/references/best-practices)
- [Real World App](https://github.com/cypress-io/cypress-realworld-app)
- [Cypress Component Testing](https://docs.cypress.io/guides/component-testing/overview)
- [Cypress 14 Release Blog](https://www.cypress.io/blog/cypress-14-is-here-see-whats-new)
- [v14 Migration Guide](https://docs.cypress.io/app/references/migration-guide)
