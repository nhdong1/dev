# Spring Core — IoC Container & Dependency Injection

> Đây là bài quan trọng nhất trong toàn bộ knowledge base. Hiểu rõ IoC và DI là nền tảng
> để hiểu mọi thứ còn lại trong Spring Boot.

---

## 📋 Mục Tiêu

- [ ] Giải thích **IoC** (Inversion of Control — Đảo Ngược Quyền Kiểm Soát) và lý do tồn tại
- [ ] Mô tả **DI** (Dependency Injection — Tiêm Phụ Thuộc) và 3 dạng inject
- [ ] Phân biệt `ApplicationContext` và `BeanFactory`
- [ ] Sử dụng đúng các **Stereotype Annotations** (Annotation Phân Loại): `@Component`, `@Service`, `@Repository`, `@Controller`
- [ ] Hiểu cơ chế **Component Scanning** (Quét Thành Phần)
- [ ] Giải quyết **Circular Dependency** (Phụ Thuộc Vòng)

---

## 1. IoC — Inversion of Control (Đảo Ngược Quyền Kiểm Soát)

### Vấn Đề Cần Giải Quyết

Giả sử bạn có hệ thống xử lý đơn hàng:

```java
// ❌ Cách truyền thống — tight coupling (ghép chặt)
public class OrderService {
    // Tự tạo dependency — rất khó thay đổi, khó test
    private final UserRepository userRepository = new UserRepositoryImpl();
    private final EmailService emailService = new EmailServiceImpl();
    private final PaymentGateway paymentGateway = new StripePaymentGateway();

    public void placeOrder(Long userId, Order order) {
        User user = userRepository.findById(userId);
        paymentGateway.charge(user.getPaymentMethod(), order.getTotal());
        emailService.sendConfirmation(user.getEmail(), order);
    }
}
// Vấn đề:
// 1. OrderService bị "ghép chặt" với UserRepositoryImpl, EmailServiceImpl, StripePaymentGateway
// 2. Muốn đổi sang PayPal? Phải sửa OrderService
// 3. Muốn test mà không gọi thực Stripe? Không được!
// 4. Hard-coded dependencies — không linh hoạt
```

### IoC Giải Quyết Thế Nào

**IoC** đảo ngược quyền kiểm soát: thay vì class tự tạo dependency, nó **nhận** dependency từ bên ngoài (từ Container).

```java
// ✅ Với IoC — loose coupling (ghép lỏng)
public class OrderService {
    // Nhận dependency qua constructor — không tự tạo
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PaymentGateway paymentGateway;

    public OrderService(
        UserRepository userRepository,
        EmailService emailService,
        PaymentGateway paymentGateway
    ) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.paymentGateway = paymentGateway;
    }
    // Bây giờ OrderService không quan tâm đến implementation cụ thể
    // Có thể inject StripeGateway hoặc PayPalGateway — OrderService không đổi
}
```

```
Trước IoC:                    Sau IoC (Spring Container):
OrderService                  Spring Container
    │ new                     ┌─────────────────┐
    ├──→ UserRepositoryImpl   │ Quản lý objects │
    ├──→ EmailServiceImpl     │ và inject chúng │
    └──→ StripeGateway        └────────┬────────┘
                                       │ inject
                              ┌────────▼────────┐
                              │  OrderService   │
                              │  (nhận vào)    │
                              └─────────────────┘
```

---

## 2. DI — Dependency Injection (Tiêm Phụ Thuộc)

DI là cách Spring **triển khai** IoC — Spring tự động **tiêm** (inject) các dependency vào bean.

### 3 Dạng Dependency Injection

#### Dạng 1: Constructor Injection (Tiêm Qua Constructor) — **Khuyến Nghị**

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final EmailService emailService;
    private final PaymentService paymentService;

    // Spring tự động inject khi có đúng 1 constructor
    // Từ Spring 4.3+ không cần @Autowired nếu chỉ có 1 constructor
    public OrderService(
        OrderRepository orderRepository,
        EmailService emailService,
        PaymentService paymentService
    ) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
        this.paymentService = paymentService;
    }
}
```

**Ưu điểm của Constructor Injection:**
- Fields là `final` — **immutable** (bất biến), thread-safe (an toàn đa luồng)
- Dependencies **bắt buộc** — không thể tạo object thiếu dependency
- Dễ **unit test** — chỉ cần truyền mock vào constructor
- Phát hiện **circular dependency** ngay lúc khởi động app

#### Dạng 2: Setter Injection (Tiêm Qua Setter) — Dùng Cho Optional Dependency

```java
@Service
public class NotificationService {

    private EmailService emailService;
    private SmsService smsService; // optional — không phải lúc nào cũng có

    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }

    @Autowired(required = false) // optional dependency
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
}
```

#### Dạng 3: Field Injection (Tiêm Qua Field) — **Không Khuyến Nghị**

```java
@Service
public class UserService {

