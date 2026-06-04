# 1 — Jest Basics (Nền Tảng Jest)

> **Jest** là **JavaScript Testing Framework** (Framework Kiểm Thử JavaScript) do Meta phát triển, hiện là tiêu chuẩn công nghiệp cho unit testing trong hệ sinh thái React. Jest cung cấp: **test runner** (trình chạy test), **assertion library** (thư viện khẳng định), **mocking capabilities** (khả năng giả lập), và **code coverage** (độ phủ mã nguồn) — tất cả trong một gói duy nhất.

---

## 🎯 Mục Tiêu

- [ ] Viết unit tests (kiểm thử đơn vị) cơ bản với Jest
- [ ] Hiểu cấu trúc test: `describe`, `test`/`it`, `expect`
- [ ] Sử dụng **matchers** (bộ so khớp) phổ biến: `toBe`, `toEqual`, `toContain`, ...
- [ ] Xử lý **async tests** (kiểm thử bất đồng bộ): `async/await`, `resolves/rejects`
- [ ] Hiểu **test lifecycle hooks** (móc vòng đời test): `beforeEach`, `afterEach`, `beforeAll`, `afterAll`
- [ ] Tính **code coverage** (độ phủ mã nguồn) và đọc báo cáo

---

## 📦 Cài Đặt

### Với Create React App (CRA)

```bash
# CRA đã tích hợp Jest — không cần cài thêm
npm test          # Chạy tests ở watch mode (chế độ theo dõi)
npm test -- --coverage  # Chạy với code coverage
```

### Với Next.js

```bash
# Cài Jest và các dependencies
npm install --save-dev jest jest-environment-jsdom @types/jest

# Tạo jest.config.ts
```

```typescript
// jest.config.ts
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({ dir: './' });

const config: Config = {
  testEnvironment: 'jsdom',         // Môi trường trình duyệt giả lập
  setupFilesAfterFramework: ['<rootDir>/jest.setup.ts'],
};

export default createJestConfig(config);
```

### Với Vite

> Xem [5-vitest.md](./5-vitest.md) — Vitest là lựa chọn tốt hơn cho Vite projects.

---

## 🏗️ Cấu Trúc Bài Test Cơ Bản

```typescript
// Button.test.ts

// describe — Nhóm các test liên quan lại
describe('Button component', () => {
  
  // test hoặc it — Một test case riêng lẻ
  test('should render with correct label', () => {
    // Arrange — Chuẩn bị (setup dữ liệu, điều kiện)
    const label = 'Click me';
    
    // Act — Hành động (thực thi code cần test)
    const result = formatButtonLabel(label);
    
    // Assert — Khẳng định (kiểm tra kết quả)
    expect(result).toBe('Click me');
  });

  // it — Alias của test, đọc tự nhiên hơn với "it should..."
  it('should return empty string for null input', () => {
    expect(formatButtonLabel(null)).toBe('');
  });
});
```

### Quy Tắc Đặt Tên Test

```
✅ Tốt:
  "should display error message when email is invalid"
  "returns user data when API call succeeds"
  "throws error when userId is missing"

❌ Không tốt:
  "test 1"
  "works correctly"
  "button test"
```

---

## ⚖️ Matchers — Bộ So Khớp

### Equality Matchers — So Sánh Bằng Nhau

```typescript
// toBe — So sánh primitive (tham trị) bằng Object.is (giống ===)
expect(2 + 2).toBe(4);
expect('hello').toBe('hello');
expect(null).toBe(null);

// toEqual — So sánh sâu cho objects/arrays (deep equality)
expect({ name: 'An', age: 25 }).toEqual({ name: 'An', age: 25 }); // ✅
expect({ name: 'An', age: 25 }).toBe({ name: 'An', age: 25 });    // ❌ (khác reference)

// toStrictEqual — Nghiêm ngặt hơn toEqual (kiểm tra undefined properties)
expect({ a: undefined }).toStrictEqual({});  // ❌ fail
expect({ a: undefined }).toEqual({});         // ✅ pass

// not — Phủ định
expect(1 + 1).not.toBe(3);
expect([]).not.toEqual([1, 2, 3]);
```

### Truthiness Matchers — Kiểm Tra Đúng Sai

```typescript
expect(true).toBeTruthy();
expect(false).toBeFalsy();
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect('defined').toBeDefined();
expect(NaN).toBeNaN();
```

### Number Matchers — So Sánh Số

```typescript
expect(10).toBeGreaterThan(5);        // > 5
expect(10).toBeGreaterThanOrEqual(10); // >= 10
expect(5).toBeLessThan(10);           // < 10
expect(5).toBeLessThanOrEqual(5);     // <= 5

// Floating point — Số thực (tránh lỗi dấu phẩy động)
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // Chính xác đến 5 chữ số thập phân
```

### String Matchers — So Khớp Chuỗi

```typescript
expect('Hello World').toContain('World');
expect('Hello World').toMatch(/^Hello/);         // Regex
expect('Hello World').toMatch('Hello');          // Substring
expect('hello').not.toMatch(/^Hello/);           // case-sensitive
```

