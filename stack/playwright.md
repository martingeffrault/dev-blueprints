# Playwright (2025)

> **Last updated**: January 2026
> **Versions covered**: Latest
> **Purpose**: End-to-end testing for modern web apps

---

## Philosophy (2025-2026)

Playwright is the **standard for reliable end-to-end testing**, offering cross-browser support, auto-waiting, and powerful debugging tools.

**Key philosophical shifts:**
- **Test from user perspective** — Interact with what users see
- **Auto-wait built-in** — No more flaky timeouts
- **Test isolation** — Fresh context per test
- **Resilient selectors** — Role-based, not CSS-based
- **API-first setup** — Use API calls, not UI, for test setup
- **Parallel by default** — Fast test execution

---

## TL;DR

- Use `page.getByRole()`, `page.getByText()`, `page.getByTestId()` for selectors
- Never use `page.waitForTimeout()` — use auto-waiting
- Isolate tests — each test gets fresh browser context
- Use API calls for setup (authentication, data seeding)
- Use Trace Viewer for debugging failed tests
- Run in CI with `npx playwright test --reporter=html`
- Use `test.step()` for readable test reports

---

## Best Practices

### Project Setup

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI, // Fail if test.only in CI
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { open: 'never' }],
    ['list'],
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry', // Capture trace on failure
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 12'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Test Structure

```typescript
// tests/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can sign up', async ({ page }) => {
    await test.step('Navigate to signup', async () => {
      await page.goto('/signup');
    });

    await test.step('Fill signup form', async () => {
      await page.getByLabel('Email').fill('test@example.com');
      await page.getByLabel('Password').fill('SecurePass123!');
      await page.getByLabel('Confirm Password').fill('SecurePass123!');
    });

    await test.step('Submit and verify', async () => {
      await page.getByRole('button', { name: 'Sign Up' }).click();
      await expect(page).toHaveURL('/dashboard');
      await expect(page.getByText('Welcome')).toBeVisible();
    });
  });

  test('user can log in', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('existing@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Log In' }).click();

    await expect(page).toHaveURL('/dashboard');
  });
});
```

### Resilient Selectors

```typescript
// ✅ BEST — Role-based (accessible)
await page.getByRole('button', { name: 'Submit' }).click();
await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com');
await page.getByRole('link', { name: 'Home' }).click();
await page.getByRole('heading', { name: 'Dashboard' });

// ✅ GOOD — Text-based
await page.getByText('Welcome back').click();
await page.getByLabel('Password').fill('secret');
await page.getByPlaceholder('Search...').fill('query');

// ✅ GOOD — Test ID (for dynamic content)
await page.getByTestId('submit-button').click();
await page.getByTestId('user-card-123').click();

// ⚠️ AVOID — CSS selectors (brittle)
await page.locator('.btn-primary').click(); // Breaks on CSS changes
await page.locator('#submit').click(); // Slightly better but still brittle

// ❌ DON'T — Long CSS paths
await page.locator('div.container > form > button.submit').click();
```

### Authentication Setup (Reuse Auth State)

```typescript
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Log In' }).click();

  await expect(page).toHaveURL('/dashboard');

  // Save authentication state
  await page.context().storageState({ path: authFile });
});

// playwright.config.ts
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

### API-Driven Test Setup

```typescript
// tests/products.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Product Management', () => {
  let productId: string;

  test.beforeEach(async ({ request }) => {
    // Create test data via API (faster than UI)
    const response = await request.post('/api/products', {
      data: {
        name: 'Test Product',
        price: 99.99,
      },
    });
    const product = await response.json();
    productId = product.id;
  });

  test.afterEach(async ({ request }) => {
    // Cleanup via API
    await request.delete(`/api/products/${productId}`);
  });

  test('can view product details', async ({ page }) => {
    await page.goto(`/products/${productId}`);
    await expect(page.getByRole('heading', { name: 'Test Product' })).toBeVisible();
    await expect(page.getByText('$99.99')).toBeVisible();
  });
});
```

### Assertions (Web-First)

```typescript
// ✅ Auto-retrying assertions
await expect(page.getByRole('button')).toBeVisible();
await expect(page.getByRole('button')).toBeEnabled();
await expect(page.getByRole('button')).toHaveText('Submit');
await expect(page).toHaveURL('/dashboard');
await expect(page).toHaveTitle('Dashboard');
await expect(page.getByRole('list')).toHaveCount(5);

