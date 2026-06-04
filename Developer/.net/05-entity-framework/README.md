# 📁 Entity Framework Core — Tổng Quan

> EF Core — Entity Framework Core — là ORM (Object-Relational Mapper — Ánh Xạ Đối Tượng Quan Hệ) chính thức của Microsoft cho .NET. Nó cho phép bạn làm việc với database bằng các đối tượng C# thay vì viết SQL thủ công.

---

## 🎯 Mục Tiêu Của Section Này

Sau khi hoàn thành, bạn sẽ có thể:

- [ ] Cấu hình `DbContext` và kết nối database đúng cách
- [ ] Tạo và quản lý migrations trong team
- [ ] Định nghĩa relationships (quan hệ) giữa các entities
- [ ] Viết LINQ queries hiệu quả với `Include`, `ThenInclude`, projection
- [ ] Phát hiện và giải quyết vấn đề N+1 Query
- [ ] Hiểu và kiểm soát Change Tracking
- [ ] Tối ưu hiệu năng EF Core trong production
- [ ] Viết tests cho code sử dụng EF Core

---

## 📁 Cấu Trúc

| File | Chủ Đề | Độ Quan Trọng |
|------|--------|---------------|
| `1-dbcontext-setup.md` | DbContext, connection string, pooling | ⭐⭐⭐ |
| `2-migrations.md` | Code-First migrations, rollback, team conflicts | ⭐⭐⭐ |
| `3-relationships.md` | 1-1, 1-N, N-N, owned entities, table splitting | ⭐⭐⭐ |
| `4-querying.md` | LINQ queries, Include, ThenInclude, projection | ⭐⭐⭐ |
| `5-n-plus-one-problem.md` | Phát hiện và giải quyết N+1 Query | ⭐⭐⭐ |
| `6-change-tracking.md` | AsNoTracking, Attach, EntityState | ⭐⭐ |
| `7-performance-tips.md` | Compiled queries, split queries, raw SQL, Dapper | ⭐⭐ |
| `8-testing-ef-core.md` | InMemory provider, SQLite, mocking DbContext | ⭐⭐ |

---

## 🧠 EF Core Là Gì?

EF Core là một **ORM — Object-Relational Mapper — Ánh Xạ Đối Tượng Quan Hệ** giúp bạn:

```
C# Objects  ←→  EF Core  ←→  SQL Database
  (Entity)        (ORM)        (SQL Server, PostgreSQL, SQLite...)
```

Thay vì viết:
```sql
SELECT * FROM Products WHERE Price > 100 ORDER BY Name
```

Bạn viết:
```csharp
var products = await context.Products
    .Where(p => p.Price > 100)
    .OrderBy(p => p.Name)
    .ToListAsync();
```

EF Core tự động dịch LINQ sang SQL tương ứng.

---

## 🏗️ Kiến Trúc EF Core

```
┌─────────────────────────────────────────────────────┐
│                  Application Code                    │
│  (LINQ queries, entity operations, migrations)       │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                    DbContext                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  DbSet<T>   │  │Change Tracker│  │  Model    │  │
│  │ (Tập thực   │  │(Theo dõi     │  │ (Schema   │  │
│  │  thể)       │  │ thay đổi)    │  │  cache)   │  │
│  └─────────────┘  └──────────────┘  └───────────┘  │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│               Database Provider                      │
│    SQL Server | PostgreSQL | SQLite | MySQL | ...    │
└─────────────────────────────────────────────────────┘
```

### Các Thành Phần Chính

| Thành Phần | Mô Tả |
|------------|-------|
| `DbContext` | Ngữ cảnh database — cửa ngõ để tương tác với DB |
| `DbSet<T>` | Tập thực thể — đại diện cho một bảng trong DB |
| `Entity` | Class C# ánh xạ với một bảng |
| `Migration` | Script SQL được sinh tự động từ thay đổi model |
| `Change Tracker` | Theo dõi trạng thái thay đổi của entities |
| `LINQ Provider` | Dịch LINQ queries sang SQL |

---

## 🔄 Hai Hướng Tiếp Cận

### Code-First — Ưu Tiên Code

```
1. Viết C# entity classes
2. Cấu hình relationships với Fluent API / Data Annotations
3. Chạy migration để tạo / cập nhật schema DB
4. EF Core quản lý schema
```

**Dùng khi:** Dự án mới, team .NET ưu tiên codebase.

### Database-First — Ưu Tiên Database

```
1. Thiết kế schema trong database trước
2. Dùng `dotnet ef dbcontext scaffold` để tạo entities
3. Code được generate từ schema hiện có
4. Thay đổi schema → scaffold lại
```

**Dùng khi:** Database đã có sẵn, DBA quản lý schema.

> **Thực tế:** Code-First với migrations là hướng tiếp cận phổ biến nhất trong dự án .NET hiện đại.

---

## 📦 Cài Đặt

```bash
# Package chính
dotnet add package Microsoft.EntityFrameworkCore

# Database provider (chọn một)
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Sqlite

# Tools để chạy migration
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet tool install --global dotnet-ef
```

---

## ⚡ Quick Start — Bắt Đầu Nhanh

### 1. Định Nghĩa Entity

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int CategoryId { get; set; }

    // Navigation property — Thuộc tính điều hướng
    public Category Category { get; set; } = null!;
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Collection navigation property
    public ICollection<Product> Products { get; set; } = new List<Product>();
}
```

### 2. Tạo DbContext

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Fluent API configuration
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Name).HasMaxLength(200).IsRequired();
            entity.Property(p => p.Price).HasColumnType("decimal(18,2)");
            entity.HasOne(p => p.Category)
                  .WithMany(c => c.Products)
                  .HasForeignKey(p => p.CategoryId);
        });
    }
}
```

### 3. Đăng Ký DI

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

### 4. Dùng Trong Service

```csharp
public class ProductService
{
    private readonly AppDbContext _context;

    public ProductService(AppDbContext context) => _context = context;

    public async Task<List<Product>> GetAllAsync()
        => await _context.Products.Include(p => p.Category).ToListAsync();

    public async Task<Product?> GetByIdAsync(int id)
        => await _context.Products.FindAsync(id);

    public async Task AddAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
    }
}
```

---

## 🗺️ Lộ Trình Học

```
Cơ Bản (phải biết)
├── 1-dbcontext-setup.md      → Hiểu DbContext lifecycle, pooling
├── 2-migrations.md           → Quản lý schema thay đổi theo thời gian
└── 3-relationships.md        → Định nghĩa quan hệ giữa entities

Trung Cấp (cần thành thạo)
├── 4-querying.md             → Viết queries hiệu quả
└── 5-n-plus-one-problem.md   → Vấn đề phổ biến nhất trong EF Core

Nâng Cao (hiểu sâu)
├── 6-change-tracking.md      → Kiểm soát performance và behavior
├── 7-performance-tips.md     → Tối ưu cho production
└── 8-testing-ef-core.md      → Test code EF Core đúng cách
```

---

## 🎯 Câu Hỏi Phỏng Vấn Quan Trọng Nhất

| Câu Hỏi | File Tham Khảo |
|---------|----------------|
| Code-First vs Database-First là gì? | README.md (file này) |
| Migration hoạt động như thế nào? | 2-migrations.md |
| N+1 query problem là gì? Cách fix? | 5-n-plus-one-problem.md |
| `AsNoTracking()` dùng khi nào? | 6-change-tracking.md |
| Khi nào nên dùng Dapper thay EF Core? | 7-performance-tips.md |
| Cách test code dùng DbContext? | 8-testing-ef-core.md |

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản EF Core:** 8.x (LTS)