    @Autowired // ❌ Không khuyến nghị
    private UserRepository userRepository;

    @Autowired // ❌ Không khuyến nghị
    private PasswordEncoder passwordEncoder;
}
```

**Tại sao không nên dùng Field Injection:**
- Field là `private` — không thể inject trong unit test mà không dùng reflection
- Không thể khai báo `final` — object có thể bị thay đổi sau khi tạo
- Không rõ ràng về dependency khi nhìn từ bên ngoài class
- Che giấu vi phạm **Single Responsibility Principle** — khi class có quá nhiều dependencies

### @Autowired và Qualifier

```java
@Service
public class PaymentService {

    private final PaymentGateway paymentGateway;

    // Nếu có nhiều implementation của PaymentGateway
    public PaymentService(@Qualifier("stripeGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}

@Component("stripeGateway")
public class StripePaymentGateway implements PaymentGateway { ... }

@Component("paypalGateway")
public class PaypalPaymentGateway implements PaymentGateway { ... }
```

### @Primary — Ưu Tiên Khi Có Nhiều Implementation

```java
@Component
@Primary // Được ưu tiên khi inject mà không chỉ định Qualifier
public class StripePaymentGateway implements PaymentGateway { ... }

@Component
public class PaypalPaymentGateway implements PaymentGateway { ... }

// Khi inject PaymentGateway mà không dùng @Qualifier,
// Spring sẽ tự động chọn StripePaymentGateway
```

---

## 3. ApplicationContext vs BeanFactory

Cả hai đều là **IoC Container** nhưng `ApplicationContext` mạnh hơn nhiều.

### BeanFactory — Container Cơ Bản

```java
// BeanFactory — chỉ quản lý Bean lifecycle cơ bản
// Lazy loading — Bean chỉ được tạo khi lần đầu được yêu cầu
BeanFactory factory = new DefaultListableBeanFactory();
// Ít dùng trực tiếp trong thực tế
```

### ApplicationContext — Container Đầy Đủ Tính Năng

```java
// ApplicationContext = BeanFactory + nhiều tính năng bổ sung
// Eager loading — tất cả Singleton Beans được tạo khi khởi động
ApplicationContext context = SpringApplication.run(MyApplication.class, args);
```

| Tính Năng | BeanFactory | ApplicationContext |
|-----------|-------------|-------------------|
| Quản lý Bean lifecycle | ✅ | ✅ |
| Bean dependency injection | ✅ | ✅ |
| Internationalization (i18n) | ❌ | ✅ |
| Event publishing (Phát Sự Kiện) | ❌ | ✅ |
| AOP (Aspect-Oriented Programming) | ❌ | ✅ |
| @Transactional, @Async | ❌ | ✅ |
| Eager loading Singleton Beans | ❌ (lazy) | ✅ |

**Trong thực tế:** Luôn dùng `ApplicationContext`. `BeanFactory` chỉ dùng trong môi trường hạn chế tài nguyên cực đoan (IoT, embedded).

### Các Loại ApplicationContext

```java
// 1. AnnotationConfigApplicationContext — cho app thuần Java (không dùng web)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// 2. AnnotationConfigServletWebServerApplicationContext — cho Spring Boot Web
// (Spring Boot tự tạo khi chạy SpringApplication.run())

// 3. ClassPathXmlApplicationContext — cũ, dùng XML config
// (Không dùng trong dự án mới)
```

---

## 4. Stereotype Annotations — Annotation Phân Loại Bean

Spring cung cấp các annotation để đánh dấu class là Bean và phân loại vai trò của nó.

```
@Component (gốc)
    ├── @Service     (tầng business logic)
    ├── @Repository  (tầng truy cập dữ liệu)
    └── @Controller  (tầng web)
           └── @RestController (@Controller + @ResponseBody)
```

### @Component — Bean Tổng Quát

```java
@Component
public class PasswordHasher {
    public String hash(String plain) {
        return BCrypt.hashpw(plain, BCrypt.gensalt());
    }
}
// Dùng cho utility class, helper — không thuộc Service/Repository/Controller cụ thể
```

### @Service — Tầng Business Logic

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordHasher passwordHasher;
    private final EmailService emailService;

    public UserService(UserRepository repo, PasswordHasher hasher, EmailService email) {
        this.userRepository = repo;
        this.passwordHasher = hasher;
        this.emailService = email;
    }

    public User register(String name, String email, String rawPassword) {
        if (userRepository.existsByEmail(email)) {
            throw new EmailAlreadyExistsException(email);
        }
        String hashedPassword = passwordHasher.hash(rawPassword);
        User user = new User(name, email, hashedPassword);
        User saved = userRepository.save(user);
        emailService.sendWelcome(email);
        return saved;
    }
}
// @Service không thêm tính năng gì so với @Component về mặt kỹ thuật
// nhưng thể hiện rõ ý định — đây là tầng xử lý business logic
```

### @Repository — Tầng Truy Cập Dữ Liệu

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
// @Repository tự động kích hoạt:
// 1. Exception translation — chuyển database exceptions thành Spring DataAccessException
//    (SQLException → DataIntegrityViolationException, v.v.)
// 2. Không cần implements — Spring Data JPA tự implement
```

### @Controller vs @RestController

```java
// @Controller — trả về tên View (HTML template)
@Controller
public class HomeController {
    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Hello");
        return "home"; // trả về tên template (home.html, home.jsp)
    }
}

// @RestController = @Controller + @ResponseBody
// Tự động serialize object thành JSON/XML
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id); // tự động chuyển thành JSON
    }
}
```

---

## 5. Component Scanning — Quét Thành Phần

Spring tự động tìm và đăng ký Bean bằng cách **quét** (scan) các package để tìm class có annotation.

### Cơ Chế Hoạt Động

```java
@SpringBootApplication // bao gồm @ComponentScan
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

