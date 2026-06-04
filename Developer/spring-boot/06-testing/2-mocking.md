# Mocking với Mockito

> Mockito là framework mocking (giả lập) phổ biến nhất trong hệ sinh thái Java —
> cho phép tạo ra các đối tượng giả (mock objects) thay thế cho các phụ thuộc thực,
> giúp kiểm thử từng unit một cách cô lập hoàn toàn.

---

## 1. Tại Sao Cần Mocking?

```
Tình huống: Kiểm thử OrderService.createOrder()

Không Mock:                          Có Mock:
┌─────────────────────────┐          ┌─────────────────────────┐
│ OrderService             │          │ OrderService             │
│   ↓ gọi thực            │          │   ↓ gọi mock            │
│ OrderRepository          │          │ MockOrderRepository      │
│   ↓ kết nối thực        │          │   └─► trả về dữ liệu    │
│ PostgreSQL Database      │          │       giả lập ngay lập tức│
│   → Test chậm, phụ thuộc│          │                          │
│     vào DB, không ổn định│         │ → Test nhanh, cô lập,    │
└─────────────────────────┘          │   kiểm soát hoàn toàn   │
                                      └─────────────────────────┘
```

**Lợi ích của Mocking:**
- Cô lập unit cần test — không phụ thuộc DB, network, file system
- Test chạy rất nhanh (milli giây)
- Kiểm soát hoàn toàn behavior của phụ thuộc
- Test các edge cases khó tái tạo (network timeout, DB error)

---

## 2. Cài Đặt

Mockito được bao gồm tự động trong `spring-boot-starter-test`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- Bao gồm mockito-core và mockito-junit-jupiter -->
</dependency>
```

---

## 3. Tạo Mock Objects

### Cách 1: Annotations (Khuyến Nghị)

```java
@ExtendWith(MockitoExtension.class)  // ← Bắt buộc khi dùng Mockito annotations
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;   // ← Tạo mock tự động

    @Mock
    private EmailService emailService;

    @Spy
    private PriceCalculator priceCalculator = new PriceCalculator();  // ← Wrap object thực

    @InjectMocks
    private OrderService orderService;         // ← Inject tất cả @Mock và @Spy vào đây

    @Captor
    private ArgumentCaptor<Order> orderCaptor; // ← Captor để bắt đối số
}
```

### Cách 2: Programmatic (Tạo Bằng Code)

```java
class OrderServiceTest {

    private OrderRepository orderRepository = mock(OrderRepository.class);
    private EmailService emailService = mock(EmailService.class);
    private OrderService orderService = new OrderService(orderRepository, emailService);

