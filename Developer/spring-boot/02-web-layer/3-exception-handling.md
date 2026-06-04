# Exception Handling — @ControllerAdvice & ProblemDetail (RFC 7807)

> Bài này trình bày cách xây dựng cơ chế xử lý ngoại lệ toàn cục (Global Exception Handling)
> nhất quán, chuẩn mực và thân thiện với client — theo chuẩn RFC 7807 (ProblemDetail).

---

## 📋 Mục Tiêu

- [ ] Hiểu tại sao cần **Global Exception Handling** (Xử Lý Ngoại Lệ Toàn Cục)
- [ ] Implement `@ControllerAdvice` để bắt mọi exception
- [ ] Dùng **ProblemDetail** (RFC 7807) — chuẩn response lỗi hiện đại
- [ ] Tạo **Custom Exception Hierarchy** (Phân Cấp Ngoại Lệ Tùy Chỉnh)
- [ ] Xử lý **Validation Errors** (Lỗi Xác Thực) với message rõ ràng
- [ ] Phân biệt lỗi client (4xx) và lỗi server (5xx)

---

## 1. Vấn Đề Khi Không Có Global Exception Handler

```java
// ❌ Xử lý lỗi phân tán — không nhất quán
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new RuntimeException("User not found"));
    return userMapper.toResponse(user);
}

// → Spring trả về response mặc định — không thân thiện:
// {
//   "timestamp": "2026-06-02T10:00:00.000+00:00",
//   "status": 500,
//   "error": "Internal Server Error",
//   "path": "/users/999"
// }
// Vấn đề: Status 500 trong khi đây là lỗi 404, message tiếng Anh, không có detail
```

---

## 2. RFC 7807 — ProblemDetail (Chuẩn Phản Hồi Lỗi)

**RFC 7807** (Problem Details for HTTP APIs) định nghĩa cấu trúc chuẩn cho response lỗi:

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User với ID 999 không tồn tại trong hệ thống",
  "instance": "/api/v1/users/999",
  "timestamp": "2026-06-02T10:00:00Z",
  "traceId": "abc123def456"
}
```

| Field | Ý Nghĩa |
|-------|---------|
| `type` | URI định danh loại lỗi — link đến tài liệu |
| `title` | Tên ngắn gọn của loại lỗi (không thay đổi theo ngôn ngữ) |
| `status` | HTTP status code |
| `detail` | Mô tả chi tiết cho lần xảy ra lỗi cụ thể này |
| `instance` | URI của request gây ra lỗi |

Spring Boot 3.x hỗ trợ `ProblemDetail` sẵn trong `org.springframework.http`.

---

## 3. Custom Exception Hierarchy — Phân Cấp Ngoại Lệ

```java
// Base exception — tất cả custom exceptions kế thừa
public abstract class ApplicationException extends RuntimeException {

    private final String errorCode;    // Mã lỗi nội bộ (VD: "USER_NOT_FOUND")
    private final HttpStatus status;   // HTTP status tương ứng
    private final Map<String, Object> properties; // Thông tin bổ sung

    protected ApplicationException(String message, String errorCode, HttpStatus status) {
        super(message);
        this.errorCode = errorCode;
        this.status = status;
        this.properties = new LinkedHashMap<>();
    }

    protected ApplicationException addProperty(String key, Object value) {
        this.properties.put(key, value);
        return this;
    }

    // Getters...
}

// ─────────────────────────────────────────────────────────────────
// 4xx — Lỗi Phía Client
// ─────────────────────────────────────────────────────────────────

// 404 Not Found — Không tìm thấy tài nguyên
public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super(
            "%s với ID '%s' không tồn tại".formatted(resourceType, id),
            "RESOURCE_NOT_FOUND",
            HttpStatus.NOT_FOUND
        );
    }
}

// 409 Conflict — Xung đột (tài nguyên đã tồn tại)
public class ResourceAlreadyExistsException extends ApplicationException {
    public ResourceAlreadyExistsException(String message) {
        super(message, "RESOURCE_ALREADY_EXISTS", HttpStatus.CONFLICT);
    }
}

// 400 Bad Request — Yêu cầu không hợp lệ về nghiệp vụ
public class BusinessException extends ApplicationException {
    public BusinessException(String message) {
        super(message, "BUSINESS_RULE_VIOLATION", HttpStatus.BAD_REQUEST);
    }
}

