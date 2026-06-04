# REST Controllers — @RestController, Routing & HTTP Methods

> Bài này bao gồm toàn bộ kiến thức về xây dựng REST API với Spring MVC:
> từ khai báo controller, định nghĩa route, xử lý tham số, đến trả về response đúng chuẩn.

---

## 📋 Mục Tiêu

- [ ] Phân biệt `@Controller` và `@RestController`
- [ ] Định nghĩa route với các mapping annotations
- [ ] Xử lý **Path Variable** (Biến Đường Dẫn), **Request Param** (Tham Số Query) và **Request Body** (Thân Request)
- [ ] Trả về response với đúng HTTP status code
- [ ] Tổ chức API versioning (Quản Lý Phiên Bản API)
- [ ] Áp dụng Content Negotiation (Thương Lượng Nội Dung)

---

## 1. @Controller vs @RestController

### @Controller

`@Controller` là stereotype annotation (annotation phân loại) đánh dấu class là Spring MVC Controller.
Theo mặc định, các method trả về **tên view** (để render template như Thymeleaf).

```java
@Controller
public class PageController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("title", "Trang Chủ");
        return "home"; // trả về tên view "home.html" trong templates/
    }
}
```

### @RestController

`@RestController` = `@Controller` + `@ResponseBody`.  
Mọi method tự động serialize (chuyển đổi tuần tự) return value thành JSON/XML thay vì tên view.

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // Tự động serialize UserResponse thành JSON
        return new UserResponse(id, "Nguyễn Văn A");
    }
}
```

```
@RestController
    │
    ├── @Controller         ← Đánh dấu là Spring MVC Controller
    └── @ResponseBody       ← Serialize return value thành response body
```

---

## 2. Request Mapping Annotations (Annotation Ánh Xạ Request)

### @RequestMapping — Ánh Xạ Tổng Quát

```java
@RestController
@RequestMapping("/api/v1/users")  // Prefix cho toàn bộ controller
public class UserController {

    // Tương đương: GET /api/v1/users
    @RequestMapping(method = RequestMethod.GET)
    public List<UserResponse> getAll() { ... }
}
```

### Shortcut Annotations (Annotation Rút Gọn)

Các annotation rút gọn tương ứng với từng HTTP method:

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    // GET /api/v1/products — Lấy danh sách
    @GetMapping
    public List<ProductResponse> getAll() { ... }

    // GET /api/v1/products/{id} — Lấy chi tiết
    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) { ... }

    // POST /api/v1/products — Tạo mới
    @PostMapping
    public ResponseEntity<ProductResponse> create(@RequestBody @Valid CreateProductRequest request) { ... }

    // PUT /api/v1/products/{id} — Cập nhật toàn bộ
    @PutMapping("/{id}")
    public ProductResponse update(@PathVariable Long id, @RequestBody @Valid UpdateProductRequest request) { ... }

    // PATCH /api/v1/products/{id} — Cập nhật một phần
    @PatchMapping("/{id}")
    public ProductResponse patch(@PathVariable Long id, @RequestBody @Valid PatchProductRequest request) { ... }

    // DELETE /api/v1/products/{id} — Xóa
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { ... }
}
```

---

## 3. Xử Lý Tham Số Request

### @PathVariable — Biến Đường Dẫn

Lấy giá trị từ URL path:

```java
// GET /users/42
@GetMapping("/{id}")
public UserResponse getById(@PathVariable Long id) {
    return userService.findById(id);
}

// GET /users/42/orders/7
@GetMapping("/{userId}/orders/{orderId}")
public OrderResponse getOrder(
    @PathVariable Long userId,
    @PathVariable Long orderId
) {
    return orderService.findByUserAndId(userId, orderId);
}

// Tên biến khác với tên trong URL
@GetMapping("/{user-id}")
public UserResponse getByCustomName(@PathVariable("user-id") Long userId) {
    return userService.findById(userId);
}
```

### @RequestParam — Tham Số Query String

Lấy giá trị từ query string (`?key=value`):

```java
// GET /users?page=0&size=10&sort=name
@GetMapping
public Page<UserResponse> search(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "id") String sort,
    @RequestParam(required = false) String keyword // Optional — không bắt buộc
) {
    return userService.search(keyword, PageRequest.of(page, size, Sort.by(sort)));
}

// GET /users?ids=1,2,3 — Nhận list
@GetMapping("/batch")
public List<UserResponse> getByIds(@RequestParam List<Long> ids) {
    return userService.findAllByIds(ids);
}
```

### @RequestBody — Thân Request

Deserialize (giải tuần tự) JSON body thành Java object:

```java
@PostMapping
public ResponseEntity<UserResponse> create(
    @RequestBody @Valid CreateUserRequest request  // @Valid kích hoạt Bean Validation
) {
    UserResponse created = userService.create(request);
    URI location = ServletUriComponentsBuilder
        .fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(created.id())
        .toUri();
    return ResponseEntity.created(location).body(created);
}
```

