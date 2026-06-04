# ORM Mapping — Ánh Xạ Quan Hệ Đối Tượng

> ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng Quan Hệ) là cơ chế chuyển đổi giữa
> đối tượng Java và bảng trong cơ sở dữ liệu quan hệ. Hiểu đúng ORM mapping tránh được
> nhiều lỗi hiệu năng và dữ liệu nghiêm trọng trong production.

---

## 📋 Mục Tiêu

- [ ] Ánh xạ các loại quan hệ: **`@OneToOne`**, **`@OneToMany`**, **`@ManyToOne`**, **`@ManyToMany`**
- [ ] Hiểu và chọn đúng **`FetchType`** — `LAZY` vs `EAGER`
- [ ] Cấu hình **`CascadeType`** — Khi nào cascade, khi nào không
- [ ] Thiết kế **Bidirectional Relationship** (Quan Hệ Hai Chiều) tránh vòng lặp vô hạn
- [ ] Dùng **`@MappedSuperclass`** và **`@Inheritance`** cho Entity hierarchy (Phân Cấp Thực Thể)
- [ ] Áp dụng **Optimistic Locking** (Khóa Lạc Quan) với `@Version`

---

## 1. @ManyToOne và @OneToMany — Quan Hệ Nhiều-Một / Một-Nhiều

Đây là quan hệ phổ biến nhất. Ví dụ: Nhiều `Order` thuộc về 1 `User`.

### Cách Thiết Kế Đúng — Bidirectional

```java
// Phía "nhiều" — sở hữu foreign key
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ManyToOne: Nhiều Order → 1 User
    // @JoinColumn: Tên cột foreign key trong bảng "orders"
    @ManyToOne(fetch = FetchType.LAZY)      // ✅ LAZY là mặc định tốt nhất
    @JoinColumn(name = "user_id",            // Tên cột FK trong bảng orders
                nullable = false,
                foreignKey = @ForeignKey(name = "fk_orders_user_id"))
    private User user;

    @Column(nullable = false)
    private BigDecimal total;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // Getter/Setter...
}

// Phía "một" — không sở hữu FK, mappedBy trỏ đến field bên Order
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    // OneToMany: 1 User → Nhiều Order
    // mappedBy: tên field "user" trong class Order — KHÔNG tạo join table mới
    @OneToMany(mappedBy = "user",
               cascade = CascadeType.ALL,    // Cascade tất cả operations (dùng cẩn thận!)
               orphanRemoval = true,         // Xóa Order khi bị remove khỏi collection
               fetch = FetchType.LAZY)       // ✅ Mặc định LAZY với @OneToMany
    private List<Order> orders = new ArrayList<>();

    // Helper methods — đảm bảo đồng bộ 2 chiều
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);  // Đồng bộ phía Order
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);  // Đồng bộ phía Order
    }
}
```

```
Bảng trong DB:
┌──────────────┐      ┌──────────────┐
│    users     │      │    orders    │
├──────────────┤      ├──────────────┤
│ id (PK)      │◄─────│ user_id (FK) │
│ email        │      │ id (PK)      │
│ ...          │      │ total        │
└──────────────┘      └──────────────┘
```

### Cascade Types — Các Loại Lan Truyền

```java
// CascadeType.PERSIST  — Khi save User → tự save Orders chưa có trong DB
// CascadeType.MERGE    — Khi merge User → tự merge Orders
// CascadeType.REMOVE   — Khi xóa User → xóa tất cả Orders ⚠️ Nguy hiểm!
// CascadeType.REFRESH  — Khi refresh User → refresh Orders
// CascadeType.DETACH   — Khi detach User → detach Orders
// CascadeType.ALL      — Tất cả các loại trên

// ✅ Thực tế thường dùng
@OneToMany(mappedBy = "user", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<Order> orders;

// ✅ Với orphanRemoval — xóa child khi bị remove khỏi collection
@OneToMany(mappedBy = "user",
           cascade = CascadeType.ALL,
           orphanRemoval = true)  // orphanRemoval = true ngầm định bao gồm REMOVE
private List<Address> addresses;

// ⚠️ TRÁNH CascadeType.REMOVE cho entity quan trọng
// Nếu vô tình xóa User → xóa cả nghìn Orders → thảm họa!
// Thay vào đó: xóa thủ công hoặc dùng soft delete
```

