# ☕ Java Spring Boot — Lộ Trình Học Toàn Diện

> Hướng dẫn toàn diện về Java Spring Boot, từ nền tảng cốt lõi đến kiến trúc microservices sản xuất thực tế — dành cho lập trình viên Backend muốn nắm vững framework phổ biến nhất trong hệ sinh thái Java.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Java Core hiện đại — Generics, Streams, Optional, Records (Java 17+)
- [ ] Spring IoC Container — Inversion of Control (Đảo Ngược Quyền Kiểm Soát) & DI (Dependency Injection — Tiêm Phụ Thuộc)
- [ ] Spring Boot Auto-configuration (Tự Động Cấu Hình) & Starters
- [ ] Bean Lifecycle (Vòng Đời Bean) & Scopes (Phạm Vi)
- [ ] Cấu hình qua `application.properties` / YAML & Profiles (Môi Trường)

### **Giai Đoạn 2: Phát Triển API (Tuần 3–6)**

- [ ] REST API với `@RestController`, `@RequestMapping`
- [ ] DTO (Data Transfer Object — Đối Tượng Truyền Dữ Liệu) & Validation (Bean Validation — Xác Thực)
- [ ] Global Exception Handling (Xử Lý Ngoại Lệ Toàn Cục) với `@ControllerAdvice`
- [ ] Spring Data JPA — Repository Pattern, JPQL, Criteria API
- [ ] Transaction Management (Quản Lý Giao Dịch) với `@Transactional`
- [ ] Spring Security — Authentication (Xác Thực) & Authorization (Phân Quyền)

### **Giai Đoạn 3: Kỹ Năng Nâng Cao (Tuần 7–10)**

- [ ] JWT (JSON Web Token) & OAuth2 / OIDC (OpenID Connect)
- [ ] Caching (Bộ Nhớ Đệm) với Spring Cache & Redis
- [ ] Async Programming (Lập Trình Bất Đồng Bộ) — `@Async`, CompletableFuture
- [ ] Messaging (Nhắn Tin) — Apache Kafka & RabbitMQ
- [ ] Testing (Kiểm Thử) — JUnit 5, Mockito, Testcontainers
- [ ] Docker & Kubernetes deployment (Triển Khai)

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Microservices Architecture (Kiến Trúc Microservices) với Spring Cloud
- [ ] Reactive Programming (Lập Trình Phản Ứng) — Spring WebFlux, Project Reactor
- [ ] CQRS (Command Query Responsibility Segregation — Tách Biệt Trách Nhiệm Lệnh & Truy Vấn) & Event Sourcing
- [ ] Performance Tuning (Tinh Chỉnh Hiệu Năng) — JVM, HikariCP, profiling
- [ ] GraalVM Native Image (Ảnh Nhị Phân Gốc)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| --------------------------------------- | ---------- | --------- | ---------- |
| **Spring Core & DI**                    | ⭐⭐⭐      | 1 tuần    | -          |
| **REST API Development**                | ⭐⭐⭐      | 2 tuần    | -          |
| **Spring Data JPA & ORM**               | ⭐⭐⭐      | 2 tuần    | -          |
| **Spring Security & JWT**               | ⭐⭐⭐      | 2 tuần    | -          |
| **Testing (Unit & Integration)**        | ⭐⭐⭐      | 2 tuần    | -          |
| **Transaction Management**              | ⭐⭐⭐      | 1 tuần    | -          |
| **Caching & Performance**               | ⭐⭐⭐      | 1 tuần    | -          |
| **Async & Messaging**                   | ⭐⭐       | 2 tuần    | -          |
| **Microservices & Spring Cloud**        | ⭐⭐       | 3 tuần    | -          |
| **Docker & Kubernetes Deployment**      | ⭐⭐       | 2 tuần    | -          |
| **Reactive Programming (WebFlux)**      | ⭐         | 2 tuần    | -          |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Nền Tảng** (`01-fundamentals/`)

