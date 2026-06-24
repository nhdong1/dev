# Node.js & JavaScript — Chỉ Mục Toàn Diện

> Bản đồ điều hướng toàn bộ kiến thức Node.js, JavaScript Backend, và hệ sinh thái xung quanh

## 📁 Cấu Trúc Thư Mục

```
Developer/nodejs/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục đầy đủ (file này)
│
├── 01-fundamentals/                            Nền tảng JavaScript & Node.js
│   ├── README.md                               Tổng quan chủ đề nền tảng
│   ├── 1-javascript-es6-plus.md                let/const, arrow functions, destructuring, spread
│   ├── 2-nodejs-runtime.md                     V8, libuv, process model, global objects
│   ├── 3-module-systems.md                     CommonJS vs ESM, import/export, dynamic import
│   ├── 4-npm-ecosystem.md                      package.json, SemVer, scripts, workspaces
│   ├── 5-typescript-basics.md                  Type system, interfaces, generics, utility types
│   └── 6-builtin-modules.md                    fs, path, crypto, http, events, buffer
│
├── 02-async-programming/                       Lập trình bất đồng bộ
│   ├── README.md                               Tổng quan async patterns
│   ├── 1-event-loop.md                         Event loop phases, call stack, microtasks
│   ├── 2-callbacks.md                          Callback pattern, callback hell, error-first
│   ├── 3-promises.md                           Promise API, chaining, combinators
│   ├── 4-async-await.md                        async/await syntax, parallel vs sequential
│   ├── 5-error-handling.md                     try/catch, unhandled rejection, process handlers
│   └── 6-concurrency-patterns.md               Throttle, debounce, queue, batch processing
│
├── 03-web-frameworks/                          Web Frameworks & REST API
│   ├── README.md                               So sánh Express, Fastify, NestJS
│   ├── 1-express-basics.md                     Middleware, routing, request/response
│   ├── 2-express-advanced.md                   Error handling, validation, file upload
│   ├── 3-fastify.md                            Schema validation, plugins, performance
│   ├── 4-nestjs.md                             Modules, DI, guards, interceptors, pipes
│   ├── 5-rest-api-design.md                    Resource naming, HTTP methods, versioning
│   ├── 6-request-validation.md                 Zod, Joi, class-validator
│   └── 7-api-documentation.md                    Swagger/OpenAPI, Scalar, API versioning
│
├── 04-data-access/                             Truy cập dữ liệu
│   ├── README.md                               Tổng quan data layer strategies
│   ├── 1-postgresql-pg.md                      pg driver, connection pool, parameterized queries
│   ├── 2-prisma-orm.md                         Schema, migrations, Prisma Client, relations
│   ├── 3-typeorm.md                            Entity, repository, query builder, relations
│   ├── 4-sequelize.md                          Models, associations, hooks, migrations
│   ├── 5-mongodb-mongoose.md                   Document schema, aggregation, indexing
│   ├── 6-redis-caching.md                      ioredis, cache patterns, pub/sub, sessions
│   ├── 7-transactions.md                       ACID transactions, optimistic locking
│   └── 8-n-plus-one.md                         Eager loading, DataLoader, query optimization
│
├── 05-security/                                Bảo mật
│   ├── README.md                               Tổng quan bảo mật Node.js API
│   ├── 1-authentication-jwt.md                 JWT access/refresh tokens, token rotation
│   ├── 2-passport-strategies.md                Local, JWT, OAuth2, social login strategies
│   ├── 3-authorization-rbac.md                 Role-based access control, permissions
│   ├── 4-owasp-top10.md                        SQL injection, XSS, CSRF prevention
│   ├── 5-helmet-rate-limiting.md               Security headers, rate limiting algorithms
│   ├── 6-input-validation.md                   Sanitization, whitelist, SQL parameterization
│   └── 7-secrets-management.md                 env vars, Vault, AWS Secrets Manager
│
├── 06-testing/                                 Kiểm thử
│   ├── README.md                               Chiến lược kiểm thử Node.js
│   ├── 1-jest-basics.md                        Unit testing, mocking, snapshots, coverage
│   ├── 2-vitest.md                             Fast test runner cho ESM/Vite projects
│   ├── 3-supertest.md                          HTTP assertion testing cho API
│   ├── 4-testcontainers.md                     Integration tests với real databases
│   ├── 5-mocking-strategies.md                 jest.mock, DI mocking, MSW
│   └── 6-e2e-testing.md                        Playwright API testing, Newman/Postman
│
├── 07-performance/                             Hiệu năng
│   ├── README.md                               Phương pháp tối ưu hiệu năng Node.js
│   ├── 1-event-loop-monitoring.md              clinic.js, 0x flame graphs, event loop lag
│   ├── 2-memory-management.md                  Heap snapshots, GC, memory leak detection
│   ├── 3-cluster-worker-threads.md           Multi-process scaling, CPU offloading
│   ├── 4-caching-strategies.md               Cache-aside, write-through, TTL patterns
│   ├── 5-connection-pooling.md                 DB pool sizing, pool exhaustion
│   ├── 6-load-testing.md                       k6, Artillery, throughput benchmarks
│   └── 7-profiling.md                          --inspect, Chrome DevTools, perf_hooks
│
├── 08-architecture/                            Kiến trúc ứng dụng
│   ├── README.md                               Tổng quan kiến trúc Node.js
│   ├── 1-clean-architecture.md                 Layers, dependency rule, use cases
│   ├── 2-hexagonal-architecture.md             Ports & adapters pattern
│   ├── 3-design-patterns.md                    Singleton, factory, observer, repository
│   ├── 4-microservices.md                      Service decomposition, API gateway
│   ├── 5-event-driven.md                       Event sourcing, CQRS, saga pattern
│   ├── 6-monorepo.md                           Turborepo, Nx, pnpm workspaces
│   └── 7-error-handling-architecture.md        Result pattern, global error boundary
│
├── 09-cloud-deployment/                        Triển khai & Observability
│   ├── README.md                               Tổng quan deployment & production readiness
│   ├── 1-docker-nodejs.md                      Multi-stage Dockerfile, layer caching
│   ├── 2-pm2-process-manager.md                Cluster mode, zero-downtime reload
│   ├── 3-kubernetes-deployment.md              Deployment, Service, ConfigMap, HPA, probes
│   ├── 4-logging.md                            Pino/Winston, structured logging, ELK/Loki
│   ├── 5-metrics-tracing.md                    Prometheus, Grafana, OpenTelemetry, Jaeger
│   ├── 6-cicd-pipeline.md                      GitHub Actions, automated rollback
│   └── 7-production-checklist.md               Pre-deployment checklist
│
├── 10-advanced/                                Chủ đề nâng cao
│   ├── README.md                               Tổng quan chủ đề nâng cao
│   ├── 1-streams-api.md                        Readable, writable, transform, pipeline
│   ├── 2-child-processes.md                    spawn, exec, fork, IPC
│   ├── 3-graphql.md                            Apollo Server, resolvers, DataLoader
│   ├── 4-websockets.md                         Socket.io, ws, real-time broadcasting
│   ├── 5-grpc.md                               Protocol Buffers, streaming RPC
│   ├── 6-serverless.md                         AWS Lambda, cold start optimization
│   └── 7-job-queues.md                         BullMQ, Agenda, scheduled tasks, retry
│
├── 11-interview-prep/                          Chuẩn bị phỏng vấn
│   ├── README.md                               Tổng quan ôn thi
│   ├── INTERVIEW_GUIDE.md                      Top 50 câu hỏi + đáp án chi tiết
│   ├── 1-event-loop-questions.md               Câu hỏi sâu về Event Loop & async
│   ├── 2-api-design-questions.md               REST API, middleware, error handling
│   ├── 3-database-questions.md                 ORM, N+1, transactions, caching
│   ├── 4-security-questions.md                 JWT, OWASP, rate limiting
│   ├── 5-system-design-scenarios.md            Bài toán thiết kế hệ thống
│   ├── 6-star-stories.md                       Template câu chuyện STAR
│   └── 7-90-day-study-plan.md                  Kế hoạch học 90 ngày

```

