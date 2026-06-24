# Top 50 Câu Hỏi Phỏng Vấn Node.js — Đáp Án Chi Tiết

> Bộ 50 câu hỏi phỏng vấn Node.js & JavaScript Backend được hỏi nhiều nhất, kèm đáp án chi tiết và liên kết tài liệu sâu hơn.

## Mục Lục

- [Phần 1: JavaScript & Event Loop (Câu 1–10)](#phần-1-javascript--event-loop-câu-110)
- [Phần 2: Node.js Runtime & Modules (Câu 11–15)](#phần-2-nodejs-runtime--modules-câu-1115)
- [Phần 3: API & Web Frameworks (Câu 16–25)](#phần-3-api--web-frameworks-câu-1625)
- [Phần 4: Database & Caching (Câu 26–35)](#phần-4-database--caching-câu-2635)
- [Phần 5: Security (Câu 36–42)](#phần-5-security-câu-3642)
- [Phần 6: Testing & DevOps (Câu 43–47)](#phần-6-testing--devops-câu-4347)
- [Phần 7: System Design & Architecture (Câu 48–50)](#phần-7-system-design--architecture-câu-4850)

---

## Phần 1: JavaScript & Event Loop (Câu 1–10)

### Câu 1: Event Loop hoạt động như thế nào?

**Đáp án ngắn:** Node.js chạy JavaScript trên single main thread. Event Loop liên tục kiểm tra Call Stack — khi rỗng, lấy callback từ queues và thực thi. Async I/O được delegate cho libuv thread pool, khi hoàn thành callback vào queue.

**Chi tiết:** Microtasks (Promise, `process.nextTick`) chạy trước macrotasks (`setTimeout`, `setImmediate`, I/O callbacks). Mỗi phase của Event Loop (timers → pending → idle → poll → check → close) xử lý loại callback riêng.

📖 Xem thêm: [1-event-loop-questions.md](./1-event-loop-questions.md) | [02-async-programming/1-event-loop.md](../02-async-programming/1-event-loop.md)

---

### Câu 2: `var`, `let`, `const` — khác biệt?

| | `var` | `let` | `const` |
| - | ----- | ----- | ------- |
| **Scope** | Function | Block | Block |
| **Hoisting** | Hoisted, undefined | Temporal dead zone | Temporal dead zone |
| **Reassign** | ✅ | ✅ | ❌ (object properties vẫn mutable) |

**Best practice:** Dùng `const` mặc định, `let` khi cần reassign, tránh `var`.

---

### Câu 3: Closure là gì? Ví dụ thực tế?

Closure là function "nhớ" lexical scope (phạm vi từ) nơi nó được tạo, kể cả khi outer function đã return.

```javascript
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    getCount: () => count,
  };
}
```

**Use case thực tế:** Module pattern, factory functions, middleware với shared state, debounce/throttle.

---

### Câu 4: `this` trong JavaScript — các trường hợp?

| Context | `this` |
| ------- | ------ |
| Global | `global` (Node) / `window` (browser) |
| Method call | Object gọi method |
| Arrow function | Lexical `this` (từ enclosing scope) |
| `new` constructor | Instance mới tạo |
| `call/apply/bind` | Object được chỉ định |

**Arrow function không có `this` riêng** — lý do nên dùng trong class methods và callbacks.

---

### Câu 5: `Promise.all` vs `Promise.allSettled` vs `Promise.race`?

- **`Promise.all`:** Tất cả resolve → array kết quả. Một reject → reject ngay (fail-fast).
- **`Promise.allSettled`:** Chờ tất cả, trả `[{status, value/reason}]` — không fail-fast.
- **`Promise.race`:** Kết quả của promise đầu tiên settle — dùng cho timeout pattern.

---

### Câu 6: `async/await` vs Promises `.then()`?

`async/await` là syntactic sugar trên Promises — behavior giống nhau, nhưng:
- `async/await`: Code đọc như sync, try/catch cho error handling
- `.then()`: Chaining, functional style

```javascript
// Tương đương:
const data = await fetchData();
// fetchData().then(data => ...);
```

**Lưu ý:** `await` trong loop = sequential. Dùng `Promise.all` cho parallel.

---

### Câu 7: `process.nextTick()` vs `setImmediate()`?

- **`process.nextTick()`:** Microtask queue, ưu tiên cao nhất, chạy trước Event Loop tiếp tục
- **`setImmediate()`:** Macrotask, check phase của Event Loop, chạy sau I/O callbacks

`nextTick` lặp vô hạn → starvation (Event Loop không tiếp tục).

---

### Câu 8: Unhandled Promise Rejection — xử lý?

```javascript
process.on('unhandledRejection', (reason) => {
  logger.error(reason);
  // Production: graceful shutdown
});
```

**Prevention:** Luôn `await` hoặc `.catch()`, ESLint `no-floating-promises`, wrap Express async handlers.

---

### Câu 9: Event Loop bị block — nguyên nhân và fix?

**Nguyên nhân:** Sync CPU-intensive code (JSON.parse file lớn, sync crypto, tight loop).

**Fix:**
- Worker Threads cho CPU-bound tasks
- `setImmediate` để yield control
- Cluster module cho multi-process
- Offload sang external service

---

### Câu 10: Giải thích output — Event Loop ordering?

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
process.nextTick(() => console.log('4'));
console.log('5');
// Output: 1, 5, 4, 3, 2
```

**Giải thích:** Sync (1,5) → nextTick (4) → microtask (3) → macrotask (2).

---

## Phần 2: Node.js Runtime & Modules (Câu 11–15)

### Câu 11: V8 Engine và libuv là gì?

- **V8:** JavaScript engine (Google), compile JS thành machine code, quản lý heap và GC (Garbage Collection — Thu Gom Rác)
- **libuv:** C library cung cấp Event Loop, thread pool (4 threads), async I/O abstractions cho Node.js

---

### Câu 12: CommonJS vs ESM — khác biệt?

| | CommonJS | ESM |
| - | -------- | --- |
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous, runtime | Asynchronous, static analysis |
| **Tree shaking** | Không | Có |
| **File ext** | `.js`, `.cjs` | `.mjs`, `"type": "module"` |

Node.js hỗ trợ cả hai. TypeScript/NestJS thường dùng ESM.

---

### Câu 13: `package.json` — fields quan trọng?

- **`name`, `version`:** SemVer (Semantic Versioning)
- **`main` / `exports`:** Entry point
- **`scripts`:** `start`, `dev`, `test`, `build`
- **`dependencies` vs `devDependencies`**
- **`engines`:** Node.js version requirement
- **`type: "module"`:** Enable ESM

---

### Câu 14: Stream (luồng dữ liệu) là gì? Khi nào dùng?

Stream xử lý data theo chunks thay vì load toàn bộ vào memory.

| Loại | Ví Dụ |
| ---- | ----- |
| Readable | `fs.createReadStream`, HTTP response |
| Writable | `fs.createWriteStream`, HTTP request |
| Transform | `zlib.createGzip`, encryption |
| Duplex | TCP socket, WebSocket |

**Dùng khi:** File lớn, real-time data, proxy/transformation pipeline.

---

### Câu 15: Cluster module — tại sao cần?

Node.js single-threaded → không tận dụng multi-core CPU.

```javascript
const cluster = require('cluster');
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
} else {
  require('./server.js');
}
```

**Lưu ý:** Shared state cần external store (Redis). PM2 cluster mode tự động hóa.

---

## Phần 3: API & Web Frameworks (Câu 16–25)

### Câu 16: Middleware trong Express hoạt động thế nào?

Function `(req, res, next)` chạy tuần tự. Gọi `next()` chuyển sang middleware tiếp. Gọi `next(err)` skip đến error handler. Response kết thúc chain.

📖 [2-api-design-questions.md](./2-api-design-questions.md)

---

### Câu 17: REST API design principles?

- Resource-based URLs (`/users/123`)
- HTTP methods đúng semantics (GET safe, POST create, PUT replace, PATCH partial, DELETE)
- Stateless — mỗi request self-contained
- Proper status codes
- Versioning strategy
- Pagination, filtering, sorting

---

### Câu 18: Error handling strategy cho production API?

1. Custom error classes (`AppError`, `NotFoundError`)
2. Operational vs programming errors
3. Global error handler middleware (4 params)
4. Structured error response `{ error: { message, code } }`
5. Log đầy đủ, không leak stack trace trong production

---

### Câu 19: Express vs Fastify vs NestJS?

| | Express | Fastify | NestJS |
| - | ------- | ------- | ------ |
| Speed | Baseline | 2–3x faster | ~Express |
| Structure | Flexible | Plugin-based | Opinionated |
| TS | Manual setup | Native | First-class |
| Best for | MVP, prototypes | High-throughput | Enterprise |

---

### Câu 20: Request validation — tại sao cần?

- Prevent invalid data vào business logic
- Security — reject malicious input sớm
- Type safety với Zod/class-validator
- Consistent error responses

```javascript
const schema = z.object({ email: z.string().email() });
const result = schema.safeParse(req.body);
```

---

### Câu 21: CORS là gì? Cách config?

Cross-Origin Resource Sharing — browser security policy. Server phải trả `Access-Control-Allow-Origin` header cho cross-origin requests.

```javascript
app.use(cors({ origin: 'https://myapp.com', credentials: true }));
```

---

### Câu 22: API pagination — cursor vs offset?

- **Offset:** `?page=2&limit=20` — đơn giản, chậm với offset lớn
- **Cursor:** `?cursor=abc&limit=20` — performance ổn định, phù hợp infinite scroll

---

### Câu 23: Idempotency trong API?

GET, PUT, DELETE idempotent. POST không. Payment systems dùng **Idempotency-Key** header để prevent duplicate charges.

---

### Câu 24: Webhook implementation best practices?

- Verify signature (HMAC)
- Respond 200 nhanh, process async (queue)
- Idempotent processing
- Retry handling từ sender
- Log payload cho debugging

---

### Câu 25: API versioning strategies?

URL path (`/v1/`), header (`Accept: application/vnd.api.v1+json`), query param. URL path phổ biến nhất cho public APIs.

---

## Phần 4: Database & Caching (Câu 26–35)

### Câu 26: N+1 Problem — giải thích và fix?

1 query lấy N records + N queries lấy related data.

**Fix:** Eager loading (`include`), JOIN, DataLoader batching.

📖 [3-database-questions.md](./3-database-questions.md)

---

### Câu 27: Connection pool sizing?

```
pool_size ≈ (num_cores × 2) + spindle_count
```

Node.js: 10–20 per instance. Tổng tất cả instances < PostgreSQL `max_connections`.

---

### Câu 28: ACID transactions trong Node.js?

```javascript
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } });
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } });
});
```

Rollback tự động nếu bất kỳ operation fail.

---

### Câu 29: SQL vs NoSQL — khi nào dùng gì?

- **PostgreSQL:** Structured data, relations, ACID, complex queries
- **MongoDB:** Flexible schema, document-heavy, horizontal scaling
- **Redis:** Caching, sessions, pub/sub, rate limiting

---

### Câu 30: Redis caching patterns?

- **Cache-aside:** App check cache → miss → DB → populate cache
- **Write-through:** Write cache + DB simultaneously
- **TTL:** Time-based expiration
- **Invalidation:** Delete cache key on update

---

### Câu 31: Database indexing — khi nào?

Index columns trong WHERE, JOIN, ORDER BY. Composite index cho multi-column queries. Trade-off: faster reads, slower writes, more storage.

---

### Câu 32: Optimistic vs Pessimistic locking?

- **Optimistic:** Version column, check before update, retry on conflict
- **Pessimistic:** `SELECT FOR UPDATE` lock row — cho high-contention scenarios

---

### Câu 33: Database migration best practices?

- Reversible migrations
- Không sửa migration đã deploy
- Backward compatible changes (expand-contract)
- Test trên staging với production-like data

---

### Câu 34: ORM vs Raw SQL?

ORM cho CRUD, relations, type safety. Raw SQL cho complex reports, bulk operations, performance-critical queries. Prisma hỗ trợ cả hai (`$queryRaw`).

---

### Câu 35: DataLoader pattern?

Batch multiple load requests trong cùng Event Loop tick → single DB query. Tạo mới per HTTP request. Giải quyết N+1 trong GraphQL resolvers.

---

## Phần 5: Security (Câu 36–42)

### Câu 36: JWT flow — access token + refresh token?

Login → access token (short-lived, 15min) + refresh token (long-lived, 7 days, stored server-side). API dùng access token. Expired → refresh endpoint → new tokens. Logout → invalidate refresh token.

📖 [4-security-questions.md](./4-security-questions.md)

---

### Câu 37: JWT vs Session — trade-offs?

JWT stateless, dễ scale, khó revoke. Session stateful, dễ revoke, cần shared store (Redis) cho multi-instance.

---

### Câu 38: Prevent SQL Injection trong Node.js?

Parameterized queries (`$1`, `?`), ORM, input validation. Không concatenate user input vào SQL string.

---

### Câu 39: OWASP Top 10 — áp dụng Node.js?

Broken Access Control → RBAC middleware. Injection → parameterized queries. Auth Failures → rate limit login, MFA. Security Misconfiguration → Helmet, disable `X-Powered-By`. Vulnerable Components → `npm audit`.

---

### Câu 40: Password hashing?

bcrypt (12 rounds) hoặc argon2. Không MD5/SHA1/plain text. Salt tự động trong bcrypt.

---

### Câu 41: Rate limiting implementation?

`express-rate-limit` với Redis store cho distributed systems. Token bucket hoặc sliding window algorithm. Return 429 với `Retry-After` header.

---

### Câu 42: HTTPS và security headers?

Helmet.js set: HSTS, CSP, X-Frame-Options, X-Content-Type-Options. TLS termination tại load balancer hoặc reverse proxy.

---

## Phần 6: Testing & DevOps (Câu 43–47)

### Câu 43: Unit test vs Integration test?

- **Unit:** Test function isolated, mock dependencies, nhanh (<1ms)
- **Integration:** Test API + DB + middleware, Supertest + Testcontainers, chậm hơn (100ms–5s)

Testing pyramid: nhiều unit, ít integration, rất ít E2E.

---

### Câu 44: Mocking strategies trong Jest?

`jest.mock()` module level, DI injection mock, MSW (Mock Service Worker) cho HTTP. Mock behavior, không mock implementation details.

---

### Câu 45: Docker multi-stage build cho Node.js?

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/main.js"]
```

