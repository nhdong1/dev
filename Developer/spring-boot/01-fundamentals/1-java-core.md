# Java Core Hiện Đại — Java 17+

> Tổng quan các tính năng Java hiện đại quan trọng nhất mà mọi Spring Boot developer cần nắm vững.
> Tập trung vào Java 17 (LTS) và các tính năng đã được sử dụng rộng rãi trong thực tế.

---

## 📋 Mục Tiêu

- [ ] Hiểu và sử dụng **Records** để tạo immutable data class ngắn gọn
- [ ] Dùng **Sealed Classes** (Lớp Sealed) để mô hình hóa domain có giới hạn
- [ ] Làm chủ **Stream API** (API Luồng) cho xử lý tập hợp dữ liệu
- [ ] Sử dụng **Optional** để xử lý giá trị null an toàn
- [ ] Hiểu **Virtual Threads** (Luồng Ảo) và ứng dụng trong Spring Boot
- [ ] Dùng **Pattern Matching** (Khớp Mẫu) để code gọn hơn

---

## 1. Records — Lớp Dữ Liệu Bất Biến

**Records** (ra mắt Java 16, stable) là cú pháp ngắn gọn để tạo **immutable data class** (lớp dữ liệu bất biến).

### Tại Sao Cần Records?

Trước Java 16, để tạo một class chỉ chứa dữ liệu, bạn phải viết rất nhiều boilerplate code (code lặp không cần thiết):

```java
// Cách cũ — quá nhiều code thừa
public class UserDto {
    private final String name;
    private final String email;
    private final int age;

    public UserDto(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }

    public String getName()  { return name; }
    public String getEmail() { return email; }
    public int getAge()      { return age; }

    @Override
    public boolean equals(Object o) { /* ... */ }
    @Override
    public int hashCode() { /* ... */ }
    @Override
    public String toString() { /* ... */ }
}

// Với Records — chỉ 1 dòng, tương đương hoàn toàn
public record UserDto(String name, String email, int age) {}
```

### Đặc Điểm Của Records

```java
public record UserDto(String name, String email, int age) {}

// Records tự động có:
// 1. Constructor có tất cả fields
// 2. Accessor methods: name(), email(), age() (không có get prefix)
// 3. equals(), hashCode() dựa trên tất cả fields
// 4. toString() dễ đọc

UserDto user = new UserDto("Alice", "alice@example.com", 25);
System.out.println(user.name());   // Alice
System.out.println(user.email());  // alice@example.com
System.out.println(user);          // UserDto[name=Alice, email=alice@example.com, age=25]
```

### Records Trong Spring Boot — DTO Pattern

Records rất phù hợp để làm **DTO** (Data Transfer Object — Đối Tượng Truyền Dữ Liệu):

```java
// Request DTO — nhận dữ liệu từ client
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Min(18) int age
) {}

// Response DTO — trả dữ liệu về cho client
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}

// Trong Controller
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(
    @Valid @RequestBody CreateUserRequest request) {
    // request.name(), request.email(), request.age()
    UserResponse response = userService.create(request);
    return ResponseEntity.status(201).body(response);
}
```

### Compact Constructor — Validation Trong Records

```java
public record MoneyAmount(BigDecimal value, String currency) {

    // Compact constructor — validate ngay khi tạo object
    public MoneyAmount {
        Objects.requireNonNull(value, "value must not be null");
        Objects.requireNonNull(currency, "currency must not be null");
        if (value.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("value must be non-negative");
        }
        currency = currency.toUpperCase(); // normalize
    }
}
```

### Records Với Jackson (JSON Serialization)

Spring Boot dùng Jackson để chuyển đổi JSON ↔ Java object:

```java
// Records tương thích tốt với Jackson từ Spring Boot 2.7+
// Không cần thêm cấu hình gì thêm

public record ProductDto(Long id, String name, BigDecimal price) {}

// GET /products/1 sẽ trả về:
// {"id": 1, "name": "Laptop", "price": 29999.99}
```

---

## 2. Sealed Classes — Lớp Sealed (Giới Hạn Kế Thừa)

**Sealed Classes** (Java 17) cho phép bạn **kiểm soát chính xác** những lớp nào được phép kế thừa.

