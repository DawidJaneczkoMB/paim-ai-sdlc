---
name: e2e-testing
description: Use when writing, debugging, or configuring Playwright E2E tests — locators, assertions, Page Object Model, fixtures, network mocking, authentication state, flaky test diagnosis, or playwright.config.ts. Triggers on any Playwright task.
---

# E2E Testing (Playwright)

**Official Docs**: [Playwright](https://playwright.dev/docs/intro)

## Locator Priority

Use locators in this order. Never use CSS/XPath when a semantic locator exists.

1. `getByRole` — most resilient, matches accessibility tree
2. `getByLabel` — form inputs with labels
3. `getByPlaceholder` — when no label exists
4. `getByText` — non-interactive visible text
5. `getByTestId` — when semantic locators are not feasible
6. CSS/XPath — last resort only, document why

```typescript
// BAD
page.locator('button.buttonIcon.episode-actions-later');
page.locator('div.container > ul > li:nth-child(3) > span.text');

// GOOD
page.getByRole('button', { name: 'Submit' });
page.getByLabel('Email address');
page.getByText(/total: \$\d+/i);
```

Use `filter()` and chaining to narrow locators:

```typescript
page
  .getByRole('listitem')
  .filter({ hasText: 'Product' })
  .getByRole('button', { name: 'Buy' });
```

## Arrange-Act-Assert (AAA)

```typescript
test('should show error for invalid credentials', async ({ page }) => {
  // Arrange
  const loginPage = new LoginPage(page);
  await loginPage.goto();

  // Act
  await loginPage.login('user@example.com', 'wrong-password');

  // Assert
  await expect(page.getByRole('alert')).toContainText('Invalid credentials');
});
```

One behavior per test. Split tests that assert unrelated things.

## Assertions — Web-First Only

Always use web-first assertions that auto-retry. Never use manual assertions with `isVisible()`.

```typescript
// BAD - no auto-retry
expect(await page.getByText('welcome').isVisible()).toBe(true);

// GOOD - auto-retries
await expect(page.getByText('welcome')).toBeVisible();
await expect(locator).toHaveText('Expected text');
await expect(locator).toBeDisabled();
await expect(page.getByRole('listitem')).toHaveCount(5);
await expect(page).toHaveURL('/dashboard');
```

Use soft assertions for multiple non-critical conditions:

```typescript
await expect.soft(page.getByRole('heading')).toHaveText('Dashboard');
await expect.soft(page.getByRole('button', { name: 'Save' })).toBeEnabled();
```

## Waiting

Never use `waitForTimeout`. Always wait for specific conditions.

```typescript
// BAD
await page.waitForTimeout(3000);

// GOOD - wait for condition
await expect(page.getByRole('heading')).toBeVisible();

// GOOD - wait for network response
const responsePromise = page.waitForResponse('**/api/submit');
await page.getByRole('button', { name: 'Submit' }).click();
await responsePromise;

// GOOD - polling
await expect(async () => {
  const response = await page.request.get('/api/status');
  expect(response.status()).toBe(200);
}).toPass({ intervals: [1000, 2000, 5000], timeout: 30000 });
```

## Test Isolation

Each test must be independent. Never share mutable state between tests.

```typescript
// BAD - shared page leaks state
let sharedPage: Page;
test.beforeAll(async ({ browser }) => {
  sharedPage = await browser.newPage();
});

// GOOD - fresh page per test
test('first test', async ({ page }) => {
  /* fresh page */
});
```

Group related tests with `test.describe`. Use `beforeEach` for common navigation.

## Page Object Model

Encapsulate selectors and actions per page. Keep assertions in tests, not page objects (except reusable `expect*` helpers).

```typescript
export class LoginPage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
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
    await expect(this.errorMessage).toContainText(message);
  }
}
```

Compose page objects from component objects:

```typescript
export class NavbarComponent {
  readonly container: Locator;
  readonly searchInput: Locator;

  constructor(page: Page) {
    this.container = page.getByRole('navigation');
    this.searchInput = this.container.getByRole('searchbox');
  }
}

export class DashboardPage {
  readonly navbar: NavbarComponent;
  constructor(page: Page) {
    this.navbar = new NavbarComponent(page);
  }
}
```

## Custom Fixtures

Prefer fixtures over `beforeAll`/`afterAll` — cleanup runs even on test failure.

```typescript
type MyFixtures = { loginPage: LoginPage; authenticatedPage: Page };

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await use(loginPage);
  },
  authenticatedPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: '.auth/user.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },
});
```

### Authentication State

Log in once in a setup project, reuse storage state:

```typescript
// auth.setup.ts
setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/dashboard');
  await page.context().storageState({ path: '.auth/user.json' });
});
```

```typescript
// playwright.config.ts
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  { name: 'chromium', use: { ...devices['Desktop Chrome'], storageState: '.auth/user.json' }, dependencies: ['setup'] },
],
```

## Network Interception

Mock external APIs. Never test third-party services directly.

```typescript
test('should display products from API', async ({ page }) => {
  await page.route('**/api/products', (route) =>
    route.fulfill({
      status: 200,
      json: [{ id: 1, name: 'Test Product', price: 9.99 }],
    }),
  );
  await page.goto('/products');
  await expect(page.getByText('Test Product')).toBeVisible();
});
```

## Test Data Factories

```typescript
let userIdCounter = 0;
export function createUser(overrides: Partial<User> = {}): User {
  userIdCounter++;
  return {
    id: `user-${userIdCounter}`,
    email: `user${userIdCounter}@test.com`,
    name: `Test User ${userIdCounter}`,
    role: 'user',
    ...overrides,
  };
}
```

## Configuration

```typescript
export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

Always configure `baseURL`. Set `forbidOnly` in CI. Enable `trace: 'on-first-retry'` for CI debugging.

## Tagging and Filtering

```typescript
test('checkout flow @smoke @critical', async ({ page }) => {
  /* ... */
});
```

```bash
npx playwright test --grep @smoke
npx playwright test --grep-invert @slow
```

Parameterize tests with arrays to avoid duplication:

```typescript
const users = [
  { role: 'admin', canDelete: true },
  { role: 'viewer', canDelete: false },
] as const;

for (const { role, canDelete } of users) {
  test(`${role} delete permission`, async ({ page }) => { /* ... */ });
}
```

## Common Scenarios

### Dialogs

```typescript
page.on('dialog', async (dialog) => {
  expect(dialog.type()).toBe('confirm');
  await dialog.accept();
});
await page.getByRole('button', { name: 'Delete' }).click();
```

### File Upload

```typescript
await page.getByLabel('Upload document').setInputFiles('test-data/sample.pdf');
await expect(page.getByText('sample.pdf')).toBeVisible();
```

### Iframes

```typescript
const iframe = page.frameLocator('#payment-iframe');
await iframe.getByLabel('Card number').fill('4111111111111111');
```

### Dropdowns

```typescript
// Native select
await page.getByLabel('Country').selectOption('US');

// Custom dropdown
await page.getByRole('combobox', { name: 'Country' }).click();
await page.getByRole('option', { name: 'United States' }).click();
```

## Debugging

```bash
npx playwright test --headed                          # Run with visible browser
npx playwright test --ui                              # Interactive UI mode
npx playwright test --debug tests/login.spec.ts       # Pause and inspect
npx playwright codegen https://example.com            # Record test interactions
npx playwright show-trace test-results/trace.zip      # Inspect failure trace
```

Use `await page.pause()` inside a test to pause execution mid-run.

## Flaky Test Prevention

- Never increase global timeout to "fix" flakes — find the root cause.
- Run new tests multiple times before merging: `npx playwright test --repeat-each=20`.
- Mock external APIs to eliminate network variability.
- Use progressive assertions to narrow down root cause:

```typescript
// BAD - single opaque failure
await expect(page.locator('.items')).toHaveCount(5);

// GOOD - progressive checks aid diagnosis
await expect(page.locator('.items-container')).toBeVisible();
await expect(page.locator('.loading')).not.toBeVisible();
await expect(page.locator('.items')).toHaveCount(5);
```

## Project Structure

```
tests/
  e2e/
    auth/
      login.spec.ts
    checkout/
      cart.spec.ts
    fixtures/
      auth.fixture.ts
    pages/
      login.page.ts
      dashboard.page.ts
    components/
      navbar.component.ts
    factories/
      user.factory.ts
playwright.config.ts
```

## Anti-Patterns

| Anti-Pattern                     | Problem                 | Solution                                   |
| -------------------------------- | ----------------------- | ------------------------------------------ |
| `waitForTimeout()`               | Slow, flaky, arbitrary  | Use auto-waiting assertions                |
| CSS/XPath selectors              | Break on layout changes | Use `getByRole`, `getByLabel`, `getByText` |
| Shared mutable state             | Order-dependent, racy   | Use fixtures with per-test isolation       |
| Manual `isVisible()` checks      | No auto-retry           | Use `await expect(locator).toBeVisible()`  |
| Testing third-party APIs         | Uncontrollable failures | Mock with `page.route()`                   |
| Increasing timeout to fix flakes | Masks root cause        | Diagnose with traces, fix real issue       |
