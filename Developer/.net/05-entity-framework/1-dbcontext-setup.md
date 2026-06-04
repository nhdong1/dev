# 1 — DbContext: Cài Đặt, Vòng Đời & Pooling

> `DbContext` là trung tâm của EF Core — nó đại diện cho một session với database. Hiểu vòng đời và cách cấu hình đúng là nền tảng để tránh bug khó chịu.

---

## 📌 Tổng Quan

```
DbContext
├── Quản lý connection đến database
├── Theo dõi thay đổi entities (Change Tracker)
├── Cung cấp DbSet<T> cho từng bảng
├── Thực thi queries và SaveChanges
└── Lưu trữ model metadata (schema cache)
```

**Quy tắc quan trọng nhất:** Mỗi `DbContext` instance **không thread-safe** — không dùng chung giữa nhiều thread.

---

## 1. Khai Báo DbContext Cơ Bản

```csharp
public class AppDbContext : DbContext
{
    // Constructor nhận DbContextOptions — được inject từ DI container
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    // DbSet<T> — Tập thực thể ánh xạ với bảng trong database
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Áp dụng tất cả IEntityTypeConfiguration trong assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Tách Cấu Hình Ra File Riêng (Best Practice)

Thay vì viết tất cả trong `OnModelCreating`, dùng `IEntityTypeConfiguration<T>`:

```csharp
// ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(p => p.Price)
            .HasColumnType("decimal(18,2)");

        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // Index — Chỉ mục để tối ưu query
        builder.HasIndex(p => p.CategoryId);
        builder.HasIndex(p => p.Name);
    }
}
```

---

## 2. Đăng Ký DbContext Trong DI

### SQL Server

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sqlOptions =>
        {
            // Retry tự động khi kết nối tạm thời thất bại (transient failures)
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
        }));
```

### PostgreSQL (Npgsql)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

### SQLite (thường dùng trong tests)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));
```

### Connection String Trong `appsettings.json`

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;User Id=sa;Password=YourPassword;TrustServerCertificate=True"
  }
}
```

---

## 3. Vòng Đời — Lifetime của DbContext

DbContext mặc định được đăng ký với lifetime **`Scoped`** — nghĩa là:

```
HTTP Request bắt đầu
    └── DI tạo 1 DbContext instance
        ├── Service A dùng DbContext này
        ├── Service B dùng cùng DbContext này
        └── SaveChanges() lưu toàn bộ thay đổi
HTTP Request kết thúc
    └── DbContext bị dispose
```

### Vì Sao Scoped Là Đúng?

```csharp
// Giả sử có 2 services trong cùng 1 request
public class OrderService
{
    private readonly AppDbContext _context;
    public OrderService(AppDbContext context) => _context = context;

    public async Task CreateOrderAsync(Order order)
    {
        _context.Orders.Add(order);
        // Chưa SaveChanges ở đây
    }
}

public class InventoryService
{
    private readonly AppDbContext _context;
    public InventoryService(AppDbContext context) => _context = context;

    public async Task ReduceStockAsync(int productId, int quantity)
    {
        var product = await _context.Products.FindAsync(productId);
        product!.Stock -= quantity;
        // Chưa SaveChanges ở đây
    }
}

// Controller dùng cả 2 services
public class CheckoutController : ControllerBase
{
    public async Task<IActionResult> Checkout(
        OrderService orderService,
        InventoryService inventoryService)
    {
        await orderService.CreateOrderAsync(order);
        await inventoryService.ReduceStockAsync(productId, qty);
        // Cả 2 services dùng CÙNG 1 DbContext (vì Scoped)
        // 1 lần SaveChanges lưu toàn bộ — ATOMIC!
        await _context.SaveChangesAsync();
        return Ok();
    }
}
```

### Nguy Hiểm Khi Dùng Singleton DbContext

```csharp
// ❌ SAI — KHÔNG BAO GIỜ làm thế này
builder.Services.AddSingleton<AppDbContext>(); // BUG!

// Vấn đề:
// 1. Nhiều requests chia sẻ cùng DbContext → không thread-safe
// 2. Change Tracker tích lũy entities theo thời gian → memory leak
// 3. Connection không được trả về pool đúng cách
```

---

## 4. DbContext Pooling — Tái Sử Dụng Hiệu Quả

### Vấn Đề Với `AddDbContext` Thông Thường

Mỗi request tạo một `DbContext` mới:
- **Chi phí khởi tạo:** Load metadata model (schema), tạo object
- **Chi phí GC:** Garbage collect sau mỗi request

### Giải Pháp: `AddDbContextPool`

```csharp
// Program.cs — Dùng DbContext Pool
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")),
    poolSize: 128); // Số lượng context trong pool (mặc định: 1024)
```

### Cách Hoạt Động

```
Request 1 bắt đầu  → Lấy DbContext từ pool (hoặc tạo mới nếu pool trống)
Request 1 kết thúc → Trả DbContext về pool (Reset state)

Request 2 bắt đầu  → Lấy DbContext đã được reset từ pool (nhanh hơn tạo mới)
```

### Hạn Chế Của Pool

