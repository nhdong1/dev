# Web Layer Testing với @WebMvcTest & MockMvc

> `@WebMvcTest` (Kiểm Thử Tầng Web) chỉ load Controller layer —
> không load Service, Repository hay database, giúp test nhanh hơn `@SpringBootTest`.
> `MockMvc` (Bộ Giả Lập MVC) cho phép gửi HTTP request giả và kiểm tra response
> mà không cần khởi động HTTP server thực.

---

## 1. Tại Sao Cần @WebMvcTest?

```
@SpringBootTest:          Load toàn bộ context → Chậm (3–10 giây)
@WebMvcTest:              Load chỉ web layer → Nhanh (1–2 giây)

@WebMvcTest chỉ load:
  ✅ @Controller, @RestController, @ControllerAdvice
  ✅ @Filter, HandlerInterceptor
  ✅ WebMvcConfigurer
  ✅ Spring Security configuration
  ✅ ArgumentResolver, MessageConverter

@WebMvcTest KHÔNG load:
  ❌ @Service, @Component, @Repository
  ❌ JPA, DataSource
  ❌ @Async, @Scheduled
  → Cần @MockBean cho tất cả phụ thuộc của Controller
```

---

## 2. Cài Đặt Cơ Bản

### Controller Cần Test

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable UUID id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody @Valid CreateOrderRequest request) {
        OrderResponse order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }

    @PutMapping("/{id}/cancel")
    public ResponseEntity<Void> cancelOrder(@PathVariable UUID id) {
        orderService.cancelOrder(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<Page<OrderResponse>> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(orderService.findAll(PageRequest.of(page, size)));
    }
}
```

### Test Class Cơ Bản

```java
@WebMvcTest(OrderController.class)  // ← Chỉ load OrderController và web infrastructure
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;  // ← Tự động được inject

    @MockBean
    private OrderService orderService;  // ← Mock service vì không load @Service

    @Autowired
    private ObjectMapper objectMapper;  // ← Dùng để serialize/deserialize JSON

    // ... test methods
}
```

---

## 3. MockMvc — Gửi Request và Kiểm Tra Response

### GET Request

```java
@Test
void getOrder_withExistingId_returnsOrder() throws Exception {
    UUID orderId = UUID.randomUUID();
    OrderResponse order = OrderResponse.builder()
        .id(orderId)
        .productId("product-1")
        .quantity(2)
        .totalPrice(100_000L)
        .status(OrderStatus.PENDING)
        .build();

    when(orderService.findById(orderId)).thenReturn(Optional.of(order));

    mockMvc.perform(
            get("/api/orders/{id}", orderId)      // ← Gửi GET request
                .accept(MediaType.APPLICATION_JSON)  // ← Accept header
        )
        .andExpect(status().isOk())               // ← Kiểm tra HTTP status 200
        .andExpect(content().contentType(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$.id").value(orderId.toString()))
        .andExpect(jsonPath("$.productId").value("product-1"))
        .andExpect(jsonPath("$.quantity").value(2))
        .andExpect(jsonPath("$.totalPrice").value(100_000))
        .andExpect(jsonPath("$.status").value("PENDING"))
        .andDo(print());  // ← In ra request/response để debug
}
```

### POST Request — Tạo Đơn Hàng

```java
@Test
void createOrder_withValidRequest_returnsCreated() throws Exception {
    CreateOrderRequest request = new CreateOrderRequest("product-1", 2, 50_000L);
    OrderResponse createdOrder = OrderResponse.builder()
        .id(UUID.randomUUID())
        .productId("product-1")
        .quantity(2)
        .totalPrice(100_000L)
        .status(OrderStatus.PENDING)
        .build();

    when(orderService.createOrder(any(CreateOrderRequest.class))).thenReturn(createdOrder);

    mockMvc.perform(
            post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))  // ← Serialize request body
        )
        .andExpect(status().isCreated())             // ← HTTP 201
        .andExpect(header().exists("Location"))      // ← Location header tồn tại
        .andExpect(jsonPath("$.id").isNotEmpty())
        .andExpect(jsonPath("$.status").value("PENDING"));
}
```

### Kiểm Tra Validation (Bean Validation)

```java
@Test
void createOrder_withMissingProductId_returnsBadRequest() throws Exception {
    // Request thiếu productId
    String invalidJson = """
        {
            "quantity": 2,
            "price": 50000
        }
        """;

    mockMvc.perform(
            post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson)
        )
        .andExpect(status().isBadRequest())          // ← HTTP 400
        .andExpect(jsonPath("$.errors").isArray())
        .andExpect(jsonPath("$.errors[0].field").value("productId"))
        .andExpect(jsonPath("$.errors[0].message").value("không được null"));

    // Xác minh service KHÔNG được gọi khi validation fail
    verify(orderService, never()).createOrder(any());
}

