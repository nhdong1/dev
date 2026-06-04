# Kế Hoạch Học 90 Ngày — C# .NET Developer

> Lộ trình học có cấu trúc 90 ngày, từ nền tảng đến sẵn sàng phỏng vấn Senior .NET Developer. Phù hợp cho người đang nâng cấp hoặc chuẩn bị chuyển job.

---

## Tổng Quan Kế Hoạch

```
Giai Đoạn 1 — Foundation (Ngày 1–30):    Vững nền tảng C# & ASP.NET Core
Giai Đoạn 2 — Intermediate (Ngày 31–60): Architecture, Testing, Security
Giai Đoạn 3 — Advanced (Ngày 61–90):     Performance, Microservices, Interview Prep
```

**Cam kết thời gian:**
- Minimum: 1.5 giờ/ngày (45 phút buổi sáng + 45 phút tối)
- Optimal: 2-3 giờ/ngày
- Ngày cuối tuần: 3-4 giờ (project practice)

**Cách học hiệu quả:**
```
Đọc lý thuyết (30%) → Xem ví dụ code (20%) → Tự viết code (40%) → Review & teach (10%)
```

---

## Giai Đoạn 1 — Foundation — Ngày 1 Đến 30

### Tuần 1 (Ngày 1–7) — C# Core & CLR

**Mục tiêu tuần:** Nắm chắc type system, memory model, và LINQ

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 1 | Value type vs Reference type, Nullable | `01-fundamentals/1-type-system.md` | Viết 10 ví dụ phân biệt value vs reference |
| 2 | CLR — Common Language Runtime — GC generations | `01-fundamentals/2-clr-and-memory.md` | Debug GC pressure với BenchmarkDotNet |
| 3 | Collections: List, Dictionary, HashSet, Queue | `01-fundamentals/3-collections.md` | Implement generic Stack từ đầu |
| 4 | LINQ: deferred execution, method chain | `01-fundamentals/4-linq.md` | Viết 5 LINQ queries phức tạp trên sample data |
| 5 | Delegates, Func, Action, event | `01-fundamentals/5-delegates-events.md` | Implement EventBus đơn giản |
| 6 | Exception handling, custom exceptions | `01-fundamentals/6-exception-handling.md` | Viết global exception handler |
| 7 | **Review + Mini Project** | — | Console app: File parser với LINQ + error handling |

**Checklist Tuần 1:**
- [ ] Giải thích được GC generations mà không nhìn tài liệu
- [ ] Viết được LINQ query phức tạp (GroupBy, Join, SelectMany)
- [ ] Tạo được custom exception hierarchy
- [ ] Implement được event system cơ bản

---

### Tuần 2 (Ngày 8–14) — OOP & Design Patterns

**Mục tiêu tuần:** SOLID và 10 design patterns quan trọng nhất

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 8 | SRP + OCP — Single Responsibility & Open/Closed | `02-oop-patterns/1-solid-principles.md` | Refactor god class thành SRP |
| 9 | LSP + ISP + DIP — Liskov, Interface Segregation, Dependency Inversion | `02-oop-patterns/1-solid-principles.md` | Viết ví dụ vi phạm và fix cho từng principle |
| 10 | Creational: Singleton, Factory, Builder | `02-oop-patterns/2-creational-patterns.md` | Implement Builder cho config object |
| 11 | Structural: Adapter, Decorator, Proxy | `02-oop-patterns/3-structural-patterns.md` | Decorator cho caching layer |
| 12 | Behavioral: Strategy, Observer, Mediator | `02-oop-patterns/4-behavioral-patterns.md` | Strategy cho payment processing |
| 13 | Anti-patterns: God Object, Service Locator | `02-oop-patterns/5-anti-patterns.md` | Nhận diện anti-pattern trong code sample |
| 14 | **Review + Project** | — | Refactor e-commerce cart với đúng patterns |

