# Jest Basics — Unit Testing, Mocking, Snapshots và Coverage

> Jest là test runner (trình chạy kiểm thử) phổ biến nhất trong hệ sinh thái Node.js — zero-config, built-in mocking, snapshot testing, và coverage reports. Đây là công cụ bắt buộc cho phỏng vấn Backend JavaScript.

## Mục Lục

1. [Jest Là Gì](#jest-là-gì)
2. [Cài Đặt và Cấu Hình](#cài-đặt-và-cấu-hình)
3. [Cấu Trúc Test Cơ Bản](#cấu-trúc-test-cơ-bản)
4. [Matchers — Toán Tử So Sánh](#matchers--toán-tử-so-sánh)
5. [Setup và Teardown Hooks](#setup-và-teardown-hooks)
6. [Mocking với Jest](#mocking-với-jest)
7. [Snapshot Testing](#snapshot-testing)
8. [Test Async Code](#test-async-code)
9. [Code Coverage](#code-coverage)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Jest Là Gì

Jest được phát triển bởi Facebook/Meta, thiết kế cho **JavaScript projects** với focus vào simplicity và developer experience.

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Zero-config** | Chạy ngay với defaults hợp lý |
| **Built-in mocking** | `jest.fn()`, `jest.mock()` không cần thư viện riêng |
| **Snapshot testing** | Capture output, detect unintended changes |
| **Coverage reports** | Istanbul built-in — line, branch, function coverage |
| **Watch mode** | Tự động re-run tests khi file thay đổi |
| **Parallel execution** | Chạy tests song song — tận dụng multi-core |

---

## Cài Đặt và Cấu Hình

```bash
npm install -D jest @types/jest ts-jest
```

### package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

### jest.config.ts (TypeScript Project)

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.ts', '**/*.test.ts', '**/*.spec.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70,
    },
  },
  clearMocks: true,
  restoreMocks: true,
};

export default config;
```

### ESM Support (Node.js Native Modules)

```typescript
// jest.config.ts cho ESM
const config: Config = {
  extensionsToTreatAsEsm: ['.ts'],
  moduleNameMapper: {
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
  transform: {
    '^.+\\.ts$': ['ts-jest', { useESM: true }],
  },
};
```

---

## Cấu Trúc Test Cơ Bản

```typescript
// src/utils/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}
```

```typescript
// src/utils/math.test.ts
import { add, divide } from './math';

describe('Math utilities', () => {
  describe('add', () => {
    it('cộng hai số dương', () => {
      expect(add(2, 3)).toBe(5);
    });

    it('xử lý số âm', () => {
      expect(add(-1, 1)).toBe(0);
    });
  });

  describe('divide', () => {
    it('chia hai số', () => {
      expect(divide(10, 2)).toBe(5);
    });

    it('throw error khi chia cho 0', () => {
      expect(() => divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

### AAA Pattern — Arrange, Act, Assert

```typescript
it('tính discount đúng cho VIP user', () => {
  // Arrange — chuẩn bị data
  const user = { tier: 'VIP', total: 1000 };
  const discountRate = 0.1;

  // Act — thực hiện hành động
  const result = calculateDiscount(user, discountRate);

  // Assert — kiểm tra kết quả
  expect(result).toBe(100);
});
```

---

## Matchers — Toán Tử So Sánh

### Equality Matchers

```typescript
expect(value).toBe(42);              // Strict equality (===)
expect(value).toEqual({ a: 1 });     // Deep equality cho objects/arrays
expect(value).not.toBe(0);
```

### Truthiness

```typescript
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();
```

### Numbers

```typescript
expect(0.1 + 0.2).toBeCloseTo(0.3);  // Floating point
expect(value).toBeGreaterThan(3);
expect(value).toBeLessThanOrEqual(5);
```

### Strings và Arrays

```typescript
expect('hello world').toMatch(/world/);
expect(['apple', 'banana']).toContain('apple');
expect(['a', 'b', 'c']).toHaveLength(3);
```

### Objects

```typescript
expect(user).toHaveProperty('email');
expect(user).toHaveProperty('email', 'test@example.com');
expect(user).toMatchObject({ name: 'John', active: true });
```

### Exceptions

```typescript
expect(() => riskyOperation()).toThrow();
expect(() => riskyOperation()).toThrow(Error);
expect(() => riskyOperation()).toThrow('specific message');
await expect(asyncFn()).rejects.toThrow('error');
```

---

## Setup và Teardown Hooks

```typescript
describe('UserRepository', () => {
  let db: DatabaseConnection;

  beforeAll(async () => {
    // Chạy 1 lần trước tất cả tests trong describe block
    db = await connectTestDatabase();
  });

  afterAll(async () => {
    // Chạy 1 lần sau tất cả tests
    await db.close();
  });

  beforeEach(async () => {
    // Chạy trước mỗi test — reset state
    await db.query('TRUNCATE users CASCADE');
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('tạo user mới', async () => {
    const user = await db.createUser({ email: 'test@example.com' });
    expect(user.id).toBeDefined();
  });
});
```

| Hook | Scope | Use Case |
| ---- | ----- | -------- |
| `beforeAll` | Toàn describe block | Connect DB, start server |
| `afterAll` | Toàn describe block | Close connections, cleanup |
| `beforeEach` | Mỗi test | Reset state, seed data |
| `afterEach` | Mỗi test | Clear mocks, rollback transaction |

---

## Mocking với Jest

### jest.fn() — Mock Function

```typescript
const mockCallback = jest.fn();
mockCallback.mockReturnValue(42);
mockCallback.mockReturnValueOnce(10).mockReturnValueOnce(20);

mockCallback('arg1', 'arg2');

expect(mockCallback).toHaveBeenCalled();
expect(mockCallback).toHaveBeenCalledTimes(1);
expect(mockCallback).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockCallback.mock.results[0].value).toBe(42);
```

### jest.mock() — Mock Module

```typescript
// Mock toàn bộ module
jest.mock('../services/email.service', () => ({
  sendEmail: jest.fn().mockResolvedValue({ success: true }),
}));

import { sendEmail } from '../services/email.service';
import { registerUser } from './user.service';

describe('registerUser', () => {
  it('gửi welcome email sau khi đăng ký', async () => {
    await registerUser({ email: 'new@example.com', password: 'secret' });

    expect(sendEmail).toHaveBeenCalledWith(
      'new@example.com',
      'Welcome!',
      expect.any(String)
    );
  });
});
```

### jest.spyOn() — Spy Existing Method

```typescript
import * as logger from '../utils/logger';

it('log error khi operation fail', async () => {
  const spy = jest.spyOn(logger, 'error').mockImplementation(() => {});

  await failingOperation();

  expect(spy).toHaveBeenCalledWith(
    expect.stringContaining('Operation failed')
  );

  spy.mockRestore(); // Quan trọng — restore original implementation
});
```

### Manual Mocks (__mocks__ folder)

```
src/
├── services/
│   ├── payment.service.ts
│   └── __mocks__/
│       └── payment.service.ts   ← Jest tự động dùng khi jest.mock()
```

```typescript
// __mocks__/payment.service.ts
export const chargeCard = jest.fn().mockResolvedValue({ transactionId: 'mock-123' });
```

---

## Snapshot Testing

Snapshot capture **output tại thời điểm test** — phát hiện thay đổi không mong muốn.

```typescript
import { formatUserResponse } from './user.formatter';

it('format user response đúng structure', () => {
  const user = { id: '1', email: 'test@example.com', createdAt: new Date('2024-01-01') };
  const result = formatUserResponse(user);

  expect(result).toMatchSnapshot();
});
```

**Lần đầu chạy:** Jest tạo file `__snapshots__/user.formatter.test.ts.snap`

```bash
# Update snapshot khi thay đổi có chủ đích
jest --updateSnapshot
# hoặc
jest -u
```

**Khi nào dùng snapshot:**
- API response structure
- Error message formats
- Serialized config objects

**Khi KHÔNG dùng snapshot:**
- Logic có random values (IDs, timestamps) — dùng `expect.any(String)` thay thế
- Snapshot quá lớn — khó review trong PR

---

## Test Async Code

### async/await (Khuyến Nghị)

```typescript
it('fetch user từ database', async () => {
  const user = await userRepository.findById('123');
  expect(user.email).toBe('test@example.com');
});
```

### resolves / rejects Matchers

```typescript
it('resolve với user data', () => {
  return expect(fetchUser('123')).resolves.toEqual({
    id: '123',
    name: 'John',
  });
});

it('reject khi user không tồn tại', () => {
  return expect(fetchUser('invalid')).rejects.toThrow('User not found');
});
```

### Fake Timers

```typescript
jest.useFakeTimers();

it('gọi callback sau 1 giây', () => {
  const callback = jest.fn();
  setTimeout(callback, 1000);

  jest.advanceTimersByTime(1000);

  expect(callback).toHaveBeenCalled();
});

afterEach(() => {
  jest.useRealTimers();
});
```

---

## Code Coverage

```bash
npm run test:coverage
```

### Coverage Report Output

```
----------------------|---------|----------|---------|---------|
File                  | % Stmts | % Branch | % Funcs | % Lines |
----------------------|---------|----------|---------|---------|
All files             |   85.71 |    75.00 |   88.89 |   85.71 |
 user.service.ts      |   90.00 |    80.00 |  100.00 |   90.00 |
 order.service.ts     |   75.00 |    60.00 |   75.00 |   75.00 |
----------------------|---------|----------|---------|---------|
```

| Metric | Ý Nghĩa |
| ------ | ------- |
| **Statements** | % dòng code được execute |
| **Branches** | % if/else, switch cases được test |
| **Functions** | % functions được gọi |
| **Lines** | % lines được execute |

### Coverage Threshold trong CI

```typescript
// jest.config.ts
coverageThreshold: {
  global: { branches: 70, functions: 70, lines: 70, statements: 70 },
  './src/services/': { branches: 90, functions: 90, lines: 90 },
},
```

**Lưu ý:** 100% coverage ≠ bug-free. Focus test **critical business logic**, không chase coverage số.

---

## Best Practices

### DO ✅

```typescript
// Test behavior, không implementation
it('returns 404 when user not found', async () => {
  const result = await getUser('nonexistent');
  expect(result).toBeNull();
});

// Descriptive test names
it('rejects password shorter than 8 characters', () => { ... });

// Test edge cases
it('handles empty array', () => { ... });
it('handles null input', () => { ... });
```

### DON'T ❌

```typescript
// Đừng test private methods trực tiếp
it('calls _internalHelper', () => { ... }); // BAD

// Đừng dùng test.only trong committed code
it.only('debug test', () => { ... }); // BAD — block CI

// Đừng test framework/library code
it('Array.push works', () => {
  const arr = [];
  arr.push(1);
  expect(arr).toHaveLength(1); // BAD — test JavaScript, không test code của bạn
});

// Đừng hardcode timing
it('async test', async () => {
  await new Promise(r => setTimeout(r, 1000)); // BAD — flaky
});
```

### Test File Organization

```
src/
├── services/
│   ├── user.service.ts
│   └── user.service.test.ts      ← Co-located (khuyến nghị)
├── __tests__/
│   └── integration/
│       └── api.integration.test.ts
```

---

## Câu Hỏi Phỏng Vấn

**Q: Jest hoạt động như thế nào?**  
A: Jest discover test files theo pattern, chạy parallel trong worker processes, collect results, generate coverage với Istanbul.

**Q: `toBe` vs `toEqual`?**  
A: `toBe` dùng `===` (reference equality). `toEqual` deep compare objects/arrays.

**Q: Khi nào dùng `jest.mock()` vs `jest.spyOn()`?**  
A: `jest.mock()` thay thế toàn module. `jest.spyOn()` wrap method cụ thể, có thể restore sau.

**Q: Làm sao test private methods?**  
A: Không test trực tiếp — test qua public API. Nếu cần test riêng, extract thành pure function.

**Q: `clearMocks` vs `resetMocks` vs `restoreMocks`?**  
A: `clearMocks`: xóa call history. `resetMocks`: clear + reset implementation. `restoreMocks`: reset + restore original (cần `jest.spyOn`).

**Q: Coverage 100% có đủ không?**  
A: Không — có thể có tests nhưng miss edge cases. Combine coverage với meaningful assertions và integration tests.

---

**Tiếp theo:** [2-vitest.md](./2-vitest.md) — Test runner nhanh cho ESM/Vite projects
