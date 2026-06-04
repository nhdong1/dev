# 7 — Performance Tips: Compiled Queries, Split Queries, Raw SQL & Dapper

> Tối ưu hiệu năng EF Core không phải "chuyển sang Dapper". Phần lớn vấn đề giải quyết được bằng cách dùng EF Core đúng cách. Dapper chỉ là lựa chọn cho trường hợp đặc biệt.

---

## 📌 Thứ Tự Tối Ưu (Từ Dễ Đến Khó)

```
1. Projection thay vì Select *          → Ít data truyền từ DB
2. AsNoTracking()                        → Tắt Change Tracker
3. Indexes đúng chỗ                      → Query DB nhanh hơn
4. Include đúng cách / tránh N+1         → Ít round-trips
5. Compiled Queries                      → Bỏ qua LINQ translation
6. Split Queries / Raw SQL               → Tránh cartesian explosion
7. DbContextPool                         → Tái sử dụng context
8. Dapper (nếu cần)                      → Khi EF Core không đủ
```

---

## 1. Projection — Chọn Đúng Cột Cần Thiết

```csharp
// ❌ SELECT * — load toàn bộ entity
var products = await context.Products
    .Include(p => p.Category)
    .ToListAsync();

// Từ đó lấy tên
var names = products.Select(p => $"{p.Name} ({p.Category.Name})").ToList();

// ✅ Projection — chỉ SELECT đúng cột cần
var names = await context.Products
    .Select(p => $"{p.Name} ({p.Category.Name})")
    .ToListAsync();
// SQL: SELECT p.Name, c.Name FROM Products p JOIN Categories c ON ...
```

**Nguyên tắc:** Nếu bạn biết mình cần gì, dùng `Select()`. Chỉ dùng `Include()` khi thật sự cần full entity.

---

## 2. Compiled Queries — Bỏ Qua Bước Dịch LINQ

Mỗi lần EF Core thực thi LINQ query, nó phải:
1. Parse LINQ expression tree
2. Translate sang SQL
3. Compile SQL parameters

**Compiled Queries** làm bước 1-2 chỉ 1 lần:

```csharp
public class ProductRepository
{
    // Static field — khởi tạo 1 lần duy nhất khi app start
    private static readonly Func<AppDbContext, int, Task<Product?>> GetByIdQuery =
        EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
            ctx.Products
               .Include(p => p.Category)
               .FirstOrDefault(p => p.Id == id));

    private static readonly Func<AppDbContext, bool, IAsyncEnumerable<Product>> GetByStatusQuery =
        EF.CompileAsyncQuery((AppDbContext ctx, bool isActive) =>
            ctx.Products
               .AsNoTracking()
               .Where(p => p.IsActive == isActive)
               .OrderBy(p => p.Name));

    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context) => _context = context;

    // Gọi compiled query — nhanh hơn ~15-25% so với thường
    public Task<Product?> GetByIdAsync(int id)
        => GetByIdQuery(_context, id);

    public IAsyncEnumerable<Product> GetByStatusAsync(bool isActive)
        => GetByStatusQuery(_context, isActive);
}
```

**Khi nào dùng:** Queries chạy thường xuyên (hàng nghìn lần/phút), hot paths trong ứng dụng.

---

## 3. Split Queries — Tránh Cartesian Explosion

```csharp
// ❌ Vấn đề: 1 Order có 100 Items và 50 Tags
// SQL JOIN tạo ra 100 × 50 = 5,000 rows để truyền về
var orders = await context.Orders
    .Include(o => o.Items)  // 100 items
    .Include(o => o.Tags)   // 50 tags
    .ToListAsync();

// ✅ Split Query: 3 queries nhỏ thay vì 1 query khổng lồ
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .AsSplitQuery()  // ← Thêm dòng này
    .ToListAsync();

// EF Core tạo:
// Query 1: SELECT * FROM Orders WHERE ...
// Query 2: SELECT * FROM OrderItems WHERE OrderId IN (1,2,3,...)
// Query 3: SELECT * FROM OrderTags WHERE OrderId IN (1,2,3,...)
```

### Bật Split Query Mặc Định

```csharp
options.UseSqlServer(connectionString, sql =>
    sql.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

**Nhược điểm của Split Query:** Không nhất quán nếu có thay đổi data giữa các queries (race condition). Dùng khi dữ liệu tương đối ổn định.

---

## 4. Indexes — Chỉ Mục Database

Quan trọng như viết query đúng. EF Core tự tạo index cho PK và FK, nhưng bạn cần thêm cho các cột hay query:

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        // Index cho cột thường dùng trong WHERE
        builder.HasIndex(p => p.CategoryId);    // Đã có vì là FK
        builder.HasIndex(p => p.IsActive);      // Thường filter theo IsActive

        // Composite index — Chỉ mục ghép (thứ tự quan trọng!)
        builder.HasIndex(p => new { p.CategoryId, p.IsActive })
               .HasDatabaseName("IX_Products_CategoryId_IsActive");

        // Unique index — Chỉ mục duy nhất
        builder.HasIndex(p => p.Slug)
               .IsUnique()
               .HasFilter("IsActive = 1"); // Filtered index (SQL Server)

        // Index với Include columns (Covering index — Chỉ mục bao phủ)
        builder.HasIndex(p => p.CategoryId)
               .IncludeProperties(p => new { p.Name, p.Price }); // EF Core 8+
    }
}
```