### @RequestHeader — Header HTTP

```java
@GetMapping("/profile")
public UserResponse getProfile(
    @RequestHeader("Authorization") String authHeader,           // Bắt buộc
    @RequestHeader(value = "X-Tenant-Id", required = false) String tenantId // Tùy chọn
) {
    return userService.getProfile(authHeader, tenantId);
}
```

### @CookieValue — Giá Trị Cookie

```java
@GetMapping("/session")
public SessionInfo getSession(
    @CookieValue(value = "JSESSIONID", required = false) String sessionId
) {
    return sessionService.getInfo(sessionId);
}
```

---

## 4. ResponseEntity — Kiểm Soát Response Toàn Diện

`ResponseEntity<T>` cho phép kiểm soát đầy đủ: status code, headers và body.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // 201 Created với Location header
    @PostMapping
    public ResponseEntity<UserResponse> create(@RequestBody @Valid CreateUserRequest request) {
        UserResponse user = userService.create(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)                 // 201
            .header("X-User-Id", user.id().toString())  // Custom header
            .body(user);
    }

    // 200 OK với body
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)                     // 200 nếu tìm thấy
            .orElse(ResponseEntity.notFound().build());  // 404 nếu không tìm thấy
    }

    // 204 No Content — không có body
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();  // 204
    }

    // 200 với cache headers
    @GetMapping("/{id}/avatar")
    public ResponseEntity<byte[]> getAvatar(@PathVariable Long id) {
        byte[] image = userService.getAvatar(id);
        return ResponseEntity.ok()
            .contentType(MediaType.IMAGE_PNG)
            .cacheControl(CacheControl.maxAge(7, TimeUnit.DAYS))
            .body(image);
    }
}
```

### @ResponseStatus — Khai Báo Status Code Mặc Định

Nếu không cần custom headers, dùng `@ResponseStatus` cho gọn hơn:

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)  // Luôn trả 201
public UserResponse create(@RequestBody @Valid CreateUserRequest request) {
    return userService.create(request);
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)  // Luôn trả 204
public void delete(@PathVariable Long id) {
    userService.delete(id);
}
```

---

## 5. API Versioning — Quản Lý Phiên Bản API

### Chiến Lược 1: URL Path Versioning (Phiên Bản Trong URL) — Phổ Biến Nhất

```java
// Version 1
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {

    @GetMapping("/{id}")
    public UserResponseV1 getUser(@PathVariable Long id) {
        return userService.findByIdV1(id);
    }
}

// Version 2 — có thêm trường mới
@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {

    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) {
        return userService.findByIdV2(id); // V2 có thêm avatarUrl, phone...
    }
}
```

### Chiến Lược 2: Header Versioning (Phiên Bản Qua Header)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // Accept: application/vnd.app.v1+json
    @GetMapping(value = "/{id}", headers = "API-Version=1")
    public UserResponseV1 getUserV1(@PathVariable Long id) { ... }

    // Accept: application/vnd.app.v2+json
    @GetMapping(value = "/{id}", headers = "API-Version=2")
    public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
}
```

### Chiến Lược 3: Request Param Versioning

```java
// GET /api/users/1?version=1
@GetMapping(value = "/{id}", params = "version=1")
public UserResponseV1 getUserV1(@PathVariable Long id) { ... }

// GET /api/users/1?version=2
@GetMapping(value = "/{id}", params = "version=2")
public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
```

> **Khuyến Nghị:** Dùng **URL Path Versioning** — rõ ràng nhất, dễ test với curl/Postman, dễ cache.

---

## 6. Content Negotiation — Thương Lượng Nội Dung

Content Negotiation cho phép cùng một endpoint trả về định dạng khác nhau (JSON, XML) dựa vào `Accept` header.

```java
// pom.xml — thêm dependency cho XML support
// <dependency>
//     <groupId>com.fasterxml.jackson.dataformat</groupId>
//     <artifactId>jackson-dataformat-xml</artifactId>
// </dependency>

@RestController
@RequestMapping("/api/v1/reports")
public class ReportController {

    // GET /api/v1/reports/123
    // Accept: application/json  → trả JSON
    // Accept: application/xml   → trả XML
    @GetMapping(
        value = "/{id}",
        produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE }
    )
    public ReportResponse getReport(@PathVariable Long id) {
        return reportService.findById(id);
    }
}
```

---

## 7. Ví Dụ Hoàn Chỉnh — CRUD API

```java
@RestController
@RequestMapping("/api/v1/articles")
@RequiredArgsConstructor  // Lombok — tạo constructor cho final fields
public class ArticleController {

    private final ArticleService articleService;

