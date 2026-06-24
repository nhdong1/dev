# Kiến Trúc Ứng Dụng Node.js — Tổng Quan

> Hướng dẫn thiết kế kiến trúc (architecture) cho Backend Node.js — từ layered monolith (monolith phân lớp) đến microservices (kiến trúc vi dịch vụ), event-driven systems (hệ thống hướng sự kiện), và patterns giúp codebase dễ bảo trì, test, và scale.

## Mục Lục

1. [Tại Sao Kiến Trúc Quan Trọng](#tại-sao-kiến-trúc-quan-trọng)
2. [Phổ Kiến Trúc Node.js Backend](#phổ-kiến-trúc-nodejs-backend)
3. [Workflow Chọn Kiến Trúc](#workflow-chọn-kiến-trúc)
4. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
5. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
6. [Bài Tập Thực Hành](#bài-tập-thực-hành)
7. [Architecture Checklist](#architecture-checklist)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Kiến Trúc Quan Trọng

Code chạy được ≠ code sống được lâu. Kiến trúc kém thường biểu hiện qua:

| Triệu Chứng | Nguyên Nhân Kiến Trúc |
| ----------- | --------------------- |
| Sửa 1 bug phá 3 feature khác | Business logic dính chặt framework/DB |
| Không test được use case | Không tách layer, mock khó |
| Migration DB mất hàng tuần | ORM queries rải khắp controllers |
| Team conflict liên tục | Monolith không có module boundaries |
| Scale một phần phải scale cả app | Không decompose theo bounded context |

**Nguyên tắc cốt lõi:** Kiến trúc phục vụ **business requirements (yêu cầu nghiệp vụ)** và **team structure (cấu trúc team)** — không phải ngược lại. Bắt đầu đơn giản, evolve khi có bằng chứng cần thiết.

---

## Phổ Kiến Trúc Node.js Backend

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SPECTRUM                          │
│                                                                 │
│  Simple ◄──────────────────────────────────────────► Complex   │
│                                                                 │
│  MVC/Layered    Clean/Hexagonal    Modular Monolith    Microservices
│  (Express)      (Use Cases)        (Monorepo)          (Distributed)
│                                                                 │
│  1 team         2-5 devs           5-20 devs           20+ devs  │
│  MVP/startup    Growing product    Scale-up            Enterprise│
└─────────────────────────────────────────────────────────────────┘
```

| Kiến Trúc | Phù Hợp Khi | Trade-off Chính |
| --------- | ----------- | --------------- |
| **Layered MVC** | MVP, prototype, team nhỏ | Dễ thành "big ball of mud" |
| **Clean Architecture** | Dự án dài hạn, cần testability | Boilerplate nhiều hơn |
| **Hexagonal** | Nhiều integration (DB, API, queue) | Learning curve cao |
| **Modular Monolith** | Team đang lớn, chưa cần distributed | Cần discipline về boundaries |
| **Microservices** | Scale độc lập, team autonomously | Operational complexity |

---

## Workflow Chọn Kiến Trúc

```
1. HIỂU DOMAIN           → Bounded contexts, core vs supporting
        ↓
2. ĐÁNH GIÁ TEAM         → Conway's Law — team structure → system design
        ↓
3. CHỌN STARTING POINT   → Layered hoặc Clean cho hầu hết Node.js apps
        ↓
4. ENFORCE BOUNDARIES    → Modules, interfaces, dependency rule
        ↓
5. MEASURE PAIN POINTS   → Deploy coupling? Scale bottleneck? Team friction?
        ↓
6. EVOLVE                → Extract service khi có lý do rõ ràng
```

**Conway's Law (Định Luật Conway):** Hệ thống phần mềm phản ánh cấu trúc giao tiếp của tổ chức tạo ra nó. Thiết kế kiến trúc phải align với cách team collaborate.

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 8–10 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-clean-architecture.md](./1-clean-architecture.md) | Layers, dependency rule, use cases | 1.5 giờ |
| 2 | [2-hexagonal-architecture.md](./2-hexagonal-architecture.md) | Ports & adapters pattern | 1.5 giờ |
| 3 | [3-design-patterns.md](./3-design-patterns.md) | Singleton, factory, observer, repository | 1.5 giờ |
| 4 | [4-microservices.md](./4-microservices.md) | Service decomposition, API gateway | 1.5 giờ |
| 5 | [5-event-driven.md](./5-event-driven.md) | Event sourcing, CQRS, saga pattern | 1.5 giờ |
| 6 | [6-monorepo.md](./6-monorepo.md) | Turborepo, Nx, pnpm workspaces | 1 giờ |
| 7 | [7-error-handling-architecture.md](./7-error-handling-architecture.md) | Result pattern, global error boundary | 1 giờ |

---

## Các Tài Liệu Chi Tiết

| File | Chủ Đề Chính |
| ---- | ------------ |
| [1-clean-architecture.md](./1-clean-architecture.md) | Entities, use cases, dependency inversion |
| [2-hexagonal-architecture.md](./2-hexagonal-architecture.md) | Primary/secondary ports, adapter swapping |
| [3-design-patterns.md](./3-design-patterns.md) | GoF patterns phổ biến trong Node.js |
| [4-microservices.md](./4-microservices.md) | Decomposition strategies, API Gateway, service mesh |
| [5-event-driven.md](./5-event-driven.md) | Kafka/RabbitMQ, event sourcing, distributed sagas |
| [6-monorepo.md](./6-monorepo.md) | Workspace setup, shared packages, CI optimization |
| [7-error-handling-architecture.md](./7-error-handling-architecture.md) | Result/Either, error taxonomy, middleware chain |

---

## Bài Tập Thực Hành

### Bài 1: Refactor Express App sang Clean Architecture

1. Lấy Express app CRUD đơn giản (users)
2. Tách thành: `domain/`, `application/`, `infrastructure/`, `presentation/`
3. Viết unit test cho use case mà không cần HTTP server

### Bài 2: Hexagonal với Repository Pattern

1. Định nghĩa `UserRepository` port (interface)
2. Implement `PostgresUserRepository` và `InMemoryUserRepository`
3. Swap adapter trong test vs production

### Bài 3: Modular Monolith

1. Chia monolith thành modules: `auth`, `orders`, `catalog`
2. Mỗi module có public API riêng — không import internal trực tiếp
3. Dùng ESLint `no-restricted-imports` enforce boundaries

### Bài 4: Event-Driven Mini System

1. Service A publish `OrderCreated` event qua Redis Pub/Sub
2. Service B subscribe và gửi email notification
3. Implement idempotent consumer (xử lý trùng lặp an toàn)

---

## Architecture Checklist

### Trước Khi Bắt Đầu Dự Án

- [ ] Xác định bounded contexts và core domain
- [ ] Chọn starting architecture phù hợp team size
- [ ] Định nghĩa folder structure và naming conventions
- [ ] Setup linting rules cho module boundaries
- [ ] Document ADR (Architecture Decision Record — Bản Ghi Quyết Định Kiến Trúc) cho quyết định quan trọng

### Trong Quá Trình Phát Triển

- [ ] Business logic không nằm trong controllers/routes
- [ ] Dependencies point inward (dependency rule)
- [ ] Interfaces cho external systems (DB, cache, third-party API)
- [ ] Error handling nhất quán across layers
- [ ] Tests cover use cases độc lập infrastructure

### Trước Khi Chuyển Microservices

- [ ] Đã có modular monolith với clear boundaries
- [ ] Team có thể deploy và operate services độc lập
- [ ] Observability stack sẵn sàng (logs, metrics, traces)
- [ ] Có lý do cụ thể: scale, deploy, team autonomy — không phải "because Netflix"

---

## Câu Hỏi Phỏng Vấn Thường Gặp

| Câu Hỏi | Điểm Cần Trả Lời |
| ------- | ---------------- |
| Clean Architecture vs Hexagonal? | Clean: layers + dependency rule. Hexagonal: ports/adapters, focus integration |
| Khi nào dùng microservices? | Team lớn, scale độc lập, deploy độc lập — không phải default |
| Repository pattern là gì? | Abstraction over data access — domain không biết PostgreSQL vs MongoDB |
| CQRS là gì? | Tách read model và write model — optimize riêng từng side |
| Saga pattern giải quyết gì? | Distributed transactions qua compensating actions |
| Monorepo vs polyrepo? | Monorepo: shared code, atomic changes. Polyrepo: team autonomy, deploy isolation |
| Dependency Inversion Principle? | High-level modules không phụ thuộc low-level — cả hai phụ thuộc abstractions |
| Result pattern vs throw? | Result: explicit errors, type-safe. Throw: simpler nhưng dễ miss catch |

---

## Liên Kết Chủ Đề Liên Quan

- **NestJS Modules & DI:** [03-web-frameworks/4-nestjs.md](../03-web-frameworks/4-nestjs.md) — Framework áp dụng kiến trúc có cấu trúc
- **Error Handling Async:** [02-async-programming/5-error-handling.md](../02-async-programming/5-error-handling.md) — Nền tảng xử lý lỗi
- **Performance & Scaling:** [07-performance/](../07-performance/) — Constraints khi thiết kế distributed system
- **Docker & K8s:** [09-cloud-deployment/](../09-cloud-deployment/) — Deploy kiến trúc microservices
- **Message Queues:** [10-advanced/7-job-queues.md](../10-advanced/7-job-queues.md) — BullMQ cho async processing

---

**Tiếp theo:** [1-clean-architecture.md](./1-clean-architecture.md) — Layers, dependency rule, và use cases