### orphanRemoval vs CascadeType.REMOVE

```
orphanRemoval = true:
    user.getAddresses().remove(address);  → address bị xóa khỏi DB
    (address không còn cha nào → xóa "orphan")

CascadeType.REMOVE:
    repository.delete(user);  → xóa user + tất cả addresses
    (lan truyền DELETE operation)

Dùng cả hai:
    - orphanRemoval: Xóa child khi tách khỏi collection của parent
    - REMOVE: Xóa child khi parent bị xóa
```

---

## 2. @OneToOne — Quan Hệ Một-Một

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    // OneToOne: 1 User → 1 UserProfile
    // Đặt @JoinColumn ở phía nào thì phía đó sở hữu FK
    @OneToOne(cascade = CascadeType.ALL,
              fetch = FetchType.LAZY,  // ✅ Luôn dùng LAZY với @OneToOne
              optional = false)        // NOT NULL — user phải có profile
    @JoinColumn(name = "profile_id",   // FK trong bảng users
                unique = true,
                foreignKey = @ForeignKey(name = "fk_users_profile_id"))
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String bio;
    private String avatarUrl;
    private LocalDate birthDate;

    // Phía không sở hữu FK — mappedBy trỏ đến field "profile" trong User
    @OneToOne(mappedBy = "profile", fetch = FetchType.LAZY)
    private User user;
}
```

> **Lưu ý:** `@OneToOne` với `FetchType.EAGER` (mặc định!) là nguyên nhân phổ biến gây N+1.
> Luôn chỉ định `fetch = FetchType.LAZY` cho `@OneToOne`.

---

## 3. @ManyToMany — Quan Hệ Nhiều-Nhiều

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // ManyToMany: Nhiều Product ↔ Nhiều Tag
    // @JoinTable: Định nghĩa bảng trung gian
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE},
                fetch = FetchType.LAZY)
    @JoinTable(
        name = "product_tags",                      // Tên bảng trung gian
        joinColumns = @JoinColumn(name = "product_id"),   // FK → Product
        inverseJoinColumns = @JoinColumn(name = "tag_id") // FK → Tag
    )
    private Set<Tag> tags = new HashSet<>();  // ✅ Dùng Set thay List cho ManyToMany

    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getProducts().add(this);  // Đồng bộ 2 chiều
    }

    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getProducts().remove(this);
    }
}

@Entity
@Table(name = "tags")
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;

    @ManyToMany(mappedBy = "tags", fetch = FetchType.LAZY)
    private Set<Product> products = new HashSet<>();
}
```

### Khi Cần Thêm Thuộc Tính Vào Bảng Trung Gian — Dùng Entity Trung Gian

```java
// ❌ @ManyToMany không cho phép thêm cột vào bảng trung gian
// ✅ Dùng Entity trung gian với Composite Key hoặc Surrogate Key

@Entity
@Table(name = "student_courses")
public class StudentCourse {

    @EmbeddedId
    private StudentCourseId id = new StudentCourseId();

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")  // Ánh xạ vào field studentId của EmbeddedId
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    @JoinColumn(name = "course_id")
    private Course course;

    // Thêm cột tùy chỉnh vào bảng trung gian
    private LocalDate enrolledAt;

    @Enumerated(EnumType.STRING)
    private EnrollmentStatus status;

    private Double grade;
}

// Composite Primary Key (Khóa Chính Tổng Hợp)
@Embeddable
public class StudentCourseId implements Serializable {
    private Long studentId;
    private Long courseId;

    // equals() và hashCode() BẮT BUỘC phải implement đúng!
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof StudentCourseId that)) return false;
        return Objects.equals(studentId, that.studentId) &&
               Objects.equals(courseId, that.courseId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(studentId, courseId);
    }
}

// Student Entity
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "student", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<StudentCourse> enrollments = new ArrayList<>();
}
```

---

## 4. FetchType — Chiến Lược Tải Dữ Liệu

```
FetchType.LAZY  — Tải khi cần (lazy load):
    - Không tải dữ liệu liên quan khi load entity chính
    - Chỉ tải khi truy cập field đó (proxy call)
    - ✅ Mặc định cho @OneToMany và @ManyToMany
    - ✅ Khuyến nghị cho @ManyToOne và @OneToOne

FetchType.EAGER — Tải ngay lập tức:
    - Tải dữ liệu liên quan cùng lúc với entity chính
    - ✅ Mặc định cho @ManyToOne và @OneToOne (nhưng nên override)
    - ❌ Nguy hiểm với collection — gây N+1 Problem
```

