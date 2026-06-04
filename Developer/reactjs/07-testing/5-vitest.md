# 5 — Vitest (Kiểm Thử Nhanh Cho Vite)

> **Vitest** là **unit testing framework** (framework kiểm thử đơn vị) được xây dựng cho **Vite** — tận dụng cùng cấu hình và pipeline của Vite để chạy tests cực nhanh. Vitest có API tương thích với Jest nên việc migrate (chuyển đổi) từ Jest sang Vitest gần như không cần đổi code test — chỉ đổi cấu hình.

---

## 🎯 Mục Tiêu

- [ ] Hiểu lý do Vitest nhanh hơn Jest trong Vite projects
- [ ] Cài đặt và cấu hình Vitest trong dự án Vite/React
- [ ] Sử dụng **Vitest UI** — giao diện trực quan cho tests
- [ ] **Migrate** (chuyển đổi) dự án từ Jest sang Vitest
- [ ] Dùng **in-source testing** (kiểm thử trong mã nguồn) — đặc trưng của Vitest
- [ ] Cấu hình **coverage** (độ phủ) với `@vitest/coverage-v8`

---

## ⚡ Tại Sao Vitest Nhanh Hơn?

```
Jest (với CRA/Webpack):
  Test file → Babel transform → Jest runner → JSDOM
  → Mỗi file phải transform riêng, không tận dụng cache Webpack

Vitest (với Vite):
  Test file → Vite pipeline (ESBuild) → Vitest runner → JSDOM
  → Tận dụng ESBuild (viết bằng Go, nhanh hơn Babel 10-100x)
  → Tận dụng Vite module graph → hot reload tests
  → Không cần transpile TypeScript riêng
```

| Chỉ Số | Jest + Babel | Vitest + ESBuild |
| ------ | ------------ | ---------------- |
| **Cold start** (khởi động lạnh) | 5–15s | 1–3s |
| **Watch mode** (chế độ theo dõi) | 1–5s/change | < 1s/change |
| **TypeScript support** | Cần `ts-jest` hoặc `babel-jest` | Native, không cần cấu hình |
| **ESM support** | Phức tạp | Native |
| **Vite config reuse** | ❌ | ✅ |

---

## 📦 Cài Đặt

```bash
# Cài Vitest và dependencies
npm install --save-dev vitest @vitest/ui jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event

# Coverage (độ phủ mã nguồn)
npm install --save-dev @vitest/coverage-v8
```

---

## ⚙️ Cấu Hình

### vite.config.ts — Cấu Hình Vitest Trong Vite

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  
  test: {
    // Môi trường giả lập trình duyệt
    environment: 'jsdom',
    
    // Import jest-dom matchers toàn cục
    setupFiles: ['./src/test-setup.ts'],
    
    // Tự động import describe, test, expect (không cần import)
    globals: true,
    
    // Coverage config
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'src/test-setup.ts',
        '**/*.d.ts',
        '**/*.config.*',
        '**/mocks/**',
      ],
      thresholds: {
        statements: 80,
        branches: 70,
        functions: 80,
        lines: 80,
      },
    },
    
    // Bao gồm/loại trừ files
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    exclude: ['node_modules', 'dist', 'e2e'],
  },
});
```

### tsconfig.json — Thêm Vitest Types

```json
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

### src/test-setup.ts — Setup File

```typescript
// src/test-setup.ts
import '@testing-library/jest-dom';

// Mock global browser APIs nếu cần
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});
```

### package.json — Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

## 🔄 API Tương Thích Với Jest

Vitest hỗ trợ gần như toàn bộ Jest API — hầu hết test files không cần đổi:

```typescript
// Những thứ này hoạt động giống hệt Jest
describe('suite name', () => {
  beforeAll(() => {});
  afterAll(() => {});
  beforeEach(() => {});
  afterEach(() => {});
  
  test('test name', () => {
    expect(1 + 1).toBe(2);
  });
  
  it('alias of test', () => {});
  test.skip('skipped test', () => {});
  test.only('focused test', () => {});
  test.each([...])('parameterized', () => {});
});
```

### vi — Thay Thế jest

