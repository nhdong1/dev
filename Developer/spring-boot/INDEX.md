# Java Spring Boot — Chỉ Mục Đầy Đủ

> Chỉ mục toàn bộ nội dung của knowledge base Java Spring Boot

## 📁 Cấu Trúc Thư Mục

```
Developer/spring-boot/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Chỉ mục này
│
├── 01-fundamentals/                            Nền tảng Java & Spring
│   ├── README.md                               ✅ Tổng quan nền tảng
│   ├── 1-java-core.md                            ✅ Java 17+ — Records, Sealed, Streams, Virtual Threads
│   ├── 2-spring-core.md                          ✅ IoC, DI, ApplicationContext, BeanFactory
│   ├── 3-spring-boot-basics.md                   ✅ Auto-configuration, Starters, @SpringBootApplication
│   ├── 4-bean-lifecycle.md                       ✅ Bean lifecycle & scopes — Singleton, Prototype
│   └── 5-configuration.md                        ✅ @Value, @ConfigurationProperties, Profiles
│
├── 02-web-layer/                               Tầng Web — REST API
│   ├── README.md                               ✅ Tổng quan Web layer
│   ├── 1-rest-controllers.md                     ✅ @RestController, routing, HTTP methods
│   ├── 2-request-response.md                     ✅ DTO, Bean Validation, MapStruct, serialization
│   ├── 3-exception-handling.md                   ✅ @ControllerAdvice, ProblemDetail (RFC 7807)
│   ├── 4-filters-interceptors.md                 ✅ OncePerRequestFilter, HandlerInterceptor
│   └── 5-openapi-swagger.md                      ✅ springdoc-openapi, Swagger UI
│
├── 03-data-access/                             Truy Cập Dữ Liệu
│   ├── README.md                               ✅ Tổng quan data layer
│   ├── 1-spring-data-jpa.md                      ✅ JpaRepository, @Entity, JPQL, Criteria API
│   ├── 2-orm-mapping.md                          ✅ Relationships, @OneToMany, cascade types
│   ├── 3-transactions.md                         ✅ @Transactional, propagation, isolation levels
│   ├── 4-n-plus-one-problem.md                   ✅ Lazy/Eager loading, @EntityGraph, JOIN FETCH
│   ├── 5-spring-data-redis.md                    ✅ RedisTemplate, @Cacheable, RedisRepository
│   └── 6-mongodb-integration.md                  ✅ MongoRepository, aggregation pipeline
│
├── 04-security/                                Bảo Mật
│   ├── README.md                               ✅ Tổng quan bảo mật
│   ├── 1-spring-security-basics.md               ✅ SecurityFilterChain, AuthenticationManager
│   ├── 2-jwt-implementation.md                   ✅ JWT access/refresh tokens, JwtFilter
│   ├── 3-oauth2-oidc.md                          ✅ OAuth2 flows, Resource Server, OIDC
│   ├── 4-method-security.md                      ✅ @PreAuthorize, @PostAuthorize, @Secured
│   ├── 5-cors-csrf.md                            ✅ CORS configuration, CSRF protection
│   └── 6-security-best-practices.md              ✅ BCrypt, secrets management, security hardening
│
├── 05-async-messaging/                         Bất Đồng Bộ & Nhắn Tin
│   ├── README.md                               ✅ Tổng quan async & messaging
│   ├── 1-async-annotations.md                    ✅ @Async, CompletableFuture, ThreadPoolTaskExecutor
│   ├── 2-spring-events.md                        ✅ ApplicationEvent, @EventListener, @TransactionalEventListener
│   ├── 3-kafka-integration.md                    ✅ @KafkaListener, KafkaTemplate, consumer groups
│   ├── 4-rabbitmq-integration.md                 ✅ @RabbitListener, exchanges, queues, bindings
│   └── 5-scheduled-tasks.md                      ✅ @Scheduled, Quartz Scheduler, cron expressions
│
├── 06-testing/                                 Kiểm Thử
│   ├── README.md                               ✅ Tổng quan chiến lược kiểm thử
│   ├── 1-unit-testing.md                         ✅ JUnit 5, @Test, @ParameterizedTest, assertions
│   ├── 2-mocking.md                              ✅ Mockito, @Mock, @InjectMocks, ArgumentCaptor
│   ├── 3-integration-testing.md                  ✅ @SpringBootTest, Testcontainers, @DirtiesContext
│   ├── 4-web-layer-testing.md                    ✅ @WebMvcTest, MockMvc, @MockBean
│   ├── 5-data-layer-testing.md                   ✅ @DataJpaTest, in-memory H2, Flyway test
│   └── 6-test-coverage.md                        ✅ JaCoCo, mutation testing, coverage thresholds
│
├── 07-performance/                             Hiệu Năng
│   ├── README.md                               ✅ Tổng quan tối ưu hiệu năng
│   ├── 1-caching-strategies.md                   ✅ @Cacheable, @CacheEvict, Redis caching, cache patterns
│   ├── 2-connection-pooling.md                   ✅ HikariCP — maximumPoolSize, leakDetection, monitoring
│   ├── 3-jvm-tuning.md                           ✅ Heap sizing, G1GC vs ZGC, GC tuning flags, OOM handling
│   ├── 4-query-optimization.md                   ✅ EXPLAIN ANALYZE, indexes, batch operations, pagination
│   ├── 5-profiling.md                            ✅ async-profiler, JFR (Java Flight Recorder), flame graphs
│   └── 6-load-testing.md                         ✅ Gatling, k6 — load vs stress vs spike vs soak testing
│
├── 08-architecture/                            Kiến Trúc Ứng Dụng
│   ├── README.md                               ✅ Tổng quan kiến trúc
│   ├── 1-clean-architecture.md                   ✅ Hexagonal architecture, Ports & Adapters
│   ├── 2-layered-architecture.md                 ✅ Controller-Service-Repository pattern
│   ├── 3-microservices.md                        ✅ Service decomposition, domain boundaries, DDD
│   ├── 4-api-gateway.md                          ✅ Spring Cloud Gateway, rate limiting, routing
│   ├── 5-service-discovery.md                    ✅ Eureka Server & Client, Consul, load balancing
│   ├── 6-event-driven.md                         ✅ CQRS, Event Sourcing, Saga pattern
│   └── 7-circuit-breaker.md                      ✅ Resilience4j — CircuitBreaker, Retry, Bulkhead
│
├── 09-cloud-deployment/                        Triển Khai Đám Mây
│   ├── README.md                               ✅ Tổng quan deployment & production readiness checklist
│   ├── 1-docker-spring.md                        ✅ Multi-stage Dockerfile, Jib, layer caching
│   ├── 2-kubernetes-deployment.md                ✅ Deployment, Service, ConfigMap, HPA, Probes
│   ├── 3-spring-boot-actuator.md                 ✅ /health, /metrics, custom endpoints, info
│   ├── 4-observability.md                        ✅ Micrometer, Prometheus, Grafana, Zipkin, Jaeger
│   ├── 5-spring-cloud-config.md                  ✅ Config Server, Git backend, refresh scope
│   └── 6-cicd-pipeline.md                        ✅ GitHub Actions, Jenkins, Maven/Gradle CI
│
├── 10-advanced/                                Chủ Đề Nâng Cao
│   ├── README.md                               ✅ Tổng quan chủ đề nâng cao
│   ├── 1-reactive-programming.md                 ✅ Spring WebFlux, Mono, Flux, Project Reactor, Backpressure
│   ├── 2-spring-batch.md                         ✅ Job, Step, ItemReader/Processor/Writer, Partitioning
│   ├── 3-grpc-integration.md                     ✅ Protocol Buffers, gRPC server/client, Streaming
│   ├── 4-graphql.md                              ✅ Spring GraphQL, schema-first, DataLoader, N+1
│   └── 5-native-image.md                         ✅ GraalVM, AOT compilation, Native Build Tools, hints
│
├── 11-interview-prep/                          Chuẩn Bị Phỏng Vấn
│   ├── README.md                               ✅ Hướng dẫn tổng quan phỏng vấn
│   ├── INTERVIEW_GUIDE.md                        ✅ Top 50 câu hỏi & câu trả lời mẫu
│   ├── 1-common-questions.md                     ✅ Câu hỏi thường gặp theo chủ đề
│   ├── 2-system-design-scenarios.md              ✅ Bài toán thiết kế hệ thống
│   ├── 3-star-stories.md                         ✅ Template câu chuyện STAR
│   ├── 4-behavioral-questions.md                 ✅ Câu hỏi văn hóa & teamwork
│   └── 5-90-day-study-plan.md                    ✅ Kế hoạch học chi tiết 90 ngày
│
├── GLOSSARY.md                                 (Tạo) Bảng thuật ngữ kỹ thuật
└── RESOURCES.md                                (Tạo) Sách, blog, khóa học, công cụ
```

