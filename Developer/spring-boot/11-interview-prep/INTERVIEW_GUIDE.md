# 🏆 Top 50 Câu Hỏi Phỏng Vấn Spring Boot

> Tổng hợp 50 câu hỏi phỏng vấn phổ biến nhất về Java Spring Boot, có câu trả lời mẫu chi tiết. Mỗi câu trả lời tuân theo cấu trúc WHAT–WHY–HOW để dễ nhớ và trình bày.

---

## 📋 Danh Sách Nhanh

| # | Câu Hỏi | Chủ Đề | Mức Độ |
|---|---------|--------|--------|
| 1 | IoC và DI là gì? | Spring Core | ⭐ Cơ bản |
| 2 | Vòng đời Bean trong Spring | Spring Core | ⭐ Cơ bản |
| 3 | @Component vs @Service vs @Repository vs @Controller | Spring Core | ⭐ Cơ bản |
| 4 | Constructor Injection vs Field Injection | Spring Core | ⭐⭐ Trung cấp |
| 5 | Circular Dependency là gì? | Spring Core | ⭐⭐ Trung cấp |
| 6 | Auto-configuration hoạt động thế nào? | Spring Boot | ⭐⭐ Trung cấp |
| 7 | @SpringBootApplication gồm những gì? | Spring Boot | ⭐ Cơ bản |
| 8 | Bean Scope: Singleton vs Prototype | Spring Core | ⭐⭐ Trung cấp |
| 9 | @Transactional hoạt động thế nào? | Data Access | ⭐⭐ Trung cấp |
| 10 | Transaction Propagation là gì? | Data Access | ⭐⭐⭐ Nâng cao |
| 11 | Isolation Level trong Transaction | Data Access | ⭐⭐⭐ Nâng cao |
| 12 | N+1 Problem là gì? Cách giải quyết? | Data Access | ⭐⭐⭐ Rất quan trọng |
| 13 | LAZY vs EAGER Loading | Data Access | ⭐⭐ Trung cấp |
| 14 | JPQL vs Criteria API vs Native Query | Data Access | ⭐⭐ Trung cấp |
| 15 | @Query vs Derived Query Methods | Data Access | ⭐⭐ Trung cấp |
| 16 | Spring Security Filter Chain là gì? | Security | ⭐⭐ Trung cấp |
| 17 | Authentication vs Authorization | Security | ⭐ Cơ bản |
| 18 | JWT hoạt động thế nào? | Security | ⭐⭐⭐ Rất quan trọng |
| 19 | Access Token vs Refresh Token | Security | ⭐⭐⭐ Rất quan trọng |
| 20 | OAuth2 Authorization Code Flow | Security | ⭐⭐⭐ Nâng cao |
| 21 | CORS là gì? Cách cấu hình? | Security | ⭐⭐ Trung cấp |
| 22 | CSRF Protection — khi nào cần? | Security | ⭐⭐ Trung cấp |
| 23 | @PreAuthorize vs @Secured | Security | ⭐⭐ Trung cấp |
| 24 | REST API best practices | Web Layer | ⭐⭐ Trung cấp |
| 25 | @ControllerAdvice hoạt động thế nào? | Web Layer | ⭐⭐ Trung cấp |
| 26 | Filter vs Interceptor — khác nhau thế nào? | Web Layer | ⭐⭐ Trung cấp |
| 27 | @Async hoạt động thế nào? | Async | ⭐⭐ Trung cấp |
| 28 | CompletableFuture là gì? | Async | ⭐⭐ Trung cấp |
| 29 | Kafka vs RabbitMQ — khi nào dùng cái nào? | Messaging | ⭐⭐⭐ Nâng cao |
| 30 | Consumer Group trong Kafka | Messaging | ⭐⭐ Trung cấp |
| 31 | @SpringBootTest vs @WebMvcTest vs @DataJpaTest | Testing | ⭐⭐ Trung cấp |
| 32 | Testcontainers là gì? Tại sao dùng? | Testing | ⭐⭐ Trung cấp |
| 33 | Mockito — @Mock vs @MockBean | Testing | ⭐⭐ Trung cấp |
| 34 | Caching strategy: Local vs Distributed | Performance | ⭐⭐ Trung cấp |
| 35 | @Cacheable, @CacheEvict, @CachePut | Performance | ⭐⭐ Trung cấp |
| 36 | HikariCP tuning cho production | Performance | ⭐⭐⭐ Nâng cao |
| 37 | G1GC vs ZGC — khi nào chọn cái nào? | Performance | ⭐⭐⭐ Nâng cao |
| 38 | Cách debug memory leak | Performance | ⭐⭐⭐ Nâng cao |
| 39 | Monolith vs Microservices — khi nào migrate? | Architecture | ⭐⭐⭐ Rất quan trọng |
| 40 | Circuit Breaker pattern là gì? | Architecture | ⭐⭐⭐ Nâng cao |
| 41 | Saga Pattern trong Microservices | Architecture | ⭐⭐⭐ Nâng cao |
| 42 | CQRS là gì? | Architecture | ⭐⭐⭐ Nâng cao |
| 43 | Event Sourcing là gì? | Architecture | ⭐⭐⭐ Nâng cao |
| 44 | Spring Boot Actuator — /health vs /metrics | DevOps | ⭐⭐ Trung cấp |
| 45 | Micrometer + Prometheus + Grafana | DevOps | ⭐⭐ Trung cấp |
| 46 | Cách cấu hình HPA trên Kubernetes | DevOps | ⭐⭐⭐ Nâng cao |
| 47 | Spring WebFlux vs Spring MVC | Advanced | ⭐⭐⭐ Nâng cao |
| 48 | Backpressure trong Reactive Programming | Advanced | ⭐⭐⭐ Nâng cao |
| 49 | GraalVM Native Image — ưu và nhược điểm | Advanced | ⭐⭐⭐ Nâng cao |
| 50 | Spring Batch vs Quartz Scheduler | Advanced | ⭐⭐ Trung cấp |

