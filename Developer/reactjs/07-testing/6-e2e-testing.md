# 6 — E2E Testing với Playwright và Cypress (Kiểm Thử Đầu Cuối)

> **E2E Testing** — End-to-End Testing (Kiểm Thử Đầu Cuối) — là kiểm thử toàn bộ luồng người dùng trên trình duyệt thực (hoặc gần thực). Không mock API, không mock modules — ứng dụng chạy như trong production. **Playwright** và **Cypress** là hai công cụ E2E phổ biến nhất cho React applications.

---

## 🎯 Mục Tiêu

- [ ] Phân biệt E2E testing với unit và integration testing
- [ ] Cài đặt và cấu hình **Playwright** — công cụ E2E hiện đại từ Microsoft
- [ ] Viết **test scripts** (kịch bản kiểm thử) với Playwright API
- [ ] Sử dụng **Page Object Model** — POM (Mô Hình Đối Tượng Trang)
- [ ] Cài đặt và sử dụng **Cypress** — công cụ E2E phổ biến
- [ ] Chạy E2E tests trong **CI/CD pipeline** (đường dẫn tích hợp/triển khai liên tục)
- [ ] **Visual testing** (kiểm thử giao diện trực quan) và screenshot comparison

---

## 🆚 Playwright vs Cypress

| Tiêu Chí | Playwright | Cypress |
| -------- | ---------- | ------- |
| **Ngôn ngữ** | TypeScript, JS, Python, Java, C# | JavaScript, TypeScript |
| **Trình duyệt** | Chrome, Firefox, Safari, Edge | Chrome, Firefox, Edge |
| **Kiến trúc** | Out-of-process (ngoài tiến trình browser) | In-process (trong tiến trình browser) |
| **Tốc độ** | Nhanh hơn (parallel native) | Chậm hơn (serial mặc định) |
| **API** | async/await tự nhiên | Custom chain (jQuery-like) |
| **iframe** | ✅ Hỗ trợ đầy đủ | ⚠️ Hạn chế |
| **Multi-tab** | ✅ | ❌ |
| **Network intercept** | ✅ `route()` | ✅ `intercept()` |
| **Debug** | Trace viewer, VS Code extension | Time-travel debug trong browser |
| **CI/CD** | Xuất sắc | Cypress Cloud (tốn phí cho parallel) |
| **Khuyến nghị** | Dự án mới, cần multi-browser | Dự án đã có Cypress, muốn DX tốt |

---

## 🎭 PLAYWRIGHT

### Cài Đặt

```bash
# Cài Playwright và trình duyệt
npm install --save-dev @playwright/test
npx playwright install          # Cài tất cả browsers
npx playwright install chromium # Chỉ cài Chromium (nhẹ hơn cho CI)

# Tạo config tự động
npx playwright init
```

### playwright.config.ts — Cấu Hình

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // Thư mục chứa test files
  testDir: './e2e',
  
  // Timeout (thời gian chờ tối đa) cho mỗi test
  timeout: 30 * 1000, // 30 giây
  
  // Số lần thử lại khi test fail (chỉ trong CI)
  retries: process.env.CI ? 2 : 0,
  
  // Chạy test song song
  fullyParallel: true,
  workers: process.env.CI ? 1 : undefined,
  
  // Reporter (bộ báo cáo)
  reporter: [
    ['html'],           // HTML report chi tiết
    ['list'],           // List trong terminal
    ['junit', { outputFile: 'test-results/results.xml' }], // Cho CI
  ],
  
  // Config chung cho tất cả tests
  use: {
    // URL cơ sở — thay thế cho URL đầy đủ
    baseURL: 'http://localhost:3000',
    
    // Chụp screenshot khi test fail
    screenshot: 'only-on-failure',
    
    // Ghi video khi test fail
    video: 'retain-on-failure',
    
    // Trace để debug
    trace: 'on-first-retry',
    
    // Locale và timezone
    locale: 'vi-VN',
    timezoneId: 'Asia/Ho_Chi_Minh',
  },
  
  // Test trên nhiều browsers
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
    // Mobile viewport
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
  
  // Tự động khởi động dev server trước khi chạy tests
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Viết Playwright Tests

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

