# Reactive Programming — Spring WebFlux & Project Reactor

> **Reactive Programming** (Lập Trình Phản Ứng) là mô hình lập trình bất đồng bộ, non-blocking (không chặn luồng) xoay quanh luồng dữ liệu và truyền tải sự thay đổi. Spring WebFlux là module web phản ứng của Spring, xây dựng trên **Project Reactor** và tuân theo đặc tả **Reactive Streams** (Luồng Phản Ứng).

---

## 📋 Mục Lục

1. [Tại Sao Cần Reactive?](#tại-sao-cần-reactive)
2. [Reactive Streams Specification](#reactive-streams-specification)
3. [Project Reactor — Mono & Flux](#project-reactor--mono--flux)
4. [Operators Quan Trọng](#operators-quan-trọng)
5. [Spring WebFlux — Cấu Trúc](#spring-webflux--cấu-trúc)
6. [Annotated Controllers vs Functional Endpoints](#annotated-controllers-vs-functional-endpoints)
7. [WebClient — Thay Thế RestTemplate](#webclient--thay-thế-resttemplate)
8. [R2DBC — Reactive Database Access](#r2dbc--reactive-database-access)
9. [Reactive Security](#reactive-security)
10. [Backpressure — Áp Lực Ngược](#backpressure--áp-lực-ngược)
11. [Testing Reactive Code](#testing-reactive-code)
12. [WebFlux vs Spring MVC — Khi Nào Dùng Cái Nào?](#webflux-vs-spring-mvc--khi-nào-dùng-cái-nào)
13. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Reactive?

### Vấn Đề Của Blocking I/O (I/O Chặn Luồng)

```
BLOCKING MODEL (Mô Hình Chặn):
─────────────────────────────

Thread 1 ──► [Request] ──► [DB Query 200ms] ──────────────────► [Response]
               │                │                                    │
               │         Thread BỊ BLOCK                            │
               │         (không làm gì cả)                          │
               │                │                                    │
              CPU               │                                   CPU
              rảnh     ─────────────────────────                   làm việc

Vấn đề: 500 concurrent requests → cần 500 threads
        Mỗi thread tốn ~1MB RAM → 500MB chỉ cho threads
        Thread context switching (Chuyển Đổi Ngữ Cảnh) tốn CPU
```

```
REACTIVE NON-BLOCKING MODEL (Mô Hình Không Chặn Phản Ứng):
───────────────────────────────────────────────────────────

Event Loop Thread ──► [Request 1] ──► [Register DB callback] ──► [Request 2] ──► ...
                                              │
                                    [DB trả kết quả]
                                              │
                             Event Loop ─────► [Xử lý kết quả Request 1]

Lợi ích: 10 threads phục vụ 10,000 concurrent requests
         CPU không idle — luôn xử lý task khác trong khi đợi I/O
```

### Khi Nào Reactive Hiệu Quả?

| Tình Huống | Reactive | Blocking |
|-----------|---------|---------|
| Nhiều concurrent connections (1000+) | ✅ | ❌ |
| I/O-intensive (gọi nhiều service) | ✅ | ❌ |
| CPU-intensive (tính toán nặng) | ❌ | ✅ |
| CRUD API đơn giản với JPA | ❌ | ✅ |
| Streaming data (SSE, WebSocket) | ✅ | ❌ |
| API Gateway / BFF | ✅ | Tùy |

---

## Reactive Streams Specification

**Reactive Streams** là đặc tả chuẩn (4 interface) để xử lý luồng dữ liệu bất đồng bộ với backpressure.

```java
// Publisher (Nhà Xuất Bản) — nguồn phát dữ liệu
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// Subscriber (Người Đăng Ký) — nhận dữ liệu
public interface Subscriber<T> {
    void onSubscribe(Subscription s);   // Được gọi khi đăng ký thành công
    void onNext(T t);                   // Nhận từng item
    void onError(Throwable t);          // Xử lý lỗi
    void onComplete();                  // Hoàn thành
}

// Subscription (Đăng Ký) — kiểm soát luồng dữ liệu (backpressure)
public interface Subscription {
    void request(long n);  // Yêu cầu n items tiếp theo
    void cancel();         // Hủy đăng ký
}

// Processor (Bộ Xử Lý) — vừa là Publisher vừa là Subscriber
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

```
Luồng Dữ Liệu Reactive Streams:

Publisher ──onSubscribe()──► Subscriber
    │                              │
    │◄───────request(n)────────────┘   ← Backpressure: Subscriber kiểm soát tốc độ
    │
    ├──onNext(item1)──────────────►
    ├──onNext(item2)──────────────►
    ├──onNext(item3)──────────────►
    └──onComplete() / onError()──►
```

---

## Project Reactor — Mono & Flux

**Project Reactor** là implementation của Reactive Streams được Spring chọn làm nền tảng.

### Mono — 0 hoặc 1 phần tử

```java
import reactor.core.publisher.Mono;

// Tạo Mono
Mono<String> mono1 = Mono.just("Hello");                    // Mono có 1 giá trị
Mono<String> mono2 = Mono.empty();                          // Mono rỗng
Mono<String> mono3 = Mono.error(new RuntimeException("Lỗi")); // Mono lỗi
Mono<String> mono4 = Mono.fromCallable(() -> fetchFromDB()); // Lazy evaluation

// Chuyển đổi
Mono<Integer> length = mono1.map(s -> s.length());          // Đồng bộ transform
Mono<String> upper  = mono1.map(String::toUpperCase);

// FlatMap — transform sang Mono khác (async)
Mono<User> user = Mono.just(1L)
    .flatMap(id -> userRepository.findById(id));            // userRepository trả Mono<User>

// Subscribe (đăng ký — trigger thực thi)
mono1.subscribe(
    value   -> System.out.println("Nhận: " + value),
    error   -> System.err.println("Lỗi: " + error),
    ()      -> System.out.println("Hoàn thành")
);
```

### Flux — 0 đến N phần tử

```java
import reactor.core.publisher.Flux;

// Tạo Flux
Flux<Integer> flux1 = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> flux2 = Flux.range(1, 100);                   // 1 đến 100
Flux<String>  flux3 = Flux.fromIterable(Arrays.asList("a", "b", "c"));
Flux<Long>    flux4 = Flux.interval(Duration.ofSeconds(1)); // Phát 1 item/giây

// Tạo từ luồng bất đồng bộ
Flux<String> flux5 = Flux.create(sink -> {
    for (String item : getItems()) {
        sink.next(item);
    }
    sink.complete();
});

// Chuyển đổi
Flux<String> result = flux1
    .filter(n -> n % 2 == 0)          // Lọc số chẵn
    .map(n -> "Item-" + n)            // Transform
    .take(5);                          // Chỉ lấy 5 item đầu

// Xử lý bất đồng bộ mỗi item
Flux<User> users = Flux.just(1L, 2L, 3L)
    .flatMap(id -> userRepository.findById(id));  // Gọi song song
    // .concatMap(...)  — gọi tuần tự, giữ thứ tự
```

---

## Operators Quan Trọng

### Chuyển Đổi Dữ Liệu

```java
Flux<Integer> numbers = Flux.range(1, 10);

// map — transform đồng bộ (1:1)
Flux<String> strings = numbers.map(n -> "Number " + n);

// flatMap — transform bất đồng bộ (1:N, không giữ thứ tự)
Flux<Order> orders = Flux.just(user1, user2)
    .flatMap(user -> orderRepository.findByUserId(user.getId()));

// concatMap — như flatMap nhưng giữ thứ tự, tuần tự
Flux<Order> orderedOrders = Flux.just(user1, user2)
    .concatMap(user -> orderRepository.findByUserId(user.getId()));

// switchMap — hủy flatMap trước khi emit item mới (tìm kiếm live)
Flux<SearchResult> results = searchInput
    .switchMap(query -> searchService.search(query));
```

### Lọc & Giới Hạn

```java
Flux<Integer> flux = Flux.range(1, 100);

flux.filter(n -> n > 50)          // Lọc điều kiện
    .take(5)                       // Lấy tối đa 5 item
    .skip(2)                       // Bỏ qua 2 item đầu
    .distinct()                    // Loại bỏ trùng lặp
    .distinctUntilChanged()        // Loại bỏ trùng lặp liền kề
    .first()                       // Lấy item đầu tiên (trả Mono)
    .last();                       // Lấy item cuối (trả Mono)
```

### Xử Lý Lỗi

```java
Mono<User> user = userRepository.findById(id)
    // Trả về giá trị mặc định nếu lỗi
    .onErrorReturn(new User("default"))

    // Chuyển sang Mono khác nếu lỗi
    .onErrorResume(ex -> cacheService.getUser(id))

    // Chuyển đổi loại lỗi
    .onErrorMap(DataAccessException.class,
                ex -> new ServiceException("DB error", ex))

    // Retry — thử lại tối đa 3 lần
    .retry(3)

    // Retry với điều kiện và backoff
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                   .filter(ex -> ex instanceof TimeoutException));
```

### Kết Hợp Luồng

```java
// zip — kết hợp từng cặp item
Mono<String> combined = Mono.zip(
    userService.findById(1L),
    orderService.findLatestByUserId(1L),
    (user, order) -> user.getName() + ": " + order.getId()
);

// merge — kết hợp nhiều Flux, không giữ thứ tự
Flux<Event> allEvents = Flux.merge(
    userEventStream,
    orderEventStream,
    paymentEventStream
);

// concat — kết hợp tuần tự, giữ thứ tự
Flux<User> allUsers = Flux.concat(
    userRepository.findByCountry("VN"),
    userRepository.findByCountry("SG")
);

// combineLatest — kết hợp item mới nhất từ mỗi luồng
Flux<String> prices = Flux.combineLatest(
    stockAFlux, stockBFlux,
    (a, b) -> "A=" + a + ", B=" + b
);
```

### Tối Ưu Hiệu Năng

```java
// buffer — nhóm items thành batch
Flux.range(1, 100)
    .buffer(10)                    // Nhóm 10 items
    .subscribe(batch -> processBatch(batch));

// window — tạo sub-Flux
Flux.interval(Duration.ofMillis(100))
    .window(Duration.ofSeconds(1)) // Cửa sổ 1 giây
    .flatMap(window -> window.count());

// publishOn — chuyển sang scheduler khác để xử lý
Flux.range(1, 10)
    .publishOn(Schedulers.boundedElastic())  // CPU-intensive hoặc blocking
    .map(n -> heavyCompute(n));

// subscribeOn — scheduler cho source
Flux.fromCallable(() -> blockingDBCall())
    .subscribeOn(Schedulers.boundedElastic()); // Không block event loop
```

---

## Spring WebFlux — Cấu Trúc

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Kiến Trúc WebFlux

```
Client Request
      │
      ▼
Netty (Non-blocking Server — Máy Chủ Không Chặn)
      │
      ▼
DispatcherHandler (tương đương DispatcherServlet trong MVC)
      │
      ▼
HandlerMapping ──► HandlerAdapter ──► Handler (Controller / RouterFunction)
                                            │
                                            ▼
                                     Mono/Flux<Response>
                                            │
                                            ▼
                                     HttpMessageWriter
                                            │
                                            ▼
                                       Client Response
```

---

## Annotated Controllers vs Functional Endpoints

### Cách 1: Annotated Controllers (Giống Spring MVC)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserRepository userRepository;

    @GetMapping
    public Flux<UserResponse> getAllUsers() {
        return userRepository.findAll()
            .map(UserResponse::from);
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<UserResponse>> getUserById(@PathVariable Long id) {
        return userRepository.findById(id)
            .map(user -> ResponseEntity.ok(UserResponse.from(user)))
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<UserResponse> createUser(@Valid @RequestBody Mono<CreateUserRequest> request) {
        return request
            .flatMap(req -> {
                User user = new User(req.getName(), req.getEmail());
                return userRepository.save(user);
            })
            .map(UserResponse::from);
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<UserEvent> streamUsers() {
        // SSE — Server-Sent Events (Sự Kiện Từ Máy Chủ)
        return userEventService.getStream();
    }
}
```

### Cách 2: Functional Endpoints (Router Functions)

```java
// Handler — chứa logic xử lý
@Component
public class UserHandler {

    private final UserRepository userRepository;

    public Mono<ServerResponse> getAllUsers(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(userRepository.findAll().map(UserResponse::from), UserResponse.class);
    }

    public Mono<ServerResponse> getUserById(ServerRequest request) {
        Long id = Long.valueOf(request.pathVariable("id"));
        return userRepository.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(UserResponse.from(user)))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
}

// Router — định nghĩa routes
@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users", handler::getAllUsers)
            .GET("/api/users/{id}", handler::getUserById)
            .POST("/api/users", handler::createUser)
            .build();
    }
}
```

---

## WebClient — Thay Thế RestTemplate

`WebClient` là HTTP client không đồng bộ, non-blocking thay thế `RestTemplate` (blocking).

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient orderServiceClient() {
        return WebClient.builder()
            .baseUrl("http://order-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .codecs(configurer ->
                configurer.defaultCodecs().maxInMemorySize(1024 * 1024)) // 1MB buffer
            .build();
    }
}

@Service
public class OrderService {

    private final WebClient webClient;

    public Mono<OrderDto> getOrder(Long orderId) {
        return webClient.get()
            .uri("/api/orders/{id}", orderId)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                      response -> Mono.error(new OrderNotFoundException(orderId)))
            .onStatus(HttpStatusCode::is5xxServerError,
                      response -> Mono.error(new ServiceUnavailableException()))
            .bodyToMono(OrderDto.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
    }

    // Lấy nhiều orders song song (parallel)
    public Flux<OrderDto> getOrdersByIds(List<Long> orderIds) {
        return Flux.fromIterable(orderIds)
            .flatMap(id -> getOrder(id)
                .onErrorResume(ex -> Mono.empty()), // Bỏ qua lỗi cho id này
                10) // maxConcurrency = 10
            ;
    }
}
```

---

## R2DBC — Reactive Database Access

**R2DBC** (Reactive Relational Database Connectivity — Kết Nối Cơ Sở Dữ Liệu Quan Hệ Phản Ứng) là driver non-blocking cho database quan hệ.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
  sql:
    init:
      mode: always
```

```java
// Entity với Spring Data R2DBC
@Table("users")
public class User {
    @Id
    private Long id;
    private String name;
    private String email;
    // getters/setters
}

// Repository — trả về Mono/Flux thay vì Optional/List
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByEmailContaining(String email);
    Mono<Long> countByActive(boolean active);

    @Query("SELECT * FROM users WHERE created_at > :date")
    Flux<User> findRecentUsers(LocalDateTime date);
}

// Service
@Service
@Transactional  // Với R2DBC, cần reactive transaction manager
public class UserService {

    public Mono<User> createUserWithTransaction(CreateUserRequest req) {
        return userRepository.save(new User(req.getName(), req.getEmail()))
            .flatMap(user ->
                auditLogRepository.save(new AuditLog("USER_CREATED", user.getId()))
                    .thenReturn(user)
            );
    }
}
```

> **Lưu ý:** JPA (Hibernate) không tương thích với Reactive — JPA dùng blocking JDBC.
> Khi dùng WebFlux với DB quan hệ, phải chọn R2DBC hoặc dùng `subscribeOn(Schedulers.boundedElastic())`.

---

## Reactive Security

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers(HttpMethod.GET, "/api/users").hasRole("USER")
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(Customizer.withDefaults())
            )
            .build();
    }

    // Lấy user hiện tại trong reactive context
    @GetMapping("/api/me")
    public Mono<UserDto> getCurrentUser() {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .map(auth -> (Jwt) auth.getPrincipal())
            .map(jwt -> new UserDto(jwt.getSubject(), jwt.getClaimAsString("email")));
    }
}
```

---

## Backpressure — Áp Lực Ngược

**Backpressure** là cơ chế cho phép Subscriber báo cho Publisher biết nó có thể xử lý bao nhiêu item, ngăn chặn overflow (tràn bộ nhớ).

```
KHÔNG CÓ BACKPRESSURE:
Publisher phát 10,000 items/giây
Subscriber chỉ xử lý 100 items/giây
→ Buffer tràn → OutOfMemoryError ❌

CÓ BACKPRESSURE:
Publisher phát 10,000 items/giây
Subscriber request(100) → nhận 100 items
Xử lý xong → request(100) tiếp
→ Publisher điều chỉnh tốc độ ✅
```

```java
// Chiến lược xử lý backpressure
Flux.range(1, 1_000_000)
    // DROP — bỏ qua item khi downstream không kịp
    .onBackpressureDrop(dropped -> log.warn("Bỏ qua: {}", dropped))
    // BUFFER — buffer items (nguy cơ OOM nếu chậm quá)
    // .onBackpressureBuffer(1000)
    // LATEST — chỉ giữ item mới nhất
    // .onBackpressureLatest()
    // ERROR — ném lỗi khi overflow
    // .onBackpressureError()
    .subscribe(item -> {
        Thread.sleep(10); // Giả lập xử lý chậm
        process(item);
    });
```

---

## Testing Reactive Code

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import reactor.test.StepVerifier;

@Test
void testGetAllUsers() {
    // Arrange
    when(userRepository.findAll())
        .thenReturn(Flux.just(user1, user2));

    // Act & Assert — dùng StepVerifier (Kiểm Tra Từng Bước)
    StepVerifier.create(userService.getAllUsers())
        .expectNextMatches(u -> u.getId().equals(1L))
        .expectNextMatches(u -> u.getId().equals(2L))
        .verifyComplete();
}

@Test
void testGetUserNotFound() {
    when(userRepository.findById(99L))
        .thenReturn(Mono.empty());

    StepVerifier.create(userService.getUserById(99L))
        .expectError(UserNotFoundException.class)
        .verify();
}

@Test
void testFluxWithVirtualTime() {
    // Test Flux với thời gian ảo (không cần đợi thật)
    StepVerifier.withVirtualTime(() ->
        Flux.interval(Duration.ofHours(1)).take(3)
    )
    .thenAwait(Duration.ofHours(3))
    .expectNextCount(3)
    .verifyComplete();
}

// WebTestClient — test WebFlux controllers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void testGetUsers() {
        webTestClient.get().uri("/api/users")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(UserResponse.class)
            .hasSize(2);
    }
}
```

---

## WebFlux vs Spring MVC — Khi Nào Dùng Cái Nào?

```
                    SPRING MVC                    SPRING WEBFLUX
                    ──────────                    ──────────────
Model:             Thread-per-request            Event Loop + Callbacks
Concurrency:       Hundreds of threads           Few threads, millions connections
I/O:               Blocking                      Non-blocking
DB:                JPA / JDBC                    R2DBC
HTTP Client:       RestTemplate / OpenFeign      WebClient
Debug:             Stack trace rõ ràng           Reactive stack trace khó đọc
Learning Curve:    Thấp                          Cao
Library Support:   Rất phong phú                 Đang lớn dần

CHỌN SPRING MVC KHI:
✅ Team chưa có kinh nghiệm Reactive
✅ Cần JPA/Hibernate (không muốn R2DBC)
✅ Ứng dụng CRUD với ít concurrent users
✅ Tích hợp với thư viện chỉ hỗ trợ blocking

CHỌN SPRING WEBFLUX KHI:
✅ Gateway hoặc proxy service (gọi nhiều service)
✅ Streaming data (SSE, WebSocket)
✅ Microservices với high I/O concurrency (1000+ req/s)
✅ Team đã quen với Reactive paradigm
✅ Kết hợp với Kotlin Coroutines (cú pháp đẹp hơn)
```

### Ví Dụ Benchmark (So Sánh Hiệu Năng)

```
Scenario: 5000 concurrent users, mỗi request gọi 3 external services

Spring MVC (200 threads):
  - Thread pool exhausted ở ~500 concurrent users
  - Latency tăng mạnh
  - Memory: ~512MB cho thread stacks

Spring WebFlux (16 threads):
  - Xử lý tốt 5000 concurrent users
  - Latency ổn định
  - Memory: ~64MB cho thread stacks
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa `map` và `flatMap` trong Project Reactor?**

A: `map` là transform đồng bộ 1-1, nhận `T` → trả `R`. `flatMap` là transform bất đồng bộ, nhận `T` → trả `Publisher<R>`, và tự động subscribe vào Publisher đó. Dùng `flatMap` khi transform là async (ví dụ: gọi database, HTTP request).

---

**Q: Backpressure là gì và tại sao quan trọng?**

A: Backpressure là cơ chế kiểm soát tốc độ producer không vượt quá tốc độ consumer. Quan trọng để tránh OutOfMemoryError khi producer nhanh hơn consumer. Project Reactor implement qua `Subscription.request(n)`, cho phép consumer chỉ định số lượng item muốn nhận.

---

**Q: Tại sao không nên dùng `block()` trong WebFlux?**

A: `block()` chặn event loop thread, phá vỡ mô hình non-blocking, có thể gây deadlock khi tất cả threads bị block. Nếu cần kết quả đồng bộ, hãy dùng `subscribeOn(Schedulers.boundedElastic())` để delegate cho dedicated thread pool.

---

**Q: Khi nào dùng `concatMap` thay vì `flatMap`?**

A: `flatMap` xử lý concurrently và không đảm bảo thứ tự. `concatMap` xử lý tuần tự, giữ thứ tự, phù hợp khi thứ tự output quan trọng (ví dụ: bước xử lý phụ thuộc nhau). Trade-off: `concatMap` chậm hơn `flatMap`.

---

**Q: Mô tả sự khác biệt giữa `publishOn` và `subscribeOn`?**

A: `subscribeOn` quyết định Scheduler (Bộ Lập Lịch) chạy source emission — ảnh hưởng từ đầu pipeline. `publishOn` chuyển sang Scheduler khác tại điểm đó trong pipeline — ảnh hưởng từ điểm đó trở xuống. Thường dùng `subscribeOn(Schedulers.boundedElastic())` cho blocking calls và `publishOn(Schedulers.parallel())` cho CPU-intensive work.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