---

## 🔵 Phần 1: Spring Core & Dependency Injection

### Câu 1: IoC và DI là gì?

**Mức độ:** Cơ bản | **Xuất hiện:** 95% phỏng vấn

**Câu trả lời:**

**WHAT:**
- **IoC (Inversion of Control — Đảo Ngược Quyền Kiểm Soát)**: Nguyên tắc thiết kế trong đó framework kiểm soát luồng chương trình thay vì code ứng dụng.
- **DI (Dependency Injection — Tiêm Phụ Thuộc)**: Cơ chế hiện thực hóa IoC — framework cung cấp (inject) các dependencies cho object thay vì object tự tạo.

**WHY:** Giảm coupling (sự phụ thuộc chặt chẽ) giữa các components, dễ test, dễ thay thế implementation.

**HOW:**

```java
// ❌ TRƯỚC IoC: UserService tự tạo dependency
public class UserService {
    private final UserRepository repo = new UserRepository(); // tight coupling
}

// ✅ SAU DI: Spring inject dependency vào
@Service
public class UserService {
    private final UserRepository repo; // Spring cung cấp

    public UserService(UserRepository repo) { // Constructor Injection
        this.repo = repo;
    }
}
```

**Điểm thêm:** Spring Container (ApplicationContext) quản lý toàn bộ vòng đời của beans và wiring giữa chúng.

---

### Câu 2: Vòng Đời Bean (Bean Lifecycle) trong Spring là gì?

**Mức độ:** Cơ bản | **Xuất hiện:** 80% phỏng vấn

**Câu trả lời:**

**WHAT:** Các giai đoạn một Spring Bean trải qua từ khi được tạo đến khi bị hủy.

**Các giai đoạn chính:**

```
1. Instantiation    — Spring tạo instance (gọi constructor)
2. Populate Props   — Spring inject dependencies (@Autowired)
3. BeanNameAware    — set tên bean (nếu implement interface)
4. BeanFactoryAware — set reference đến BeanFactory
5. @PostConstruct   — gọi init method
6. Ready to Use     — Bean sẵn sàng phục vụ requests
7. @PreDestroy      — gọi cleanup method khi context đóng
```