// ✅ Negative assertions
await expect(page.getByRole('alert')).not.toBeVisible();
await expect(page.getByRole('button')).not.toBeDisabled();

// ✅ Text assertions
await expect(page.getByTestId('balance')).toHaveText(/\$[\d,]+\.\d{2}/);
await expect(page.getByRole('status')).toContainText('Success');
```

### Page Object Model

```typescript
// pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Log In' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toHaveText(message);
  }
}

// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('shows error for invalid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('wrong@email.com', 'wrongpassword');
  await loginPage.expectError('Invalid email or password');
});
```

### Visual Testing

```typescript
test('homepage matches snapshot', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100, // Allow small differences
  });
});

test('button states match snapshots', async ({ page }) => {
  await page.goto('/components/button');

  const button = page.getByRole('button', { name: 'Submit' });

  // Default state
  await expect(button).toHaveScreenshot('button-default.png');

  // Hover state
  await button.hover();
  await expect(button).toHaveScreenshot('button-hover.png');
});
```

---

## Anti-Patterns

### ❌ Using Hard-Coded Waits

**Why it's bad**: Slow, flaky tests.

```typescript
// ❌ DON'T
await page.click('button');
await page.waitForTimeout(3000); // Bad!
await expect(page.locator('.result')).toBeVisible();

// ✅ DO — Auto-waiting assertions
await page.getByRole('button').click();
await expect(page.getByTestId('result')).toBeVisible(); // Auto-waits
```

### ❌ Testing Implementation Details

**Why it's bad**: Tests break when implementation changes.

```typescript
// ❌ DON'T
await expect(page.locator('[data-loading="true"]')).toBeVisible();
await expect(page.locator('.MuiButton-root')).toHaveClass('MuiButton-contained');

// ✅ DO — Test what users see
await expect(page.getByRole('progressbar')).toBeVisible();
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();
```

### ❌ Not Isolating Tests

**Why it's bad**: Tests affect each other, causing flakiness.

```typescript
// ❌ DON'T — Shared state
let userId: string;

test('create user', async ({ page }) => {
  // Creates user, sets userId
});

test('view user', async ({ page }) => {
  await page.goto(`/users/${userId}`); // Depends on previous test!
});

// ✅ DO — Each test sets up its own data
test('view user', async ({ page, request }) => {
  const user = await request.post('/api/users', { ... });
  await page.goto(`/users/${user.id}`);
  // Cleanup in afterEach
});
```

### ❌ Using UI for Test Setup

**Why it's bad**: Slow, duplicates logic across tests.

```typescript
// ❌ DON'T — Log in via UI for every test
test.beforeEach(async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name=email]', 'test@example.com');
  await page.fill('[name=password]', 'password');
  await page.click('button[type=submit]');
});

// ✅ DO — Use API or storage state
test.use({ storageState: 'playwright/.auth/user.json' });
```

---

## Quick Reference

| Task | Solution |
|------|----------|
| Run all tests | `npx playwright test` |
| Run specific file | `npx playwright test tests/auth.spec.ts` |
| Run in headed mode | `npx playwright test --headed` |
| Run specific browser | `npx playwright test --project=chromium` |
| Debug mode | `npx playwright test --debug` |
| UI mode | `npx playwright test --ui` |
| Update snapshots | `npx playwright test --update-snapshots` |
| Show report | `npx playwright show-report` |
| Generate code | `npx playwright codegen localhost:3000` |

| Selector | Usage |
|----------|-------|
| By role | `page.getByRole('button', { name: 'Submit' })` |
| By text | `page.getByText('Welcome')` |
| By label | `page.getByLabel('Email')` |
| By placeholder | `page.getByPlaceholder('Search')` |
| By test ID | `page.getByTestId('submit-btn')` |
| By alt text | `page.getByAltText('Logo')` |
| By title | `page.getByTitle('Close')` |

---

## Resources

- [Official Playwright Documentation](https://playwright.dev/)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [Locators Guide](https://playwright.dev/docs/locators)
- [Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [BrowserStack Guide](https://www.browserstack.com/guide/playwright-best-practices)