---

## ✅ Trạng Thái Tạo Nội Dung

| Chủ Đề                              | File / Thư Mục                        | Trạng Thái | Chất Lượng      |
| ----------------------------------- | ------------------------------------- | ---------- | --------------- |
| **Tổng Quan & Lộ Trình**            | README.md                             | ✅          | Toàn diện       |
| **Chỉ Mục Đầy Đủ**                  | INDEX.md                              | ✅          | Toàn diện       |
| **Nền Tảng Java & Spring**          | 01-fundamentals/ (6 files)            | ✅          | Hoàn thành      |
| **Tầng Web — REST API**             | 02-web-layer/ (5 files)               | ✅          | Hoàn thành      |
| **Truy Cập Dữ Liệu**                | 03-data-access/ (7 files)             | ✅          | Hoàn thành      |
| **Bảo Mật**                         | 04-security/ (6 files)                | ✅          | Hoàn thành      |
| **Bất Đồng Bộ & Nhắn Tin**          | 05-async-messaging/ (6 files)         | ✅          | Hoàn thành      |
| **Kiểm Thử**                        | 06-testing/ (7 files)                 | ✅          | Hoàn thành      |
| **Hiệu Năng**                       | 07-performance/ (7 files)             | ✅          | Hoàn thành      |
| **Kiến Trúc**                       | 08-architecture/ (8 files)            | ✅          | Hoàn thành      |
| **Triển Khai Đám Mây**              | 09-cloud-deployment/ (7 files)        | ✅          | Hoàn thành      |
| **Chủ Đề Nâng Cao**                 | 10-advanced/ (6 files)                | ✅          | Hoàn thành      |
| **Chuẩn Bị Phỏng Vấn**             | 11-interview-prep/ (7 files)          | ✅          | Hoàn thành      |