    // GET /api/v1/articles?page=0&size=20&category=tech
    @GetMapping
    public Page<ArticleResponse> getAll(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(required = false) String category
    ) {
        return articleService.findAll(category, PageRequest.of(page, size));
    }

    // GET /api/v1/articles/42
    @GetMapping("/{id}")
    public ResponseEntity<ArticleResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(articleService.findById(id));
        // articleService ném ResourceNotFoundException nếu không tìm thấy
        // → @ControllerAdvice bắt và trả 404
    }

    // POST /api/v1/articles
    @PostMapping
    public ResponseEntity<ArticleResponse> create(
        @RequestBody @Valid CreateArticleRequest request
    ) {
        ArticleResponse article = articleService.create(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(article.id())
            .toUri();
        return ResponseEntity.created(location).body(article);
    }

    // PUT /api/v1/articles/42
    @PutMapping("/{id}")
    public ArticleResponse update(
        @PathVariable Long id,
        @RequestBody @Valid UpdateArticleRequest request
    ) {
        return articleService.update(id, request);
    }

    // PATCH /api/v1/articles/42/publish
    @PatchMapping("/{id}/publish")
    public ArticleResponse publish(@PathVariable Long id) {
        return articleService.publish(id);
    }

    // DELETE /api/v1/articles/42
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        articleService.delete(id);
    }
}
```

---

## 8. Best Practices — Thực Tiễn Tốt Nhất

### Thiết Kế URL

```
✅ Đúng:
GET    /api/v1/users              — Danh sách users
GET    /api/v1/users/{id}         — User theo ID
POST   /api/v1/users              — Tạo user mới
PUT    /api/v1/users/{id}         — Cập nhật toàn bộ user
PATCH  /api/v1/users/{id}         — Cập nhật một phần
DELETE /api/v1/users/{id}         — Xóa user
GET    /api/v1/users/{id}/orders  — Orders của user

❌ Sai:
GET  /api/v1/getUser              — Không dùng verb trong URL
POST /api/v1/createUser           — Sai convention REST
GET  /api/v1/user/{id}            — Nên dùng số nhiều (users, not user)
POST /api/v1/users/delete/{id}    — Dùng DELETE method thay vì path
```

### Trả Về Response Nhất Quán

```java
// ✅ Tốt — Luôn wrap trong cùng một structure
public record ApiResponse<T>(
    boolean success,
    T data,
    String message,
    Instant timestamp
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null, Instant.now());
    }

    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message, Instant.now());
    }
}

// Dùng trong controller:
@GetMapping("/{id}")
public ApiResponse<UserResponse> getUser(@PathVariable Long id) {
    return ApiResponse.ok(userService.findById(id));
}
```

### Phân Tách Controller và Service

```java
// ✅ Controller chỉ xử lý HTTP — không chứa business logic
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;     // Business logic
    private final UserMapper userMapper;        // Chuyển đổi Entity ↔ DTO

    @PostMapping
    public ResponseEntity<UserResponse> create(@RequestBody @Valid CreateUserRequest request) {
        // Controller không biết về database, validation logic...
        User user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(userMapper.toResponse(user));
    }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa `@Controller` và `@RestController`?**

> `@Controller` trả về tên view (cho server-side rendering). `@RestController` = `@Controller` + `@ResponseBody` — tự động serialize return value thành JSON/XML.

**Q: Khi nào dùng `PUT` vs `PATCH`?**

> `PUT` thay thế toàn bộ resource — client gửi đầy đủ thông tin. `PATCH` cập nhật một phần — client chỉ gửi các field cần thay đổi. Cả hai đều **idempotent** (bất biến).

**Q: `@PathVariable` vs `@RequestParam` — khi nào dùng cái nào?**

> `@PathVariable` cho phần định danh tài nguyên trong URL path (`/users/42`). `@RequestParam` cho tham số tùy chọn trong query string (`?page=0&sort=name`). Quy tắc: dùng path variable cho ID/định danh, query param cho filtering/sorting/paging.

**Q: Tại sao nên dùng Constructor Injection thay vì `@Autowired` trên field?**

> Constructor Injection giúp: (1) dependency rõ ràng và bắt buộc, (2) dễ viết unit test (inject mock), (3) phát hiện circular dependency sớm hơn, (4) object luôn ở trạng thái hợp lệ sau khi khởi tạo.

---

## ✅ Checklist

- [ ] Phân biệt được `@Controller` vs `@RestController`
- [ ] Dùng đúng HTTP method cho từng thao tác CRUD
- [ ] Trả về đúng HTTP status code (201, 204, 404...)
- [ ] Xử lý được PathVariable, RequestParam, RequestBody
- [ ] Dùng `ResponseEntity` để control đầy đủ response
- [ ] Áp dụng URL naming convention (số nhiều, không có verb)

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Tiếp Theo:** [2-request-response.md](2-request-response.md) — DTO & Bean Validation
