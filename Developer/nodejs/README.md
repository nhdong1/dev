# 🟢 Node.js & JavaScript — Lộ Trình Học Toàn Diện

> Hướng dẫn toàn diện về Node.js và JavaScript cho Backend Development (Phát Triển Phía Server), từ nền tảng ngôn ngữ đến kiến trúc microservices sản xuất thực tế — dành cho lập trình viên Backend muốn nắm vững hệ sinh thái JavaScript/TypeScript phía server.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Theo Công Nghệ & Framework](#theo-công-nghệ--framework)
5. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng JavaScript & Node.js (Tuần 1–2)**

- [ ] JavaScript ES6+ — Arrow Functions, Destructuring, Spread/Rest, Modules (ESM — ECMAScript Modules)
- [ ] Node.js Runtime — V8 Engine, libuv, Single-threaded Event Loop (Vòng Lặp Sự Kiện Đơn Luồng)
- [ ] NPM (Node Package Manager — Trình Quản Lý Gói Node) & Package Management — `package.json`, `package-lock.json`, SemVer
- [ ] CommonJS vs ESM — `require()` vs `import/export`
- [ ] TypeScript Basics — Static Typing (Kiểu Tĩnh), Interfaces, Generics

### **Giai Đoạn 2: Lập Trình Bất Đồng Bộ & API (Tuần 3–6)**

- [ ] Event Loop — Call Stack, Callback Queue, Microtask Queue
- [ ] Promises & async/await — Error Handling (Xử Lý Lỗi) với try/catch
- [ ] Express.js — Middleware (Phần Mềm Trung Gian), Routing, Request/Response
- [ ] REST API Design — HTTP Methods, Status Codes, Validation
- [ ] Error Handling — Centralized Error Handler, Custom Error Classes
- [ ] Environment Configuration — dotenv, `process.env`, Config per Environment

### **Giai Đoạn 3: Dữ Liệu, Bảo Mật & Kiểm Thử (Tuần 7–10)**

- [ ] Database Integration — PostgreSQL, MongoDB với ORM/ODM (Object-Document Mapper)
- [ ] Prisma / TypeORM / Mongoose — Schema, Migrations, Query Optimization
- [ ] Authentication — JWT (JSON Web Token), Passport.js, OAuth2
- [ ] Security Best Practices — Helmet, Rate Limiting, Input Sanitization, CORS
- [ ] Testing — Jest, Vitest, Supertest, Integration Tests
- [ ] Caching — Redis, In-Memory Cache, Cache Invalidation

### **Giai Đoạn 4: Hiệu Năng & Triển Khai (Tuần 11–14)**

- [ ] Performance Tuning — Profiling, Memory Leaks, Event Loop Lag
- [ ] Cluster Module & Worker Threads — Horizontal Scaling (Mở Rộng Theo Chiều Ngang)
- [ ] Message Queues — BullMQ, RabbitMQ, Kafka với Node.js
- [ ] Docker & Kubernetes — Containerization (Đóng Gói Container), Health Checks
- [ ] Observability — Logging (Winston/Pino), Metrics (Prometheus), Tracing (OpenTelemetry)
- [ ] CI/CD Pipeline — GitHub Actions, Automated Testing & Deployment

### **Giai Đoạn 5: Chuyên Sâu (Tuần 15+)**

- [ ] NestJS — Dependency Injection (Tiêm Phụ Thuộc), Modules, Guards, Interceptors
- [ ] Microservices — gRPC, Message Broker, Service Discovery
- [ ] GraphQL — Apollo Server, Schema Design, DataLoader
- [ ] WebSockets & Real-time — Socket.io, Server-Sent Events (SSE)
- [ ] Streams & Buffers — Readable/Writable Streams, Backpressure (Áp Lực Ngược)
- [ ] System Design với Node.js — Scalability Patterns, Trade-offs

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                   | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------------ | ---------- | --------- | ---------- |
| **JavaScript ES6+ & TypeScript**           | ⭐⭐⭐      | 2 tuần    | -          |
| **Event Loop & Async Programming**         | ⭐⭐⭐      | 2 tuần    | -          |
| **Express.js / Fastify REST API**          | ⭐⭐⭐      | 2 tuần    | -          |
| **Database Integration (SQL & NoSQL)**     | ⭐⭐⭐      | 2 tuần    | -          |
| **Authentication & Security**              | ⭐⭐⭐      | 2 tuần    | -          |
| **Testing (Unit & Integration)**           | ⭐⭐⭐      | 1.5 tuần  | -          |
| **Performance & Memory Management**        | ⭐⭐⭐      | 1.5 tuần  | -          |
| **Docker & Production Deployment**         | ⭐⭐       | 2 tuần    | -          |
| **NestJS Framework**                       | ⭐⭐       | 2 tuần    | -          |
| **Message Queues & Event-Driven**          | ⭐⭐       | 2 tuần    | -          |
| **Microservices & gRPC**                   | ⭐         | 2 tuần    | -          |
| **GraphQL & Real-time**                    | ⭐         | 1.5 tuần  | -          |

---

## 🗂️ Tổng Quan Chủ Đề

### 📁 **1. Nền Tảng JavaScript & Node.js** (`01-fundamentals/`)

- **JavaScript ES6+** — let/const, Arrow Functions, Template Literals, Destructuring
- **Node.js Runtime** — V8 Engine, libuv, Process Model (Mô Hình Tiến Trình)
- **Module Systems** — CommonJS (`require`/`module.exports`) vs ESM (`import`/`export`)
- **NPM Ecosystem** — `package.json`, Scripts, `npx`, Workspaces (Monorepo)
- **TypeScript** — Type System, Interfaces, Enums, Utility Types
- **Node.js Built-in Modules** — `fs`, `path`, `os`, `crypto`, `http`, `events`

### 📁 **2. Lập Trình Bất Đồng Bộ** (`02-async-programming/`)

- **Event Loop** — Phases (Timers, I/O Callbacks, Idle, Poll, Check, Close)
- **Callbacks** — Callback Hell, Error-First Callback Pattern
- **Promises** — `Promise.all`, `Promise.race`, `Promise.allSettled`, Chaining
- **async/await** — Syntactic Sugar, Parallel vs Sequential Execution
- **Error Handling** — Unhandled Rejection, `process.on('uncaughtException')`
- **Concurrency Patterns** — Throttling, Debouncing, Queue-based Processing

### 📁 **3. Web Frameworks & REST API** (`03-web-frameworks/`)

- **Express.js** — Middleware Chain, Router, `express.json()`, Static Files
- **Fastify** — High-performance Framework, Schema Validation, Plugin System
- **NestJS** — Module-based Architecture, Decorators, DI Container
- **REST API Design** — Resource Naming, HTTP Verbs, HATEOAS (Hypermedia)
- **Request Validation** — Zod, Joi, class-validator
- **API Documentation** — Swagger/OpenAPI, Scalar, Redoc
- **Middleware Patterns** — Auth Middleware, Logging, Request ID, Timeout

### 📁 **4. Truy Cập Dữ Liệu** (`04-data-access/`)

- **PostgreSQL với Node.js** — `pg` driver, Connection Pool (Bể Kết Nối)
- **Prisma ORM** — Schema Definition, Migrations, Prisma Client, Raw Queries
- **TypeORM** — Entity, Repository Pattern, Relations, Query Builder
- **Sequelize** — Model Definition, Associations, Hooks
- **MongoDB & Mongoose** — Document Schema, Aggregation Pipeline, Indexing
- **Redis** — `ioredis`, Caching Patterns, Pub/Sub, Session Store
- **Database Transactions** — ACID trong Node.js, Optimistic Locking
- **N+1 Problem** — Eager Loading, DataLoader Pattern

### 📁 **5. Bảo Mật** (`05-security/`)

- **Authentication** — JWT Access/Refresh Token, Session-based Auth
- **Passport.js** — Strategies (Local, JWT, OAuth2, Google, GitHub)
- **Authorization** — RBAC (Role-Based Access Control — Phân Quyền Theo Vai Trò)
- **OWASP Top 10** — SQL Injection, XSS (Cross-Site Scripting), CSRF Protection
- **Helmet.js** — Security Headers (HTTP Headers Bảo Mật)
- **Rate Limiting** — `express-rate-limit`, Token Bucket Algorithm
- **Input Validation & Sanitization** — Whitelist Approach, SQL Parameterization
- **Secrets Management** — Environment Variables, Vault, AWS Secrets Manager
- **HTTPS & TLS** — Certificate Management, HSTS (HTTP Strict Transport Security)

### 📁 **6. Kiểm Thử** (`06-testing/`)

- **Jest** — Unit Testing, Mocking, Snapshot Testing, Coverage Reports
- **Vitest** — Fast Test Runner cho Vite/ESM Projects
- **Supertest** — HTTP Assertion Testing cho API Endpoints
- **Testcontainers** — Integration Tests với Real Databases trong Docker
- **Mocking Strategies** — `jest.mock()`, Dependency Injection, MSW (Mock Service Worker)
- **TDD (Test-Driven Development — Phát Triển Hướng Kiểm Thử)** — Red-Green-Refactor Cycle
- **E2E Testing** — Playwright, API-level E2E với Newman/Postman

### 📁 **7. Hiệu Năng** (`07-performance/`)

- **Event Loop Monitoring** — `clinic.js`, `0x` Flame Graphs
- **Memory Management** — Heap Snapshots, Garbage Collection (GC — Thu Gom Rác), Memory Leaks
- **Cluster Module** — Multi-process Scaling, Sticky Sessions
- **Worker Threads** — CPU-intensive Tasks Offloading
- **Caching Strategies** — Cache-Aside, Write-Through, TTL (Time To Live)
- **Connection Pooling** — Database Pool Sizing, Pool Exhaustion
- **Load Testing** — k6, Artillery, Autocannon — Throughput & Latency Benchmarks
- **Profiling** — Node.js `--inspect`, Chrome DevTools, `perf_hooks`

### 📁 **8. Kiến Trúc** (`08-architecture/`)

- **Clean Architecture** — Layers, Dependency Rule, Use Cases
- **Hexagonal Architecture** — Ports & Adapters Pattern
- **Design Patterns** — Singleton, Factory, Observer, Strategy, Repository
- **Microservices** — Service Decomposition, API Gateway, Service Mesh
- **Event-Driven Architecture** — Event Sourcing, CQRS (Command Query Responsibility Segregation)
- **Monorepo** — Turborepo, Nx, pnpm Workspaces
- **API Versioning** — URL Versioning, Header Versioning, Deprecation Strategy
- **Error Handling Architecture** — Result Pattern, Either Monad, Global Error Boundary

### 📁 **9. Triển Khai & Observability** (`09-cloud-deployment/`)

- **Docker** — Multi-stage Dockerfile, `.dockerignore`, Layer Caching
- **PM2** — Process Manager, Cluster Mode, Zero-downtime Reload
- **Kubernetes (K8s)** — Deployment, Service, ConfigMap, Secret, HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang)
- **Health Checks** — Liveness, Readiness, Startup Probes
- **Logging** — Structured Logging với Pino/Winston, Log Aggregation (ELK, Loki)
- **Metrics** — Prometheus, Grafana Dashboards, Custom Metrics
- **Distributed Tracing** — OpenTelemetry, Jaeger, Zipkin
- **CI/CD** — GitHub Actions, GitLab CI, Automated Rollback