```typescript
// Jest                         // Vitest (tương đương)
jest.fn()                    → vi.fn()
jest.spyOn()                 → vi.spyOn()
jest.mock()                  → vi.mock()
jest.useFakeTimers()         → vi.useFakeTimers()
jest.useRealTimers()         → vi.useRealTimers()
jest.clearAllMocks()         → vi.clearAllMocks()
jest.resetAllMocks()         → vi.resetAllMocks()
jest.restoreAllMocks()       → vi.restoreAllMocks()
jest.advanceTimersByTime()   → vi.advanceTimersByTime()
jest.setSystemTime()         → vi.setSystemTime()
jest.requireActual()         → vi.importActual() (async!)
```

---

## 📝 Viết Tests Với Vitest

### Unit Test Cơ Bản

```typescript
// utils/formatters.test.ts
import { describe, test, expect } from 'vitest'; // Hoặc dùng globals: true

import { formatCurrency, formatDate, truncateText } from './formatters';

describe('formatCurrency', () => {
  test('should format VND correctly', () => {
    expect(formatCurrency(100000, 'VND')).toBe('100.000 ₫');
    expect(formatCurrency(1500000, 'VND')).toBe('1.500.000 ₫');
  });

  test('should format USD correctly', () => {
    expect(formatCurrency(99.99, 'USD')).toBe('$99.99');
  });

  test('should return "Miễn phí" for zero', () => {
    expect(formatCurrency(0, 'VND')).toBe('Miễn phí');
  });
});

describe('truncateText', () => {
  test.each([
    { input: 'Hello World', maxLength: 5, expected: 'Hello...' },
    { input: 'Hi',          maxLength: 10, expected: 'Hi' },
    { input: '',            maxLength: 5,  expected: '' },
  ])('truncateText($input, $maxLength) → $expected', ({ input, maxLength, expected }) => {
    expect(truncateText(input, maxLength)).toBe(expected);
  });
});
```

### Mock Với vi

```typescript
// services/userService.test.ts
import { vi, describe, test, expect, beforeEach } from 'vitest';
import { getUserById, createUser } from './userService';
import * as api from '../lib/api';

vi.mock('../lib/api');

const mockApi = vi.mocked(api);

describe('userService', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  test('should fetch user by id', async () => {
    mockApi.get.mockResolvedValue({ data: { id: 1, name: 'An' } });
    
    const user = await getUserById(1);
    
    expect(mockApi.get).toHaveBeenCalledWith('/users/1');
    expect(user).toEqual({ id: 1, name: 'An' });
  });

  test('should handle API error', async () => {
    mockApi.get.mockRejectedValue(new Error('Network Error'));
    
    await expect(getUserById(999)).rejects.toThrow('Network Error');
  });
});
```

---

## 🖥️ Vitest UI — Giao Diện Trực Quan

```bash
# Mở Vitest UI trong browser
npm run test:ui
# → Mở tại http://localhost:51204/__vitest__/
```

**Vitest UI cung cấp:**
- Danh sách tất cả test files và kết quả
- Re-run test khi click
- Xem coverage trực quan ngay trong UI
- Filter tests theo trạng thái (pass, fail, skip)
- Module graph — đồ thị phụ thuộc module

---

## 🔬 In-Source Testing — Kiểm Thử Ngay Trong Mã Nguồn

Vitest cho phép đặt tests ngay trong file source — tính năng độc đáo không có trong Jest:

```typescript
// utils/math.ts
export function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

export function isPrime(n: number): boolean {
  if (n < 2) return false;
  for (let i = 2; i <= Math.sqrt(n); i++) {
    if (n % i === 0) return false;
  }
  return true;
}

// In-source tests — chỉ chạy trong development/test, được tree-shake trong production
if (import.meta.vitest) {
  const { test, expect } = import.meta.vitest;
  
  test('fibonacci(10) = 55', () => {
    expect(fibonacci(10)).toBe(55);
  });
  
  test('isPrime(7) = true', () => {
    expect(isPrime(7)).toBe(true);
    expect(isPrime(4)).toBe(false);
  });
}
```