- Java 17+ — Records, Sealed Classes, Pattern Matching, Virtual Threads
- Spring IoC Container — ApplicationContext, BeanFactory
- Auto-configuration (Tự Động Cấu Hình) & `@SpringBootApplication`
- Bean Scopes — Singleton, Prototype, Request, Session
- Configuration Properties — `@Value`, `@ConfigurationProperties`, Profiles

### 📁 **2. Tầng Web — REST API** (`02-web-layer/`)

- `@RestController`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- **DTO Pattern** — Request/Response Objects, MapStruct
- **Bean Validation** — `@Valid`, `@NotNull`, Custom Validators
- **Global Exception Handler** — `@ControllerAdvice`, `ProblemDetail` (RFC 7807)
- **Filter** (Bộ Lọc) & **Interceptor** (Bộ Chặn) — `OncePerRequestFilter`, `HandlerInterceptor`
- OpenAPI / Swagger — springdoc-openapi

### 📁 **3. Truy Cập Dữ Liệu** (`03-data-access/`)

- **Spring Data JPA** — `JpaRepository`, `@Entity`, JPQL, Criteria API
- **ORM Mapping** — One-to-Many, Many-to-Many, `@JoinColumn`, `@MappedSuperclass`
- **Transaction** — `@Transactional`, Propagation (Lan Truyền), Isolation Levels (Mức Cô Lập)
- **N+1 Problem** — Eager vs Lazy Loading, `@EntityGraph`, `JOIN FETCH`
- **Spring Data Redis** — `RedisTemplate`, `@Cacheable`
- **Spring Data MongoDB** — `MongoRepository`, Aggregation Pipeline

### 📁 **4. Bảo Mật** (`04-security/`)

- **Spring Security** — `SecurityFilterChain`, `AuthenticationManager`
- **JWT** (JSON Web Token) — Access Token, Refresh Token, `JwtFilter`
- **OAuth2** — Authorization Code Flow, Resource Server, `@AuthenticationPrincipal`
- **OIDC** (OpenID Connect) — Identity Provider, Claims
- **Method Security** — `@PreAuthorize`, `@PostAuthorize`, `@Secured`
- **CORS** (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Chéo Nguồn Gốc) & **CSRF** Protection
- Mã hóa mật khẩu với BCrypt

### 📁 **5. Bất Đồng Bộ & Nhắn Tin** (`05-async-messaging/`)

- **`@Async`** & `CompletableFuture` — Non-blocking Operations (Thao Tác Không Chặn)
- **Spring Events** — `ApplicationEvent`, `@EventListener`, `@TransactionalEventListener`
- **Apache Kafka** — `@KafkaListener`, `KafkaTemplate`, Consumer Groups
- **RabbitMQ** — `@RabbitListener`, Exchanges, Queues, Bindings
- **`@Scheduled`** & **Quartz Scheduler** — Tác Vụ Định Kỳ

### 📁 **6. Kiểm Thử** (`06-testing/`)

- **JUnit 5** — `@Test`, `@ParameterizedTest`, `@ExtendWith`
- **Mockito** — `@Mock`, `@InjectMocks`, `verify()`, `ArgumentCaptor`
- **`@SpringBootTest`** — Full Context Integration Test
- **`@WebMvcTest`** & `MockMvc` — Controller Layer Testing
- **`@DataJpaTest`** — Repository Layer Testing với H2
- **Testcontainers** — Test với DB/Kafka thực tế trong Docker
- **JaCoCo** — Code Coverage Report (Báo Cáo Độ Phủ Kiểm Thử)

### 📁 **7. Hiệu Năng** (`07-performance/`)