### 📁 **10. Chủ Đề Nâng Cao** (`10-advanced/`)

- **Streams API** — Readable, Writable, Transform, Duplex, Pipeline
- **Child Processes** — `spawn`, `exec`, `fork`, IPC (Inter-Process Communication)
- **GraphQL** — Apollo Server, Resolvers, Subscriptions, N+1 với DataLoader
- **WebSockets** — Socket.io, ws library, Real-time Broadcasting
- **gRPC** — Protocol Buffers, Streaming RPC, `@grpc/grpc-js`
- **Serverless** — AWS Lambda, Vercel Functions, Cold Start Optimization
- **BullMQ / Agenda** — Job Queues, Scheduled Tasks, Retry Policies
- **Native Addons** — N-API, `node-gyp`, FFI (Foreign Function Interface)

### 📁 **11. Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 50 câu hỏi phỏng vấn Node.js & JavaScript
- Event Loop Deep Dive Questions
- System Design Scenarios (Tình Huống Thiết Kế Hệ Thống)
- Coding Challenges — Async Patterns, API Design
- STAR Stories (Situation, Task, Action, Result) cho tình huống thực tế
- Kế hoạch học 90 ngày

---

## 🎓 Theo Công Nghệ & Framework

### **Express.js**

```
Điểm mạnh: Đơn giản, hệ sinh thái lớn, middleware phong phú, dễ học
Phù hợp: REST API, MVP, startup, prototype nhanh
Học trong: 03-web-frameworks/express
```