### Tại Sao Cần Sealed Classes?

Khi bạn muốn mô hình hóa một tập hợp **closed** (đóng) các trường hợp, ví dụ kết quả một thao tác:

```java
// Không dùng sealed — bất kỳ ai cũng có thể tạo subclass
public abstract class PaymentResult {}
public class Success extends PaymentResult {}
public class Failure extends PaymentResult {}
// Ai đó ở đâu đó có thể tạo thêm subclass không kiểm soát được

// Dùng sealed — chỉ cho phép Success, Failure, Pending kế thừa
public sealed class PaymentResult
    permits PaymentSuccess, PaymentFailure, PaymentPending {}

public final class PaymentSuccess extends PaymentResult {
    private final String transactionId;
    public PaymentSuccess(String transactionId) {
        this.transactionId = transactionId;
    }
    public String transactionId() { return transactionId; }
}

public final class PaymentFailure extends PaymentResult {
    private final String errorCode;
    private final String message;
    // constructor, accessors...
}

public final class PaymentPending extends PaymentResult {
    private final String referenceId;
    // constructor, accessors...
}
```

### Kết Hợp Sealed Classes + Pattern Matching

Java 21 có **Pattern Matching for switch** — rất mạnh khi kết hợp với Sealed Classes:

```java
// Compiler biết chính xác các trường hợp có thể xảy ra
String message = switch (result) {
    case PaymentSuccess s  -> "Thanh toán thành công: " + s.transactionId();
    case PaymentFailure f  -> "Lỗi " + f.errorCode() + ": " + f.message();
    case PaymentPending p  -> "Đang xử lý, ref: " + p.referenceId();
    // Không cần default vì đã cover hết các trường hợp
};
```

### Records + Sealed Classes — Domain Modeling

```java
// Kết hợp cả hai để mô hình hóa domain một cách rõ ràng
public sealed interface ValidationResult
    permits ValidationResult.Valid, ValidationResult.Invalid {}

public record Valid(String value) implements ValidationResult {}
public record Invalid(String value, List<String> errors) implements ValidationResult {}

// Sử dụng
ValidationResult result = validate(input);
String response = switch (result) {
    case Valid v    -> "OK: " + v.value();
    case Invalid i  -> "Lỗi: " + String.join(", ", i.errors());
};
```

---

## 3. Stream API — API Xử Lý Dữ Liệu Theo Luồng

**Stream API** (Java 8+) là cách khai báo (declarative) để xử lý tập hợp dữ liệu — tránh dùng vòng lặp thủ công.

### Các Thao Tác Cơ Bản

```java
List<User> users = userRepository.findAll();

// filter — lọc phần tử thỏa điều kiện
List<User> activeUsers = users.stream()
    .filter(u -> u.isActive())
    .toList();  // Java 16+ — không cần .collect(Collectors.toList())

// map — chuyển đổi từng phần tử
List<String> emails = users.stream()
    .map(User::getEmail)
    .toList();

// sorted — sắp xếp
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getName))
    .toList();

// distinct — loại bỏ trùng lặp
List<String> uniqueDomains = users.stream()
    .map(u -> u.getEmail().split("@")[1])
    .distinct()
    .toList();
```

### Intermediate vs Terminal Operations (Thao Tác Trung Gian vs Kết Thúc)

```java
users.stream()
    .filter(User::isActive)      // intermediate — lazy, chưa chạy
    .map(User::getName)          // intermediate — lazy, chưa chạy
    .sorted()                    // intermediate — lazy, chưa chạy
    .toList();                   // terminal — thực sự chạy toàn bộ pipeline
//            ↑ Toàn bộ pipeline chỉ chạy khi gặp terminal operation
```

### Các Terminal Operations Quan Trọng