---

## ✅ Trạng Thái Nội Dung

| Chủ Đề                              | Thư Mục / File                        | Trạng Thái | Mức Độ      |
| ----------------------------------- | ------------------------------------- | ---------- | ----------- |
| **Tổng Quan & Lộ Trình**            | README.md                             | ✅         | Toàn diện   |
| **Chỉ Mục**                         | INDEX.md                              | ✅         | Toàn diện   |
| **Nền Tảng JavaScript & Node.js**   | 01-fundamentals/                      | ✅         | Toàn diện   |
| **Lập Trình Bất Đồng Bộ**           | 02-async-programming/                 | ✅         | Toàn diện   |
| **Web Frameworks & REST API**     | 03-web-frameworks/                    | ✅         | Toàn diện   |
| **Truy Cập Dữ Liệu**                | 04-data-access/                       | ✅         | Toàn diện   |
| **Bảo Mật**                         | 05-security/                          | ✅         | Toàn diện   |
| **Kiểm Thử**                        | 06-testing/                           | ✅         | Toàn diện   |
| **Hiệu Năng**                       | 07-performance/                       | ✅         | Toàn diện   |
| **Kiến Trúc**                       | 08-architecture/                      | ✅         | Toàn diện   |
| **Triển Khai & Observability**      | 09-cloud-deployment/                  | ✅         | Toàn diện   |
| **Chủ Đề Nâng Cao**                 | 10-advanced/                          | ✅         | Toàn diện   |
| **Chuẩn Bị Phỏng Vấn**              | 11-interview-prep/                    | ✅         | Toàn diện   |