    // Hoặc dùng Mockito.mock() với generics
    @SuppressWarnings("unchecked")
    private List<String> mockList = mock(List.class);
}
```

---

## 4. Stubbing — Định Nghĩa Behavior Cho Mock

### when().thenReturn() — Trả Về Giá Trị

```java
@Test
void stubbing_basicReturn() {
    // Stub: khi gọi findById(1L) → trả về Optional của user
    User alice = new User(1L, "alice@example.com", "Alice");
    when(userRepository.findById(1L)).thenReturn(Optional.of(alice));

    // Gọi nhiều lần khác nhau
    when(userRepository.findById(999L)).thenReturn(Optional.empty());

    // Verify
    assertThat(userService.findById(1L)).isPresent();
    assertThat(userService.findById(999L)).isEmpty();
}
```

### Stubbing Với Argument Matchers (Bộ Khớp Đối Số)

```java
@Test
void stubbing_withMatchers() {
    // any() — khớp mọi đối số không null
    when(orderRepository.save(any(Order.class))).thenReturn(savedOrder);

    // any() — kể cả null
    when(emailService.send(any())).thenReturn(true);

    // anyString(), anyLong(), anyInt()
    when(userRepository.findByEmail(anyString())).thenReturn(Optional.empty());

    // eq() — khớp chính xác giá trị
    when(productRepository.findById(eq("product-001"))).thenReturn(Optional.of(product));

    // argThat() — kiểm tra theo điều kiện tùy chỉnh
    when(orderRepository.findAll(argThat(spec -> spec != null))).thenReturn(List.of());

    // QUAN TRỌNG: Khi mix matchers và giá trị cụ thể, PHẢI wrap tất cả trong matcher
    // ❌ Sai: when(service.method(eq("a"), "b"))
    // ✅ Đúng: when(service.method(eq("a"), eq("b")))
}
```

### thenAnswer() — Trả Về Giá Trị Động

```java
@Test
void stubbing_withAnswer() {
    // Trả về đối số đầu tiên được truyền vào (hữu ích khi test save())
    when(orderRepository.save(any(Order.class)))
        .thenAnswer(invocation -> {
            Order order = invocation.getArgument(0);
            order.setId(UUID.randomUUID());  // Giả lập DB auto-generate ID
            order.setCreatedAt(LocalDateTime.now());
            return order;
        });

    // Lấy đối số theo index
    when(emailService.sendWithTemplate(anyString(), anyMap()))
        .thenAnswer(inv -> {
            String template = inv.getArgument(0);
            Map<String, Object> vars = inv.getArgument(1);
            System.out.printf("Sending template: %s with vars: %s%n", template, vars);
            return true;
        });
}
```

### thenThrow() — Ném Ngoại Lệ

```java
@Test
void stubbing_throwException() {
    // Giả lập DB connection error
    when(orderRepository.findById(anyLong()))
        .thenThrow(new DataAccessException("Không thể kết nối database") {});

    // Giả lập lần đầu thành công, lần sau thất bại
    when(externalService.call())
        .thenReturn("success")
        .thenThrow(new RuntimeException("Service unavailable"));
}
```

### Stubbing Với Multiple Returns (Trả Về Nhiều Giá Trị Khác Nhau)

```java
@Test
void stubbing_multipleReturns() {
    // Lần gọi 1 → "first", lần gọi 2 → "second", lần gọi 3+ → "default"
    when(randomService.generate())
        .thenReturn("first")
        .thenReturn("second")
        .thenReturn("default");

    assertThat(randomService.generate()).isEqualTo("first");
    assertThat(randomService.generate()).isEqualTo("second");
    assertThat(randomService.generate()).isEqualTo("default");
    assertThat(randomService.generate()).isEqualTo("default"); // Lần 4 vẫn "default"
}
```

---

## 5. verify() — Xác Minh Tương Tác

### Xác Minh Số Lần Gọi

```java
@Test
void verify_interactionCount() {
    orderService.createOrder(request);

    // Xác minh được gọi đúng 1 lần (mặc định)
    verify(orderRepository).save(any(Order.class));
    verify(emailService, times(1)).sendConfirmation(any());

    // Xác minh không bao giờ được gọi
    verify(fraudDetectionService, never()).check(any());

    // Xác minh được gọi ít nhất / nhiều nhất N lần
    verify(auditLog, atLeastOnce()).log(anyString());
    verify(retryService, atMost(3)).retry(any());
    verify(cacheService, atLeast(2)).evict(anyString());
}
```

### Xác Minh Thứ Tự Gọi

```java
@Test
void verify_callOrder() {
    orderService.processOrder(orderId);

    // Xác minh thứ tự: validate trước → save sau → notify cuối
    InOrder inOrder = inOrder(validatorService, orderRepository, notificationService);
    inOrder.verify(validatorService).validate(any());
    inOrder.verify(orderRepository).save(any());
    inOrder.verify(notificationService).notify(anyLong());
}
```

### verifyNoMoreInteractions() và verifyNoInteractions()

```java
@Test
void verify_noUnexpectedInteractions() {
    userService.getPublicProfile(userId);

    verify(userRepository).findById(userId);

    // Đảm bảo không có tương tác nào khác với userRepository
    verifyNoMoreInteractions(userRepository);

    // Đảm bảo emailService không bị gọi lần nào
    verifyNoInteractions(emailService);
}
```

---

## 6. ArgumentCaptor — Bắt Và Kiểm Tra Đối Số

ArgumentCaptor (Bộ Chụp Đối Số) cho phép bắt đối số được truyền vào mock để kiểm tra chi tiết.

### Dùng Khi Nào?

```
Khi cần xác minh CHÍNH XÁC đối tượng được truyền vào mock,
nhưng đối tượng đó được tạo ra bên trong method đang test
→ không thể dùng eq() matcher.
```

### Ví Dụ Cơ Bản

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Captor
    private ArgumentCaptor<Order> orderCaptor;

    @Captor
    private ArgumentCaptor<EmailRequest> emailCaptor;

    @Test
    void createOrder_shouldSaveOrderWithCorrectData() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest("product-1", 3, 50_000L);
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        orderService.createOrder(userId, request);

        // Bắt đối số được truyền vào orderRepository.save()
        verify(orderRepository).save(orderCaptor.capture());
        Order capturedOrder = orderCaptor.getValue();

        // Kiểm tra chi tiết đối tượng
        assertThat(capturedOrder.getProductId()).isEqualTo("product-1");
        assertThat(capturedOrder.getQuantity()).isEqualTo(3);
        assertThat(capturedOrder.getTotalPrice()).isEqualTo(150_000L); // 3 * 50_000
        assertThat(capturedOrder.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(capturedOrder.getUserId()).isEqualTo(userId);
        assertThat(capturedOrder.getCreatedAt()).isNotNull();
    }
}
```