@ParameterizedTest
@MethodSource("provideInvalidRequests")
void createOrder_withInvalidData_returnsBadRequest(String invalidJson, String expectedField) throws Exception {
    mockMvc.perform(
            post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson)
        )
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors[?(@.field == '" + expectedField + "')]").exists());
}

static Stream<Arguments> provideInvalidRequests() {
    return Stream.of(
        Arguments.of("""{"quantity": 2}""", "productId"),
        Arguments.of("""{"productId": "p1", "quantity": 0}""", "quantity"),
        Arguments.of("""{"productId": "p1", "quantity": -1}""", "quantity")
    );
}
```

### DELETE và PUT Request

```java
@Test
void cancelOrder_withExistingOrder_returnsNoContent() throws Exception {
    UUID orderId = UUID.randomUUID();
    doNothing().when(orderService).cancelOrder(orderId);

    mockMvc.perform(put("/api/orders/{id}/cancel", orderId))
        .andExpect(status().isNoContent());  // ← HTTP 204

    verify(orderService).cancelOrder(orderId);
}

@Test
void cancelOrder_withNonExistingOrder_returnsNotFound() throws Exception {
    UUID orderId = UUID.randomUUID();
    doThrow(new OrderNotFoundException(orderId)).when(orderService).cancelOrder(orderId);

    mockMvc.perform(put("/api/orders/{id}/cancel", orderId))
        .andExpect(status().isNotFound())    // ← HTTP 404
        .andExpect(jsonPath("$.title").value("Order Not Found"))
        .andExpect(jsonPath("$.status").value(404));
}
```

### Query Parameters và Pagination (Phân Trang)

```java
@Test
void listOrders_withPaginationParams_returnsPagedResult() throws Exception {
    Page<OrderResponse> page = new PageImpl<>(
        List.of(mockOrder1, mockOrder2),
        PageRequest.of(0, 20),
        100 // tổng 100 records
    );

    when(orderService.findAll(any(PageRequest.class))).thenReturn(page);

    mockMvc.perform(
            get("/api/orders")
                .param("page", "0")
                .param("size", "20")
        )
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.content").isArray())
        .andExpect(jsonPath("$.content.length()").value(2))
        .andExpect(jsonPath("$.totalElements").value(100))
        .andExpect(jsonPath("$.totalPages").value(5));
}
```

---

## 4. ResultActions — Tái Sử Dụng Với andDo()

```java
@Test
void getOrder_logsRequestResponse() throws Exception {
    // andDo() cho phép thực hiện hành động sau khi request
    mockMvc.perform(get("/api/orders/{id}", orderId))
        .andDo(print())                    // ← In ra console để debug
        .andDo(document("get-order"))      // ← Spring REST Docs documentation
        .andExpect(status().isOk());
}
```

---

## 5. Testing Spring Security — Bảo Mật Trong Web Test

### Cấu Hình Security Cho Test

```java
@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)   // ← Import security configuration nếu cần
class OrderControllerSecurityTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private OrderService orderService;
    @MockBean private JwtService jwtService;             // ← Mock security dependencies
    @MockBean private UserDetailsService userDetailsService;

    // ============================================================
    // Dùng @WithMockUser — giả lập user đã đăng nhập
    // ============================================================

    @Test
    @WithMockUser(username = "alice", roles = {"USER"})
    void getOrder_withAuthenticatedUser_returnsOrder() throws Exception {
        when(orderService.findById(any())).thenReturn(Optional.of(mockOrder));

        mockMvc.perform(get("/api/orders/{id}", orderId))
            .andExpect(status().isOk());
    }

    @Test
    void getOrder_withoutAuthentication_returns401() throws Exception {
        mockMvc.perform(get("/api/orders/{id}", orderId))
            .andExpect(status().isUnauthorized());  // ← HTTP 401
    }

    @Test
    @WithMockUser(username = "alice", roles = {"USER"})
    void deleteOrder_withUserRole_returns403() throws Exception {
        // Endpoint yêu cầu ADMIN role
        mockMvc.perform(delete("/api/admin/orders/{id}", orderId))
            .andExpect(status().isForbidden());  // ← HTTP 403
    }

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void deleteOrder_withAdminRole_returnsNoContent() throws Exception {
        doNothing().when(orderService).deleteOrder(any());

        mockMvc.perform(delete("/api/admin/orders/{id}", orderId))
            .andExpect(status().isNoContent());
    }
}
```

### @WithMockUser Tùy Chỉnh

```java
// Tạo custom annotation cho user thường xuyên dùng
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "premium-user@example.com", roles = {"USER", "PREMIUM"})
public @interface WithPremiumUser { }