---

## 🎯 Thứ Tự Tạo Nội Dung (Ưu Tiên)

### Ưu Tiên Cao — Core Node.js Skills

- [x] `01-fundamentals/` — JavaScript ES6+, Node.js runtime, modules, TypeScript
- [x] `02-async-programming/` — Event Loop, Promises, async/await (luôn được hỏi phỏng vấn)
- [x] `03-web-frameworks/` — Express.js, REST API design, validation
- [x] `04-data-access/` — Prisma/TypeORM, PostgreSQL, MongoDB, Redis
- [x] `11-interview-prep/` — Bộ câu hỏi phỏng vấn, system design, STAR stories, kế hoạch 90 ngày

### Ưu Tiên Trung Bình — Production Skills

- [x] `05-security/` — JWT, OWASP, rate limiting, secrets management
- [x] `06-testing/` — Jest, Supertest, integration tests
- [x] `07-performance/` — Memory leaks, profiling, cluster module
- [x] `09-cloud-deployment/` — Docker, PM2, K8s, observability

### Ưu Tiên Thấp — Advanced & Reference

- [x] `08-architecture/` — Clean architecture, microservices, design patterns
- [x] `10-advanced/` — Streams, GraphQL, WebSockets, gRPC
- [x] `03-web-frameworks/4-nestjs.md` — NestJS deep dive
- [ ] `GLOSSARY.md` — Bảng thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu học tập
- [ ] `ROADMAP.md` — Lộ trình chi tiết 90 ngày

---

## 🚀 Cách Sử Dụng Knowledge Base

### Tự Học

```
1. Bắt đầu với README.md
2. Chọn lộ trình (Beginner / Intermediate / Advanced)
3. Học tuần tự từ 01-fundamentals → 11-interview-prep
4. Viết code thực hành cho mỗi chủ đề
5. Xây dựng portfolio project (REST API hoàn chỉnh)
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Ôn sâu 02-async-programming/ (Event Loop — câu hỏi số 1)
3. Ôn 03-web-frameworks/ (Express middleware chain)
4. Ôn 04-data-access/ (N+1 problem, connection pooling)
5. Ôn 05-security/ (JWT flow, OWASP)
6. Chuẩn bị 2–3 câu chuyện STAR từ kinh nghiệm thực tế
7. Practice mock interview với đồng nghiệp
```

### Vai Trò Backend Developer

```
Dùng làm tài liệu tham khảo:
- Tích hợp DB: 04-data-access/
- Implement auth: 05-security/
- Debug production: 07-performance/
- Trước deploy: 09-cloud-deployment/production-checklist.md
- Thiết kế API: 03-web-frameworks/5-rest-api-design.md
```

### System Design

```
1. Đọc 08-architecture/ cho patterns
2. Dùng 07-performance/ cho scalability constraints của Node.js
3. Dùng 09-cloud-deployment/ cho deployment architecture
4. Dùng 10-advanced/ cho real-time và event-driven requirements
```