```java
// count — đếm số phần tử
long count = users.stream().filter(User::isActive).count();

// findFirst — lấy phần tử đầu tiên
Optional<User> first = users.stream()
    .filter(u -> u.getAge() > 18)
    .findFirst();

// anyMatch / allMatch / noneMatch — kiểm tra điều kiện
boolean hasAdmin = users.stream().anyMatch(u -> u.getRole() == Role.ADMIN);
boolean allVerified = users.stream().allMatch(User::isEmailVerified);

// reduce — gộp tất cả thành một giá trị
int totalAge = users.stream()
    .mapToInt(User::getAge)
    .sum();  // shortcut của reduce cho int/long/double

// collect — thu thập vào collection
Map<Role, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole));

// joining — nối String
String namesList = users.stream()
    .map(User::getName)
    .collect(Collectors.joining(", ", "[", "]"));
// Kết quả: [Alice, Bob, Charlie]
```

### flatMap — Xử Lý Stream Lồng Nhau

```java
List<Order> orders = orderRepository.findAll();

// Lấy tất cả OrderItem từ tất cả Orders
List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())  // "flatten" — làm phẳng
    .toList();

// Tính tổng giá trị tất cả đơn hàng
BigDecimal total = orders.stream()
    .flatMap(o -> o.getItems().stream())
    .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

### Stream Trong Thực Tế — Ví Dụ Service Layer

```java
@Service
public class ReportService {

    public ProductSalesReport buildReport(List<Order> orders) {
        // Tính doanh thu theo sản phẩm
        Map<String, BigDecimal> revenueByProduct = orders.stream()
            .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
            .flatMap(o -> o.getItems().stream())
            .collect(Collectors.groupingBy(
                OrderItem::getProductName,
                Collectors.reducing(
                    BigDecimal.ZERO,
                    item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())),
                    BigDecimal::add
                )
            ));

        // Top 10 sản phẩm bán chạy nhất
        List<String> topProducts = revenueByProduct.entrySet().stream()
            .sorted(Map.Entry.<String, BigDecimal>comparingByValue().reversed())
            .limit(10)
            .map(Map.Entry::getKey)
            .toList();

        return new ProductSalesReport(revenueByProduct, topProducts);
    }
}
```

---

## 4. Optional — Xử Lý Null An Toàn

**Optional** (Java 8+) là wrapper xung quanh một giá trị có thể null, giúp tránh `NullPointerException`.

### Tạo Optional

```java
Optional<User> withValue  = Optional.of(user);          // user KHÔNG được null
Optional<User> nullable   = Optional.ofNullable(user);  // user CÓ THỂ null
Optional<User> empty      = Optional.empty();            // luôn rỗng
```

### Cách Dùng Đúng Optional

```java
// Trong Repository — trả về Optional khi kết quả có thể không tồn tại
Optional<User> userOpt = userRepository.findByEmail("alice@example.com");

// Cách 1: orElseThrow — throw exception nếu không có
User user = userOpt.orElseThrow(() ->
    new UserNotFoundException("User not found: alice@example.com"));

// Cách 2: orElse — giá trị mặc định
User user = userOpt.orElse(User.guest());

// Cách 3: orElseGet — lazy — chỉ tính khi cần
User user = userOpt.orElseGet(() -> userService.createGuestUser());

// Cách 4: map + orElse — chuyển đổi nếu có
String email = userOpt.map(User::getEmail).orElse("unknown");

// Cách 5: ifPresent — thực hiện hành động nếu có
userOpt.ifPresent(u -> emailService.sendWelcome(u.getEmail()));

// Cách 6: filter — lọc thêm điều kiện
Optional<User> activeUser = userOpt.filter(User::isActive);
```

### Lỗi Thường Gặp Với Optional

```java
// ❌ KHÔNG làm thế này — mất đi lợi ích của Optional
Optional<User> opt = userRepository.findById(id);
if (opt.isPresent()) {
    User user = opt.get();  // get() chỉ dùng sau isPresent() — nhưng vẫn không phải best practice
}

// ✅ Dùng orElseThrow hoặc map
User user = userRepository.findById(id)
    .orElseThrow(() -> new EntityNotFoundException("User " + id + " not found"));

// ❌ KHÔNG dùng Optional làm field trong class
public class User {
    private Optional<String> nickname;  // SAI — Optional không implement Serializable
}

