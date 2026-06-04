# Method Security — Bảo Mật Cấp Phương Thức

> **Method Security** (Bảo Mật Cấp Phương Thức) cho phép kiểm soát quyền truy cập ở tầng service/method,
> thay vì chỉ ở tầng URL. Điều này giúp phân quyền chi tiết hơn, sát với logic nghiệp vụ hơn.

---

## 1. Tại Sao Cần Method Security?

URL-level security (bảo mật cấp URL) có giới hạn:
- Không biết context của dữ liệu (record này có thuộc về user không?)
- Cần pattern phức tạp để mô tả business rules
- Logic phân quyền bị phân tán ở nhiều nơi

Method Security giải quyết những vấn đề này:

```java
// URL security — chỉ kiểm tra role
.requestMatchers("/api/orders/**").hasRole("USER")

// Method security — kiểm tra cả logic nghiệp vụ
@PreAuthorize("hasRole('USER') and #order.userId == authentication.principal.id")
public Order updateOrder(Order order) { ... }
```

---

## 2. Bật Method Security

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(            // Bật method security (Spring Security 6+)
    prePostEnabled = true,        // Bật @PreAuthorize, @PostAuthorize
    securedEnabled = true,        // Bật @Secured
    jsr250Enabled = true          // Bật @RolesAllowed (JSR-250)
)
public class SecurityConfig {
    // ...
}
```

> **Lưu ý:** Trong Spring Security 6+, `@EnableMethodSecurity` thay thế `@EnableGlobalMethodSecurity` (deprecated).

---

## 3. @PreAuthorize — Kiểm Tra Trước Khi Thực Thi

`@PreAuthorize` kiểm tra điều kiện **trước khi** phương thức được gọi. Đây là annotation phổ biến nhất.

### 3.1 Kiểm Tra Role Cơ Bản

```java
@Service
public class ProductService {

    // Chỉ ADMIN mới có thể xóa sản phẩm
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // ADMIN hoặc MANAGER mới xem được báo cáo
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<SalesReport> getSalesReports() {
        return reportRepository.findAll();
    }

    // Chỉ cần đã đăng nhập
    @PreAuthorize("isAuthenticated()")
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }
}
```

### 3.2 Truy Cập Tham Số Phương Thức

SpEL (Spring Expression Language — Ngôn Ngữ Biểu Thức Spring) cho phép truy cập tham số bằng `#paramName`:

```java
@Service
public class OrderService {

    // User chỉ được cập nhật order của chính mình, trừ ADMIN
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Order> getOrdersByUser(Long userId) {
        return orderRepository.findByUserId(userId);
    }

    // Kiểm tra field của object được truyền vào
    @PreAuthorize("hasRole('ADMIN') or #order.createdBy == authentication.name")
    public Order updateOrder(@P("order") Order order) {
        return orderRepository.save(order);
    }

    // Kết hợp nhiều điều kiện
    @PreAuthorize("hasRole('USER') and #amount <= 10000 or hasRole('ADMIN')")
    public Payment createPayment(Long userId, BigDecimal amount) {
        return paymentService.process(userId, amount);
    }
}
```

### 3.3 Truy Cập SecurityContext Trong SpEL

```java
@PreAuthorize("authentication.name == #username")
public UserProfile getProfile(String username) { ... }

// authentication.principal — trả về UserDetails object
@PreAuthorize("authentication.principal.username == #username")
public void updateProfile(String username, ProfileRequest request) { ... }

// Kiểm tra permission (authority không có ROLE_ prefix)
@PreAuthorize("hasAuthority('ORDER_WRITE')")
public Order createOrder(OrderRequest request) { ... }
```

---

## 4. @PostAuthorize — Kiểm Tra Sau Khi Thực Thi

`@PostAuthorize` kiểm tra điều kiện **sau khi** phương thức chạy xong, dựa trên **kết quả trả về** (`returnObject`).