### Array Matchers — So Khớp Mảng

```typescript
const fruits = ['apple', 'banana', 'cherry'];

expect(fruits).toContain('banana');
expect(fruits).toHaveLength(3);
expect(fruits).toEqual(expect.arrayContaining(['apple', 'cherry'])); // Có chứa, thứ tự không quan trọng
```

### Object Matchers — So Khớp Đối Tượng

```typescript
const user = { id: 1, name: 'An', email: 'an@example.com', role: 'admin' };

// Chỉ kiểm tra một số properties, không cần exact match
expect(user).toMatchObject({ name: 'An', role: 'admin' });

// objectContaining — tương tự toMatchObject nhưng dùng trong nested expects
expect(user).toEqual(
  expect.objectContaining({ name: 'An' })
);
```

### Error Matchers — Kiểm Tra Lỗi

```typescript
function divide(a: number, b: number) {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}

// Kiểm tra function throw error
expect(() => divide(1, 0)).toThrow();
expect(() => divide(1, 0)).toThrow('Division by zero');
expect(() => divide(1, 0)).toThrow(Error);
expect(() => divide(1, 0)).toThrowError(/division/i);
```

---

## ⚡ Async Tests — Kiểm Thử Bất Đồng Bộ

### Cách 1: async/await (Khuyến Nghị)

```typescript
// Giả lập API call
async function fetchUser(id: number) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('User not found');
  return response.json();
}

test('should fetch user successfully', async () => {
  // Mock fetch (xem thêm ở 3-mocking.md)
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ id: 1, name: 'An' }),
  });

  const user = await fetchUser(1);
  
  expect(user).toEqual({ id: 1, name: 'An' });
});

test('should throw when user not found', async () => {
  global.fetch = jest.fn().mockResolvedValue({ ok: false });

  await expect(fetchUser(999)).rejects.toThrow('User not found');
});
```

### Cách 2: resolves / rejects

```typescript
test('should resolve with user data', () => {
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ id: 1, name: 'An' }),
  });

  // Lưu ý: PHẢI return promise, nếu không Jest không đợi
  return expect(fetchUser(1)).resolves.toEqual({ id: 1, name: 'An' });
});

test('should reject for invalid user', () => {
  global.fetch = jest.fn().mockResolvedValue({ ok: false });

  return expect(fetchUser(999)).rejects.toThrow('User not found');
});
```

### Callback-based (Ít Dùng)

```typescript
// PHẢI gọi done() để báo test kết thúc, nếu không test timeout
test('should call callback with data', (done) => {
  fetchUserCallback(1, (error, user) => {
    try {
      expect(error).toBeNull();
      expect(user.name).toBe('An');
      done(); // ✅ Báo Jest test đã xong
    } catch (e) {
      done(e); // ✅ Báo Jest test fail với lỗi
    }
  });
});
```

---

## 🔄 Test Lifecycle Hooks — Móc Vòng Đời

```typescript
describe('Database operations', () => {
  let db: Database;

  // Chạy một lần trước tất cả tests trong describe
  beforeAll(async () => {
    db = await Database.connect('test-db');
    await db.migrate();
  });

  // Chạy một lần sau tất cả tests trong describe
  afterAll(async () => {
    await db.cleanup();
    await db.disconnect();
  });

  // Chạy trước MỖI test
  beforeEach(async () => {
    await db.seed(); // Thêm dữ liệu mẫu (seed data)
  });

  // Chạy sau MỖI test
  afterEach(async () => {
    await db.truncate(); // Xóa dữ liệu sau mỗi test (clean slate)
  });

  test('should create user', async () => {
    const user = await db.users.create({ name: 'An' });
    expect(user.id).toBeDefined();
  });

  test('should find user by id', async () => {
    const user = await db.users.findById(1);
    expect(user.name).toBe('Seeded User');
  });
});
```

---

## 📊 Code Coverage — Độ Phủ Mã Nguồn

### Các Loại Coverage

| Loại | Ý Nghĩa | Ví Dụ |
| ---- | -------- | ----- |
| **Statements** (câu lệnh) | Bao nhiêu % dòng code được thực thi | `if (a) { b() }` → cần test cả `a=true` và `a=false` |
| **Branches** (nhánh) | Bao nhiêu % nhánh điều kiện | Mỗi `if/else`, `switch case`, `ternary` |
| **Functions** (hàm) | Bao nhiêu % hàm được gọi | Các hàm không có test sẽ bị flag |
| **Lines** (dòng) | Tương tự Statements | Đôi khi khác Statements với minified code |

### Chạy Coverage

```bash
# Jest
npx jest --coverage

# Chỉ test specific file
npx jest Button --coverage

# Coverage threshold (ngưỡng độ phủ) — fail nếu dưới ngưỡng
```

