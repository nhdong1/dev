# 4 — Querying: LINQ, Include, ThenInclude & Projection

> EF Core dịch LINQ queries sang SQL. Hiểu cách dịch này giúp bạn viết queries hiệu quả thay vì vô tình gây ra queries chậm hoặc N+1 problem.

---

## 📌 Tổng Quan

```
LINQ Query  →  EF Core LINQ Provider  →  SQL Query  →  Database
    ↑                                                        ↓
C# Objects  ←  Materialization (vật chất hóa)  ←  ResultSet
```

---

## 1. Deferred vs Immediate Execution — Thực Thi Trì Hoãn vs Ngay Lập Tức

### Deferred Execution — Trì Hoãn

Query chỉ được gửi đến database khi **thực sự cần dữ liệu**:

```csharp
// Chưa có SQL nào được gửi
IQueryable<Product> query = context.Products
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name);

// Thêm điều kiện tiếp theo — vẫn chưa gửi SQL
if (minPrice.HasValue)
    query = query.Where(p => p.Price >= minPrice.Value);

// ← SQL được gửi tại đây (ToListAsync)
var products = await query.ToListAsync();
```

SQL được tạo ra chứa đầy đủ tất cả điều kiện:
```sql
SELECT * FROM Products WHERE IsActive = 1 AND Price >= 50 ORDER BY Name
```

### Immediate Execution — Ngay Lập Tức

```csharp
// Các operator này kích hoạt query ngay:
var count = await context.Products.CountAsync();
var first = await context.Products.FirstOrDefaultAsync(p => p.Id == id);
var exists = await context.Products.AnyAsync(p => p.Price > 1000);
var list = await context.Products.ToListAsync();
var array = await context.Products.ToArrayAsync();
```

---

## 2. Basic Queries — Queries Cơ Bản

```csharp
// Lấy tất cả
var all = await context.Products.ToListAsync();

// Lọc (WHERE)
var active = await context.Products
    .Where(p => p.IsActive && p.Price > 0)
    .ToListAsync();

// Sắp xếp (ORDER BY)
var sorted = await context.Products
    .OrderBy(p => p.CategoryId)
    .ThenBy(p => p.Name) // Sắp xếp phụ khi CategoryId bằng nhau
    .ToListAsync();

// Phân trang (OFFSET/FETCH)
var page2 = await context.Products
    .OrderBy(p => p.Id)
    .Skip(10)      // Bỏ qua 10 bản ghi đầu
    .Take(10)      // Lấy 10 bản ghi tiếp theo
    .ToListAsync();

// Tìm theo Primary Key — dùng FindAsync (có cache trong Change Tracker)
var product = await context.Products.FindAsync(id);

// FirstOrDefault — trả về null nếu không tìm thấy
var product = await context.Products
    .FirstOrDefaultAsync(p => p.Id == id);

// Single — throw exception nếu không đúng 1 kết quả
var product = await context.Products
    .SingleOrDefaultAsync(p => p.Slug == slug);
```

---

## 3. Include & ThenInclude — Eager Loading (Tải Sớm)

**Eager Loading** — tải related entities trong cùng 1 query (dùng JOIN):

```csharp
// Include 1 cấp — Load Category cùng với Products
var products = await context.Products
    .Include(p => p.Category)
    .ToListAsync();
```

SQL được tạo:
```sql
SELECT p.*, c.*
FROM Products p
LEFT JOIN Categories c ON p.CategoryId = c.Id
```

### ThenInclude — Include Lồng Nhau

```csharp
// Include 2 cấp — Order → Items → Product
var orders = await context.Orders
    .Include(o => o.Items)
        .ThenInclude(item => item.Product)
    .Include(o => o.Customer)  // Include thêm 1 nhánh khác
    .ToListAsync();
```

```csharp
// Include nhiều cấp hơn
var orders = await context.Orders
    .Include(o => o.Items)
        .ThenInclude(item => item.Product)
            .ThenInclude(product => product.Category)
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();
```