// ✅ Dùng @Nullable hoặc kiểm tra null trực tiếp cho field
public class User {
    @Nullable
    private String nickname;  // ĐÚNG
    public Optional<String> getNickname() {
        return Optional.ofNullable(nickname);
    }
}
```

---

## 5. Virtual Threads — Luồng Ảo (Java 21)

**Virtual Threads** (Java 21, Project Loom) là luồng nhẹ được JVM quản lý, giải quyết vấn đề **thread per request** (mỗi yêu cầu một luồng) bị bottleneck.

### Vấn Đề Trước Virtual Threads

```
Traditional Model (Mô Hình Truyền Thống):
┌───────────────────────────────────────────────────┐
│  HTTP Request → Platform Thread (OS Thread)        │
│                                                    │
│  Thread 1: [=====WAIT DB=====][==process==]        │
│  Thread 2: [========WAIT HTTP========][=process=]  │
│  Thread 3: [===WAIT===][===process===]             │
│  ...                                               │
│  Thread N: BLOCKED — chờ request mới               │
│                                                    │
│  Vấn đề: Thread Pool = 200 → max 200 concurrent   │
│  requests, dù CPU chỉ dùng 5%!                    │
└───────────────────────────────────────────────────┘
```

### Virtual Threads Giải Quyết Thế Nào

```
Virtual Threads Model:
┌───────────────────────────────────────────────────────┐
│  HTTP Request → Virtual Thread (JVM managed)          │
│                                                        │
│  VThread 1: [=====WAIT DB=====] ← unmounted từ carrier│
│             Carrier thread làm việc khác              │
│  VThread 1: [resumes when DB done][==process==]       │
│                                                        │
│  Có thể có HÀNG TRIỆU virtual threads đồng thời       │
│  mà không cần hàng triệu OS threads!                  │
└───────────────────────────────────────────────────────┘
```

### Bật Virtual Threads Trong Spring Boot

```yaml
# application.yml — Spring Boot 3.2+
spring:
  threads:
    virtual:
      enabled: true  # Bật là xong! Không cần thay đổi code
```

```java
// Hoặc cấu hình manual
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutor() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

### Khi Nào Virtual Threads Có Ích

```
✅ Phù hợp (I/O-bound workloads — công việc chờ I/O):
   - REST API calls đến external services
   - Database queries
   - File I/O operations
   - Gọi Kafka, RabbitMQ

❌ Không cải thiện nhiều (CPU-bound workloads — công việc tính toán):
   - Nén/giải nén dữ liệu
   - Xử lý ảnh/video
   - Mã hóa phức tạp
   - Machine learning inference
```

---

## 6. Pattern Matching — Khớp Mẫu

### instanceof Pattern Matching (Java 16+)

```java
// Cách cũ — phải cast thủ công
Object obj = getObject();
if (obj instanceof String) {
    String s = (String) obj;  // phải cast
    System.out.println(s.length());
}

// Cách mới — kết hợp kiểm tra và khai báo biến
if (obj instanceof String s) {
    System.out.println(s.length());  // s đã là String, không cần cast
}

// Ứng dụng trong Service Layer
public String formatValue(Object value) {
    if (value instanceof Integer i)        return "Số nguyên: " + i;
    if (value instanceof Double d)         return String.format("Số thực: %.2f", d);
    if (value instanceof String s)         return "Chuỗi: " + s;
    if (value instanceof List<?> list)     return "Danh sách có " + list.size() + " phần tử";
    return "Không xác định: " + value;
}
```

### Switch Pattern Matching (Java 21)

```java
// Rất mạnh khi kết hợp với Sealed Classes
public String processEvent(DomainEvent event) {
    return switch (event) {
        case UserCreated e   -> "Người dùng mới: " + e.username();
        case UserDeleted e   -> "Xóa người dùng: " + e.userId();
        case OrderPlaced e   -> "Đặt hàng #" + e.orderId() + " bởi " + e.userId();
        case PaymentReceived e -> "Thanh toán " + e.amount() + " cho đơn " + e.orderId();
        // Compiler cảnh báo nếu bạn thiếu một case
    };
}
```

---

## 7. Text Blocks — Chuỗi Nhiều Dòng (Java 15+)

