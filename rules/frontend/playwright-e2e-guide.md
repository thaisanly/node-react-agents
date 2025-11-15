# Playwright E2E Testing Guide

## Overview

This guide covers end-to-end (E2E) testing for React applications using Playwright. Playwright enables reliable cross-browser testing with powerful automation capabilities.

## ⚠️ MANDATORY REQUIREMENT

**All E2E tests MUST use the Page Object Model (POM) pattern.**

Direct page interactions in test files are NOT allowed. This ensures:
- **Maintainability**: Changes to UI only require updating the Page Object
- **Reusability**: Page Objects can be shared across multiple tests
- **Readability**: Tests describe user workflows, not implementation details
- **Type Safety**: Full TypeScript support with autocomplete

## Table of Contents

- [Setup](#setup)
- [Configuration](#configuration)
- [Project Structure](#project-structure)
- [Writing Tests](#writing-tests)
- [Page Object Model](#page-object-model)
- [Best Practices](#best-practices)
- [Running Tests](#running-tests)
- [CI/CD Integration](#cicd-integration)
- [Common Patterns](#common-patterns)

## Setup

### Installation

```bash
npm init playwright@latest
```

This command will:
- Install Playwright and browsers
- Create `playwright.config.ts`
- Create example tests in `e2e/` or `tests/` folder
- Set up GitHub Actions workflow (optional)

### Manual Installation

```bash
npm install -D @playwright/test
npx playwright install
```

## Configuration

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',

  // Run tests in files in parallel
  fullyParallel: true,

  // Fail the build on CI if test.only is left
  forbidOnly: !!process.env.CI,

  // Retry on CI only
  retries: process.env.CI ? 2 : 0,

  // Opt out of parallel tests on CI
  workers: process.env.CI ? 1 : undefined,

  // Reporter configuration
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    ['junit', { outputFile: 'test-results/results.xml' }]
  ],

  use: {
    // Base URL for navigation
    baseURL: 'http://localhost:5173',

    // Collect trace on first retry
    trace: 'on-first-retry',

    // Screenshot on failure
    screenshot: 'only-on-failure',

    // Video on failure
    video: 'retain-on-failure',

    // Default timeout for actions
    actionTimeout: 10000,
  },

  // Configure projects for different browsers
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },

    // Mobile testing
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],

  // Run dev server before tests
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000,
  },
});
```

## Project Structure

```
frontend/
├── e2e/
│   ├── fixtures/
│   │   ├── test-data.ts          # Test data fixtures
│   │   └── custom-fixtures.ts    # Custom Playwright fixtures
│   ├── pages/
│   │   ├── BasePage.ts           # Base page class
│   │   ├── HomePage.ts           # Home page object
│   │   ├── LoginPage.ts          # Login page object
│   │   └── DashboardPage.ts      # Dashboard page object
│   ├── tests/
│   │   ├── auth/
│   │   │   ├── login.spec.ts
│   │   │   └── signup.spec.ts
│   │   ├── features/
│   │   │   └── user-management.spec.ts
│   │   └── smoke/
│   │       └── critical-path.spec.ts
│   └── utils/
│       ├── helpers.ts            # Test helper functions
│       └── test-ids.ts           # Centralized test IDs
├── playwright.config.ts
└── .env.test                      # Test environment variables
```

## Writing Tests

### ❌ WRONG: Direct Page Interactions (DO NOT USE)

The following pattern is NOT allowed in our codebase:

```typescript
import { test, expect } from '@playwright/test';

// ❌ BAD: Direct page interactions in test
test('should login with valid credentials', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com'); // ❌ Direct interaction
  await page.getByLabel('Password').fill('password123');   // ❌ Direct interaction
  await page.getByRole('button', { name: 'Login' }).click(); // ❌ Direct interaction

  await expect(page).toHaveURL('/dashboard');
});
```

**Why is this bad?**
- If the UI changes (e.g., label text), you must update ALL tests
- No reusability across tests
- Difficult to maintain
- No abstraction of page logic

### ✅ CORRECT: Using Page Object Model (REQUIRED)

All tests MUST use Page Objects. See the [Page Object Model](#page-object-model) section below for implementation details.

### Using Test IDs

**Component (React):**
```tsx
function LoginForm() {
  return (
    <form data-testid="login-form">
      <input data-testid="email-input" type="email" />
      <input data-testid="password-input" type="password" />
      <button data-testid="submit-button">Login</button>
    </form>
  );
}
```

**Test:**
```typescript
test('should submit login form', async ({ page }) => {
  await page.getByTestId('email-input').fill('user@example.com');
  await page.getByTestId('password-input').fill('password123');
  await page.getByTestId('submit-button').click();

  await expect(page).toHaveURL('/dashboard');
});
```

## Page Object Model

### Base Page

```typescript
// e2e/pages/BasePage.ts
import { Page } from '@playwright/test';

export class BasePage {
  constructor(protected page: Page) {}

  async goto(path: string) {
    await this.page.goto(path);
  }

  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }

  async takeScreenshot(name: string) {
    await this.page.screenshot({ path: `screenshots/${name}.png` });
  }
}
```

### Feature Page

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Login' });
    this.errorMessage = page.getByTestId('error-message');
  }

  async goto() {
    await super.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectErrorMessage(message: string) {
    await expect(this.errorMessage).toHaveText(message);
  }
}
```

### Using Page Objects

```typescript
import { test } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('should login successfully', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await expect(page).toHaveURL('/dashboard');
});
```

## Best Practices

### 0. ALWAYS Use Page Object Model (MANDATORY)

**Every test MUST use Page Objects. Direct page interactions are prohibited.**

✅ **CORRECT:**
```typescript
// e2e/tests/auth/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/LoginPage';
import { DashboardPage } from '../../pages/DashboardPage';

test('should login successfully', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await dashboardPage.expectWelcomeMessage('Welcome back');
});
```

❌ **INCORRECT:**
```typescript
// ❌ DO NOT DO THIS
test('should login successfully', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com'); // ❌ Direct interaction
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
});
```

### 1. Use Meaningful Locators

**Good:**
```typescript
// Use semantic queries
await page.getByRole('button', { name: 'Submit' });
await page.getByLabel('Email address');
await page.getByText('Welcome back');
await page.getByTestId('user-profile');
```

**Avoid:**
```typescript
// Fragile selectors
await page.locator('.btn-primary');
await page.locator('#form > div:nth-child(2) > input');
```

### 2. Wait for Elements Properly

```typescript
// Wait for element to be visible
await expect(page.getByText('Success')).toBeVisible();

// Wait for navigation
await page.waitForURL('/dashboard');

// Wait for API response
await page.waitForResponse(response =>
  response.url().includes('/api/users') && response.status() === 200
);

// Wait for network idle
await page.waitForLoadState('networkidle');
```

### 3. Isolate Tests

```typescript
test.describe('User Management', () => {
  test.beforeEach(async ({ page }) => {
    // Setup: Create fresh test data
    await page.request.post('/api/test/setup');
  });

  test.afterEach(async ({ page }) => {
    // Cleanup: Remove test data
    await page.request.post('/api/test/cleanup');
  });

  test('should create user', async ({ page }) => {
    // Test is isolated and won't affect other tests
  });
});
```

### 4. Use Custom Fixtures

```typescript
// e2e/fixtures/custom-fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

type MyFixtures = {
  loginPage: LoginPage;
  authenticatedPage: Page;
};

export const test = base.extend<MyFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  authenticatedPage: async ({ page }, use) => {
    // Auto-login before test
    await page.goto('/login');
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();
    await page.waitForURL('/dashboard');
    await use(page);
  },
});

export { expect } from '@playwright/test';
```

**Usage:**
```typescript
import { test, expect } from '../fixtures/custom-fixtures';

test('should access dashboard', async ({ authenticatedPage }) => {
  // Already logged in!
  await expect(authenticatedPage.getByText('Dashboard')).toBeVisible();
});
```

### 5. Handle Authentication

```typescript
// Save authenticated state
test('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();

  await page.context().storageState({ path: 'auth.json' });
});

// Reuse authenticated state
test.use({ storageState: 'auth.json' });

test('use authenticated state', async ({ page }) => {
  // Already logged in
  await page.goto('/dashboard');
});
```

## Running Tests

### Run All Tests

```bash
# Run all tests
npx playwright test

# Run tests in headed mode (see browser)
npx playwright test --headed

# Run tests in debug mode
npx playwright test --debug

# Run specific test file
npx playwright test e2e/tests/auth/login.spec.ts

# Run tests matching pattern
npx playwright test --grep "login"
```

### Run Tests in UI Mode

```bash
npx playwright test --ui
```

### View Test Report

```bash
npx playwright show-report
```

### Generate Code

```bash
# Record actions and generate test code
npx playwright codegen http://localhost:5173
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test

      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## Common Patterns

### Testing Forms

```typescript
test('should submit form with validation', async ({ page }) => {
  await page.goto('/contact');

  // Test empty form validation
  await page.getByRole('button', { name: 'Submit' }).click();
  await expect(page.getByText('Name is required')).toBeVisible();

  // Fill and submit
  await page.getByLabel('Name').fill('John Doe');
  await page.getByLabel('Email').fill('john@example.com');
  await page.getByLabel('Message').fill('Hello world');
  await page.getByRole('button', { name: 'Submit' }).click();

  // Verify success
  await expect(page.getByText('Message sent successfully')).toBeVisible();
});
```

### Testing API Interactions

```typescript
test('should load data from API', async ({ page }) => {
  // Mock API response
  await page.route('**/api/users', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' }
      ])
    });
  });

  await page.goto('/users');

  await expect(page.getByText('John')).toBeVisible();
  await expect(page.getByText('Jane')).toBeVisible();
});
```

### Testing File Upload

```typescript
test('should upload file', async ({ page }) => {
  await page.goto('/upload');

  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('path/to/file.pdf');

  await page.getByRole('button', { name: 'Upload' }).click();

  await expect(page.getByText('File uploaded successfully')).toBeVisible();
});
```

### Testing Responsive Design

```typescript
test('should display mobile menu', async ({ page }) => {
  // Set mobile viewport
  await page.setViewportSize({ width: 375, height: 667 });

  await page.goto('/');

  // Desktop menu should be hidden
  await expect(page.getByTestId('desktop-menu')).not.toBeVisible();

  // Mobile menu button should be visible
  await expect(page.getByTestId('mobile-menu-button')).toBeVisible();

  // Open mobile menu
  await page.getByTestId('mobile-menu-button').click();
  await expect(page.getByTestId('mobile-menu')).toBeVisible();
});
```

### Visual Regression Testing

```typescript
test('should match screenshot', async ({ page }) => {
  await page.goto('/');

  // Compare full page screenshot
  await expect(page).toHaveScreenshot('homepage.png');

  // Compare specific element
  const header = page.getByRole('banner');
  await expect(header).toHaveScreenshot('header.png');
});
```

## Debugging Tips

### 1. Use Playwright Inspector

```bash
npx playwright test --debug
```

### 2. Add Debug Points

```typescript
test('debug test', async ({ page }) => {
  await page.goto('/');
  await page.pause(); // Pauses execution
});
```

### 3. Console Logs

```typescript
test('log test', async ({ page }) => {
  page.on('console', msg => console.log(msg.text()));
  await page.goto('/');
});
```

### 4. Slow Motion

```typescript
test.use({ launchOptions: { slowMo: 1000 } });
```

## Resources

- [Playwright Documentation](https://playwright.dev/)
- [Playwright API Reference](https://playwright.dev/docs/api/class-playwright)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [Selectors Guide](https://playwright.dev/docs/selectors)

## Related Documentation

- [Frontend Development Guide](./frontend-development-guide.md)
- [Form Validation Guide](./form-validation-guide.md)
- [Testing Guide](../backend/testing-guide.md)