@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "admin@example.com", roles = {"ADMIN", "USER"})
public @interface WithAdminUser { }

// Sử dụng
@Test
@WithPremiumUser
void getPremiumContent_withPremiumUser_returnsContent() throws Exception { ... }

@Test
@WithAdminUser
void adminEndpoint_withAdminUser_succeeds() throws Exception { ... }
```

### JWT Bearer Token Trong Test

```java
@Test
@WithMockUser  // ← Đơn giản nhất cho hầu hết test
void withJwtToken_alternativeApproach() throws Exception {
    // Hoặc truyền JWT token thực vào header
    String jwtToken = "Bearer eyJhbGci...";  // Token hợp lệ cho test

    mockMvc.perform(
            get("/api/orders")
                .header("Authorization", jwtToken)
        )
        .andExpect(status().isOk());
}
```

---

## 6. MockMvc Request Builders

```java
// Static imports giúp code gọn hơn
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;

// ── GET ──────────────────────────────────────────────
mockMvc.perform(get("/api/users")
    .param("page", "0")
    .param("size", "10")
    .header("Accept-Language", "vi-VN")
    .accept(MediaType.APPLICATION_JSON));

// ── POST ─────────────────────────────────────────────
mockMvc.perform(post("/api/orders")
    .contentType(MediaType.APPLICATION_JSON)
    .content("""{"productId": "p-1", "quantity": 2}""")
    .header("X-Request-ID", "req-123"));

// ── PUT ──────────────────────────────────────────────
mockMvc.perform(put("/api/orders/{id}", orderId)
    .contentType(MediaType.APPLICATION_JSON)
    .content(objectMapper.writeValueAsString(updateRequest)));

// ── PATCH ────────────────────────────────────────────
mockMvc.perform(patch("/api/orders/{id}/status", orderId)
    .contentType(MediaType.APPLICATION_JSON)
    .content("""{"status": "SHIPPED"}"""));

// ── DELETE ───────────────────────────────────────────
mockMvc.perform(delete("/api/orders/{id}", orderId));

// ── Multipart Upload ─────────────────────────────────
mockMvc.perform(multipart("/api/uploads")
    .file("file", "test content".getBytes())
    .param("description", "Test file"));
```

---

## 7. ResultMatchers — Kiểm Tra Response Chi Tiết

```java
mockMvc.perform(get("/api/orders"))
    // ── Status ───────────────────────────────────────
    .andExpect(status().isOk())                // 200
    .andExpect(status().isCreated())           // 201
    .andExpect(status().isBadRequest())        // 400
    .andExpect(status().isUnauthorized())      // 401
    .andExpect(status().isForbidden())         // 403
    .andExpect(status().isNotFound())          // 404
    .andExpect(status().is5xxServerError())    // 5xx

    // ── Content ──────────────────────────────────────
    .andExpect(content().contentType(MediaType.APPLICATION_JSON))
    .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
    .andExpect(content().string(containsString("orderId")))

    // ── Headers ──────────────────────────────────────
    .andExpect(header().string("Content-Type", "application/json"))
    .andExpect(header().exists("X-Request-ID"))
    .andExpect(header().doesNotExist("X-Internal-Token"))

    // ── JSON Path ────────────────────────────────────
    .andExpect(jsonPath("$.id").isNotEmpty())
    .andExpect(jsonPath("$.id").value("abc-123"))
    .andExpect(jsonPath("$.status").value("PENDING"))
    .andExpect(jsonPath("$.items").isArray())
    .andExpect(jsonPath("$.items.length()").value(3))
    .andExpect(jsonPath("$.items[0].productId").value("p-001"))
    .andExpect(jsonPath("$.metadata").doesNotExist())

    // ── JSON Path với Hamcrest Matchers ──────────────
    .andExpect(jsonPath("$.price").value(greaterThan(0)))
    .andExpect(jsonPath("$.name").value(containsString("Laptop")));
```

---

## 8. Custom ResultMatcher — Tái Sử Dụng Assertions

```java
// ResultMatcher tái sử dụng cho ProblemDetail response
public class ProblemDetailMatchers {

    public static ResultMatcher hasErrorStatus(int status) {
        return ResultMatcher.matchAll(
            jsonPath("$.status").value(status),
            jsonPath("$.title").isNotEmpty(),
            jsonPath("$.detail").isNotEmpty(),
            jsonPath("$.timestamp").isNotEmpty()
        );
    }