---

## 📊 Ước Tính Thời Gian Học

| Phần                          | Thời Gian     | Độ Khó   | Ưu Tiên    |
| ----------------------------- | ------------- | -------- | ---------- |
| Nền Tảng JavaScript & Node.js | 6–8 giờ       | ⭐       | Bắt buộc   |
| Lập Trình Bất Đồng Bộ         | 8–10 giờ      | ⭐⭐     | Bắt buộc   |
| Web Frameworks & REST API     | 10–12 giờ     | ⭐⭐     | Bắt buộc   |
| Truy Cập Dữ Liệu              | 10–12 giờ     | ⭐⭐     | Bắt buộc   |
| Bảo Mật                       | 6–8 giờ       | ⭐⭐     | Bắt buộc   |
| Kiểm Thử                      | 6–8 giờ       | ⭐⭐     | Nên học    |
| Hiệu Năng                     | 8–10 giờ      | ⭐⭐⭐   | Nên học    |
| Kiến Trúc                     | 8–10 giờ      | ⭐⭐⭐   | Nên học    |
| Triển Khai & Observability    | 6–8 giờ       | ⭐⭐     | Nên học    |
| Chủ Đề Nâng Cao               | 12–15 giờ     | ⭐⭐⭐   | Tùy chọn   |
| Chuẩn Bị Phỏng Vấn            | 8–10 giờ      | ⭐⭐     | Trước PV   |

**Tổng: 80–110 giờ cho kiến thức Node.js Backend toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Beginner (0–1 năm kinh nghiệm)

- [ ] JavaScript ES6+ cơ bản
- [ ] Hiểu Event Loop ở mức cơ bản
- [ ] Tạo REST API với Express
- [ ] Kết nối database đơn giản
- [ ] Unit test với Jest

**Thời gian nắm vững:** 2–3 tháng

### Intermediate (1–3 năm kinh nghiệm)

- [ ] Thiết kế REST API production-ready
- [ ] JWT authentication & RBAC
- [ ] Database query optimization
- [ ] Integration testing
- [ ] Docker deployment
- [ ] Redis caching

**Thời gian nắm vững:** 2–3 tháng để đi sâu

### Advanced (3–5+ năm kinh nghiệm)

- [ ] Microservices architecture
- [ ] Event-driven systems
- [ ] Production performance tuning
- [ ] Full observability stack
- [ ] System design cho high-traffic apps
- [ ] Security hardening & compliance

**Thời gian nắm vững:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                    | Vị Trí                                                                               |
| -------------------------- | ------------------------------------------------------------------------------------ |
| Tổng quan                  | [README.md](README.md)                                                               |
| Event Loop                 | [02-async-programming/1-event-loop.md](02-async-programming/1-event-loop.md)         |
| Express.js                 | [03-web-frameworks/1-express-basics.md](03-web-frameworks/1-express-basics.md)     |
| Prisma ORM                 | [04-data-access/2-prisma-orm.md](04-data-access/2-prisma-orm.md)                     |
| JWT Authentication         | [05-security/1-authentication-jwt.md](05-security/1-authentication-jwt.md)         |
| Jest Testing               | [06-testing/1-jest-basics.md](06-testing/1-jest-basics.md)                         |
| Memory Leaks               | [07-performance/2-memory-management.md](07-performance/2-memory-management.md)   |
| Clean Architecture         | [08-architecture/1-clean-architecture.md](08-architecture/1-clean-architecture.md) |
| Microservices              | [08-architecture/4-microservices.md](08-architecture/4-microservices.md)         |
| Docker Deployment          | [09-cloud-deployment/1-docker-nodejs.md](09-cloud-deployment/1-docker-nodejs.md)   |
| Câu hỏi phỏng vấn          | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md)       |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Tạo bản sao và theo dõi tiến độ:

