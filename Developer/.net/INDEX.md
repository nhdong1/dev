# C# .NET Knowledge Base — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về lập trình C# và hệ sinh thái .NET

## 📁 Cấu Trúc Thư Mục

```
Developer/.net/
├── README.md                                   [BẮT ĐẦU ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    [FILE NÀY] Chỉ mục đầy đủ
│
├── 01-fundamentals/
│   ├── README.md                               Nền tảng C#, CLR, type system
│   ├── 1-type-system.md                          Value types vs Reference types, Nullable
│   ├── 2-clr-and-memory.md                       CLR, GC, Stack vs Heap, generations
│   ├── 3-collections.md                          List, Dictionary, IEnumerable, Span
│   ├── 4-linq.md                                 LINQ queries: deferred, immediate execution
│   ├── 5-delegates-events.md                     Delegate, Func, Action, event, EventHandler
│   └── 6-exception-handling.md                   try/catch/finally, custom exceptions, AggregateException
│
├── 02-oop-patterns/
│   ├── README.md                               OOP, SOLID, Design Patterns tổng quan
│   ├── 1-solid-principles.md                     SRP, OCP, LSP, ISP, DIP với ví dụ C#
│   ├── 2-creational-patterns.md                  Singleton, Factory, Abstract Factory, Builder, Prototype
│   ├── 3-structural-patterns.md                  Adapter, Decorator, Facade, Proxy, Composite
│   ├── 4-behavioral-patterns.md                  Strategy, Observer, Mediator, Command, Chain of Responsibility
│   └── 5-anti-patterns.md                        God Object, Tight Coupling, Service Locator — cái cần tránh
│
├── 03-async-concurrency/
│   ├── README.md                               Async/Await, Task, Thread, concurrency overview
│   ├── 1-async-await-deep-dive.md                State machine, SynchronizationContext, ConfigureAwait
│   ├── 2-task-parallel-library.md                Task, Parallel.For, PLINQ, Task.WhenAll/WhenAny
│   ├── 3-cancellation.md                         CancellationToken, timeout, cooperative cancellation
│   ├── 4-thread-safety.md                        lock, Monitor, Interlocked, volatile, immutability
│   ├── 5-channels-and-dataflow.md                Channel<T>, producer-consumer, System.Threading.Channels
│   └── 6-deadlock-prevention.md                  Nguyên nhân deadlock, cách phát hiện và phòng tránh
│
├── 04-aspnet-core/
│   ├── README.md                               ASP.NET Core pipeline, hosting, DI tổng quan
│   ├── 1-middleware-pipeline.md                  Request pipeline, custom middleware, short-circuit
│   ├── 2-dependency-injection.md                 Singleton/Scoped/Transient, lifetime pitfalls
│   ├── 3-routing.md                              Attribute routing, conventional routing, route constraints
│   ├── 4-minimal-apis.md                         Minimal API vs Controllers, endpoint filters
│   ├── 5-model-binding-validation.md             Model binding, Data Annotations, FluentValidation
│   ├── 6-filters.md                              Action, Exception, Authorization, Resource filters
│   ├── 7-configuration.md                        appsettings, IOptions<T>, environment variables, secrets
│   └── 8-health-checks.md                        Health endpoint, liveness, readiness, dependency checks
│
├── 05-entity-framework/
│   ├── README.md                               EF Core overview, DbContext, Code-First
│   ├── 1-dbcontext-setup.md                      DbContext, connection string, pooling
│   ├── 2-migrations.md                           Code-First migrations, rollback, team conflicts
│   ├── 3-relationships.md                        1-1, 1-N, N-N, owned entities, table splitting
│   ├── 4-querying.md                             LINQ queries, Include, ThenInclude, projection
│   ├── 5-n-plus-one-problem.md                   Phát hiện và giải quyết N+1 Query
│   ├── 6-change-tracking.md                      AsNoTracking, Attach, EntityState
│   ├── 7-performance-tips.md                     Compiled queries, split queries, raw SQL, Dapper so sánh
│   └── 8-testing-ef-core.md                      InMemory provider, SQLite, mocking DbContext
│
├── 06-testing/
│   ├── README.md                               Chiến lược kiểm thử, test pyramid
│   ├── 1-unit-testing.md                         xUnit, NUnit, MSTest — best practices
│   ├── 2-mocking.md                              Moq, NSubstitute — mock, stub, spy
│   ├── 3-integration-testing.md                  WebApplicationFactory, TestServer, test containers
│   ├── 4-tdd-guide.md                            TDD — Test-Driven Development — Red/Green/Refactor
│   ├── 5-test-coverage.md                        Coverlet, test coverage analysis, what to test
│   └── 6-performance-testing.md                  BenchmarkDotNet, load testing với k6/NBomber
│
├── 07-performance/
│   ├── README.md                               Performance overview, profiling tools
│   ├── 1-memory-management.md                    GC internals, Gen 0/1/2, LOH — Large Object Heap
│   ├── 2-span-and-memory.md                      Span<T>, Memory<T>, ReadOnlySpan — zero-allocation
│   ├── 3-pooling-strategies.md                   ArrayPool<T>, MemoryPool<T>, ObjectPool<T>
│   ├── 4-benchmarking.md                         BenchmarkDotNet, thiết lập benchmark đúng cách
│   ├── 5-caching.md                              IMemoryCache, IDistributedCache, Redis, cache-aside pattern
│   └── 6-profiling-tools.md                      dotnet-trace, dotnet-dump, PerfView, JetBrains dotMemory
│
├── 08-security/
│   ├── README.md                               Security overview, authentication vs authorization
│   ├── 1-jwt-authentication.md                   JWT — JSON Web Token — cấu trúc, signing, validation
│   ├── 2-aspnet-identity.md                      ASP.NET Core Identity — user, role, claim management
│   ├── 3-oauth2-openidconnect.md                 OAuth 2.0, OpenID Connect, authorization code flow
│   ├── 4-authorization-policies.md               Policy-based, claims-based, resource-based authorization
│   ├── 5-data-protection.md                      ASP.NET Core Data Protection API, key rotation
│   ├── 6-https-and-tls.md                        HTTPS enforcement, HSTS, certificate management
│   ├── 7-cors.md                                 CORS — Cross-Origin Resource Sharing — cấu hình đúng cách
│   └── 8-secure-coding.md                        Input validation, SQL Injection, XSS, CSRF prevention
│
├── 09-architecture/
│   ├── README.md                               Kiến trúc phần mềm overview, khi nào dùng gì
│   ├── 1-clean-architecture.md                   Layers: Domain / Application / Infrastructure / Presentation
│   ├── 2-ddd-fundamentals.md                     DDD — Domain-Driven Design: Aggregate, Entity, Value Object
│   ├── 3-cqrs-pattern.md                         CQRS — Command/Query split, MediatR integration
│   ├── 4-event-sourcing.md                       Event Sourcing, event store, replay, snapshots
│   ├── 5-microservices-basics.md                 Decomposition, inter-service communication, API Gateway
│   ├── 6-messaging-patterns.md                   MassTransit, RabbitMQ, Azure Service Bus, Outbox Pattern
│   └── 7-vertical-slice.md                       Vertical Slice Architecture — thay thế cho layered
│
├── 10-cloud-deployment/
│   ├── README.md                               Cloud & deployment overview
│   ├── 1-docker-dotnet.md                        Dockerfile cho .NET, multi-stage build, image optimization
│   ├── 2-kubernetes-deployment.md                K8s Deployment, Service, ConfigMap, Secret cho .NET app
│   ├── 3-azure-services.md                       Azure App Service, AKS, Azure Functions, Azure SQL
│   ├── 4-cicd-pipeline.md                        GitHub Actions và Azure DevOps cho .NET
│   ├── 5-configuration-management.md             Secrets, environment variables, Azure Key Vault
│   └── 6-observability.md                        OpenTelemetry, structured logging (Serilog), metrics, tracing
│
├── 11-interview-prep/
│   ├── README.md                               Hướng dẫn ôn tập phỏng vấn
│   ├── INTERVIEW_GUIDE.md                      Top 30 câu hỏi phỏng vấn C# .NET với đáp án
│   ├── 1-system-design-scenarios.md              Kịch bản thiết kế hệ thống thực tế
│   ├── 2-star-stories.md                         Mẫu câu chuyện STAR theo từng chủ đề
│   ├── 3-code-challenges.md                      Bài tập code thường gặp và cách tiếp cận
│   ├── 4-behavioral-questions.md                 Câu hỏi về teamwork, conflict, growth
│   └── 5-90-day-study-plan.md                    Kế hoạch học 90 ngày có cấu trúc
│
├── ROADMAP.md                                  (Sẽ tạo) Lộ trình 90 ngày chi tiết
├── GLOSSARY.md                                 ✅ Từ điển thuật ngữ C# .NET
├── RESOURCES.md                                (Sẽ tạo) Sách, blog, khóa học, công cụ
└── CHECKLIST.md                                (Sẽ tạo) Checklist trước phỏng vấn, trước production
```