```java
@Service
public class DocumentService {

    // Phương thức chạy xong, sau đó kiểm tra xem result có thuộc về user không
    @PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    public Document getDocument(Long id) {
        return documentRepository.findById(id)
            .orElseThrow(() -> new DocumentNotFoundException(id));
    }

    // Kiểm tra list trả về
    @PostAuthorize("returnObject.createdBy == authentication.name")
    public Report generateReport(ReportRequest request) {
        return reportGenerator.generate(request);
    }
}
```

> **Cảnh báo:** `@PostAuthorize` vẫn **thực thi phương thức** rồi mới kiểm tra quyền. Tránh dùng cho các thao tác có side effect (ghi DB, gọi external API) — thay vào đó dùng `@PreAuthorize`.

---

## 5. @PreFilter & @PostFilter — Lọc Collection

`@PreFilter` lọc collection **đầu vào** trước khi xử lý. `@PostFilter` lọc collection **đầu ra** sau khi xử lý.

```java
@Service
public class FileService {

    // Chỉ xử lý các file thuộc về user hiện tại
    @PreFilter("filterObject.ownerId == authentication.principal.id")
    public List<File> processFiles(List<File> files) {
        // files đã được lọc, chỉ còn file của user hiện tại
        return files.stream().map(this::process).toList();
    }

    // Lọc kết quả — chỉ trả về record thuộc về user
    @PostFilter("filterObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    public List<Order> getAllOrders() {
        return orderRepository.findAll(); // Trả về tất cả nhưng sẽ bị lọc
    }
}
```

> **Lưu ý hiệu năng:** `@PostFilter` fetch tất cả dữ liệu rồi mới lọc. Với dataset lớn, nên filter trực tiếp trong query thay vì dùng `@PostFilter`.

---

## 6. @Secured — Kiểm Tra Role Đơn Giản

`@Secured` là annotation đơn giản hơn, chỉ hỗ trợ kiểm tra role, không hỗ trợ SpEL:

```java
@Service
public class AdminService {

    // Chỉ một role
    @Secured("ROLE_ADMIN")
    public void performAdminAction() { ... }

    // Nhiều role — có ít nhất một là đủ
    @Secured({"ROLE_ADMIN", "ROLE_SUPER_ADMIN"})
    public void sensitiveOperation() { ... }
}
```

**So sánh `@Secured` vs `@PreAuthorize`:**

| Tính Năng | @Secured | @PreAuthorize |
|-----------|---------|---------------|
| SpEL expression | ❌ | ✅ |
| Truy cập tham số | ❌ | ✅ |
| Truy cập returnObject | ❌ | ❌ (dùng @PostAuthorize) |
| Cú pháp | Đơn giản | Linh hoạt hơn |
| Khuyến nghị | Ít dùng | Ưu tiên dùng |

---

## 7. @RolesAllowed — JSR-250 Standard

`@RolesAllowed` là annotation chuẩn JSR-250, tương tự `@Secured` nhưng không cần `ROLE_` prefix:

```java
import jakarta.annotation.security.RolesAllowed;

@Service
public class ReportService {

    @RolesAllowed("ADMIN")           // Không cần "ROLE_" prefix
    public List<Report> getReports() { ... }

    @RolesAllowed({"ADMIN", "MANAGER"})
    public Report generateReport() { ... }
}
```

---

## 8. Custom Permission Evaluator (Đánh Giá Quyền Tùy Chỉnh)

Dùng khi cần logic phân quyền phức tạp, không thể viết gọn trong SpEL:

### 8.1 Implement PermissionEvaluator

```java
@Component("permissionEvaluator")
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final PermissionRepository permissionRepository;

    // hasPermission(targetObject, permission)
    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject,
                                 Object permission) {
        if (targetDomainObject instanceof Document doc) {
            String username = auth.getName();
            String permissionName = (String) permission;
            return permissionRepository.userHasPermission(username, doc.getId(), permissionName);
        }
        return false;
    }

    // hasPermission(targetId, targetType, permission)
    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId,
                                 String targetType, Object permission) {
        String username = auth.getName();
        return permissionRepository.userHasPermission(
            username, (Long) targetId, targetType, (String) permission);
    }
}
```