    public static ResultMatcher hasValidationError(String field, String message) {
        return jsonPath("$.errors[?(@.field == '" + field + "' && @.message =~ /.*" + message + ".*/))]").exists();
    }
}

// Sử dụng
@Test
void createOrder_withInvalidInput_returnsStructuredError() throws Exception {
    mockMvc.perform(post("/api/orders").contentType(APPLICATION_JSON).content("{}"))
        .andExpect(status().isBadRequest())
        .andExpect(ProblemDetailMatchers.hasErrorStatus(400))
        .andExpect(ProblemDetailMatchers.hasValidationError("productId", "không được null"));
}
```

---

## 9. @WebMvcTest vs @SpringBootTest — So Sánh

| Đặc Điểm | `@WebMvcTest` | `@SpringBootTest(webEnv=MOCK)` |
|-----------|--------------|-------------------------------|
| **Tốc độ** | Rất nhanh (~1s) | Chậm (~5–10s) |
| **Context** | Chỉ web layer | Toàn bộ application |
| **Database** | Không load | Load (nếu có DataSource) |
| **Service** | Cần @MockBean | Load thực |
| **MockMvc** | Tự động inject | Cần @AutoConfigureMockMvc |
| **Khi dùng** | Kiểm thử Controller logic, validation, security | Kiểm thử tích hợp qua HTTP |

---

## 10. Ví Dụ Đầy Đủ — Controller Test Hoàn Chỉnh

```java
@WebMvcTest(ProductController.class)
@ActiveProfiles("test")
class ProductControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @MockBean private ProductService productService;

    // ── Helper method ────────────────────────────────
    private ProductResponse createMockProduct(String id, String name, long price) {
        return ProductResponse.builder()
            .id(id).name(name).price(price).stockQuantity(10).build();
    }

    // ── GET /api/products ─────────────────────────────
    @Test
    void listProducts_returnsPagedProducts() throws Exception {
        List<ProductResponse> products = List.of(
            createMockProduct("p-001", "Laptop", 25_000_000L),
            createMockProduct("p-002", "Mouse", 500_000L)
        );
        Page<ProductResponse> page = new PageImpl<>(products, PageRequest.of(0, 10), 2);

        when(productService.findAll(any())).thenReturn(page);

        mockMvc.perform(get("/api/products").accept(APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content.length()").value(2))
            .andExpect(jsonPath("$.content[0].name").value("Laptop"))
            .andExpect(jsonPath("$.content[1].price").value(500_000));
    }

    // ── GET /api/products/{id} — Not Found ───────────
    @Test
    void getProduct_withNonExistentId_returns404() throws Exception {
        when(productService.findById("non-existent"))
            .thenThrow(new ProductNotFoundException("non-existent"));

        mockMvc.perform(get("/api/products/non-existent"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.title").value("Product Not Found"));
    }

    // ── POST /api/products — Validation ──────────────
    @Test
    @WithMockUser(roles = "ADMIN")
    void createProduct_withInvalidPrice_returnsBadRequest() throws Exception {
        String invalidProduct = """
            {
                "name": "Test Product",
                "price": -100,
                "stockQuantity": 5
            }
            """;

        mockMvc.perform(post("/api/products")
                .contentType(APPLICATION_JSON)
                .content(invalidProduct))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[?(@.field == 'price')]").exists());

        verify(productService, never()).create(any());
    }
}
```

---

## 11. Checklist Web Layer Testing

- [ ] Mỗi endpoint có test cho: success, validation error, not found, unauthorized/forbidden
- [ ] Dùng `@WebMvcTest(MyController.class)` — chỉ định rõ controller cần test
- [ ] `@MockBean` cho tất cả phụ thuộc của Controller
- [ ] Dùng `objectMapper.writeValueAsString()` cho request body, không hardcode JSON
- [ ] Kiểm tra cả response body và HTTP status code
- [ ] Test security với `@WithMockUser` — không bỏ qua security trong test

---

## 📚 Tài Liệu Tham Khảo

- [Spring MVC Test Framework](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#spring-mvc-test-framework)
- [MockMvc Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/web/servlet/MockMvc.html)
- [JsonPath Syntax](https://github.com/json-path/JsonPath)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] `@WebMvcTest` khác `@SpringBootTest` ở điểm gì? Khi nào dùng cái nào?
- [ ] Tại sao phải dùng `@MockBean` trong `@WebMvcTest` thay vì `@Mock`?
- [ ] Cách test endpoint được bảo vệ bởi Spring Security mà không cần JWT thực?
- [ ] `MockMvc` và `TestRestTemplate` — khác biệt và khi nào dùng cái nào?
- [ ] Làm thế nào test request validation (`@Valid`, `@NotNull`) với MockMvc?
