# 6 — Change Tracking: AsNoTracking, Attach & EntityState

> Change Tracking — Theo Dõi Thay Đổi — là cơ chế EF Core dùng để biết entity nào thay đổi và cần UPDATE khi gọi `SaveChanges()`. Hiểu nó giúp tối ưu performance đáng kể.

---

## 📌 Change Tracker Là Gì?

```
EF Core Load Entity  →  Change Tracker lưu snapshot  →  Bạn thay đổi entity
                                                                    ↓
SaveChanges() ←  Phát hiện thay đổi (DetectChanges)  ←  So sánh với snapshot
      ↓
  UPDATE SQL chỉ các cột đã thay đổi
```

Change Tracker tiêu tốn **bộ nhớ** và **CPU** (vì phải lưu snapshot và so sánh). Với read-only operations, đây là chi phí không cần thiết.

---

## 1. EntityState — Trạng Thái Thực Thể

Mỗi entity được EF Core theo dõi với một trong 5 trạng thái:

| State | Mô Tả | SaveChanges sẽ làm gì |
|-------|-------|----------------------|
| `Detached` | Không được theo dõi | Không làm gì |
| `Unchanged` | Load từ DB, chưa thay đổi | Không làm gì |
| `Added` | Mới thêm vào context | INSERT |
| `Modified` | Đã thay đổi sau khi load | UPDATE |
| `Deleted` | Đánh dấu xóa | DELETE |

```csharp
// Kiểm tra EntityState
var product = await context.Products.FindAsync(1);
var state = context.Entry(product).State; // EntityState.Unchanged

product.Name = "New Name";
state = context.Entry(product).State; // EntityState.Modified (tự động!)

context.Products.Remove(product);
state = context.Entry(product).State; // EntityState.Deleted
```

---

## 2. AsNoTracking — Tắt Change Tracking

Dùng khi **chỉ đọc** dữ liệu, không cần update:

```csharp
// ✅ Read-only query — không cần tracking
var products = await context.Products
    .AsNoTracking()     // Không lưu snapshot, không theo dõi
    .Where(p => p.IsActive)
    .ToListAsync();

// Hiệu quả:
// - Nhanh hơn ~30-40% so với tracked query
// - Ít bộ nhớ hơn (không lưu snapshot)
// - Garbage collect sớm hơn
```

### AsNoTrackingWithIdentityResolution (EF Core 5+)

```csharp
// AsNoTracking có vấn đề với Include — tạo duplicate objects
var orders = await context.Orders
    .AsNoTracking()
    .Include(o => o.Customer) // Nếu 10 orders có cùng customer → 10 Customer objects khác nhau!
    .ToListAsync();

// Giải pháp: AsNoTrackingWithIdentityResolution
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution() // Đảm bảo cùng customer → cùng object
    .Include(o => o.Customer)
    .ToListAsync();
```

### Tắt Tracking Cho Toàn Bộ Context

```csharp
// Nếu app mostly read-only, đặt default là NoTracking
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

// Bật lại tracking khi cần update
var product = await context.Products
    .AsTracking() // Ghi đè default
    .FirstOrDefaultAsync(p => p.Id == id);
product!.Name = "Updated";
await context.SaveChangesAsync();
```

---

## 3. Attach — Gắn Entity Vào Context

Dùng khi entity được tạo ở nơi khác (không qua EF query) nhưng cần update:

```csharp
// Scenario: Entity đến từ API request body (không qua EF query)
[HttpPut("{id}")]
public async Task<IActionResult> UpdateProduct(int id, ProductDto dto)
{
    var product = new Product
    {
        Id = id,           // Biết ID
        Name = dto.Name,
        Price = dto.Price,
        CategoryId = dto.CategoryId
    };

    // Cách 1: Attach rồi đánh dấu Modified
    context.Attach(product);
    context.Entry(product).State = EntityState.Modified;
    await context.SaveChangesAsync();
    // → UPDATE tất cả các cột (kể cả cột không thay đổi)

    // Cách 2: Attach rồi đánh dấu từng property
    context.Attach(product);
    context.Entry(product).Property(p => p.Name).IsModified = true;
    context.Entry(product).Property(p => p.Price).IsModified = true;
    await context.SaveChangesAsync();
    // → UPDATE chỉ Name và Price (tối ưu hơn)

    return NoContent();
}
```

