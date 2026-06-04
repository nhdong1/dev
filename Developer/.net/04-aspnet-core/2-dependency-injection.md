# 2 — Dependency Injection (Tiêm Phụ Thuộc)

> DI — Dependency Injection — Tiêm Phụ Thuộc — là cơ chế ASP.NET Core tích hợp sẵn để quản lý lifetime của object và giải quyết phụ thuộc giữa các class. Hiểu đúng ba lifetime (Singleton, Scoped, Transient) là điều kiện tiên quyết để tránh những lỗi tinh vi và khó debug.

---

## 📋 Tổng Quan Nhanh

| Lifetime | Tạo Khi Nào | Hủy Khi Nào | Dùng Cho |
| -------- | ----------- | ----------- | -------- |
| **Singleton** | App khởi động (lần đầu resolve) | App shutdown | Cache, config, stateless service |
| **Scoped** | Mỗi HTTP request | Request kết thúc | DbContext, Unit of Work, per-request service |
| **Transient** | Mỗi lần resolve | Khi scope dispose | Lightweight, stateless operation |

---

## 1. Ba Lifetime Cơ Bản

### Singleton — Một Instance Cho Toàn App

```csharp
// Đăng ký Singleton
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();
builder.Services.AddSingleton<ICurrencyConverter, CurrencyConverter>();

// Hoặc đăng ký với instance cụ thể
var settings = new AppSettings { ApiKey = "abc123" };
builder.Services.AddSingleton<IAppSettings>(settings);

// Hoặc với factory
builder.Services.AddSingleton<IHeavyService>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new HeavyService(config["ApiUrl"]);
});
```

```
App Start → [Singleton Instance Created] ─────────────────────────────→ App Shutdown
                    ↑                 ↑                  ↑
              Request 1          Request 2          Request 3
              (same instance)  (same instance)   (same instance)
```

**Khi nào dùng Singleton:**
- Service **stateless** hoàn toàn (không giữ trạng thái mutable)
- Resource tốn kém để khởi tạo (HTTP client, connection pool)
- Cache service, configuration reader
- Logging infrastructure

### Scoped — Một Instance Cho Mỗi Request

```csharp
// Đăng ký Scoped
builder.Services.AddScoped<AppDbContext>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
```

```
Request 1 → [Scoped Instance A] → Request ends → Disposed
Request 2 → [Scoped Instance B] → Request ends → Disposed
Request 3 → [Scoped Instance C] → Request ends → Disposed
```

**Khi nào dùng Scoped:**
- `DbContext` — phải là Scoped để theo dõi thay đổi trong một request
- Service cần trạng thái trong phạm vi một request
- Unit of Work pattern
- Service cần biết user hiện tại (đọc từ HttpContext)

### Transient — Tạo Mới Mỗi Lần

```csharp
// Đăng ký Transient
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddTransient<IPasswordHasher, BcryptPasswordHasher>();
builder.Services.AddTransient<IGuidGenerator, GuidGenerator>();
```

```
Request → ServiceA resolves ITransient → [Instance 1 created]
        → ServiceB resolves ITransient → [Instance 2 created]  (khác instance!)
        → ServiceC resolves ITransient → [Instance 3 created]  (khác instance!)
```

**Khi nào dùng Transient:**
- Service rất nhẹ, không tốn tài nguyên
- Service không thread-safe (cần instance riêng cho mỗi lần dùng)
- Email sender, password hasher, validator
- Khi muốn chắc chắn không có shared state

---

## 2. Captive Dependency — Bẫy Phụ Thuộc Bị Giam Cầm

**Captive dependency** xảy ra khi service có lifetime dài hơn inject service có lifetime ngắn hơn. Đây là lỗi tinh vi nhất trong DI.

### Ví Dụ Lỗi Kinh Điển

```csharp
// ❌ VẤN ĐỀ: Singleton inject Scoped service
public class OrderService  // Singleton
{
    private readonly AppDbContext _db;  // Scoped — BỊ GẮN CHẶT vào Singleton!

    public OrderService(AppDbContext db)
    {
        _db = db;  // _db là instance của request đầu tiên
                   // Mọi request sau đều dùng chung DbContext cũ!
    }
}

// Đăng ký
builder.Services.AddSingleton<OrderService>();  // ← Singleton
builder.Services.AddScoped<AppDbContext>();      // ← Scoped
```

**Hậu quả:**
- `AppDbContext` bị "giam cầm" trong Singleton, sống mãi mãi
- Change tracking bị nhiễm dữ liệu từ các request khác
- Memory leak: DbContext không bao giờ được dispose
- Thread-safety issues trong môi trường concurrent