```java
// Cách cũ — khó đọc
String json = "{\n" +
    "    \"name\": \"" + name + "\",\n" +
    "    \"email\": \"" + email + "\"\n" +
    "}";

// Text Block — dễ đọc, giữ nguyên formatting
String json = """
    {
        "name": "%s",
        "email": "%s"
    }
    """.formatted(name, email);

// Rất hữu ích cho SQL queries
@Query("""
    SELECT u FROM User u
    WHERE u.active = true
    AND u.createdAt > :since
    ORDER BY u.name
    """)
List<User> findActiveUsersSince(@Param("since") LocalDateTime since);
```

---

## 8. Generics Quan Trọng Cần Nhớ

### Bounded Type Parameters (Tham Số Kiểu Có Giới Hạn)

```java
// <T extends Comparable<T>> — T phải implement Comparable
public <T extends Comparable<T>> T max(List<T> list) {
    return list.stream().max(Comparator.naturalOrder()).orElseThrow();
}

// Wildcard — ? extends / ? super
public void printAll(List<? extends Number> numbers) {
    numbers.forEach(System.out::println);  // Producer — dùng extends
}

public void addNumbers(List<? super Integer> list) {
    list.add(42);  // Consumer — dùng super
}
// Quy tắc PECS: Producer Extends, Consumer Super
```

### Generic Trong Spring Boot

```java
// Repository với Generic — JpaRepository<Entity, IdType>
public interface UserRepository extends JpaRepository<User, Long> {}

// Service với Generic Response wrapper
public record ApiResponse<T>(T data, String message, boolean success) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(data, "Success", true);
    }
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(null, message, false);
    }
}

// Controller
@GetMapping("/users/{id}")
public ResponseEntity<ApiResponse<UserResponse>> getUser(@PathVariable Long id) {
    UserResponse user = userService.findById(id);
    return ResponseEntity.ok(ApiResponse.ok(user));
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Record vs Class thông thường — khi nào dùng Record?

**Trả lời:** Dùng Record khi:
1. Class chỉ dùng để **chứa dữ liệu** (data carrier) — DTO, Value Object
2. Object cần **immutable** (bất biến) — không thay đổi sau khi tạo
3. Muốn equals/hashCode dựa trên **tất cả fields**

Không dùng Record khi:
1. Cần kế thừa (Record không thể extend class khác)
2. Cần mutable fields
3. Cần lazy initialization

### Q: Stream API lazy evaluation là gì? Tại sao quan trọng?

**Trả lời:** Intermediate operations (filter, map, sorted...) không thực thi ngay mà chỉ xây dựng **pipeline**. Terminal operation mới kích hoạt thực thi. Lợi ích:
- **Short-circuit** — `findFirst()` dừng lại khi tìm được phần tử đầu tiên, không cần duyệt hết
- **Tiết kiệm bộ nhớ** — không tạo collection trung gian cho từng bước
- **Tối ưu hóa** — JVM có thể tối ưu hóa pipeline tốt hơn

### Q: Virtual Threads khác Platform Threads thế nào?

| Tiêu chí | Platform Thread | Virtual Thread |
|----------|----------------|----------------|
| Quản lý bởi | OS | JVM |
| Số lượng | Hàng nghìn | Hàng triệu |
| Stack size | ~1MB mặc định | Nhỏ hơn, co dãn |
| Chi phí tạo | Cao | Rất thấp |
| Phù hợp | CPU-bound | I/O-bound |

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Tạo được Record với compact constructor có validation
- [ ] Dùng Sealed Classes để model hóa domain với 3–4 trường hợp
- [ ] Viết Stream pipeline có filter, map, collect, groupingBy
- [ ] Xử lý Optional đúng cách — không dùng `.get()` trực tiếp
- [ ] Bật Virtual Threads trong Spring Boot và giải thích khi nào có ích
- [ ] Dùng Pattern Matching với instanceof và switch

---

## 🔗 Tài Liệu Tham Khảo

- [JEP 395 — Records](https://openjdk.org/jeps/395)
- [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [Baeldung — Java 17 New Features](https://www.baeldung.com/java-17-new-features)

---

**Tiếp Theo:** [2-spring-core.md](2-spring-core.md) — IoC Container và Dependency Injection
