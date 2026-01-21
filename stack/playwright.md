# Playwright (2025)

> **Last updated**: January 2026
> **Versions covered**: Playwright 1.49-1.57
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

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **1.57** | Nov 2025 | Chrome for Testing (no more Chromium), Speedboard, Service Worker routing |
| **1.56** | Oct 2025 | Playwright Agents (AI test generation), testStepInfo.titlePath |
| **1.55** | Sep 2025 | Debian 13 Trixie support |
| **1.49** | Jan 2025 | New headless mode default, accessibility API removed |

### Playwright 1.57 Breaking Changes

**Chrome for Testing (No More Chromium):**
```typescript
// ✅ No code changes needed — Chromium → Chrome for Testing
// Your tests should continue to work
// Main visible change: Chrome icon in toolbar instead of Chromium

// If you need old behavior (rare):
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: 'chromium',
      use: {
        channel: 'chromium', // Explicitly use Chromium (not recommended)
      },
    },
  ],
});
```

**Speedboard — Find Slow Tests:**
```bash
# View slowest tests in HTML reporter
npx playwright test
npx playwright show-report
# → Click "Speedboard" tab to see tests sorted by duration
```

**Service Worker Network Routing:**
```typescript
// NEW — Route Service Worker requests (Chromium only)
await context.route('**/api/**', route => {
  // Now also intercepts Service Worker requests!
  return route.fulfill({ json: mockData });
});

// Opt-out if needed:
// Set PLAYWRIGHT_DISABLE_SERVICE_WORKER_NETWORK=1

// Service Worker console messages:
worker.on('console', msg => console.log('SW:', msg.text()));
// Set PLAYWRIGHT_DISABLE_SERVICE_WORKER_CONSOLE=1 to opt-out
```

**WebServer Wait Field:**
```typescript
// NEW — Wait for specific log message before starting tests
// playwright.config.ts
export default defineConfig({
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    wait: /Server started/, // Wait for this regex in stdout
    reuseExistingServer: !process.env.CI,
  },
});
```

### Playwright 1.56 — AI Test Agents

**Playwright Agents for AI Test Generation:**
```bash
# Generate agent definitions for AI assistants
npx playwright generate-agents

# Three agents are created:
# 🎭 planner — explores app, produces Markdown test plan
# 🎭 generator — transforms plan into Playwright test files
# 🎭 healer — executes tests, auto-repairs failures
```

**Playwright MCP Server (AI Integration):**
```json
// Configure for Claude Desktop (~/.config/claude/claude_desktop_config.json)
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```
```typescript
// MCP uses accessibility tree — no screenshots needed!
// AI can now:
// - Generate tests from natural language
// - Navigate and interact with pages
// - Self-heal broken selectors
// - Debug and fix failing tests
```

**Auto-generate toBeVisible() Assertions:**
```bash
# In codegen UI, enable auto-assertions
npx playwright codegen localhost:3000
# → Settings → Enable "Auto-generate assertions"
# Now clicking elements auto-adds toBeVisible() checks
```

**testStepInfo.titlePath:**
```typescript
// NEW — Get full title path in custom reporters
class MyReporter {
  onStepEnd(test, result, step) {
    // Returns: ['auth.spec.ts', 'login flow', 'fill credentials']
    console.log(step.titlePath);
  }
}
```

### Playwright 1.49 Breaking Changes

**New Headless Mode Default:**
```typescript
// Chrome's new headless = real Chrome (not stripped-down Chromium)
// More accurate for E2E testing
// Opt into explicitly:
export default defineConfig({
  projects: [{
    name: 'chromium',
    use: {
      channel: 'chromium', // Uses new headless Chrome
    },
  }],
});
```

**page.accessibility Removed:**
```typescript
// ❌ REMOVED — After 3 years deprecated
const snapshot = await page.accessibility.snapshot();

// ✅ DO — Use dedicated accessibility testing libraries
import AxeBuilder from '@axe-core/playwright';

test('should be accessible', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

**Deprecated Packages:**
```bash
# ❌ No longer updated:
@playwright/experimental-ct-vue2   # Vue 2 is EOL
@playwright/experimental-ct-solid  # Use regular Playwright

# ❌ No updates for:
# WebKit on Ubuntu 20.04 / Debian 11
# Upgrade your OS to get WebKit updates
```

**Chromium Extension Changes:**
```typescript
// ❌ BREAKING — Manifest V2 extensions no longer supported
// Migrate your extensions to Manifest V3

// Manifest V3 still works:
const context = await chromium.launchPersistentContext('/tmp/profile', {
  headless: false,
  args: [
    `--disable-extensions-except=${extensionPath}`,
    `--load-extension=${extensionPath}`,
  ],
});
```

---

## Resources

- [Official Playwright Documentation](https://playwright.dev/)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [Locators Guide](https://playwright.dev/docs/locators)
- [Trace Viewer](https://playwright.dev/docs/trace-viewer)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [Playwright Test Agents](https://playwright.dev/docs/test-agents)
- [Release Notes](https://playwright.dev/docs/release-notes)
