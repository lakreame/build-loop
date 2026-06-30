# Testing & CI Pattern Reference

Playwright e2e patterns and GitHub Actions matrix build configs, covering every stack
combination in build-loop: React/TS, Blazor, Flutter Web, Node.js, .NET, and Python backends.

---

## Playwright — Project Setup

### Install & Config
```bash
npm init playwright@latest
# Choose: TypeScript, tests/ folder, GitHub Actions workflow, install browsers
```

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html', { open: 'never' }], ['github']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

---

## Playwright — Core Patterns

### Page Object Model
```ts
// e2e/pages/LoginPage.ts
import { type Page, type Locator } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator
  readonly errorMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('Email')
    this.passwordInput = page.getByLabel('Password')
    this.submitButton = page.getByRole('button', { name: 'Sign in' })
    this.errorMessage = page.getByRole('alert')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }
}
```

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'

test.describe('Authentication', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await loginPage.login('user@example.com', 'correct-password')
    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Welcome back')).toBeVisible()
  })

  test('invalid credentials show error', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await loginPage.login('user@example.com', 'wrong-password')
    await expect(loginPage.errorMessage).toContainText('Invalid credentials')
  })

  test('rate limiting blocks after repeated failures', async ({ page }) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    for (let i = 0; i < 11; i++) {
      await loginPage.login('user@example.com', 'wrong-password')
    }
    await expect(loginPage.errorMessage).toContainText('Too many requests')
  })
})
```

### API Mocking / Interception
```ts
test('handles API failure gracefully', async ({ page }) => {
  await page.route('**/api/dashboard', (route) =>
    route.fulfill({ status: 500, body: JSON.stringify({ error: 'Server error' }) })
  )
  await page.goto('/dashboard')
  await expect(page.getByText('Something went wrong')).toBeVisible()
})
```

### Authenticated Session Reuse (avoid logging in every test)
```ts
// e2e/auth.setup.ts
import { test as setup } from '@playwright/test'

const authFile = 'playwright/.auth/user.json'

setup('authenticate', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('user@example.com')
  await page.getByLabel('Password').fill('correct-password')
  await page.getByRole('button', { name: 'Sign in' }).click()
  await page.waitForURL('/dashboard')
  await page.context().storageState({ path: authFile })
})
```

```ts
// playwright.config.ts — reference the auth state
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'chromium',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
    dependencies: ['setup'],
  },
]
```

### Visual Regression
```ts
test('dashboard matches snapshot', async ({ page }) => {
  await page.goto('/dashboard')
  await expect(page).toHaveScreenshot('dashboard.png', { maxDiffPixels: 100 })
})
```

### Stripe Checkout Flow (test mode)
```ts
test('completes checkout with test card', async ({ page }) => {
  await page.goto('/billing/upgrade')
  await page.getByRole('button', { name: 'Upgrade to Pro' }).click()

  const stripeFrame = page.frameLocator('iframe[name^="__privateStripeFrame"]')
  await stripeFrame.locator('[name="cardnumber"]').fill('4242424242424242')
  await stripeFrame.locator('[name="exp-date"]').fill('12/34')
  await stripeFrame.locator('[name="cvc"]').fill('123')

  await page.getByRole('button', { name: 'Subscribe' }).click()
  await expect(page.getByText('Subscription active')).toBeVisible({ timeout: 10000 })
})
```

---

## Playwright — Screenshots

Screenshots serve three purposes in this skill: failure diagnosis, explicit visual proof for
the Verify phase, and visual regression baselines. Use the right one for the job.

### Automatic — On Failure (already in config)

`playwright.config.ts` should always set this (see Project Setup above):
```ts
use: {
  screenshot: 'only-on-failure',
  trace: 'on-first-retry',
  video: 'retain-on-failure',
}
```
This requires no test code — Playwright captures automatically and attaches to the HTML report.

### Explicit — Full Page Capture

Use when a loop's `DONE WHEN` criterion is visual ("user sees the dashboard render correctly")
and you want a screenshot saved regardless of pass/fail, not just on failure:

```ts
test('dashboard renders after login', async ({ page }) => {
  await page.goto('/dashboard')
  await expect(page.getByText('Welcome back')).toBeVisible()

  await page.screenshot({
    path: 'test-results/screenshots/dashboard-loaded.png',
    fullPage: true,
  })
})
```

### Explicit — Single Element Capture

Useful for verifying a specific component (e.g. the bounding-box overlay, a chart, a form state)
without noise from the rest of the page:

```ts
test('detection overlay renders correctly', async ({ page }) => {
  await page.goto('/')
  await detector.uploadImage('e2e/fixtures/dog.jpg')
  await expect(page.locator('.detector__status')).toBeVisible({ timeout: 15000 })

  await page.locator('.detector__canvas').screenshot({
    path: 'test-results/screenshots/detection-overlay.png',
  })
})
```

### Visual Regression — Snapshot Comparison

For catching unintended UI drift across loops (already referenced in Core Patterns, expanded here):

```ts
test('login page matches baseline', async ({ page }) => {
  await page.goto('/login')
  await expect(page).toHaveScreenshot('login-page.png', {
    maxDiffPixels: 100,
    fullPage: true,
  })
})
```

First run creates the baseline under `e2e/__screenshots__/`. Commit the baseline to the repo.
Update it deliberately when a design change is intentional:
```bash
npx playwright test --update-snapshots
```

Mask dynamic content (timestamps, avatars, ids) that would otherwise cause false failures:
```ts
await expect(page).toHaveScreenshot('dashboard.png', {
  mask: [page.locator('.timestamp'), page.locator('.user-avatar')],
})
```

### Multi-Viewport Screenshots

Useful for responsive layouts — capture the same flow at multiple breakpoints in one test:

```ts
const viewports = [
  { name: 'mobile', width: 375, height: 667 },
  { name: 'tablet', width: 768, height: 1024 },
  { name: 'desktop', width: 1440, height: 900 },
]