**HOW:**

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct
    public void init() {
        // Khởi tạo pool kết nối — chạy SAU khi inject xong
        System.out.println("Pool initialized");
    }

    @PreDestroy
    public void cleanup() {
        // Đóng tất cả kết nối — chạy TRƯỚC khi bean bị hủy
        System.out.println("Pool closed");
    }
}
```

---

### Câu 3: @Component vs @Service vs @Repository vs @Controller — Khác nhau thế nào?

**Mức độ:** Cơ bản | **Xuất hiện:** 85% phỏng vấn

**Câu trả lời:**

Tất cả đều là **stereotype annotations** (annotation phân loại) — khai báo để Spring tự động scan và tạo bean. Về mặt kỹ thuật, chúng đều giống nhau (đều là `@Component`), nhưng khác nhau về **ngữ nghĩa** và **chức năng bổ sung**:

| Annotation | Tầng | Chức Năng Bổ Sung |
| ---------- | ---- | ----------------- |
| `@Component` | Bất kỳ | Không có — generic |
| `@Service` | Business Logic | Không có — chỉ khác tên |
| `@Repository` | Data Access | Tự động translate database exceptions thành `DataAccessException` |
| `@Controller` | Web Layer | Tích hợp với Spring MVC dispatcher |
| `@RestController` | Web Layer | `@Controller` + `@ResponseBody` |

**Thực tiễn:** Luôn dùng annotation phù hợp với tầng — giúp code tự document và Spring có thể áp dụng AOP (Aspect-Oriented Programming — Lập Trình Hướng Khía Cạnh) đúng tầng.

---

### Câu 4: Constructor Injection vs Field Injection — Khi nào dùng cái nào?

**Mức độ:** Trung cấp | **Xuất hiện:** 75% phỏng vấn

**Câu trả lời:**

**Constructor Injection (được khuyến nghị):**

```java
@Service
public class OrderService {
    private final PaymentService paymentService; // final — immutable

