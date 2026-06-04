# Layered Architecture — Kiến Trúc Phân Tầng

> Layered Architecture (Kiến Trúc Phân Tầng) hay còn gọi là N-Tier Architecture (Kiến Trúc N Tầng) — mô hình phổ biến nhất trong Spring Boot: Controller → Service → Repository. Hiểu sâu pattern này là nền tảng trước khi học các kiến trúc phức tạp hơn.

---

## 📋 Mục Lục

1. [Tổng Quan Kiến Trúc Phân Tầng](#tổng-quan-kiến-trúc-phân-tầng)
2. [Presentation Layer — Tầng Trình Bày](#presentation-layer--tầng-trình-bày)
3. [Business Layer — Tầng Nghiệp Vụ](#business-layer--tầng-nghiệp-vụ)
4. [Data Access Layer — Tầng Truy Cập Dữ Liệu](#data-access-layer--tầng-truy-cập-dữ-liệu)
5. [DTO Pattern & Data Flow](#dto-pattern--data-flow)
6. [Best Practices & Anti-patterns](#best-practices--anti-patterns)
7. [Khi Nào Outgrow Layered Architecture?](#khi-nào-outgrow-layered-architecture)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Kiến Trúc Phân Tầng

```
┌────────────────────────────────────────────────┐
│         PRESENTATION LAYER (Tầng Trình Bày)    │
│   @RestController — xử lý HTTP request/response│
└───────────────────────┬────────────────────────┘
                        │ gọi Service
┌───────────────────────▼────────────────────────┐
│        BUSINESS LAYER (Tầng Nghiệp Vụ)         │
│   @Service — chứa business logic               │
└───────────────────────┬────────────────────────┘
                        │ gọi Repository
┌───────────────────────▼────────────────────────┐
│     DATA ACCESS LAYER (Tầng Truy Cập Dữ Liệu)  │
│   @Repository — tương tác với database         │
└───────────────────────┬────────────────────────┘
                        │
┌───────────────────────▼────────────────────────┐
│              DATABASE (Cơ Sở Dữ Liệu)          │
│   PostgreSQL / MySQL / MongoDB                  │
└────────────────────────────────────────────────┘

Nguyên Tắc:
- Mỗi tầng chỉ giao tiếp với tầng liền kề
- Tầng trên phụ thuộc tầng dưới (không ngược lại)
- Mỗi tầng có responsibility (trách nhiệm) rõ ràng
```

---

## Presentation Layer — Tầng Trình Bày

### Trách Nhiệm

- Nhận HTTP request và parse thành DTO (Data Transfer Object — Đối Tượng Truyền Dữ Liệu)
- Validate đầu vào với Bean Validation
- Gọi Service layer
- Chuyển đổi kết quả thành HTTP response
- **Không** chứa business logic

### Controller Chuẩn

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Validated
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public ResponseEntity<PagedResponse<ProductSummaryDto>> getAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String category) {

        Pageable pageable = PageRequest.of(page, size);
        Page<ProductSummaryDto> products = productService.findAll(pageable, category);
        return ResponseEntity.ok(PagedResponse.of(products));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDetailDto> getById(@PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductDetailDto create(@Valid @RequestBody CreateProductRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductDetailDto> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(productService.update(id, request));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

### Request DTO với Validation

```java
// CreateProductRequest.java
public record CreateProductRequest(
    @NotBlank(message = "Tên sản phẩm không được trống")
    @Size(max = 255, message = "Tên sản phẩm tối đa 255 ký tự")
    String name,

    @NotNull(message = "Giá không được null")
    @Positive(message = "Giá phải lớn hơn 0")
    BigDecimal price,

    @NotBlank(message = "Danh mục không được trống")
    String category,

    @Min(value = 0, message = "Số lượng không được âm")
    int stock
) {}

// Response DTO
public record ProductDetailDto(
    Long id,
    String name,
    BigDecimal price,
    String category,
    int stock,
    LocalDateTime createdAt
) {
    public static ProductDetailDto fromEntity(Product product) {
        return new ProductDetailDto(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getCategory(),
            product.getStock(),
            product.getCreatedAt()
        );
    }
}
```

---

## Business Layer — Tầng Nghiệp Vụ

### Trách Nhiệm

- Chứa toàn bộ business logic (logic nghiệp vụ)
- Điều phối giữa nhiều repositories
- Xử lý transaction (giao dịch)
- Throw business exceptions
- **Không** biết gì về HTTP, JSON, request/response

### Service Implementation

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional(readOnly = true)
    public Page<ProductSummaryDto> findAll(Pageable pageable, String category) {
        Page<Product> products = (category != null)
            ? productRepository.findByCategory(category, pageable)
            : productRepository.findAll(pageable);
        return products.map(productMapper::toSummaryDto);
    }

    @Transactional(readOnly = true)
    public ProductDetailDto findById(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException("Không tìm thấy sản phẩm với id: " + id));
        return productMapper.toDetailDto(product);
    }

    @Transactional
    public ProductDetailDto create(CreateProductRequest request) {
        // Business rule: kiểm tra danh mục tồn tại
        categoryRepository.findByName(request.category())
            .orElseThrow(() -> new CategoryNotFoundException("Danh mục không tồn tại: " + request.category()));

        // Business rule: kiểm tra tên không trùng
        if (productRepository.existsByName(request.name())) {
            throw new DuplicateProductException("Sản phẩm đã tồn tại: " + request.name());
        }

        Product product = productMapper.toEntity(request);
        product = productRepository.save(product);

        // Publish domain event (Sự Kiện Miền)
        eventPublisher.publishEvent(new ProductCreatedEvent(product.getId(), product.getName()));

        log.info("Tạo sản phẩm thành công: id={}, name={}", product.getId(), product.getName());
        return productMapper.toDetailDto(product);
    }

    @Transactional
    public ProductDetailDto update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException("Không tìm thấy sản phẩm: " + id));

        productMapper.updateEntity(product, request);
        product = productRepository.save(product);
        return productMapper.toDetailDto(product);
    }

    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new ProductNotFoundException("Không tìm thấy sản phẩm: " + id);
        }
        productRepository.deleteById(id);
    }

    // Business logic phức tạp — điều phối nhiều repositories
    @Transactional
    public void reduceStock(Long productId, int quantity) {
        Product product = productRepository.findByIdForUpdate(productId) // Pessimistic Lock
            .orElseThrow(() -> new ProductNotFoundException("Sản phẩm không tồn tại"));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException(
                "Không đủ hàng. Tồn kho: %d, Yêu cầu: %d".formatted(product.getStock(), quantity)
            );
        }

        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }
}
```

### Service Interface vs Concrete Class

```java
// Có nên tạo interface cho Service không?

// Cách 1: Interface + Implementation (Nhiều team áp dụng)
public interface ProductService {
    ProductDetailDto findById(Long id);
    ProductDetailDto create(CreateProductRequest request);
}

@Service
public class ProductServiceImpl implements ProductService {
    // implementation
}

// Cách 2: Chỉ concrete class (Spring khuyến nghị khi không cần đa hình)
@Service
public class ProductService {
    // implementation trực tiếp
}

// Kết Luận:
// - Dùng interface nếu có nhiều implementation (VD: ProductService cho web và ProductService cho batch)
// - Dùng concrete class nếu chỉ có một implementation — tránh boilerplate không cần thiết
// - Spring tạo proxy qua CGLIB bất kể có interface hay không
```

---

## Data Access Layer — Tầng Truy Cập Dữ Liệu

### Trách Nhiệm

- Tương tác với database (CRUD operations)
- Thực hiện queries (truy vấn)
- Không chứa business logic
- Chuyển đổi database result sang Entity

### Repository Patterns

```java
// Standard Spring Data JPA Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Query Method (Phương Thức Truy Vấn) — Spring tự generate
    Optional<Product> findByName(String name);
    boolean existsByName(String name);
    Page<Product> findByCategory(String category, Pageable pageable);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    // JPQL (Java Persistence Query Language)
    @Query("SELECT p FROM Product p WHERE p.stock > 0 AND p.category = :category")
    List<Product> findAvailableByCategory(@Param("category") String category);

    // Native Query — khi cần SQL phức tạp
    @Query(value = """
        SELECT p.*, AVG(r.rating) as avg_rating
        FROM products p
        LEFT JOIN reviews r ON r.product_id = p.id
        WHERE p.category = :category
        GROUP BY p.id
        HAVING COUNT(r.id) >= :minReviews
        ORDER BY avg_rating DESC
        """, nativeQuery = true)
    List<ProductWithRating> findTopRatedByCategory(
        @Param("category") String category,
        @Param("minReviews") int minReviews
    );

    // Pessimistic Lock — tránh race condition
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    // Projection — chỉ lấy các field cần thiết
    @Query("SELECT p.id as id, p.name as name, p.price as price FROM Product p")
    List<ProductSummaryProjection> findAllSummaries();
}

// Custom Repository — khi cần Criteria API hoặc dynamic queries
public interface ProductRepositoryCustom {
    List<Product> search(ProductSearchCriteria criteria);
}

@Repository
public class ProductRepositoryCustomImpl implements ProductRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Product> search(ProductSearchCriteria criteria) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> root = query.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.name() != null) {
            predicates.add(cb.like(cb.lower(root.get("name")),
                "%" + criteria.name().toLowerCase() + "%"));
        }
        if (criteria.minPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), criteria.minPrice()));
        }
        if (criteria.category() != null) {
            predicates.add(cb.equal(root.get("category"), criteria.category()));
        }

        query.where(predicates.toArray(new Predicate[0]));
        return em.createQuery(query).getResultList();
    }
}
```

### Entity Design

```java
@Entity
@Table(name = "products",
    indexes = {
        @Index(name = "idx_product_category", columnList = "category"),
        @Index(name = "idx_product_name", columnList = "name")
    }
)
@EntityListeners(AuditingEntityListener.class)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String name;

    @Column(nullable = false, precision = 15, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private String category;

    @Column(nullable = false)
    private int stock;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Version  // Optimistic Lock (Khóa Lạc Quan) — tránh lost updates
    private Long version;
    // getters/setters
}
```

---

## DTO Pattern & Data Flow

### Sơ Đồ Luồng Dữ Liệu

```
HTTP Request (JSON)
        │
        ▼