---

## ✅ Đã Hoàn Thành

| Chủ Đề                        | File                                                              | Trạng Thái | Chất Lượng    |
| ----------------------------- | ----------------------------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**      | README.md                                                         | ✅         | Toàn diện     |
| **Chỉ Mục Đầy Đủ**            | INDEX.md                                                          | ✅         | Toàn diện     |
| **Nền Tảng C# — Overview**    | 01-fundamentals/README.md                                         | ✅         | Toàn diện     |
| **Type System**               | 01-fundamentals/1-type-system.md                                  | ✅         | Toàn diện     |
| **CLR & Memory**              | 01-fundamentals/2-clr-and-memory.md                               | ✅         | Toàn diện     |
| **Collections**               | 01-fundamentals/3-collections.md                                  | ✅         | Toàn diện     |
| **LINQ**                      | 01-fundamentals/4-linq.md                                         | ✅         | Toàn diện     |
| **Delegates & Events**        | 01-fundamentals/5-delegates-events.md                             | ✅         | Toàn diện     |
| **Exception Handling**        | 01-fundamentals/6-exception-handling.md                           | ✅         | Toàn diện     |
| **OOP & Patterns — Overview** | 02-oop-patterns/README.md                                         | ✅         | Toàn diện     |
| **SOLID Principles**          | 02-oop-patterns/1-solid-principles.md                             | ✅         | Toàn diện     |
| **Creational Patterns**       | 02-oop-patterns/2-creational-patterns.md                          | ✅         | Toàn diện     |
| **Structural Patterns**       | 02-oop-patterns/3-structural-patterns.md                          | ✅         | Toàn diện     |
| **Behavioral Patterns**       | 02-oop-patterns/4-behavioral-patterns.md                          | ✅         | Toàn diện     |
| **Anti-Patterns**             | 02-oop-patterns/5-anti-patterns.md                                | ✅         | Toàn diện     |
| **Async — Overview**          | 03-async-concurrency/README.md                                    | ✅         | Toàn diện     |
| **Async/Await Deep Dive**     | 03-async-concurrency/1-async-await-deep-dive.md                   | ✅         | Toàn diện     |
| **Task Parallel Library**     | 03-async-concurrency/2-task-parallel-library.md                   | ✅         | Toàn diện     |
| **Cancellation**              | 03-async-concurrency/3-cancellation.md                            | ✅         | Toàn diện     |
| **Thread Safety**             | 03-async-concurrency/4-thread-safety.md                           | ✅         | Toàn diện     |
| **Channels & Dataflow**       | 03-async-concurrency/5-channels-and-dataflow.md                   | ✅         | Toàn diện     |
| **Deadlock Prevention**       | 03-async-concurrency/6-deadlock-prevention.md                     | ✅         | Toàn diện     |
| **ASP.NET Core — Overview**   | 04-aspnet-core/README.md                                          | ✅         | Toàn diện     |
| **Middleware Pipeline**        | 04-aspnet-core/1-middleware-pipeline.md                           | ✅         | Toàn diện     |
| **Dependency Injection**       | 04-aspnet-core/2-dependency-injection.md                          | ✅         | Toàn diện     |
| **Routing**                    | 04-aspnet-core/3-routing.md                                       | ✅         | Toàn diện     |
| **Minimal APIs**               | 04-aspnet-core/4-minimal-apis.md                                  | ✅         | Toàn diện     |
| **Model Binding & Validation** | 04-aspnet-core/5-model-binding-validation.md                      | ✅         | Toàn diện     |
| **Filters**                    | 04-aspnet-core/6-filters.md                                       | ✅         | Toàn diện     |
| **Configuration**              | 04-aspnet-core/7-configuration.md                                 | ✅         | Toàn diện     |
| **Health Checks**              | 04-aspnet-core/8-health-checks.md                                 | ✅         | Toàn diện     |
| **EF Core — Overview**         | 05-entity-framework/README.md                                     | ✅         | Toàn diện     |
| **DbContext Setup**            | 05-entity-framework/1-dbcontext-setup.md                          | ✅         | Toàn diện     |
| **Migrations**                 | 05-entity-framework/2-migrations.md                               | ✅         | Toàn diện     |
| **Relationships**              | 05-entity-framework/3-relationships.md                            | ✅         | Toàn diện     |
| **Querying**                   | 05-entity-framework/4-querying.md                                 | ✅         | Toàn diện     |
| **N+1 Problem**                | 05-entity-framework/5-n-plus-one-problem.md                       | ✅         | Toàn diện     |
| **Change Tracking**            | 05-entity-framework/6-change-tracking.md                          | ✅         | Toàn diện     |
| **Performance Tips**           | 05-entity-framework/7-performance-tips.md                         | ✅         | Toàn diện     |
| **Testing EF Core**            | 05-entity-framework/8-testing-ef-core.md                          | ✅         | Toàn diện     |
| **Testing — Overview**         | 06-testing/README.md                                              | ✅         | Toàn diện     |
| **Unit Testing**               | 06-testing/1-unit-testing.md                                      | ✅         | Toàn diện     |
| **Mocking**                    | 06-testing/2-mocking.md                                           | ✅         | Toàn diện     |
| **Integration Testing**        | 06-testing/3-integration-testing.md                               | ✅         | Toàn diện     |
| **TDD Guide**                  | 06-testing/4-tdd-guide.md                                         | ✅         | Toàn diện     |
| **Test Coverage**              | 06-testing/5-test-coverage.md                                     | ✅         | Toàn diện     |
| **Performance Testing**        | 06-testing/6-performance-testing.md                               | ✅         | Toàn diện     |
| **Performance — Overview**     | 07-performance/README.md                                          | ✅         | Toàn diện     |
| **Memory Management**          | 07-performance/1-memory-management.md                             | ✅         | Toàn diện     |
| **Span & Memory**              | 07-performance/2-span-and-memory.md                               | ✅         | Toàn diện     |
| **Pooling Strategies**         | 07-performance/3-pooling-strategies.md                            | ✅         | Toàn diện     |
| **Benchmarking**               | 07-performance/4-benchmarking.md                                  | ✅         | Toàn diện     |
| **Caching**                    | 07-performance/5-caching.md                                       | ✅         | Toàn diện     |
| **Profiling Tools**            | 07-performance/6-profiling-tools.md                               | ✅         | Toàn diện     |
| **Security — Overview**        | 08-security/README.md                                             | ✅         | Toàn diện     |
| **JWT Authentication**         | 08-security/1-jwt-authentication.md                               | ✅         | Toàn diện     |
| **ASP.NET Core Identity**      | 08-security/2-aspnet-identity.md                                  | ✅         | Toàn diện     |
| **OAuth2 & OpenID Connect**    | 08-security/3-oauth2-openidconnect.md                             | ✅         | Toàn diện     |
| **Authorization Policies**     | 08-security/4-authorization-policies.md                           | ✅         | Toàn diện     |
| **Data Protection API**        | 08-security/5-data-protection.md                                  | ✅         | Toàn diện     |
| **HTTPS & TLS**                | 08-security/6-https-and-tls.md                                    | ✅         | Toàn diện     |
| **CORS**                       | 08-security/7-cors.md                                             | ✅         | Toàn diện     |
| **Secure Coding**              | 08-security/8-secure-coding.md                                    | ✅         | Toàn diện     |
| **Architecture — Overview**    | 09-architecture/README.md                                         | ✅         | Toàn diện     |
| **Clean Architecture**         | 09-architecture/1-clean-architecture.md                           | ✅         | Toàn diện     |
| **DDD Fundamentals**           | 09-architecture/2-ddd-fundamentals.md                             | ✅         | Toàn diện     |
| **CQRS Pattern**               | 09-architecture/3-cqrs-pattern.md                                 | ✅         | Toàn diện     |
| **Event Sourcing**             | 09-architecture/4-event-sourcing.md                               | ✅         | Toàn diện     |
| **Microservices Basics**       | 09-architecture/5-microservices-basics.md                         | ✅         | Toàn diện     |
| **Messaging Patterns**         | 09-architecture/6-messaging-patterns.md                           | ✅         | Toàn diện     |
| **Vertical Slice Architecture**| 09-architecture/7-vertical-slice.md                               | ✅         | Toàn diện     |
| **Cloud & Deployment — Overview** | 10-cloud-deployment/README.md                              | ✅         | Toàn diện     |
| **Docker cho .NET**            | 10-cloud-deployment/1-docker-dotnet.md                            | ✅         | Toàn diện     |
| **Kubernetes Deployment**      | 10-cloud-deployment/2-kubernetes-deployment.md                    | ✅         | Toàn diện     |
| **Azure Services**             | 10-cloud-deployment/3-azure-services.md                           | ✅         | Toàn diện     |
| **CI/CD Pipeline**             | 10-cloud-deployment/4-cicd-pipeline.md                            | ✅         | Toàn diện     |
| **Configuration Management**   | 10-cloud-deployment/5-configuration-management.md                 | ✅         | Toàn diện     |
| **Observability**              | 10-cloud-deployment/6-observability.md                            | ✅         | Toàn diện     |
| **Interview Prep — Overview**  | 11-interview-prep/README.md                                       | ✅         | Toàn diện     |
| **Top 30 Q&A Phỏng Vấn**       | 11-interview-prep/INTERVIEW_GUIDE.md                              | ✅         | Toàn diện     |
| **System Design Scenarios**    | 11-interview-prep/1-system-design-scenarios.md                    | ✅         | Toàn diện     |
| **STAR Stories**               | 11-interview-prep/2-star-stories.md                               | ✅         | Toàn diện     |
| **Code Challenges**            | 11-interview-prep/3-code-challenges.md                            | ✅         | Toàn diện     |
| **Behavioral Questions**       | 11-interview-prep/4-behavioral-questions.md                       | ✅         | Toàn diện     |
| **Kế Hoạch Học 90 Ngày**       | 11-interview-prep/5-90-day-study-plan.md                          | ✅         | Toàn diện     |
| **Từ Điển Thuật Ngữ**          | GLOSSARY.md                                                       | ✅         | Toàn diện     |

