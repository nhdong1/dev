# Unit Testing với JUnit 5

> Unit Test (Kiểm Thử Đơn Vị) là nền tảng của mọi chiến lược kiểm thử —
> kiểm tra từng class/method một cách cô lập, không phụ thuộc vào database, network hay Spring Context.
> JUnit 5 (Jupiter) là framework kiểm thử chuẩn cho Java hiện đại.

---

## 1. Tại Sao Unit Test Quan Trọng?

```
Không có Unit Test:
  Bug xuất hiện ở production → debug mất giờ → fix → deploy lại
  Mỗi thay đổi = nỗi sợ hãi regression (hỏng chức năng cũ)

Có Unit Test:
  Bug bị bắt trong vài giây khi chạy test → fix ngay
  Refactor (tái cấu trúc) tự tin vì test bảo vệ behavior
  Code trở thành documentation (tài liệu sống) cho người đọc
```

---

## 2. Cài Đặt JUnit 5 trong Spring Boot

Spring Boot Starter Test bao gồm JUnit 5 theo mặc định:

```xml
<!-- pom.xml — Spring Boot Starter Test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- Bao gồm: JUnit 5, Mockito, AssertJ, Hamcrest, JSONAssert, JsonPath -->
</dependency>
```

### Các Thư Viện Được Bao Gồm

| Thư Viện | Phiên Bản | Mục Đích |
|----------|-----------|----------|
| **JUnit Jupiter** | 5.x | Framework kiểm thử cốt lõi |
| **Mockito** | 5.x | Mocking framework (Khung Giả Lập) |
| **AssertJ** | 3.x | Fluent assertions (Khẳng Định Linh Hoạt) |
| **Hamcrest** | 2.x | Matcher library |
| **JSONAssert** | 1.x | JSON assertion |
| **JsonPath** | 2.x | JSON path expressions |

---

## 3. Cấu Trúc Một Unit Test Chuẩn

### Pattern AAA — Arrange, Act, Assert

```java
@ExtendWith(MockitoExtension.class)  // ← Kích hoạt Mockito trong JUnit 5
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;   // ← Arrange: setup mock

    @Mock
    private PaymentService paymentService;

    @InjectMocks
    private OrderService orderService;         // ← Class cần test

    @Test
    void createOrder_withValidRequest_shouldReturnOrderId() {
        // Arrange (Chuẩn Bị) — setup dữ liệu và mock behavior
        CreateOrderRequest request = new CreateOrderRequest("product-1", 2, 50_000L);
        Order savedOrder = Order.builder()
                .id(UUID.randomUUID())
                .productId("product-1")
                .quantity(2)
                .totalPrice(100_000L)
                .status(OrderStatus.PENDING)
                .build();

        when(orderRepository.save(any(Order.class))).thenReturn(savedOrder);
        when(paymentService.charge(any(), anyLong())).thenReturn(true);

        // Act (Thực Hiện) — gọi method cần test
        UUID orderId = orderService.createOrder(request);

        // Assert (Khẳng Định) — xác minh kết quả
        assertThat(orderId).isNotNull();
        assertThat(orderId).isEqualTo(savedOrder.getId());
        verify(orderRepository).save(any(Order.class));
        verify(paymentService).charge(any(), eq(100_000L));
    }
}
```

---

## 4. JUnit 5 Annotations (Chú Thích)

### Annotations Cơ Bản

```java
class CalculatorTest {

    @Test                           // ← Đánh dấu đây là test method
    void add_twoPositiveNumbers_returnsSum() {
        assertThat(2 + 3).isEqualTo(5);
    }

    @Test
    @DisplayName("Kiểm tra chia cho số 0 phải ném ArithmeticException")
    void divide_byZero_throwsException() {
        assertThrows(ArithmeticException.class, () -> 10 / 0);
    }

    @Test
    @Disabled("Bug #123 — chưa fix, skip tạm")
    void brokenFeature_shouldBeFixed() {
        // Test này bị skip — không chạy
    }
}
```

### Annotations Lifecycle (Vòng Đời Test)