```java
// ❌ Anti-pattern — EAGER với collection
@OneToMany(mappedBy = "order", fetch = FetchType.EAGER)
private List<OrderItem> items;
// Mỗi lần load Order → tự động load tất cả OrderItems → N+1!

// ✅ Pattern đúng — LAZY + tải khi cần
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
private List<OrderItem> items;

// Khi cần items, dùng JOIN FETCH trong query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

### FetchType Mặc Định

| Annotation | FetchType Mặc Định | Khuyến Nghị |
|------------|-------------------|-------------|
| `@OneToMany` | LAZY | LAZY ✅ |
| `@ManyToMany` | LAZY | LAZY ✅ |
| `@ManyToOne` | EAGER ⚠️ | Override → LAZY ✅ |
| `@OneToOne` | EAGER ⚠️ | Override → LAZY ✅ |

> **Quy tắc vàng:** Luôn dùng `LAZY`, chỉ dùng `EAGER` khi có benchmark chứng minh nó nhanh hơn.

---

## 5. Equals và HashCode — Bắt Buộc Cho Entity

```java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ✅ Dùng natural key (khóa tự nhiên) nếu có
    @Column(unique = true, nullable = false)
    private String sku;

    // ✅ equals/hashCode dựa trên business key, không phải id
    // (id có thể null trước khi persist)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product product)) return false;
        return sku != null && sku.equals(product.sku);
    }

    @Override
    public int hashCode() {
        // Trả về hằng số cố định — đơn giản nhưng đúng với JPA lifecycle
        // (Tránh thay đổi hashCode sau khi đã add vào HashSet)
        return getClass().hashCode();
    }
}
```

---

## 6. @MappedSuperclass — Base Entity Tái Sử Dụng

```java
// Base class — không tạo bảng riêng, chỉ kế thừa fields
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Version
    private Long version;  // Optimistic Locking — xem phần dưới
}

// Các entity kế thừa — tất cả fields của BaseEntity được thêm vào bảng
@Entity
@Table(name = "products")
public class Product extends BaseEntity {
    private String name;
    private BigDecimal price;
    // id, createdAt, updatedAt, version được kế thừa từ BaseEntity
}

@Entity
@Table(name = "categories")
public class Category extends BaseEntity {
    private String name;
    private String slug;
    // id, createdAt, updatedAt, version được kế thừa từ BaseEntity
}
```

---

## 7. @Inheritance — Phân Cấp Entity (Entity Hierarchy)

### Strategy 1: SINGLE_TABLE — Một Bảng Cho Toàn Bộ Hierarchy

```java
@Entity
@Table(name = "notifications")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "notification_type",    // Cột phân biệt loại
                     discriminatorType = DiscriminatorType.STRING)
public abstract class Notification {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;
    private String message;
    private boolean read;
    private LocalDateTime createdAt;
}

@Entity
@DiscriminatorValue("EMAIL")  // Giá trị trong cột notification_type
public class EmailNotification extends Notification {
    private String emailSubject;
    private String emailTo;
    // Các cột này có thể NULL với các type khác → đánh đổi
}

@Entity
@DiscriminatorValue("PUSH")
public class PushNotification extends Notification {
    private String deviceToken;
    private String pushTitle;
}

/*
 Bảng DB kết quả:
 ┌──────────────────────────────────────────────────────────────────────┐
 │                         notifications                                │
 ├────┬─────────┬──────────┬───────────────────┬─────────┬─────────────┤
 │ id │ user_id │ message  │ notification_type │ email_to│ device_token│
 ├────┼─────────┼──────────┼───────────────────┼─────────┼─────────────┤
 │  1 │   100   │ "Hello"  │ EMAIL             │ a@b.com │   NULL      │
 │  2 │   200   │ "Update" │ PUSH              │   NULL  │ token123    │
 └────┴─────────┴──────────┴───────────────────┴─────────┴─────────────┘
 
 ✅ Ưu điểm: 1 bảng → JOIN nhanh, query đơn giản
 ❌ Nhược điểm: Nhiều cột NULL → lãng phí, ràng buộc NOT NULL khó áp dụng
*/
```

### Strategy 2: JOINED — Mỗi Class Một Bảng, JOIN Khi Truy Vấn

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private BigDecimal amount;
    private LocalDateTime processedAt;
}

@Entity
@Table(name = "credit_card_payments")
@PrimaryKeyJoinColumn(name = "payment_id")  // FK → bảng payments
public class CreditCardPayment extends Payment {
    private String cardLastFour;
    private String cardNetwork;  // VISA, MASTERCARD...
}

@Entity
@Table(name = "bank_transfer_payments")
@PrimaryKeyJoinColumn(name = "payment_id")
public class BankTransferPayment extends Payment {
    private String bankName;
    private String accountNumber;
}

/*
 ✅ Ưu điểm: Schema sạch, không có NULL, ràng buộc đầy đủ
 ❌ Nhược điểm: JOIN mỗi lần query → chậm hơn SINGLE_TABLE
*/
```

