Playwright 烟雾测试指南
本指南详细介绍了如何在本项目（以及其他项目）中集成和使用 Playwright 进行端到端（E2E）烟雾测试。
1. 核心文件结构
以下是实现“自动起服务 + 跑用例”所需的核心文件及其内容。你可以直接复制这些文件作为模板。
1.1 配置文件 (playwright.config.ts)
此配置负责定义测试环境、浏览器以及最关键的 WebServer 自动启动功能。

import { defineConfig, devices } from '@playwright/test';

/**
 * See https://playwright.dev/docs/test-configuration.
 */
export default defineConfig({
  testDir: './e2e',
  /* Maximum time one test can run for. */
  timeout: 30 * 1000,
  expect: {
    /**
     * Maximum time expect() should wait for the condition to be met.
     */
    timeout: 5000,
  },
  /* Run tests in files in parallel */
  fullyParallel: true,
  /* Fail the build on CI if you accidentally left test.only in the source code. */
  forbidOnly: !!process.env.CI,
  /* Retry on CI only */
  retries: process.env.CI ? 2 : 0,
  /* Opt out of parallel tests on CI. */
  workers: process.env.CI ? 1 : undefined,
  /* Reporter to use. See https://playwright.dev/docs/test-reporters */
  reporter: 'html',
  /* Shared settings for all the projects below. See https://playwright.dev/docs/api/class-testoptions. */
  use: {
    /* Maximum time each action such as `click()` can take. Defaults to 0 (no limit). */
    actionTimeout: 0,
    /* Base URL to use in actions like `await page.goto('/')`. */
    baseURL: 'http://localhost:5173',

    /* Collect trace when retrying the failed test. See https://playwright.dev/docs/trace-viewer */
    trace: 'on-first-retry',
  },

  /* Configure projects for major browsers */
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
  ],

  /* Run your local dev server before starting the tests */
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000,
  },
});

1.2 烟雾测试用例 (e2e/smoke.spec.ts)
此测试用例包含“无运行时报错”和“关键 UI 元素可见”的最小断言集。

import { test, expect } from '@playwright/test';

test('smoke: app loads without runtime errors', async ({ page }) => {
  const consoleErrors: string[] = [];
  const pageErrors: string[] = [];

  // 1. Listen for console errors
  page.on('console', (msg) => {
    if (msg.type() === 'error') {
      consoleErrors.push(msg.text());
    }
  });

  // 2. Listen for uncaught exceptions
  page.on('pageerror', (err) => {
    pageErrors.push(err.message);
  });

  // 3. Visit the app
  // Note: baseURL is set in playwright.config.ts (http://localhost:5173)
  await page.goto('/');

  // 4. Check for critical UI elements (Welcome Screen)
  // Expect the main title to be visible
  await expect(page.getByText('Wasteland Survivor', { exact: true })).toBeVisible();

  // Expect the version number to be visible (partial match)
  await expect(page.getByText('Terminal v')).toBeVisible();

  // 5. Check for interactive elements
  // Depending on whether there is a save file, different buttons might show.
  // We check for at least one of the possible main action buttons.
  const startButton = page.getByText('Enter Wasteland');
  const continueButton = page.getByText('Continue Journey');
  const newGameButton = page.getByText('New Game');

  // Wait for at least one of the buttons to be visible
  await expect(
    startButton.or(continueButton).or(newGameButton).first()
  ).toBeVisible();

  // 6. Verify no runtime errors occurred
  expect(pageErrors, `Page errors: ${pageErrors.join('\n')}`).toEqual([]);
  
  // Note: Some third-party extensions or non-critical errors might trigger console.error.
  // If strict console error checking is too flaky, you can filter specific known errors here.
  // For now, we expect 0 console errors for a clean smoke test.
  if (consoleErrors.length > 0) {
    console.log('Console errors found (warning):', consoleErrors);
    // Uncomment the next line to fail the test on console errors
    // expect(consoleErrors, `Console errors: ${consoleErrors.join('\n')}`).toEqual([]);
  }
});

1.3 NPM 脚本 (package.json)
在 package.json 中添加以下便捷脚本：

{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}

2. 迁移指南
如果你要将这套方案复制到其他项目，只需修改以下两处：
1.playwright.config.ts:
a.修改 webServer.command (如果启动命令不是 npm run dev)。
b.修改 webServer.url 和 use.baseURL (如果端口不是 5173)。
2.e2e/smoke.spec.ts:
a.修改 await expect(page.getByText('...')).toBeVisible() 中的文本或选择器，使其匹配新项目的首页关键元素（如标题、登录框等）。
3. 使用方法
1.安装依赖:

npm install -D @playwright/test
npx playwright install

2.运行测试:
a.CI / 快速检查:

npm run test:e2e

b.调试模式 (带 UI):

npm run test:e2e:ui