Giảm image size, không include dev dependencies và source code.

---

### Câu 46: Health checks — liveness vs readiness?

- **Liveness:** Process còn sống? Fail → restart container
- **Readiness:** Sẵn sàng nhận traffic? Fail → remove from load balancer
- **Startup:** App đã khởi động xong? (cho slow-starting apps)

---

### Câu 47: CI/CD pipeline cho Node.js?

```
Push → Lint → Unit Test → Build → Integration Test → Docker Build → Deploy Staging → E2E Test → Deploy Production
```

GitHub Actions, automated rollback nếu health check fail sau deploy.

---

## Phần 7: System Design & Architecture (Câu 48–50)

### Câu 48: Thiết kế URL shortener với Node.js?

API (Express cluster) → Redis cache → PostgreSQL. Base62 encoding cho short code. Cache-aside pattern (99% hit rate). Read-heavy (1000:1 ratio) → Redis critical.

📖 [5-system-design-scenarios.md](./5-system-design-scenarios.md)

---

### Câu 49: Scale Node.js app lên 10,000 RPS?

1. **Horizontal scaling:** Multiple instances behind load balancer
2. **Cluster/PM2:** Multi-process per machine
3. **Caching:** Redis cho hot data
4. **DB optimization:** Connection pooling, read replicas, indexing
5. **CDN:** Static assets
6. **Offload:** CPU work → Worker Threads hoặc separate service
7. **Monitoring:** Identify bottlenecks trước khi scale