```json
// jest.config.json — cấu hình ngưỡng coverage
{
  "coverageThreshold": {
    "global": {
      "branches": 70,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

### Đọc Coverage Report

```
-----------------------------|---------|----------|---------|---------|---
File                         | % Stmts | % Branch | % Funcs | % Lines |
-----------------------------|---------|----------|---------|---------|---
 components/                 |         |          |         |         |
  Button.tsx                 |   100   |   100    |   100   |   100   |   ✅
  UserProfile.tsx            |   72.7  |   50     |   83.3  |   72.7  |   ⚠️
  PaymentForm.tsx            |   45.8  |   30     |   50    |   45.8  |   ❌
```

> **Mục tiêu thực tế:** 80% statement coverage là tốt cho hầu hết dự án. 100% coverage không đảm bảo không có bug — có thể test sai điều sai.

---

## 🏷️ Tổ Chức Tests

### Skip và Only

```typescript
// Skip test — chạy tất cả trừ test này
test.skip('this is temporarily disabled', () => {
  // ...
});

// Only — chỉ chạy test này (rất hữu ích khi debug)
test.only('focus on this test', () => {
  // ...
});

// describe.only — chỉ chạy describe block này
describe.only('UserProfile', () => {
  // ...
});
```

### Each — Test Nhiều Trường Hợp

```typescript
// test.each — test nhiều input cùng lúc
test.each([
  [1, 2, 3],
  [0, 0, 0],
  [-1, 1, 0],
  [100, -50, 50],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});

// Với object — dễ đọc hơn
test.each([
  { email: 'valid@example.com', isValid: true },
  { email: 'invalid-email',     isValid: false },
  { email: '',                  isValid: false },
  { email: 'test@',             isValid: false },
])('validateEmail($email) → $isValid', ({ email, isValid }) => {
  expect(validateEmail(email)).toBe(isValid);
});
```

---

## 🛠️ Snapshot Testing — Kiểm Thử Ảnh Chụp

> **Snapshot testing** (Kiểm Thử Ảnh Chụp) lưu lại "ảnh chụp" output của component và so sánh ở lần chạy tiếp theo.

```typescript
import { render } from '@testing-library/react';

test('Button matches snapshot', () => {
  const { asFragment } = render(<Button label="Click me" variant="primary" />);
  expect(asFragment()).toMatchSnapshot(); // Lần đầu: tạo snapshot; Lần sau: so sánh
});
```

### Khi Nào Dùng Snapshot

```
✅ Dùng snapshot khi:
  - Component chỉ render dữ liệu (không có logic phức tạp)
  - Muốn phát hiện thay đổi UI không có chủ ý

❌ Không dùng snapshot khi:
  - Component phức tạp → snapshot quá lớn, khó đọc
  - Thay đổi thường xuyên → phải update snapshot liên tục
  - Thay thế RTL queries → kém tin cậy hơn
```

```bash
# Cập nhật snapshot sau khi intentionally thay đổi UI
npx jest --updateSnapshot
npx jest -u  # Shorthand
```

---

## 📋 Custom Matchers — Bộ So Khớp Tùy Chỉnh

```typescript
// jest.setup.ts — thêm matchers tùy chỉnh
import '@testing-library/jest-dom'; // Matchers cho DOM: toBeInTheDocument, toBeVisible, ...

// Tạo custom matcher
expect.extend({
  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () =>
        `expected ${received} ${pass ? 'not ' : ''}to be within range [${floor}, ${ceiling}]`,
    };
  },
});

// Sử dụng
test('value is within range', () => {
  expect(50).toBeWithinRange(10, 100); // ✅
  expect(5).not.toBeWithinRange(10, 100); // ✅
});
```

---

## 🎯 Best Practices — Thực Hành Tốt Nhất

### 1. Một Test — Một Khẳng Định Chính

```typescript
// ❌ Quá nhiều assertions — khó biết cái nào fail
test('user functionality', () => {
  expect(createUser('An')).toBeDefined();
  expect(createUser('An').name).toBe('An');
  expect(createUser(null)).toBeNull();
  expect(updateUser(1, 'Binh').name).toBe('Binh');
});

// ✅ Tách thành nhiều tests có tên rõ ràng
test('createUser returns user with correct name', () => {
  expect(createUser('An').name).toBe('An');
});

test('createUser returns null for invalid input', () => {
  expect(createUser(null)).toBeNull();
});
```

### 2. AAA Pattern — Arrange, Act, Assert

```typescript
test('should calculate discount correctly', () => {
  // Arrange — Chuẩn bị
  const price = 100;
  const discountPercent = 20;

  // Act — Thực thi
  const finalPrice = applyDiscount(price, discountPercent);

  // Assert — Khẳng định
  expect(finalPrice).toBe(80);
});
```

### 3. Test Tên File Đúng Convention

```
Button.test.tsx    ← Phổ biến nhất
Button.spec.tsx    ← Cũng phổ biến (specification)
__tests__/Button.tsx ← Thư mục tests riêng
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-react-testing-library.md](./2-react-testing-library.md) — Test React components
- **Trở lại:** [README.md](./README.md) — Tổng quan testing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