// Test: Đăng nhập thành công
test('should login with valid credentials', async ({ page }) => {
  // Điều hướng đến trang đăng nhập
  await page.goto('/login');
  
  // Điền form
  await page.getByLabel('Email').fill('an@example.com');
  await page.getByLabel('Mật khẩu').fill('password123');
  
  // Click nút đăng nhập
  await page.getByRole('button', { name: /đăng nhập/i }).click();
  
  // Đợi chuyển trang và kiểm tra
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByRole('heading', { name: /xin chào, an/i })).toBeVisible();
});

// Test: Hiển thị lỗi với credentials sai
test('should show error with invalid credentials', async ({ page }) => {
  await page.goto('/login');
  
  await page.getByLabel('Email').fill('wrong@example.com');
  await page.getByLabel('Mật khẩu').fill('wrongpassword');
  await page.getByRole('button', { name: /đăng nhập/i }).click();
  
  // Kiểm tra message lỗi
  await expect(page.getByRole('alert')).toContainText('Email hoặc mật khẩu không đúng');
  
  // Vẫn ở trang login
  await expect(page).toHaveURL('/login');
});
```

### Locators — Cách Tìm Phần Tử

```typescript
// Playwright Locators — theo thứ tự ưu tiên (tương tự RTL)

// 1. getByRole — ưu tiên nhất (accessible)
page.getByRole('button', { name: 'Submit' });
page.getByRole('textbox', { name: 'Email' });
page.getByRole('heading', { level: 1 });

// 2. getByLabel — form elements
page.getByLabel('Email');

// 3. getByPlaceholder
page.getByPlaceholder('Nhập email...');

// 4. getByText
page.getByText('Xin chào');
page.getByText(/xin chào/i); // Regex

// 5. getByAltText — images
page.getByAltText('Logo');

// 6. getByTitle
page.getByTitle('Close dialog');

// 7. getByTestId — phương án cuối
page.getByTestId('submit-button');

// CSS selector — tránh dùng nếu có thể
page.locator('.btn-primary');
page.locator('#email-input');

// Kết hợp locators
page.getByRole('listitem').filter({ hasText: 'Sản phẩm A' });
page.locator('ul').getByRole('listitem').first();
```

### Actions — Tương Tác

```typescript
// Click
await page.getByRole('button').click();
await page.getByRole('button').dblclick(); // Double click
await page.getByRole('button').click({ button: 'right' }); // Right click

// Nhập text
await page.getByLabel('Search').fill('react testing');   // Thay thế toàn bộ
await page.getByLabel('Search').type('react testing');    // Gõ từng ký tự (chậm hơn)
await page.getByLabel('Search').press('Enter');           // Nhấn phím

// Xóa và điền lại
await page.getByLabel('Email').clear();
await page.getByLabel('Email').fill('new@example.com');

// Select
await page.getByLabel('Màu sắc').selectOption('blue');
await page.getByLabel('Tags').selectOption(['tag1', 'tag2']); // Multi-select

// Checkbox và Radio
await page.getByLabel('Đồng ý').check();
await page.getByLabel('Đồng ý').uncheck();

// Upload file
await page.getByLabel('Avatar').setInputFiles('/path/to/image.jpg');

// Hover
await page.getByRole('button').hover();

// Drag and drop
await page.getByTestId('drag-item').dragTo(page.getByTestId('drop-zone'));

// Scroll
await page.getByRole('list').scrollIntoViewIfNeeded();
```

### Assertions — Khẳng Định

```typescript
// Element assertions
await expect(page.getByRole('button')).toBeVisible();
await expect(page.getByRole('button')).toBeHidden();
await expect(page.getByRole('button')).toBeEnabled();
await expect(page.getByRole('button')).toBeDisabled();
await expect(page.getByRole('checkbox')).toBeChecked();

// Text assertions
await expect(page.getByRole('heading')).toContainText('Tiêu đề');
await expect(page.getByRole('heading')).toHaveText('Tiêu đề chính xác');