**Checklist Tuần 2:**
- [ ] Giải thích mỗi SOLID principle với ví dụ code cụ thể
- [ ] Implement được 5 patterns không cần nhìn tài liệu
- [ ] Nhận diện được anti-patterns trong code review

---

### Tuần 3 (Ngày 15–21) — Async/Await & Concurrency

**Mục tiêu tuần:** Không còn deadlock, hiểu state machine

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 15 | State machine, SynchronizationContext, ConfigureAwait | `03-async-concurrency/1-async-await-deep-dive.md` | Decompile async method, đọc state machine |
| 16 | Task.WhenAll, WhenAny, Parallel.ForEachAsync | `03-async-concurrency/2-task-parallel-library.md` | Parallel download 10 URLs với throttling |
| 17 | CancellationToken — thẻ hủy tác vụ | `03-async-concurrency/3-cancellation.md` | Implement cancellable file uploader |
| 18 | lock, Monitor, Interlocked, volatile | `03-async-concurrency/4-thread-safety.md` | Thread-safe counter và cache |
| 19 | Channel<T>, producer-consumer | `03-async-concurrency/5-channels-and-dataflow.md` | File processing pipeline |
| 20 | Deadlock causes và prevention | `03-async-concurrency/6-deadlock-prevention.md` | Recreate deadlock, then fix it |
| 21 | **Review + Project** | — | Async web scraper với rate limiting |

**Checklist Tuần 3:**
- [ ] Tạo ra deadlock có chủ ý và fix được
- [ ] Implement producer-consumer với Channel<T>
- [ ] Giải thích `ConfigureAwait(false)` khi nào dùng

---

### Tuần 4 (Ngày 22–30) — ASP.NET Core

**Mục tiêu tuần:** Build REST API hoàn chỉnh với DI, middleware, validation

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 22 | Middleware pipeline — chuỗi xử lý yêu cầu | `04-aspnet-core/1-middleware-pipeline.md` | Custom request timing middleware |
| 23 | DI: Singleton/Scoped/Transient, captive dependency | `04-aspnet-core/2-dependency-injection.md` | Reproduce captive dependency bug |
| 24 | Routing, attribute routing, route constraints | `04-aspnet-core/3-routing.md` | API versioning với route constraints |
| 25 | Minimal APIs vs Controllers | `04-aspnet-core/4-minimal-apis.md` | Build same API 2 cách, so sánh |
| 26 | Model binding, FluentValidation | `04-aspnet-core/5-model-binding-validation.md` | Validation pipeline với custom validators |
| 27 | Filters, global exception handler | `04-aspnet-core/6-filters.md` | Global error response format |
| 28 | Configuration, IOptions<T>, secrets | `04-aspnet-core/7-configuration.md` | Multi-environment configuration |
| 29 | Health checks — kiểm tra tình trạng | `04-aspnet-core/8-health-checks.md` | Health check cho DB + Redis + external API |
| 30 | **Review + Giai Đoạn 1 Project** | — | REST API: Product catalog với tất cả features |

**Giai Đoạn 1 Project:** Product Catalog API
```
Requirements:
- CRUD endpoints với validation
- JWT authentication (basic)
- Middleware: logging, timing, error handling
- Health check endpoint
- DI đúng lifetime
- Unit tests cơ bản
```

---

## Giai Đoạn 2 — Intermediate — Ngày 31 Đến 60

### Tuần 5 (Ngày 31–37) — Entity Framework Core

**Mục tiêu tuần:** EF Core production-ready, không N+1, optimize queries

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 31 | DbContext setup, pooling, connection string | `05-entity-framework/1-dbcontext-setup.md` | Configure DbContext với SQL Server |
| 32 | Code-First migrations, team conflicts | `05-entity-framework/2-migrations.md` | Simulate migration conflict và resolve |
| 33 | Relationships: 1-1, 1-N, N-N | `05-entity-framework/3-relationships.md` | E-commerce schema với tất cả relationships |
| 34 | LINQ queries, Include, ThenInclude, projection | `05-entity-framework/4-querying.md` | Complex reporting query với projection |
| 35 | N+1 Problem — phát hiện và fix | `05-entity-framework/5-n-plus-one-problem.md` | Dùng MiniProfiler để detect N+1 |
| 36 | Change tracking, AsNoTracking | `05-entity-framework/6-change-tracking.md` | So sánh performance tracking vs no-tracking |
| 37 | Compiled queries, raw SQL, Dapper | `05-entity-framework/7-performance-tips.md` | Benchmark compiled query vs LINQ |

