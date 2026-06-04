# OpenAPI & Swagger UI — springdoc-openapi

> Bài này hướng dẫn tích hợp **springdoc-openapi** để tự động tạo tài liệu API theo chuẩn
> **OpenAPI 3.0** và hiển thị giao diện **Swagger UI** tương tác trực tiếp từ code.

---

## 📋 Mục Tiêu

- [ ] Tích hợp **springdoc-openapi** vào dự án Spring Boot
- [ ] Cấu hình **OpenAPI Spec** (Đặc Tả OpenAPI) với metadata dự án
- [ ] Dùng các annotation để tài liệu hóa Controller, DTO và Security
- [ ] Nhóm API theo tags và tổ chức theo version
- [ ] Tích hợp **JWT Bearer Token** vào Swagger UI
- [ ] Customize Swagger UI và endpoint path

---

## 1. OpenAPI vs Swagger — Tổng Quan

```
OpenAPI Specification (OAS)    ← Chuẩn mở do Linux Foundation quản lý
       │                          Định nghĩa format JSON/YAML cho API docs
       │
       ├── Swagger UI          ← Giao diện web tương tác để test API
       ├── Swagger Editor      ← Trình soạn thảo OpenAPI spec
       └── springdoc-openapi   ← Thư viện tự động sinh OAS từ Spring Boot code
```

**springdoc-openapi** quét annotations của Spring MVC (`@RestController`, `@GetMapping`...) và tự động sinh file `openapi.json` / `openapi.yaml` theo chuẩn OpenAPI 3.0.

---

## 2. Tích Hợp springdoc-openapi

### Dependency — Maven

```xml
<!-- Spring Boot 3.x — dùng springdoc-openapi-starter-webmvc-ui -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

### Dependency — Gradle

```groovy
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
```

Sau khi thêm dependency, Swagger UI tự động có tại:
- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON:** `http://localhost:8080/v3/api-docs`
- **OpenAPI YAML:** `http://localhost:8080/v3/api-docs.yaml`

---