### Strategy 3: TABLE_PER_CLASS — Mỗi Concrete Class Một Bảng Riêng

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {
    @Id @GeneratedValue(strategy = GenerationType.AUTO) // Không dùng IDENTITY!
    private Long id;
    private String brand;
}

@Entity
@Table(name = "cars")
public class Car extends Vehicle {
    private int numberOfDoors;
}

@Entity
@Table(name = "motorcycles")
public class Motorcycle extends Vehicle {
    private boolean hasSidecar;
}
/*
 ✅ Ưu điểm: Bảng độc lập, không NULL
 ❌ Nhược điểm: Polymorphic query → UNION ALL → rất chậm
 ❌ Không hỗ trợ IDENTITY generation (phải dùng SEQUENCE hoặc TABLE)
 → Ít dùng trong thực tế
*/
```

### So Sánh Các Strategy

| Strategy | Bảng DB | NULL | JOIN | Polymorphic Query | Sử Dụng |
|----------|---------|------|------|-------------------|---------|
| SINGLE_TABLE | 1 | Nhiều | Không | Nhanh | Hierarchy đơn giản |
| JOINED | N+1 | Không | Mỗi query | Trung bình | Schema sạch quan trọng |
| TABLE_PER_CLASS | N | Không | UNION ALL | Chậm | Tránh dùng |

---

## 8. @Version — Optimistic Locking (Khóa Lạc Quan)

Optimistic Locking ngăn **Lost Update Problem** (Vấn Đề Mất Bản Ghi Cập Nhật) trong môi trường concurrent (đồng thời).

```java
@Entity
public class Product {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stockQuantity;

    @Version  // Hibernate tự động quản lý — tăng mỗi khi UPDATE
    private Long version;
}
```

```
Luồng hoạt động khi 2 user cùng update:

User A: SELECT * FROM products WHERE id=1  → version=5, stock=100
User B: SELECT * FROM products WHERE id=1  → version=5, stock=100

User A: UPDATE products SET stock=90, version=6 WHERE id=1 AND version=5
        → Thành công (version=5 khớp)

User B: UPDATE products SET stock=110, version=6 WHERE id=1 AND version=5
        → THẤT BẠI! version đã là 6 rồi, WHERE version=5 không khớp
        → Hibernate ném OptimisticLockException
        → Ứng dụng bắt exception → thông báo user retry
```

```java
// Xử lý OptimisticLockException
@Service
public class InventoryService {

    @Retryable(
        retryFor = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        if (product.getStockQuantity() < quantity) {
            throw new InsufficientStockException(productId, quantity);
        }

        product.setStockQuantity(product.getStockQuantity() - quantity);
        productRepository.save(product);
        // Nếu version conflict → OptimisticLockingFailureException → @Retryable retry
    }
}
```

### Pessimistic Locking — Khóa Bi Quan

```java
// Dùng khi xác suất conflict cao và không muốn retry
public interface ProductRepository extends JpaRepository<Product, Long> {

    // SELECT ... FOR UPDATE — khóa dòng tại DB
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    // SELECT ... FOR SHARE — nhiều reader, không writer
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<Product> findById(Long id);
}

