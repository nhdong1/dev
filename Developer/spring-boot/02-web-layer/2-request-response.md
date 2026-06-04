# Request & Response — DTO, Bean Validation, MapStruct & Serialization

> Bài này bao gồm cách thiết kế DTO (Data Transfer Object — Đối Tượng Truyền Dữ Liệu),
> xác thực input với Bean Validation, ánh xạ giữa Entity và DTO với MapStruct,
> và cấu hình Jackson để serialize/deserialize JSON đúng cách.

---

## 📋 Mục Tiêu

- [ ] Thiết kế **DTO** (Data Transfer Object) riêng cho Request và Response
- [ ] Áp dụng **Bean Validation** (JSR-380) với các constraint annotations
- [ ] Tạo **Custom Validator** (Bộ Xác Thực Tùy Chỉnh) cho logic nghiệp vụ phức tạp
- [ ] Dùng **MapStruct** để tự động ánh xạ Entity ↔ DTO
- [ ] Cấu hình **Jackson** để serialize/deserialize JSON linh hoạt
- [ ] Sử dụng **Java Records** làm DTO bất biến (immutable)

---

## 1. DTO Pattern — Tại Sao Không Expose Entity Trực Tiếp?

### Vấn Đề Khi Expose Entity

```java
// ❌ Sai — Expose JPA Entity trực tiếp ra API
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userRepository.findById(id).orElseThrow();
        // Vấn đề:
        // 1. Expose password, internalNotes ra client
        // 2. Jackson có thể trigger lazy loading → N+1
        // 3. Thay đổi DB schema sẽ break API contract
        // 4. Validation logic lẫn lộn với domain logic
    }
}
```

### Giải Pháp — DTO Riêng Biệt

```
Client Request JSON
       │
       ▼
  CreateUserRequest DTO    ← Validate input, chỉ có fields cần thiết
       │
       ▼ (MapStruct / manual mapping)
    User Entity            ← Domain model — làm việc với DB
       │
       ▼ (MapStruct / manual mapping)
  UserResponse DTO         ← Chỉ expose fields cần thiết ra ngoài
       │
       ▼
Client Response JSON
```

---

## 2. Thiết Kế DTO với Java Records

Java 17+ **Records** là lựa chọn tốt nhất cho DTO — bất biến (immutable), ngắn gọn, tự có equals/hashCode/toString.

### Request DTO

```java
// Dùng Record cho DTO bất biến
public record CreateUserRequest(

    @NotBlank(message = "Email không được để trống")
    @Email(message = "Email không hợp lệ")
    @Size(max = 255, message = "Email tối đa 255 ký tự")
    String email,

    @NotBlank(message = "Tên đầy đủ không được để trống")
    @Size(min = 2, max = 100, message = "Tên phải từ 2 đến 100 ký tự")
    String fullName,

    @NotBlank(message = "Mật khẩu không được để trống")
    @Size(min = 8, message = "Mật khẩu tối thiểu 8 ký tự")
    @Pattern(
        regexp = "^(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%^&*]).{8,}$",
        message = "Mật khẩu phải có chữ hoa, số và ký tự đặc biệt"
    )
    String password,

    @NotNull(message = "Vai trò không được để trống")
    UserRole role
) {}
```

### Response DTO

```java
// Response DTO — chỉ expose fields an toàn
public record UserResponse(
    Long id,
    String email,
    String fullName,
    UserRole role,
    Instant createdAt,
    boolean active
) {}

// Response khi cần phân trang
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isLast()
        );
    }
}
```

### Update DTO — Hỗ Trợ Partial Update

```java
// Dùng class thông thường khi cần Optional fields (cho PATCH)
public class UpdateUserRequest {

    @Size(min = 2, max = 100)
    private String fullName;   // null = không cập nhật

    @Size(max = 500)
    private String bio;

    // Getters, setters...
}
```

---

## 3. Bean Validation — Xác Thực Dữ Liệu

### Dependency