```java
class UserServiceTest {

    private UserService userService;
    private List<String> testLog;

    @BeforeAll                      // ← Chạy MỘT LẦN trước tất cả tests trong class
    static void setupClass() {
        System.out.println("Khởi tạo resources tốn kém một lần");
    }

    @AfterAll                       // ← Chạy MỘT LẦN sau tất cả tests trong class
    static void teardownClass() {
        System.out.println("Dọn dẹp resources");
    }

    @BeforeEach                     // ← Chạy TRƯỚC MỖI test method
    void setUp() {
        userService = new UserService();
        testLog = new ArrayList<>();
    }

    @AfterEach                      // ← Chạy SAU MỖI test method
    void tearDown() {
        testLog.clear();
    }

    @Test
    void test1() { /* userService và testLog luôn fresh */ }

    @Test
    void test2() { /* Không bị ảnh hưởng bởi test1 */ }
}
```

---

## 5. Assertions với AssertJ

AssertJ cung cấp fluent API (API Linh Hoạt) giúp test dễ đọc hơn JUnit assertions thuần túy.

### Assertions Cơ Bản

```java
import static org.assertj.core.api.Assertions.*;

@Test
void assertjExamples() {
    // Primitive và String
    assertThat(42).isEqualTo(42);
    assertThat("hello").isEqualTo("hello")
                       .startsWith("he")
                       .endsWith("lo")
                       .hasSize(5)
                       .isNotBlank();

    // Boolean
    assertThat(true).isTrue();
    assertThat(false).isFalse();

    // Null checking
    assertThat(null).isNull();
    assertThat("value").isNotNull();

    // Số học
    assertThat(3.14).isCloseTo(3.0, within(0.2));
    assertThat(100).isBetween(50, 200);
    assertThat(-5).isNegative();
    assertThat(10).isPositive().isGreaterThan(5);
}
```

### Assertions cho Collections (Bộ Sưu Tập)

```java
@Test
void collectionAssertions() {
    List<String> fruits = List.of("apple", "banana", "cherry");

    assertThat(fruits)
        .hasSize(3)
        .contains("apple", "banana")           // chứa các phần tử này (bất kỳ thứ tự)
        .containsExactly("apple", "banana", "cherry")  // đúng thứ tự
        .doesNotContain("mango")
        .isNotEmpty();

    // Filter và kiểm tra
    assertThat(fruits)
        .filteredOn(f -> f.startsWith("b"))
        .containsOnly("banana");

    // Assertions trên từng phần tử
    List<User> users = List.of(
        new User("Alice", 25),
        new User("Bob", 30)
    );
    assertThat(users)
        .extracting(User::getName)
        .containsExactly("Alice", "Bob");

    assertThat(users)
        .extracting("name", "age")
        .containsExactly(
            tuple("Alice", 25),
            tuple("Bob", 30)
        );
}
```

### Exception Assertions

```java
@Test
void exceptionAssertions() {
    // Cách 1: assertThrows (JUnit 5)
    ArithmeticException ex = assertThrows(
        ArithmeticException.class,
        () -> divide(10, 0)
    );
    assertThat(ex.getMessage()).contains("by zero");

    // Cách 2: assertThatThrownBy (AssertJ — fluent hơn)
    assertThatThrownBy(() -> userService.findById(-1L))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("ID không được âm")
        .hasNoCause();

    // Kiểm tra không ném exception
    assertThatNoException().isThrownBy(() -> userService.findById(1L));
}
```

### Soft Assertions — Kiểm Tra Nhiều Điều Cùng Lúc

```java
@Test
void softAssertions_checkMultipleFields() {
    User user = userService.create("alice@example.com", "Alice");

    // Hard assertion: dừng lại ở assertion đầu tiên thất bại
    // → Khó biết tất cả field nào sai

    // Soft assertion: chạy tất cả, báo cáo tất cả lỗi cùng lúc
    SoftAssertions softly = new SoftAssertions();
    softly.assertThat(user.getId()).isNotNull();
    softly.assertThat(user.getEmail()).isEqualTo("alice@example.com");
    softly.assertThat(user.getName()).isEqualTo("Alice");
    softly.assertThat(user.getCreatedAt()).isNotNull();
    softly.assertThat(user.isActive()).isTrue();
    softly.assertAll();  // ← Throw nếu có bất kỳ assertion nào fail
}
```

