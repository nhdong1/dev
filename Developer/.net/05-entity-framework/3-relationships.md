# 3 — Relationships: Quan Hệ Giữa Các Entities

> Hiểu cách EF Core ánh xạ quan hệ giữa entities là kỹ năng thiết yếu để thiết kế schema đúng và tránh lỗi runtime.

---

## 📌 Tổng Quan Các Loại Quan Hệ

| Quan Hệ | Ví Dụ Thực Tế | EF Core |
|---------|--------------|---------|
| **One-to-Many** (1-N) | 1 Category → nhiều Products | Phổ biến nhất |
| **One-to-One** (1-1) | 1 User → 1 UserProfile | Ít phổ biến |
| **Many-to-Many** (N-N) | Products ↔ Tags | Dùng join table |
| **Owned Entity** | Address là một phần của Order | Không có bảng riêng |
| **Table Splitting** | 1 bảng → 2 entity classes | Tối ưu load selectively |

---

## 1. One-to-Many — Một Đến Nhiều (Phổ Biến Nhất)

### Định Nghĩa Entity

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Collection navigation property — Thuộc tính điều hướng tập hợp
    public ICollection<Product> Products { get; set; } = new List<Product>();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // Foreign key property — Thuộc tính khóa ngoại
    public int CategoryId { get; set; }

    // Reference navigation property — Thuộc tính điều hướng tham chiếu
    public Category Category { get; set; } = null!;
}
```

### Cấu Hình Fluent API

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder
            .HasOne(p => p.Category)       // Product có 1 Category
            .WithMany(c => c.Products)     // Category có nhiều Products
            .HasForeignKey(p => p.CategoryId)  // Khóa ngoại
            .OnDelete(DeleteBehavior.Restrict); // Không cascade delete
    }
}
```

### DeleteBehavior — Hành Vi Khi Xóa

| Giá Trị | Hành Vi |
|---------|---------|
| `Cascade` | Xóa Category → tự xóa tất cả Products (mặc định) |
| `Restrict` | Xóa Category → báo lỗi nếu còn Products |
| `SetNull` | Xóa Category → set `CategoryId = NULL` cho Products |
| `NoAction` | Để database xử lý (có thể gây lỗi FK violation) |

---

## 2. One-to-One — Một Đến Một

### Ví Dụ: User và UserProfile

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;

    // Navigation property
    public UserProfile? Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime? DateOfBirth { get; set; }

    // Foreign key — cũng là Primary Key (shared primary key pattern)
    public int UserId { get; set; }
    public User User { get; set; } = null!;
}
```

### Cấu Hình Fluent API

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder
            .HasOne(u => u.Profile)
            .WithOne(p => p.User)
            .HasForeignKey<UserProfile>(p => p.UserId) // Phải chỉ rõ FK nằm ở bên nào
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### Shared Primary Key Pattern (Tối Ưu)

```csharp
// UserProfile dùng UserId làm cả FK lẫn PK
public class UserProfile
{
    // Id = UserId (không cần cột Id riêng)
    public int UserId { get; set; } // Vừa là PK, vừa là FK
    // ... các thuộc tính khác
}

// Configuration
builder.HasKey(p => p.UserId); // PK là UserId
builder
    .HasOne(p => p.User)
    .WithOne(u => u.Profile)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

---

## 3. Many-to-Many — Nhiều Đến Nhiều

### Cách 1: EF Core 5+ — Implicit Join Table (Ẩn)

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Collection navigation property
    public ICollection<Tag> Tags { get; set; } = new List<Tag>();
}

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Product> Products { get; set; } = new List<Product>();
}
```

```csharp
// Cấu hình — EF Core tự tạo bảng join ProductTag
modelBuilder.Entity<Product>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Products);
```

EF Core tự tạo bảng `ProductTag` với cột `ProductsId` và `TagsId`.

### Cách 2: Explicit Join Entity — Bảng Join Có Thuộc Tính Thêm

Khi bảng join cần thêm dữ liệu (VD: ngày tạo, số lượng):

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }

    public ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    public ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();
}

// Explicit join entity — Thực thể join tường minh
public class OrderItem
{
    // Composite PK — Khóa chính ghép
    public int OrderId { get; set; }
    public int ProductId { get; set; }

    // Thuộc tính riêng của quan hệ
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; } // Giá tại thời điểm mua

    // Navigation properties
    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
}
```

```csharp
public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        // Composite primary key — Khóa chính ghép
        builder.HasKey(oi => new { oi.OrderId, oi.ProductId });

        builder
            .HasOne(oi => oi.Order)
            .WithMany(o => o.Items)
            .HasForeignKey(oi => oi.OrderId);

        builder
            .HasOne(oi => oi.Product)
            .WithMany(p => p.OrderItems)
            .HasForeignKey(oi => oi.ProductId);
    }
}
```

---

## 4. Owned Entities — Thực Thể Sở Hữu (Không Có Bảng Riêng)

Dùng cho Value Objects trong DDD — các đối tượng không có identity riêng, là một phần của entity cha.

### Ví Dụ: Address Là Một Phần Của Order

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }

    // Owned entity — được lưu vào cùng bảng Orders
    public Address ShippingAddress { get; set; } = null!;
    public Address BillingAddress { get; set; } = null!;
}