### ArgumentCaptor Với Multiple Captures

```java
@Test
void processMultipleOrders_shouldSaveEachCorrectly() {
    List<CreateOrderRequest> requests = List.of(
        new CreateOrderRequest("p-001", 1, 100L),
        new CreateOrderRequest("p-002", 2, 200L),
        new CreateOrderRequest("p-003", 3, 300L)
    );

    orderService.processAll(requests);

    // Bắt tất cả lần gọi save()
    verify(orderRepository, times(3)).save(orderCaptor.capture());
    List<Order> allCapturedOrders = orderCaptor.getAllValues();

    assertThat(allCapturedOrders).hasSize(3);
    assertThat(allCapturedOrders)
        .extracting(Order::getProductId)
        .containsExactly("p-001", "p-002", "p-003");

    assertThat(allCapturedOrders)
        .extracting(Order::getTotalPrice)
        .containsExactly(100L, 400L, 900L); // quantity * price
}
```

---

## 7. @Spy — Partial Mock (Giả Lập Một Phần)

`@Spy` bọc đối tượng thực — chỉ override một số method, các method còn lại gọi thực.

```java
@ExtendWith(MockitoExtension.class)
class ReportServiceTest {

    @Spy
    private PriceCalculator priceCalculator = new PriceCalculator();  // ← Object thực

    @InjectMocks
    private ReportService reportService;

    @Test
    void generateReport_usesRealPriceCalculation() {
        // Hầu hết method chạy thực
        // Chỉ override một method cụ thể
        doReturn(0.15).when(priceCalculator).getVatRate();  // ← Override

        Report report = reportService.generateMonthlyReport();

        // getVatRate() trả về 0.15 (mock)
        // Các method khác của PriceCalculator chạy thực
        assertThat(report.getVatRate()).isEqualTo(0.15);
    }
}
```

### Lưu Ý Quan Trọng Với @Spy

```java
// ❌ Sai với @Spy — tránh when().then() vì gọi method thực trước
when(spy.realMethod()).thenReturn("value");  // Gọi realMethod() trước rồi stub!

// ✅ Đúng với @Spy — dùng doReturn().when()
doReturn("value").when(spy).realMethod();   // Không gọi realMethod()
```