### Include Với Filter (EF Core 5+)

```csharp
// Chỉ load OrderItems có Quantity > 0
var orders = await context.Orders
    .Include(o => o.Items.Where(i => i.Quantity > 0))
    .ToListAsync();
```

---

## 4. Projection — Chọn Chỉ Những Gì Cần

Thay vì load toàn bộ entity, chỉ SELECT những cột cần thiết:

```csharp
// ❌ Kém hiệu quả — load toàn bộ entity kể cả các cột không dùng
var products = await context.Products.ToListAsync();
var names = products.Select(p => p.Name).ToList();

// ✅ Tốt hơn — chỉ SELECT Name từ database
var names = await context.Products
    .Select(p => p.Name)
    .ToListAsync();

// ✅ Project vào DTO — Data Transfer Object
var dtos = await context.Products
    .Where(p => p.IsActive)
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name // EF Core tự JOIN để lấy CategoryName
    })
    .ToListAsync();
```

SQL được tạo cho DTO query:
```sql
SELECT p.Id, p.Name, p.Price, c.Name AS CategoryName
FROM Products p
LEFT JOIN Categories c ON p.CategoryId = c.Id
WHERE p.IsActive = 1
```

### Anonymous Type Projection

```csharp
var result = await context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        Count = g.Count(),
        AveragePrice = g.Average(p => p.Price),
        MaxPrice = g.Max(p => p.Price)
    })
    .ToListAsync();
```

---

## 5. Aggregation — Tổng Hợp

```csharp
var count = await context.Products.CountAsync();
var countActive = await context.Products.CountAsync(p => p.IsActive);
var totalValue = await context.Products.SumAsync(p => p.Price * p.Stock);
var avgPrice = await context.Products.AverageAsync(p => p.Price);
var maxPrice = await context.Products.MaxAsync(p => p.Price);
var minPrice = await context.Products.MinAsync(p => p.Price);
var anyExpensive = await context.Products.AnyAsync(p => p.Price > 10000);
var allActive = await context.Products.AllAsync(p => p.IsActive);
```

---

## 6. GroupBy — Nhóm Dữ Liệu

```csharp
// Đếm sản phẩm theo danh mục
var categoryCounts = await context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        ProductCount = g.Count()
    })
    .ToListAsync();

// Tổng doanh thu theo tháng
var monthlySales = await context.Orders
    .Where(o => o.Status == OrderStatus.Completed)
    .GroupBy(o => new { o.CreatedAt.Year, o.CreatedAt.Month })
    .Select(g => new
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        Total = g.Sum(o => o.TotalAmount)
    })
    .OrderBy(x => x.Year).ThenBy(x => x.Month)
    .ToListAsync();
```

---

## 7. Join — Kết Hợp Bảng Không Có Navigation Property

Khi không có navigation property, hoặc cần join phức tạp:

```csharp
// LINQ Join syntax
var result = await (
    from order in context.Orders
    join customer in context.Customers on order.CustomerId equals customer.Id
    join item in context.OrderItems on order.Id equals item.OrderId
    where order.Status == OrderStatus.Pending
    select new
    {
        OrderId = order.Id,
        CustomerName = customer.Name,
        ItemCount = context.OrderItems.Count(i => i.OrderId == order.Id)
    }
).ToListAsync();
```

---

## 8. Raw SQL Queries — Query SQL Thô

Khi LINQ không đủ mạnh hoặc cần tối ưu đặc biệt:

```csharp
// FromSqlRaw — trả về entities
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0}", minPrice)
    .Where(p => p.IsActive)  // Có thể kết hợp thêm LINQ
    .ToListAsync();

// FromSqlInterpolated — an toàn hơn với SQL Injection prevention
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .ToListAsync();

// ExecuteSqlRaw — cho DELETE/UPDATE, không trả về entity
var affected = await context.Database
    .ExecuteSqlRawAsync("DELETE FROM Products WHERE IsActive = 0");
```

---