```xml
<!-- spring-boot-starter-validation đã bao gồm Hibernate Validator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Các Constraint Annotations Phổ Biến

```java
public record ProductRequest(

    // Chuỗi
    @NotNull(message = "Tên sản phẩm không được null")
    @NotBlank(message = "Tên sản phẩm không được rỗng")  // Khác NotNull: không cho phép " "
    @Size(min = 3, max = 200)
    String name,

    // Số
    @NotNull
    @DecimalMin(value = "0.01", message = "Giá phải lớn hơn 0")
    @DecimalMax(value = "999999999.99")
    @Digits(integer = 9, fraction = 2)  // Tối đa 9 chữ số nguyên, 2 thập phân
    BigDecimal price,

    @Min(value = 0, message = "Số lượng tồn kho không âm")
    @Max(value = 100000)
    int stockQuantity,

    // Email & URL
    @Email
    String contactEmail,

    @URL
    String imageUrl,

    // Ngày
    @Future(message = "Ngày hết hạn phải trong tương lai")
    LocalDate expiryDate,

    @Past(message = "Ngày sản xuất phải trong quá khứ")
    LocalDate manufacturedDate,

    // Collection
    @NotEmpty(message = "Phải có ít nhất một tag")
    @Size(max = 10, message = "Tối đa 10 tags")
    List<@NotBlank String> tags,

    // Boolean
    @NotNull
    Boolean active
) {}
```

### Kích Hoạt Validation với @Valid

```java
@PostMapping
public ResponseEntity<ProductResponse> create(
    @RequestBody @Valid ProductRequest request  // @Valid kích hoạt Bean Validation
) {
    // Nếu validation thất bại → MethodArgumentNotValidException được ném
    // → @ControllerAdvice bắt và trả 400 Bad Request
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(productService.create(request));
}
```

### @Validated — Validation Theo Nhóm (Group Validation)

```java
// Định nghĩa nhóm validation
public interface OnCreate {}
public interface OnUpdate {}

public class UserRequest {

    @NotNull(groups = OnUpdate.class)       // Bắt buộc khi update
    private Long id;

    @NotBlank(groups = { OnCreate.class, OnUpdate.class })
    private String fullName;

    @NotBlank(groups = OnCreate.class)      // Chỉ bắt buộc khi tạo mới
    @Null(groups = OnUpdate.class)          // Không cho phép khi update
    private String password;
}

// Dùng @Validated thay @Valid để chỉ định nhóm
@PostMapping
public UserResponse create(@RequestBody @Validated(OnCreate.class) UserRequest request) { ... }

@PutMapping("/{id}")
public UserResponse update(@PathVariable Long id, @RequestBody @Validated(OnUpdate.class) UserRequest request) { ... }
```

---

## 4. Custom Validator — Xác Thực Tùy Chỉnh

### Ví Dụ: Validate Email Chưa Tồn Tại

```java
// Bước 1: Định nghĩa annotation
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
@Documented
public @interface UniqueEmail {
    String message() default "Email đã được sử dụng";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Bước 2: Implement logic xác thực
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return true; // Để @NotBlank xử lý trường hợp này
        }
        return !userRepository.existsByEmail(email);
    }
}

// Bước 3: Dùng trong DTO
public record CreateUserRequest(
    @Email
    @UniqueEmail  // Custom validator
    String email,

    @NotBlank
    String password
) {}
```

### Ví Dụ: Validate Cross-Field (Nhiều Fields Với Nhau)

```java
// Annotation áp dụng ở class level
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Mật khẩu xác nhận không khớp";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String password();
    String confirmPassword();
}

@Component
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {

    private String passwordField;
    private String confirmPasswordField;

    @Override
    public void initialize(PasswordMatch annotation) {
        this.passwordField = annotation.password();
        this.confirmPasswordField = annotation.confirmPassword();
    }

    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext context) {
        try {
            BeanWrapper wrapper = new BeanWrapperImpl(obj);
            Object password = wrapper.getPropertyValue(passwordField);
            Object confirmPassword = wrapper.getPropertyValue(confirmPasswordField);
            return Objects.equals(password, confirmPassword);
        } catch (Exception e) {
            return false;
        }
    }
}

// Dùng ở class level
@PasswordMatch(password = "password", confirmPassword = "confirmPassword")
public class ChangePasswordRequest {

    @NotBlank
    private String password;

    @NotBlank
    private String confirmPassword;
}
```

---

## 5. MapStruct — Ánh Xạ Entity ↔ DTO Tự Động

MapStruct — thư viện sinh code ánh xạ lúc compile time (không dùng reflection), hiệu suất cao.

### Dependency

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.0</version>
</dependency>
<annotationProcessorPaths>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>1.6.0</version>
    </path>
</annotationProcessorPaths>
```

### Entity và DTO

```java
// Entity
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue
    private Long id;
    private String email;
    private String fullName;
    private String passwordHash;  // Không expose ra DTO
    @Enumerated(EnumType.STRING)
    private UserRole role;
    private Instant createdAt;
    private boolean active;
}

// Request DTO
public record CreateUserRequest(String email, String fullName, String password, UserRole role) {}

// Response DTO
public record UserResponse(Long id, String email, String fullName, UserRole role, Instant createdAt) {}
```

### MapStruct Mapper