---

## 8. doReturn, doThrow, doAnswer, doNothing

Dùng khi stubbing `void` methods hoặc khi dùng `@Spy`:

```java
@Test
void stubbing_voidMethods() {
    // doNothing() — phương thức void không làm gì
    doNothing().when(emailService).sendNotification(anyString(), anyString());

    // doThrow() — phương thức void ném exception
    doThrow(new MailException("SMTP server down") {})
        .when(emailService).sendNotification(eq("error@example.com"), anyString());

    // doAnswer() — phương thức void với side effect
    doAnswer(invocation -> {
        String recipient = invocation.getArgument(0);
        System.out.println("Mock sending to: " + recipient);
        return null;  // void method phải return null
    }).when(emailService).sendNotification(anyString(), anyString());

    // doCallRealMethod() — gọi method thực (chỉ dùng với @Spy)
    doCallRealMethod().when(spyService).calculateTotal(anyList());
}
```

---

## 9. Mock Static Methods và Constructors

Với Mockito 3.4+ (mockito-inline), có thể mock static methods:

```java
@Test
void mockStaticMethod() {
    try (MockedStatic<LocalDateTime> mockedDateTime = mockStatic(LocalDateTime.class)) {
        LocalDateTime fixedTime = LocalDateTime.of(2026, 6, 1, 12, 0);
        mockedDateTime.when(LocalDateTime::now).thenReturn(fixedTime);

        // Trong khoảng này, LocalDateTime.now() trả về fixedTime
        Order order = orderService.createOrder(request);
        assertThat(order.getCreatedAt()).isEqualTo(fixedTime);
    }
    // Ngoài try-with-resources: LocalDateTime.now() hoạt động bình thường
}
```

### Mock Constructor

```java
@Test
void mockConstructor() {
    try (MockedConstruction<HttpClient> mockedClient = mockConstruction(HttpClient.class,
            (mock, context) -> {
                when(mock.send(any(), any())).thenReturn(mockResponse);
            })) {

        // Mọi new HttpClient() trong code sẽ trả về mock
        String result = apiService.fetchData("https://api.example.com");
        assertThat(result).isNotNull();
    }
}
```

---

## 10. So Sánh @Mock vs @MockBean

| Đặc Điểm | `@Mock` (Mockito) | `@MockBean` (Spring) |
|-----------|-------------------|---------------------|
| **Context** | Không cần Spring Context | Cần Spring Context (`@SpringBootTest`, `@WebMvcTest`) |
| **Tốc độ** | Rất nhanh | Chậm hơn (load Spring context) |
| **Inject** | `@InjectMocks` | Tự động inject vào Spring beans |
| **Dùng khi** | Unit Test thuần túy | Integration Test hoặc Web Layer Test |
| **Reset** | Tự động sau mỗi test | Tự động sau mỗi test |

```java
// Unit Test — dùng @Mock
@ExtendWith(MockitoExtension.class)
class OrderServiceUnitTest {
    @Mock OrderRepository orderRepository;
    @InjectMocks OrderService orderService;
}

// Integration Test — dùng @MockBean
@SpringBootTest
class OrderServiceIntegrationTest {
    @MockBean OrderRepository orderRepository; // ← Thay thế bean trong Spring Context
    @Autowired OrderService orderService;
}
```

---