### 8.2 Đăng Ký PermissionEvaluator

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {

    @Bean
    public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
            CustomPermissionEvaluator permissionEvaluator) {
        DefaultMethodSecurityExpressionHandler handler =
            new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);
        return handler;
    }
}
```

### 8.3 Sử Dụng hasPermission() Trong SpEL

```java
@Service
public class DocumentService {

    // Truyền object vào
    @PreAuthorize("hasPermission(#document, 'EDIT')")
    public Document editDocument(Document document) { ... }

    // Truyền ID và type
    @PreAuthorize("hasPermission(#documentId, 'Document', 'VIEW')")
    public Document getDocument(Long documentId) { ... }

    // Kết hợp
    @PreAuthorize("hasRole('ADMIN') or hasPermission(#documentId, 'Document', 'DELETE')")
    public void deleteDocument(Long documentId) { ... }
}
```

---

## 9. SpEL Expressions Hay Dùng

```java
// Kiểm tra role
@PreAuthorize("hasRole('ADMIN')")
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")

// Kiểm tra authority (không có ROLE_ prefix)
@PreAuthorize("hasAuthority('WRITE')")
@PreAuthorize("hasAnyAuthority('WRITE', 'ADMIN')")

// Trạng thái xác thực
@PreAuthorize("isAuthenticated()")
@PreAuthorize("isAnonymous()")
@PreAuthorize("isFullyAuthenticated()")  // Loại trừ "remember me"

// Truy cập authentication object
@PreAuthorize("authentication.name == #username")
@PreAuthorize("authentication.principal.id == #userId")

// Kết hợp logic
@PreAuthorize("hasRole('ADMIN') or (#userId == authentication.principal.id and hasRole('USER'))")

// Kiểm tra null
@PreAuthorize("#request.userId != null and hasRole('USER')")
```

---

## 10. Method Security Trong Controller vs Service

**Khuyến nghị:** Đặt `@PreAuthorize` ở **Service layer** (tầng nghiệp vụ), không phải Controller:

```java
// ❌ Không nên — ở Controller
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    @PreAuthorize("hasRole('USER')")
    public Order getOrder(@PathVariable Long id) {
        return orderService.getOrder(id);
    }
}

// ✅ Nên — ở Service, bảo vệ cả khi được gọi từ nơi khác
@Service
public class OrderService {
    @PreAuthorize("hasRole('USER') and #userId == authentication.principal.id")
    public Order getOrder(Long id, Long userId) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

---

## 11. Testing Method Security

```java
@SpringBootTest
class OrderServiceSecurityTest {

    @Autowired
    private OrderService orderService;

    @Test
    @WithMockUser(roles = "USER")
    void shouldAllowUserToGetOwnOrder() {
        // Given
        Order order = createOrderForUser("testuser");

        // Then — không throw exception
        assertDoesNotThrow(() -> orderService.getOrder(order.getId()));
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldDenyUserAccessingOtherUserOrder() {
        // Given
        Order order = createOrderForUser("anotheruser");

        // Then — throw AccessDeniedException
        assertThrows(AccessDeniedException.class,
            () -> orderService.getOrder(order.getId()));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldAllowAdminToDeleteAnyOrder() {
        Order order = createOrderForUser("anyuser");
        assertDoesNotThrow(() -> orderService.deleteOrder(order.getId()));
    }

    @Test
    void shouldDenyAnonymousUserAccess() {
        assertThrows(AuthenticationCredentialsNotFoundException.class,
            () -> orderService.getAllOrders());
    }
}
```

---

## 12. Checklist Method Security

- [ ] Bật `@EnableMethodSecurity` trong `@Configuration` class
- [ ] Hiểu khi nào dùng `@PreAuthorize` vs `@PostAuthorize`
- [ ] Sử dụng được SpEL để truy cập tham số và authentication
- [ ] Implement `PermissionEvaluator` khi cần logic phức tạp
- [ ] Đặt security annotation ở service layer, không phải controller
- [ ] Viết security tests với `@WithMockUser`

---

## 🔗 Bài Tiếp Theo

→ [5-cors-csrf.md](5-cors-csrf.md) — CORS & CSRF — Bảo Vệ API Khỏi Tấn Công Cross-Origin