---

## 🎯 Thứ Tự Tạo Nội Dung (Theo Ưu Tiên)

### Ưu Tiên Cao — Nền Tảng Bắt Buộc

- [x] `01-fundamentals/README.md` — Spring Core, IoC/DI, Bean lifecycle ✅
- [x] `01-fundamentals/2-spring-core.md` — IoC Container chi tiết ✅
- [x] `02-web-layer/README.md` — REST API overview ✅
- [x] `02-web-layer/1-rest-controllers.md` — Controller patterns ✅
- [x] `02-web-layer/2-request-response.md` — DTO, Bean Validation, MapStruct ✅
- [x] `02-web-layer/3-exception-handling.md` — @ControllerAdvice, ProblemDetail ✅
- [x] `02-web-layer/4-filters-interceptors.md` — Filter vs Interceptor ✅
- [x] `02-web-layer/5-openapi-swagger.md` — springdoc-openapi, Swagger UI ✅
- [x] `03-data-access/README.md` — JPA & data access overview ✅
- [x] `03-data-access/1-spring-data-jpa.md` — JPA Repository chi tiết ✅
- [x] `03-data-access/3-transactions.md` — Transaction management ✅
- [x] `04-security/README.md` — Spring Security overview ✅
- [x] `04-security/2-jwt-implementation.md` — JWT từ A đến Z ✅

### Ưu Tiên Cao — Kỹ Năng Thiết Yếu