### **Fastify**

```
Điểm mạnh: Hiệu năng cao (2-3x Express), Schema validation tích hợp, TypeScript-first
Phù hợp: High-throughput API, microservices, yêu cầu latency thấp
Học trong: 03-web-frameworks/fastify
```

### **NestJS**

```
Điểm mạnh: Kiến trúc có cấu trúc, DI, decorators, tương tự Spring Boot
Phù hợp: Enterprise applications, team lớn, dự án dài hạn
Học trong: 03-web-frameworks/nestjs
```

### **Prisma**

```
Điểm mạnh: Type-safe ORM, migrations tự động, Prisma Studio, DX tốt
Phù hợp: PostgreSQL/MySQL projects, TypeScript-first teams
Học trong: 04-data-access/prisma
```

### **MongoDB + Mongoose**

```
Điểm mạnh: Schema linh hoạt, horizontal scaling, aggregation mạnh
Phù hợp: Document-heavy apps, rapid prototyping, flexible data models
Học trong: 04-data-access/mongodb
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                    | Thư Mục                                                          | Độ Ưu Tiên        |
| ------------------------- | ---------------------------------------------------------------- | ----------------- |
| Bắt đầu tại đây           | [README.md](./README.md)                                         | Đọc trước tiên    |
| Event Loop giải thích     | [02-async-programming/event-loop.md](./02-async-programming/event-loop.md) | Bắt buộc          |
| REST API với Express      | [03-web-frameworks/express.md](./03-web-frameworks/express.md) | Cốt lõi           |
| Bảo mật API               | [05-security/README.md](./05-security/README.md)                 | Bắt buộc          |
| Câu hỏi phỏng vấn         | [11-interview-prep/INTERVIEW_GUIDE.md](./11-interview-prep/INTERVIEW_GUIDE.md) | Trước phỏng vấn   |
| Production Checklist    | [09-cloud-deployment/production-checklist.md](./09-cloud-deployment/production-checklist.md) | Trước go-live     |

---

## 📊 Ma Trận Kỹ Năng

### Beginner (0–1 năm kinh nghiệm)

- [ ] Hiểu JavaScript ES6+ cơ bản (arrow functions, destructuring, modules)
- [ ] Giải thích được Event Loop ở mức cơ bản
- [ ] Tạo REST API đơn giản với Express
- [ ] Kết nối database (PostgreSQL hoặc MongoDB)
- [ ] Viết unit test cơ bản với Jest
- [ ] Sử dụng `async/await` và xử lý lỗi đúng cách

### Intermediate (1–3 năm kinh nghiệm)

- [ ] Thiết kế REST API chuẩn với validation, error handling
- [ ] Implement JWT authentication & authorization
- [ ] Tối ưu database queries, hiểu N+1 problem
- [ ] Viết integration tests với Supertest
- [ ] Caching với Redis, hiểu cache invalidation
- [ ] Dockerize ứng dụng Node.js
- [ ] Debug memory leaks và performance issues

### Advanced (3–5+ năm kinh nghiệm)

- [ ] Thiết kế microservices architecture với Node.js
- [ ] Implement event-driven systems (Kafka, RabbitMQ)
- [ ] Tuning production performance (cluster, worker threads)
- [ ] Observability stack hoàn chỉnh (logs, metrics, traces)
- [ ] System design cho high-traffic Node.js applications
- [ ] Security hardening và compliance (OWASP, GDPR)
- [ ] Mentoring và code review best practices

---

## 🚀 Bắt Đầu

### Bước 1: Chọn Lộ Trình Học

```
Chọn con đường phù hợp:
- Backend Generalist — Express + PostgreSQL + Redis (phổ biến nhất)
- TypeScript-first — NestJS + Prisma + TypeScript (enterprise)
- Full-stack JavaScript — Node.js + MongoDB + GraphQL
- Performance-focused — Fastify + Streams + Worker Threads
```

### Bước 2: Thiết Lập Môi Trường Lab

```bash
# Cài đặt Node.js LTS (khuyến nghị dùng nvm)
nvm install --lts
nvm use --lts