```csharp
// ❌ Không thể thêm constructor injection vào DbContext khi dùng Pool
public class AppDbContext : DbContext
{
    // KHÔNG thể inject service khác ở đây nếu dùng AddDbContextPool
    // private readonly IUserService _userService; // KHÔNG ĐƯỢC
}

// ✅ Thay thế: Dùng IDbContextFactory<T>
services.AddDbContextFactory<AppDbContext>(options => ...);

// Rồi inject factory
public class SomeService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public SomeService(IDbContextFactory<AppDbContext> factory)
        => _factory = factory;

    public async Task DoWorkAsync()
    {
        await using var context = await _factory.CreateDbContextAsync();
        // Dùng context...
    }
}
```

---

## 5. IDbContextFactory — Tạo Context Theo Yêu Cầu

Dùng khi cần tạo `DbContext` ngoài phạm vi của DI request scope:

```csharp
// Đăng ký
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Dùng trong Background Service (long-running, không có HTTP request scope)
public class DataSyncService : BackgroundService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public DataSyncService(IDbContextFactory<AppDbContext> factory)
        => _factory = factory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Tạo context mới cho mỗi lần sync
            await using var context = await _factory.CreateDbContextAsync(stoppingToken);
            await SyncDataAsync(context);
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

---

## 6. Cấu Hình Nâng Cao

### Logging SQL Queries (Debug)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information) // Log ra console
        .EnableSensitiveDataLogging()    // Hiển thị giá trị tham số (chỉ dev!)
        .EnableDetailedErrors());        // Lỗi chi tiết hơn (chỉ dev!)
```

### Dùng Với ILogger (Production)

```csharp
builder.Services.AddDbContext<AppDbContext>((serviceProvider, options) =>
{
    var loggerFactory = serviceProvider.GetRequiredService<ILoggerFactory>();
    options
        .UseSqlServer(connectionString)
        .UseLoggerFactory(loggerFactory);
});
```

### Query Tracking Behavior Mặc Định

```csharp
// Tắt Change Tracking mặc định cho toàn bộ context (tối ưu read-heavy apps)
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
// Sau đó bật lại khi cần update: context.Products.AsTracking()...
```

---

## 7. Intercept Database Operations — Chặn Thao Tác DB

`DbCommandInterceptor` — bắt mọi câu lệnh SQL trước/sau khi thực thi:

```csharp
public class QueryLoggingInterceptor : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Duration > TimeSpan.FromMilliseconds(500))
        {
            // Cảnh báo slow query (truy vấn chậm)
            Console.WriteLine($"[SLOW QUERY] {eventData.Duration.TotalMs}ms:\n{command.CommandText}");
        }
        return base.ReaderExecutedAsync(command, eventData, result, cancellationToken);
    }
}

// Đăng ký interceptor
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(connectionString)
        .AddInterceptors(new QueryLoggingInterceptor()));
```

---

## 8. SaveChanges Và Transaction

### SaveChanges Tự Động Dùng Transaction

```csharp
// Tất cả thay đổi trong 1 SaveChanges được bọc trong 1 transaction
await context.SaveChangesAsync(); // Thành công tất cả HOẶC rollback tất cả
```

### Transaction Thủ Công (Khi Cần Nhiều SaveChanges)

```csharp
await using var transaction = await context.Database.BeginTransactionAsync();
try
{
    context.Orders.Add(order);
    await context.SaveChangesAsync();

    context.Payments.Add(payment);
    await context.SaveChangesAsync();

    await transaction.CommitAsync(); // Commit — Xác nhận toàn bộ
}
catch
{
    await transaction.RollbackAsync(); // Rollback — Hoàn tác toàn bộ
    throw;
}
```

---

## 📋 Checklist Cấu Hình DbContext

- [ ] Dùng `AddDbContext` (Scoped) hoặc `AddDbContextPool` (hiệu năng cao hơn)
- [ ] Connection string lưu trong `appsettings` hoặc Secrets, không hardcode
- [ ] Bật `EnableRetryOnFailure` cho môi trường production
- [ ] Tách cấu hình entity ra `IEntityTypeConfiguration<T>` riêng biệt
- [ ] Chỉ bật `EnableSensitiveDataLogging` trong môi trường development
- [ ] Không inject `DbContext` vào Singleton services
- [ ] Dùng `IDbContextFactory` cho Background Services

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: DbContext có thread-safe không?**
> Không. Mỗi thread phải dùng instance DbContext riêng. Đây là lý do DbContext được đăng ký Scoped trong web app — mỗi request có 1 instance riêng.

**Q: `AddDbContext` vs `AddDbContextPool` khác nhau thế nào?**
> `AddDbContext` tạo mới instance mỗi request (có chi phí khởi tạo). `AddDbContextPool` tái sử dụng instances từ pool, giảm overhead cho ứng dụng có lượng request cao. Giới hạn: DbContext pooled không thể có constructor injection state.

**Q: Khi nào dùng `IDbContextFactory`?**
> Khi cần tạo DbContext ngoài phạm vi HTTP request: Background Services, Blazor Server components (có thể re-render bất kỳ lúc nào), parallel processing.