// Owned type — không có Id riêng
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}
```

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Khai báo là owned entity
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            // Đặt tên cột trong bảng Orders
            addr.Property(a => a.Street).HasColumnName("ShippingStreet").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("ShippingCity").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("ShippingPostalCode").HasMaxLength(20);
            addr.Property(a => a.Country).HasColumnName("ShippingCountry").HasMaxLength(100);
        });

        builder.OwnsOne(o => o.BillingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("BillingStreet").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("BillingCity").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("BillingPostalCode").HasMaxLength(20);
            addr.Property(a => a.Country).HasColumnName("BillingCountry").HasMaxLength(100);
        });
    }
}
```

### Schema Được Tạo Ra

```sql
-- Tất cả trong 1 bảng Orders (không có bảng Addresses riêng)
CREATE TABLE Orders (
    Id INT PRIMARY KEY,
    CreatedAt DATETIME2,
    ShippingStreet NVARCHAR(200),
    ShippingCity NVARCHAR(100),
    ShippingPostalCode NVARCHAR(20),
    ShippingCountry NVARCHAR(100),
    BillingStreet NVARCHAR(200),
    -- ...
)
```

### Owned Entity Trong Bảng Riêng (EF Core 7+)

```csharp
// Nếu muốn owned entity có bảng riêng
builder.OwnsOne(o => o.ShippingAddress, addr =>
{
    addr.ToTable("OrderShippingAddresses"); // Bảng riêng
});
```

---

## 5. Table Splitting — Tách Entity Từ 1 Bảng

Khi 1 bảng lớn và bạn muốn load một phần nhỏ thường xuyên, phần còn lại ít khi cần:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public bool IsActive { get; set; }

    // Navigation property đến phần chi tiết
    public ProductDetails Details { get; set; } = null!;
}

public class ProductDetails
{
    public int Id { get; set; }  // Cùng PK với Product
    public string Description { get; set; } = string.Empty;
    public string Specifications { get; set; } = string.Empty; // JSON hoặc text lớn
    public byte[]? Image { get; set; } // Ảnh sản phẩm — có thể rất nặng
}
```

```csharp
// Cả 2 entity dùng chung bảng Products
modelBuilder.Entity<Product>().ToTable("Products");
modelBuilder.Entity<ProductDetails>(entity =>
{
    entity.ToTable("Products"); // Cùng bảng
    entity.HasOne<Product>()
          .WithOne(p => p.Details)
          .HasForeignKey<ProductDetails>(d => d.Id);
});
```

**Lợi ích:** Khi query Products thông thường, không load `Description` và `Image` (nặng). Chỉ load khi cần: `context.Products.Include(p => p.Details)`.

---

## 6. Self-Referencing — Quan Hệ Tự Tham Chiếu

Dùng cho cấu trúc cây (tree structure) như danh mục phân cấp, comment thread:

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Khóa ngoại tự tham chiếu — nullable vì root không có parent
    public int? ParentId { get; set; }
    public Category? Parent { get; set; }

    // Children navigation
    public ICollection<Category> Children { get; set; } = new List<Category>();
}
```

```csharp
builder.HasOne(c => c.Parent)
       .WithMany(c => c.Children)
       .HasForeignKey(c => c.ParentId)
       .OnDelete(DeleteBehavior.Restrict);
```

---

## 7. Shadow Properties — Thuộc Tính Ẩn

Thuộc tính tồn tại trong database nhưng không có trong entity class:

```csharp
// Trong OnModelCreating — thêm CreatedAt cho mọi entity mà không cần property trong class
foreach (var entityType in modelBuilder.Model.GetEntityTypes())
{
    modelBuilder.Entity(entityType.ClrType)
        .Property<DateTime>("CreatedAt")
        .HasDefaultValueSql("GETUTCDATE()");
}

// Đọc shadow property
var createdAt = context.Entry(product).Property<DateTime>("CreatedAt").CurrentValue;
```

---

## 📋 Checklist Relationships

- [ ] Luôn định nghĩa rõ `OnDelete` behavior (không dùng mặc định Cascade cho tất cả)
- [ ] Với N-N có data thêm, dùng explicit join entity thay implicit
- [ ] Dùng `OwnsOne` cho Value Objects trong DDD (địa chỉ, tiền tệ)
- [ ] Cân nhắc `Table Splitting` khi entity có cột dữ liệu lớn ít dùng
- [ ] Index FK columns để tối ưu JOIN queries
- [ ] Đặt tên navigation properties nhất quán (số ít cho reference, số nhiều cho collection)

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng implicit Many-to-Many vs explicit join entity?**
> Dùng implicit khi quan hệ không có thêm dữ liệu. Dùng explicit join entity khi cần lưu thêm thông tin về quan hệ (VD: số lượng trong giỏ hàng, ngày tag, trạng thái). Explicit cũng dễ query hơn khi cần filter trên quan hệ.

**Q: Owned Entity vs Separate Table khác nhau thế nào?**
> Owned Entity lưu trong cùng bảng với entity cha, không có identity (ID) riêng — phù hợp cho Value Objects. Separate Table có bảng riêng, có ID riêng. Owned Entity tốt hơn cho dữ liệu nhỏ, luôn load cùng với entity cha.

**Q: `DeleteBehavior.Cascade` có phải luôn tốt không?**
> Không. Cascade delete nguy hiểm nếu dữ liệu quan trọng và quan hệ phức tạp. Mặc định nên dùng `Restrict` — nếu xóa parent mà còn child, EF báo lỗi. Điều này buộc developer xử lý việc xóa rõ ràng thay vì xóa ngầm.