- **Spring Cache Abstraction** — `@Cacheable`, `@CacheEvict`, `@CachePut`
- **HikariCP** — Connection Pool (Bể Kết Nối) tuning — `maximumPoolSize`, `minimumIdle`
- **JVM Tuning** — Heap size, GC (Garbage Collector — Bộ Thu Gom Rác) selection, G1GC vs ZGC
- **Query Optimization** — Execution plans, index hints, batch operations
- **Profiling** — async-profiler, JFR (Java Flight Recorder), JProfiler
- **Load Testing** (Kiểm Thử Tải) — Gatling, k6

### 📁 **8. Kiến Trúc** (`08-architecture/`)

- **Clean Architecture** (Kiến Trúc Sạch) — Ports & Adapters (Hexagonal)
- **Layered Architecture** — Controller → Service → Repository
- **Microservices** — Service Decomposition, Domain Boundaries
- **API Gateway** (Cổng API) — Spring Cloud Gateway, Rate Limiting
- **Service Discovery** (Khám Phá Dịch Vụ) — Eureka, Consul
- **CQRS** & **Event Sourcing** — Command/Query Separation

### 📁 **9. Triển Khai Đám Mây** (`09-cloud-deployment/`)

- **Docker** — Multi-stage Dockerfile cho Spring Boot
- **Kubernetes** (K8s) — Deployment, Service, ConfigMap, Secret, HPA
- **Spring Boot Actuator** — Health checks, Metrics endpoints
- **Micrometer** & **Prometheus** — Metrics collection (Thu Thập Chỉ Số)
- **Distributed Tracing** (Theo Dõi Phân Tán) — Micrometer Tracing, Zipkin, Jaeger
- **CI/CD Pipeline** — GitHub Actions, Jenkins

### 📁 **10. Chủ Đề Nâng Cao** (`10-advanced/`)

- **Spring WebFlux** — Reactive Streams (Luồng Phản Ứng), `Mono`, `Flux`
- **Project Reactor** — Operators, Backpressure (Áp Lực Ngược)
- **Spring Batch** — Job, Step, ItemReader/ItemProcessor/ItemWriter
- **gRPC** — Protocol Buffers, streaming RPC
- **Spring GraphQL** — Schema-first approach, DataFetcher
- **GraalVM Native Image** — AOT (Ahead-of-Time) compilation

### 📁 **11. Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 50 câu hỏi phỏng vấn Spring Boot
- System Design Scenarios (Tình Huống Thiết Kế Hệ Thống)
- Câu chuyện STAR cho các tình huống thực tế
- Bài tập coding & live coding challenges
- Kế hoạch học 90 ngày

---

## 🎓 Theo Loại Công Nghệ

### **Spring MVC (Web truyền thống)**

```
Điểm mạnh: Blocking I/O, dễ debug, hệ sinh thái phong phú
Phù hợp: CRUD API, doanh nghiệp, tích hợp với JPA
Học trong: 01-fundamentals, 02-web-layer, 03-data-access
```

### **Spring WebFlux (Reactive)**

```
Điểm mạnh: Non-blocking I/O, throughput cao, streaming
Phù hợp: Microservices, gateway, real-time data
Học trong: 10-advanced/reactive-programming
```

### **Spring Security**

```
Điểm mạnh: Mạnh mẽ, linh hoạt, tích hợp sâu với Spring
Phù hợp: Mọi ứng dụng web & API
Học trong: 04-security
```

### **Spring Data**

```
Điểm mạnh: Giảm boilerplate, đa nền tảng (JPA, Redis, Mongo, Elasticsearch)
Phù hợp: Mọi project cần truy cập DB
Học trong: 03-data-access
```

### **Spring Cloud**

