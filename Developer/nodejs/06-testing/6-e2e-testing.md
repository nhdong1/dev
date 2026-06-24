# E2E Testing — Playwright API Testing và Newman/Postman

> E2E (End-to-End — Đầu Cuối) tests verify **toàn bộ system** hoạt động đúng từ đầu đến cuối — client request qua network đến running server, real database, real services. Confidence cao nhất nhưng chậm và tốn kém maintain.

## Mục Lục

1. [E2E Testing Là Gì](#e2e-testing-là-gì)
2. [E2E vs Integration vs Unit](#e2e-vs-integration-vs-unit)
3. [Playwright API Testing](#playwright-api-testing)
4. [Newman — Postman CLI](#newman--postman-cli)
5. [Test Data Management](#test-data-management)
6. [CI/CD cho E2E Tests](#cicd-cho-e2e-tests)
7. [Flaky Tests — Xử Lý](#flaky-tests--xử-lý)
8. [TDD — Test-Driven Development](#tdd--test-driven-development)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## E2E Testing Là Gì

E2E test simulate **user journey thật** — không mock internal layers, test against deployed or locally running application.

```
┌──────────┐    HTTP/HTTPS    ┌──────────────┐    SQL    ┌────────────┐
│  Client  │ ───────────────► │  Node.js API │ ────────► │ PostgreSQL │
│ (Test)   │                  │  (Running)   │           │  (Real)    │
└──────────┘                  └──────────────┘           └────────────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │    Redis     │
                              │   (Real)     │
                              └──────────────┘
```

| Đặc Điểm | Mô Tả |
| -------- | ----- |
| **Scope** | Toàn bộ stack — API + DB + cache + auth |
| **Speed** | Chậm (giây đến phút per test) |
| **Confidence** | Cao nhất — catch integration bugs |
| **Maintenance** | Cao — brittle khi UI/API thay đổi |
| **Quantity** | Ít — chỉ critical user paths |

---

## E2E vs Integration vs Unit

| | Unit | Integration | E2E |
| - | ---- | ----------- | --- |
| **Dependencies** | Mocked | Một số real (DB) | Tất cả real |
| **Server** | Không | In-memory (Supertest) | Running server |
| **Network** | Không | Simulated | Real HTTP |
| **Số lượng** | Nhiều (100s) | Vừa (10s–50s) | Ít (5–20) |
| **CI time** | Giây | Phút | Phút đến chục phút |

### Critical Paths Nên E2E Test

1. **User registration + login flow**
2. **Checkout / payment flow**
3. **CRUD operations end-to-end**
4. **Auth-protected resource access**
5. **Webhook handling**

---

## Playwright API Testing

Playwright chủ yếu cho browser E2E, nhưng **API testing** built-in qua `request` fixture — không cần browser.

### Cài Đặt

```bash
npm install -D @playwright/test
npx playwright install  # Chỉ cần nếu test browser
```

### playwright.config.ts

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: process.env.API_URL || 'http://localhost:3000',
    extraHTTPHeaders: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
  },
  projects: [
    { name: 'api', testMatch: '**/*.api.spec.ts' },
  ],
});
```

### API Test Examples

```typescript
// e2e/users.api.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User API E2E', () => {
  let authToken: string;
  let createdUserId: string;

  test.beforeAll(async ({ request }) => {
    const loginRes = await request.post('/auth/login', {
      data: {
        email: process.env.E2E_ADMIN_EMAIL || 'admin@test.com',
        password: process.env.E2E_ADMIN_PASSWORD || 'admin123',
      },
    });
    expect(loginRes.ok()).toBeTruthy();
    const body = await loginRes.json();
    authToken = body.accessToken;
  });

  test('POST /api/users — tạo user mới', async ({ request }) => {
    const email = `e2e-${Date.now()}@test.com`;

    const response = await request.post('/api/users', {
      headers: { Authorization: `Bearer ${authToken}` },
      data: { email, name: 'E2E Test User', password: 'SecurePass123!' },
    });

    expect(response.status()).toBe(201);

    const user = await response.json();
    expect(user.email).toBe(email);
    expect(user.id).toBeDefined();
    createdUserId = user.id;
  });

  test('GET /api/users/:id — lấy user vừa tạo', async ({ request }) => {
    test.skip(!createdUserId, 'Depends on create test');

    const response = await request.get(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${authToken}` },
    });

    expect(response.status()).toBe(200);
    const user = await response.json();
    expect(user.id).toBe(createdUserId);
  });

  test('DELETE /api/users/:id — xóa user', async ({ request }) => {
    test.skip(!createdUserId, 'Depends on create test');

    const deleteRes = await request.delete(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${authToken}` },
    });
    expect(deleteRes.status()).toBe(204);

    const getRes = await request.get(`/api/users/${createdUserId}`, {
      headers: { Authorization: `Bearer ${authToken}` },
    });
    expect(getRes.status()).toBe(404);
  });
});
```

### Auth State Reuse

```typescript
// e2e/auth.setup.ts — Chạy 1 lần, lưu auth state
import { test as setup } from '@playwright/test';

const authFile = 'e2e/.auth/user.json';

setup('authenticate', async ({ request }) => {
  const response = await request.post('/auth/login', {
    data: { email: 'test@example.com', password: 'password' },
  });
  const { accessToken } = await response.json();

  await request.storageState({
    path: authFile,
    indexedDB: false,
    cookies: [],
    origins: [{
      origin: process.env.API_URL!,
      localStorage: [{ name: 'token', value: accessToken }],
    }],
  });
});
```

### Global Setup — Start Server

```typescript
// e2e/global-setup.ts
import { spawn, ChildProcess } from 'child_process';

let serverProcess: ChildProcess;

export default async function globalSetup() {
  serverProcess = spawn('npm', ['run', 'start:test'], {
    env: { ...process.env, PORT: '3001', NODE_ENV: 'test' },
    stdio: 'pipe',
  });

  await waitForServer('http://localhost:3001/health', 30000);

  return async () => {
    serverProcess.kill();
  };
}

async function waitForServer(url: string, timeout: number) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    try {
      const res = await fetch(url);
      if (res.ok) return;
    } catch { /* retry */ }
    await new Promise(r => setTimeout(r, 500));
  }
  throw new Error(`Server not ready: ${url}`);
}
```

---

## Newman — Postman CLI

Newman chạy **Postman Collections** trong CI — phù hợp team đã có Postman collections hoặc QA viết tests.

### Workflow

```
1. Viết requests + tests trong Postman UI
2. Export collection JSON
3. Chạy Newman trong CI
```

### Cài Đặt

```bash
npm install -D newman newman-reporter-htmlextra
```

### Collection Structure

```json
{
  "info": { "name": "User API E2E", "schema": "..." },
  "item": [
    {
      "name": "Create User",
      "request": {
        "method": "POST",
        "url": "{{baseUrl}}/api/users",
        "header": [
          { "key": "Authorization", "value": "Bearer {{authToken}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\"email\":\"{{$randomEmail}}\",\"name\":\"Test\"}"
        }
      },
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test('Status is 201', () => {",
              "  pm.response.to.have.status(201);",
              "});",
              "pm.test('Response has id', () => {",
              "  const json = pm.response.json();",
              "  pm.expect(json.id).to.be.a('string');",
              "  pm.collectionVariables.set('userId', json.id);",
              "});"
            ]
          }
        }
      ]
    }
  ],
  "variable": [
    { "key": "baseUrl", "value": "http://localhost:3000" }
  ]
}
```

### Chạy Newman

```bash
# Basic
npx newman run collections/user-api.json