### Cách Phát Hiện

ASP.NET Core **tự động throw exception** khi validate scope:

```csharp
// Program.cs — bật validation (mặc định bật trong Development)
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;    // Bắt captive dependency
    options.ValidateOnBuild = true;   // Validate ngay khi build
});
```

Output lỗi:
```
InvalidOperationException: Cannot consume scoped service 'AppDbContext'
from singleton 'OrderService'.
```

### Cách Sửa 1: Đổi Lifetime

```csharp
// Nếu OrderService không cần là Singleton → đổi thành Scoped
builder.Services.AddScoped<OrderService>();
builder.Services.AddScoped<AppDbContext>();
```

### Cách Sửa 2: Dùng IServiceScopeFactory

Khi Singleton thực sự cần dùng Scoped service (ví dụ: background job):

```csharp
public class BackgroundJobService  // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;

    public BackgroundJobService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task ProcessAsync()
    {
        // Tạo scope mới mỗi lần cần dùng Scoped service
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        var orders = await db.Orders.Where(o => !o.IsProcessed).ToListAsync();
        // xử lý...
    }  // scope Dispose → db cũng được Dispose
}
```

---

## 3. Các Cách Đăng Ký Service

### Đăng Ký Interface → Implementation

```csharp
// Cơ bản: interface → concrete class
builder.Services.AddScoped<IUserService, UserService>();

// Đăng ký nhiều implementation cho cùng interface
builder.Services.AddScoped<INotificationSender, EmailNotificationSender>();
builder.Services.AddScoped<INotificationSender, SmsNotificationSender>();
builder.Services.AddScoped<INotificationSender, PushNotificationSender>();

// Inject tất cả implementations
public class NotificationDispatcher
{
    private readonly IEnumerable<INotificationSender> _senders;

    public NotificationDispatcher(IEnumerable<INotificationSender> senders)
    {
        _senders = senders;
    }

    public async Task SendAllAsync(string message)
    {
        foreach (var sender in _senders)
            await sender.SendAsync(message);
    }
}
```

### Factory Registration — Đăng Ký Qua Factory

```csharp
// Factory nhận IServiceProvider để resolve dependencies
builder.Services.AddScoped<IPaymentGateway>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var logger = sp.GetRequiredService<ILogger<StripeGateway>>();

    return config["Payment:Provider"] switch
    {
        "Stripe" => new StripeGateway(config["Payment:StripeKey"], logger),
        "PayPal" => new PayPalGateway(config["Payment:PayPalKey"], logger),
        _ => throw new InvalidOperationException("Unknown payment provider")
    };
});
```

### TryAdd — Đăng Ký Nếu Chưa Có

```csharp
// TryAddScoped — chỉ đăng ký nếu chưa có service này
builder.Services.TryAddScoped<IUserService, UserService>();
builder.Services.TryAddSingleton<ICacheService, MemoryCacheService>();

// Hữu ích khi viết library: không ghi đè service mà user đã đăng ký
```

### Đăng Ký Concrete Type (Không Có Interface)

```csharp
// Không dùng interface — thường cho internal service
builder.Services.AddScoped<UserValidator>();
builder.Services.AddSingleton<ApplicationMetrics>();
```

---

## 4. Resolve Service — Lấy Service

### Constructor Injection (Khuyến Nghị)

```csharp
// Cách được khuyến nghị — DI inject qua constructor
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        IUserService userService,
        ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        return user is null ? NotFound() : Ok(user);
    }
}
```

### Method Injection với [FromServices]

```csharp
// Inject service trực tiếp vào action method
[HttpPost]
public async Task<IActionResult> CreateOrder(
    CreateOrderRequest request,
    [FromServices] IOrderService orderService,   // ← inject trực tiếp vào method
    [FromServices] ILogger<OrdersController> logger)
{
    logger.LogInformation("Creating order for user {UserId}", request.UserId);
    var order = await orderService.CreateAsync(request);
    return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
}
```

### IServiceProvider — Resolve Thủ Công (Tránh Dùng)

```csharp
// ⚠️ Tránh dùng — đây là Service Locator anti-pattern
public class BadService
{
    private readonly IServiceProvider _provider;

    public BadService(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void DoWork()
    {
        // ❌ Anti-pattern: ẩn dependency, khó test, khó hiểu
        var userService = _provider.GetRequiredService<IUserService>();
    }
}

// ✅ Chỉ dùng IServiceProvider trong trường hợp thực sự cần:
// - Factory method
// - Conditional resolution
// - Background services cần tạo scope
```

---

## 5. Keyed Services — Service Có Tên (ASP.NET Core 8+)

Khi cần nhiều implementation của cùng interface và cần phân biệt bằng key:

```csharp
// Đăng ký Keyed Services
builder.Services.AddKeyedScoped<IStorageService, LocalStorageService>("local");
builder.Services.AddKeyedScoped<IStorageService, AzureBlobStorageService>("azure");
builder.Services.AddKeyedScoped<IStorageService, S3StorageService>("s3");

// Inject bằng [FromKeyedServices]
public class FileController : ControllerBase
{
    [HttpPost("upload/local")]
    public async Task<IActionResult> UploadLocal(
        IFormFile file,
        [FromKeyedServices("local")] IStorageService storage)
    {
        await storage.UploadAsync(file.OpenReadStream(), file.FileName);
        return Ok();
    }

    [HttpPost("upload/cloud")]
    public async Task<IActionResult> UploadCloud(
        IFormFile file,
        [FromKeyedServices("azure")] IStorageService storage)
    {
        await storage.UploadAsync(file.OpenReadStream(), file.FileName);
        return Ok();
    }
}
```

---

## 6. Options Pattern — Quản Lý Cấu Hình Qua DI

`IOptions<T>` tích hợp DI để inject strongly-typed configuration:

```csharp
// Định nghĩa Options class
public class JwtOptions
{
    public string Secret { get; set; } = string.Empty;
    public int ExpiresInMinutes { get; set; } = 60;
    public string Issuer { get; set; } = string.Empty;
}

// Đăng ký
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection("Jwt"));

// Inject và sử dụng
public class JwtTokenService
{
    private readonly JwtOptions _options;

    public JwtTokenService(IOptions<JwtOptions> options)
    {
        _options = options.Value;  // .Value lấy instance đã cấu hình
    }

    public string GenerateToken(string userId)
    {
        // dùng _options.Secret, _options.ExpiresInMinutes...
    }
}
```

Xem chi tiết tại [7-configuration.md](7-configuration.md).

---

## 7. Kiểm Tra DI Registration

### Validate Toàn Bộ Service Graph

```csharp
// Validate khi build — bắt lỗi sớm
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;   // Bắt captive dependency
    options.ValidateOnBuild = true;  // Validate tất cả registrations khi app khởi động
});
```

### Extension Methods Để Tổ Chức Registration

```csharp
// Nhóm registrations theo feature module
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddUserModule(
        this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IUserValidator, UserValidator>();
        return services;
    }

    public static IServiceCollection AddOrderModule(
        this IServiceCollection services)
    {
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IOrderService, OrderService>();
        return services;
    }
}

// Program.cs gọn hơn
builder.Services
    .AddUserModule()
    .AddOrderModule();
```

---

## ⚠️ Các Lỗi Phổ Biến

| Lỗi | Nguyên Nhân | Cách Tránh |
| --- | ----------- | ---------- |
| Captive dependency | Singleton inject Scoped | Dùng `IServiceScopeFactory` hoặc đổi lifetime |
| Service không được resolve | Quên đăng ký | Dùng `ValidateOnBuild = true` |
| `GetRequiredService` throw | Service chưa đăng ký | Kiểm tra registration hoặc dùng `GetService` để null-check |
| Circular dependency | A → B → A | Tái cấu trúc, dùng lazy injection hoặc factory |
| Thread-safety với Singleton | Singleton giữ mutable state | Dùng `lock`, `ConcurrentDictionary`, hoặc immutable state |

---

## 🔑 So Sánh 3 Lifetime — Bảng Tổng Hợp

| | Singleton | Scoped | Transient |
| - | --------- | ------ | --------- |
| **Số instance** | 1 (cho toàn app) | 1 per request | Mỗi lần resolve |
| **Thread-safe** | Phải tự đảm bảo | 1 thread per request (thường OK) | Mỗi lần mới (thường OK) |
| **Có thể inject** | Singleton ✅ | Singleton ❌, Scoped ✅ | Mọi lifetime ✅ |
| **Phù hợp** | Cache, stateless service | DbContext, UoW | Validator, lightweight op |
| **Bộ nhớ** | Thấp (1 instance) | Trung bình | Cao hơn (nhiều instance) |

---

## ✅ Checklist

- [ ] Hiểu rõ ba lifetime và khi nào dùng mỗi loại
- [ ] Biết captive dependency là gì và cách tránh
- [ ] Viết được Extension Methods để tổ chức registrations
- [ ] Dùng `IServiceScopeFactory` trong background services
- [ ] Bật `ValidateOnBuild` và `ValidateScopes` trong Development
- [ ] Tránh Service Locator anti-pattern (không inject `IServiceProvider`)
- [ ] Hiểu Keyed Services trong .NET 8+