`@SpringBootApplication` kích hoạt Component Scan từ package của class này và **tất cả sub-packages**.

```
com.example.myapp/
├── MyApplication.java          ← @SpringBootApplication — quét từ đây
├── controller/
│   └── UserController.java     ← ✅ Tìm thấy @RestController
├── service/
│   └── UserService.java        ← ✅ Tìm thấy @Service
├── repository/
│   └── UserRepository.java     ← ✅ Tìm thấy @Repository
└── util/
    └── PasswordHasher.java     ← ✅ Tìm thấy @Component
```

### Cấu Hình Thủ Công Package Scanning

```java
// Scan nhiều package
@ComponentScan(basePackages = {
    "com.example.myapp",
    "com.example.shared"
})

// Loại trừ một số class
@ComponentScan(
    basePackages = "com.example.myapp",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Configuration.class
    )
)
```

---

## 6. @Configuration và @Bean — Cấu Hình Bằng Java

Khi không thể thêm annotation vào class (class của thư viện bên ngoài), dùng `@Configuration` và `@Bean`:

```java
@Configuration
public class AppConfig {

    // Tạo Bean cho BCryptPasswordEncoder (class của Spring Security — không sửa được)
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // strength = 12
    }

    // Tạo Bean cho ObjectMapper với cấu hình tùy chỉnh
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return mapper;
    }

    // Bean phụ thuộc vào Bean khác — Spring tự inject
    @Bean
    public UserService userService(UserRepository repo, PasswordEncoder encoder) {
        return new UserService(repo, encoder);
    }
}
```

### @Bean Scope Và Lifecycle

```java
@Configuration
public class CacheConfig {

    @Bean
    @Scope("singleton") // Mặc định — chỉ tạo một lần
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "products");
    }

    @Bean
    @Scope("prototype") // Tạo mới mỗi lần inject
    public HttpClient httpClient() {
        return HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();
    }
}
```

---

## 7. Circular Dependency — Phụ Thuộc Vòng

**Circular Dependency** xảy ra khi Bean A phụ thuộc vào Bean B, và Bean B cũng phụ thuộc vào Bean A.

### Phát Hiện Circular Dependency

```java
// Spring Boot 2.6+ tự động phát hiện và throw exception khi khởi động
// BeanCurrentlyInCreationException: Is there an unresolvable circular reference?

@Service
public class ServiceA {
    public ServiceA(ServiceB b) { ... } // phụ thuộc ServiceB
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { ... } // phụ thuộc ServiceA — vòng tròn!
}
```

### Cách Giải Quyết

**Cách 1: Refactor — Trích xuất phần chung ra class mới (Tốt nhất)**

```java
// Thay vì A→B và B→A, tạo class C chứa logic chung
@Service
public class SharedService {
    public void doCommonWork() { ... }
}

@Service
public class ServiceA {
    public ServiceA(SharedService shared) { ... } // A → Shared
}

@Service
public class ServiceB {
    public ServiceB(SharedService shared) { ... } // B → Shared
}
```