CreateProductRequest (Request DTO)
        │ @Valid validation
        ▼
ProductController.create()
        │ gọi service với DTO
        ▼
ProductService.create(CreateProductRequest)
        │ ProductMapper.toEntity()
        ▼
Product (JPA Entity) ──── ProductRepository.save() ───► Database
        │ ProductMapper.toDetailDto()
        ▼
ProductDetailDto (Response DTO)
        │
        ▼
HTTP Response (JSON)
```

### MapStruct Mapper

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface ProductMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    Product toEntity(CreateProductRequest request);

    ProductDetailDto toDetailDto(Product product);
    ProductSummaryDto toSummaryDto(Product product);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    void updateEntity(@MappingTarget Product product, UpdateProductRequest request);
}
```

---

## Best Practices & Anti-patterns

### ✅ Best Practices (Thực Hành Tốt)

```
1. Mỗi tầng chỉ biết tầng ngay dưới nó (không skip tầng)
2. Controller chỉ xử lý HTTP, không có business logic
3. Service không import HttpServletRequest hay annotation web
4. Entity không expose trực tiếp qua API (dùng DTO)
5. Repository không có business logic
6. Dùng @Transactional ở Service layer, không ở Controller
7. readOnly = true cho các read queries
```

### ❌ Anti-patterns (Mẫu Chống)

