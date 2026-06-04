# 📅 Kế Hoạch Học 90 Ngày — Java Spring Boot

> Lộ trình học có cấu trúc từ nền tảng đến sẵn sàng phỏng vấn cấp độ Senior Backend Developer. Mỗi tuần có mục tiêu rõ ràng, bài tập thực hành, và checkpoint (điểm kiểm tra).

---

## 🎯 Điều Kiện Tiên Quyết

Trước khi bắt đầu lộ trình này, bạn nên biết:
- Java cơ bản (OOP, Collections, Exception Handling)
- SQL cơ bản (SELECT, JOIN, GROUP BY)
- Git cơ bản
- Biết tạo project từ [start.spring.io](https://start.spring.io)

**Cam Kết Thời Gian:** 2–3 giờ/ngày, 5–6 ngày/tuần

---

## 🗓️ Tổng Quan 90 Ngày

| Giai Đoạn | Tuần | Nội Dung | Deliverable (Sản Phẩm) |
| --------- | ---- | -------- | ----------------------- |
| **Giai Đoạn 1: Nền Tảng** | 1–3 | Spring Core, JPA, Web Layer | REST API CRUD đầy đủ |
| **Giai Đoạn 2: Bảo Mật & Testing** | 4–6 | Spring Security, JWT, Testing | API có authentication |
| **Giai Đoạn 3: Hiệu Năng & Async** | 7–9 | Performance, Caching, Messaging | Production-ready features |
| **Giai Đoạn 4: Kiến Trúc & Cloud** | 10–11 | Microservices, Docker, K8s | Deployed application |
| **Giai Đoạn 5: Phỏng Vấn Prep** | 12–13 | Mock interviews, System Design | Interview-ready |

---

## 📗 Giai Đoạn 1: Nền Tảng (Tuần 1–3)

### Tuần 1: Spring Core & IoC/DI

**Mục tiêu:** Hiểu sâu cơ chế IoC Container và Dependency Injection

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu | Thời Gian |
|------|--------|----------|-----------|
| Thứ 2 | IoC Container — ApplicationContext, BeanFactory | `01-fundamentals/2-spring-core.md` | 2h |
| Thứ 3 | Bean Lifecycle & Scopes (@PostConstruct, @PreDestroy) | `01-fundamentals/4-bean-lifecycle.md` | 2h |
| Thứ 4 | Auto-configuration mechanism & Starters | `01-fundamentals/3-spring-boot-basics.md` | 2h |
| Thứ 5 | Configuration: @Value, @ConfigurationProperties, Profiles | `01-fundamentals/5-configuration.md` | 2h |
| Thứ 6 | Java 17+ features: Records, Sealed Classes, Pattern Matching | `01-fundamentals/1-java-core.md` | 2h |
| Thứ 7 | **LAB:** Tạo Spring Boot project, experiment với Beans và Profiles | Tự thực hành | 3h |

**Bài Tập Thực Hành:**

```java
// Task 1: Tạo BeanFactory experiment
// - Tạo 3 beans: DatabaseConfig, CacheConfig, ServiceConfig
// - Dùng @Profile để load khác nhau cho dev/prod
// - Verify bằng ApplicationContext.getBeanDefinitionNames()

// Task 2: @ConfigurationProperties
// - Tạo AppConfig.java record với nested objects
// - Cấu hình trong application-dev.yml và application-prod.yml
// - Validate với @Validated và @NotBlank

// Task 3: Custom BeanPostProcessor
// - Implement BeanPostProcessor để log tất cả beans khi startup
```

**Checkpoint Tuần 1:**
- [ ] Giải thích IoC/DI mà không nhìn notes (2 phút)
- [ ] Vẽ Bean Lifecycle diagram từ memory
- [ ] Demo @Profile switching giữa dev và prod

---

### Tuần 2: REST API & Web Layer

**Mục tiêu:** Build REST API đầy đủ theo chuẩn với validation và error handling

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | @RestController, routing, HTTP methods best practices | `02-web-layer/1-rest-controllers.md` |
| Thứ 3 | DTO (Data Transfer Object) pattern, MapStruct, Bean Validation | `02-web-layer/2-request-response.md` |
| Thứ 4 | Global Exception Handling — @ControllerAdvice, ProblemDetail RFC 7807 | `02-web-layer/3-exception-handling.md` |
| Thứ 5 | Filter vs Interceptor, OncePerRequestFilter, HandlerInterceptor | `02-web-layer/4-filters-interceptors.md` |
| Thứ 6 | OpenAPI / Swagger với springdoc-openapi | `02-web-layer/5-openapi-swagger.md` |
| Thứ 7 | **LAB:** Xây dựng Product API đầy đủ | Tự thực hành |

**Project Lab Tuần 2:**

```
Xây dựng Product Management API:

Endpoints:
  POST   /api/products          — tạo sản phẩm mới
  GET    /api/products          — danh sách (có pagination)
  GET    /api/products/{id}     — chi tiết
  PUT    /api/products/{id}     — cập nhật
  DELETE /api/products/{id}     — xóa mềm (soft delete)

Requirements:
  - DTO cho Request và Response (khác nhau)
  - Validation: @NotBlank name, @Positive price, @Min(0) stock
  - Custom exception: ProductNotFoundException → 404
  - @ControllerAdvice trả về ProblemDetail format
  - Logging filter log request time
  - Swagger UI hiển thị đầy đủ
```

**Checkpoint Tuần 2:**
- [ ] Demo Product API trên Swagger UI
- [ ] Error response đúng format ProblemDetail
- [ ] Validation messages rõ ràng

---

### Tuần 3: Spring Data JPA

**Mục tiêu:** Thành thạo JPA — từ mapping đến N+1 problem và transaction

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | JpaRepository, @Entity, JPQL, Criteria API | `03-data-access/1-spring-data-jpa.md` |
| Thứ 3 | ORM Mapping: @OneToMany, @ManyToMany, Cascade | `03-data-access/2-orm-mapping.md` |
| Thứ 4 | @Transactional — Propagation, Isolation Levels | `03-data-access/3-transactions.md` |
| Thứ 5 | N+1 Problem — phát hiện và giải quyết | `03-data-access/4-n-plus-one-problem.md` |
| Thứ 6 | Auditing, Soft Delete, Pagination với Pageable | Thực hành |
| Thứ 7 | **LAB:** Tích hợp JPA vào Product API | Tự thực hành |

**Project Lab Tuần 3:**

```
Mở rộng Product API với Database thật (PostgreSQL):

Schema:
  - Product (id, name, price, stock, category_id, created_at)
  - Category (id, name, description)
  - Tag (id, name) — many-to-many với Product

Yêu cầu kỹ thuật:
  - Flyway (công cụ migration database) cho schema migrations
  - JPQL query: "top 10 products by price in category"
  - @EntityGraph để giải quyết N+1 khi load products với tags
  - Soft delete: deleted_at column, filter trong @Query
  - Pagination: GET /api/products?page=0&size=20&sort=price,desc
  - @Transactional trên service methods, rollback test
```

**Checkpoint Tuần 3:**
- [ ] Giải thích N+1 problem và demo fix với SQL logs
- [ ] Mô tả 3 Transaction Propagation types với ví dụ
- [ ] Product API chạy với PostgreSQL thật

---

## 📘 Giai Đoạn 2: Bảo Mật & Testing (Tuần 4–6)

### Tuần 4: Spring Security & JWT

**Mục tiêu:** Implement end-to-end authentication và authorization

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | Security Filter Chain, UserDetailsService | `04-security/1-spring-security-basics.md` |
| Thứ 3 | JWT Implementation — sign, verify, JwtFilter | `04-security/2-jwt-implementation.md` |
| Thứ 4 | Access Token + Refresh Token rotation | `04-security/2-jwt-implementation.md` |
| Thứ 5 | OAuth2, Resource Server | `04-security/3-oauth2-oidc.md` |
| Thứ 6 | CORS, CSRF, @PreAuthorize | `04-security/5-cors-csrf.md`, `04-security/4-method-security.md` |
| Thứ 7 | **LAB:** Bảo vệ Product API với JWT | Tự thực hành |

**Project Lab Tuần 4:**

```
Thêm Authentication/Authorization vào Product API:

Endpoints mới:
  POST /api/auth/register    — đăng ký user mới
  POST /api/auth/login       — nhận access + refresh tokens
  POST /api/auth/refresh     — refresh token rotation
  POST /api/auth/logout      — revoke tokens

Authorization rules:
  - GET endpoints: PUBLIC
  - POST/PUT/DELETE: cần ROLE_ADMIN hoặc ROLE_MANAGER
  - Chỉ admin xóa được product
  - Manager chỉ update product của mình (@PostAuthorize)
```

**Checkpoint Tuần 4:**
- [ ] Demo full login flow với JWT trên Postman
- [ ] Vẽ Security Filter Chain từ memory
- [ ] 401 vs 403 response đúng scenario

---

### Tuần 5: Testing Strategy

**Mục tiêu:** Viết test suite đầy đủ với coverage ≥ 80%

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | JUnit 5 — @Test, @ParameterizedTest, @ExtendWith | `06-testing/1-unit-testing.md` |
| Thứ 3 | Mockito — @Mock, @InjectMocks, ArgumentCaptor | `06-testing/2-mocking.md` |
| Thứ 4 | @WebMvcTest, MockMvc — test controllers | `06-testing/4-web-layer-testing.md` |
| Thứ 5 | @DataJpaTest — test repositories | `06-testing/5-data-layer-testing.md` |
| Thứ 6 | @SpringBootTest + Testcontainers | `06-testing/3-integration-testing.md` |
| Thứ 7 | JaCoCo coverage report, coverage thresholds | `06-testing/6-test-coverage.md` |

**Bài Tập Testing:**

```java
// Tuần này viết tests cho Product API đã build:

// 1. Unit tests (ProductServiceTest):
//    - createProduct: success, duplicate name, invalid price
//    - deleteProduct: success, not found, unauthorized
//    - Dùng @Mock và ArgumentCaptor

// 2. Controller tests (ProductControllerTest @WebMvcTest):
//    - POST /api/products: valid request → 201, invalid → 400
//    - DELETE /api/products/{id}: unauthorized → 403

// 3. Repository tests (@DataJpaTest):
//    - findByCategoryId: returns correct products
//    - Custom query test với H2

// 4. Integration tests (@SpringBootTest + Testcontainers PostgreSQL):
//    - Full create → read → delete flow
//    - Authentication flow end-to-end

// Target: coverage ≥ 80% trên service layer
```

**Checkpoint Tuần 5:**
- [ ] JaCoCo report hiển thị ≥ 80% line coverage
- [ ] Phân biệt @Mock vs @MockBean được không?
- [ ] Integration test chạy với PostgreSQL container thật

---

### Tuần 6: Review & Project Consolidation (Hợp Nhất Dự Án)

**Mục tiêu:** Hoàn thiện project, viết thêm tests, bắt đầu chuẩn bị interview câu hỏi core

**Tasks:**
- [ ] Review và fix tất cả TODO/FIXME trong code
- [ ] Đảm bảo Swagger UI đầy đủ documentation
- [ ] Thêm logging structured với SLF4J + MDC (Mapped Diagnostic Context — Ngữ Cảnh Chẩn Đoán Ánh Xạ)
- [ ] Viết README.md cho project
- [ ] Ôn top 20 câu hỏi từ `INTERVIEW_GUIDE.md`
- [ ] Tập trả lời câu hỏi nói to — không chỉ đọc

---

## 📙 Giai Đoạn 3: Hiệu Năng & Async (Tuần 7–9)

### Tuần 7: Caching & Performance

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | Spring Cache — @Cacheable, @CacheEvict, cache strategies | `07-performance/1-caching-strategies.md` |
| Thứ 3 | Redis integration — RedisTemplate, TTL, eviction policies | `03-data-access/5-spring-data-redis.md` |
| Thứ 4 | HikariCP tuning — pool sizing, leak detection | `07-performance/2-connection-pooling.md` |
| Thứ 5 | Query optimization — EXPLAIN ANALYZE, indexes, batch | `07-performance/4-query-optimization.md` |
| Thứ 6 | JVM tuning basics — heap sizing, G1GC vs ZGC | `07-performance/3-jvm-tuning.md` |
| Thứ 7 | **LAB:** Thêm Redis caching vào Product API | Tự thực hành |

**Lab Tuần 7:**

```yaml
# docker-compose.yml — thêm Redis
services:
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

```java
// Task: Implement multi-layer caching cho Product API
// - @Cacheable("products") cho findById
// - Redis cache với TTL 10 phút
// - @CacheEvict khi update/delete
// - Cache hit/miss metrics với Micrometer
// - Benchmark: có cache vs không cache với k6
```

---

### Tuần 8: Async Programming & Messaging

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | @Async, CompletableFuture, ThreadPoolTaskExecutor | `05-async-messaging/1-async-annotations.md` |
| Thứ 3 | Spring Events, @EventListener, @TransactionalEventListener | `05-async-messaging/2-spring-events.md` |
| Thứ 4 | Apache Kafka — producer, consumer, @KafkaListener | `05-async-messaging/3-kafka-integration.md` |
| Thứ 5 | RabbitMQ — exchanges, queues, @RabbitListener | `05-async-messaging/4-rabbitmq-integration.md` |
| Thứ 6 | @Scheduled, Quartz Scheduler | `05-async-messaging/5-scheduled-tasks.md` |
| Thứ 7 | **LAB:** Product inventory event với Kafka | Tự thực hành |

**Lab Tuần 8:**

```java
// Scenario: Khi order được tạo, publish event "order.created"
// Inventory Service consume event → reserve stock
// Notification Service consume event → gửi email xác nhận (async)

// Implement:
// 1. OrderCreatedEvent → Kafka topic "orders"
// 2. InventoryConsumer @KafkaListener với idempotency check (Redis)
// 3. EmailNotificationListener @Async + @TransactionalEventListener
// 4. Dead Letter Topic cho failed messages
```

---

### Tuần 9: Load Testing & Profiling

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | Load Testing với k6 — load vs stress vs spike | `07-performance/6-load-testing.md` |
| Thứ 3 | Profiling với async-profiler, JFR (Java Flight Recorder) | `07-performance/5-profiling.md` |
| Thứ 4 | Spring Boot Actuator — /health, /metrics, custom endpoints | `09-cloud-deployment/3-spring-boot-actuator.md` |
| Thứ 5 | Micrometer + Prometheus + Grafana stack | `09-cloud-deployment/4-observability.md` |
| Thứ 6 | Distributed Tracing — Micrometer Tracing, Zipkin | `09-cloud-deployment/4-observability.md` |
| Thứ 7 | **LAB:** Setup monitoring stack với Docker Compose | Tự thực hành |

```yaml
# docker-compose.yml — monitoring stack
services:
  prometheus:
    image: prom/prometheus:latest
  grafana:
    image: grafana/grafana:latest
  zipkin:
    image: openzipkin/zipkin:latest
```

---

## 📕 Giai Đoạn 4: Kiến Trúc & Cloud (Tuần 10–11)

### Tuần 10: Microservices Architecture

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | Clean Architecture & Hexagonal — Ports & Adapters | `08-architecture/1-clean-architecture.md` |
| Thứ 3 | Microservices decomposition, DDD Bounded Contexts | `08-architecture/3-microservices.md` |
| Thứ 4 | Spring Cloud Gateway — routing, rate limiting | `08-architecture/4-api-gateway.md` |
| Thứ 5 | Circuit Breaker với Resilience4j | `08-architecture/7-circuit-breaker.md` |
| Thứ 6 | CQRS, Event Sourcing, Saga Pattern | `08-architecture/6-event-driven.md` |
| Thứ 7 | **LAB:** Tách API Gateway ra khỏi main service | Tự thực hành |

---

### Tuần 11: Docker & Kubernetes Deployment

**Nội dung học:**

| Ngày | Chủ Đề | Tài Liệu |
|------|--------|----------|
| Thứ 2 | Dockerfile cho Spring Boot — multi-stage build, Jib | `09-cloud-deployment/1-docker-spring.md` |
| Thứ 3 | Kubernetes — Deployment, Service, ConfigMap, Secret | `09-cloud-deployment/2-kubernetes-deployment.md` |
| Thứ 4 | HPA (Horizontal Pod Autoscaler), Resource Limits, Probes | `09-cloud-deployment/2-kubernetes-deployment.md` |
| Thứ 5 | Spring Cloud Config Server | `09-cloud-deployment/5-spring-cloud-config.md` |
| Thứ 6 | CI/CD Pipeline với GitHub Actions | `09-cloud-deployment/6-cicd-pipeline.md` |
| Thứ 7 | **LAB:** Deploy Product API lên local Kubernetes (minikube) | Tự thực hành |

**Final Project Deliverables (Sản Phẩm Cuối):**

```
Sau Tuần 11, bạn có:

├── product-api/              Spring Boot application
│   ├── Dockerfile            multi-stage build
│   ├── src/
│   │   ├── domain/           Clean Architecture layers
│   │   ├── application/
│   │   └── infrastructure/
│   └── src/test/
│       ├── unit/             coverage ≥ 80%
│       └── integration/      Testcontainers
│
├── k8s/
│   ├── deployment.yaml       2 replicas
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml              scale khi CPU > 70%
│   └── ingress.yaml
│
├── docker-compose.yml        local dev stack
│                             (PostgreSQL + Redis + Kafka + Prometheus + Grafana)
│
└── .github/workflows/
    └── ci-cd.yml             build + test + push image
```

---

## 📒 Giai Đoạn 5: Phỏng Vấn Prep (Tuần 12–13)

### Tuần 12: Technical Interview Preparation

**Ngày 1–2: Ôn Tập Core Topics**

```
Sáng (1h): Đọc 10 câu từ INTERVIEW_GUIDE.md
Chiều (2h): Code example cho mỗi concept
Tối (30m): Trả lời to voice — ghi hình nếu có thể
```

**Ngày 3–4: Live Coding Practice**

Luyện các bài coding phổ biến trong phỏng vấn:

```java
// Bài 1: Viết REST API Product từ đầu trong 30 phút
//   - CRUD endpoints
//   - @ControllerAdvice exception handler
//   - Service + Repository layers

// Bài 2: Implement JWT Authentication
//   - SecurityFilterChain config
//   - JwtService (sign, verify)
//   - JwtAuthenticationFilter

// Bài 3: Fix N+1 Problem
//   - Bắt đầu với code có N+1
//   - Identify bằng SQL logs
//   - Fix với @EntityGraph hoặc JOIN FETCH

// Bài 4: Viết Unit Tests
//   - ProductServiceTest với Mockito
//   - ProductControllerTest với @WebMvcTest
```

**Ngày 5–6: System Design Practice**

Luyện 2 system design scenarios từ `2-system-design-scenarios.md`:
- Buổi sáng: Vẽ architecture tự mình (45 phút)
- Buổi chiều: Compare với hướng dẫn, identify gaps

**Ngày 7: Mock Interview**

```
Tìm 1 người (đồng nghiệp, bạn học, mentor) để mock interview:
- 30 phút technical questions
- 30 phút system design
- 15 phút behavioral questions
- 15 phút feedback
```

---

### Tuần 13: Final Preparation & Interview

**Ngày 1–3: Review Yếu Điểm**

Từ mock interview, identify 3 areas yếu nhất và dành 2h mỗi ngày để deep dive.

**Ngày 4: Company Research**

```
1h: Đọc kỹ tech blog / engineering blog của công ty
1h: Nghiên cứu tech stack họ dùng (job description, LinkedIn, StackShare)
30m: Chuẩn bị 5–7 câu hỏi hỏi lại interviewer
```

**Ngày 5: Light Review + Rest**

```
Sáng (1h): Đọc lại INTERVIEW_GUIDE.md một lượt
Chiều (1h): Xem lại 5 câu chuyện STAR của mình
Tối: NGHỈ NGƠI — đừng học gì mới
```

**Ngày 6–7: Interview**

---

## 📊 Tracking Dashboard (Bảng Theo Dõi Tiến Độ)

Copy và dùng để track:

```markdown
## Sprint Log — Spring Boot 90-Day Study

### Tuần: ___
**Mục tiêu tuần này:**
- [ ] ...
- [ ] ...

**Đã hoàn thành:**
- [ ] ...

**Vấn đề gặp phải:**
- ...

**Sẽ làm khác tuần sau:**
- ...

**Câu hỏi chưa hiểu rõ:**
- ...
```

---

## 💡 Tips Học Hiệu Quả

### Kỹ Thuật Feynman (Kỹ Thuật Giải Thích)

Sau mỗi concept, giải thích nó như đang dạy cho người mới học:

```
1. Viết tên concept ra giấy
2. Giải thích bằng ngôn ngữ đơn giản, không jargon (thuật ngữ kỹ thuật)
3. Khi bị kẹt → đó là gap trong hiểu biết → đọc lại
4. Đơn giản hóa hơn nữa

Ví dụ: "@Transactional hoạt động thế nào?"
→ "Nó như một... bong bóng bảo vệ xung quanh code của bạn.
   Nếu code bên trong thành công hết → lưu tất cả vào database.
   Nếu có lỗi → xóa hết như chưa có gì xảy ra."
```

### Active Recall (Nhớ Lại Tích Cực)

```
Thay vì đọc lại notes:
1. Đóng notes lại
2. Viết ra tất cả bạn nhớ về topic
3. Mở notes → tìm gaps
4. Focus học vào gaps

Này hiệu quả hơn re-reading 5x
```

### Spaced Repetition (Lặp Lại Giãn Cách)

```
Ngày 1:  Học concept mới
Ngày 2:  Review nhanh (15 phút)
Ngày 7:  Review lần 2
Ngày 21: Review lần 3
Ngày 60: Review lần 4

Dùng Anki hoặc Notion để schedule
```

### Learning in Public (Học Công Khai)

```
Viết 1 LinkedIn post hoặc blog post về concept bạn vừa học:
- Buộc bạn phải articulate rõ ràng
- Nhận feedback từ cộng đồng
- Build personal brand
- 1 post/tuần là đủ
```

---

## ✅ Milestone Checklist

Đánh dấu khi đạt được từng milestone:

### Tuần 3 — End of Foundation Stage

- [ ] Giải thích IoC/DI mà không nhìn notes
- [ ] Demo N+1 Problem và 3 cách fix
- [ ] Product API CRUD chạy với PostgreSQL thật
- [ ] Mô tả Transaction Propagation với ví dụ

### Tuần 6 — End of Security & Testing Stage

- [ ] Demo JWT login flow end-to-end
- [ ] Vẽ Security Filter Chain từ memory
- [ ] JaCoCo report ≥ 80% coverage
- [ ] Integration test với Testcontainers chạy được

### Tuần 9 — End of Performance Stage

- [ ] Setup Prometheus + Grafana dashboard
- [ ] Benchmark API có cache vs không có cache
- [ ] Profile application và identify top hotspot
- [ ] Idempotent Kafka consumer hoạt động

### Tuần 11 — End of Architecture Stage

- [ ] Deploy application lên Kubernetes
- [ ] HPA tự động scale khi tăng load
- [ ] CI/CD pipeline chạy tự động khi push code
- [ ] Application observable với traces + metrics + logs

### Tuần 13 — Interview Ready

- [ ] Trả lời tự tin 40/50 câu từ INTERVIEW_GUIDE.md
- [ ] Hoàn thành 1 System Design interview mock
- [ ] Chuẩn bị 5 câu chuyện STAR với số liệu cụ thể
- [ ] Ghi hình mock interview và review

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