---

## 6. @ParameterizedTest — Test Nhiều Trường Hợp

### @ValueSource

```java
@ParameterizedTest
@ValueSource(strings = {"", "  ", "\t", "\n"})
@DisplayName("Email rỗng hoặc chỉ có whitespace phải không hợp lệ")
void validate_blankEmail_shouldReturnFalse(String blankEmail) {
    assertThat(emailValidator.isValid(blankEmail)).isFalse();
}

@ParameterizedTest
@ValueSource(ints = {-1, 0, Integer.MIN_VALUE})
void validate_nonPositiveQuantity_shouldThrow(int quantity) {
    assertThrows(IllegalArgumentException.class,
        () -> orderService.validateQuantity(quantity));
}
```

### @CsvSource và @CsvFileSource

```java
@ParameterizedTest
@CsvSource({
    "1, 2, 3",       // input1, input2, expectedSum
    "10, 20, 30",
    "-5, 5, 0",
    "100, -50, 50"
})
void add_variousNumbers_returnsCorrectSum(int a, int b, int expected) {
    assertThat(calculator.add(a, b)).isEqualTo(expected);
}

// Đọc từ file CSV trong src/test/resources
@ParameterizedTest
@CsvFileSource(resources = "/test-data/price-calculations.csv", numLinesToSkip = 1)
void calculatePrice_fromCsvData(String productId, int qty, long expectedPrice) {
    assertThat(priceService.calculate(productId, qty)).isEqualTo(expectedPrice);
}
```

### @MethodSource — Dữ Liệu Phức Tạp

```java
@ParameterizedTest
@MethodSource("provideInvalidOrders")
void createOrder_withInvalidInput_shouldThrow(CreateOrderRequest request, String expectedMessage) {
    assertThatThrownBy(() -> orderService.createOrder(request))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining(expectedMessage);
}

// Method cung cấp dữ liệu — phải static
static Stream<Arguments> provideInvalidOrders() {
    return Stream.of(
        Arguments.of(
            new CreateOrderRequest(null, 1, 1000L),
            "productId không được null"
        ),
        Arguments.of(
            new CreateOrderRequest("product-1", 0, 1000L),
            "quantity phải lớn hơn 0"
        ),
        Arguments.of(
            new CreateOrderRequest("product-1", 1, -100L),
            "price không được âm"
        )
    );
}
```

### @EnumSource

```java
@ParameterizedTest
@EnumSource(value = OrderStatus.class, names = {"CANCELLED", "REFUNDED"})
void cancelledOrder_shouldNotBeEditable(OrderStatus status) {
    Order order = Order.builder().status(status).build();
    assertThat(orderService.canEdit(order)).isFalse();
}
```

---

## 7. JUnit 5 Extensions (Phần Mở Rộng)

### @ExtendWith

```java
// MockitoExtension — tích hợp Mockito với JUnit 5
@ExtendWith(MockitoExtension.class)
class ServiceTest { ... }

// SpringExtension — tích hợp Spring Context (dùng trong @SpringBootTest)
@ExtendWith(SpringExtension.class)
class ControllerTest { ... }

// Nhiều extensions cùng lúc
@ExtendWith({MockitoExtension.class, TimingExtension.class})
class MultiExtensionTest { ... }
```

### Custom Extension

```java
// Extension đo thời gian thực thi từng test
public class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback {

    private static final String START_TIME = "start time";

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        getStore(context).put(START_TIME, System.currentTimeMillis());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        long startTime = getStore(context).remove(START_TIME, long.class);
        long duration = System.currentTimeMillis() - startTime;
        System.out.printf("Test [%s] took %d ms%n",
            context.getDisplayName(), duration);
    }

    private ExtensionContext.Store getStore(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.create(getClass(), context.getRequiredTestMethod()));
    }
}
```