```java
// ❌ Anti-pattern 1: Business logic trong Controller
@PostMapping("/checkout")
public ResponseEntity<?> checkout(@RequestBody CheckoutRequest request) {
    // Logic này phải ở Service
    if (request.getItems().isEmpty()) throw new BadRequestException("...");
    BigDecimal total = request.getItems().stream()
        .mapToDouble(i -> i.getPrice() * i.getQty())
        .sum();  // ← Tính tiền không phải việc của Controller
    // ...
}

// ❌ Anti-pattern 2: Repository logic trong Service
@Service
public class OrderService {
    @PersistenceContext
    private EntityManager em;  // ← Service trực tiếp dùng EntityManager

    public List<Order> findPending() {
        return em.createQuery("FROM Order WHERE status = 'PENDING'", Order.class)
            .getResultList();  // ← Đây là việc của Repository
    }
}

// ❌ Anti-pattern 3: Trả Entity trực tiếp từ Controller
@GetMapping("/{id}")
public Product getById(@PathVariable Long id) {  // ← Nên trả DTO, không phải Entity
    return productRepository.findById(id).orElseThrow();
    // Lộ schema database, có thể gây vòng lặp JSON, LazyInitializationException
}

// ❌ Anti-pattern 4: Anemic Domain Model (Mô Hình Miền Thiếu Máu)
// Entity chỉ là data holder, mọi logic ở Service
// → Nên đặt business logic vào Entity khi phù hợp
```

### Anemic vs Rich Domain Model