// Sử dụng trong service
@Transactional
public void processOrder(Long productId, int quantity) {
    Product product = productRepository.findByIdForUpdate(productId)
        .orElseThrow();  // Dòng bị LOCK cho đến khi transaction kết thúc
    product.setStockQuantity(product.getStockQuantity() - quantity);
    // Không cần @Version — DB đảm bảo chỉ 1 transaction xử lý tại 1 thời điểm
}
```

---

## 9. @Embeddable — Value Objects (Đối Tượng Giá Trị)

```java
// @Embeddable — không có bảng riêng, các fields được nhúng vào entity chứa
@Embeddable
public class Address {
    @Column(name = "street", length = 200)
    private String street;

    @Column(name = "city", length = 100)
    private String city;

    @Column(name = "country", length = 2)
    private String countryCode;

    @Column(name = "postal_code", length = 20)
    private String postalCode;
}

// Entity nhúng Address
@Entity
@Table(name = "users")
public class User {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    @Embedded
    private Address shippingAddress;

    // Khi có 2 Address embedded — dùng @AttributeOverrides để đổi tên cột
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street",     column = @Column(name = "billing_street")),
        @AttributeOverride(name = "city",       column = @Column(name = "billing_city")),
        @AttributeOverride(name = "countryCode",column = @Column(name = "billing_country")),
        @AttributeOverride(name = "postalCode", column = @Column(name = "billing_postal_code"))
    })
    private Address billingAddress;
}

/*
 Bảng users sẽ có columns:
 id, email, street, city, country, postal_code,   ← shippingAddress
 billing_street, billing_city, billing_country, billing_postal_code  ← billingAddress
*/
```

---

## 10. @ElementCollection — Collection Của Kiểu Đơn Giản

```java
@Entity
public class User {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // Collection của String — tạo bảng phụ "user_phone_numbers"
    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "user_phone_numbers",
        joinColumns = @JoinColumn(name = "user_id")
    )
    @Column(name = "phone_number", length = 20)
    private List<String> phoneNumbers = new ArrayList<>();

    // Collection của @Embeddable — tạo bảng phụ "user_addresses"
    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "user_addresses",
        joinColumns = @JoinColumn(name = "user_id")
    )
    private List<Address> addresses = new ArrayList<>();
}

/*
 Bảng user_phone_numbers:
 ┌─────────┬──────────────┐
 │ user_id │ phone_number │
 ├─────────┼──────────────┤
 │    1    │ +84901234567 │
 │    1    │ +84907654321 │
 └─────────┴──────────────┘
*/
```

---

## ✅ Checklist Thiết Kế Entity

- [ ] Mọi `@OneToMany` và `@ManyToMany` đều có `fetch = FetchType.LAZY`
- [ ] Mọi `@ManyToOne` và `@OneToOne` override về `LAZY`
- [ ] Bidirectional relationships có helper methods để đồng bộ 2 chiều
- [ ] `equals()` và `hashCode()` implement dựa trên business key (không phải id)
- [ ] `@ManyToMany` dùng `Set<>` thay vì `List<>` để tránh duplicate
- [ ] Có `@Version` cho entity hay bị concurrent update
- [ ] Base entity với `@MappedSuperclass` cho audit fields

---

## 💡 Anti-Patterns Phổ Biến

```java
// ❌ Bidirectional mà không đồng bộ 2 chiều
order.getItems().add(item);
// Quên: item.setOrder(order); → item.order là null trong DB!

// ❌ toString() include collection → StackOverflow hoặc LazyInitializationException
@Override
public String toString() {
    return "User{orders=" + orders + "}"; // Trigger lazy load hoặc vòng lặp vô hạn!
}

// ❌ equals/hashCode dựa trên id
@Override
public boolean equals(Object o) {
    return id != null && id.equals(((Product)o).id); // id null trước khi persist!
}

// ❌ CascadeType.ALL cho ManyToMany
@ManyToMany(cascade = CascadeType.ALL)
private Set<Tag> tags;
// Xóa product → xóa tag → tag của product khác cũng mất!
```

---

## 🔗 Liên Kết

- [1-spring-data-jpa.md](1-spring-data-jpa.md) — JpaRepository, @Query
- [3-transactions.md](3-transactions.md) — @Transactional với cascade
- [4-n-plus-one-problem.md](4-n-plus-one-problem.md) — FetchType và N+1
- [Vlad Mihalcea — Best Way to Map @OneToMany](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/)
- [Thorben Janssen — @MappedSuperclass vs @Inheritance](https://thorben-janssen.com/hibernate-tips-how-to-map-an-inheritance-hierarchy-to-one-database-table/)

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