---

## 8. Test Naming Conventions (Quy Ước Đặt Tên)

### Pattern: `methodName_condition_expectedBehavior`

```java
// ✅ Tên rõ ràng — biết ngay test kiểm tra gì
@Test
void createUser_withDuplicateEmail_throwsDuplicateEmailException() { }

@Test
void findById_withNonExistentId_returnsEmptyOptional() { }

@Test
void calculateDiscount_forPremiumUser_applies20Percent() { }

// ❌ Tên mơ hồ — không biết test kiểm tra gì
@Test
void test1() { }

@Test
void testCreate() { }

@Test
void shouldWork() { }
```

---

## 9. Testing Service Layer — Ví Dụ Thực Tế Đầy Đủ

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private ProductRepository productRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private OrderService orderService;

    private Product testProduct;
    private User testUser;

    @BeforeEach
    void setUp() {
        testProduct = Product.builder()
            .id("p-001")
            .name("Laptop")
            .price(20_000_000L)
            .stockQuantity(10)
            .build();

        testUser = User.builder()
            .id(UUID.randomUUID())
            .email("alice@example.com")
            .build();
    }

    // ============================================================
    // Happy Path Tests (Kiểm Thử Đường Hạnh Phúc)
    // ============================================================

    @Test
    void createOrder_withValidRequest_savesOrderAndSendsEmail() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest("p-001", 2);
        when(productRepository.findById("p-001")).thenReturn(Optional.of(testProduct));
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId(UUID.randomUUID());
            return order;
        });

        // Act
        OrderResponse response = orderService.createOrder(testUser.getId(), request);

        // Assert
        assertThat(response).isNotNull();
        assertThat(response.getTotalPrice()).isEqualTo(40_000_000L); // 2 * 20_000_000
        assertThat(response.getStatus()).isEqualTo(OrderStatus.PENDING);

        // Xác minh email được gửi
        verify(emailService).sendOrderConfirmation(eq(testUser.getId()), any(Order.class));
    }

    // ============================================================
    // Error Path Tests (Kiểm Thử Đường Lỗi)
    // ============================================================

    @Test
    void createOrder_withOutOfStockProduct_throwsInsufficientStockException() {
        // Arrange
        testProduct.setStockQuantity(1);  // Chỉ còn 1 sản phẩm
        CreateOrderRequest request = new CreateOrderRequest("p-001", 5); // Muốn mua 5
        when(productRepository.findById("p-001")).thenReturn(Optional.of(testProduct));

        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(testUser.getId(), request))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessageContaining("Không đủ hàng");

        // Xác minh không lưu order và không gửi email
        verify(orderRepository, never()).save(any());
        verify(emailService, never()).sendOrderConfirmation(any(), any());
    }

    @Test
    void createOrder_withNonExistentProduct_throwsProductNotFoundException() {
        when(productRepository.findById("non-existent")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> orderService.createOrder(
            testUser.getId(),
            new CreateOrderRequest("non-existent", 1)
        ))
        .isInstanceOf(ProductNotFoundException.class);
    }

    // ============================================================
    // Edge Cases (Trường Hợp Biên)
    // ============================================================

    @Test
    void createOrder_withExactStockQuantity_succeeds() {
        testProduct.setStockQuantity(3);
        CreateOrderRequest request = new CreateOrderRequest("p-001", 3); // Mua vừa hết hàng
        when(productRepository.findById("p-001")).thenReturn(Optional.of(testProduct));
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        assertThatNoException().isThrownBy(
            () -> orderService.createOrder(testUser.getId(), request)
        );
    }
}
```

---

## 10. Kiểm Thử Với @Nested — Tổ Chức Test Theo Nhóm

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @InjectMocks private UserService userService;
    @Mock private UserRepository userRepository;
    @Mock private PasswordEncoder passwordEncoder;

    @Nested
    @DisplayName("createUser() — tạo người dùng mới")
    class CreateUserTests {

        @Test
        void withValidData_shouldReturnCreatedUser() { /* ... */ }

        @Test
        void withDuplicateEmail_shouldThrowException() { /* ... */ }

        @Test
        void withWeakPassword_shouldThrowException() { /* ... */ }
    }

    @Nested
    @DisplayName("findByEmail() — tìm theo email")
    class FindByEmailTests {

        @Test
        void withExistingEmail_shouldReturnUser() { /* ... */ }

        @Test
        void withNonExistingEmail_shouldReturnEmpty() { /* ... */ }
    }

    @Nested
    @DisplayName("deactivate() — vô hiệu hóa tài khoản")
    class DeactivateTests {

        @Test
        void withActiveUser_shouldSetInactive() { /* ... */ }

        @Test
        void withAlreadyInactiveUser_shouldThrow() { /* ... */ }
    }
}
```

