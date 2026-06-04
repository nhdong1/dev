# 5 — N+1 Query Problem: Phát Hiện & Giải Quyết

> N+1 Query Problem — Vấn Đề N+1 Truy Vấn — là một trong những lỗi hiệu năng phổ biến nhất trong EF Core. Hiểu rõ và biết cách phát hiện là kỹ năng bắt buộc.

---

## 📌 N+1 Là Gì?

Thay vì 1 query lấy tất cả dữ liệu, code vô tình tạo ra **N+1 queries**:

```
1 query để lấy danh sách Orders (N = 10 orders)
+ 10 queries riêng để lấy Customer của từng Order
= 11 queries tổng cộng (1 + N = N+1)
```

Với N=1000 orders: **1001 round-trips đến database!**

---

## 1. Tái Hiện N+1 Problem

### Ví Dụ Kinh Điển

```csharp
// ❌ CODE GÂY N+1 PROBLEM
public async Task<List<OrderSummary>> GetOrderSummariesAsync()
{
    // Query 1: Lấy tất cả orders
    var orders = await context.Orders.ToListAsync();

    var summaries = new List<OrderSummary>();
    foreach (var order in orders) // Giả sử 100 orders
    {
        // Query 2, 3, 4... 101: Mỗi lần truy cập .Customer gây 1 query mới!
        summaries.Add(new OrderSummary
        {
            OrderId = order.Id,
            CustomerName = order.Customer.Name,  // ← N+1 xảy ra ở đây
            ItemCount = order.Items.Count         // ← Và ở đây nữa!
        });
    }
    return summaries;
}
// Kết quả: 1 + 100 + 100 = 201 queries! 💥
```

### Tại Sao Lại Xảy Ra?

EF Core dùng **Lazy Loading** (tải lười biếng) hoặc **Explicit Loading** không đúng cách:

```csharp
// Khi truy cập navigation property không được load:
order.Customer  // ← EF Core gửi SELECT * FROM Customers WHERE Id = @orderId
order.Items     // ← EF Core gửi SELECT * FROM OrderItems WHERE OrderId = @orderId
```

---

## 2. Phát Hiện N+1 Problem

### Cách 1: Bật SQL Logging

```csharp
// Trong DbContext configuration
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging());

// Xem console output — nếu thấy cùng 1 loại query lặp lại nhiều lần:
// [EF Core] Executed DbCommand: SELECT * FROM Customers WHERE Id = 1
// [EF Core] Executed DbCommand: SELECT * FROM Customers WHERE Id = 2
// [EF Core] Executed DbCommand: SELECT * FROM Customers WHERE Id = 3
// ... → N+1!
```

### Cách 2: MiniProfiler

```bash
dotnet add package MiniProfiler.AspNetCore.Mvc
dotnet add package MiniProfiler.EntityFrameworkCore
```

```csharp
// Program.cs
builder.Services.AddMiniProfiler(options =>
{
    options.RouteBasePath = "/profiler";
}).AddEntityFramework();

// appsettings.Development.json — xem profiler tại /profiler/results-index
```

### Cách 3: EF Core Query Warnings

```csharp
// Cấu hình để throw exception khi có N+1 tiềm năng
options.ConfigureWarnings(w =>
    w.Throw(RelationalEventId.MultipleCollectionIncludeWarning));
```

### Cách 4: Công Cụ Chuyên Dụng

- **Glimpse** — Dashboard cho ASP.NET
- **SQL Server Profiler** — Bắt tất cả queries đến SQL Server
- **Datadog APM** — Theo dõi query trong production
- **Application Insights** — Azure monitoring, thấy số queries/request

---

## 3. Giải Pháp 1: Eager Loading Với Include