---

## 🎯 Cần Tạo Tiếp Theo (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao (Kỹ năng cốt lõi .NET)

- [x] `01-fundamentals/` — CLR, type system, LINQ, collections ✅ **Hoàn thành 2026-06-02**
- [x] `02-oop-patterns/` — SOLID, Design Patterns với ví dụ C# thực tế ✅ **Hoàn thành 2026-06-02**
- [x] `03-async-concurrency/` — Async/Await deep dive, TPL, deadlock ✅ **Hoàn thành 2026-06-02**
- [x] `04-aspnet-core/` — Middleware, DI, routing, configuration ✅ **Hoàn thành 2026-06-02**
- [x] `05-entity-framework/` — EF Core, N+1, migrations ✅ **Hoàn thành 2026-06-02**
- [x] `06-testing/` — Unit test, integration test, TDD ✅ **Hoàn thành 2026-06-02**
- [x] `11-interview-prep/` — Bộ câu hỏi phỏng vấn, STAR stories, system design, code challenges ✅ **Hoàn thành 2026-06-02**

### Ưu Tiên Trung Bình (Kỹ năng chuyên sâu)

- [x] `08-security/` — JWT, OAuth2, authorization policies ✅ **Hoàn thành 2026-06-02**
- [x] `09-architecture/` — Clean Architecture, DDD, CQRS ✅ **Hoàn thành 2026-06-02**