```markdown
## Node.js Knowledge Completion

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [x] JavaScript ES6+ (6/6 topics)
- [x] Node.js runtime & built-in modules
- [x] CommonJS vs ESM
- [x] NPM & package management
- [x] TypeScript basics

### Giai Đoạn 2: Async & API (Tuần 3–6)

- [x] Event Loop deep understanding
- [x] Promises & async/await
- [x] Express.js middleware & routing
- [x] REST API design
- [x] Request validation (Zod/Joi)
- [x] Error handling strategy

### Giai Đoạn 3: Data & Security (Tuần 7–10)

- [x] PostgreSQL + Prisma/TypeORM
- [x] MongoDB + Mongoose
- [x] Redis caching
- [x] JWT authentication
- [x] OWASP security practices
- [x] Jest + Supertest

### Giai Đoạn 4: Production (Tuần 11–14)

- [x] Performance profiling
- [x] Memory leak debugging
- [x] Docker containerization
- [x] PM2 / K8s deployment
- [x] Logging & monitoring setup
- [x] CI/CD pipeline

### Giai Đoạn 5: Advanced (Tuần 15+)

- [ ] NestJS framework
- [x] Microservices patterns
- [x] GraphQL / WebSockets
- [x] System design practice
- [x] Mock interviews (tài liệu hướng dẫn)
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base, bạn nên có khả năng:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích Event Loop mà không cần nhìn tài liệu
- [ ] Viết REST API hoàn chỉnh với auth, validation, tests
- [ ] Chọn đúng ORM/database cho use case
- [ ] Debug memory leak bằng heap snapshot
- [ ] Thiết kế caching strategy phù hợp

### ✅ Năng Lực Vận Hành

- [ ] Troubleshoot API chậm một cách có hệ thống
- [ ] Deploy Node.js app lên production an toàn
- [ ] Implement observability stack (logs + metrics + traces)
- [ ] Xử lý common production incidents
- [ ] Security hardening cho API endpoints

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 50 câu hỏi Node.js
- [ ] Kể 2–3 câu chuyện STAR từ kinh nghiệm thực tế
- [ ] Thiết kế hệ thống với Node.js constraints
- [ ] Giải thích trade-offs (Express vs Fastify, SQL vs NoSQL, sync vs async)
- [ ] Live coding async patterns và API endpoints

---

## 🚀 Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp
3. Thiết lập môi trường (Node.js LTS + Docker)
4. Bắt đầu `01-fundamentals/`

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/` và `02-async-programming/`
2. Viết REST API đầu tiên với Express
3. Kết nối PostgreSQL với Prisma
4. Viết unit test cơ bản

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành core topics (03–06)
2. Implement JWT authentication
3. Dockerize ứng dụng
4. Chuẩn bị 1–2 câu chuyện STAR

### Dài Hạn (3 Tháng Tới)

1. Hoàn thành toàn bộ knowledge base
2. Xây dựng portfolio project production-ready
3. Practice system design scenarios
4. Mock interviews với đồng nghiệp

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng code:** Đừng chỉ đọc — viết code, gây lỗi, sửa lỗi
2. **Event Loop là chìa khóa:** Dành thời gian thực sự hiểu Event Loop trước khi đi tiếp
3. **Đọc "You Don't Know JS":** Series sách miễn phí, nền tảng JavaScript vững chắc
4. **Build portfolio project:** REST API + Auth + Tests + Docker = project hoàn chỉnh
5. **Debug production issues:** Học cách đọc heap snapshot và flame graph
6. **Theo dõi Node.js releases:** LTS schedule, breaking changes, new features
7. **Contribute open source:** Đọc code của thư viện phổ biến (Express, Fastify, Prisma)
8. **Practice system design:** Node.js có constraints riêng (single-threaded, event-driven)

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống, hoan nghênh đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa có
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn cho concept phức tạp
- [ ] Cập nhật cho Node.js version mới

---

## 📄 Giấy Phép

Knowledge base này mở cho mục đích học tập và sử dụng chuyên nghiệp.

---

**Cập Nhật Lần Cuối:** 2026-06-24
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ README & INDEX hoàn thành | ✅ 01-fundamentals hoàn thành | ✅ 02-async-programming hoàn thành | ✅ 03-web-frameworks hoàn thành | ✅ 04-data-access hoàn thành | ✅ 05-security hoàn thành | ✅ 06-testing hoàn thành | ✅ 07-performance hoàn thành | ✅ 08-architecture hoàn thành | ✅ 09-cloud-deployment hoàn thành | ✅ 10-advanced hoàn thành | ✅ 11-interview-prep hoàn thành