**Cách 2: @Lazy — Trì Hoãn Khởi Tạo**

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;

    public ServiceA(@Lazy ServiceB serviceB) { // inject lazy proxy
        this.serviceB = serviceB;
    }
}
// Cảnh báo: @Lazy tạo proxy — có thể ẩn vấn đề thiết kế
```

**Cách 3: Setter Injection (Kém hơn — chỉ dùng nếu bắt buộc)**

```java
@Service
public class ServiceA {
    private ServiceB serviceB;

    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
// Setter injection cho phép Spring tạo Bean trước rồi inject sau
// Nhưng field không final — kém an toàn hơn
```

**Nguyên tắc:** Circular Dependency thường là dấu hiệu thiết kế chưa tốt. Hãy xem xét lại responsibility của từng class.

---

## 8. Spring AOP — Aspect-Oriented Programming (Lập Trình Hướng Khía Cạnh)

AOP cho phép thêm behavior (hành vi) vào Bean mà không sửa code gốc — dùng cho logging, transaction, security.

### Proxy Pattern Trong Spring

```
Client code                Spring AOP Proxy         Actual Bean
      │                         │                       │
      │ method call             │                       │
      ├───────────────────────→ │                       │
      │                         │ (before advice)       │
      │                         │ ─────────────────────→│
      │                         │                       │ execute
      │                         │ ←─────────────────────│
      │                         │ (after advice)        │
      │ ←───────────────────────│                       │
```

Spring tạo **Proxy** (đối tượng bọc) quanh Bean. Khi gọi method trên Bean, thực ra đang gọi Proxy — Proxy thực hiện các **Advice** (lời khuyên) trước/sau khi gọi method thực.

### @Transactional Dùng AOP

```java
@Service
public class OrderService {

    @Transactional // Spring tạo proxy — tự bắt đầu và kết thúc transaction
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        inventoryService.deduct(request.items()); // nếu fail → rollback cả 2
        return order;
    }
}
// Đây là ví dụ quen thuộc nhất về AOP trong Spring
```

### Lỗi Thường Gặp Với AOP Proxy

```java
@Service
public class UserService {

    @Transactional
    public void updateUser(User user) {
        // ...
    }

    public void processUsers(List<User> users) {
        for (User user : users) {
            this.updateUser(user); // ❌ Gọi this.method() — bỏ qua proxy!
        }
    }
}
// Khi gọi this.updateUser() từ trong class, gọi trực tiếp Bean, không qua Proxy
// → @Transactional bị bỏ qua!

// ✅ Giải pháp: inject chính class vào (self-injection) hoặc tách thành 2 class
@Service
public class UserService {

    private final UserTransactionService txService;

    public UserService(UserTransactionService txService) {
        this.txService = txService;
    }

    public void processUsers(List<User> users) {
        for (User user : users) {
            txService.updateUser(user); // ✅ Gọi qua bean khác → đi qua proxy
        }
    }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Sự khác biệt giữa @Component, @Service, @Repository, @Controller?

**Trả lời:**
- Về kỹ thuật, tất cả đều là `@Component` — đều đăng ký class là Bean với Spring Container
- Điểm khác biệt chính là **semantic** (ý nghĩa) và **tính năng bổ sung**:
  - `@Repository`: kích hoạt **exception translation** — chuyển database exceptions thành Spring DataAccessException hierarchy
  - `@Controller`: đánh dấu class xử lý HTTP request, được nhận biết bởi `DispatcherServlet`
  - `@Service`: không thêm tính năng gì, nhưng thể hiện rõ đây là tầng business logic
  - `@Component`: dùng cho các class không thuộc 3 tầng trên

### Q: Constructor Injection vs Field Injection — tại sao khuyến nghị Constructor?

**Trả lời:**
1. **Immutability** — fields có thể khai báo `final`, đảm bảo object không bị thay đổi sau khi tạo
2. **Testability** — dễ inject mock trong unit test chỉ bằng `new MyService(mockRepo, mockEmail)`
3. **Fail-fast** — circular dependency được phát hiện ngay lúc khởi động, không phải lúc runtime
4. **Explicit dependencies** — dependencies hiện ra rõ ràng trong signature constructor
5. Khi constructor có quá nhiều parameters → dấu hiệu class đang vi phạm Single Responsibility, cần refactor

### Q: @Autowired hoạt động thế nào?

**Trả lời:** Spring Container scan tất cả Bean đã đăng ký, tìm Bean có kiểu phù hợp và inject vào. Nếu tìm thấy nhiều hơn một candidate:
1. Dùng `@Primary` để chỉ định ưu tiên
2. Dùng `@Qualifier("beanName")` để chỉ định chính xác
3. Dùng tên biến khớp với Bean name

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích IoC và tại sao cần IoC không cần nhìn tài liệu
- [ ] Viết Service với Constructor Injection cho 2–3 dependencies
- [ ] Tạo @Configuration class với @Bean cho thư viện bên ngoài
- [ ] Phân biệt khi nào dùng @Service, @Repository, @Component
- [ ] Giải thích tại sao `this.method()` không kích hoạt @Transactional
- [ ] Giải quyết circular dependency bằng cách refactor

---

**Tiếp Theo:** [3-spring-boot-basics.md](3-spring-boot-basics.md) — Auto-configuration và Starters