```
Điểm mạnh: Gateway, Service Discovery, Config Server, Circuit Breaker
Phù hợp: Microservices architecture
Học trong: 08-architecture, 09-cloud-deployment
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                      | Thư Mục                                                             | Ưu Tiên           |
| --------------------------- | ------------------------------------------------------------------- | ----------------- |
| Bắt đầu từ đâu?             | [Lộ trình học](#lộ-trình-học)                                        | Bắt đầu tại đây   |
| Câu hỏi phỏng vấn           | [11-interview-prep](./11-interview-prep/)                           | Trước phỏng vấn   |
| Spring Security + JWT       | [04-security](./04-security/)                                       | Thiết yếu         |
| JPA & Transaction           | [03-data-access](./03-data-access/)                                 | Thiết yếu         |
| Testing Strategy            | [06-testing](./06-testing/)                                         | Bắt buộc          |
| Docker + K8s                | [09-cloud-deployment](./09-cloud-deployment/)                       | Production-ready  |

---

## 📊 Ma Trận Kỹ Năng

### Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Hiểu IoC và DI trong Spring
- [ ] Tạo được REST API cơ bản
- [ ] CRUD với Spring Data JPA
- [ ] Viết Unit Test với JUnit 5 & Mockito
- [ ] Đọc được `application.properties` & Profiles

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Thiết kế API RESTful chuẩn với error handling
- [ ] Tích hợp Spring Security + JWT đầy đủ
- [ ] Xử lý N+1 problem và tối ưu JPA
- [ ] Viết Integration Test với Testcontainers
- [ ] Cấu hình Caching với Redis
- [ ] Deploy ứng dụng với Docker

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Thiết kế Microservices với Spring Cloud
- [ ] Viết Reactive API với Spring WebFlux
- [ ] Triển khai CI/CD pipeline hoàn chỉnh
- [ ] Profiling và tối ưu JVM performance
- [ ] Thiết kế Event-Driven Architecture (CQRS/Event Sourcing)
- [ ] GraalVM Native Image cho production

---

## 🚀 Bắt Đầu

### Bước 1: Đặt Mục Tiêu Học

```
Chọn con đường:
- Backend Developer toàn diện (API + Security + Testing + Deploy)
- Microservices Architect (Spring Cloud + Kafka + K8s)
- Performance Engineer (JVM + profiling + reactive)
```

### Bước 2: Dựng Môi Trường Lab

```bash
# Khởi động stack phát triển với Docker Compose
docker-compose up -d