### Ưu Tiên Thấp (Nâng cao & tham khảo)

- [x] `07-performance/` — Span<T>, GC tuning, benchmarking ✅ **Hoàn thành 2026-06-02**
- [x] `10-cloud-deployment/` — Docker, K8s, Azure, CI/CD ✅ **Hoàn thành 2026-06-02**
- [x] `GLOSSARY.md` — Từ điển thuật ngữ ✅ **Hoàn thành 2026-06-02**
- [ ] `RESOURCES.md` — Tài nguyên học tập
- [ ] `CHECKLIST.md` — Checklist kiểm tra

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Dành Cho Tự Học

```
1. Đọc README.md để nắm tổng quan
2. Chọn Learning Path phù hợp cấp độ của bạn
3. Học tuần tự từng section, không bỏ qua
4. Tự viết lại ví dụ code, đừng chỉ đọc
5. Xây dựng project portfolio áp dụng kiến thức
```

### Dành Cho Ôn Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md trước
2. Tập trung vào 02-oop-patterns/ (SOLID luôn được hỏi)
3. Ôn 03-async-concurrency/ (async/await rất quan trọng)
4. Chuẩn bị câu chuyện thực chiến theo STAR
5. Luyện giải thích khái niệm không cần nhìn tài liệu
```

### Dành Cho Công Việc Thực Tế

```
Dùng như tài liệu tham khảo:
- Đang thiết kế: Xem 09-architecture/ để chọn pattern phù hợp
- Debug performance: Dùng 07-performance/ để phân tích
- Cài security: Theo 08-security/ checklist
- Review code: Đối chiếu với 02-oop-patterns/solid-principles.md
- Deploy: Theo 10-cloud-deployment/ runbook
```

### Dành Cho System Design Interview

```
1. Đọc 09-architecture/ để hiểu các pattern
2. Tham khảo 04-aspnet-core/ cho API design
3. Dùng 05-entity-framework/ cho data layer
4. Áp dụng 08-security/ cho auth layer
5. Kết hợp 10-cloud-deployment/ cho infrastructure
```

---

## 📊 Ước Tính Thời Gian Học

| Section                        | Thời Gian    | Độ Khó  | Ưu Tiên  |
| ------------------------------ | ------------ | ------- | -------- |
| Fundamentals (C#, CLR)         | 4–6 giờ      | ⭐      | Bắt buộc |
| OOP & Design Patterns          | 8–10 giờ     | ⭐⭐    | Bắt buộc |
| Async & Concurrency            | 6–8 giờ      | ⭐⭐⭐  | Bắt buộc |
| ASP.NET Core                   | 8–10 giờ     | ⭐⭐    | Bắt buộc |
| Entity Framework Core          | 6–8 giờ      | ⭐⭐    | Bắt buộc |
| Testing                        | 4–6 giờ      | ⭐⭐    | Nên có   |
| Performance & Memory           | 6–8 giờ      | ⭐⭐⭐  | Nên có   |
| Security                       | 6–8 giờ      | ⭐⭐    | Bắt buộc |
| Architecture (DDD, CQRS)       | 10–15 giờ    | ⭐⭐⭐  | Nên có   |
| Cloud & Deployment             | 6–8 giờ      | ⭐⭐    | Nên có   |

**Tổng: 70–90 giờ để có kiến thức .NET toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Beginner — Người Mới (0–1 năm kinh nghiệm)

- [ ] Hiểu OOP trong C#
- [ ] Viết được API CRUD với ASP.NET Core
- [ ] Dùng EF Core cho database operations
- [ ] Xử lý ngoại lệ đúng cách
- [ ] Viết unit test cơ bản

**Thời gian đạt được:** 2–3 tháng học tập nghiêm túc

### Intermediate — Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Áp dụng SOLID và Design Patterns
- [ ] Dùng async/await không bị deadlock
- [ ] Cài JWT authentication từ đầu
- [ ] Tối ưu EF Core queries
- [ ] Viết integration tests

**Thời gian đạt được:** 2–3 tháng để nâng cấp

### Advanced — Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Thiết kế Clean Architecture toàn phần
- [ ] Implement CQRS với MediatR
- [ ] Tối ưu bộ nhớ với Span<T>/ArrayPool
- [ ] Thiết kế microservices system
- [ ] Giải quyết production incidents

**Thời gian đạt được:** Học liên tục, không có điểm dừng

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                          | Vị Trí                                                                                             |
| -------------------------------- | -------------------------------------------------------------------------------------------------- |
| Tổng quan nhanh                  | [README.md](README.md)                                                                             |
| SOLID Principles                 | [02-oop-patterns/solid-principles.md](02-oop-patterns/solid-principles.md)                         |
| Async/Await deep dive            | [03-async-concurrency/async-await-deep-dive.md](03-async-concurrency/async-await-deep-dive.md)     |
| DI lifetimes                     | [04-aspnet-core/dependency-injection.md](04-aspnet-core/dependency-injection.md)                   |
| N+1 query fix                    | [05-entity-framework/n-plus-one-problem.md](05-entity-framework/n-plus-one-problem.md)             |
| JWT setup                        | [08-security/jwt-authentication.md](08-security/jwt-authentication.md)                             |
| Clean Architecture               | [09-architecture/clean-architecture.md](09-architecture/clean-architecture.md)                     |
| Câu hỏi phỏng vấn                | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md)                       |

---

## 📈 Theo Dõi Tiến Độ Học

Tạo bản sao và tự theo dõi:

```markdown
## Tiến Độ Học C# .NET

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] C# type system (value vs reference)
- [ ] CLR và Garbage Collector
- [ ] LINQ cơ bản và nâng cao
- [ ] Collections và Generic
- [ ] Delegate, Event, Func, Action