## 9. Split Queries — Tách Queries (EF Core 5+)

Khi `Include` nhiều collections, EF Core có thể tạo ra cartesian explosion — bùng nổ tích đề-các:

```csharp
// ❌ Vấn đề — 1 query với cartesian explosion
// Order có 100 Items và 50 Tags → kết quả 100x50=5000 dòng
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .ToListAsync();

// ✅ Giải pháp — Split thành nhiều queries
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .AsSplitQuery() // Tạo 3 queries riêng biệt, join ở client
    .ToListAsync();
```

---

## 10. Compiled Queries — Query Đã Biên Dịch

Tối ưu cho queries chạy rất thường xuyên:

```csharp
public class ProductQueries
{
    // Static compiled query — chỉ compile LINQ → SQL 1 lần
    private static readonly Func<AppDbContext, int, Task<Product?>> GetByIdQuery =
        EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
            ctx.Products
               .Include(p => p.Category)
               .FirstOrDefault(p => p.Id == id));

    public async Task<Product?> GetByIdAsync(AppDbContext context, int id)
        => await GetByIdQuery(context, id);
}
```

**Lợi ích:** Bỏ qua bước translate LINQ → SQL expression tree mỗi lần gọi. Quan trọng khi query được gọi hàng nghìn lần/giây.

---

## 11. Query Filters — Bộ Lọc Query Toàn Cục

Áp dụng điều kiện WHERE tự động cho mọi query trên entity:

```csharp
// Soft Delete — Xóa mềm (không xóa thật, chỉ đánh dấu IsDeleted)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Tự động thêm WHERE IsDeleted = 0 cho mọi query Product
    modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);

    // Multi-tenancy — Đa người thuê (mỗi tenant chỉ thấy dữ liệu của mình)
    modelBuilder.Entity<Product>().HasQueryFilter(p => p.TenantId == _currentTenantId);
}

// Bỏ qua filter khi cần (VD: admin xem tất cả)
var allIncludingDeleted = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();
```

---

## 12. Async vs Sync Queries

```csharp
// ✅ Luôn dùng async trong web applications
var products = await context.Products.ToListAsync();
var count = await context.Products.CountAsync();

// ❌ Tránh sync trong async context — có thể gây deadlock
var products = context.Products.ToList(); // Blocking call
```

---

## 📋 Checklist Querying

- [ ] Dùng `Select()` projection thay vì load toàn bộ entity khi chỉ cần vài cột
- [ ] Dùng `Include()` đúng cách — không include những gì không cần
- [ ] Xem xét `AsSplitQuery()` khi có nhiều collection includes
- [ ] Dùng `FindAsync()` thay `FirstOrDefaultAsync()` khi tìm theo PK
- [ ] Luôn dùng `async`/`await` (tránh sync queries trong web)
- [ ] Dùng `AnyAsync()` thay `Count() > 0` để kiểm tra tồn tại
- [ ] Cân nhắc Compiled Queries cho hot paths (đường dẫn thường xuyên)

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: `Include()` có phải luôn tốt không?**
> Không. `Include()` thêm JOIN vào SQL query — nếu join nhiều bảng hoặc collection lớn, có thể chậm hơn chạy nhiều queries riêng. Trong trường hợp đó, dùng `AsSplitQuery()` hoặc load riêng. Luôn measure trước khi tối ưu.

**Q: Projection vs Include — khi nào dùng cái nào?**
> `Include` khi cần trả về entity đầy đủ với related data. `Projection` (Select) khi chỉ cần một số cột để hiển thị — tốt hơn về performance vì ít data truyền từ DB và không kích hoạt Change Tracker.

**Q: `FindAsync()` vs `FirstOrDefaultAsync()` khác gì?**
> `FindAsync()` kiểm tra Change Tracker trước — nếu entity đã được load trong session thì trả về từ cache không cần query DB. `FirstOrDefaultAsync()` luôn gửi query đến DB. Dùng `FindAsync()` khi tìm theo PK để tận dụng cache.