// 403 Forbidden — Không có quyền
public class AccessDeniedException extends ApplicationException {
    public AccessDeniedException(String resource) {
        super(
            "Bạn không có quyền truy cập: " + resource,
            "ACCESS_DENIED",
            HttpStatus.FORBIDDEN
        );
    }
}

// 422 Unprocessable Entity — Hợp lệ về cú pháp nhưng không hợp lệ nghiệp vụ
public class UnprocessableEntityException extends ApplicationException {
    public UnprocessableEntityException(String message) {
        super(message, "UNPROCESSABLE_ENTITY", HttpStatus.UNPROCESSABLE_ENTITY);
    }
}

// ─────────────────────────────────────────────────────────────────
// 5xx — Lỗi Phía Server
// ─────────────────────────────────────────────────────────────────

// 503 Service Unavailable — Dịch vụ phụ không khả dụng
public class ExternalServiceException extends ApplicationException {
    public ExternalServiceException(String serviceName, String reason) {
        super(
            "Dịch vụ '%s' tạm thời không khả dụng: %s".formatted(serviceName, reason),
            "EXTERNAL_SERVICE_ERROR",
            HttpStatus.SERVICE_UNAVAILABLE
        );
    }
}
```

---

## 4. @ControllerAdvice — Xử Lý Ngoại Lệ Toàn Cục

```java
@RestControllerAdvice   // = @ControllerAdvice + @ResponseBody
@Slf4j
public class GlobalExceptionHandler {

    // ─────────────────────────────────────────────────────────────
    // Bắt Custom Application Exceptions
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ProblemDetail> handleApplicationException(
        ApplicationException ex,
        HttpServletRequest request
    ) {
        log.warn("Application exception [{}]: {}", ex.getErrorCode(), ex.getMessage());

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(ex.getStatus(), ex.getMessage());
        problem.setTitle(ex.getErrorCode());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("errorCode", ex.getErrorCode());
        problem.setProperty("timestamp", Instant.now());

        // Thêm custom properties nếu có
        ex.getProperties().forEach(problem::setProperty);

        return ResponseEntity.status(ex.getStatus()).body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // Bean Validation — MethodArgumentNotValidException
    // Ném khi @Valid thất bại ở @RequestBody
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidationException(
        MethodArgumentNotValidException ex,
        HttpServletRequest request
    ) {
        // Thu thập tất cả lỗi validation thành list
        List<ValidationError> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new ValidationError(fe.getField(), fe.getDefaultMessage()))
            .toList();

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "Dữ liệu đầu vào không hợp lệ"
        );
        problem.setTitle("Validation Failed");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());

        return ResponseEntity.badRequest().body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // @RequestParam Validation — ConstraintViolationException
    // Ném khi @Validated thất bại ở @RequestParam / @PathVariable
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ProblemDetail> handleConstraintViolation(
        ConstraintViolationException ex,
        HttpServletRequest request
    ) {
        List<ValidationError> errors = ex.getConstraintViolations()
            .stream()
            .map(cv -> {
                String field = cv.getPropertyPath().toString();
                // Lấy tên field cuối cùng (bỏ method name prefix)
                field = field.substring(field.lastIndexOf('.') + 1);
                return new ValidationError(field, cv.getMessage());
            })
            .toList();

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "Tham số request không hợp lệ"
        );
        problem.setTitle("Constraint Violation");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());

        return ResponseEntity.badRequest().body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // JSON Parse Error — HttpMessageNotReadableException
    // Ném khi body JSON không đúng định dạng
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ProblemDetail> handleMessageNotReadable(
        HttpMessageNotReadableException ex,
        HttpServletRequest request
    ) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "Request body không đúng định dạng JSON"
        );
        problem.setTitle("Invalid Request Body");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", Instant.now());

        return ResponseEntity.badRequest().body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // HTTP Method Not Supported — 405 Method Not Allowed
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<ProblemDetail> handleMethodNotSupported(
        HttpRequestMethodNotSupportedException ex,
        HttpServletRequest request
    ) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.METHOD_NOT_ALLOWED,
            "Method '%s' không được hỗ trợ cho endpoint này".formatted(ex.getMethod())
        );
        problem.setTitle("Method Not Allowed");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("supportedMethods", ex.getSupportedMethods());
        problem.setProperty("timestamp", Instant.now());

        return ResponseEntity.status(HttpStatus.METHOD_NOT_ALLOWED).body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // Catch-all — 500 Internal Server Error
    // Bắt mọi exception không mong đợi
    // ─────────────────────────────────────────────────────────────

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleGenericException(
        Exception ex,
        HttpServletRequest request
    ) {
        // Log đầy đủ stack trace cho lỗi không mong đợi
        log.error("Unhandled exception at {}: {}", request.getRequestURI(), ex.getMessage(), ex);

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "Đã xảy ra lỗi nội bộ, vui lòng thử lại sau"  // Không lộ chi tiết lỗi kỹ thuật
        );
        problem.setTitle("Internal Server Error");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", Instant.now());

        return ResponseEntity.internalServerError().body(problem);
    }

    // ─────────────────────────────────────────────────────────────
    // DTO nội bộ cho validation errors
    // ─────────────────────────────────────────────────────────────

    public record ValidationError(String field, String message) {}
}
```

---

## 5. Sử Dụng Custom Exceptions Trong Service

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // ✅ Ném custom exception thay vì RuntimeException thông thường
    public UserResponse findById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
        return userMapper.toResponse(user);
    }

    public UserResponse create(CreateUserRequest request) {
        // Kiểm tra trùng email
        if (userRepository.existsByEmail(request.email())) {
            throw new ResourceAlreadyExistsException(
                "Email '%s' đã được sử dụng".formatted(request.email())
            );
        }

        // Kiểm tra nghiệp vụ
        if (isBlacklisted(request.email())) {
            throw new BusinessException("Email domain không được phép đăng ký");
        }

        User user = userMapper.toEntity(request);
        user.setPasswordHash(passwordEncoder.encode(request.password()));
        user = userRepository.save(user);
        return userMapper.toResponse(user);
    }

    public void delete(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));

        // Kiểm tra nghiệp vụ trước khi xóa
        if (user.hasActiveOrders()) {
            throw new BusinessException("Không thể xóa user đang có đơn hàng chưa hoàn thành");
        }

        userRepository.delete(user);
    }
}
```