## 11. Ví Dụ Thực Tế — Test Service Đầy Đủ

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock private PaymentGateway paymentGateway;
    @Mock private OrderRepository orderRepository;
    @Mock private NotificationService notificationService;
    @Mock private AuditLogService auditLogService;

    @InjectMocks private PaymentService paymentService;

    @Captor private ArgumentCaptor<PaymentRequest> paymentRequestCaptor;
    @Captor private ArgumentCaptor<String> auditMessageCaptor;

    @Test
    void processPayment_success_updatesOrderAndNotifies() {
        // Arrange
        UUID orderId = UUID.randomUUID();
        Order order = Order.builder()
            .id(orderId).totalPrice(500_000L).status(OrderStatus.PENDING).build();

        PaymentResult successResult = PaymentResult.success("TXN-12345");

        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(paymentGateway.charge(any(PaymentRequest.class))).thenReturn(successResult);
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        paymentService.processPayment(orderId);

        // Assert — xác minh PaymentRequest được gửi đúng
        verify(paymentGateway).charge(paymentRequestCaptor.capture());
        PaymentRequest capturedRequest = paymentRequestCaptor.getValue();
        assertThat(capturedRequest.getAmount()).isEqualTo(500_000L);
        assertThat(capturedRequest.getCurrency()).isEqualTo("VND");

        // Xác minh order được cập nhật thành PAID
        verify(orderRepository).save(orderCaptor.capture());
        // ... (nếu có @Captor orderCaptor)

        // Xác minh notification được gửi
        verify(notificationService).sendPaymentSuccess(orderId);

        // Xác minh audit log được ghi với thứ tự đúng
        InOrder inOrder = inOrder(paymentGateway, orderRepository, notificationService, auditLogService);
        inOrder.verify(paymentGateway).charge(any());
        inOrder.verify(orderRepository).save(any());
        inOrder.verify(notificationService).sendPaymentSuccess(any());
        inOrder.verify(auditLogService).log(anyString());
    }

    @Test
    void processPayment_gatewayFailure_throwsAndDoesNotUpdateOrder() {
        UUID orderId = UUID.randomUUID();
        Order order = Order.builder().id(orderId).totalPrice(100_000L).build();

        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(paymentGateway.charge(any())).thenThrow(new PaymentException("Thẻ bị từ chối"));

        assertThatThrownBy(() -> paymentService.processPayment(orderId))
            .isInstanceOf(PaymentException.class)
            .hasMessageContaining("Thẻ bị từ chối");

        // Order KHÔNG được cập nhật khi payment thất bại
        verify(orderRepository, never()).save(any());
        verify(notificationService, never()).sendPaymentSuccess(any());

        // Nhưng audit log của lỗi VẪN được ghi
        verify(auditLogService).logError(anyString(), any(PaymentException.class));
    }
}
```

---

## 12. Checklist Mockito

- [ ] Dùng `@ExtendWith(MockitoExtension.class)` thay vì `MockitoAnnotations.openMocks(this)`
- [ ] Dùng `doReturn().when()` với `@Spy`, không dùng `when().thenReturn()`
- [ ] Dùng `ArgumentCaptor` khi cần kiểm tra đối tượng được tạo bên trong method
- [ ] `verify()` sau phần Act, không trộn lẫn với Arrange
- [ ] Không over-mock — chỉ mock những gì cần thiết để cô lập unit
- [ ] Không verify những gì không liên quan đến test case đó
- [ ] Dùng `verifyNoMoreInteractions()` khi cần đảm bảo không có side effects ngoài ý muốn

---

## 📚 Tài Liệu Tham Khảo

- [Mockito Documentation](https://site.mockito.org/)
- [Mockito GitHub](https://github.com/mockito/mockito)
- [Mockito Javadoc](https://javadoc.io/doc/org.mockito/mockito-core/latest/)

---

## 🎯 Câu Hỏi Phỏng Vấn

- [ ] Sự khác biệt giữa `@Mock` và `@Spy`?
- [ ] `@InjectMocks` inject dependency bằng cơ chế nào? (Constructor → Setter → Field)
- [ ] Khi nào dùng `ArgumentCaptor` thay vì `eq()` matcher?
- [ ] Tại sao cần dùng `doReturn().when()` với `@Spy` thay vì `when().thenReturn()`?
- [ ] `verifyNoMoreInteractions()` dùng khi nào? Có nên dùng trong mọi test không?
- [ ] Strict stubbing là gì? Mockito bật strict stubbing từ phiên bản nào?