# Bao gồm: PostgreSQL, Redis, Kafka, Zookeeper, Elasticsearch
```

### Bước 3: Học & Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Viết code theo ví dụ (30 phút)
3. Tự xây dựng feature từ đầu (30–60 phút)
4. Viết tests cho code vừa viết (30 phút)
5. Kiểm tra checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation (Tình huống): Bối cảnh dự án
- Task (Nhiệm vụ): Yêu cầu cần giải quyết
- Action (Hành động): Cách bạn tiếp cận
- Result (Kết quả): Impact đo lường được
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Spring in Action"** (Craig Walls) — Tổng quan Spring/Spring Boot
- **"Cloud Native Java"** (Josh Long & Kenny Bastani) — Spring Cloud & microservices
- **"Designing Data-Intensive Applications"** (Martin Kleppmann) — System design
- **"Effective Java"** (Joshua Bloch) — Java best practices
- **"Release It!"** (Michael Nygard) — Production-ready patterns

### Tài Liệu Chính Thức

- [Spring Boot Reference Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Cloud Reference](https://spring.io/projects/spring-cloud)

### Blog & Nguồn Học

- [Baeldung](https://www.baeldung.com/) — Hướng dẫn Spring chất lượng cao
- [Spring Blog](https://spring.io/blog) — Cập nhật chính thức
- [Vlad Mihalcea Blog](https://vladmihalcea.com/) — JPA/Hibernate sâu
- [Thorben Janssen (Thoughts on Java)](https://thorben-janssen.com/) — JPA chuyên sâu

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Top Câu Hỏi Theo Chủ Đề

#### Spring Core & DI

- [ ] Sự khác biệt giữa `@Component`, `@Service`, `@Repository`, `@Controller`?
- [ ] Giải thích Bean Lifecycle trong Spring
- [ ] `@Autowired` vs Constructor Injection — khi nào dùng cái nào?
- [ ] Circular Dependency (Phụ Thuộc Vòng) xảy ra khi nào và cách giải quyết?

#### Spring Data JPA

- [ ] N+1 Problem là gì? Cách phát hiện và giải quyết?
- [ ] Sự khác biệt giữa `EAGER` và `LAZY` loading?
- [ ] `@Transactional` hoạt động thế nào với proxy?
- [ ] Khi nào dùng JPQL vs Criteria API vs Native Query?

#### Spring Security

- [ ] Mô tả Security Filter Chain (Chuỗi Bộ Lọc Bảo Mật)
- [ ] JWT Refresh Token strategy là gì?
- [ ] Sự khác biệt giữa Authentication (Xác Thực) và Authorization (Phân Quyền)?
- [ ] Cách implement Row-Level Security (Bảo Mật Cấp Hàng)?

#### Performance & Scaling

- [ ] Cách bạn tune HikariCP cho production?
- [ ] GC tuning — khi nào chuyển từ G1GC sang ZGC?
- [ ] Caching strategy: Local vs Distributed — khi nào dùng cái nào?
- [ ] Cách bạn debug memory leak trong Spring Boot?

#### Kiến Trúc

- [ ] Monolith vs Microservices — khi nào migrate?
- [ ] Cách implement Circuit Breaker (Cầu Dao Mạch) với Resilience4j?
- [ ] Distributed Transaction (Giao Dịch Phân Tán) trong microservices?
- [ ] Saga Pattern là gì?

Xem `11-interview-prep/` để có hướng dẫn đầy đủ Q&A.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc nhận dự án mới, kiểm tra:

- [ ] Có thể giải thích IoC/DI bằng lời mình mà không cần nhìn tài liệu
- [ ] Có thể thiết kế RESTful API với đầy đủ error handling
- [ ] Có thể debug N+1 problem trong JPA
- [ ] Có thể implement JWT authentication từ đầu
- [ ] Có thể viết Integration Test với Testcontainers
- [ ] Có thể Docker hóa và deploy ứng dụng Spring Boot
- [ ] Có thể giải thích `@Transactional` propagation và isolation
- [ ] Có thể thiết kế caching strategy cho ứng dụng
- [ ] Có thể xử lý concurrent requests an toàn (thread-safety)
- [ ] Có thể đọc và tối ưu slow queries từ JPA

---

## 📞 Hỗ Trợ & Tài Nguyên

### Công Cụ Phát Triển

- **Spring Initializr** — [start.spring.io](https://start.spring.io) — Khởi tạo project
- **IntelliJ IDEA** — IDE tốt nhất cho Spring
- **Postman / HTTPie** — Test API
- **DBeaver** — Database GUI client
- **Docker Desktop** — Container runtime

### Cộng Đồng

- [Spring Community Forum](https://community.spring.io/)
- [Stack Overflow — spring-boot tag](https://stackoverflow.com/questions/tagged/spring-boot)
- Reddit r/java, r/SpringBoot
- Baeldung YouTube Channel

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Học tuần tự từng giai đoạn
3. Làm bài tập thực hành sau mỗi module
4. Xây dựng project portfolio

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep](./11-interview-prep/)
2. Học sâu về Security + JPA (hầu như mọi phỏng vấn đều hỏi)
3. Chuẩn bị câu chuyện STAR về incidents thực tế
4. Mock interview với đồng nghiệp

### Học Trong Dự Án Thực Tế

1. Tham khảo [03-data-access](./03-data-access/) cho JPA & DB issues
2. Dùng [04-security](./04-security/) cho authentication/authorization
3. Xem [07-performance](./07-performance/) để tối ưu hiệu năng
4. Theo [09-cloud-deployment](./09-cloud-deployment/) để deploy production

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn con đường học (Mới bắt đầu / Trung cấp / Nâng cao)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Dựng môi trường lab với Docker
├─ 5️⃣  Hoàn thành bài tập cho từng chủ đề
├─ 6️⃣  Xây dựng project portfolio
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
