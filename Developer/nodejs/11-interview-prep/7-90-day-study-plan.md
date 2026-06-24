# Kế Hoạch Học 90 Ngày — Node.js Backend Developer

> Lộ trình học có cấu trúc 90 ngày từ Beginner đến sẵn sàng phỏng vấn Backend Node.js, kết hợp theory (lý thuyết) và practice (thực hành) project.

## Mục Lục

1. [Tổng Quan 90 Ngày](#tổng-quan-90-ngày)
2. [Giai Đoạn 1: Nền Tảng (Ngày 1–14)](#giai-đoạn-1-nền-tảng-ngày-114)
3. [Giai Đoạn 2: Async & API (Ngày 15–35)](#giai-đoạn-2-async--api-ngày-1535)
4. [Giai Đoạn 3: Data & Security (Ngày 36–56)](#giai-đoạn-3-data--security-ngày-3656)
5. [Giai Đoạn 4: Production Skills (Ngày 57–77)](#giai-đoạn-4-production-skills-ngày-5777)
6. [Giai Đoạn 5: Advanced & Interview (Ngày 78–90)](#giai-đoạn-5-advanced--interview-ngày-7890)
7. [Portfolio Project](#portfolio-project)
8. [Theo Dõi Tiến Độ](#theo-dõi-tiến-độ)

---

## Tổng Quan 90 Ngày

```
Tuần 1–2   ████████░░░░░░░░░░░░  Nền tảng JS & Node.js
Tuần 3–5  ░░░░████████░░░░░░░░  Async & REST API
Tuần 6–8  ░░░░░░░░████████░░░░  Database & Security
Tuần 9–11 ░░░░░░░░░░░░████████  Production & DevOps
Tuần 12–13░░░░░░░░░░░░░░░░████  Advanced & Interview Prep
```

| Giai Đoạn | Ngày | Giờ/Tuần | Mục Tiêu |
| --------- | ---- | -------- | -------- |
| 1. Nền Tảng | 1–14 | 10–12h | JS ES6+, Node.js runtime, TypeScript |
| 2. Async & API | 15–35 | 12–15h | Event Loop, Express, REST API project |
| 3. Data & Security | 36–56 | 12–15h | PostgreSQL, Prisma, JWT, testing |
| 4. Production | 57–77 | 10–12h | Performance, Docker, CI/CD, monitoring |
| 5. Interview | 78–90 | 15–20h | System design, mock interviews |

**Tổng ước tính:** 100–130 giờ (~1.5h/ngày trung bình)

---

## Giai Đoạn 1: Nền Tảng (Ngày 1–14)

### Tuần 1: JavaScript ES6+ & Node.js Runtime

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 1 | Setup môi trường: nvm, VS Code, Docker | README.md | `node -v`, `docker compose up` |
| 2 | let/const, arrow functions, template literals | 01-fundamentals/1-javascript-es6-plus.md | 10 bài tập destructuring |
| 3 | Spread/rest, modules (ESM) | 01-fundamentals/1-javascript-es6-plus.md | Refactor code sang modules |
| 4 | Node.js runtime: V8, libuv, process | 01-fundamentals/2-nodejs-runtime.md | Script đọc `process.env`, `os.cpus()` |
| 5 | CommonJS vs ESM | 01-fundamentals/3-module-systems.md | Project dùng ESM |
| 6 | NPM, package.json, SemVer | 01-fundamentals/4-npm-ecosystem.md | Publish package local |
| 7 | **Review tuần 1** | — | Quiz tự kiểm tra 20 câu |

### Tuần 2: TypeScript & Built-in Modules

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 8 | TypeScript basics: types, interfaces | 01-fundamentals/5-typescript-basics.md | Convert JS project sang TS |
| 9 | Generics, utility types | 01-fundamentals/5-typescript-basics.md | Typed API response wrapper |
| 10 | fs, path, crypto modules | 01-fundamentals/6-builtin-modules.md | CLI tool đọc/ghi file |
| 11 | http, events, buffer modules | 01-fundamentals/6-builtin-modules.md | HTTP server không dùng Express |
| 12 | Events & EventEmitter pattern | 01-fundamentals/6-builtin-modules.md | Custom event-driven module |
| 13 | Ôn tập + mini project | — | File organizer CLI tool |
| 14 | **Checkpoint 1** | INTERVIEW_GUIDE Câu 11–15 | Giải thích V8, ESM vs CJS |

**Milestone Giai Đoạn 1:** Hiểu Node.js runtime, viết TypeScript cơ bản, dùng built-in modules.

---

## Giai Đoạn 2: Async & API (Ngày 15–35)

### Tuần 3: Event Loop & Async Patterns

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 15 | Event Loop phases, call stack | 02-async-programming/1-event-loop.md | 5 bài tập output ordering |
| 16 | Callbacks, callback hell | 02-async-programming/2-callbacks.md | Refactor callback → named functions |
| 17 | Promises API, chaining | 02-async-programming/3-promises.md | Promise.all/race exercises |
| 18 | async/await, parallel vs sequential | 02-async-programming/4-async-await.md | Fetch multiple APIs |
| 19 | Error handling async | 02-async-programming/5-error-handling.md | Global rejection handlers |
| 20 | Concurrency: throttle, debounce, queue | 02-async-programming/6-concurrency-patterns.md | Rate-limited API client |
| 21 | **Review Event Loop** | 1-event-loop-questions.md | Trả lời 15 câu hỏi |

### Tuần 4–5: Express & REST API

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 22 | Express basics: middleware, routing | 03-web-frameworks/1-express-basics.md | Hello World API |
| 23 | Express advanced: error handling | 03-web-frameworks/2-express-advanced.md | Custom error classes |
| 24 | REST API design principles | 03-web-frameworks/5-rest-api-design.md | Design API cho blog |
| 25 | Request validation (Zod) | 03-web-frameworks/6-request-validation.md | Validate POST /users |
| 26 | API documentation (Swagger) | 03-web-frameworks/7-api-documentation.md | OpenAPI spec cho blog API |
| 27–28 | **Project: Blog REST API** | — | CRUD posts, comments, pagination |
| 29 | Fastify overview | 03-web-frameworks/3-fastify.md | Port 1 endpoint sang Fastify |
| 30 | NestJS overview | 03-web-frameworks/4-nestjs.md | Đọc architecture, so sánh |
| 31–35 | **Hoàn thiện Blog API** | 2-api-design-questions.md | Auth middleware, error handling, tests |

**Milestone Giai Đoạn 2:** Giải thích Event Loop, REST API production-ready với validation và docs.

---

## Giai Đoạn 3: Data & Security (Ngày 36–56)

### Tuần 6: Database Integration

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 36 | PostgreSQL + pg driver | 04-data-access/1-postgresql-pg.md | Raw SQL queries |
| 37 | Prisma ORM: schema, migrations | 04-data-access/2-prisma-orm.md | Migrate Blog API sang Prisma |
| 38 | Relations, eager loading | 04-data-access/2-prisma-orm.md | Include author trong posts |
| 39 | N+1 problem, DataLoader | 04-data-access/8-n-plus-one.md | Fix N+1 trong Blog API |
| 40 | Transactions, ACID | 04-data-access/7-transactions.md | Order placement transaction |
| 41 | Redis caching | 04-data-access/6-redis-caching.md | Cache-aside cho GET /posts |
| 42 | **Review Database** | 3-database-questions.md | 18 câu hỏi |

### Tuần 7: Security & Authentication

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 43 | JWT authentication flow | 05-security/1-authentication-jwt.md | Login/register endpoints |
| 44 | Passport.js strategies | 05-security/2-passport-strategies.md | JWT strategy middleware |
| 45 | RBAC authorization | 05-security/3-authorization-rbac.md | Admin vs user routes |
| 46 | OWASP Top 10 | 05-security/4-owasp-top10.md | Audit Blog API security |
| 47 | Helmet, rate limiting | 05-security/5-helmet-rate-limiting.md | Add security middleware |
| 48 | Input validation & secrets | 05-security/6,7 | Env validation at startup |
| 49 | **Review Security** | 4-security-questions.md | 18 câu hỏi |

### Tuần 8: Testing

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 50 | Jest unit testing | 06-testing/1-jest-basics.md | Test service layer |
| 51 | Supertest integration tests | 06-testing/3-supertest.md | Test API endpoints |
| 52 | Mocking strategies | 06-testing/5-mocking-strategies.md | Mock Prisma client |
| 53 | Testcontainers | 06-testing/4-testcontainers.md | Integration test với real DB |
| 54–56 | **Hoàn thiện Blog API** | — | Auth + tests + Redis cache |

**Milestone Giai Đoạn 3:** Blog API với auth, RBAC, caching, test coverage >70%.

---

## Giai Đoạn 4: Production Skills (Ngày 57–77)

### Tuần 9: Performance

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 57 | Event Loop monitoring | 07-performance/1-event-loop-monitoring.md | Measure event loop lag |
| 58 | Memory management, GC | 07-performance/2-memory-management.md | Heap snapshot analysis |
| 59 | Cluster & Worker Threads | 07-performance/3-cluster-worker-threads.md | Cluster mode với PM2 |
| 60 | Caching strategies | 07-performance/4-caching-strategies.md | Optimize cache TTL |
| 61 | Connection pooling | 07-performance/5-connection-pooling.md | Tune pool size |
| 62 | Load testing (k6) | 07-performance/6-load-testing.md | Benchmark Blog API |
| 63 | Profiling | 07-performance/7-profiling.md | Flame graph analysis |

### Tuần 10: Architecture

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 64 | Clean Architecture | 08-architecture/1-clean-architecture.md | Refactor Blog API layers |
| 65 | Design Patterns | 08-architecture/3-design-patterns.md | Repository pattern |
| 66 | Error handling architecture | 08-architecture/7-error-handling-architecture.md | Result pattern |
| 67 | Microservices overview | 08-architecture/4-microservices.md | Identify bounded contexts |
| 68 | Event-driven patterns | 08-architecture/5-event-driven.md | Đọc CQRS, saga pattern |

### Tuần 11: Deployment & Observability

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 69 | Docker multi-stage build | 09-cloud-deployment/1-docker-nodejs.md | Dockerfile cho Blog API |
| 70 | PM2 process manager | 09-cloud-deployment/2-pm2-process-manager.md | PM2 cluster mode |
| 71 | Kubernetes basics | 09-cloud-deployment/3-kubernetes-deployment.md | Deploy manifest |
| 72 | Structured logging (Pino) | 09-cloud-deployment/4-logging.md | JSON logging + request ID |
| 73 | Metrics & tracing | 09-cloud-deployment/5-metrics-tracing.md | Prometheus metrics |
| 74 | CI/CD pipeline | 09-cloud-deployment/6-cicd-pipeline.md | GitHub Actions workflow |
| 75–77 | **Production hardening** | 09-cloud-deployment/7-production-checklist.md | Full production deploy |

**Milestone Giai Đoạn 4:** Blog API Dockerized, CI/CD, monitoring, load tested.

---

## Giai Đoạn 5: Advanced & Interview (Ngày 78–90)

### Tuần 12: Advanced Topics

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 78 | Streams API | 10-advanced/1-streams-api.md | File upload stream |
| 79 | WebSockets | 10-advanced/4-websockets.md | Real-time notifications |
| 80 | Job Queues (BullMQ) | 10-advanced/7-job-queues.md | Email queue |
| 81 | GraphQL overview | 10-advanced/3-graphql.md | Đọc schema design |
| 82 | gRPC overview | 10-advanced/5-grpc.md | Đọc proto files |

### Tuần 13: Interview Preparation

| Ngày | Nội Dung | Tài Liệu | Thực Hành |
| ---- | -------- | -------- | --------- |
| 83 | INTERVIEW_GUIDE Câu 1–25 | INTERVIEW_GUIDE.md | Viết đáp án bằng lời của bạn |
| 84 | INTERVIEW_GUIDE Câu 26–50 | INTERVIEW_GUIDE.md | Practice trả lời aloud |
| 85 | System design scenarios | 5-system-design-scenarios.md | Thiết kế URL shortener (45 phút) |
| 86 | System design scenarios | 5-system-design-scenarios.md | Thiết kế chat app (45 phút) |
| 87 | STAR stories | 6-star-stories.md | Viết 5 câu chuyện cá nhân |
| 88 | Mock interview #1 | README.md | Technical round với bạn |
| 89 | Mock interview #2 | — | System design round |
| 90 | **Final review** | Toàn bộ 11-interview-prep/ | Checklist trước phỏng vấn |

**Milestone Giai Đoạn 5:** Sẵn sàng phỏng vấn — 50 câu hỏi, 3 system designs, 5 STAR stories.

---

## Portfolio Project

Xuyên suốt 90 ngày, xây dựng **Blog Platform API** (hoặc E-commerce API):

### Features Theo Giai Đoạn

```
Giai Đoạn 2: CRUD posts, comments, pagination, validation
Giai Đoạn 3: PostgreSQL + Prisma, JWT auth, RBAC, Redis cache, tests
Giai Đoạn 4: Docker, CI/CD, logging, metrics, load testing
Giai Đoạn 5: WebSocket notifications, job queue cho emails
```

### Tech Stack Đề Xuất

| Layer | Technology |
| ----- | ---------- |
| Runtime | Node.js 20 LTS |
| Language | TypeScript |
| Framework | Express.js |
| ORM | Prisma |
| Database | PostgreSQL |
| Cache | Redis |
| Auth | JWT (access + refresh) |
| Testing | Jest + Supertest |
| Container | Docker + Docker Compose |
| CI/CD | GitHub Actions |

### README Portfolio Nên Có

- Architecture diagram
- API documentation link (Swagger)
- Setup instructions (`docker compose up`)
- Test coverage badge
- Performance benchmarks (k6 results)

---

## Theo Dõi Tiến Độ

Copy template này và check hàng tuần:

```markdown
## 90-Day Progress Tracker

### Giai Đoạn 1: Nền Tảng (Ngày 1–14)
- [ ] Ngày 1–7: JavaScript ES6+ & Node.js runtime
- [ ] Ngày 8–14: TypeScript & built-in modules
- [ ] Checkpoint 1 passed

### Giai Đoạn 2: Async & API (Ngày 15–35)
- [ ] Ngày 15–21: Event Loop & async patterns
- [ ] Ngày 22–35: Express & REST API project
- [ ] Checkpoint 2 passed

### Giai Đoạn 3: Data & Security (Ngày 36–56)
- [ ] Ngày 36–42: Database & caching
- [ ] Ngày 43–49: Security & authentication
- [ ] Ngày 50–56: Testing
- [ ] Checkpoint 3 passed

### Giai Đoạn 4: Production (Ngày 57–77)
- [ ] Ngày 57–63: Performance
- [ ] Ngày 64–68: Architecture
- [ ] Ngày 69–77: Deployment & observability
- [ ] Checkpoint 4 passed

### Giai Đoạn 5: Interview (Ngày 78–90)
- [ ] Ngày 78–82: Advanced topics
- [ ] Ngày 83–90: Interview preparation
- [ ] 5 STAR stories written
- [ ] 2 mock interviews completed
- [ ] READY FOR INTERVIEW ✅
```

---

## Lịch Học Đề Xuất

### Full-time Learner (3–4h/ngày)

- Sáng (1.5h): Đọc theory + notes
- Chiều (1.5h): Coding practice
- Tối (30min): Review + flashcards

### Part-time Learner (1–1.5h/ngày)

- Kéo dài thành 120–150 ngày
- Tập trung weekend cho project work
- Ưu tiên: Event Loop → Express → Prisma → JWT → Testing

### Working Professional (30min/ngày + weekend)

- Weekday: Đọc 1 section + 1 bài tập nhỏ
- Weekend: Project work + deep dive
- Kéo dài thành 120–180 ngày

---

## Tài Liệu Tham Khảo

- [README.md](./README.md) — Tổng quan chuẩn bị phỏng vấn
- [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) — Top 50 câu hỏi
- [INDEX.md](../INDEX.md) — Chỉ mục toàn bộ knowledge base

---

**Cập Nhật:** 2026-06-24 | **Trạng Thái:** ✅ Hoàn thành