- [x] `06-testing/README.md` — Testing strategy overview ✅
- [x] `06-testing/1-unit-testing.md` — JUnit 5 & Mockito ✅
- [x] `06-testing/2-mocking.md` — Mockito, ArgumentCaptor ✅
- [x] `06-testing/3-integration-testing.md` — @SpringBootTest & Testcontainers ✅
- [x] `06-testing/4-web-layer-testing.md` — @WebMvcTest, MockMvc ✅
- [x] `06-testing/5-data-layer-testing.md` — @DataJpaTest, H2 ✅
- [x] `06-testing/6-test-coverage.md` — JaCoCo, mutation testing ✅
- [x] `11-interview-prep/INTERVIEW_GUIDE.md` — Top 50 Q&A ✅
- [x] `03-data-access/4-n-plus-one-problem.md` — N+1 diagnosis & solutions ✅

### Ưu Tiên Trung Bình — Phát Triển Chuyên Sâu

- [x] `05-async-messaging/` — @Async, Events, Kafka, RabbitMQ, Scheduled ✅
- [x] `07-performance/` — Caching, HikariCP, JVM, Query Opt, Profiling, Load Testing ✅
- [x] `08-architecture/` — Clean Architecture, Layered, Microservices, API Gateway, Service Discovery, CQRS, Event Sourcing, Circuit Breaker ✅
- [x] `09-cloud-deployment/1-docker-spring.md` — Docker hóa Spring Boot ✅
- [x] `09-cloud-deployment/2-kubernetes-deployment.md` — K8s deployment ✅
- [x] `09-cloud-deployment/3-spring-boot-actuator.md` — Actuator health & metrics ✅
- [x] `09-cloud-deployment/4-observability.md` — Micrometer, Prometheus, Grafana, Zipkin ✅
- [x] `09-cloud-deployment/5-spring-cloud-config.md` — Config Server, RefreshScope ✅
- [x] `09-cloud-deployment/6-cicd-pipeline.md` — GitHub Actions, Jenkins CI ✅

### Ưu Tiên Thấp — Nâng Cao & Tham Khảo