**Checklist Tuần 5:**
- [ ] Detect và fix N+1 trong project thực tế
- [ ] Dùng được AsNoTracking đúng chỗ
- [ ] Viết compiled query cho hot path

---

### Tuần 6 (Ngày 38–44) — Testing Strategy

**Mục tiêu tuần:** Unit test, integration test, test coverage 80%+

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 38 | Unit testing: xUnit, AAA pattern, best practices | `06-testing/1-unit-testing.md` | Test suite cho Product service |
| 39 | Mocking với Moq: mock, stub, verify | `06-testing/2-mocking.md` | Mock external API calls |
| 40 | Integration testing với WebApplicationFactory | `06-testing/3-integration-testing.md` | API integration tests với TestContainers |
| 41 | TDD — Test-Driven Development — Red/Green/Refactor | `06-testing/4-tdd-guide.md` | Implement tính năng mới theo TDD |
| 42 | Test coverage: Coverlet, what to test | `06-testing/5-test-coverage.md` | Đạt 80% coverage cho Giai Đoạn 1 project |
| 43 | Performance testing: BenchmarkDotNet | `06-testing/6-performance-testing.md` | Benchmark query trước/sau optimization |
| 44 | **Review** | — | Thêm full test suite cho Giai Đoạn 1 project |

---

### Tuần 7 (Ngày 45–51) — Security

**Mục tiêu tuần:** Implement JWT auth từ đầu, hiểu OAuth2

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 45 | JWT: cấu trúc, signing, validation | `08-security/1-jwt-authentication.md` | Implement JWT auth từ đầu (không library) |
| 46 | ASP.NET Core Identity: user, role, claim | `08-security/2-aspnet-identity.md` | User registration + login với Identity |
| 47 | OAuth 2.0 + OpenID Connect flow | `08-security/3-oauth2-openidconnect.md` | Integrate Google OAuth vào API |
| 48 | Policy-based authorization — phân quyền theo chính sách | `08-security/4-authorization-policies.md` | Custom authorization policy |
| 49 | Data Protection API, HTTPS, HSTS | `08-security/5-data-protection.md` | Encrypt sensitive data at rest |
| 50 | CORS configuration | `08-security/7-cors.md` | Cấu hình CORS đúng cho production |
| 51 | Input validation, SQL Injection, XSS prevention | `08-security/8-secure-coding.md` | Security audit Giai Đoạn 1 project |

---

### Tuần 8 (Ngày 52–60) — Architecture Patterns

**Mục tiêu tuần:** Clean Architecture và CQRS với MediatR

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 52 | Clean Architecture: layers, dependency rule | `09-architecture/1-clean-architecture.md` | Setup Clean Architecture project template |
| 53 | DDD — Domain-Driven Design: Aggregate, Entity, Value Object | `09-architecture/2-ddd-fundamentals.md` | Model Order aggregate với domain events |
| 54 | CQRS — Command Query Responsibility Segregation: command side | `09-architecture/3-cqrs-pattern.md` | Implement CreateOrder command với MediatR |
| 55 | CQRS — query side, read model optimization | `09-architecture/3-cqrs-pattern.md` | Implement GetOrders query với projection |
| 56 | Event Sourcing cơ bản | `09-architecture/4-event-sourcing.md` | Simple event store cho Order |
| 57 | Messaging: MassTransit, RabbitMQ, Outbox | `09-architecture/6-messaging-patterns.md` | Publish event sau order creation |
| 58 | Vertical Slice Architecture | `09-architecture/7-vertical-slice.md` | Refactor 1 feature sang vertical slice |
| 59-60 | **Giai Đoạn 2 Project** | — | Clean Architecture + CQRS E-commerce backend |