```java
@Mapper(
    componentModel = "spring",    // Tạo Spring Bean tự động
    unmappedTargetPolicy = ReportingPolicy.IGNORE  // Bỏ qua fields không mapped
)
public interface UserMapper {

    // Ánh xạ Entity → Response DTO
    UserResponse toResponse(User user);

    // Ánh xạ danh sách
    List<UserResponse> toResponseList(List<User> users);

    // Request DTO → Entity (bỏ qua field password — xử lý riêng)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "passwordHash", ignore = true)  // Xử lý riêng trong Service
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "active", constant = "true")
    User toEntity(CreateUserRequest request);

    // Cập nhật Entity từ DTO (partial update)
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntityFromDto(UpdateUserRequest dto, @MappingTarget User entity);
}
```

### Mapper Nâng Cao — Custom Mapping

```java
@Mapper(componentModel = "spring", uses = { RoleMapper.class })
public interface OrderMapper {

    // Ánh xạ field tên khác nhau
    @Mapping(source = "user.email", target = "customerEmail")
    @Mapping(source = "user.fullName", target = "customerName")
    @Mapping(target = "totalAmount", expression = "java(order.calculateTotal())")  // Custom expression
    @Mapping(target = "statusLabel", qualifiedByName = "mapStatusToLabel")
    OrderResponse toResponse(Order order);

    // Method custom được tham chiếu bởi qualifiedByName
    @Named("mapStatusToLabel")
    default String mapStatusToLabel(OrderStatus status) {
        return switch (status) {
            case PENDING -> "Chờ Xử Lý";
            case CONFIRMED -> "Đã Xác Nhận";
            case SHIPPED -> "Đang Vận Chuyển";
            case DELIVERED -> "Đã Giao Hàng";
            case CANCELLED -> "Đã Hủy";
        };
    }
}
```

---

## 6. Jackson — Cấu Hình Serialize/Deserialize JSON

### Cấu Hình Toàn Cục

```yaml
# application.yml
spring:
  jackson:
    # Định dạng ngày giờ
    date-format: yyyy-MM-dd'T'HH:mm:ss.SSSZ
    serialization:
      write-dates-as-timestamps: false   # Dùng chuỗi ISO 8601, không dùng số
      indent-output: false               # Compact JSON (không pretty-print) cho production
    deserialization:
      fail-on-unknown-properties: false  # Bỏ qua fields không biết — linh hoạt hơn
    default-property-inclusion: non_null # Không serialize fields null
    property-naming-strategy: SNAKE_CASE # Tự động camelCase → snake_case
```

### Jackson Annotations Quan Trọng

```java
@JsonIgnoreProperties(ignoreUnknown = true)  // Bỏ qua fields lạ khi deserialize
public class UserResponse {

    private Long id;

    @JsonProperty("full_name")    // Đặt tên khác trong JSON
    private String fullName;

    @JsonIgnore                   // Không serialize/deserialize field này
    private String passwordHash;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Ho_Chi_Minh")
    private LocalDateTime createdAt;

    @JsonInclude(JsonInclude.Include.NON_NULL)  // Bỏ qua nếu null (chỉ field này)
    private String middleName;

    @JsonSerialize(using = MoneySerializer.class)    // Custom serializer
    @JsonDeserialize(using = MoneyDeserializer.class) // Custom deserializer
    private BigDecimal price;
}
```

### Custom Serializer và Deserializer

```java
// Custom Serializer — serialize tiền tệ kèm ký hiệu
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, SerializerProvider provider)
        throws IOException {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toString() + " VND");
    }
}

// Custom Deserializer — parse chuỗi tiền tệ
public class MoneyDeserializer extends JsonDeserializer<BigDecimal> {
    @Override
    public BigDecimal deserialize(JsonParser p, DeserializationContext ctx)
        throws IOException {
        String value = p.getValueAsString().replace(" VND", "").replace(",", "");
        return new BigDecimal(value);
    }
}
```

### Xử Lý Enum trong JSON

```java
public enum OrderStatus {

    @JsonProperty("pending")    // JSON value là "pending", không phải "PENDING"
    PENDING,

    @JsonProperty("confirmed")
    CONFIRMED,

    @JsonProperty("shipped")
    SHIPPED
}

// Hoặc dùng annotation ở level class
@JsonFormat(shape = JsonFormat.Shape.STRING)  // Serialize thành chuỗi tên enum
public enum UserRole { ADMIN, USER, MODERATOR }
```

---

## 7. Handling Multipart — Upload File