---

## 11. Anti-Patterns (Mẫu Sai Cần Tránh)

### ❌ Test Quá Nhiều Thứ Cùng Lúc

```java
// ❌ Sai — một test làm quá nhiều
@Test
void testEverything() {
    // Tạo user
    User user = userService.create("alice@example.com", "Alice");
    assertThat(user).isNotNull();

    // Cập nhật user
    user.setName("Alice Smith");
    userService.update(user);

    // Xóa user
    userService.delete(user.getId());
    assertThat(userService.findById(user.getId())).isEmpty();
}

// ✅ Đúng — mỗi test một hành vi
@Test
void create_validUser_returnsPersistedUser() { /* chỉ test create */ }

@Test
void update_existingUser_persistsChanges() { /* chỉ test update */ }

@Test
void delete_existingUser_removesFromRepository() { /* chỉ test delete */ }
```

### ❌ Magic Numbers Không Rõ Nghĩa

```java
// ❌ Sai — 100000 là gì? Tại sao 0.1?
assertThat(result).isEqualTo(100000);
assertThat(discount).isCloseTo(0.1, within(0.001));

// ✅ Đúng — đặt tên rõ ràng
long PREMIUM_THRESHOLD_PRICE = 100_000L;
double PREMIUM_DISCOUNT_RATE = 0.10;
assertThat(result).isEqualTo(PREMIUM_THRESHOLD_PRICE);
assertThat(discount).isCloseTo(PREMIUM_DISCOUNT_RATE, within(0.001));
```

### ❌ Không Test Được Vì Field Injection

```java
// ❌ Sai — không thể inject mock vào @Autowired field trực tiếp
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // Field injection
}

// ✅ Đúng — Constructor injection dễ test
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
// → Trong test: new OrderService(mockRepository)
```

---

## 12. Checklist Unit Testing

- [ ] Mỗi test method kiểm tra đúng một hành vi (Single Responsibility)
- [ ] Tên test mô tả rõ điều kiện và kết quả mong đợi
- [ ] Test độc lập — không phụ thuộc vào thứ tự chạy hoặc state từ test khác
- [ ] Có test cho cả happy path và error path
- [ ] Có test cho edge cases (biên: null, rỗng, giá trị cực đại/cực tiểu)
- [ ] Không có logic phức tạp trong test code
- [ ] Sử dụng AssertJ thay vì `assertTrue`/`assertEquals` thuần túy
- [ ] Không hardcode giá trị kỳ diệu — đặt tên cho constants

---

## 📚 Tài Liệu Tham Khảo

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [AssertJ Documentation](https://assertj.github.io/doc/)
- [Practical Unit Testing with JUnit and Mockito](https://practicalunittesting.com/)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] Sự khác biệt giữa `@BeforeAll` và `@BeforeEach`?
- [ ] Tại sao nên dùng Constructor Injection thay vì Field Injection cho Unit Test?
- [ ] `@ParameterizedTest` có ưu điểm gì so với viết nhiều test method riêng lẻ?
- [ ] Soft Assertions là gì? Khi nào dùng?
- [ ] Viết test cho private method — nên hay không? Cách tiếp cận đúng là gì?
- [ ] Sự khác biệt giữa AssertJ và JUnit assertions thuần túy?