# Với environment
npx newman run collections/user-api.json \
  -e environments/staging.json

# CI với reporters
npx newman run collections/user-api.json \
  --env-var "baseUrl=http://localhost:3000" \
  --reporters cli,junit,htmlextra \
  --reporter-junit-export results/newman-results.xml \
  --reporter-htmlextra-export results/newman-report.html \
  --bail  # Stop on first failure
```

### package.json Scripts

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:postman": "newman run collections/user-api.json --env-var baseUrl=http://localhost:3000",
    "test:e2e:all": "npm run start:test & sleep 5 && npm run test:e2e && kill %1"
  }
}
```

### Playwright vs Newman

| | Playwright API | Newman |
| - | -------------- | ------ |
| **Format** | TypeScript code | JSON collection |
| **Version control** | Native code review | Export/import JSON |
| **Type safety** | ✅ TypeScript | ❌ |
| **Team collaboration** | Developers | QA + Developers |
| **Debugging** | VS Code breakpoints | Postman UI |
| **CI integration** | Excellent | Excellent |

---

## Test Data Management

### Isolated Test Data

```typescript
// Mỗi test tạo data riêng — không conflict
const uniqueEmail = () => `test-${Date.now()}-${Math.random().toString(36).slice(2)}@e2e.com`;

test('create user', async ({ request }) => {
  const email = uniqueEmail();
  const res = await request.post('/api/users', { data: { email, name: 'Test' } });
  expect(res.status()).toBe(201);
});
```

