# Testcontainers — Integration Tests với Real Databases

> Testcontainers cho phép spin up **real database containers** (PostgreSQL, Redis, MongoDB) trong integration tests — test với dependencies thật thay vì mocks, đảm bảo SQL queries, migrations, và transactions hoạt động đúng.

## Mục Lục

1. [Testcontainers Là Gì](#testcontainers-là-gì)
2. [Khi Nào Dùng Testcontainers](#khi-nào-dùng-testcontainers)
3. [Cài Đặt và Yêu Cầu](#cài-đặt-và-yêu-cầu)
4. [PostgreSQL với Testcontainers](#postgresql-với-testcontainers)
5. [Redis và MongoDB](#redis-và-mongodb)
6. [Kết Hợp với Prisma](#kết-hợp-với-prisma)
7. [Test Lifecycle và Cleanup](#test-lifecycle-và-cleanup)
8. [CI/CD Integration](#cicd-integration)
9. [Performance và Optimization](#performance-và-optimization)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Testcontainers Là Gì

Testcontainers là library khởi động **Docker containers** trong test lifecycle — mỗi test suite có database riêng, isolated, disposable.

```
┌─────────────────────────────────────────────────────────┐
│                    Test Process                          │
│                                                         │
│  beforeAll() ──► Start PostgreSQL Container (Docker)   │
│       │                                                 │
│       ▼                                                 │
│  Run Migrations ──► Seed Test Data                     │
│       │                                                 │
│       ▼                                                 │
│  Execute Tests (real SQL queries)                      │
│       │                                                 │
│       ▼                                                 │
│  afterAll() ──► Stop & Remove Container                │
└─────────────────────────────────────────────────────────┘
```

**Ưu điểm:** Test với real DB behavior (constraints, indexes, transactions).  
**Nhược điểm:** Chậm hơn unit tests, cần Docker, CI phải support Docker.

---

## Khi Nào Dùng Testcontainers

| Scenario | Unit Test + Mock | Testcontainers |
| -------- | ---------------- | -------------- |
| Business logic thuần | ✅ | ❌ Overkill |
| Repository layer với SQL | ❌ Mock miss edge cases | ✅ |
| Prisma migrations | ❌ | ✅ |
| Transaction rollback behavior | ❌ | ✅ |
| Complex JOIN queries | ❌ | ✅ |
| Redis cache invalidation | Mock đủ | ✅ Nếu logic phức tạp |

**Quy tắc:** Mock ở unit test layer. Dùng Testcontainers ở integration test layer cho data access code.

---

## Cài Đặt và Yêu Cầu

### Yêu Cầu

- Docker Desktop hoặc Docker Engine đang chạy
- Node.js 18+

```bash
npm install -D @testcontainers/postgresql @testcontainers/redis testcontainers
```

### Verify Docker

```bash
docker info   # Phải chạy được
docker pull postgres:16-alpine
```

---

## PostgreSQL với Testcontainers

### Basic Setup

```typescript
// test/setup/database.ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';

let container: StartedPostgreSqlContainer;
let pool: Pool;

export async function setupTestDatabase() {
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('testdb')
    .withUsername('test')
    .withPassword('test')
    .start();

  pool = new Pool({
    host: container.getHost(),
    port: container.getPort(),
    database: container.getDatabase(),
    user: container.getUsername(),
    password: container.getPassword(),
  });

  await runMigrations(pool);

  return { pool, connectionString: container.getConnectionUri() };
}

export async function teardownTestDatabase() {
  await pool?.end();
  await container?.stop();
}

async function runMigrations(pool: Pool) {
  await pool.query(`
    CREATE TABLE users (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      email VARCHAR(255) UNIQUE NOT NULL,
      name VARCHAR(255) NOT NULL,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );
  `);
}
```

### Integration Test

```typescript
// src/repositories/user.repository.integration.test.ts
import { setupTestDatabase, teardownTestDatabase } from '../../test/setup/database';
import { UserRepository } from './user.repository';
import { Pool } from 'pg';

describe('UserRepository Integration', () => {
  let pool: Pool;
  let repo: UserRepository;

  beforeAll(async () => {
    const setup = await setupTestDatabase();
    pool = setup.pool;
    repo = new UserRepository(pool);
  }, 60000); // Timeout 60s cho container startup

  afterAll(async () => {
    await teardownTestDatabase();
  });

  beforeEach(async () => {
    await pool.query('TRUNCATE users CASCADE');
  });

  it('tạo user và query lại', async () => {
    const created = await repo.create({
      email: 'test@example.com',
      name: 'Test User',
    });

    expect(created.id).toBeDefined();

    const found = await repo.findByEmail('test@example.com');
    expect(found?.name).toBe('Test User');
  });

  it('throw error khi duplicate email', async () => {
    await repo.create({ email: 'dup@example.com', name: 'First' });

    await expect(
      repo.create({ email: 'dup@example.com', name: 'Second' })
    ).rejects.toThrow();
  });

  it('transaction rollback khi error', async () => {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      await client.query(
        `INSERT INTO users (email, name) VALUES ($1, $2)`,
        ['tx@example.com', 'TX User']
      );
      await client.query('ROLLBACK');
    } finally {
      client.release();
    }

    const found = await repo.findByEmail('tx@example.com');
    expect(found).toBeNull();
  });
});
```

---

## Redis và MongoDB

### Redis Container

```typescript
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import Redis from 'ioredis';

let redisContainer: StartedRedisContainer;
let redis: Redis;

beforeAll(async () => {
  redisContainer = await new RedisContainer('redis:7-alpine').start();
  redis = new Redis({
    host: redisContainer.getHost(),
    port: redisContainer.getPort(),
  });
}, 30000);

afterAll(async () => {
  await redis.quit();
  await redisContainer.stop();
});

it('cache user data với TTL', async () => {
  await redis.setex('user:1', 60, JSON.stringify({ name: 'Test' }));

  const cached = await redis.get('user:1');
  expect(JSON.parse(cached!)).toEqual({ name: 'Test' });

  const ttl = await redis.ttl('user:1');
  expect(ttl).toBeLessThanOrEqual(60);
  expect(ttl).toBeGreaterThan(0);
});
```

### MongoDB Container

```typescript
import { MongoDBContainer, StartedMongoDBContainer } from '@testcontainers/mongodb';
import mongoose from 'mongoose';

let mongoContainer: StartedMongoDBContainer;

beforeAll(async () => {
  mongoContainer = await new MongoDBContainer('mongo:7').start();
  await mongoose.connect(mongoContainer.getConnectionString());
}, 60000);

afterAll(async () => {
  await mongoose.disconnect();
  await mongoContainer.stop();
});
```

---

## Kết Hợp với Prisma

```typescript
// test/setup/prisma.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { execSync } from 'child_process';

let container: StartedPostgreSqlContainer;

export async function setupPrismaTestDb() {
  container = await new PostgreSqlContainer('postgres:16-alpine').start();

  const databaseUrl = container.getConnectionUri();
  process.env.DATABASE_URL = databaseUrl;

  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: databaseUrl },
    stdio: 'inherit',
  });

  const { PrismaClient } = await import('@prisma/client');
  return new PrismaClient();
}

export async function teardownPrismaTestDb(prisma: PrismaClient) {
  await prisma.$disconnect();
  await container.stop();
}
```

```typescript
// src/services/user.service.integration.test.ts
import { setupPrismaTestDb, teardownPrismaTestDb } from '../../test/setup/prisma';
import { PrismaClient } from '@prisma/client';
import { UserService } from './user.service';

describe('UserService + Prisma', () => {
  let prisma: PrismaClient;
  let service: UserService;

  beforeAll(async () => {
    prisma = await setupPrismaTestDb();
    service = new UserService(prisma);
  }, 90000);

  afterAll(async () => {
    await teardownPrismaTestDb(prisma);
  });

  beforeEach(async () => {
    await prisma.user.deleteMany();
  });

  it('tạo user qua service layer', async () => {
    const user = await service.register({
      email: 'prisma@example.com',
      password: 'secret123',
      name: 'Prisma User',
    });

    const count = await prisma.user.count();
    expect(count).toBe(1);
    expect(user.email).toBe('prisma@example.com');
  });
});
```

### Supertest + Testcontainers — Full Stack

```typescript
import request from 'supertest';
import { createApp } from '../app';
import { setupPrismaTestDb, teardownPrismaTestDb } from '../../test/setup/prisma';

describe('User API Integration', () => {
  let prisma: PrismaClient;
  let app: Express;

  beforeAll(async () => {
    prisma = await setupPrismaTestDb();
    app = createApp({ prisma });
  }, 90000);

  afterAll(async () => {
    await teardownPrismaTestDb(prisma);
  });

  it('POST /api/users → GET /api/users/:id', async () => {
    const createRes = await request(app)
      .post('/api/users')
      .send({ email: 'api@example.com', name: 'API User', password: 'secret' })
      .expect(201);

    const userId = createRes.body.id;

    const getRes = await request(app)
      .get(`/api/users/${userId}`)
      .expect(200);

    expect(getRes.body.email).toBe('api@example.com');
  });
});
```

---

## Test Lifecycle và Cleanup

### Shared Container — Nhanh Hơn

```typescript
// test/global-setup.ts — Chạy 1 lần cho toàn bộ test suite
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { writeFileSync } from 'fs';

export default async function globalSetup() {
  const container = await new PostgreSqlContainer('postgres:16-alpine').start();
  const uri = container.getConnectionUri();

  writeFileSync('.test-db-uri', uri);
  writeFileSync('.test-container-id', container.getId());

  process.env.DATABASE_URL = uri;
}

// test/global-teardown.ts
export default async function globalTeardown() {
  const containerId = readFileSync('.test-container-id', 'utf-8');
  execSync(`docker stop ${containerId}`);
}
```

```typescript
// jest.config.ts
export default {
  globalSetup: './test/global-setup.ts',
  globalTeardown: './test/global-teardown.ts',
  setupFilesAfterEnv: ['./test/setup-each.ts'],
};
```

### Transaction Rollback per Test — Isolation

```typescript
// Mỗi test chạy trong transaction, rollback sau test
let client: PoolClient;

beforeEach(async () => {
  client = await pool.connect();
  await client.query('BEGIN');
  repo = new UserRepository(client);
});

afterEach(async () => {
  await client.query('ROLLBACK');
  client.release();
});
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Integration Tests

on: [push, pull_request]

jobs:
  integration:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Run integration tests
        run: npm run test:integration
        env:
          CI: true

      # Testcontainers tự detect Docker trên GitHub Actions runners
```

### GitLab CI

```yaml
integration-test:
  image: node:20
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  script:
    - npm ci
    - npm run test:integration
```

### Ryuk Container — Auto Cleanup

Testcontainers tự động start **Ryuk** container để cleanup orphaned containers khi test process crash — không cần manual cleanup trong hầu hết cases.

---

## Performance và Optimization

| Strategy | Impact | Trade-off |
| -------- | ------ | --------- |
| **Shared container** (globalSetup) | Startup 1 lần | Tests share state — cần cleanup |
| **Alpine images** | Pull/start nhanh hơn | Đủ cho hầu hết tests |
| **Transaction rollback** | Không cần TRUNCATE | Chỉ work với 1 connection |
| **Parallel workers = 1** | Tránh port conflicts | Chậm hơn |
| **Reuse container** `.withReuse()` | Giữ container giữa runs | Chỉ local dev |

```typescript
// Reuse container giữa test runs (local dev only)
const container = await new PostgreSqlContainer('postgres:16-alpine')
  .withReuse()
  .start();
```

```json
// package.json — Tách integration tests
{
  "scripts": {
    "test": "jest --testPathIgnorePatterns=integration",
    "test:integration": "jest --testPathPattern=integration --runInBand --forceExit"
  }
}
```

---

## Best Practices

### DO ✅

- Tăng timeout cho `beforeAll` (30–90s) — container startup mất thời gian
- Dùng `--runInBand` khi parallel gây conflicts
- Seed minimal test data trong `beforeEach`
- Test constraints, unique indexes, foreign keys — điều mocks bỏ qua
- Tách integration tests khỏi unit tests trong CI (có thể chạy parallel jobs)

### DON'T ❌

- Đừng dùng production database cho tests
- Đừng hardcode ports — dùng `container.getPort()` (dynamic port)
- Đừng skip cleanup — dù Ryuk giúp, vẫn nên `stop()` trong `afterAll`
- Đừng chạy integration tests trong watch mode thường xuyên — quá chậm

---

## Câu Hỏi Phỏng Vấn

**Q: Testcontainers vs in-memory database (SQLite)?**  
A: SQLite nhanh hơn nhưng behavior khác PostgreSQL (types, functions, constraints). Testcontainers đảm bảo test trên DB production-like.

**Q: Làm sao tăng tốc integration tests?**  
A: Shared container, transaction rollback, tách khỏi unit tests, chạy integration chỉ trên main branch hoặc nightly.

**Q: CI không có Docker?**  
A: Dùng GitHub Actions (có Docker), hoặc fallback sang mock/repository abstraction cho CI lightweight.

**Q: Testcontainers có support Kafka, RabbitMQ?**  
A: Có — `@testcontainers/kafka`, `@testcontainers/rabbitmq`, và generic `GenericContainer` cho bất kỳ Docker image.

**Q: Khi nào KHÔNG dùng Testcontainers?**  
A: Pure unit tests, CI không có Docker, tests cần chạy < 100ms, hoặc logic không touch database.

---

**Tiếp theo:** [5-mocking-strategies.md](./5-mocking-strategies.md) — Chiến lược mock dependencies hiệu quả