```csharp
// ✅ GIẢI PHÁP — Dùng Include để load trước
public async Task<List<OrderSummary>> GetOrderSummariesAsync()
{
    var orders = await context.Orders
        .Include(o => o.Customer)     // Load Customer trong cùng query
        .Include(o => o.Items)        // Load Items trong cùng query
        .ToListAsync();               // 1 query (hoặc split queries)

    return orders.Select(o => new OrderSummary
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,  // ← Không gây thêm query
        ItemCount = o.Items.Count        // ← Không gây thêm query
    }).ToList();
}
// Kết quả: 1 query duy nhất ✅
```

---

## 4. Giải Pháp 2: Projection (Tốt Nhất)

```csharp
// ✅ TỐT NHẤT — Chỉ lấy đúng dữ liệu cần, EF Core tự JOIN
public async Task<List<OrderSummary>> GetOrderSummariesAsync()
{
    return await context.Orders
        .Select(o => new OrderSummary
        {
            OrderId = o.Id,
            CustomerName = o.Customer.Name,  // EF Core tự JOIN Customer
            ItemCount = o.Items.Count()       // EF Core dùng COUNT(*)
        })
        .ToListAsync();
}
```

SQL được tạo:
```sql
SELECT o.Id, c.Name, (SELECT COUNT(*) FROM OrderItems WHERE OrderId = o.Id)
FROM Orders o
LEFT JOIN Customers c ON o.CustomerId = c.Id
```

**Tại sao Projection tốt hơn Include?**
- Không load dữ liệu thừa (không SELECT * FROM Orders)
- Không kích hoạt Change Tracker
- SQL nhỏ hơn, nhanh hơn

---

## 5. Giải Pháp 3: Split Queries

Khi cần load nhiều collections, `AsSplitQuery()` tránh cartesian explosion:

```csharp
// ✅ Split query — chạy nhiều queries nhưng không có N+1
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .AsSplitQuery()    // Tách thành 3 queries riêng biệt
    .ToListAsync();

// EF Core tự động:
// Query 1: SELECT * FROM Orders
// Query 2: SELECT * FROM OrderItems WHERE OrderId IN (1,2,3,...)
// Query 3: SELECT * FROM OrderTags WHERE OrderId IN (1,2,3,...)
// Sau đó ghép kết quả ở phía client
```

---

## 6. Giải Pháp 4: Explicit Batching

Khi cần linh hoạt hơn, load dữ liệu theo batch:

```csharp
// Load entities trước
var orders = await context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync();

var orderIds = orders.Select(o => o.Id).ToList();

// Load related entities theo batch — 1 query duy nhất
var items = await context.OrderItems
    .Where(i => orderIds.Contains(i.OrderId))
    .ToListAsync();

// Ghép dữ liệu ở phía .NET
var itemsByOrder = items.GroupBy(i => i.OrderId)
                        .ToDictionary(g => g.Key, g => g.ToList());

foreach (var order in orders)
{
    if (itemsByOrder.TryGetValue(order.Id, out var orderItems))
    {
        // Dùng orderItems...
    }
}
```

---

## 7. Lazy Loading — Tải Lười Biếng (Nguyên Nhân Thường Gặp)

Lazy Loading tự động gây N+1 nếu dùng không cẩn thận:

```bash
# Cài package
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

```csharp
// Bật Lazy Loading
options.UseLazyLoadingProxies();

// Tất cả navigation properties phải là virtual
public class Order
{
    public int Id { get; set; }
    public virtual Customer Customer { get; set; } = null!; // virtual!
    public virtual ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}
```

```csharp
// ⚠️ VẤN ĐỀ: Với Lazy Loading, truy cập navigation property gây query
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // ← Query DB mỗi lần!
}
// → N+1 problem tự động!
```

**Khuyến nghị:** Tắt Lazy Loading, dùng Eager Loading (Include) hoặc Projection một cách có chủ đích.

---

## 8. Select N+1 Trong LINQ Lồng Nhau

```csharp
// ❌ N+1 ẩn trong projection
var result = await context.Categories
    .Select(c => new
    {
        CategoryName = c.Name,
        // Subquery này được tính riêng cho MỖI category!
        ExpensiveProductCount = context.Products
            .Count(p => p.CategoryId == c.Id && p.Price > 100)
    })
    .ToListAsync();