## 3. Cấu Hình OpenAPI — OpenApiConfig

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            // Thông tin cơ bản về API
            .info(new Info()
                .title("E-Commerce API")
                .description("""
                    REST API cho nền tảng thương mại điện tử.
                    
                    ## Xác Thực
                    Hầu hết endpoints yêu cầu JWT Bearer Token.
                    Sử dụng nút **Authorize** để nhập token.
                    
                    ## Phiên Bản
                    - v1: Phiên bản hiện tại (stable)
                    - v2: Đang phát triển (beta)
                    """)
                .version("1.0.0")
                .contact(new Contact()
                    .name("Backend Team")
                    .email("backend@example.com")
                    .url("https://github.com/example/api"))
                .license(new License()
                    .name("MIT License")
                    .url("https://opensource.org/licenses/MIT"))
            )

            // Server environments — môi trường server
            .servers(List.of(
                new Server()
                    .url("https://api.example.com")
                    .description("Production (Môi Trường Sản Xuất)"),
                new Server()
                    .url("https://staging-api.example.com")
                    .description("Staging (Môi Trường Staging)"),
                new Server()
                    .url("http://localhost:8080")
                    .description("Local Development (Môi Trường Phát Triển)")
            ))

            // Security scheme — JWT Bearer Token
            .components(new Components()
                .addSecuritySchemes("bearerAuth", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .description("Nhập JWT token (không cần prefix 'Bearer')")
                )
            )

            // Áp dụng security cho toàn bộ API
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}
```

---

## 4. Cấu Hình application.yml

```yaml
springdoc:
  # Swagger UI configuration
  swagger-ui:
    path: /swagger-ui.html                # URL của Swagger UI
    tags-sorter: alpha                     # Sắp xếp tags theo alphabet
    operations-sorter: alpha               # Sắp xếp operations theo alphabet
    display-request-duration: true         # Hiển thị thời gian thực thi request
    default-models-expand-depth: 2         # Độ sâu mở rộng schema mặc định
    try-it-out-enabled: true               # Bật "Try it out" mặc định
    filter: true                           # Bật thanh tìm kiếm

  # OpenAPI spec endpoint
  api-docs:
    path: /v3/api-docs                     # URL của OpenAPI JSON spec

  # Packages cần scan cho API docs
  packages-to-scan: com.example.api.controller

  # Paths cần include/exclude
  paths-to-match: /api/**
  paths-to-exclude: /api/internal/**

  # Show actuator endpoints trong docs
  show-actuator: false

  # Group APIs
  group-configs:
    - group: public
      paths-to-match: /api/v1/public/**
      display-name: "Public API (Không Cần Xác Thực)"
    - group: secured
      paths-to-match: /api/v1/**
      paths-to-exclude: /api/v1/public/**
      display-name: "Secured API (Cần JWT)"
```

---

## 5. Tài Liệu Hóa Controller

### @Tag — Nhóm Endpoints

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Tag(
    name = "User Management",
    description = "Quản lý người dùng — đăng ký, cập nhật thông tin, phân quyền"
)
public class UserController {
    // ...
}
```

### @Operation — Tài Liệu Hóa Method

```java
@GetMapping("/{id}")
@Operation(
    summary = "Lấy thông tin user theo ID",
    description = "Trả về thông tin chi tiết của user. Yêu cầu quyền ADMIN hoặc chính user đó.",
    tags = { "User Management" },
    security = @SecurityRequirement(name = "bearerAuth")
)
@ApiResponses({
    @ApiResponse(
        responseCode = "200",
        description = "Thành công",
        content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = UserResponse.class),
            examples = @ExampleObject(
                name = "Ví dụ thành công",
                value = """
                    {
                      "id": 1,
                      "email": "user@example.com",
                      "fullName": "Nguyễn Văn A",
                      "role": "USER",
                      "createdAt": "2026-06-02T10:00:00Z"
                    }
                    """
            )
        )
    ),
    @ApiResponse(
        responseCode = "404",
        description = "User không tồn tại",
        content = @Content(
            mediaType = "application/problem+json",
            schema = @Schema(implementation = ProblemDetail.class)
        )
    ),
    @ApiResponse(
        responseCode = "401",
        description = "Chưa xác thực — thiếu hoặc token không hợp lệ",
        content = @Content(schema = @Schema(hidden = true))
    )
})
public ResponseEntity<UserResponse> getById(
    @Parameter(description = "ID của user", example = "1", required = true)
    @PathVariable Long id
) {
    return ResponseEntity.ok(userService.findById(id));
}
```

### @Parameter — Tài Liệu Hóa Tham Số

```java
@GetMapping
@Operation(summary = "Tìm kiếm danh sách users")
public Page<UserResponse> search(
    @Parameter(description = "Từ khóa tìm kiếm (email hoặc tên)", example = "nguyen")
    @RequestParam(required = false) String keyword,

    @Parameter(description = "Số trang, bắt đầu từ 0", example = "0")
    @RequestParam(defaultValue = "0") int page,

    @Parameter(description = "Số lượng mỗi trang (tối đa 100)", example = "20")
    @RequestParam(defaultValue = "20") int size,

    @Parameter(
        description = "Trường sắp xếp",
        schema = @Schema(allowableValues = { "id", "email", "fullName", "createdAt" })
    )
    @RequestParam(defaultValue = "id") String sort,

    @Parameter(description = "Chiều sắp xếp", schema = @Schema(allowableValues = { "asc", "desc" }))
    @RequestParam(defaultValue = "asc") String direction
) {
    return userService.search(keyword, PageRequest.of(page, size, Sort.by(Sort.Direction.fromString(direction), sort)));
}
```

---

## 6. Tài Liệu Hóa DTO — Schema Annotations

```java
@Schema(
    name = "CreateUserRequest",
    description = "Dữ liệu để tạo tài khoản người dùng mới"
)
public record CreateUserRequest(

    @Schema(
        description = "Địa chỉ email — dùng để đăng nhập",
        example = "user@example.com",
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotBlank @Email
    String email,

    @Schema(
        description = "Tên đầy đủ của người dùng",
        example = "Nguyễn Văn A",
        minLength = 2,
        maxLength = 100
    )
    @NotBlank @Size(min = 2, max = 100)
    String fullName,

    @Schema(
        description = "Mật khẩu — tối thiểu 8 ký tự, bao gồm chữ hoa, số và ký tự đặc biệt",
        example = "SecurePass123!",
        minLength = 8,
        accessMode = Schema.AccessMode.WRITE_ONLY  // Không hiển thị trong response
    )
    @NotBlank @Size(min = 8)
    String password,

    @Schema(
        description = "Vai trò người dùng trong hệ thống",
        example = "USER",
        allowableValues = { "USER", "ADMIN", "MODERATOR" }
    )
    @NotNull
    UserRole role
) {}
```

```java
@Schema(
    name = "UserResponse",
    description = "Thông tin người dùng trả về — không bao gồm mật khẩu"
)
public record UserResponse(

    @Schema(description = "ID duy nhất của user", example = "1")
    Long id,

    @Schema(description = "Địa chỉ email", example = "user@example.com")
    String email,

    @Schema(description = "Tên đầy đủ", example = "Nguyễn Văn A")
    String fullName,

    @Schema(description = "Vai trò", example = "USER")
    UserRole role,

    @Schema(description = "Thời điểm tạo tài khoản (ISO 8601)", example = "2026-06-02T10:00:00Z")
    Instant createdAt,

    @Schema(description = "Tài khoản đang active hay không", example = "true")
    boolean active
) {}
```

---

## 7. Tài Liệu Hóa Security — Endpoints Không Cần Auth

```java
@RestController
@RequestMapping("/api/v1/auth")
@Tag(name = "Authentication", description = "Đăng nhập, đăng ký và refresh token")
public class AuthController {

    // Endpoint không cần token — tắt security trong docs
    @PostMapping("/login")
    @Operation(
        summary = "Đăng nhập",
        description = "Trả về JWT access token và refresh token",
        security = {} // Override global security — endpoint này không cần auth
    )
    @ApiResponse(
        responseCode = "200",
        description = "Đăng nhập thành công",
        content = @Content(schema = @Schema(implementation = LoginResponse.class))
    )
    public LoginResponse login(@RequestBody @Valid LoginRequest request) {
        return authService.login(request);
    }

    @PostMapping("/register")
    @Operation(
        summary = "Đăng ký tài khoản",
        security = {}
    )
    public ResponseEntity<UserResponse> register(@RequestBody @Valid CreateUserRequest request) {
        UserResponse user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }

    @PostMapping("/refresh")
    @Operation(
        summary = "Làm mới access token",
        description = "Dùng refresh token để lấy access token mới",
        security = {} // Refresh token truyền qua body, không qua Bearer header
    )
    public TokenResponse refresh(@RequestBody @Valid RefreshTokenRequest request) {
        return authService.refresh(request.refreshToken());
    }
}
```

---

## 8. Ẩn Endpoints Khỏi Swagger

```java
// Ẩn toàn bộ controller khỏi Swagger docs
@Hidden
@RestController
@RequestMapping("/api/internal")
public class InternalController { ... }

// Ẩn một method cụ thể
@Operation(hidden = true)
@GetMapping("/debug-info")
public Map<String, Object> debugInfo() { ... }
```

---

## 9. Tắt Swagger UI Trên Production

```yaml
# application-production.yml
springdoc:
  swagger-ui:
    enabled: false   # Tắt Swagger UI trên production
  api-docs:
    enabled: false   # Tắt cả OpenAPI spec endpoint
```

Hoặc kiểm soát bằng profile:

```java
@Configuration
@Profile("!production")  // Chỉ active khi KHÔNG phải production
public class SwaggerConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info().title("API Docs (Dev Only)").version("1.0.0"));
    }
}
```

---

## 10. Swagger UI Với JWT — Cách Sử Dụng

Sau khi đăng nhập, copy JWT token và:

1. Mở Swagger UI tại `http://localhost:8080/swagger-ui.html`
2. Click nút **Authorize** (🔒) góc trên phải
3. Nhập token vào ô **bearerAuth** (không cần prefix "Bearer")
4. Click **Authorize** → **Close**
5. Tất cả requests tiếp theo sẽ tự động gắn `Authorization: Bearer <token>`

---

## 11. Tích Hợp Swagger Với Actuator

```yaml
springdoc:
  show-actuator: true                   # Hiển thị /actuator endpoints trong docs
  actuator:
    paths-to-match: /health, /info      # Chỉ show các actuator endpoints chọn lọc
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác nhau giữa OpenAPI 2.0 (Swagger) và OpenAPI 3.0?**

> OpenAPI 3.0 cải tiến đáng kể so với 2.0: (1) Không còn `basePath` và `host` riêng — thay bằng `servers` array hỗ trợ nhiều môi trường. (2) `requestBody` tách riêng khỏi parameters — rõ ràng hơn. (3) `components` thay thế `definitions` — tổ chức tốt hơn. (4) Hỗ trợ `oneOf`, `anyOf`, `not` cho schema phức tạp. (5) Hỗ trợ link (liên kết giữa responses).

**Q: springdoc-openapi vs springfox — nên dùng cái nào?**

> Dùng **springdoc-openapi**. springfox không còn được maintain tích cực và không hỗ trợ Spring Boot 3.x / Spring 6. springdoc-openapi hỗ trợ OpenAPI 3.0 đầy đủ, tương thích Spring Boot 3.x, WebFlux, và được maintain tốt.

**Q: Làm sao để không expose Swagger UI trên production?**

> Cách 1: `springdoc.swagger-ui.enabled=false` và `springdoc.api-docs.enabled=false` trong `application-production.yml`. Cách 2: Dùng `@Profile("!production")` trên `@Configuration` bean. Cách 3: Bảo vệ endpoint `/swagger-ui/**` và `/v3/api-docs/**` bằng Spring Security — chỉ cho phép IP nội bộ hoặc role ADMIN.

**Q: Tại sao cần tài liệu hóa API?**

> (1) **Developer experience** — frontend/mobile dev không cần đọc code server để hiểu API. (2) **Contract first** — OpenAPI spec làm contract giữa teams. (3) **Testing** — Swagger UI cho phép test nhanh không cần Postman. (4) **Code generation** — sinh client SDK tự động từ spec. (5) **Onboarding** — docs luôn đồng bộ với code (không cần maintain thủ công như wiki).

---

## ✅ Checklist

- [ ] Thêm springdoc-openapi dependency đúng version
- [ ] Cấu hình OpenAPI bean với Info, Servers, SecurityScheme
- [ ] @Tag trên controller để nhóm endpoints
- [ ] @Operation và @ApiResponse trên method quan trọng
- [ ] @Schema trên DTO với example values
- [ ] Tắt Swagger UI trên production profile
- [ ] JWT Bearer Token hoạt động trong Swagger UI

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Module Tiếp Theo:** [03-data-access/](../03-data-access/) — Spring Data JPA & Truy Cập Cơ Sở Dữ Liệu