**Giai Đoạn 2 Project:** E-Commerce Backend
```
Requirements:
- Clean Architecture (4 layers)
- CQRS với MediatR cho orders
- JWT auth + role-based authorization
- EF Core với proper N+1 fix
- Integration tests với TestContainers
- Outbox pattern cho email notification
```

---

## Giai Đoạn 3 — Advanced & Interview Prep — Ngày 61 Đến 90

### Tuần 9 (Ngày 61–67) — Performance & Memory

**Mục tiêu tuần:** Profiling, Span<T>, zero-allocation patterns

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 61 | GC internals, Gen 0/1/2, LOH | `07-performance/1-memory-management.md` | Memory profiling với dotMemory |
| 62 | Span<T>, Memory<T> — zero-allocation | `07-performance/2-span-and-memory.md` | Rewrite string parser với Span<T> |
| 63 | ArrayPool<T>, ObjectPool<T> — tái dụng bộ nhớ | `07-performance/3-pooling-strategies.md` | HTTP middleware dùng ArrayPool |
| 64 | BenchmarkDotNet — đo lường hiệu năng | `07-performance/4-benchmarking.md` | Benchmark trước/sau tối ưu |
| 65 | Caching: IMemoryCache, Redis, cache-aside | `07-performance/5-caching.md` | Thêm Redis cache vào Giai Đoạn 2 project |
| 66 | Profiling: dotnet-trace, dotnet-dump, PerfView | `07-performance/6-profiling-tools.md` | Profile Giai Đoạn 2 project, tìm hotspot |
| 67 | **Performance Review** | — | Optimize 3 hotspot tìm được |

---

### Tuần 10 (Ngày 68–74) — Cloud & Deployment

**Mục tiêu tuần:** Docker, K8s deployment, CI/CD, observability

| Ngày | Chủ Đề | File Tham Khảo | Bài Tập |
| ---- | ------- | -------------- | ------- |
| 68 | Dockerfile cho .NET, multi-stage build | `10-cloud-deployment/1-docker-dotnet.md` | Containerize Giai Đoạn 2 project |
| 69 | Kubernetes: Deployment, Service, ConfigMap | `10-cloud-deployment/2-kubernetes-deployment.md` | Deploy lên local K8s (minikube) |
| 70 | Azure App Service hoặc AKS | `10-cloud-deployment/3-azure-services.md` | Deploy lên Azure (free tier) |
| 71 | GitHub Actions CI/CD | `10-cloud-deployment/4-cicd-pipeline.md` | CI pipeline: build → test → deploy |
| 72 | Secrets management, environment variables | `10-cloud-deployment/5-configuration-management.md` | Azure Key Vault integration |
| 73 | OpenTelemetry, Serilog, structured logging | `10-cloud-deployment/6-observability.md` | Distributed tracing cho API calls |
| 74 | **Review** | — | Full deployment pipeline Giai Đoạn 2 project |

---

### Tuần 11 (Ngày 75–81) — Interview Prep Phase 1

**Mục tiêu tuần:** Nắm chắc 30 câu hỏi phỏng vấn, system design

