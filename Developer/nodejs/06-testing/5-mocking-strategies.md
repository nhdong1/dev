# Mocking Strategies — jest.mock, DI Mocking và MSW

> Mocking (giả lập) là kỹ thuật cốt lõi trong unit testing — cô lập code under test bằng cách thay thế dependencies (phụ thuộc) bằng controlled doubles (bản sao kiểm soát). Hiểu khi nào mock, mock gì, và cách mock đúng là kỹ năng phỏng vấn quan trọng.

## Mục Lục

1. [Test Doubles — Các Loại Giả Lập](#test-doubles--các-loại-giả-lập)
2. [Khi Nào Mock, Khi Nào Không](#khi-nào-mock-khi-nào-không)
3. [jest.mock() — Module Mocking](#jestmock--module-mocking)
4. [Dependency Injection Mocking](#dependency-injection-mocking)
5. [MSW — Mock Service Worker](#msw--mock-service-worker)
6. [Mocking Database Layer](#mocking-database-layer)
7. [Mocking Date, Random, Environment](#mocking-date-random-environment)
8. [Anti-Patterns](#anti-patterns)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Test Doubles — Các Loại Giả Lập

| Loại | Mục Đích | Ví Dụ |
| ---- | -------- | ----- |
| **Dummy** | Placeholder — không dùng | Pass `null` vào param không cần |
| **Stub** | Trả về giá trị cố định | `mockReturnValue({ id: 1 })` |
| **Spy** | Ghi lại cách gọi, có thể gọi implementation thật | `jest.spyOn(logger, 'info')` |
| **Mock** | Spy + preset behavior + assertions | `jest.fn().mockResolvedValue(data)` |
| **Fake** | Implementation đơn giản thay thế | In-memory database thay PostgreSQL |

```
┌─────────────────────────────────────────────────────────┐
│                  Code Under Test                         │
│                  (UserService)                          │
│                       │                                 │
│         ┌─────────────┼─────────────┐                    │
│         ▼             ▼             ▼                    │
│   UserRepository  EmailService  PaymentGateway           │
│      (MOCK)         (MOCK)         (MSW)                  │
└─────────────────────────────────────────────────────────┘
```

---

## Khi Nào Mock, Khi Nào Không

### Nên Mock

- **External APIs** — Stripe, SendGrid, third-party services
- **Database** — trong unit tests (dùng Testcontainers cho integration)
- **File system** — `fs` operations
- **Time/Random** — `Date.now()`, `Math.random()`, UUID generation
- **Slow operations** — network, encryption

### Không Nên Mock

- **Code under test** — test behavior thật, không mock chính nó
- **Simple pure functions** — test trực tiếp
- **Framework internals** — Express, không mock `req`/`res` nếu dùng Supertest
- **Value objects / DTOs** — dùng real instances

### Quy Tắc Thumb

> **Mock ở boundaries (ranh giới)** — nơi code của bạn giao tiếp với thế giới bên ngoài (I/O boundary).

---

## jest.mock() — Module Mocking

### Auto Mock — Toàn Bộ Module

```typescript
jest.mock('../repositories/user.repository');

import { findById, create } from '../repositories/user.repository';

// Tất cả exports tự động thành jest.fn() return undefined
```

### Manual Mock Factory

```typescript
jest.mock('../services/email.service', () => ({
  sendWelcomeEmail: jest.fn().mockResolvedValue({ messageId: 'mock-123' }),
  sendPasswordReset: jest.fn().mockResolvedValue({ messageId: 'mock-456' }),
}));
```

### Partial Mock — Giữ Một Phần Module

```typescript
jest.mock('../utils/crypto', () => ({
  ...jest.requireActual('../utils/crypto'),
  generateSecureToken: jest.fn().mockReturnValue('fixed-token-for-test'),
}));
```

### Dynamic Mock Values

```typescript
const mockFindById = jest.fn();

jest.mock('../repositories/user.repository', () => ({
  findById: (...args: unknown[]) => mockFindById(...args),
}));

// Trong test — control behavior
beforeEach(() => {
  mockFindById.mockReset();
});

it('test case', async () => {
  mockFindById.mockResolvedValue({ id: '1', email: 'test@example.com' });
  // ...
});
```

### Mock Hoisting — Thứ Tự Quan Trọng

```typescript
// jest.mock() được hoist lên đầu file tự động
jest.mock('./api');  // Phải trước import

import { fetchData } from './api';  // Đã là mock
import { processData } from './processor';  // Code under test
```

---

## Dependency Injection Mocking

DI (Dependency Injection — Tiêm Phụ Thuộc) làm mocking dễ hơn — inject mock qua constructor.

### Constructor Injection

```typescript
// src/services/order.service.ts
export class OrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly emailService: EmailService,
  ) {}

  async placeOrder(userId: string, items: OrderItem[]): Promise<Order> {
    const order = await this.orderRepo.create({ userId, items });
    await this.paymentGateway.charge(order.total);
    await this.emailService.sendConfirmation(userId, order.id);
    return order;
  }
}
```

```typescript
// src/services/order.service.test.ts
describe('OrderService', () => {
  let service: OrderService;
  let mockOrderRepo: jest.Mocked<OrderRepository>;
  let mockPayment: jest.Mocked<PaymentGateway>;
  let mockEmail: jest.Mocked<EmailService>;

  beforeEach(() => {
    mockOrderRepo = {
      create: jest.fn(),
      findById: jest.fn(),
    };
    mockPayment = {
      charge: jest.fn().mockResolvedValue({ transactionId: 'tx-1' }),
    };
    mockEmail = {
      sendConfirmation: jest.fn().mockResolvedValue(undefined),
    };

    service = new OrderService(mockOrderRepo, mockPayment, mockEmail);
  });

  it('placeOrder tạo order, charge payment, gửi email', async () => {
    const mockOrder = { id: 'order-1', total: 100, userId: 'user-1' };
    mockOrderRepo.create.mockResolvedValue(mockOrder);

    const result = await service.placeOrder('user-1', [{ productId: 'p1', qty: 1 }]);

    expect(mockOrderRepo.create).toHaveBeenCalledWith({
      userId: 'user-1',
      items: [{ productId: 'p1', qty: 1 }],
    });
    expect(mockPayment.charge).toHaveBeenCalledWith(100);
    expect(mockEmail.sendConfirmation).toHaveBeenCalledWith('user-1', 'order-1');
    expect(result).toEqual(mockOrder);
  });

  it('không gửi email khi payment fail', async () => {
    mockOrderRepo.create.mockResolvedValue({ id: 'order-1', total: 100, userId: 'user-1' });
    mockPayment.charge.mockRejectedValue(new Error('Payment declined'));

    await expect(service.placeOrder('user-1', [])).rejects.toThrow('Payment declined');
    expect(mockEmail.sendConfirmation).not.toHaveBeenCalled();
  });
});
```

### NestJS — overrideProvider

```typescript
const module = await Test.createTestingModule({
  providers: [
    OrderService,
    { provide: OrderRepository, useValue: mockOrderRepo },
    { provide: PaymentGateway, useValue: mockPayment },
    { provide: EmailService, useValue: mockEmail },
  ],
}).compile();

const service = module.get(OrderService);
```

### Factory Pattern

```typescript
// test/factories/mocks.ts
export function createMockUserRepository(
  overrides: Partial<jest.Mocked<UserRepository>> = {}
): jest.Mocked<UserRepository> {
  return {
    findById: jest.fn(),
    findByEmail: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
    ...overrides,
  };
}

// Trong test
const repo = createMockUserRepository({
  findById: jest.fn().mockResolvedValue({ id: '1', email: 'test@example.com' }),
});
```

---

## MSW — Mock Service Worker

MSW intercept HTTP requests ở network level — test code gọi `fetch`/`axios` thật mà không hit external API.

### Setup

```bash
npm install -D msw
```

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('https://api.stripe.com/v1/charges', () => {
    return HttpResponse.json({
      id: 'ch_mock_123',
      status: 'succeeded',
      amount: 1000,
    });
  }),

  http.post('https://api.sendgrid.com/v3/mail/send', () => {
    return HttpResponse.json({}, { status: 202 });
  }),

  http.get('https://api.weather.com/v1/current', ({ request }) => {
    const url = new URL(request.url);
    const city = url.searchParams.get('city');

    if (city === 'InvalidCity') {
      return HttpResponse.json({ error: 'City not found' }, { status: 404 });
    }

    return HttpResponse.json({ city, temperature: 25, unit: 'celsius' });
  }),
];
```

```typescript
// test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// test/setup.ts
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Test với MSW

```typescript
// src/services/weather.service.test.ts
import { getWeather } from './weather.service';

describe('WeatherService', () => {
  it('fetch weather data thành công', async () => {
    const weather = await getWeather('Hanoi');
    expect(weather.temperature).toBe(25);
    expect(weather.city).toBe('Hanoi');
  });

  it('handle 404 từ external API', async () => {
    await expect(getWeather('InvalidCity')).rejects.toThrow('City not found');
  });

  it('override handler cho test case cụ thể', async () => {
    server.use(
      http.get('https://api.weather.com/v1/current', () => {
        return HttpResponse.json({ city: 'Hanoi', temperature: -999, unit: 'celsius' });
      })
    );

    const weather = await getWeather('Hanoi');
    expect(weather.temperature).toBe(-999);
  });
});
```

### MSW vs jest.mock cho HTTP

| | jest.mock('axios') | MSW |
| - | ------------------ | --- |
| **Test realism** | Mock implementation | Real HTTP client code path |
| **Setup** | Per-module mock | Centralized handlers |
| **Override** | Per-test mock | `server.use()` runtime override |
| **Unhandled requests** | Silent pass | `onUnhandledRequest: 'error'` catch bugs |

---

## Mocking Database Layer

### Repository Interface Mock

```typescript
// Không mock Prisma trực tiếp — mock repository interface
interface UserRepository {
  findById(id: string): Promise<User | null>;
  create(data: CreateUserDto): Promise<User>;
}

// Production
class PrismaUserRepository implements UserRepository { ... }

// Test
const mockRepo: jest.Mocked<UserRepository> = {
  findById: jest.fn(),
  create: jest.fn(),
};
```

### Prisma Mock (Khi Cần)

```typescript
import { PrismaClient } from '@prisma/client';
import { mockDeep, mockReset, DeepMockProxy } from 'jest-mock-extended';

jest.mock('@prisma/client', () => ({
  PrismaClient: jest.fn(),
}));

import { prisma } from '../lib/prisma';

const prismaMock = prisma as unknown as DeepMockProxy<PrismaClient>;

beforeEach(() => {
  mockReset(prismaMock);
});

it('query users', async () => {
  prismaMock.user.findMany.mockResolvedValue([
    { id: '1', email: 'test@example.com', name: 'Test' },
  ]);

  const users = await getAllUsers();
  expect(users).toHaveLength(1);
});
```

---

## Mocking Date, Random, Environment

### Fake Timers + System Time

```typescript
beforeEach(() => {
  jest.useFakeTimers();
  jest.setSystemTime(new Date('2024-06-15T10:00:00Z'));
});

afterEach(() => {
  jest.useRealTimers();
});

it('token expires sau 15 phút', () => {
  const token = generateToken({ sub: 'user-1' });
  const decoded = decodeToken(token);
  expect(decoded.exp).toBe(
    Math.floor(new Date('2024-06-15T10:15:00Z').getTime() / 1000)
  );
});
```

### Mock UUID

```typescript
jest.mock('uuid', () => ({
  v4: jest.fn().mockReturnValue('fixed-uuid-1234'),
}));
```

### Environment Variables

```typescript
const originalEnv = process.env;

beforeEach(() => {
  process.env = { ...originalEnv, JWT_SECRET: 'test-secret', NODE_ENV: 'test' };
});

afterEach(() => {
  process.env = originalEnv;
});
```

---

## Anti-Patterns

### ❌ Over-Mocking

```typescript
// BAD — mock quá nhiều, test không còn ý nghĩa
it('processOrder', async () => {
  mockValidate.mockReturnValue(true);
  mockCalculate.mockReturnValue(100);
  mockFormat.mockReturnValue({ formatted: true });
  mockSave.mockResolvedValue({ id: 1 });
  // Test chỉ verify mocks gọi nhau — không test logic thật
});
```

### ❌ Mocking Code Under Test

```typescript
// BAD
jest.spyOn(service, 'processOrder').mockImplementation(() => ({ success: true }));
const result = await service.processOrder(data);
expect(result.success).toBe(true); // Test mock của chính nó
```

### ❌ Brittle Mocks — Test Implementation

```typescript
// BAD — test thứ tự internal calls
expect(mockRepo.findById).toHaveBeenCalledBefore(mockCache.get);
// Thay đổi implementation → test break dù behavior đúng
```

### ❌ Shared Mock State

```typescript
// BAD — mock state leak giữa tests
const sharedMock = jest.fn(); // Không reset → flaky tests
```

---

## Best Practices

### DO ✅

```typescript
// Mock ở boundary
mockPaymentGateway.charge.mockResolvedValue({ success: true });

// Assert behavior, không implementation
expect(result.status).toBe('completed');
expect(mockEmail.send).toHaveBeenCalledWith(
  expect.objectContaining({ to: 'user@example.com' })
);

// Reset mocks mỗi test
beforeEach(() => jest.clearAllMocks());

// Dùng typed mocks
const mockRepo = {
  findById: jest.fn<Promise<User | null>, [string]>(),
};
```

### Mock Checklist

- [ ] Mock chỉ external dependencies và I/O
- [ ] Mỗi test setup mock behavior riêng
- [ ] Assert output/behavior, không assert call order (trừ khi order quan trọng)
- [ ] `mockRestore()` sau `spyOn` nếu affect tests khác
- [ ] MSW `onUnhandledRequest: 'error'` để catch missing handlers

---

## Câu Hỏi Phỏng Vấn

**Q: Mock vs Stub vs Spy?**  
A: Stub trả về preset data. Spy ghi lại calls, có thể delegate to real. Mock = Spy + preset behavior + verification.

**Q: Khi nào dùng MSW thay jest.mock?**  
A: Khi test HTTP client code (fetch/axios) — MSW giữ real request flow. jest.mock phù hợp khi mock service class trực tiếp.

**Q: Mock có làm test kém giá trị?**  
A: Over-mocking có — test pass nhưng integration fail. Balance: unit test mock boundaries, integration test real deps.

**Q: Làm sao mock ES modules?**  
A: Jest: `jest.mock()` với factory. Vitest: `vi.mock()` + `vi.hoisted()`. Node native: `mock` property trong package.json hoặc test runner support.

**Q: Dependency Injection giúp testing thế nào?**  
A: Inject interfaces — swap real implementation bằng mock mà không thay đổi code under test. NestJS `overrideProvider` là ví dụ điển hình.

---

**Tiếp theo:** [6-e2e-testing.md](./6-e2e-testing.md) — E2E testing với Playwright và Newman