- [x] `10-advanced/1-reactive-programming.md` — Spring WebFlux ✅
- [x] `10-advanced/2-spring-batch.md` — Spring Batch ✅
- [x] `10-advanced/3-grpc-integration.md` — gRPC với Spring ✅
- [x] `10-advanced/4-graphql.md` — Spring GraphQL ✅
- [x] `10-advanced/5-native-image.md` — GraalVM Native ✅
- [ ] `GLOSSARY.md` — Bảng thuật ngữ
- [ ] `RESOURCES.md` — Tài liệu tham khảo

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình Học (Mới bắt đầu / Trung cấp / Nâng cao)
3. Học tuần tự từng section
4. Làm bài tập thực hành (xây dựng lab environment)
5. Xây dựng project portfolio
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 11-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào 03-data-access/ (JPA, Transaction — hỏi nhiều nhất)
3. Ôn tập 04-security/ (Spring Security, JWT — bắt buộc)
4. Chuẩn bị câu chuyện STAR từ kinh nghiệm thực tế
5. Mock interview với đồng nghiệp hoặc mentor
```

### Dùng Trong Dự Án Thực Tế

```
Dùng làm tài liệu tham khảo:
- Lỗi JPA/N+1: Xem 03-data-access/n-plus-one-problem.md
- Security issues: Xem 04-security/
- Performance: Xem 07-performance/
- Deploy: Xem 09-cloud-deployment/
- Architecture decisions: Xem 08-architecture/
```

### Thiết Kế Hệ Thống

```
1. Đọc 08-architecture/README.md để chọn kiến trúc phù hợp
2. Tham khảo 08-architecture/microservices.md cho microservices
3. Dùng 09-cloud-deployment/ cho production deployment
4. Xem 07-performance/ để estimate capacity
```

---

## 📊 Ước Tính Thời Gian Học

| Module                       | Thời Gian    | Độ Khó   | Ưu Tiên       |
| ---------------------------- | ------------ | --------- | ------------- |
| Fundamentals                 | 4–6 giờ      | ⭐        | Bắt buộc      |
| REST API (Web Layer)         | 6–8 giờ      | ⭐⭐      | Bắt buộc      |
| Data Access (JPA)            | 8–10 giờ     | ⭐⭐      | Bắt buộc      |
| Security (Spring Security)   | 8–10 giờ     | ⭐⭐⭐    | Bắt buộc      |
| Async & Messaging            | 6–8 giờ      | ⭐⭐      | Nên học       |
| Testing                      | 6–8 giờ      | ⭐⭐      | Bắt buộc      |
| Performance & Tuning         | 8–10 giờ     | ⭐⭐⭐    | Nên học       |
| Architecture & Design        | 10–12 giờ    | ⭐⭐⭐    | Nên học       |
| Cloud & Deployment           | 8–10 giờ     | ⭐⭐      | Nên học       |
| Advanced Topics              | 15–20 giờ    | ⭐⭐⭐    | Tùy chọn      |

**Tổng: 80–110 giờ để nắm vững Spring Boot toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Được Hỗ Trợ

### Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Hiểu IoC và DI
- [ ] Tạo REST API với CRUD cơ bản
- [ ] Dùng Spring Data JPA
- [ ] Viết Unit Tests
- [ ] Cấu hình application.properties

**Thời gian để thành thạo:** 2–3 tháng

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Spring Security + JWT hoàn chỉnh
- [ ] Tối ưu JPA — giải quyết N+1
- [ ] Integration Testing với Testcontainers
- [ ] Caching với Redis
- [ ] Async programming với @Async, Kafka
- [ ] Deploy với Docker

**Thời gian để nâng lên:** 2–3 tháng

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Microservices với Spring Cloud
- [ ] Reactive với Spring WebFlux
- [ ] JVM tuning & profiling
- [ ] Event-driven architecture (CQRS, Event Sourcing)
- [ ] CI/CD pipeline đầy đủ
- [ ] GraalVM Native Image

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu                          | Vị Trí                                                                    |
| --------------------------------- | ------------------------------------------------------------------------- |
| Tổng quan nhanh                   | [README.md](README.md)                                                    |
| Spring Core & DI                  | [01-fundamentals/spring-core.md](01-fundamentals/spring-core.md)          |
| REST API thiết kế                 | [02-web-layer/README.md](02-web-layer/README.md)                          |
| JPA & Transaction                 | [03-data-access/README.md](03-data-access/README.md)                      |
| N+1 Problem                       | [03-data-access/n-plus-one-problem.md](03-data-access/n-plus-one-problem.md) |
| Spring Security + JWT             | [04-security/README.md](04-security/README.md)                            |
| Testing Strategy                  | [06-testing/README.md](06-testing/README.md)                              |
| Performance Tuning                | [07-performance/README.md](07-performance/README.md)                      |
| Microservices Architecture        | [08-architecture/microservices.md](08-architecture/microservices.md)       |
| Docker & K8s Deploy               | [09-cloud-deployment/README.md](09-cloud-deployment/README.md)            |
| Interview Questions               | [11-interview-prep/INTERVIEW_GUIDE.md](11-interview-prep/INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học

Sao chép và điền vào để theo dõi:

```markdown
## Tiến Độ Học Spring Boot

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] Spring IoC & DI — hiểu ApplicationContext
- [ ] Bean Lifecycle & Scopes
- [ ] Auto-configuration hoạt động thế nào
- [ ] Configuration Properties & Profiles
- [ ] Java 17+ features — Records, Sealed Classes

### Giai Đoạn 2: Phát Triển API (Tuần 3–6)

- [ ] RESTful API với đầy đủ HTTP methods
- [ ] DTO & Bean Validation
- [ ] Global Exception Handling
- [ ] Spring Data JPA — CRUD, JPQL
- [ ] @Transactional — propagation & isolation
- [ ] Spring Security + JWT end-to-end

### Giai Đoạn 3: Nâng Cao (Tuần 7–10)

