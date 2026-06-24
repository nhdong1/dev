# Kiểm Thử Node.js — Tổng Quan

> Chiến lược kiểm thử (testing strategy) cho Backend Node.js: Unit Testing (Kiểm Thử Đơn Vị), Integration Testing (Kiểm Thử Tích Hợp), E2E Testing (Kiểm Thử Đầu Cuối), mocking, coverage, và CI/CD integration.

## Mục Lục

1. [Tại Sao Testing Quan Trọng](#tại-sao-testing-quan-trọng)
2. [Testing Pyramid — Kim Tự Tháp Kiểm Thử](#testing-pyramid--kim-tự-tháp-kiểm-thử)
3. [Các Loại Test Trong Node.js Backend](#các-loại-test-trong-nodejs-backend)
4. [Chọn Test Runner Phù Hợp](#chọn-test-runner-phù-hợp)
5. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
6. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
7. [Bài Tập Thực Hành](#bài-tập-thực-hành)
8. [Testing Checklist Production](#testing-checklist-production)
9. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Testing Quan Trọng

Backend API xử lý **business logic (logic nghiệp vụ)**, **data persistence (lưu trữ dữ liệu)**, và **authentication (xác thực)** — mọi bug đều có thể ảnh hưởng trực tiếp đến user và doanh thu.

| Không Có Test | Có Test Suite Tốt |
| ------------- | ----------------- |
| Refactor sợ phá vỡ tính năng | Refactor tự tin với safety net |
| Bug phát hiện ở production | Bug bắt sớm trong CI pipeline |
| Regression (lỗi tái phát) thường xuyên | Regression được phát hiện tự động |
| Onboarding khó — phải đọc toàn bộ code | Test là living documentation |
| Deploy cuối tuần lo lắng | Deploy với confidence cao hơn |

**Nguyên tắc cốt lõi:** Test behavior (hành vi), không test implementation details (chi tiết triển khai). Test phải nhanh, độc lập (isolated), và repeatable (lặp lại được).

---

## Testing Pyramid — Kim Tự Tháp Kiểm Thử

```
                    ┌───────────┐
                    │    E2E    │  ← Ít nhất, chậm nhất, confidence cao nhất
                    │  (Few)    │
                ┌───┴───────────┴───┐
                │   Integration     │  ← Vừa phải — test nhiều component cùng nhau
                │    (Some)       │
            ┌───┴─────────────────┴───┐
            │      Unit Tests         │  ← Nhiều nhất, nhanh nhất, chi phí thấp
            │       (Many)            │
            └─────────────────────────┘
```

| Tầng | Mục Đích | Công Cụ Phổ Biến | Tốc Độ |
| ---- | -------- | ---------------- | ------ |
| **Unit** | Test function/class độc lập | Jest, Vitest | < 1ms/test |
| **Integration** | Test API + DB + middleware | Supertest, Testcontainers | 100ms–5s/test |
| **E2E** | Test toàn bộ user flow | Playwright, Newman | 5s–60s/test |

**Anti-pattern:** Ice Cream Cone — quá nhiều E2E, quá ít unit test → CI chậm, flaky tests (test không ổn định), khó debug.

---

## Các Loại Test Trong Node.js Backend

### Unit Test — Kiểm Thử Đơn Vị

Test **một đơn vị logic** (function, class method) với dependencies được mock.

```typescript
// service/order.service.ts
export function calculateTotal(items: { price: number; qty: number }[]): number {
  return items.reduce((sum, item) => sum + item.price * item.qty, 0);
}

// service/order.service.test.ts
describe('calculateTotal', () => {
  it('tính tổng đúng với nhiều items', () => {
    expect(calculateTotal([
      { price: 100, qty: 2 },
      { price: 50, qty: 1 },
    ])).toBe(250);
  });
});
```

### Integration Test — Kiểm Thử Tích Hợp

Test **nhiều layer cùng nhau** — thường là HTTP request → Express app → database thật (hoặc container).

```typescript
// Không mock database — dùng Testcontainers PostgreSQL
const response = await request(app)
  .post('/api/users')
  .send({ email: 'test@example.com', password: 'secret123' });

expect(response.status).toBe(201);
```

### E2E Test — Kiểm Thử Đầu Cuối

Test **toàn bộ flow** từ client perspective — có thể qua HTTP API hoặc browser.

---

## Chọn Test Runner Phù Hợp

| Test Runner | Ưu Điểm | Phù Hợp Khi |
| ----------- | ------- | ----------- |
| **Jest** | Ecosystem lớn, mocking built-in, snapshot | Express projects, CommonJS/ESM hybrid |
| **Vitest** | Nhanh (Vite-powered), ESM native, Jest-compatible API | Vite, NestJS mới, TypeScript-first |
| **Node.js Test Runner** | Zero dependency, built-in từ Node 18+ | Minimal projects, không cần mocking phức tạp |
| **Mocha + Chai** | Linh hoạt, mature | Legacy projects, custom setup |

**Khuyến nghị:** Bắt đầu với **Jest** (phổ biến nhất trong phỏng vấn), sau đó học **Vitest** cho projects hiện đại.

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-jest-basics.md](./1-jest-basics.md) | Unit testing, matchers, mocking, snapshots, coverage | 1.5 giờ |
| 2 | [2-vitest.md](./2-vitest.md) | Vitest setup, ESM, watch mode, so sánh Jest | 1 giờ |
| 3 | [3-supertest.md](./3-supertest.md) | HTTP assertion testing cho Express/Fastify API | 1 giờ |
| 4 | [4-testcontainers.md](./4-testcontainers.md) | Integration tests với real PostgreSQL/Redis | 1.5 giờ |
| 5 | [5-mocking-strategies.md](./5-mocking-strategies.md) | jest.mock, DI mocking, MSW, test doubles | 1 giờ |
| 6 | [6-e2e-testing.md](./6-e2e-testing.md) | Playwright API testing, Newman/Postman collections | 1 giờ |

---

## Các Tài Liệu Chi Tiết

| File | Chủ Đề Chính |
| ---- | ------------ |
| [1-jest-basics.md](./1-jest-basics.md) | `describe`/`it`, matchers, `beforeEach`, `jest.fn()`, coverage thresholds |
| [2-vitest.md](./2-vitest.md) | `vitest.config.ts`, `vi.mock()`, in-source testing, workspace mode |
| [3-supertest.md](./3-supertest.md) | `request(app)`, auth headers, testing middleware chain |
| [4-testcontainers.md](./4-testcontainers.md) | Docker containers trong test, database seeding, teardown |
| [5-mocking-strategies.md](./5-mocking-strategies.md) | Stub vs Mock vs Spy, MSW cho external APIs |
| [6-e2e-testing.md](./6-e2e-testing.md) | Playwright `request` API, Newman CLI, CI integration |

---

## Bài Tập Thực Hành

### Bài 1: Unit Test Service Layer (Cơ Bản)

1. Tạo `UserService` với method `validateEmail(email: string): boolean`
2. Viết unit test cover: valid email, invalid email, empty string
3. Đạt 100% coverage cho function đó

### Bài 2: API Integration Test với Supertest

1. Tạo Express app với `POST /api/users` và `GET /api/users/:id`
2. Mock database layer với `jest.mock()`
3. Test: 201 Created, 400 Bad Request, 404 Not Found

### Bài 3: Integration Test với Testcontainers

1. Spin up PostgreSQL container trong `beforeAll`
2. Run Prisma migrations
3. Test CRUD operations end-to-end
4. Tear down container trong `afterAll`

### Bài 4: MSW cho External API

1. Service gọi external payment API
2. Dùng MSW intercept HTTP requests
3. Test success và failure scenarios

---

## Testing Checklist Production

### Trước Khi Merge PR

- [ ] Unit tests pass locally (`npm test`)
- [ ] Coverage không giảm dưới threshold (thường 70–80%)
- [ ] Không có `test.only` hoặc `describe.only` committed
- [ ] Integration tests pass (nếu có Docker)
- [ ] Test names mô tả rõ behavior (`it('returns 404 when user not found')`)

### CI Pipeline

- [ ] Tests chạy trên mỗi PR (GitHub Actions / GitLab CI)
- [ ] Parallel test execution để giảm thời gian
- [ ] Test database riêng — không dùng production DB
- [ ] Flaky test policy — fix hoặc quarantine, không ignore

### Code Quality

- [ ] Test Arrange-Act-Assert (AAA) pattern
- [ ] Một assertion concept per test (tránh test quá nhiều thứ)
- [ ] Factory/fixture cho test data — không hardcode magic values
- [ ] `afterEach` cleanup — reset mocks, close connections

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Unit test vs Integration test? | Unit: isolated, fast, mock deps. Integration: real deps, slower, higher confidence |
| Khi nào mock, khi nào dùng real DB? | Mock cho unit test logic. Real DB (Testcontainers) cho integration test data layer |
| Jest vs Vitest? | Jest: mature, built-in mocking. Vitest: faster, ESM-native, Vite ecosystem |
| Làm sao test async code? | `async/await` trong test, `resolves`/`rejects` matchers |
| Flaky test — xử lý thế nào? | Tìm root cause (timing, shared state), không dùng `setTimeout` arbitrary |
| Test coverage bao nhiêu % là đủ? | 70–80% là baseline; 100% không phải mục tiêu — focus critical paths |
| TDD (Test-Driven Development) có nên dùng? | Hữu ích cho complex logic; không bắt buộc mọi project |
| Supertest test gì? | HTTP layer — status code, headers, response body qua Express app |

---

## Liên Kết Chủ Đề Liên Quan

- **Web Frameworks:** [03-web-frameworks/](../03-web-frameworks/) — Express app để test với Supertest
- **Data Access:** [04-data-access/](../04-data-access/) — Prisma, connection pool cho integration tests
- **Security:** [05-security/](../05-security/) — Test JWT auth middleware
- **CI/CD:** [09-cloud-deployment/6-cicd-pipeline.md](../09-cloud-deployment/6-cicd-pipeline.md) — Chạy tests trong pipeline

---

**Tiếp theo:** [1-jest-basics.md](./1-jest-basics.md) — Nền tảng unit testing với Jest