---

## 6. Cấu Hình ProblemDetail Trong Spring Boot 3.x

```yaml
# application.yml
spring:
  mvc:
    problemdetails:
      enabled: true  # Kích hoạt ProblemDetail tự động cho các lỗi Spring MVC

# Khi enabled: Spring tự trả ProblemDetail cho các lỗi như:
# - NoHandlerFoundException (404)
# - HttpRequestMethodNotSupportedException (405)
# - HttpMediaTypeNotSupportedException (415)
```

---

## 7. Exception Handling Nâng Cao

### Ghi Log Có Cấu Trúc (Structured Logging)

```java
@ExceptionHandler(ApplicationException.class)
public ResponseEntity<ProblemDetail> handleApplicationException(
    ApplicationException ex,
    HttpServletRequest request
) {
    // Log có cấu trúc — dễ query trong ELK Stack, Grafana Loki
    MDC.put("errorCode", ex.getErrorCode());
    MDC.put("path", request.getRequestURI());
    MDC.put("method", request.getMethod());

    if (ex.getStatus().is4xxClientError()) {
        log.warn("Client error [{}] at {} {}: {}",
            ex.getErrorCode(), request.getMethod(), request.getRequestURI(), ex.getMessage());
    } else {
        log.error("Server error [{}] at {} {}: {}",
            ex.getErrorCode(), request.getMethod(), request.getRequestURI(), ex.getMessage(), ex);
    }

    MDC.clear();
    // ... tạo ProblemDetail và trả về
}
```

### Tùy Chỉnh Per-Controller

```java
// @ControllerAdvice có thể giới hạn scope
@RestControllerAdvice(basePackages = "com.example.api.payment")
public class PaymentExceptionHandler {

    @ExceptionHandler(PaymentDeclinedException.class)
    public ResponseEntity<ProblemDetail> handlePaymentDeclined(PaymentDeclinedException ex) {
        // Xử lý đặc biệt cho lỗi thanh toán
    }
}

// Handler trong controller — chỉ áp dụng cho controller đó
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<ProblemDetail> handleOptimisticLock(OptimisticLockingFailureException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT,
            "Dữ liệu đã được thay đổi bởi người khác, vui lòng tải lại và thử lại"
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(problem);
    }
}
```