**Nguyên tắc chọn index:** Ưu tiên cột trong `WHERE`, `JOIN ON`, `ORDER BY`. Quá nhiều index → INSERT/UPDATE chậm hơn.

---

## 5. Bulk Operations — Thao Tác Hàng Loạt

### EF Core 7+ — ExecuteUpdate / ExecuteDelete (Built-in)

```csharp
// Bulk UPDATE — không load entities, 1 SQL duy nhất
await context.Products
    .Where(p => p.CategoryId == 5)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));
// SQL: UPDATE Products SET IsActive = 0, UpdatedAt = ... WHERE CategoryId = 5

// Bulk DELETE — không load entities
await context.Products
    .Where(p => p.IsActive == false && p.CreatedAt < cutoffDate)
    .ExecuteDeleteAsync();
```

### EFCore.BulkExtensions (Third-party — Cho Bulk Insert)

```bash
dotnet add package EFCore.BulkExtensions
```

```csharp
// Bulk INSERT — 10,000 rows trong 1 operation
var products = GenerateProducts(10_000);
await context.BulkInsertAsync(products);

// Bulk UPSERT — Insert or Update
await context.BulkInsertOrUpdateAsync(products);

// So sánh hiệu năng INSERT 10,000 rows:
// context.AddRange + SaveChangesAsync: ~5,000ms
// BulkInsertAsync:                     ~200ms
```

---

## 6. Pagination — Phân Trang Hiệu Quả

```csharp
// ❌ Offset-based pagination — chậm khi page lớn
// Database phải đọc và bỏ qua (pageNumber-1)*pageSize rows
var page100 = await context.Products
    .OrderBy(p => p.Id)
    .Skip((100 - 1) * 20)  // Bỏ qua 1,980 rows — phí!
    .Take(20)
    .ToListAsync();

// ✅ Keyset pagination (Cursor-based) — tốt hơn cho dữ liệu lớn
// Bắt đầu từ cursor (ID cuối của trang trước)
var nextPage = await context.Products
    .Where(p => p.Id > lastSeenId)  // Bắt đầu sau cursor
    .OrderBy(p => p.Id)
    .Take(20)
    .ToListAsync();
```

**Keyset pagination tốt hơn khi:** Dataset lớn (>100K rows), cần scroll vô hạn (infinite scroll).

---

## 7. Raw SQL Với EF Core

```csharp
// FromSqlRaw — Query trả về entities (có thể kết hợp LINQ)
var products = await context.Products
    .FromSqlRaw(@"
        SELECT p.*
        FROM Products p
        INNER JOIN Categories c ON p.CategoryId = c.Id
        WHERE c.Name = {0} AND p.Price > {1}",
        categoryName, minPrice)
    .Where(p => p.IsActive)       // LINQ sau Raw SQL
    .OrderBy(p => p.Name)
    .ToListAsync();

// FromSqlInterpolated — An toàn hơn (parameterized)
var products = await context.Products
    .FromSqlInterpolated($@"
        SELECT * FROM Products
        WHERE CategoryId = {categoryId}
          AND Price BETWEEN {minPrice} AND {maxPrice}")
    .ToListAsync();

// ExecuteSqlRaw — Không trả về entity (INSERT/UPDATE/DELETE/EXEC)
await context.Database.ExecuteSqlRawAsync(
    "EXEC sp_UpdateProductPrices @CategoryId = {0}, @Multiplier = {1}",
    categoryId, 1.1m);

// SqlQuery — Trả về scalar hoặc non-entity type (EF Core 7+)
var priceSums = await context.Database
    .SqlQuery<decimal>($"SELECT SUM(Price) FROM Products WHERE IsActive = 1")
    .FirstAsync();
```

---

## 8. EF Core vs Dapper — So Sánh

| Tiêu Chí | EF Core | Dapper |
|---------|---------|--------|
| **Ease of use** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Complex queries** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Type safety** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Migration support** | ⭐⭐⭐⭐⭐ | ❌ |
| **Change tracking** | ⭐⭐⭐⭐⭐ | ❌ |
| **Learning curve** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### Khi Nào Dùng Dapper?