```java
// DTO cho upload file
public record FileUploadRequest(
    @NotBlank String name,
    @NotNull MultipartFile file
) {}

@RestController
@RequestMapping("/api/v1/files")
public class FileController {

    private final FileStorageService fileStorageService;

    // Upload single file
    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<FileResponse> upload(
        @RequestParam("file") MultipartFile file,
        @RequestParam("category") String category
    ) {
        if (file.isEmpty()) {
            throw new BadRequestException("File không được rỗng");
        }

        String contentType = file.getContentType();
        if (!Set.of("image/jpeg", "image/png", "image/webp").contains(contentType)) {
            throw new BadRequestException("Chỉ chấp nhận JPEG, PNG, WebP");
        }

        if (file.getSize() > 5 * 1024 * 1024) {  // 5MB
            throw new BadRequestException("File không được vượt quá 5MB");
        }

        FileResponse response = fileStorageService.store(file, category);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    // Download file
    @GetMapping("/{filename}")
    public ResponseEntity<Resource> download(@PathVariable String filename) {
        Resource resource = fileStorageService.load(filename);
        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + resource.getFilename() + "\"")
            .body(resource);
    }
}
```

---

## 8. Best Practices — Thực Tiễn Tốt Nhất

### Phân Tách Request và Response DTO

```java
// ✅ Tách biệt rõ ràng
public record CreateProductRequest(...) {}  // Nhận từ client
public record UpdateProductRequest(...) {}  // Cập nhật toàn bộ
public record PatchProductRequest(...) {}   // Cập nhật một phần
public record ProductResponse(...) {}       // Gửi ra client
public record ProductSummaryResponse(...){} // Response rút gọn cho danh sách
```

### Validation Message Nhất Quán

```java
// ✅ Định nghĩa messages tập trung trong messages.properties
# src/main/resources/messages.properties
user.email.blank=Email không được để trống
user.email.invalid=Email không hợp lệ
user.password.weak=Mật khẩu phải có ít nhất 8 ký tự, bao gồm chữ hoa, số và ký tự đặc biệt

// Dùng trong annotation:
@NotBlank(message = "{user.email.blank}")
@Email(message = "{user.email.invalid}")
String email
```

### Dùng Records Cho DTO Bất Biến

```java
// ✅ Records — immutable, compact, no boilerplate
public record CreateOrderRequest(
    @NotNull Long userId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @NotBlank String shippingAddress
) {}

// ❌ Tránh dùng class với fields mutable khi không cần
public class CreateOrderRequest {
    private Long userId;          // Có thể bị thay đổi sau khi tạo
    private List<OrderItemRequest> items;
    private String shippingAddress;
    // getter, setter, constructor...  → Nhiều boilerplate
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao không expose Entity trực tiếp ra REST API?**

> (1) Bảo mật — tránh lộ thông tin nhạy cảm (password hash, internal fields). (2) Vòng đời độc lập — thay đổi DB schema không break API contract. (3) Tránh LazyInitializationException khi Jackson serialize lazy-loaded collections. (4) DTO cho phép thiết kế request/response phù hợp với nhu cầu client.

**Q: `@Valid` vs `@Validated` khác nhau thế nào?**

> `@Valid` (JSR-380) là annotation chuẩn Java, không hỗ trợ group validation. `@Validated` là annotation của Spring, hỗ trợ validation groups — dùng khi cần validate khác nhau giữa CREATE và UPDATE.

**Q: MapStruct vs ModelMapper vs manual mapping — khi nào dùng cái nào?**

> **MapStruct** (khuyến nghị): Code sinh lúc compile time → type-safe, hiệu suất tốt, IDE hỗ trợ tốt. **ModelMapper**: Dùng reflection runtime → flexible nhưng chậm hơn, ít type-safe. **Manual mapping**: Dùng khi logic phức tạp, mapping rất đơn giản, hoặc performance tối quan trọng.

**Q: `@NotNull` vs `@NotBlank` vs `@NotEmpty` khác nhau thế nào?**

> `@NotNull`: Không được null, nhưng chuỗi rỗng `""` vẫn hợp lệ. `@NotEmpty`: Không null và không rỗng (size > 0), nhưng chuỗi chỉ có spaces `"   "` vẫn hợp lệ. `@NotBlank`: Không null, không rỗng, không chỉ có khoảng trắng — nghiêm ngặt nhất.

---

## ✅ Checklist

- [ ] DTO tách biệt rõ: Request DTO, Response DTO, Summary DTO
- [ ] Bean Validation với @NotBlank, @Email, @Size, @Valid
- [ ] Custom validator cho logic nghiệp vụ đặc thù
- [ ] MapStruct mapper với @Mapping annotations
- [ ] Jackson cấu hình: date format, null handling, property naming
- [ ] Validation groups cho create/update khác nhau

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Tiếp Theo:** [3-exception-handling.md](3-exception-handling.md) — @ControllerAdvice & ProblemDetail