# Khởi tạo project thực hành
mkdir nodejs-lab && cd nodejs-lab
npm init -y
npm install express typescript @types/node ts-node

# Docker compose cho database
docker compose up -d  # PostgreSQL, Redis, MongoDB
```

### Bước 3: Học + Thực Hành

```
1. Đọc một module (30 phút)
2. Viết code thực hành (45–60 phút)
3. Viết test cho code vừa viết (20 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Cho mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation — Tình huống
- Task — Nhiệm vụ
- Action — Hành động đã thực hiện
- Result — Kết quả đạt được
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Thiết Yếu

- **"Node.js Design Patterns"** by Mario Casciaro — Design patterns cho Node.js
- **"You Don't Know JS"** by Kyle Simpson — JavaScript sâu (bắt buộc đọc)
- **"Eloquent JavaScript"** by Marijn Haverbeke — Nền tảng JavaScript miễn phí
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — System design

### Tài Liệu Chính Thức

- [Node.js Documentation](https://nodejs.org/docs/)
- [MDN JavaScript Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [Express.js Guide](https://expressjs.com/en/guide/routing.html)
- [NestJS Documentation](https://docs.nestjs.com/)
- [Prisma Documentation](https://www.prisma.io/docs)

### Blog & Newsletter

- Node.js Weekly Newsletter
- JavaScript Weekly
- RisingStack Blog (Node.js performance)
- Goldbergyoni — Node.js Best Practices (GitHub)

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### Event Loop & Async

- [ ] Giải thích Event Loop hoạt động như thế nào?
- [ ] Sự khác biệt giữa `setImmediate()` và `process.nextTick()`?
- [ ] `Promise.all` vs `Promise.allSettled` — khi nào dùng gì?
- [ ] Làm sao xử lý unhandled promise rejection?

#### API & Framework

- [ ] Middleware trong Express hoạt động thế nào?
- [ ] Thiết kế REST API cho hệ thống e-commerce
- [ ] Error handling strategy cho production API
- [ ] So sánh Express vs Fastify vs NestJS

#### Database & Performance

- [ ] Giải quyết N+1 query problem trong Node.js
- [ ] Connection pool sizing — bao nhiêu connections là đủ?
- [ ] Caching strategy cho read-heavy application
- [ ] Debug memory leak trong Node.js production

#### Security

- [ ] JWT vs Session-based authentication — trade-offs?
- [ ] Cách prevent SQL Injection trong Node.js
- [ ] Rate limiting implementation và algorithm
- [ ] OWASP Top 10 — áp dụng thế nào với Node.js?

#### Tình Huống Thực Tế

- [ ] Kể về incident production với Node.js (STAR format)
- [ ] Root cause analysis cho API chậm
- [ ] Migration từ monolith sang microservices

Xem `11-interview-prep/` để có bộ câu hỏi đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc khi nhận vai trò mới, xác nhận:

- [ ] Giải thích được Event Loop mà không cần nhìn tài liệu
- [ ] Viết REST API hoàn chỉnh với auth, validation, error handling
- [ ] Debug được memory leak bằng heap snapshot
- [ ] Thiết kế caching strategy phù hợp với use case
- [ ] Viết integration test cho API endpoints
- [ ] Dockerize và deploy Node.js app lên production
- [ ] Implement JWT authentication flow đầy đủ
- [ ] Giải thích trade-offs của microservices vs monolith
- [ ] Đọc và hiểu flame graph từ profiling tool
- [ ] Thiết kế hệ thống xử lý 10,000 requests/giây

---

## 📞 Hỗ Trợ & Cộng Đồng

### Học Tập

- [Node.js Official Docs](https://nodejs.org/docs/)
- [JavaScript.info](https://javascript.info/) — Tutorial JavaScript hiện đại
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices) — GitHub repo

### Công Cụ

- **VS Code** — IDE phổ biến nhất cho Node.js
- **Postman / Insomnia** — API Testing
- **DBeaver** — Universal Database Client
- **clinic.js** — Node.js Performance Profiling Suite
- **PM2** — Production Process Manager

### Cộng Đồng

- r/node (Reddit)
- Node.js Discord Official
- Stack Overflow — tag `node.js`
- Dev.to — Node.js tag

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn tuần tự
3. Làm bài tập thực hành cho mỗi chủ đề
4. Xây dựng portfolio project (ví dụ: REST API + Auth + Tests + Docker)

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep](./11-interview-prep/)
2. Ôn sâu Event Loop và async patterns (luôn được hỏi)
3. Chuẩn bị 2–3 câu chuyện STAR từ kinh nghiệm thực tế
4. Practice system design với Node.js constraints

### Học Trên Công Việc

1. Tham khảo [04-data-access](./04-data-access/) khi tích hợp database
2. Dùng [05-security](./05-security/) khi implement auth
3. Kiểm tra [09-cloud-deployment](./09-cloud-deployment/) trước khi deploy
4. Dùng [07-performance](./07-performance/) khi troubleshoot production issues

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner/Intermediate/Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Thiết lập môi trường lab (Node.js + Docker)
├─ 5️⃣  Hoàn thành bài tập cho mỗi chủ đề
├─ 6️⃣  Xây dựng portfolio project
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-23
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