---

### Câu 50: Monolith vs Microservices — trade-offs?

| | Monolith | Microservices |
| - | -------- | ------------- |
| **Complexity** | Thấp | Cao (network, deployment, monitoring) |
| **Scaling** | Scale toàn bộ | Scale từng service |
| **Team** | Small team | Multiple teams |
| **Node.js fit** | Excellent starting point | API Gateway, BFF pattern |

**Khuyến nghị:** Bắt đầu monolith (modular), extract microservices khi có clear bounded context và scaling need.

---

## Bảng Tra Cứu Nhanh

| Chủ Đề | File Chi Tiết |
| ------ | ------------- |
| Event Loop & Async | [1-event-loop-questions.md](./1-event-loop-questions.md) |
| API & Frameworks | [2-api-design-questions.md](./2-api-design-questions.md) |
| Database & Caching | [3-database-questions.md](./3-database-questions.md) |
| Security | [4-security-questions.md](./4-security-questions.md) |
| System Design | [5-system-design-scenarios.md](./5-system-design-scenarios.md) |
| STAR Stories | [6-star-stories.md](./6-star-stories.md) |
| 90-Day Plan | [7-90-day-study-plan.md](./7-90-day-study-plan.md) |

---

**Cập Nhật:** 2026-06-24 | **Tổng:** 50 câu hỏi | **Trạng Thái:** ✅ Hoàn thành
