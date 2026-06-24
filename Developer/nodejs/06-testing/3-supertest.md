# Supertest — HTTP Assertion Testing cho API Endpoints

> Supertest cho phép test HTTP layer của Express/Fastify app mà **không cần start server thật** — gửi request trực tiếp vào app instance và assert response. Đây là công cụ chuẩn cho API integration testing trong Node.js.

## Mục Lục

1. [Supertest Là Gì](#supertest-là-gì)
2. [Cài Đặt](#cài-đặt)
3. [Test Express API Cơ Bản](#test-express-api-cơ-bản)
4. [Testing CRUD Endpoints](#testing-crud-endpoints)
5. [Authentication Testing](#authentication-testing)
6. [Error Handling Tests](#error-handling-tests)
7. [Testing Middleware](#testing-middleware)
8. [Fastify với Supertest](#fastify-với-supertest)
9. [NestJS Testing Module](#nestjs-testing-module)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Supertest Là Gì

Supertest wrap HTTP assertions library (superagent) để test web frameworks. App được pass vào `request(app)` — Supertest simulate HTTP request internally qua Node.js `http` module.

```
┌─────────────┐     request(app)      ┌─────────────┐
│  Test File  │ ────────────────────► │ Express App │
│  (Jest)     │                       │ (in-memory) │
└─────────────┘                       └──────┬──────┘
       ▲                                     │
       │         response (status, body)      │
       └─────────────────────────────────────┘
```

**Ưu điểm:** Không bind port, không race conditions, nhanh, parallel-safe.  
**Giới hạn:** Không test WebSocket, không test real network issues.

---

## Cài Đặt

```bash
npm install -D supertest @types/supertest
```

### Express App Structure cho Testing

```typescript
// src/app.ts — Export app, KHÔNG listen ở đây
import express from 'express';
import { userRouter } from './routes/user.routes';
import { errorHandler } from './middleware/error.middleware';

export function createApp() {
  const app = express();
  app.use(express.json());
  app.use('/api/users', userRouter);
  app.use(errorHandler);
  return app;
}

// src/server.ts — Chỉ file này listen
import { createApp } from './app';

const app = createApp();
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server on port ${PORT}`));
```

**Quan trọng:** Tách `createApp()` để test import app mà không start server.

---

## Test Express API Cơ Bản

```typescript
// src/routes/health.routes.test.ts
import request from 'supertest';
import { createApp } from '../app';

const app = createApp();

describe('Health Check', () => {
  it('GET /health returns 200', async () => {
    const response = await request(app)
      .get('/health')
      .expect(200);

    expect(response.body).toEqual({ status: 'ok' });
  });

  it('response có Content-Type application/json', async () => {
    await request(app)
      .get('/health')
      .expect('Content-Type', /json/);
  });
});
```

### Chaining Assertions

```typescript
it('POST /api/users tạo user mới', async () => {
  const response = await request(app)
    .post('/api/users')
    .send({ email: 'new@example.com', name: 'New User' })
    .set('Accept', 'application/json')
    .expect('Content-Type', /json/)
    .expect(201);

  expect(response.body).toMatchObject({
    email: 'new@example.com',
    name: 'New User',
  });
  expect(response.body.id).toBeDefined();
});
```

### expect() Shorthand

```typescript
// Cả hai cách tương đương
await request(app).get('/health').expect(200);
await request(app).get('/health').expect('Content-Type', /json/).expect(200);
```

---

## Testing CRUD Endpoints

```typescript
// src/routes/user.routes.test.ts
import request from 'supertest';
import { createApp } from '../app';

jest.mock('../repositories/user.repository');

import * as userRepo from '../repositories/user.repository';

const app = createApp();

describe('User API', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('GET /api/users/:id', () => {
    it('returns 200 và user data', async () => {
      jest.mocked(userRepo.findById).mockResolvedValue({
        id: '1',
        email: 'test@example.com',
        name: 'Test',
      });

      const response = await request(app)
        .get('/api/users/1')
        .expect(200);

      expect(response.body.email).toBe('test@example.com');
    });

    it('returns 404 khi user không tồn tại', async () => {
      jest.mocked(userRepo.findById).mockResolvedValue(null);

      const response = await request(app)
        .get('/api/users/999')
        .expect(404);

      expect(response.body.error).toBe('User not found');
    });
  });

  describe('POST /api/users', () => {
    it('returns 201 khi tạo thành công', async () => {
      jest.mocked(userRepo.create).mockResolvedValue({
        id: '2',
        email: 'new@example.com',
        name: 'New',
      });

      await request(app)
        .post('/api/users')
        .send({ email: 'new@example.com', name: 'New' })
        .expect(201);
    });

    it('returns 400 khi thiếu email', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({ name: 'No Email' })
        .expect(400);

      expect(response.body.errors).toContainEqual(
        expect.objectContaining({ field: 'email' })
      );
    });

    it('returns 409 khi email đã tồn tại', async () => {
      jest.mocked(userRepo.create).mockRejectedValue(
        new Error('DUPLICATE_EMAIL')
      );

      await request(app)
        .post('/api/users')
        .send({ email: 'exists@example.com', name: 'Dup' })
        .expect(409);
    });
  });

  describe('DELETE /api/users/:id', () => {
    it('returns 204 khi xóa thành công', async () => {
      jest.mocked(userRepo.delete).mockResolvedValue(true);

      await request(app)
        .delete('/api/users/1')
        .expect(204);
    });
  });
});
```

---

## Authentication Testing

### JWT Bearer Token

```typescript
import jwt from 'jsonwebtoken';

const JWT_SECRET = 'test-secret';

function generateTestToken(payload: object, expiresIn = '1h') {
  return jwt.sign(payload, JWT_SECRET, { expiresIn });
}

describe('Protected Routes', () => {
  it('returns 401 khi không có token', async () => {
    await request(app)
      .get('/api/profile')
      .expect(401);
  });

  it('returns 401 khi token invalid', async () => {
    await request(app)
      .get('/api/profile')
      .set('Authorization', 'Bearer invalid-token')
      .expect(401);
  });

  it('returns 200 với valid token', async () => {
    const token = generateTestToken({ sub: 'user-1', roles: ['user'] });

    const response = await request(app)
      .get('/api/profile')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(response.body.id).toBe('user-1');
  });

  it('returns 403 khi thiếu permission', async () => {
    const token = generateTestToken({ sub: 'user-1', roles: ['user'] });

    await request(app)
      .delete('/api/admin/users/1')
      .set('Authorization', `Bearer ${token}`)
      .expect(403);
  });
});
```

### Cookie-based Auth

```typescript
it('login sets HttpOnly cookie', async () => {
  const agent = request.agent(app); // Giữ cookies giữa requests

  await agent
    .post('/auth/login')
    .send({ email: 'test@example.com', password: 'secret' })
    .expect(200);

  const cookies = agent.jar.getCookies({ path: '/' });
  expect(cookies.some(c => c.name === 'sessionId')).toBe(true);

  // Request tiếp theo tự động gửi cookie
  await agent
    .get('/api/profile')
    .expect(200);
});
```

---

## Error Handling Tests

```typescript
describe('Error Handling', () => {
  it('returns structured error response', async () => {
    const response = await request(app)
      .get('/api/users/invalid-id')
      .expect(400);

    expect(response.body).toMatchObject({
      error: expect.any(String),
      statusCode: 400,
      timestamp: expect.any(String),
    });
  });

  it('không leak stack trace trong production', async () => {
    process.env.NODE_ENV = 'production';
    jest.mocked(userRepo.findById).mockRejectedValue(new Error('DB connection failed'));

    const response = await request(app)
      .get('/api/users/1')
      .expect(500);

    expect(response.body.stack).toBeUndefined();
    expect(response.body.error).toBe('Internal Server Error');

    process.env.NODE_ENV = 'test';
  });

  it('validation errors trả về 422 với field details', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'not-an-email' })
      .expect(422);

    expect(response.body.errors).toEqual(
      expect.arrayContaining([
        expect.objectContaining({
          field: 'email',
          message: expect.any(String),
        }),
      ])
    );
  });
});
```

---

## Testing Middleware

```typescript
// Test middleware độc lập
import { rateLimitMiddleware } from '../middleware/rate-limit.middleware';
import express from 'express';
import request from 'supertest';

describe('Rate Limit Middleware', () => {
  const testApp = express();
  testApp.use(rateLimitMiddleware({ windowMs: 60000, max: 3 }));
  testApp.get('/test', (req, res) => res.json({ ok: true }));

  it('allows requests under limit', async () => {
    for (let i = 0; i < 3; i++) {
      await request(testApp).get('/test').expect(200);
    }
  });

  it('returns 429 khi vượt limit', async () => {
    for (let i = 0; i < 3; i++) {
      await request(testApp).get('/test');
    }
    await request(testApp).get('/test').expect(429);
  });
});
```

### Request ID Middleware

```typescript
it('sets X-Request-Id header', async () => {
  const response = await request(app).get('/health');

  expect(response.headers['x-request-id']).toBeDefined();
  expect(response.headers['x-request-id']).toMatch(
    /^[0-9a-f-]{36}$/  // UUID format
  );
});
```

---

## Fastify với Supertest

```typescript
import Fastify from 'fastify';
import request from 'supertest';

async function buildApp() {
  const app = Fastify();
  app.get('/health', async () => ({ status: 'ok' }));
  await app.ready();
  return app;
}

describe('Fastify API', () => {
  let app: Awaited<ReturnType<typeof buildApp>>;

  beforeAll(async () => {
    app = await buildApp();
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /health', async () => {
    // Fastify expose .server property
    const response = await request(app.server)
      .get('/health')
      .expect(200);

    expect(response.body.status).toBe('ok');
  });
});
```

---

## NestJS Testing Module

NestJS có testing utilities riêng — thường dùng thay Supertest trực tiếp.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import { AppModule } from '../src/app.module';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    })
      .overrideProvider(UserRepository)
      .useValue(mockUserRepository)
      .compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('/api/users (GET)', () => {
    return request(app.getHttpServer())
      .get('/api/users')
      .expect(200)
      .expect((res) => {
        expect(Array.isArray(res.body)).toBe(true);
      });
  });
});
```

---

## Best Practices

### DO ✅

```typescript
// Factory function cho test app
function createTestApp(overrides?: Partial<AppConfig>) {
  return createApp({ ...defaultConfig, ...overrides });
}

// Helper cho auth headers
function authRequest(app: Express, token: string) {
  return request(app).set('Authorization', `Bearer ${token}`);
}

// Test data factories
const userFactory = (overrides = {}) => ({
  email: 'test@example.com',
  name: 'Test User',
  ...overrides,
});
```

### DON'T ❌

```typescript
// Đừng start server trong test
app.listen(3000); // BAD — port conflicts, slow

// Đừng dùng real external APIs
await request(app).post('/api/payment'); // BAD nếu gọi Stripe thật

// Đừng share mutable state giữa tests
let createdUserId: string; // BAD — order-dependent tests
```

### File Organization

```
src/
├── app.ts
├── routes/
│   ├── user.routes.ts
│   └── user.routes.test.ts    ← Co-located
└── __tests__/
    └── integration/
        └── api.integration.test.ts  ← Full flow tests
```

---

## Câu Hỏi Phỏng Vấn

**Q: Supertest test ở layer nào?**  
A: HTTP/Integration layer — test routing, middleware, request/response. Không test business logic thuần (dùng unit test).

**Q: Supertest có start server không?**  
A: Không — pass app instance, Supertest dùng Node.js `http.Server` internally mà không bind port.

**Q: Làm sao test file upload?**  
A: `.attach('fieldName', 'path/to/file')` hoặc `.attach('fieldName', Buffer.from('content'), 'filename.txt')`.

**Q: Supertest vs Postman/Newman?**  
A: Supertest: code-based, CI-friendly, mock dependencies. Newman: collection-based, test against running server, phù hợp E2E/smoke tests.

**Q: Test async middleware error?**  
A: Đảm bảo error handler middleware được mount. Supertest catch unhandled errors và return 500.

```typescript
it('upload file', async () => {
  await request(app)
    .post('/api/upload')
    .attach('file', Buffer.from('test content'), 'test.txt')
    .expect(200);
});
```

---

**Tiếp theo:** [4-testcontainers.md](./4-testcontainers.md) — Integration tests với real databases trong Docker