// Value assertions
await expect(page.getByLabel('Email')).toHaveValue('an@example.com');

// Count assertions
await expect(page.getByRole('listitem')).toHaveCount(5);

// URL assertions
await expect(page).toHaveURL('/dashboard');
await expect(page).toHaveURL(/\/dashboard/);

// Title assertions
await expect(page).toHaveTitle(/React App/);

// Screenshot assertions (visual testing)
await expect(page).toHaveScreenshot('homepage.png');
await expect(page.getByRole('dialog')).toHaveScreenshot();
```

### Page Object Model — POM (Mô Hình Đối Tượng Trang)

```typescript
// e2e/pages/LoginPage.ts — Page Object
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
    this.passwordInput = page.getByLabel('Mật khẩu');
    this.submitButton = page.getByRole('button', { name: /đăng nhập/i });
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

// e2e/auth.spec.ts — Dùng Page Object
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test('full login flow', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);
  
  await loginPage.goto();
  await loginPage.login('an@example.com', 'password123');
  
  await dashboardPage.expectWelcome('An');
});
```

### Network Interception — Chặn Network

```typescript
test('should handle API error gracefully', async ({ page }) => {
  // Intercept và override API response
  await page.route('/api/users', route => {
    route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Server Error' }),
    });
  });
  
  await page.goto('/users');
  
  await expect(page.getByRole('alert')).toContainText('Lỗi server');
});

// Chặn và theo dõi requests
test('should call correct API endpoint', async ({ page }) => {
  const apiPromise = page.waitForRequest('/api/search*');
  
  await page.goto('/');
  await page.getByRole('searchbox').fill('react');
  await page.getByRole('searchbox').press('Enter');
  
  const request = await apiPromise;
  expect(request.url()).toContain('q=react');
});
```

### Fixtures — Dữ Liệu Test Cố Định

```typescript
// e2e/fixtures.ts — Custom fixtures
import { test as base, Page } from '@playwright/test';

type Fixtures = {
  authenticatedPage: Page;
};

export const test = base.extend<Fixtures>({
  // Fixture: trang đã đăng nhập
  authenticatedPage: async ({ page }, use) => {
    // Setup: đăng nhập trước
    await page.goto('/login');
    await page.getByLabel('Email').fill('an@example.com');
    await page.getByLabel('Mật khẩu').fill('password123');
    await page.getByRole('button', { name: /đăng nhập/i }).click();
    await page.waitForURL('/dashboard');
    
    // Chạy test
    await use(page);
    
    // Teardown: đăng xuất sau
    await page.getByRole('button', { name: /đăng xuất/i }).click();
  },
});

// Dùng fixture trong test
// e2e/dashboard.spec.ts
import { test, expect } from './fixtures';

test('should display user stats on dashboard', async ({ authenticatedPage: page }) => {
  await expect(page.getByText('Tổng đơn hàng')).toBeVisible();
});
```

---

## 🌲 CYPRESS

### Cài Đặt

```bash
npm install --save-dev cypress