for (const vp of viewports) {
  test(`upload flow at ${vp.name}`, async ({ page }) => {
    await page.setViewportSize({ width: vp.width, height: vp.height })
    await page.goto('/')
    await page.screenshot({
      path: `test-results/screenshots/upload-${vp.name}.png`,
      fullPage: true,
    })
  })
}
```

### Uploading Screenshots as CI Artifacts

Add this to every Playwright job in the GitHub Actions matrix (see below) so screenshots are
downloadable from the run summary even when tests pass:

```yaml
- run: npx playwright test
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: screenshots-${{ matrix.browser }}
    path: test-results/screenshots/
    retention-days: 14
- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: playwright-report-${{ matrix.browser }}
    path: playwright-report/
    retention-days: 7
```

### Loop Integration — Screenshots

When `playwright` is in the stack and a Plan step has `UI? YES`:
1. Every new screen or component gets at least one explicit full-page or element screenshot
   saved to `test-results/screenshots/` — not just relied-on failure captures
2. If the step's `DONE WHEN` criterion is visual, the Verify phase should reference the
   screenshot path as evidence, not just a pass/fail assertion
3. For components with a defined visual design (anything that went through
   `references/frontend-design.md`), add a `toHaveScreenshot` baseline so future loops can't
   silently regress the design
4. Mask any dynamic content (timestamps, random data, avatars) before snapshotting

## Playwright — Flutter Web

```ts
// e2e/flutter-app.spec.ts
import { test, expect } from '@playwright/test'

test('Flutter Web app loads and navigates', async ({ page }) => {
  await page.goto('/')
  // Flutter renders to canvas — use semantics tree, not DOM selectors
  await page.waitForSelector('flt-semantics-host', { state: 'attached' })
  await page.getByRole('button', { name: 'Get Started' }).click()
  await expect(page.getByText('Dashboard')).toBeVisible()
})
```

Flutter Web tip: enable `--web-renderer html` in dev/CI builds for reliable semantics-based
Playwright selectors. CanvasKit renderer is harder to target with accessibility selectors.

---

## Playwright — Blazor

```ts
// e2e/blazor-app.spec.ts
import { test, expect } from '@playwright/test'

test('Blazor Server app loads dashboard', async ({ page }) => {
  await page.goto('/dashboard')
  // Wait for SignalR circuit to connect
  await page.waitForSelector('[data-blazor-loaded]', { state: 'attached' })
  await expect(page.getByText('Welcome')).toBeVisible()
})

test('Blazor form submission with anti-forgery token', async ({ page }) => {
  await page.goto('/settings')
  await page.getByLabel('Display Name').fill('New Name')
  await page.getByRole('button', { name: 'Save' }).click()
  await expect(page.getByText('Saved successfully')).toBeVisible()
})
```

---

## GitHub Actions — Matrix Builds

### Multi-OS, Multi-Version Backend Matrix (Node.js)
```yaml
name: Backend CI
on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

### Multi-Browser Playwright Matrix
```yaml
name: E2E Tests
on: [push, pull_request]

jobs:
  e2e:
    strategy:
      fail-fast: false
      matrix:
        browser: [chromium, firefox, webkit]
        shard: [1/4, 2/4, 3/4, 4/4]
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps ${{ matrix.browser }}
      - run: npx playwright test --project=${{ matrix.browser }} --shard=${{ matrix.shard }}
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report-${{ matrix.browser }}-${{ strategy.job-index }}
          path: playwright-report/
          retention-days: 7
```

### .NET Multi-Target Matrix (Blazor / WinUI / DaaS APIs)
```yaml
name: .NET CI
on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet-version: ['8.0.x']
        exclude:
          # WinUI only builds on Windows
          - os: ubuntu-latest
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: ${{ matrix.dotnet-version }} }
      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --no-build --verbosity normal
```

### Python Multi-Version Matrix (FastAPI / AIaaS)
```yaml
name: Python CI
on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        python-version: ['3.11', '3.12', '3.13']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: pip install -r requirements-dev.txt
      - run: pytest --cov=app --cov-report=xml
      - uses: codecov/codecov-action@v4
```

### Combined Full-Stack Pipeline (lint → unit → e2e → build)
```yaml
name: Full Pipeline
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint

  unit-test:
    needs: lint
    strategy:
      matrix:
        workspace: [api, web, shared]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm test --workspace=${{ matrix.workspace }}

  e2e-test:
    needs: unit-test
    strategy:
      matrix:
        browser: [chromium, webkit]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps ${{ matrix.browser }}
      - run: npx playwright test --project=${{ matrix.browser }}

  build:
    needs: e2e-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
```

---

## Sharding Strategy for Large E2E Suites

When the e2e suite exceeds ~5 minutes on a single runner, shard it:

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/4
```

Merge HTML reports from all shards after the matrix completes:
```yaml
  merge-reports:
    needs: e2e
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { pattern: playwright-report-*, merge-multiple: true }
      - run: npx playwright merge-reports --reporter html ./all-reports
      - uses: actions/upload-artifact@v4
        with: { name: merged-report, path: playwright-report/ }
```

---

## Loop Integration

When `playwright` is in the stack and `UI? YES`:
1. Generate Page Object classes alongside the UI components in the same Execute step
2. Write at least one happy-path and one failure-path test per new flow
3. Add the relevant matrix CI job from this file to `.github/workflows/`
4. Verify phase checks: do new UI components have a corresponding e2e test? Flag as a gap if not.