```typescript
// vite.config.ts — Kích hoạt in-source testing
export default defineConfig({
  test: {
    includeSource: ['src/**/*.{ts,tsx}'], // Bao gồm source files
  },
  define: {
    // Loại bỏ in-source tests khỏi production build
    'import.meta.vitest': 'undefined',
  },
});
```

---

## 📊 Coverage — Độ Phủ Mã Nguồn

```bash
# Chạy và tạo coverage report
npm run test:coverage

# Coverage HTML report mở trong browser
npx vite preview --outDir coverage
```

```typescript
// vite.config.ts
test: {
  coverage: {
    provider: 'v8',         // hoặc 'istanbul'
    reporter: [
      'text',               // Bảng trong terminal
      'html',               // Report HTML chi tiết
      'lcov',               // Cho CI/CD tools
      'json-summary',       // Cho badges
    ],
    
    // Exclude files không cần test
    exclude: [
      'node_modules/',
      'src/test-setup.ts',
      '**/*.config.*',
      '**/types/**',
      '**/mocks/**',
      'src/main.tsx',
    ],
    
    // Fail nếu coverage dưới ngưỡng
    thresholds: {
      global: {
        statements: 80,
        branches: 70,
        functions: 80,
        lines: 80,
      },
      // Per-file thresholds
      'src/utils/': {
        statements: 95,
      },
    },
  },
}
```

---

## 🔀 Migrate Từ Jest Sang Vitest

### Checklist Migration

```
1. Cài đặt Vitest:
   npm install --save-dev vitest @vitest/ui @vitest/coverage-v8

2. Xóa Jest packages:
   npm uninstall jest jest-environment-jsdom @types/jest babel-jest ts-jest

3. Cập nhật vite.config.ts:
   → Thêm test config

4. Đổi jest → vi trong test files:
   → jest.fn() → vi.fn()
   → jest.mock() → vi.mock()
   → jest.spyOn() → vi.spyOn()

5. Cập nhật tsconfig.json:
   → Thêm "types": ["vitest/globals"]

6. Cập nhật package.json scripts:
   → "test": "vitest"

7. Xóa jest.config.ts

8. Chạy tests và fix lỗi nếu có
```

### Tự Động Migrate

```bash
# Dùng codemod tự động (không chính thức, cần review)
npx @vitest/codemod
```

### Những Điểm Khác Biệt Quan Trọng

```typescript
// jest.requireActual() — ĐỒNG BỘ
const { default: actual } = jest.requireActual('../module');

// vi.importActual() — BẤT ĐỒNG BỘ (cần await)
const { default: actual } = await vi.importActual('../module');

// jest.mock() factory với requireActual
jest.mock('../module', () => ({
  ...jest.requireActual('../module'),
  foo: jest.fn(),
}));

// vi.mock() factory với importActual
vi.mock('../module', async () => ({
  ...(await vi.importActual('../module')),
  foo: vi.fn(),
}));
```

---

## 🧪 Test Với React Testing Library

Không cần thay đổi gì — RTL hoạt động giống hệt với Vitest:

```typescript
// Button.test.tsx
import { describe, test, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  test('should call onClick when clicked', async () => {
    const user = userEvent.setup();
    const onClick = vi.fn(); // vi.fn() thay vì jest.fn()
    
    render(<Button onClick={onClick}>Click me</Button>);
    
    await user.click(screen.getByRole('button'));
    
    expect(onClick).toHaveBeenCalledOnce(); // Vitest extra matcher
  });
});
```

### Vitest Extra Matchers

```typescript
// Vitest có thêm một số matchers tiện ích
expect(fn).toHaveBeenCalledOnce();         // Thay vì toHaveBeenCalledTimes(1)
expect(arr).toHaveLength(3);
expect(fn).toHaveBeenCalledBefore(fn2);   // Gọi trước fn2
expect(fn).toHaveBeenCalledAfter(fn2);    // Gọi sau fn2
```

---

## 🔗 Điều Hướng

- **Trước đó:** [4-integration-testing.md](./4-integration-testing.md) — Integration tests với MSW
- **Tiếp theo:** [6-e2e-testing.md](./6-e2e-testing.md) — E2E với Playwright và Cypress
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