    // Spring tự inject nếu chỉ có 1 constructor (không cần @Autowired)
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Field Injection (KHÔNG khuyến nghị):**

```java
@Service
public class OrderService {
    @Autowired // inject trực tiếp vào field
    private PaymentService paymentService; // KHÔNG thể final
}
```

**Tại sao Constructor Injection tốt hơn:**

| Tiêu Chí | Constructor | Field |
| -------- | ----------- | ----- |
| Testability (Khả năng test) | ✅ Dễ mock | ❌ Cần Spring context |
| Immutability (Bất biến) | ✅ `final` được | ❌ Không được |
| Circular Dependency detection | ✅ Phát hiện lúc startup | ❌ Phát hiện lúc runtime |
| Null safety | ✅ Rõ ràng | ❌ Có thể NPE |

---

### Câu 5: Circular Dependency (Phụ Thuộc Vòng) là gì? Cách giải quyết?

**Mức độ:** Trung cấp | **Xuất hiện:** 60% phỏng vấn

**Câu trả lời:**

**WHAT:** Xảy ra khi Bean A phụ thuộc vào Bean B và Bean B cũng phụ thuộc vào Bean A.

```java
// ❌ Circular Dependency
@Service
class ServiceA {
    ServiceA(ServiceB b) {} // A cần B
}
@Service
class ServiceB {
    ServiceB(ServiceA a) {} // B cần A — VÒNG TRÒN!
}
// → BeanCurrentlyInCreationException
```

**Cách giải quyết:**

1. **Tái cấu trúc code** (tốt nhất): Tách interface chung, dùng `@EventListener` để giảm coupling
2. **`@Lazy`**: Trì hoãn inject đến khi thực sự cần
3. **Setter Injection**: Cho phép inject sau khi constructor chạy xong

```java
// Giải pháp @Lazy
@Service
class ServiceA {
    ServiceA(@Lazy ServiceB b) { ... } // inject lười biếng
}
```

---

### Câu 6: Spring Boot Auto-configuration (Tự Động Cấu Hình) hoạt động thế nào?

**Mức độ:** Trung cấp | **Xuất hiện:** 70% phỏng vấn

**Câu trả lời:**

**WHAT:** Cơ chế Spring Boot tự động cấu hình các beans dựa trên classpath, properties và điều kiện.

**Chuỗi hoạt động:**

```
1. @SpringBootApplication kích hoạt @EnableAutoConfiguration
2. Spring Boot scan file META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
3. Mỗi AutoConfiguration class có @Conditional annotations
4. Nếu điều kiện thỏa mãn → bean được tạo tự động
```

**Ví dụ thực tế:**

```java
// Spring Boot tự động cấu hình DataSource nếu:
// 1. spring-boot-starter-data-jpa có trong classpath
// 2. spring.datasource.url được config

// Bạn có thể override bằng cách định nghĩa bean của mình:
@Bean
@Primary
public DataSource customDataSource() {
    return DataSourceBuilder.create()
        .url("jdbc:postgresql://localhost/mydb")
        .build();
}
```

**Debug auto-configuration:** `--debug` flag hoặc `application.properties`: `debug=true`

---

### Câu 8: Bean Scope — Singleton vs Prototype

**Mức độ:** Trung cấp | **Xuất hiện:** 65% phỏng vấn

**Câu trả lời:**

| Scope | Số Instance | Khi Nào Tạo | Dùng Khi |
| ----- | ----------- | ----------- | -------- |
| `singleton` (mặc định) | 1 per ApplicationContext | Lúc startup | Stateless services |
| `prototype` | Mỗi lần inject | Mỗi lần yêu cầu | Stateful objects |
| `request` | 1 per HTTP request | Mỗi request | Web — request-scoped data |
| `session` | 1 per HTTP session | Mỗi session mới | Web — user session data |

```java
@Component
@Scope("prototype") // mỗi inject = 1 instance mới
public class ShoppingCart { // stateful — cần prototype
    private List<Item> items = new ArrayList<>();
}
```

---

## 🟢 Phần 2: Spring Data JPA & Transaction

### Câu 9: @Transactional hoạt động thế nào?

**Mức độ:** Trung cấp | **Xuất hiện:** 90% phỏng vấn

**Câu trả lời:**

**WHAT:** Annotation khai báo transaction boundary — Spring tự động bắt đầu, commit hoặc rollback transaction.

**HOW (cơ chế proxy):**

```
Caller → Spring AOP Proxy → @Transactional method → actual code
         [begin tx]                                  [commit/rollback]
```

**Lưu ý quan trọng — self-invocation không hoạt động:**

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        // ✅ Transaction hoạt động — gọi từ bên ngoài
        repo.save(order);
        sendNotification(order); // ❌ KHÔNG có transaction — self-invocation
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // Không có transaction vì gọi qua this, không qua proxy
    }
}
```

**Rollback:** Mặc định chỉ rollback với `RuntimeException`. Dùng `rollbackFor = Exception.class` để rollback với checked exceptions.

---

### Câu 10: Transaction Propagation (Lan Truyền Giao Dịch) là gì?

**Mức độ:** Nâng cao | **Xuất hiện:** 75% phỏng vấn

**Câu trả lời:**

Xác định hành vi khi một `@Transactional` method gọi một `@Transactional` method khác:

| Propagation | Hành Vi |
| ----------- | ------- |
| `REQUIRED` (mặc định) | Tham gia tx đang có; nếu không có thì tạo mới |
| `REQUIRES_NEW` | Luôn tạo tx mới; suspend tx đang có |
| `SUPPORTS` | Tham gia nếu có; không có cũng chạy |
| `NOT_SUPPORTED` | Chạy ngoài tx; suspend tx đang có |
| `MANDATORY` | Bắt buộc phải có tx hiện tại; không có → exception |
| `NEVER` | Không được có tx; nếu có → exception |
| `NESTED` | Tạo nested tx (savepoint) trong tx đang có |

**Ví dụ thực tế:**

```java
@Transactional
public void processOrder(Order order) {
    orderRepo.save(order);
    // Nếu gửi email lỗi → KHÔNG rollback order (dùng REQUIRES_NEW)
    emailService.sendConfirmation(order);
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendConfirmation(Order order) {
    // TX độc lập — lỗi ở đây không ảnh hưởng tx của processOrder
    emailLog.save(new EmailLog(order));
}
```

---

### Câu 12: N+1 Problem là gì? Cách Phát Hiện và Giải Quyết?

**Mức độ:** Rất quan trọng | **Xuất hiện:** 75% phỏng vấn

**Câu trả lời:**

**WHAT:** Khi load 1 list entity (1 query), JPA thực hiện thêm N queries riêng lẻ để load associations.

```java
// 1 query để load 100 orders
List<Order> orders = orderRepo.findAll();
// + 100 queries riêng lẻ để load customer của mỗi order!
orders.forEach(o -> o.getCustomer().getName()); // LAZY → N+1
```

**Cách phát hiện:**

```yaml
# application.yml — bật log SQL để phát hiện
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
# Dùng p6spy hoặc datasource-proxy để đếm số queries
```

**3 Cách Giải Quyết:**

**1. JOIN FETCH trong JPQL:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findByStatusWithCustomer(@Param("status") OrderStatus status);
```

**2. @EntityGraph:**
```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus status);
```

**3. Batch Size (cho collections):**
```java
@OneToMany
@BatchSize(size = 20) // load 20 collections cùng lúc thay vì 1 lần 1
private List<OrderItem> items;
```

**Trade-off:** JOIN FETCH có thể gây CartesianProduct nếu fetch nhiều collections — cân nhắc dùng separate queries.

---

### Câu 11: Isolation Level (Mức Cô Lập Giao Dịch) trong Transaction

**Mức độ:** Nâng cao | **Xuất hiện:** 60% phỏng vấn

**Câu trả lời:**

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
| --------------- | ---------- | ------------------- | ------------ |
| `READ_UNCOMMITTED` | ✅ Có thể | ✅ Có thể | ✅ Có thể |
| `READ_COMMITTED` (default Postgres) | ❌ Không | ✅ Có thể | ✅ Có thể |
| `REPEATABLE_READ` (default MySQL) | ❌ Không | ❌ Không | ✅ Có thể |
| `SERIALIZABLE` | ❌ Không | ❌ Không | ❌ Không |

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void processInventory(Long productId) {
    // Đảm bảo đọc số lượng tồn kho nhất quán trong toàn bộ transaction
    int stock = inventoryRepo.getStock(productId);
    if (stock > 0) {
        inventoryRepo.decrementStock(productId);
    }
}
```

---

## 🔴 Phần 3: Spring Security & JWT

### Câu 16: Spring Security Filter Chain (Chuỗi Bộ Lọc Bảo Mật) là gì?

**Mức độ:** Trung cấp | **Xuất hiện:** 80% phỏng vấn

**Câu trả lời:**

**WHAT:** Một chuỗi các Servlet Filters (Bộ Lọc Servlet) mà mọi HTTP request phải đi qua để được xử lý bảo mật.

**Thứ tự các filter chính:**

```
Request đến
    ↓
SecurityContextPersistenceFilter  — load SecurityContext từ session
    ↓
UsernamePasswordAuthenticationFilter — xử lý form login
    ↓
JwtAuthenticationFilter (custom)  — xác thực JWT token
    ↓
ExceptionTranslationFilter        — chuyển exception thành HTTP 401/403
    ↓
FilterSecurityInterceptor         — kiểm tra authorization cuối cùng
    ↓
Controller / Handler
```

**Cấu hình SecurityFilterChain:**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable) // stateless API không cần CSRF
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated()
        )
        .build();
}
```

---

### Câu 18: JWT (JSON Web Token) hoạt động thế nào?

**Mức độ:** Rất quan trọng | **Xuất hiện:** 85% phỏng vấn

**Câu trả lời:**

**Cấu trúc JWT:** `header.payload.signature` — 3 phần base64 encoded, ngăn cách bởi dấu chấm.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSIsInJvbGVzIjpbIlVTRVIiXX0.abc123
      HEADER                          PAYLOAD                         SIGNATURE
```

**Luồng xác thực:**

```
1. User gửi username/password → POST /api/auth/login
2. Server verify credentials → tạo JWT, ký bằng secret key
3. Server trả về access_token (15 phút) + refresh_token (7 ngày)
4. Client lưu token, gửi kèm mỗi request: Authorization: Bearer <token>
5. Server verify chữ ký JWT → extract claims → xác định user
6. Khi access_token hết hạn → dùng refresh_token để lấy token mới
```

**JwtFilter implementation:**

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String token = extractBearerToken(req);
        
        if (token != null && jwtService.isTokenValid(token)) {
            String username = jwtService.extractUsername(token);
            UserDetails user = userDetailsService.loadUserByUsername(username);
            
            // Set authentication vào SecurityContext
            SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities())
            );
        }
        
        chain.doFilter(req, res);
    }
}
```

---

### Câu 19: Access Token vs Refresh Token — Strategy là gì?

**Mức độ:** Rất quan trọng | **Xuất hiện:** 75% phỏng vấn

**Câu trả lời:**

| | Access Token | Refresh Token |
|-| ------------ | ------------- |
| **TTL (Time-to-Live)** | Ngắn (5–15 phút) | Dài (7–30 ngày) |
| **Lưu ở client** | Memory / LocalStorage | HttpOnly Cookie (bảo mật hơn) |
| **Gửi kèm request** | Mọi API request | Chỉ endpoint refresh |
| **Revocable** | Khó (stateless) | Có thể (lưu DB/Redis) |

**Refresh Token Rotation:**

```java
@PostMapping("/api/auth/refresh")
public TokenResponse refresh(@CookieValue("refresh_token") String refreshToken) {
    // 1. Validate refresh token (check DB/Redis)
    RefreshToken storedToken = refreshTokenRepo.findByToken(refreshToken)
        .orElseThrow(() -> new UnauthorizedException("Invalid token"));
    
    // 2. Kiểm tra hết hạn
    if (storedToken.isExpired()) {
        throw new UnauthorizedException("Refresh token expired");
    }
    
    // 3. Tạo access token mới
    // 4. Xoay vòng refresh token (xóa cũ, tạo mới) — Rotation
    refreshTokenRepo.delete(storedToken);
    String newRefreshToken = tokenService.generateRefreshToken(storedToken.getUser());
    
    return new TokenResponse(newAccessToken, newRefreshToken);
}
```

---

## 🟡 Phần 4: Testing

### Câu 31: @SpringBootTest vs @WebMvcTest vs @DataJpaTest — Khác nhau thế nào?

**Mức độ:** Trung cấp | **Xuất hiện:** 80% phỏng vấn

**Câu trả lời:**

| Annotation | Loads Context | Dùng Khi | Tốc Độ |
| ---------- | ------------- | -------- | ------ |
| `@SpringBootTest` | Toàn bộ ApplicationContext | Integration test full stack | Chậm (5–30s) |
| `@WebMvcTest` | Chỉ Web Layer (Controller, Filter) | Test Controller logic, validation | Nhanh (<2s) |
| `@DataJpaTest` | Chỉ JPA layer (Repository, Entity) | Test Repository queries | Nhanh (<2s) |
| `@Service` + Mockito | Không có Spring Context | Unit test Service thuần | Rất nhanh (<1s) |

```java
// @WebMvcTest — chỉ test controller, mock service
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService; // mock — không load service thật

    @Test
    void createUser_shouldReturn201() throws Exception {
        given(userService.createUser(any())).willReturn(new UserDTO(1L, "John"));
        
        mockMvc.perform(post("/api/users")
            .contentType(APPLICATION_JSON)
            .content("""{"name": "John", "email": "john@example.com"}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

---

### Câu 32: Testcontainers là gì? Tại sao dùng?

**Mức độ:** Trung cấp | **Xuất hiện:** 65% phỏng vấn

**Câu trả lời:**

**WHAT:** Thư viện Java tự động spin up Docker containers (PostgreSQL, Redis, Kafka...) cho integration tests — đảm bảo test chạy với database thật thay vì H2 in-memory.

**WHY:** H2 không tương thích 100% với PostgreSQL/MySQL — có thể pass test nhưng fail production.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        // Override connection properties để point đến container
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository orderRepo;

    @Test
    void findByStatus_shouldReturnMatchingOrders() {
        // Test với PostgreSQL thực — không phải H2
        orderRepo.save(new Order(OrderStatus.PENDING));
        assertThat(orderRepo.findByStatus(PENDING)).hasSize(1);
    }
}
```

---

## 🟠 Phần 5: Performance & Kiến Trúc

### Câu 36: HikariCP tuning cho production — Các thông số quan trọng?

**Mức độ:** Nâng cao | **Xuất hiện:** 55% phỏng vấn

**Câu trả lời:**

**HikariCP** là Connection Pool (Bể Kết Nối) mặc định trong Spring Boot — quản lý pool kết nối JDBC tới database.

**Công thức tính pool size (từ HikariCP docs):**
```
pool_size = (core_count * 2) + effective_spindle_count
```

**Cấu hình production:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20       # tối đa 20 connections
      minimum-idle: 5             # luôn duy trì 5 connections
      idle-timeout: 600000        # đóng idle connection sau 10 phút
      connection-timeout: 30000   # timeout khi chờ lấy connection: 30s
      max-lifetime: 1800000       # tối đa tuổi thọ connection: 30 phút
      leak-detection-threshold: 60000  # cảnh báo nếu connection bị giữ > 1 phút
```

**Dấu hiệu pool có vấn đề:**
- `Connection is not available, request timed out` → pool quá nhỏ hoặc có connection leak
- Nhiều `WAITING` threads → tăng `maximumPoolSize`
- Theo dõi qua: `/actuator/metrics/hikaricp.connections.active`

---

### Câu 39: Monolith vs Microservices — Khi nào migrate?

**Mức độ:** Rất quan trọng | **Xuất hiện:** 70% phỏng vấn

**Câu trả lời:**

**KHÔNG nên migrate sớm.** Microservices giải quyết vấn đề của scale — không phải vấn đề của startup.

**Dấu hiệu NÊN migrate sang Microservices:**

```
✅ Team > 20 engineers — khó coordinate trên 1 codebase
✅ Deployment bottleneck — 1 thay đổi nhỏ phải deploy cả system
✅ Scale không đồng đều — chỉ cần scale payment service, không cần scale tất cả
✅ Technology diversity — team cần dùng ngôn ngữ/framework khác nhau
✅ Domain rõ ràng — bounded contexts (ngữ cảnh ràng buộc) được xác định tốt
```

**Dấu hiệu NÊN giữ Monolith:**

```
❌ Startup / MVP — overhead không xứng với lợi ích
❌ Team nhỏ < 10 người — distributed overhead quá lớn
❌ Domain chưa ổn định — refactor microservice rất tốn kém
❌ Chưa có infrastructure — K8s, service mesh, observability
```

---

### Câu 40: Circuit Breaker (Cầu Dao Mạch) Pattern là gì?

**Mức độ:** Nâng cao | **Xuất hiện:** 65% phỏng vấn

**Câu trả lời:**

**WHAT:** Pattern bảo vệ service khỏi cascade failures (lỗi dây chuyền) khi downstream service bị chậm hoặc down.

**3 trạng thái:**

```
CLOSED (normal)
  ↓ failure rate > threshold
OPEN (reject all calls, return fallback)
  ↓ after wait duration
HALF_OPEN (allow trial calls)
  ↓ success → CLOSED | fail → OPEN
```

**Resilience4j implementation:**

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @Retry(name = "payment")
    public PaymentResult processPayment(PaymentRequest request) {
        return externalPaymentGateway.charge(request); // có thể fail
    }

    // Fallback được gọi khi circuit OPEN hoặc exception xảy ra
    private PaymentResult paymentFallback(PaymentRequest req, Exception e) {
        // Xử lý graceful: queue cho retry sau, trả về pending state
        pendingPaymentQueue.add(req);
        return PaymentResult.pending("Payment queued for retry");
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment:
        failure-rate-threshold: 50        # mở circuit khi 50% calls fail
        wait-duration-in-open-state: 30s  # chờ 30s trước khi thử lại
        sliding-window-size: 10           # đánh giá trên 10 calls gần nhất
```

---

### Câu 42: CQRS (Command Query Responsibility Segregation — Tách Biệt Trách Nhiệm Lệnh & Truy Vấn) là gì?

**Mức độ:** Nâng cao | **Xuất hiện:** 55% phỏng vấn

**Câu trả lời:**

**WHAT:** Tách biệt model cho write operations (Commands — Lệnh) và read operations (Queries — Truy Vấn).

**WHY:** Read và write có access patterns khác nhau — CQRS cho phép optimize độc lập.

```
Write side (Command):
  POST /orders → OrderCommandService → validates → saves to PostgreSQL → publishes event

Read side (Query):
  GET /orders → OrderQueryService → reads from Redis/Elasticsearch (optimized read model)
```

**Spring implementation:**

```java
// Command side — normalization, strong consistency
@Service
public class OrderCommandService {
    public void createOrder(CreateOrderCommand cmd) {
        Order order = Order.create(cmd); // domain validation
        orderRepo.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order)); // → update read model
    }
}

// Query side — denormalized, optimized for display
@Service
public class OrderQueryService {
    public OrderSummaryDTO getOrder(Long id) {
        return orderReadRepo.findSummaryById(id); // pre-aggregated view
    }
}
```

---

## ⚫ Phần 6: DevOps & Cloud

### Câu 44: Spring Boot Actuator — /health vs /metrics

**Mức độ:** Trung cấp | **Xuất hiện:** 65% phỏng vấn

**Câu trả lời:**

**`/actuator/health`** — Kubernetes dùng để kiểm tra pod có sống không:

```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "redis": { "status": "UP" },
    "diskSpace": { "status": "UP", "details": {"free": "50GB"} }
  }
}
```

**`/actuator/metrics`** — Thu thập số liệu cho Prometheus/Grafana:

```
/actuator/metrics/http.server.requests    — request count, latency
/actuator/metrics/jvm.memory.used         — heap usage
/actuator/metrics/hikaricp.connections    — DB connection pool stats
```

**Cấu hình production:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized  # ẩn details với user thường
  health:
    probes:
      enabled: true  # /health/liveness và /health/readiness cho K8s
```

---

### Câu 46: Cách cấu hình HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang) trên Kubernetes

**Mức độ:** Nâng cao | **Xuất hiện:** 45% phỏng vấn

**Câu trả lời:**

**HPA** tự động scale số lượng pods dựa trên metrics (CPU, memory, custom metrics).

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-app
  minReplicas: 2          # tối thiểu 2 pods
  maxReplicas: 10         # tối đa 10 pods
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # scale up khi CPU > 70%
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second  # custom metric từ Prometheus
        target:
          type: AverageValue
          averageValue: "100"
```

**Yêu cầu:** Pod phải có `resources.requests` được set để HPA tính % utilization.

---

## 🟣 Phần 7: Advanced Topics

### Câu 47: Spring WebFlux vs Spring MVC — Khi nào chọn cái nào?

**Mức độ:** Nâng cao | **Xuất hiện:** 45% phỏng vấn

**Câu trả lời:**

| | Spring MVC | Spring WebFlux |
|-| ---------- | -------------- |
| **I/O Model** | Blocking (Chặn) | Non-blocking (Không chặn) |
| **Thread Model** | 1 thread per request | Event loop |
| **Learning Curve** | Thấp | Cao (reactive mindset) |
| **JDBC/JPA** | ✅ Tự nhiên | ❌ Cần R2DBC |
| **Tốt cho** | CRUD apps, monolith | High-concurrency, streaming |
| **Debugging** | Dễ | Khó (stack trace không tuyến tính) |

**Chọn WebFlux khi:**
- Số lượng concurrent connections rất lớn (10k+)
- Streaming data (Server-Sent Events, WebSocket)
- Gọi nhiều external APIs đồng thời (fan-out)
- Đã dùng R2DBC + reactive drivers

**Chọn MVC khi:**
- CRUD thông thường với JPA/JDBC
- Team chưa quen reactive programming
- Domain logic phức tạp — blocking code dễ đọc hơn

---

### Câu 49: GraalVM Native Image — Ưu và Nhược Điểm?

**Mức độ:** Nâng cao | **Xuất hiện:** 35% phỏng vấn

**Câu trả lời:**

**GraalVM Native Image** biên dịch Spring Boot application thành native binary (không cần JVM runtime) thông qua AOT (Ahead-of-Time — Biên Dịch Trước).

**Ưu điểm:**
- Startup time: từ 3–10 giây xuống còn 50–200ms
- Memory footprint: giảm 50–80%
- Lý tưởng cho serverless (AWS Lambda, Google Cloud Run)

**Nhược điểm:**
- Build time: 5–15 phút (thay vì 30 giây)
- Reflection, dynamic proxies cần khai báo hints
- Không hỗ trợ tất cả libraries (cần kiểm tra)
- Debug khó hơn

```xml
<!-- pom.xml — bật native build -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
# Build native image
./mvnw -Pnative native:compile

# Chạy binary — không cần JVM!
./target/myapp
# Started in 0.089s (thay vì 3.2s với JVM)
```

---

## 📝 Checklist Ôn Tập Nhanh

### Trước Phỏng Vấn — Kiểm Tra Từng Mục

**Spring Core:**
- [ ] Giải thích IoC/DI được không nhìn notes?
- [ ] Vẽ được Bean Lifecycle đầy đủ?
- [ ] Phân biệt @Component vs @Service vs @Repository?
- [ ] Lý do chọn Constructor Injection?

**Data Access:**
- [ ] Demo N+1 problem và 3 cách fix?
- [ ] Giải thích 6 Propagation types?
- [ ] Phân biệt 4 Isolation Levels?

**Security:**
- [ ] Vẽ Security Filter Chain?
- [ ] Giải thích JWT structure và verification flow?
- [ ] Access Token vs Refresh Token rotation?

**Testing:**
- [ ] Phân biệt 3 slice test annotations?
- [ ] Khi nào dùng @Mock vs @MockBean?
- [ ] Testcontainers vs H2 — trade-offs?

**Architecture:**
- [ ] Khi nào migrate sang microservices?
- [ ] Giải thích Circuit Breaker 3 states?
- [ ] CQRS — khi nào áp dụng?

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