### Database Seeding

```typescript
// e2e/helpers/seed.ts
export async function seedTestUser(prisma: PrismaClient) {
  return prisma.user.upsert({
    where: { email: 'e2e-admin@test.com' },
    update: {},
    create: {
      email: 'e2e-admin@test.com',
      password: await hash('admin123'),
      role: 'ADMIN',
    },
  });
}

// e2e/global-setup.ts
export default async function globalSetup() {
  const prisma = new PrismaClient();
  await seedTestUser(prisma);
  await prisma.$disconnect();
}
```

### Cleanup Strategy

| Strategy | Pros | Cons |
| -------- | ---- | ---- |
| **Delete after each test** | Clean state | Slow, order-dependent |
| **Unique data per test** | Parallel-safe | DB grows over time |
| **Reset DB before suite** | Fresh start | Slow startup |
| **Dedicated test environment** | Isolated | Infra cost |

---

## CI/CD cho E2E Tests

### GitHub Actions — Full E2E Pipeline

```yaml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb

      - name: Start API server
        run: npm run start:test &
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          PORT: 3000

      - name: Wait for server
        run: npx wait-on http://localhost:3000/health --timeout 30000

      - name: Run Playwright E2E
        run: npx playwright test
        env:
          API_URL: http://localhost:3000

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

### Khi Chạy E2E trong CI

- Chạy **sau** unit + integration tests pass
- Chỉ trên `main` branch hoặc PR labeled `e2e`
- Retry flaky tests (2 lần trong CI)
- Upload artifacts (reports, screenshots) khi fail

---

## Flaky Tests — Xử Lý

Flaky test (test không ổn định) — pass/fail ngẫu nhiên không do code change.

### Nguyên Nhân Phổ Biến

| Nguyên Nhân | Giải Pháp |
| ----------- | --------- |
| **Race conditions** | `await` đúng, `waitFor` thay `setTimeout` |
| **Shared state** | Isolated data per test |
| **Timing dependencies** | Fake timers hoặc polling với timeout |
| **External service down** | Mock hoặc health check trước test |
| **Port conflicts** | Dynamic ports, `wait-on` |

### Playwright Auto-Waiting

```typescript
// Playwright tự đợi — không cần sleep
const response = await request.get('/api/users');
expect(response.status()).toBe(200); // Playwright retry assertions

// Explicit wait
await expect.poll(async () => {
  const res = await request.get('/api/orders/123/status');
  const body = await res.json();
  return body.status;
}).toBe('completed');
```

### Quarantine Flaky Tests

```typescript
test('flaky test — quarantined', async ({ request }) => {
  test.fixme(true, 'Flaky — ticket JIRA-123');
  // ...
});
```

**Policy:** Không ignore flaky tests — fix hoặc quarantine với ticket tracking.

---

## TDD — Test-Driven Development

TDD (Test-Driven Development — Phát Triển Hướng Kiểm Thử) — viết test trước, implement code sau.

### Red-Green-Refactor Cycle

```
┌─────────┐     ┌─────────┐     ┌──────────┐
│   RED   │ ──► │  GREEN  │ ──► │ REFACTOR │
│ Test    │     │ Minimal │     │ Improve  │
│ Fails   │     │ Code    │     │ Code     │
└─────────┘     └─────────┘     └────┬─────┘
     ▲                               │
     └───────────────────────────────┘