```csharp
// Cài Dapper
// dotnet add package Dapper

using Dapper;

public class ReportService
{
    private readonly IDbConnection _connection;

    public ReportService(IDbConnection connection) => _connection = connection;

    // Dapper: Query phức tạp với nhiều JOINs, GROUP BY, subqueries
    public async Task<IEnumerable<SalesReport>> GetSalesReportAsync(
        DateTime from, DateTime to)
    {
        const string sql = @"
            SELECT
                c.Name AS CategoryName,
                COUNT(DISTINCT o.Id) AS OrderCount,
                SUM(oi.Quantity * oi.UnitPrice) AS Revenue,
                AVG(oi.UnitPrice) AS AvgPrice,
                RANK() OVER (ORDER BY SUM(oi.Quantity * oi.UnitPrice) DESC) AS RevenueRank
            FROM Orders o
            INNER JOIN OrderItems oi ON o.Id = oi.OrderId
            INNER JOIN Products p ON oi.ProductId = p.Id
            INNER JOIN Categories c ON p.CategoryId = c.Id
            WHERE o.CreatedAt BETWEEN @From AND @To
              AND o.Status = 'Completed'
            GROUP BY c.Id, c.Name
            HAVING COUNT(DISTINCT o.Id) > 10
            ORDER BY Revenue DESC";

        return await _connection.QueryAsync<SalesReport>(sql, new { From = from, To = to });
    }
}
```

**Nên dùng Dapper khi:**
- Query quá phức tạp để EF Core dịch tốt (window functions, CTEs phức tạp)
- Hiệu năng là yêu cầu cực kỳ quan trọng (reporting, analytics, data export)
- Stored Procedures trả về multiple result sets
- Team quen với SQL hơn LINQ

**Không nên thay EF Core bằng Dapper khi:**
- Code CRUD thông thường
- Team chưa quen SQL
- Cần Change Tracking và Migration

### Dùng Cả Hai Cùng Nhau

```csharp
// Đăng ký cả 2
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IDbConnection>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new SqlConnection(config.GetConnectionString("Default"));
});

// Service dùng EF Core cho CRUD
public class ProductService
{
    private readonly AppDbContext _context; // EF Core
    private readonly IDbConnection _dapper; // Dapper

    public Task<Product?> GetByIdAsync(int id)
        => _context.Products.FindAsync(id).AsTask(); // EF Core

    public Task<IEnumerable<SalesReport>> GetReportAsync(...)
        => _dapper.QueryAsync<SalesReport>(sql, ...); // Dapper
}
```

---

## 9. Connection Pool — Tái Sử Dụng Kết Nối

```csharp
// Connection string với pool settings
"Server=...;Database=...;
 Max Pool Size=100;       // Tối đa 100 connections trong pool
 Min Pool Size=5;         // Luôn giữ ít nhất 5 connections mở
 Connection Lifetime=300; // Connections sống tối đa 300 giây"
```

ADO.NET Connection Pooling — Tái Sử Dụng Kết Nối hoạt động tự động. `DbContext.Dispose()` không đóng connection thật, chỉ trả về pool.

---

## 10. Async Streaming — Xử Lý Dữ liệu Lớn

```csharp
// ❌ Load toàn bộ vào RAM
var allProducts = await context.Products.ToListAsync(); // 1 triệu rows!

// ✅ Stream từng row — ít RAM hơn nhiều
await foreach (var product in context.Products
    .AsNoTracking()
    .AsAsyncEnumerable()) // IAsyncEnumerable<Product>
{
    await ProcessProductAsync(product);
    // Chỉ giữ 1 product trong RAM tại 1 thời điểm
}
```

---

## 📋 Performance Checklist

- [ ] Projection cho read-only APIs (không `SELECT *`)
- [ ] `AsNoTracking()` cho mọi query không cần update
- [ ] Index trên FK columns và cột hay dùng trong WHERE
- [ ] `AsSplitQuery()` khi Include nhiều collections
- [ ] `ExecuteUpdateAsync/DeleteAsync` cho bulk operations
- [ ] `DbContextPool` thay `AddDbContext` (high-throughput apps)
- [ ] Compiled Queries cho hot paths
- [ ] Dapper chỉ khi EF Core không đủ (reporting, complex analytics)

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Khi nào nên dùng Dapper thay EF Core?**
> Khi cần (1) queries rất phức tạp khó diễn đạt bằng LINQ (window functions, recursive CTEs), (2) performance là yêu cầu cực kỳ quan trọng (sub-millisecond cho millions rows), (3) gọi stored procedures trả về nhiều result sets. Không nên thay toàn bộ EF Core bằng Dapper — dùng cả 2 trong cùng project cho mục đích khác nhau.

**Q: `ExecuteUpdateAsync` vs load rồi update khác gì?**
> `ExecuteUpdateAsync` tạo 1 SQL UPDATE duy nhất, không load entities, không Change Tracker overhead — phù hợp cho bulk updates. Load rồi update cần 2 round-trips (SELECT + UPDATE), nhưng an toàn hơn với Optimistic Concurrency và rõ ràng hơn trong business logic.

**Q: Compiled Queries có đáng dùng không?**
> Có trong hot paths — EF Core mất khoảng 10-20% thời gian để translate LINQ sang SQL mỗi lần. Compiled Queries bỏ qua bước này. Đáng dùng khi query chạy hàng nghìn lần/giây. Với query thông thường, không đáng tốn effort.