### Error Response Nhất Quán — Không Lộ Thông Tin Nhạy Cảm

```java
// ❌ Sai — lộ thông tin kỹ thuật
problem.setDetail(ex.getMessage()); // "Connection refused to db01.internal:5432"

// ✅ Đúng — ẩn chi tiết kỹ thuật, log đầy đủ ở server
log.error("Database connection failed: {}", ex.getMessage(), ex);
problem.setDetail("Không thể kết nối tới cơ sở dữ liệu, vui lòng thử lại");
```

---

## 8. Ví Dụ Response Thực Tế

### 404 Not Found

```json
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{
  "type": "about:blank",
  "title": "RESOURCE_NOT_FOUND",
  "status": 404,
  "detail": "User với ID '999' không tồn tại",
  "instance": "/api/v1/users/999",
  "errorCode": "RESOURCE_NOT_FOUND",
  "timestamp": "2026-06-02T10:00:00Z"
}
```

### 400 Validation Failed

```json
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "about:blank",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Dữ liệu đầu vào không hợp lệ",
  "instance": "/api/v1/users",
  "errors": [
    { "field": "email", "message": "Email không hợp lệ" },
    { "field": "password", "message": "Mật khẩu tối thiểu 8 ký tự" },
    { "field": "fullName", "message": "Tên đầy đủ không được để trống" }
  ],
  "timestamp": "2026-06-02T10:00:00Z"
}
```

### 409 Conflict

```json
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "about:blank",
  "title": "RESOURCE_ALREADY_EXISTS",
  "status": 409,
  "detail": "Email 'user@example.com' đã được sử dụng",
  "instance": "/api/v1/users",
  "errorCode": "RESOURCE_ALREADY_EXISTS",
  "timestamp": "2026-06-02T10:00:00Z"
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: `@ControllerAdvice` vs `@RestControllerAdvice` khác nhau thế nào?**

> `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. Nghĩa là các method handler tự động serialize return value thành JSON/XML thay vì cần `@ResponseBody` trên từng method. Thường dùng `@RestControllerAdvice` cho REST API.

**Q: Thứ tự ưu tiên khi nhiều `@ExceptionHandler` bắt cùng một exception?**

> Spring chọn handler **đặc biệt nhất** (most specific): (1) Handler trong chính controller đó ưu tiên hơn `@ControllerAdvice`. (2) Trong `@ControllerAdvice`, handler bắt exception cụ thể hơn được ưu tiên (VD: `ResourceNotFoundException` trước `ApplicationException` trước `Exception`).

**Q: ProblemDetail RFC 7807 là gì và tại sao nên dùng?**

> RFC 7807 là chuẩn IETF định nghĩa format JSON/XML cho response lỗi HTTP API. Lợi ích: (1) Chuẩn hóa — client biết cách xử lý lỗi từ mọi API tuân thủ RFC 7807. (2) Machine-readable `type` URI — có thể link đến docs. (3) Có `instance` để debug request cụ thể. Spring Boot 3.x hỗ trợ sẵn qua class `ProblemDetail`.

**Q: Làm sao để không lộ thông tin nhạy cảm trong error response?**

> (1) Luôn log đầy đủ stack trace ở server (với log level ERROR). (2) Response trả ra chỉ chứa message thân thiện với người dùng. (3) Catch-all handler (`Exception.class`) không trả chi tiết lỗi kỹ thuật. (4) Không include `ex.getMessage()` trực tiếp cho lỗi 5xx.

---

## ✅ Checklist

- [ ] `@RestControllerAdvice` với `@ExceptionHandler` cho từng loại lỗi
- [ ] Custom exception hierarchy kế thừa từ ApplicationException
- [ ] ProblemDetail với đầy đủ type, title, status, detail, instance
- [ ] Xử lý riêng MethodArgumentNotValidException → list validation errors
- [ ] Catch-all handler cho Exception.class → 500 không lộ stack trace
- [ ] Log 4xx ở WARN, 5xx ở ERROR với đầy đủ context

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Tiếp Theo:** [4-filters-interceptors.md](4-filters-interceptors.md) — Filter & Interceptor