### Update Đơn Giản Với context.Update()

```csharp
// context.Update() = Attach + MarkAllModified
context.Products.Update(product);
await context.SaveChangesAsync();
// → UPDATE tất cả cột (kể cả cột không đổi — ít tối ưu hơn)
```

---

## 4. DetectChanges — Phát Hiện Thay Đổi

EF Core gọi `DetectChanges()` tự động trước `SaveChanges()`:

```csharp
// DetectChanges so sánh state hiện tại với snapshot đã lưu
var product = await context.Products.FindAsync(1);
product.Name = "New Name";

// Khi SaveChanges được gọi:
// 1. DetectChanges() → phát hiện Name thay đổi
// 2. Tạo SQL: UPDATE Products SET Name = 'New Name' WHERE Id = 1

// Tắt AutoDetectChanges (cho bulk operations)
context.ChangeTracker.AutoDetectChangesEnabled = false;
// ... thực hiện nhiều thay đổi ...
context.ChangeTracker.DetectChanges(); // Gọi thủ công khi cần
await context.SaveChangesAsync();
context.ChangeTracker.AutoDetectChangesEnabled = true;
```

---

## 5. Tracking Trong Update Patterns

### Pattern 1: Load Rồi Update (An Toàn Nhất)

```csharp
public async Task<bool> UpdateProductAsync(int id, UpdateProductRequest request)
{
    var product = await context.Products.FindAsync(id); // Load vào tracker
    if (product is null) return false;

    // Chỉ update những gì thay đổi
    product.Name = request.Name;
    product.Price = request.Price;

    await context.SaveChangesAsync(); // UPDATE chỉ Name và Price
    return true;
}
```

**Ưu điểm:** An toàn với concurrency, chỉ update cột cần thiết.
**Nhược điểm:** 2 round-trips đến DB (SELECT + UPDATE).

### Pattern 2: Disconnected Update (Hiệu Năng Cao Hơn)

```csharp
public async Task<bool> UpdateProductAsync(int id, UpdateProductRequest request)
{
    var exists = await context.Products.AnyAsync(p => p.Id == id);
    if (!exists) return false;

    // Tạo entity với dữ liệu mới, gắn vào context
    var product = new Product
    {
        Id = id,
        Name = request.Name,
        Price = request.Price
    };

    context.Attach(product);
    context.Entry(product).Property(p => p.Name).IsModified = true;
    context.Entry(product).Property(p => p.Price).IsModified = true;
    await context.SaveChangesAsync();
    return true;
}
// 2 queries: AnyAsync (SELECT 1) + UPDATE (chỉ 2 cột)
```

### Pattern 3: ExecuteUpdate (EF Core 7+ — Nhanh Nhất)

```csharp
// Bulk update — không load entity, không Change Tracker, 1 SQL duy nhất
var updated = await context.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));
// → UPDATE Products SET IsActive = 0, UpdatedAt = ... WHERE CategoryId = @categoryId
```

### Pattern 4: ExecuteDelete (EF Core 7+ — Nhanh Nhất)

```csharp
// Bulk delete — không load entities
var deleted = await context.Products
    .Where(p => !p.IsActive && p.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();
// → DELETE FROM Products WHERE IsActive = 0 AND CreatedAt < ...
```

---

## 6. Change Tracker API

```csharp
// Xem tất cả entities đang được theo dõi
foreach (var entry in context.ChangeTracker.Entries())
{
    Console.WriteLine($"{entry.Entity.GetType().Name}: {entry.State}");
}

// Xem entities theo state cụ thể
var modifiedEntities = context.ChangeTracker
    .Entries<Product>()
    .Where(e => e.State == EntityState.Modified)
    .Select(e => e.Entity)
    .ToList();

// Clear toàn bộ tracking (sau bulk import)
context.ChangeTracker.Clear();
```

### Tự Động Set Timestamps