### Giai Đoạn 2: Kỹ Năng Trung Cấp (Tuần 3–6)

- [ ] SOLID Principles (5/5)
- [ ] Design Patterns chính (ít nhất 10)
- [ ] async/await, CancellationToken
- [ ] ASP.NET Core Middleware, DI, Routing
- [ ] EF Core: migrations, relationships, N+1

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)

- [ ] JWT Authentication & Authorization Policies
- [ ] Unit Testing + Integration Testing
- [ ] Performance: Span<T>, caching, profiling
- [ ] Clean Architecture implementation

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] DDD: Aggregate, Domain Events
- [ ] CQRS + MediatR
- [ ] Microservices patterns
- [ ] Cloud deployment, CI/CD
- [ ] Mock interview (30 câu hỏi)
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn phải:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích GC hoạt động mà không cần ghi chú
- [ ] Viết async code đúng cách, tránh deadlock
- [ ] Thiết kế API với ASP.NET Core middleware đúng chuẩn
- [ ] Giải thích từng principle trong SOLID với ví dụ
- [ ] Cài EF Core từ đầu trong 30 phút

### ✅ Năng Lực Vận Hành

- [ ] Debug N+1 query trong EF Core
- [ ] Implement JWT Auth từ đầu
- [ ] Viết unit + integration tests cho API
- [ ] Tối ưu memory allocation khi cần
- [ ] Deploy .NET app lên Docker / Kubernetes

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 30 câu hỏi C# .NET phổ biến
- [ ] Kể được 2–3 câu chuyện thực chiến (STAR)
- [ ] Thiết kế hệ thống với kiến trúc rõ ràng
- [ ] Thảo luận được trade-off giữa các approach
- [ ] Làm live coding challenge trong 45 phút

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc `README.md` kỹ để hiểu toàn bộ lộ trình
2. Chọn learning path phù hợp cấp độ hiện tại
3. Bắt đầu `01-fundamentals/` nếu chưa chắc nền tảng
4. Cài .NET SDK và chuẩn bị môi trường dev

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/`
2. Đọc `02-oop-patterns/solid-principles.md` (quan trọng nhất)
3. Bắt đầu `03-async-concurrency/`
4. Viết code thực hành cho từng chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành ASP.NET Core và EF Core
2. Xây dựng một API project hoàn chỉnh
3. Cài authentication từ đầu
4. Viết tests cho project đó

### Dài Hạn (3 Tháng Tới)

1. Hoàn thành Clean Architecture một lần
2. Nắm vững một pattern nâng cao (DDD hoặc CQRS)
3. Deploy project lên cloud
4. Luyện phỏng vấn mock với người khác

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng cách làm:** Không chỉ đọc — hãy tự gõ lại mọi ví dụ code
2. **Đặt câu hỏi "tại sao":** Tại sao dùng `Scoped` thay `Singleton`? Tại sao `async void` nguy hiểm?
3. **Break and fix:** Cố tình gây deadlock, memory leak rồi tìm cách sửa
4. **Review code người khác:** Đọc open source .NET projects trên GitHub
5. **Viết blog hoặc giải thích cho người khác:** Dạy người khác là cách học tốt nhất
6. **Dùng dotnet watch:** Hot reload khi thực hành giúp tiết kiệm thời gian
7. **Đọc stack trace kỹ:** Mọi exception đều có câu chuyện để kể
8. **Benchmark trước khi tối ưu:** Đừng tối ưu mù, hãy đo đạc trước

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Mọi đóng góp đều được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm ví dụ code thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Thêm câu hỏi phỏng vấn từ kinh nghiệm thực tế
- [ ] Cập nhật nội dung theo phiên bản .NET mới nhất

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 3.1 (Toàn bộ 11 topics + GLOSSARY.md đã hoàn thành)
**Trạng Thái:** ✅ README.md | ✅ INDEX.md | ✅ GLOSSARY.md | ✅ 01-fundamentals (7 files) | ✅ 02-oop-patterns (6 files) | ✅ 03-async-concurrency (7 files) | ✅ 04-aspnet-core (9 files) | ✅ 05-entity-framework (9 files) | ✅ 06-testing (7 files) | ✅ 07-performance (7 files) | ✅ 08-security (9 files) | ✅ 09-architecture (8 files) | ✅ 10-cloud-deployment (7 files) | ✅ 11-interview-prep (7 files)