- [ ] N+1 Problem — diagnosis & fix
- [ ] Caching với Spring Cache + Redis
- [ ] Integration Testing với Testcontainers
- [ ] @Async & Spring Events
- [ ] Docker + Kubernetes deployment
- [ ] Spring Boot Actuator + Prometheus

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Microservices với Spring Cloud
- [ ] Kafka integration end-to-end
- [ ] JVM profiling & GC tuning
- [ ] Spring WebFlux reactive programming
- [ ] System design với Spring Boot
- [ ] Mock interviews & STAR stories
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Cốt Lõi

- [ ] Giải thích IoC/DI không cần nhìn tài liệu
- [ ] Thiết kế REST API chuẩn với error handling đầy đủ
- [ ] Implement Spring Security + JWT từ đầu
- [ ] Debug và tối ưu N+1 problem trong JPA
- [ ] Viết test coverage ≥ 80% (unit + integration)

### ✅ Năng Lực Vận Hành

- [ ] Docker hóa Spring Boot application
- [ ] Deploy lên Kubernetes với health checks
- [ ] Thiết lập monitoring với Micrometer + Prometheus
- [ ] Debug memory leak và performance issues
- [ ] Tích hợp Kafka cho event-driven flows

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 50 câu hỏi Spring Boot
- [ ] Kể 2–3 câu chuyện STAR về kỹ thuật
- [ ] Thiết kế hệ thống có xem xét scalability
- [ ] Thảo luận trade-offs giữa các approach
- [ ] Live coding — viết API + test trong 30 phút

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp với level hiện tại
3. Tạo project Spring Boot từ [start.spring.io](https://start.spring.io)
4. Cài đặt Docker Desktop và dựng lab environment

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành `01-fundamentals/` — nắm vững IoC/DI
2. Xây dựng REST API CRUD đầy đủ
3. Tích hợp Spring Data JPA với PostgreSQL thực
4. Viết unit tests với Mockito cho service layer

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành tất cả core modules (01–06)
2. Implement Spring Security + JWT hoàn chỉnh
3. Giải quyết ít nhất 1 N+1 problem thực tế
4. Deploy application lên Docker

### Dài Hạn (3 Tháng Tới)

1. Thành thạo một kiến trúc pattern (Clean Architecture / Hexagonal)
2. Xây dựng project microservices demo
3. Đạt code coverage ≥ 80% trong project
4. Bắt đầu apply hoặc nhận Spring Boot features trong dự án thực

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng cách làm:** Đừng chỉ đọc — hãy code ngay sau mỗi concept
2. **Debug intentionally:** Cố ý gây lỗi để hiểu cơ chế xử lý lỗi
3. **Đọc stack trace:** Đừng bỏ qua — stack trace là bản đồ debug
4. **Viết tests trước:** TDD (Test-Driven Development) giúp hiểu requirements rõ hơn
5. **Dùng profiler:** `async-profiler` hoặc IntelliJ Profiler — đừng đoán performance
6. **Review code thực tế:** Đọc source code của Spring Boot để hiểu cơ chế
7. **Ghi chép incident:** Mỗi lần fix bug là cơ hội học — document lại
8. **Chia sẻ kiến thức:** Giải thích cho người khác = hiểu sâu hơn

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống. Đóng góp được hoan nghênh:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Bổ sung ví dụ thực tế từ kinh nghiệm dự án
- [ ] Viết thêm sections còn thiếu
- [ ] Giải thích rõ hơn các concept phức tạp
- [ ] Thêm code samples và anti-patterns

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 3.0 (11-interview-prep hoàn thành — Knowledge base đầy đủ)
**Trạng Thái:** ✅ README hoàn thành | ✅ INDEX hoàn thành | ✅ 01-fundamentals hoàn thành | ✅ 02-web-layer hoàn thành | ✅ 03-data-access hoàn thành | ✅ 04-security hoàn thành | ✅ 05-async-messaging hoàn thành | ✅ 06-testing hoàn thành | ✅ 07-performance hoàn thành | ✅ 08-architecture hoàn thành | ✅ 09-cloud-deployment hoàn thành | ✅ 10-advanced hoàn thành | ✅ 11-interview-prep hoàn thành
