# Vitest — Fast Test Runner cho ESM và Vite Projects

> Vitest là test runner thế hệ mới, powered by Vite — tốc độ nhanh hơn Jest đáng kể, native ESM support, và API tương thích Jest. Phổ biến trong NestJS, Vite, và TypeScript-first projects.

## Mục Lục

1. [Vitest Là Gì](#vitest-là-gì)
2. [Vitest vs Jest](#vitest-vs-jest)
3. [Cài Đặt và Cấu Hình](#cài-đặt-và-cấu-hình)
4. [Viết Test với Vitest](#viết-test-với-vitest)
5. [Mocking với vi](#mocking-với-vi)
6. [Watch Mode và UI](#watch-mode-và-ui)
7. [In-Source Testing](#in-source-testing)
8. [Workspace Mode — Monorepo](#workspace-mode--monorepo)
9. [Migration từ Jest](#migration-từ-jest)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vitest Là Gì

Vitest được tạo bởi team Vite, chia sẻ cùng transformation pipeline (esbuild/SWC) — không cần compile riêng như Jest + ts-jest.

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Vite-powered** | Hot Module Replacement (HMR — Thay Module Nóng) cho tests |
| **ESM native** | Không cần workarounds cho `import/export` |
| **Jest-compatible API** | `describe`, `it`, `expect` — migration dễ |
| **TypeScript out-of-box** | Không cần ts-jest |
| **Vitest UI** | Web UI xem tests, coverage, filter |
| **Benchmark mode** | Built-in performance benchmarking |

---

## Vitest vs Jest

| Tiêu Chí | Jest | Vitest |
| -------- | ---- | ------ |
| **Tốc độ** | Chậm hơn (Babel/ts-jest transform) | Nhanh 2–5x (esbuild) |
| **ESM** | Cần config phức tạp | Native support |
| **Ecosystem** | Rất mature, nhiều plugins | Đang phát triển nhanh |
| **Config** | `jest.config.js` | `vitest.config.ts` (extends Vite config) |
| **Mocking API** | `jest.fn()`, `jest.mock()` | `vi.fn()`, `vi.mock()` |
| **Snapshot** | Built-in | Built-in (tương thích) |
| **Watch mode** | Có | Có + HMR nhanh hơn |
| **Phổ biến phỏng vấn** | ⭐⭐⭐ | ⭐⭐ (đang tăng) |

**Khi chọn Vitest:** Project dùng Vite, NestJS mới, ESM-only, cần tốc độ CI.  
**Khi chọn Jest:** Legacy codebase, team đã quen Jest, cần ecosystem plugins cụ thể.

---

## Cài Đặt và Cấu Hình

```bash
npm install -D vitest @vitest/coverage-v8
```

### vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,                    // Không cần import describe/it/expect
    environment: 'node',              // 'node' cho backend, 'jsdom' cho frontend
    include: ['src/**/*.{test,spec}.ts'],
    exclude: ['node_modules', 'dist'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 70,
        functions: 70,
        branches: 70,
        statements: 70,
      },
    },
    setupFiles: ['./src/test/setup.ts'],
    testTimeout: 10000,
    hookTimeout: 10000,
  },
});
```

### package.json Scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  }
}
```

### Setup File (Global Mocks)

```typescript
// src/test/setup.ts
import { beforeEach, vi } from 'vitest';

beforeEach(() => {
  vi.clearAllMocks();
});

// Mock environment variables
vi.stubEnv('NODE_ENV', 'test');
vi.stubEnv('JWT_SECRET', 'test-secret-key');
```

### TypeScript — globals: true

```json
// tsconfig.json
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

---

## Viết Test với Vitest

```typescript
// src/services/price.service.ts
export function applyDiscount(price: number, discountPercent: number): number {
  if (discountPercent < 0 || discountPercent > 100) {
    throw new Error('Invalid discount percentage');
  }
  return price * (1 - discountPercent / 100);
}
```

```typescript
// src/services/price.service.test.ts
import { describe, it, expect } from 'vitest';
import { applyDiscount } from './price.service';

describe('applyDiscount', () => {
  it('áp dụng discount 10%', () => {
    expect(applyDiscount(100, 10)).toBe(90);
  });

  it('không discount khi percent = 0', () => {
    expect(applyDiscount(100, 0)).toBe(100);
  });

  it('throw error khi discount âm', () => {
    expect(() => applyDiscount(100, -5)).toThrow('Invalid discount percentage');
  });

  it('throw error khi discount > 100', () => {
    expect(() => applyDiscount(100, 150)).toThrow('Invalid discount percentage');
  });
});
```

### Parameterized Tests — test.each

```typescript
import { describe, it, expect } from 'vitest';

describe('validateEmail', () => {
  it.each([
    ['valid@example.com', true],
    ['user.name+tag@domain.co.uk', true],
    ['invalid', false],
    ['@missing-local.com', false],
    ['missing-domain@', false],
    ['', false],
  ])('validateEmail("%s") => %s', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});
```

---

## Mocking với vi

Vitest dùng prefix `vi` thay `jest` — API tương tự.

### vi.fn() — Mock Function

```typescript
import { vi, describe, it, expect } from 'vitest';

const mockFetch = vi.fn();
mockFetch.mockResolvedValue({ data: 'test' });

it('calls fetch with correct URL', async () => {
  await fetchData(mockFetch, '/api/users');
  expect(mockFetch).toHaveBeenCalledWith('/api/users');
});
```

### vi.mock() — Mock Module

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest';

vi.mock('../repositories/user.repository', () => ({
  findById: vi.fn(),
  create: vi.fn(),
}));

import { findById } from '../repositories/user.repository';
import { getUserProfile } from './user.service';

describe('getUserProfile', () => {
  beforeEach(() => {
    vi.mocked(findById).mockReset();
  });

  it('returns user profile', async () => {
    vi.mocked(findById).mockResolvedValue({
      id: '1',
      email: 'test@example.com',
      name: 'Test User',
    });

    const profile = await getUserProfile('1');

    expect(profile.email).toBe('test@example.com');
    expect(findById).toHaveBeenCalledWith('1');
  });
});
```

### vi.spyOn() — Spy

```typescript
import { vi } from 'vitest';
import * as logger from '../utils/logger';

const errorSpy = vi.spyOn(logger, 'error').mockImplementation(() => {});

// ... test ...

errorSpy.mockRestore();
```

### vi.hoisted() — Hoisting cho vi.mock

```typescript
// Vitest không auto-hoist như Jest — dùng vi.hoisted cho variables trong mock factory
const { mockSendEmail } = vi.hoisted(() => ({
  mockSendEmail: vi.fn(),
}));

vi.mock('../email.service', () => ({
  sendEmail: mockSendEmail,
}));
```

### Mock Timers

```typescript
import { vi, beforeEach, afterEach, it, expect } from 'vitest';

beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});

it('retries after delay', async () => {
  const retryFn = vi.fn();
  scheduleRetry(retryFn, 3000);

  vi.advanceTimersByTime(3000);

  expect(retryFn).toHaveBeenCalledTimes(1);
});
```

---

## Watch Mode và UI

### Watch Mode

```bash
vitest                    # Watch mode — re-run khi file thay đổi
vitest --changed          # Chỉ chạy tests liên quan files đã thay đổi
vitest src/services       # Filter theo path
vitest -t "applyDiscount" # Filter theo test name
```

### Vitest UI

```bash
npm install -D @vitest/ui
vitest --ui
```

Mở browser tại `http://localhost:51234` — xem test tree, filter, re-run, coverage visualization.

---

## In-Source Testing

Vitest hỗ trợ viết test **ngay trong source file** — hữu ích cho utility functions.

```typescript
// src/utils/string.ts
export function capitalize(str: string): string {
  if (!str) return '';
  return str.charAt(0).toUpperCase() + str.slice(1);
}

// In-source test — chỉ chạy khi import.meta.vitest
if (import.meta.vitest) {
  const { it, expect } = import.meta.vitest;

  it('capitalizes first letter', () => {
    expect(capitalize('hello')).toBe('Hello');
  });

  it('handles empty string', () => {
    expect(capitalize('')).toBe('');
  });
}
```

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    includeSource: ['src/**/*.{ts,js}'],
  },
});
```

---

## Workspace Mode — Monorepo

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  'packages/api/vitest.config.ts',
  'packages/shared/vitest.config.ts',
  'packages/worker/vitest.config.ts',
]);
```

```bash
vitest --workspace vitest.workspace.ts
```

Mỗi package có config riêng — shared setup, different environments.

---

## Migration từ Jest

### Find & Replace

| Jest | Vitest |
| ---- | ------ |
| `jest.fn()` | `vi.fn()` |
| `jest.mock()` | `vi.mock()` |
| `jest.spyOn()` | `vi.spyOn()` |
| `jest.clearAllMocks()` | `vi.clearAllMocks()` |
| `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| `@jest/globals` | `vitest` |

### Codemod Tự Động

```bash
npx @vitest/codemod jest-to-vitest src/
```

### Khác Biệt Cần Lưu Ý

1. **Module hoisting:** Vitest không auto-hoist `jest.mock()` — dùng `vi.hoisted()`
2. **Globals:** Cần `globals: true` + `types: ["vitest/globals"]` hoặc import explicit
3. **jest.config.js → vitest.config.ts:** Cấu trúc khác, extends Vite config
4. **@types/jest:** Thay bằng `vitest` types

---

## Best Practices

### NestJS + Vitest

```typescript
// nest-cli.json hoặc package.json
{
  "scripts": {
    "test": "vitest run",
    "test:e2e": "vitest run --config vitest.e2e.config.ts"
  }
}
```

```typescript
// vitest.config.ts cho NestJS
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  plugins: [swc.vite({ module: { type: 'es6' } })],
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./test/setup.ts'],
  },
});
```

### Performance Tips

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: 'forks',           // 'threads' | 'forks' — forks ổn định hơn với native modules
    poolOptions: {
      forks: { singleFork: true },  // Cho integration tests dùng shared DB
    },
    isolate: false,          // Tắt isolation nếu tests không conflict — nhanh hơn
  },
});
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao Vitest nhanh hơn Jest?**  
A: Dùng esbuild/SWC transform thay Babel, Vite's module graph caching, HMR cho watch mode, parallel execution tối ưu hơn.

**Q: Có thể dùng Vitest với Express không?**  
A: Có — `environment: 'node'`, test logic và Supertest integration bình thường.

**Q: Vitest có thay thế hoàn toàn Jest?**  
A: Với projects mới (ESM, Vite, NestJS) — có. Legacy Jest projects với nhiều custom config có thể migration tốn effort.

**Q: `vi.mocked()` là gì?**  
A: Type helper — cast mocked function về đúng type với mock methods (`mockResolvedValue`, etc.).

**Q: Làm sao mock `import.meta.env`?**  
A: `vi.stubEnv('KEY', 'value')` hoặc define trong `vitest.config.ts` → `env: { KEY: 'value' }`.

---

**Tiếp theo:** [3-supertest.md](./3-supertest.md) — HTTP assertion testing cho API endpoints