| Ngày | Hoạt Động | Tài Liệu |
| ---- | --------- | -------- |
| 75 | Đọc toàn bộ INTERVIEW_GUIDE.md, note câu chưa chắc | `11-interview-prep/INTERVIEW_GUIDE.md` |
| 76 | Deep dive Q1–Q10 (C# Language & CLR), giải thích to | Không nhìn tài liệu |
| 77 | Deep dive Q11–Q20 (ASP.NET Core, EF Core), giải thích to | Không nhìn tài liệu |
| 78 | Deep dive Q21–Q30 (Architecture, Testing, Security) | Không nhìn tài liệu |
| 79 | System Design: URL Shortener + E-Commerce Cart | `11-interview-prep/1-system-design-scenarios.md` |
| 80 | System Design: Rate Limiter + Notification System | `11-interview-prep/1-system-design-scenarios.md` |
| 81 | Code Challenges: String, LINQ, Async | `11-interview-prep/3-code-challenges.md` |

---

### Tuần 12 (Ngày 82–90) — Final Sprint & Mock Interviews

**Mục tiêu tuần:** Tự tin, fluent, sẵn sàng phỏng vấn

| Ngày | Hoạt Động | Ghi Chú |
| ---- | --------- | ------- |
| 82 | Viết/review 6 câu chuyện STAR | `11-interview-prep/2-star-stories.md` |
| 83 | Practice behavioral questions to một mình | `11-interview-prep/4-behavioral-questions.md` |
| 84 | **Mock Interview #1** — Technical (nhờ bạn bè hoặc tự record) | Tập trung: C#, ASP.NET, Architecture |
| 85 | Review Mock #1, note điểm yếu | — |
| 86 | **Mock Interview #2** — System Design | URL Shortener hoặc E-Commerce |
| 87 | Review Mock #2, strengthen weak areas | — |
| 88 | **Mock Interview #3** — Behavioral | 10 câu behavioral questions |
| 89 | Final review: top 10 câu quan trọng nhất | — |
| 90 | **Ngày cuối:** Nghỉ ngơi, review CV, chuẩn bị tâm lý | ✅ Bạn đã sẵn sàng! |

---

## Tracking Tiến Độ

### Giai Đoạn 1 — Checklist

```
Tuần 1 — C# Core:
  [ ] Value type vs reference type (thuộc lòng)
  [ ] GC generations và LOH
  [ ] LINQ: deferred vs immediate execution
  [ ] Delegates, Func, Action, event
  [ ] Exception hierarchy + global handler

Tuần 2 — Design Patterns:
  [ ] SOLID: 1 ví dụ code cho mỗi principle
  [ ] Creational: Singleton, Factory, Builder
  [ ] Structural: Adapter, Decorator, Proxy
  [ ] Behavioral: Strategy, Observer, Mediator
  [ ] Anti-patterns: God Object, Service Locator

Tuần 3 — Async:
  [ ] async/await: không deadlock khi dùng .Result
  [ ] ConfigureAwait(false): khi nào dùng
  [ ] CancellationToken: propagate đúng cách
  [ ] Thread-safe code: lock vs Interlocked
  [ ] Channel<T>: producer-consumer

Tuần 4 — ASP.NET Core:
  [ ] Middleware order: Exception → HTTPS → Auth → Endpoints
  [ ] DI lifetimes: captive dependency pitfall
  [ ] Model validation: FluentValidation
  [ ] Filters: Action vs Exception vs Authorization
  [ ] Health checks: liveness vs readiness
```

### Giai Đoạn 2 — Checklist

```
Tuần 5 — EF Core:
  [ ] N+1: detect với logging, fix với Include
  [ ] AsNoTracking: read-only queries
  [ ] Compiled queries: hot path optimization
  [ ] Migration: team conflict resolution

Tuần 6 — Testing:
  [ ] Unit test: AAA, mock dependencies
  [ ] Integration test: WebApplicationFactory
  [ ] TDD: Red → Green → Refactor cycle
  [ ] Coverage: 80%+ cho business logic

Tuần 7 — Security:
  [ ] JWT: structure, signing, validation flow
  [ ] OAuth 2.0: authorization code flow
  [ ] Authorization: policy-based vs role-based
  [ ] CORS: whitelist origins, không AllowAll production

Tuần 8 — Architecture:
  [ ] Clean Architecture: dependency rule
  [ ] DDD: Aggregate root, domain events
  [ ] CQRS: command vs query separation
  [ ] Outbox pattern: at-least-once delivery
```

### Giai Đoạn 3 — Checklist

```
Tuần 9 — Performance:
  [ ] Span<T>: zero-allocation string processing
  [ ] ArrayPool: avoid LOH allocations
  [ ] Redis cache: cache-aside pattern
  [ ] BenchmarkDotNet: measure before optimize

Tuần 10 — Cloud:
  [ ] Docker: multi-stage build, minimal image
  [ ] K8s: Deployment, liveness/readiness probes
  [ ] GitHub Actions: build → test → deploy pipeline
  [ ] OpenTelemetry: traces, logs, metrics

Tuần 11-12 — Interview Ready:
  [ ] 30 câu hỏi: giải thích được không nhìn
  [ ] 2 system design scenarios: vẽ được sơ đồ
  [ ] 6 STAR stories: kể trong 2-3 phút mỗi cái
  [ ] 5 câu hỏi để hỏi ngược lại interviewer
  [ ] Mock interview: thực hành ≥ 3 lần
```

---

## Kế Hoạch Dự Phòng

### Nếu Chậm Hơn Kế Hoạch

```
Bỏ qua (có thể học sau):
• Event Sourcing chi tiết
• Azure services specific
• Performance testing chi tiết

Không bỏ qua:
• SOLID + top 5 Design Patterns
• Async/await fundamentals
• JWT authentication
• N+1 Problem fix
• Clean Architecture cơ bản
• 30 câu hỏi phỏng vấn
• 3 STAR stories
```

### Nếu Có Phỏng Vấn Gấp (1-2 tuần)

Xem [README.md](./README.md) — phần "Có 1 Tuần"

---

## Resources Học Kèm

### Đọc Hàng Ngày (5-10 phút)

- [Andrew Lock's Blog](https://andrewlock.net/) — ASP.NET Core tips
- [Nick Chapsas YouTube](https://www.youtube.com/@nickchapsas) — .NET performance
- [Milan Jovanović Blog](https://www.milanjovanovic.tech/) — Architecture

### Sách Theo Thứ Tự Ưu Tiên

```
Ưu tiên 1 (Giai Đoạn 1-2):
• "C# in Depth" — Jon Skeet
• "Pro ASP.NET Core" — Adam Freeman

Ưu tiên 2 (Giai Đoạn 2-3):
• "Clean Architecture" — Robert C. Martin
• "Domain-Driven Design" — Eric Evans (chọn lọc)

Ưu tiên 3 (Nâng cao):
• "Designing Data-Intensive Applications" — Martin Kleppmann
• "Building Microservices" — Sam Newman
```

### Môi Trường Thực Hành

```bash
# Setup cơ bản
dotnet new sln -n PracticeProjects
dotnet new webapi -n PracticeApi -o src/PracticeApi
dotnet new xunit -n PracticeApi.Tests -o tests/PracticeApi.Tests
dotnet sln add src/PracticeApi
dotnet sln add tests/PracticeApi.Tests

# Docker local development
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=YourPassword123!" \
  -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest

docker run -p 6379:6379 -d redis:latest

# Khởi động Redis nhanh cho test
dotnet add package StackExchange.Redis
```

---

## Đo Lường Thành Công Sau 90 Ngày

### Kỹ Thuật

- [ ] Giải thích async/await state machine không cần notes
- [ ] Viết Clean Architecture project từ đầu trong 2 giờ
- [ ] Debug N+1 query bằng mắt khi đọc code
- [ ] Implement JWT auth trong 30 phút
- [ ] Thiết kế system với 10,000 RPS reasonable

### Phỏng Vấn Ready

- [ ] Trả lời 25/30 câu hỏi tự tin, fluent
- [ ] System design session 45 phút không bị bí
- [ ] 3 câu STAR stories kể tự nhiên, không đọc
- [ ] Đặt 5 câu hỏi thông minh cho interviewer
- [ ] Đã làm ít nhất 3 mock interviews

### Portfolio

- [ ] 2 projects trên GitHub với README rõ ràng
- [ ] CI/CD pipeline chạy được
- [ ] Test coverage ≥ 75%
- [ ] Docker-ready (có Dockerfile)

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