```csharp
// Override SaveChangesAsync để tự động cập nhật CreatedAt/UpdatedAt
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var now = DateTime.UtcNow;

    foreach (var entry in ChangeTracker.Entries<BaseEntity>())
    {
        switch (entry.State)
        {
            case EntityState.Added:
                entry.Entity.CreatedAt = now;
                entry.Entity.UpdatedAt = now;
                break;
            case EntityState.Modified:
                entry.Entity.UpdatedAt = now;
                // Đảm bảo CreatedAt không bị thay đổi
                entry.Property(e => e.CreatedAt).IsModified = false;
                break;
        }
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```

---

## 7. Concurrency Conflict Handling — Xử Lý Xung Đột Đồng Thời

### Optimistic Concurrency — Kiểm Soát Lạc Quan

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // RowVersion — EF tự động thêm WHERE RowVersion = @original vào UPDATE
    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
}
```

```csharp
// Xử lý DbUpdateConcurrencyException
public async Task<bool> UpdateProductAsync(int id, string newName)
{
    try
    {
        var product = await context.Products.FindAsync(id);
        if (product is null) return false;

        product.Name = newName;
        await context.SaveChangesAsync();
        return true;
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Có người khác cũng update cùng lúc
        var entry = ex.Entries.Single();
        var dbValues = await entry.GetDatabaseValuesAsync();

        if (dbValues is null)
        {
            // Entity đã bị xóa bởi người khác
            throw new Exception("Product was deleted by another user.");
        }

        // Refresh với giá trị mới nhất từ DB
        entry.OriginalValues.SetValues(dbValues);

        // Retry
        await context.SaveChangesAsync();
        return true;
    }
}
```

---

## 8. Memory Leak Với Long-Lived DbContext

```csharp
// ❌ NGUY HIỂM — DbContext tích lũy entities theo thời gian
public class BadReportService
{
    private readonly AppDbContext _context; // Singleton context!

    public async Task GenerateReportAsync()
    {
        for (int i = 0; i < 10000; i++)
        {
            var product = await _context.Products.FindAsync(i);
            // Context giữ tham chiếu đến 10,000 entities → memory leak!
        }
    }
}

// ✅ Đúng — Clear tracking sau khi xử lý batch
public async Task GenerateReportAsync()
{
    var ids = Enumerable.Range(1, 10000);
    foreach (var batch in ids.Chunk(100))
    {
        var products = await _context.Products
            .Where(p => batch.Contains(p.Id))
            .AsNoTracking() // Không track để tiết kiệm bộ nhớ
            .ToListAsync();

        // Xử lý batch...

        _context.ChangeTracker.Clear(); // Giải phóng bộ nhớ
    }
}
```

---

## 📋 Checklist Change Tracking

- [ ] Dùng `AsNoTracking()` cho mọi read-only query
- [ ] Đặt `QueryTrackingBehavior.NoTracking` nếu app thiên về đọc
- [ ] Dùng `ExecuteUpdateAsync/ExecuteDeleteAsync` cho bulk operations
- [ ] Override `SaveChangesAsync` để tự động set audit fields
- [ ] Clear Change Tracker sau bulk import để tránh memory leak
- [ ] Hiểu khi nào cần `Attach` vs load entity từ DB

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: `AsNoTracking()` dùng khi nào? Lợi ích gì?**
> Dùng cho mọi query chỉ đọc dữ liệu (không update). Lợi ích: nhanh hơn 30-40%, ít bộ nhớ hơn (không lưu snapshot), object được GC sớm hơn. Không dùng được khi cần update entity sau khi load.

**Q: Khi nhận entity từ API request, làm sao update hiệu quả?**
> 3 cách: (1) Load từ DB rồi cập nhật — an toàn nhất, (2) Attach + đánh dấu property IsModified — tránh 1 round-trip, (3) `ExecuteUpdateAsync()` (EF Core 7+) — nhanh nhất cho bulk, không cần load entity. Chọn tùy ngữ cảnh.

**Q: Optimistic Concurrency là gì trong EF Core?**
> Dùng `[Timestamp]` hoặc `[ConcurrencyCheck]` để thêm điều kiện kiểm tra vào SQL UPDATE. Nếu record bị người khác thay đổi trong lúc bạn đang xử lý, EF ném `DbUpdateConcurrencyException`. Phù hợp hơn Pessimistic Locking cho hầu hết web apps.
