# Spring GraphQL — API Linh Hoạt

> **GraphQL** là ngôn ngữ truy vấn API do Facebook phát triển, cho phép client chỉ định chính xác dữ liệu cần lấy — không over-fetching (lấy dư dữ liệu) và không under-fetching (lấy thiếu dữ liệu). **Spring for GraphQL** (từ Spring 5.3+) tích hợp GraphQL Java với Spring Boot, hỗ trợ schema-first approach (tiếp cận schema trước).

---

## 📋 Mục Lục

1. [GraphQL vs REST — Khi Nào Dùng Cái Nào?](#graphql-vs-rest--khi-nào-dùng-cái-nào)
2. [Schema Definition Language — SDL](#schema-definition-language--sdl)
3. [Thiết Lập Spring for GraphQL](#thiết-lập-spring-for-graphql)
4. [Controllers & Data Fetchers](#controllers--data-fetchers)
5. [Mutations — Thay Đổi Dữ Liệu](#mutations--thay-đổi-dữ-liệu)
6. [N+1 Problem & DataLoader](#n1-problem--dataloader)
7. [Subscriptions — Dữ Liệu Thời Gian Thực](#subscriptions--dữ-liệu-thời-gian-thực)
8. [Error Handling — Xử Lý Lỗi](#error-handling--xử-lý-lỗi)
9. [Security — Bảo Mật](#security--bảo-mật)
10. [Pagination — Phân Trang](#pagination--phân-trang)
11. [Testing — Kiểm Thử](#testing--kiểm-thử)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## GraphQL vs REST — Khi Nào Dùng Cái Nào?

### Vấn Đề Của REST

```
VẤN ĐỀ OVER-FETCHING (Lấy Dư):
REST: GET /api/users/1
→ { id, name, email, phone, address, createdAt, updatedAt, ... }
Client chỉ cần name và email nhưng nhận cả object

VẤN ĐỀ UNDER-FETCHING (Lấy Thiếu):
Để hiển thị User Profile đầy đủ:
1. GET /api/users/1          → user info
2. GET /api/users/1/orders   → order list
3. GET /api/orders/123/items → order items
→ 3 round trips, N+1 requests

GRAPHQL GIẢI QUYẾT:
query {
  user(id: 1) {
    name
    email
    orders {
      id
      total
      items { name, price }
    }
  }
}
→ 1 request, chính xác dữ liệu cần
```

```
                    REST                    GRAPHQL
                    ────                    ───────
Endpoint:           Nhiều (/users, /orders) 1 endpoint (/graphql)
Data Shape:         Fixed (server decides)  Flexible (client decides)
Over-fetching:      Thường xảy ra           Không có
Under-fetching:     Thường xảy ra           Không có
Versioning:         /v1, /v2                Schema evolution, @deprecated
Caching:            HTTP cache tự nhiên     Cần custom caching
Real-time:          WebSocket riêng         Subscriptions built-in
Documentation:      OpenAPI riêng           Schema IS documentation

DÙNG GRAPHQL KHI:
✅ Mobile BFF (Backend For Frontend) — bandwidth quan trọng
✅ Client cần data từ nhiều nguồn khác nhau
✅ Nhanh prototyping, thay đổi UI thường xuyên
✅ Public API với nhiều loại client khác nhau

DÙNG REST KHI:
✅ Simple CRUD API
✅ File upload/download
✅ HTTP caching là ưu tiên
✅ Team chưa quen với GraphQL ecosystem
```

---

## Schema Definition Language — SDL

```graphql
# src/main/resources/graphql/schema.graphqls

# Query — đọc dữ liệu
type Query {
    user(id: ID!): User
    users(filter: UserFilter, pagination: PaginationInput): UserPage!
    currentUser: User
}

# Mutation — thay đổi dữ liệu
type Mutation {
    createUser(input: CreateUserInput!): UserPayload!
    updateUser(id: ID!, input: UpdateUserInput!): UserPayload!
    deleteUser(id: ID!): Boolean!
}

# Subscription — dữ liệu thời gian thực
type Subscription {
    userCreated: User!
    orderStatusChanged(orderId: ID!): Order!
}

# Object types
type User {
    id: ID!
    name: String!
    email: String!
    age: Int
    role: UserRole!
    orders: [Order!]!
    createdAt: String!
}

type Order {
    id: ID!
    status: OrderStatus!
    total: Float!
    items: [OrderItem!]!
    user: User!
}

type OrderItem {
    id: ID!
    productName: String!
    quantity: Int!
    price: Float!
}

# Input types — chỉ dùng cho arguments
input CreateUserInput {
    name: String!
    email: String!
    age: Int
}

input UpdateUserInput {
    name: String
    email: String
}

input UserFilter {
    name: String
    email: String
    role: UserRole
}

input PaginationInput {
    page: Int = 0
    size: Int = 20
}

# Payload — bao gồm cả lỗi
type UserPayload {
    user: User
    errors: [UserError!]
}

type UserError {
    field: String!
    message: String!
}

# Pagination (Cursor-based — tốt hơn offset-based)
type UserPage {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type UserEdge {
    node: User!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}

# Enums
enum UserRole {
    ADMIN
    USER
    MODERATOR
}

enum OrderStatus {
    PENDING
    PROCESSING
    SHIPPED
    DELIVERED
    CANCELLED
}
```

---

## Thiết Lập Spring for GraphQL

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<!-- Với WebMVC (blocking) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- Hoặc WebFlux (reactive) -->
<!-- <dependency>spring-boot-starter-webflux</dependency> -->
```

```yaml
# application.yml
spring:
  graphql:
    graphiql:
      enabled: true        # GraphiQL IDE tại /graphiql (chỉ dev)
      path: /graphiql
    path: /graphql         # GraphQL endpoint
    schema:
      locations: classpath:graphql/    # Thư mục chứa .graphqls files
      file-extensions: .graphqls, .gqls
    websocket:
      path: /graphql       # WebSocket endpoint cho Subscriptions
```

---

## Controllers & Data Fetchers

```java
@Controller  // Không phải @RestController
public class UserController {

    private final UserService userService;

    // Query — @QueryMapping cho root field trong type Query
    @QueryMapping
    public User user(@Argument Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    @QueryMapping
    public UserPage users(@Argument UserFilter filter,
                          @Argument PaginationInput pagination) {
        return userService.findAll(filter, pagination);
    }

    // Nested field resolver — @SchemaMapping cho field trong type
    @SchemaMapping(typeName = "User", field = "orders")
    public List<Order> getOrders(User user) {
        // Gọi cho MỖI user → N+1 problem!
        // Xem DataLoader section để giải quyết
        return orderService.findByUserId(user.getId());
    }

    // Shortcut: @SchemaMapping khi method name = field name
    @SchemaMapping
    public String createdAt(User user) {
        return user.getCreatedAt().toString();
    }

    // Lấy thông tin user hiện tại từ security context
    @QueryMapping
    public User currentUser(@AuthenticationPrincipal UserPrincipal principal) {
        return userService.findById(principal.getId())
            .orElseThrow();
    }
}
```

---

## Mutations — Thay Đổi Dữ Liệu

```java
@Controller
public class UserMutationController {

    private final UserService userService;

    @MutationMapping
    public UserPayload createUser(@Argument @Valid CreateUserInput input,
                                  BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            List<UserError> errors = bindingResult.getFieldErrors().stream()
                .map(fe -> new UserError(fe.getField(), fe.getDefaultMessage()))
                .toList();
            return UserPayload.withErrors(errors);
        }

        try {
            User user = userService.create(input);
            return UserPayload.success(user);
        } catch (EmailAlreadyExistsException e) {
            return UserPayload.withErrors(List.of(
                new UserError("email", "Email đã được sử dụng")
            ));
        }
    }

    @MutationMapping
    public UserPayload updateUser(@Argument Long id,
                                  @Argument UpdateUserInput input) {
        User updated = userService.update(id, input);
        return UserPayload.success(updated);
    }

    @MutationMapping
    public boolean deleteUser(@Argument Long id) {
        userService.delete(id);
        return true;
    }
}
```

---

## N+1 Problem & DataLoader

### Vấn Đề N+1 Trong GraphQL

```graphql
# Query này gây N+1 problem:
query {
  users {        # 1 query → lấy 100 users
    id
    name
    orders {     # 100 queries → mỗi user 1 query orders!
      id
      total
    }
  }
}
```

```
N+1 Problem:
SELECT * FROM users;                    -- 1 query
SELECT * FROM orders WHERE user_id = 1; -- 1 query per user
SELECT * FROM orders WHERE user_id = 2;
SELECT * FROM orders WHERE user_id = 3;
... (100 queries tổng cộng)
```

### Giải Quyết với DataLoader

**DataLoader** (Trình Tải Dữ Liệu) là pattern batching (gom nhóm) — thu thập tất cả IDs cần load, rồi load 1 lần duy nhất.

```java
// 1. Định nghĩa BatchLoader — hàm load nhiều items cùng lúc
@Component
public class OrdersBatchLoader
        implements BatchLoaderWithContext<Long, List<Order>> {

    private final OrderRepository orderRepository;

    @Override
    public CompletionStage<List<List<Order>>> load(
            List<Long> userIds,
            BatchLoaderEnvironment environment) {

        // Load tất cả orders cho tất cả userIds trong 1 query
        Map<Long, List<Order>> ordersByUserId = orderRepository
            .findByUserIdIn(userIds)
            .stream()
            .collect(Collectors.groupingBy(Order::getUserId));

        // Trả về theo đúng thứ tự userIds
        return CompletableFuture.completedFuture(
            userIds.stream()
                .map(id -> ordersByUserId.getOrDefault(id, Collections.emptyList()))
                .toList()
        );
    }
}

// 2. Đăng ký DataLoaderRegistrar
@Configuration
public class GraphQLConfig {

    @Bean
    public RuntimeWiringConfigurer runtimeWiringConfigurer(
            OrdersBatchLoader ordersLoader) {
        return wiringBuilder -> wiringBuilder
            .type("User", typeWiring -> typeWiring
                .dataFetcher("orders", environment -> {
                    DataLoader<Long, List<Order>> loader =
                        environment.getDataLoader("ordersLoader");
                    User user = environment.getSource();
                    return loader.load(user.getId());
                })
            );
    }

    @Bean
    public DataLoaderRegistrar ordersDataLoaderRegistrar(
            OrdersBatchLoader batchLoader) {
        return DataLoaderRegistrar.ofBatchLoader("ordersLoader", batchLoader);
    }
}

// Hoặc đơn giản hơn với @BatchMapping (Spring GraphQL 1.1+)
@Controller
public class UserController {

    // @BatchMapping tự động tạo DataLoader!
    @BatchMapping(typeName = "User", field = "orders")
    public Map<User, List<Order>> getOrdersForUsers(List<User> users) {
        List<Long> userIds = users.stream().map(User::getId).toList();

        // 1 query thay vì N queries
        Map<Long, List<Order>> ordersByUserId = orderRepository
            .findByUserIdIn(userIds)
            .stream()
            .collect(Collectors.groupingBy(Order::getUserId));

        return users.stream()
            .collect(Collectors.toMap(
                user -> user,
                user -> ordersByUserId.getOrDefault(user.getId(), List.of())
            ));
    }
}
```

```
VỚI DATALOADER:
SELECT * FROM users;                          -- 1 query
SELECT * FROM orders WHERE user_id IN (1,2,...,100); -- 1 query (batched)
TỔNG: 2 queries thay vì 101 queries ✅
```

---

## Subscriptions — Dữ Liệu Thời Gian Thực

```java
// Subscription controller
@Controller
public class UserSubscriptionController {

    private final Sinks.Many<User> userCreatedSink =
        Sinks.many().multicast().onBackpressureBuffer();

    @SubscriptionMapping
    public Flux<User> userCreated() {
        return userCreatedSink.asFlux();
    }

    // Khi user được tạo, publish vào sink
    public void notifyUserCreated(User user) {
        userCreatedSink.tryEmitNext(user);
    }
}

// Service publish event khi tạo user
@Service
public class UserService {

    private final UserSubscriptionController subscriptionController;

    @Transactional
    public User createUser(CreateUserInput input) {
        User user = userRepository.save(new User(input));
        subscriptionController.notifyUserCreated(user);
        return user;
    }
}
```

```graphql
# Client subscription query
subscription {
  userCreated {
    id
    name
    email
  }
}
```

---

## Error Handling — Xử Lý Lỗi

```java
// Cách 1: Throw exception — Spring tự convert
@QueryMapping
public User user(@Argument Long id) {
    return userService.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// Custom exception
public class UserNotFoundException extends RuntimeException {
    private final Long userId;

    public UserNotFoundException(Long userId) {
        super("User " + userId + " không tồn tại");
        this.userId = userId;
    }
}

// Exception resolver — format lỗi GraphQL
@Component
public class CustomExceptionResolver implements DataFetcherExceptionResolver {

    @Override
    public Mono<List<GraphQLError>> resolveException(
            Throwable exception,
            ErrorHandlerParameters parameters) {

        if (exception instanceof UserNotFoundException e) {
            GraphQLError error = GraphQLError.newError()
                .errorType(ErrorType.NOT_FOUND)
                .message(e.getMessage())
                .location(parameters.getField().getSingleField().getSourceLocation())
                .path(parameters.getPath())
                .extension("userId", e.getUserId())
                .build();
            return Mono.just(List.of(error));
        }

        if (exception instanceof ValidationException e) {
            GraphQLError error = GraphQLError.newError()
                .errorType(ErrorType.BAD_REQUEST)
                .message(e.getMessage())
                .build();
            return Mono.just(List.of(error));
        }

        return Mono.empty(); // Xử lý mặc định
    }
}
```

---

## Security — Bảo Mật

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/graphql").authenticated()
                .requestMatchers("/graphiql/**").permitAll() // Chỉ dev
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}

// Field-level security với @PreAuthorize
@Controller
public class AdminController {

    @QueryMapping
    @PreAuthorize("hasRole('ADMIN')")  // Chỉ ADMIN được gọi
    public List<User> allUsersAdmin() {
        return userService.findAll();
    }

    // Hoặc kiểm tra trong resolver
    @SchemaMapping(typeName = "User", field = "email")
    public String getEmail(User user,
                           @AuthenticationPrincipal UserPrincipal principal) {
        // Chỉ cho phép xem email của chính mình hoặc ADMIN
        if (!principal.getId().equals(user.getId()) && !principal.isAdmin()) {
            return "***@***.***";  // Ẩn email
        }
        return user.getEmail();
    }
}
```

---

## Pagination — Phân Trang

### Cursor-Based Pagination (Tốt Hơn Offset)

```java
// Cursor pagination — tốt cho infinite scroll, ổn định khi data thay đổi
@QueryMapping
public UserPage users(@Argument UserFilter filter,
                      @Argument PaginationInput pagination) {

    String decodedCursor = pagination.getAfter() != null
        ? new String(Base64.getDecoder().decode(pagination.getAfter()))
        : null;

    Page<User> page = userRepository.findAll(
        buildSpec(filter, decodedCursor),
        PageRequest.of(0, pagination.getFirst(), Sort.by("id").ascending())
    );

    List<UserEdge> edges = page.getContent().stream()
        .map(user -> UserEdge.builder()
            .node(user)
            .cursor(Base64.getEncoder().encodeToString(user.getId().toString().getBytes()))
            .build())
        .toList();

    PageInfo pageInfo = PageInfo.builder()
        .hasNextPage(page.hasNext())
        .hasPreviousPage(decodedCursor != null)
        .startCursor(edges.isEmpty() ? null : edges.get(0).getCursor())
        .endCursor(edges.isEmpty() ? null : edges.get(edges.size() - 1).getCursor())
        .build();

    return UserPage.builder()
        .edges(edges)
        .pageInfo(pageInfo)
        .totalCount((int) page.getTotalElements())
        .build();
}
```

---

## Testing — Kiểm Thử

```java
@GraphQlTest(UserController.class)  // Chỉ load GraphQL context
class UserControllerTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @MockBean
    private UserService userService;

    @Test
    void testGetUser() {
        when(userService.findById(1L))
            .thenReturn(Optional.of(new User(1L, "Nguyen Van A", "a@test.com")));

        graphQlTester.document("""
            query {
              user(id: 1) {
                id
                name
                email
              }
            }
            """)
            .execute()
            .path("user.id").entity(Long.class).isEqualTo(1L)
            .path("user.name").entity(String.class).isEqualTo("Nguyen Van A");
    }

    @Test
    void testGetUserNotFound() {
        when(userService.findById(99L)).thenReturn(Optional.empty());

        graphQlTester.document("""
            query {
              user(id: 99) { id }
            }
            """)
            .execute()
            .errors()
            .satisfy(errors -> {
                assertThat(errors).hasSize(1);
                assertThat(errors.get(0).getErrorType()).isEqualTo(ErrorType.NOT_FOUND);
            });
    }

    @Test
    void testCreateUser() {
        when(userService.create(any()))
            .thenReturn(new User(1L, "Test User", "test@example.com"));

        graphQlTester.document("""
            mutation {
              createUser(input: { name: "Test User", email: "test@example.com" }) {
                user {
                  id
                  name
                }
                errors { field message }
              }
            }
            """)
            .execute()
            .path("createUser.user.name").entity(String.class).isEqualTo("Test User")
            .path("createUser.errors").entityList(Object.class).hasSize(0);
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Over-fetching và under-fetching là gì? GraphQL giải quyết như thế nào?**

A: Over-fetching là API trả về nhiều dữ liệu hơn client cần (lãng phí bandwidth). Under-fetching là 1 endpoint không đủ dữ liệu, cần gọi thêm endpoint khác (nhiều round trips). GraphQL giải quyết bằng cách để client khai báo chính xác fields cần thiết trong query — server chỉ trả về đúng dữ liệu đó.

---

**Q: N+1 problem trong GraphQL là gì? DataLoader giải quyết thế nào?**

A: Khi resolve field phức tạp (ví dụ `users.orders`), GraphQL gọi resolver cho mỗi user riêng lẻ → 1 query users + N queries orders. DataLoader giải quyết bằng batching: thu thập tất cả userIds trong một execution, rồi load orders cho tất cả bằng `IN` query duy nhất. `@BatchMapping` trong Spring GraphQL tự động tạo DataLoader.

---

**Q: Sự khác biệt giữa Query, Mutation và Subscription?**

A: **Query** đọc dữ liệu (idempotent, có thể cached). **Mutation** thay đổi dữ liệu (create, update, delete). **Subscription** là long-lived connection (thường WebSocket) nhận updates thời gian thực khi server có sự kiện mới.

---

**Q: Schema-first vs Code-first trong GraphQL — bạn dùng approach nào?**

A: **Schema-first** (tiếp cận schema trước): viết `.graphqls` file trước, rồi implement resolvers — Spring GraphQL theo hướng này. Ưu điểm: schema là nguồn sự thật, team backend/frontend có thể làm song song. **Code-first**: generate schema từ annotations Java (Netflix DGS, graphql-java-kickstart). Ưu điểm: type-safe hơn trong Java. Spring GraphQL chọn schema-first vì tách biệt rõ contract và implementation.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