# Mở Cypress lần đầu (tạo cấu trúc thư mục)
npx cypress open
```

### cypress.config.ts

```typescript
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    
    // Thư mục chứa test files
    specPattern: 'cypress/e2e/**/*.cy.{ts,tsx}',
    
    // Timeouts
    defaultCommandTimeout: 5000,  // 5s cho mỗi command
    pageLoadTimeout: 30000,       // 30s cho page load
    
    // Screenshots và videos
    screenshotOnRunFailure: true,
    video: false, // Tắt video để test nhanh hơn
    
    setupNodeEvents(on, config) {
      // Plugins, tasks
    },
  },
  
  viewportWidth: 1280,
  viewportHeight: 720,
});
```

### Viết Cypress Tests

```typescript
// cypress/e2e/checkout.cy.ts
describe('Checkout Flow', () => {
  beforeEach(() => {
    // Đăng nhập trước mỗi test (dùng command tùy chỉnh)
    cy.login('an@example.com', 'password123');
    cy.visit('/products');
  });

  it('should complete full checkout', () => {
    // Thêm sản phẩm vào giỏ hàng
    cy.get('[data-testid="product-card"]').first()
      .find('button[aria-label="Thêm vào giỏ"]')
      .click();
    
    // Kiểm tra badge giỏ hàng
    cy.get('[data-testid="cart-count"]').should('contain', '1');
    
    // Vào trang checkout
    cy.get('[data-testid="cart-icon"]').click();
    cy.contains('button', 'Tiến hành thanh toán').click();
    
    // Điền thông tin giao hàng
    cy.get('#shipping-name').type('Nguyen Van A');
    cy.get('#shipping-address').type('123 Nguyen Trai, Ha Noi');
    cy.get('#shipping-phone').type('0901234567');
    
    // Chọn phương thức thanh toán
    cy.contains('label', 'Thanh toán khi nhận hàng').click();
    
    // Đặt hàng
    cy.contains('button', 'Đặt hàng').click();
    
    // Kiểm tra trang xác nhận
    cy.url().should('include', '/order-confirmation');
    cy.contains('Đặt hàng thành công!').should('be.visible');
  });

  it('should show error for out-of-stock item', () => {
    // Intercept API trả về lỗi hết hàng
    cy.intercept('POST', '/api/cart/add', {
      statusCode: 409,
      body: { error: 'Sản phẩm đã hết hàng' },
    });
    
    cy.get('[data-testid="product-card"]').first()
      .find('button[aria-label="Thêm vào giỏ"]')
      .click();
    
    cy.get('[role="alert"]').should('contain', 'Sản phẩm đã hết hàng');
  });
});
```

### Custom Commands — Lệnh Tùy Chỉnh

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      addToCart(productId: string): Chainable<void>;
    }
  }
}

// Đăng nhập qua API thay vì UI (nhanh hơn)
Cypress.Commands.add('login', (email: string, password: string) => {
  cy.request('POST', '/api/auth/login', { email, password })
    .then(({ body }) => {
      // Lưu token vào localStorage
      window.localStorage.setItem('auth_token', body.token);
    });
});

Cypress.Commands.add('addToCart', (productId: string) => {
  cy.request('POST', '/api/cart/add', { productId });
});

// Dùng trong tests
cy.login('an@example.com', 'password123');
cy.addToCart('product-123');
```

---

## 🔄 CI/CD — Tích Hợp Pipeline

### GitHub Actions Với Playwright

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium
      
      - name: Build app
        run: npm run build
      
      - name: Run Playwright tests
        run: npx playwright test --project=chromium
        env:
          CI: true
      
      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: failure() # Chỉ upload khi fail
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

---

## 📋 Best Practices — Thực Hành Tốt Nhất

### Nguyên Tắc Viết E2E Tests

```
✅ Test happy path (luồng thành công) và critical error paths
✅ Dùng data-testid cho elements khó tìm bằng role/text
✅ Dùng Page Object Model để tổ chức code
✅ Reset state giữa các tests (database, localStorage)
✅ Sử dụng fixtures để tái sử dụng setup code
✅ Chạy E2E trên CI với môi trường isolated (cô lập)

❌ Không test mọi thứ với E2E — quá chậm và tốn kém
❌ Không dùng sleep/wait cứng nhắc (cy.wait(2000))
❌ Không phụ thuộc vào thứ tự chạy tests
❌ Không share state giữa các tests
```

### Khi Nào Chạy E2E Tests

```
Development:   Chạy locally khi implement feature mới
Pre-commit:    Không khuyến nghị (quá chậm)
CI/CD:
  PR:          Chạy subset tests quan trọng nhất (smoke tests)
  Merge main:  Chạy toàn bộ E2E test suite
  Nightly:     Chạy full suite bao gồm edge cases
```

---

## 🔗 Điều Hướng

- **Trước đó:** [5-vitest.md](./5-vitest.md) — Vitest cho Vite projects
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Tiếp theo:** [08-styling/](../08-styling/) — Tạo kiểu dáng UI
- **Chỉ mục:** [INDEX.md](../INDEX.md)