```java
// Anemic Domain Model (Thiếu Máu) — Anti-pattern
@Entity
public class Order {
    private OrderStatus status;
    // chỉ có getter/setter, không có logic
}

@Service
public class OrderService {
    public void cancelOrder(Order order) {
        if (order.getStatus() != OrderStatus.PENDING) { // logic ở Service
            throw new IllegalStateException("...");
        }
        order.setStatus(OrderStatus.CANCELLED); // Service "điều khiển" entity
    }
}

// Rich Domain Model (Phong Phú) — Khuyến Nghị
@Entity
public class Order {
    private OrderStatus status;

    public void cancel() {
        if (this.status != OrderStatus.PENDING) { // logic ở Entity — đúng hơn
            throw new IllegalStateException("Chỉ hủy được đơn PENDING");
        }
        this.status = OrderStatus.CANCELLED;
    }
}

@Service
public class OrderService {
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel(); // gọi method của entity
        orderRepository.save(order);
    }
}
```

---

## Khi Nào Outgrow Layered Architecture?

### Dấu Hiệu Cần Xem Xét Kiến Trúc Khác

```
🚨 Warning Signs (Dấu Hiệu Cảnh Báo):

1. "God Service" (Dịch Vụ Thượng Đế):
   - OrderService có 50+ methods
   - Service file dài 2000+ dòng
   → Chia nhỏ: OrderCreationService, OrderPaymentService, OrderFulfillmentService

2. Cross-cutting Concerns (Mối Quan Tâm Chéo):
   - Business logic bị lặp nhiều service
   → Dùng AOP (Aspect-Oriented Programming — Lập Trình Hướng Khía Cạnh)

3. Testability vấn đề:
   - Khó mock vì quá nhiều dependencies
   → Xem xét Clean Architecture / Hexagonal

4. Scale vấn đề:
   - Một tính năng deploy ảnh hưởng toàn bộ app
   → Xem xét Microservices

5. Team vấn đề:
   - 10+ developers cùng sửa 1 codebase
   → Feature modules hoặc Microservices
```

### Modular Monolith — Bước Trung Gian

```java
// Tổ chức theo feature module, vẫn là 1 process
// Mỗi module có layers riêng, interface rõ ràng

com.example/
├── order/
│   ├── api/         ← public interface của module
│   │   └── OrderFacade.java
│   ├── web/         ← Controller
│   ├── service/     ← Service
│   └── repository/  ← Repository
├── inventory/
│   ├── api/
│   │   └── InventoryFacade.java
│   ├── web/
│   ├── service/
│   └── repository/
└── payment/
    ├── api/
    │   └── PaymentFacade.java
    └── ...

// Modules giao tiếp qua Facade API, không phụ thuộc internal class
// Dễ tách thành microservice khi cần
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao không đặt logic vào Controller hay Repository?**

> Controller có trách nhiệm xử lý HTTP protocol — parse request, validate, trả response. Repository có trách nhiệm persistence (lưu trữ). Nếu đặt business logic ở đây, code bị coupled với transport layer hoặc database, khó test, khó reuse. Service layer là nơi duy nhất business logic được tập trung, orchestrate (điều phối) các operations và đảm bảo transaction boundaries.

**Q: Khi nào nên tạo interface cho Service?**

> Chỉ tạo interface khi: (1) có nhiều implementation cần swap, ví dụ `NotificationService` với `EmailNotificationService` và `SmsNotificationService`; (2) cần proxy thủ công; (3) đang áp dụng Ports & Adapters. Với service thông thường chỉ có 1 implementation, interface tạo thêm boilerplate không có giá trị — Spring CGLIB proxy hoạt động tốt mà không cần interface.

**Q: Layered Architecture có vấn đề gì với testability?**

> Vấn đề chính: nếu Service phụ thuộc nhiều tầng, test phải mock nhiều. Giải pháp: (1) Constructor injection để dễ mock; (2) Giữ Service nhỏ — một service chỉ 1 responsibility; (3) Tránh service gọi service khác — nếu cần thì xem xét event-driven.

**Q: @Transactional đặt ở đâu là đúng?**

> Service layer. Controller không nên có `@Transactional` vì nó xử lý HTTP không cần transaction. Repository layer Spring Data JPA đã tự quản lý transaction cho từng method. Service là nơi define transaction boundary — vì một business operation có thể gọi nhiều repository, tất cả cần cùng transaction.

---

## ✅ Checklist

- [ ] Controller chỉ xử lý HTTP — không có `if`, `for` business logic
- [ ] Service không import `HttpServletRequest` hay `@RequestMapping`
- [ ] Entity không expose trực tiếp qua API endpoint
- [ ] `@Transactional` ở Service, `readOnly = true` cho read operations
- [ ] Repository chỉ có queries, không có business validation
- [ ] DTO tách biệt cho Request và Response

---

**Xem tiếp:** [3-microservices.md](3-microservices.md) — Khi nào và cách migrate sang Microservices