// Kết quả: 1 query Categories + N queries COUNT → N+1!

// ✅ Giải pháp — dùng navigation property thay subquery thủ công
var result = await context.Categories
    .Select(c => new
    {
        CategoryName = c.Name,
        // EF Core tối ưu thành subquery trong cùng 1 SQL
        ExpensiveProductCount = c.Products.Count(p => p.Price > 100)
    })
    .ToListAsync();
```

---

## 9. Benchmark So Sánh

```
Kịch bản: 100 Orders, mỗi order có 5 items, 1 customer

❌ N+1 (không Include):
   - Số queries: 201 (1 + 100 orders × 2 navigation props)
   - Thời gian: ~500ms (mỗi round-trip ~2.5ms)
   - Network overhead: Cao

✅ Include():
   - Số queries: 1 (JOIN)
   - Thời gian: ~20ms
   - Network overhead: Thấp

✅ Projection với Select():
   - Số queries: 1 (JOIN + aggregation)
   - Thời gian: ~15ms
   - Network overhead: Thấp nhất (ít cột nhất)
```

---

## 10. Repository Pattern Ẩn N+1

Cẩn thận với Repository Pattern che giấu vấn đề:

```csharp
// ❌ Repository trả về IEnumerable đã materialized — mất đi lazy Include
public class OrderRepository
{
    public IEnumerable<Order> GetAll() // Đã ToList() bên trong!
        => _context.Orders.ToList();   // Mất đi lazy evaluation
}

// Khi dùng:
var orders = _repo.GetAll(); // Chỉ load Orders
foreach (var o in orders)
    Console.WriteLine(o.Customer.Name); // N+1 với Lazy Loading!

// ✅ Tốt hơn — trả về IQueryable hoặc nhận specification
public IQueryable<Order> GetAll()
    => _context.Orders; // Caller tự thêm Include, Where, ...

// Hoặc dùng Specification Pattern
public Task<List<Order>> GetAllWithCustomersAsync()
    => _context.Orders.Include(o => o.Customer).ToListAsync();
```

---

## 📋 Checklist N+1 Prevention

- [ ] Luôn log SQL queries trong môi trường development
- [ ] Review mọi `Include()` — đảm bảo load đúng những gì cần
- [ ] Ưu tiên dùng Projection (`Select()`) thay vì `Include()` khi có thể
- [ ] Không bật Lazy Loading trong production (hoặc dùng rất cẩn thận)
- [ ] Dùng `AsSplitQuery()` khi Include nhiều collections
- [ ] Thêm integration test kiểm tra số lượng queries

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: N+1 Query Problem là gì? Cho ví dụ cụ thể.**
> N+1 xảy ra khi code load 1 danh sách (1 query), rồi với mỗi item lại load thêm related data (N queries). Ví dụ: load 100 Orders xong, trong vòng lặp truy cập `order.Customer` → 100 queries thêm = 101 tổng. Fix bằng `Include(o => o.Customer)` hoặc Projection.

**Q: Nên dùng Include hay Projection?**
> Projection (`Select()`) tốt hơn trong hầu hết trường hợp: ít data truyền từ DB, không kích hoạt Change Tracker, query nhanh hơn. `Include()` dùng khi cần trả về full entity với related data để tiếp tục thao tác (save changes, navigate...).

**Q: Làm sao phát hiện N+1 trong code production?**
> Dùng APM tools (Application Performance Monitoring) như Application Insights, Datadog — chúng hiển thị số lượng SQL queries mỗi request. Nếu thấy số queries tỉ lệ thuận với data rows thay vì cố định → dấu hiệu N+1. Trong development, bật `LogTo(Console.WriteLine)`.