```

### Ví Dụ TDD

```typescript
// Step 1: RED — Viết test trước (fail)
describe('calculateShipping', () => {
  it('free shipping cho đơn > 500k', () => {
    expect(calculateShipping(600000, 'standard')).toBe(0);
  });

  it('tính phí standard 30k cho đơn < 500k', () => {
    expect(calculateShipping(100000, 'standard')).toBe(30000);
  });
});

// Step 2: GREEN — Implement minimal
export function calculateShipping(total: number, method: string): number {
  if (method === 'standard' && total >= 500000) return 0;
  if (method === 'standard') return 30000;
  throw new Error('Unknown shipping method');
}

// Step 3: REFACTOR — Cải thiện không đổi behavior
const SHIPPING_RATES = {
  standard: { fee: 30000, freeThreshold: 500000 },
} as const;

export function calculateShipping(total: number, method: keyof typeof SHIPPING_RATES): number {
  const rate = SHIPPING_RATES[method];
  if (!rate) throw new Error('Unknown shipping method');
  return total >= rate.freeThreshold ? 0 : rate.fee;
}
```

### Khi Nào Dùng TDD

| Phù Hợp | Không Phù Hợp |
| ------- | ------------- |
| Business logic phức tạp | CRUD đơn giản |
| Algorithms, calculations | UI exploration |
| Bug fixes (regression test) | Spike/prototype |
| API contract design | Không rõ requirements |

---

## Best Practices

### E2E Test Design

```typescript
// ✅ Test user journey, không test implementation
test('user can complete checkout', async ({ request }) => {
  // 1. Add to cart
  // 2. Apply coupon
  // 3. Checkout
  // 4. Verify order created
});

// ✅ Independent tests — không depend on order
test('login returns valid token', async ({ request }) => {
  const res = await request.post('/auth/login', { data: credentials });
  expect(res.ok()).toBeTruthy();
});

// ❌ Không test mọi edge case ở E2E layer — để unit test handle
```

### E2E Checklist

- [ ] Chỉ cover critical happy paths + 1–2 sad paths
- [ ] Tests độc lập — có thể chạy riêng lẻ
- [ ] Unique test data — không hardcode IDs
- [ ] Environment variables cho secrets
- [ ] Retry policy trong CI
- [ ] Reports upload khi fail
- [ ] E2E chạy trên staging trước production deploy

---

## Câu Hỏi Phỏng Vấn

**Q: E2E vs Integration test khác gì?**  
A: Integration test thường dùng Supertest (in-memory, không network). E2E test running server qua real HTTP — giống production hơn.

**Q: Bao nhiêu E2E tests là đủ?**  
A: 5–20 critical paths. Nếu có 100+ E2E, CI quá chậm — chuyển edge cases xuống unit/integration.

**Q: Playwright chỉ cho browser?**  
A: Không — `request` API test HTTP endpoints mà không launch browser. Nhanh hơn browser E2E.

**Q: Newman vs Playwright cho API E2E?**  
A: Newman nếu team dùng Postman. Playwright nếu muốn TypeScript, code review, và unified tool cho API + browser E2E.

**Q: TDD có bắt buộc không?**  
A: Không — nhưng hữu ích cho complex logic. Quan trọng hơn là có test suite tốt, không nhất thiết TDD strict.

**Q: Xử lý flaky E2E trong CI?**  
A: Retry 1–2 lần, fix root cause, quarantine với ticket nếu không fix ngay. Không `test.skip` vĩnh viễn.

---

**Quay lại:** [README.md](./README.md) — Tổng quan chủ đề Kiểm Thử
