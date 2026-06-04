# Structural Patterns — Mẫu Thiết Kế Cấu Trúc

> Structural Patterns — Mẫu Cấu Trúc — giải quyết vấn đề **kết hợp các class và object** thành cấu trúc lớn hơn, trong khi vẫn giữ cấu trúc linh hoạt và hiệu quả.

---

## 📋 Tổng Quan

| Pattern | Mục Đích | Khi Nào Dùng | .NET Ví Dụ |
| ------- | -------- | ------------ | ----------- |
| **Adapter** | Chuyển đổi interface không tương thích | Tích hợp API/library bên thứ ba | `IEnumerable<T>` wrapper |
| **Decorator** | Thêm behavior mà không sửa class gốc | Thêm logging, caching, validation | ASP.NET Middleware, `StreamReader` |
| **Facade** | Giao diện đơn giản cho hệ thống phức tạp | Che giấu độ phức tạp bên trong | Service layer, `HttpClient` |
| **Proxy** | Kiểm soát truy cập vào object | Lazy loading, caching, auth proxy | EF Core lazy loading, mock objects |
| **Composite** | Xử lý cây object đồng nhất | Tree structures, hierarchical data | File system, menu system, UI tree |

---

## 1. Adapter Pattern — Mẫu Bộ Chuyển Đổi

### Định Nghĩa

> Chuyển đổi interface của một class thành interface khác mà client mong đợi. Adapter — Bộ Chuyển Đổi — cho phép các class không tương thích làm việc cùng nhau.

**Tương tự thực tế:** Ổ cắm điện quốc tế — adapter điện giúp thiết bị 2 chân cắm vào ổ 3 chân.

### Khi Nào Dùng

- Tích hợp thư viện bên thứ ba có interface không khớp
- Tái sử dụng class cũ trong hệ thống mới
- Thay đổi thư viện mà không sửa business logic

### ✅ Object Adapter — Bộ Chuyển Đổi Đối Tượng

```csharp
// Hệ thống thanh toán cũ của công ty
public class LegacyPaymentSystem
{
    public string ProcessPaymentXml(string xmlData)
    {
        // Xử lý XML, giao tiếp với hệ thống ngân hàng cũ
        return "<result><status>SUCCESS</status><txId>TXN-001</txId></result>";
    }
}

// Interface mới mà hệ thống hiện tại cần
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request);
}

public record PaymentRequest(decimal Amount, string Currency, string CardToken);
public record PaymentResult(bool Success, string TransactionId, string? ErrorMessage);

// Adapter — chuyển đổi giữa hai interface
public class LegacyPaymentAdapter : IPaymentProcessor
{
    private readonly LegacyPaymentSystem _legacy;

    public LegacyPaymentAdapter(LegacyPaymentSystem legacy) => _legacy = legacy;

    public async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        // Chuyển đổi request mới → format XML cũ
        var xmlData = $"""
            <payment>
                <amount>{request.Amount}</amount>
                <currency>{request.Currency}</currency>
                <token>{request.CardToken}</token>
            </payment>
            """;

        // Gọi hệ thống cũ (giả lập async bằng Task.Run)
        var xmlResult = await Task.Run(() => _legacy.ProcessPaymentXml(xmlData));

        // Chuyển đổi kết quả XML → PaymentResult mới
        var success = xmlResult.Contains("<status>SUCCESS</status>");
        var txId = ExtractFromXml(xmlResult, "txId");
        return new PaymentResult(success, txId, success ? null : "Thanh toán thất bại");
    }

    private static string ExtractFromXml(string xml, string tag)
    {
        var start = xml.IndexOf($"<{tag}>") + tag.Length + 2;
        var end = xml.IndexOf($"</{tag}>");
        return start > 0 && end > 0 ? xml[start..end] : string.Empty;
    }
}

// Đăng ký trong DI Container
builder.Services.AddSingleton<LegacyPaymentSystem>();
builder.Services.AddScoped<IPaymentProcessor, LegacyPaymentAdapter>();

// Business logic không biết đến hệ thống cũ
public class OrderService
{
    private readonly IPaymentProcessor _payment;

    public OrderService(IPaymentProcessor payment) => _payment = payment;

    public async Task<bool> CheckoutAsync(Order order)
    {
        var result = await _payment.ProcessAsync(
            new PaymentRequest(order.Total, "VND", order.CardToken));

        return result.Success;
    }
}
```

### Adapter Cho ILogger — Tích Hợp Log Library Cũ

```csharp
// Thư viện log cũ
public class OldLogger
{
    public void Log(string level, string message) =>
        Console.WriteLine($"[{level}] {message}");
}

// Adapter cho ILogger<T> của .NET
public class OldLoggerAdapter<T> : ILogger<T>
{
    private readonly OldLogger _oldLogger;
    public OldLoggerAdapter(OldLogger oldLogger) => _oldLogger = oldLogger;

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state,
        Exception? exception, Func<TState, Exception?, string> formatter)
    {
        _oldLogger.Log(logLevel.ToString(), formatter(state, exception));
    }

    public bool IsEnabled(LogLevel logLevel) => true;
    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => null;
}
```

---

## 2. Decorator Pattern — Mẫu Trang Trí

### Định Nghĩa

> Gắn thêm trách nhiệm mới vào đối tượng một cách linh hoạt. Decorator — Trang Trí — cung cấp giải pháp thay thế linh hoạt cho inheritance khi cần mở rộng chức năng.

**Tương tự thực tế:** Ly cà phê + thêm sữa + thêm kem + thêm đường = mỗi thêm là một decorator.

### Khi Nào Dùng

- Thêm chức năng (logging, caching, validation) mà không sửa class gốc
- Cần kết hợp nhiều chức năng độc lập
- Thay thế subclassing để tránh class explosion — bùng nổ class

### ✅ Decorator Cho Repository — Thêm Caching

```csharp
// Interface gốc
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<IEnumerable<Product>> GetAllAsync();
    Task SaveAsync(Product product);
}

// Implementation gốc — truy cập database
public class SqlProductRepository : IProductRepository
{
    private readonly AppDbContext _db;
    public SqlProductRepository(AppDbContext db) => _db = db;

    public async Task<Product?> GetByIdAsync(Guid id)
        => await _db.Products.FindAsync(id);

    public async Task<IEnumerable<Product>> GetAllAsync()
        => await _db.Products.ToListAsync();

    public async Task SaveAsync(Product product)
    {
        _db.Products.Update(product);
        await _db.SaveChangesAsync();
    }
}

// Caching Decorator — thêm cache mà không sửa SqlProductRepository
public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner; // decorated object
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public CachedProductRepository(IProductRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Product?> GetByIdAsync(Guid id)
    {
        var cacheKey = $"product:{id}";
        if (_cache.TryGetValue(cacheKey, out Product? cached))
        {
            return cached;
        }

        var product = await _inner.GetByIdAsync(id);
        if (product != null)
            _cache.Set(cacheKey, product, _cacheDuration);

        return product;
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        const string cacheKey = "products:all";
        if (_cache.TryGetValue(cacheKey, out IEnumerable<Product>? cached))
            return cached!;

        var products = await _inner.GetAllAsync();
        _cache.Set(cacheKey, products, _cacheDuration);
        return products;
    }

    public async Task SaveAsync(Product product)
    {
        await _inner.SaveAsync(product);
        // Xóa cache khi có update
        _cache.Remove($"product:{product.Id}");
        _cache.Remove("products:all");
    }
}

// Logging Decorator — thêm logging
public class LoggingProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly ILogger<LoggingProductRepository> _logger;

    public LoggingProductRepository(IProductRepository inner,
        ILogger<LoggingProductRepository> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Product?> GetByIdAsync(Guid id)
    {
        _logger.LogDebug("Lấy product {ProductId}", id);
        var sw = Stopwatch.StartNew();
        var result = await _inner.GetByIdAsync(id);
        _logger.LogDebug("Lấy product {ProductId} trong {ElapsedMs}ms, tìm thấy: {Found}",
            id, sw.ElapsedMilliseconds, result != null);
        return result;
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        _logger.LogDebug("Lấy tất cả products");
        return await _inner.GetAllAsync();
    }

    public async Task SaveAsync(Product product)
    {
        _logger.LogInformation("Lưu product {ProductId}", product.Id);
        await _inner.SaveAsync(product);
    }
}

// Đăng ký với decorator chain: Logging → Caching → SQL
builder.Services.AddScoped<IProductRepository>(sp =>
{
    var db = sp.GetRequiredService<AppDbContext>();
    var cache = sp.GetRequiredService<IMemoryCache>();
    var logger = sp.GetRequiredService<ILogger<LoggingProductRepository>>();

    IProductRepository repo = new SqlProductRepository(db);
    repo = new CachedProductRepository(repo, cache);   // Sql → Cached
    repo = new LoggingProductRepository(repo, logger); // Cached → Logged
    return repo;
});
```

### ASP.NET Middleware Là Decorator Pattern

```csharp
// Mỗi middleware là một decorator bao quanh pipeline tiếp theo
app.Use(async (context, next) =>
{
    // Before — trước khi request đi tiếp
    var sw = Stopwatch.StartNew();

    await next(context); // gọi middleware kế tiếp (decorated object)

    // After — sau khi response quay lại
    context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
});
```

---

## 3. Facade Pattern — Mẫu Mặt Tiền

### Định Nghĩa

> Cung cấp một giao diện đơn giản, thống nhất cho một hệ thống con phức tạp. Facade — Mặt Tiền — che giấu độ phức tạp bên trong và cung cấp API dễ dùng hơn cho client.

**Tương tự thực tế:** Lễ tân khách sạn — một điểm liên lạc che giấu sự phức tạp của buồng phòng, nhà bếp, dịch vụ vận chuyển...

### ✅ Facade Cho Order Processing

```csharp
// Các subsystem — hệ thống con phức tạp
public class InventoryService
{
    public async Task<bool> CheckStockAsync(Guid productId, int quantity)
    {
        // Kiểm tra tồn kho trong warehouse management system
        await Task.Delay(50);
        return true;
    }

    public async Task ReserveStockAsync(Guid productId, int quantity)
    {
        // Khóa tồn kho để không bị đặt trùng
        await Task.Delay(50);
    }
}

public class PaymentService
{
    public async Task<string> ChargeAsync(string cardToken, decimal amount)
    {
        // Giao tiếp với payment gateway
        await Task.Delay(100);
        return $"TXN-{Guid.NewGuid():N}";
    }
}

public class ShippingService
{
    public async Task<string> CreateShipmentAsync(Order order)
    {
        // Tạo vận đơn với shipping provider
        await Task.Delay(80);
        return $"SHIP-{Guid.NewGuid():N[..8]}";
    }
}

public class NotificationService
{
    public async Task SendOrderConfirmationAsync(string email, string orderId)
    {
        // Gửi email xác nhận
        await Task.Delay(30);
    }
}

public class AuditService
{
    public async Task LogOrderEventAsync(string orderId, string eventType)
    {
        // Ghi log audit
        await Task.Delay(10);
    }
}

// Facade — che giấu toàn bộ quy trình phức tạp
public class OrderFacade
{
    private readonly InventoryService _inventory;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;
    private readonly NotificationService _notification;
    private readonly AuditService _audit;
    private readonly ILogger<OrderFacade> _logger;

    public OrderFacade(
        InventoryService inventory,
        PaymentService payment,
        ShippingService shipping,
        NotificationService notification,
        AuditService audit,
        ILogger<OrderFacade> logger)
    {
        _inventory = inventory;
        _payment = payment;
        _shipping = shipping;
        _notification = notification;
        _audit = audit;
        _logger = logger;
    }

    // Client chỉ cần gọi một method đơn giản này
    public async Task<OrderResult> PlaceOrderAsync(PlaceOrderCommand command)
    {
        _logger.LogInformation("Bắt đầu xử lý đơn hàng cho customer {CustomerId}",
            command.CustomerId);

        // 1. Kiểm tra tồn kho
        var hasStock = await _inventory.CheckStockAsync(command.ProductId, command.Quantity);
        if (!hasStock)
            return OrderResult.Failed("Sản phẩm đã hết hàng");

        // 2. Khóa tồn kho
        await _inventory.ReserveStockAsync(command.ProductId, command.Quantity);

        // 3. Xử lý thanh toán
        string transactionId;
        try
        {
            transactionId = await _payment.ChargeAsync(command.CardToken, command.Amount);
        }
        catch (PaymentException ex)
        {
            _logger.LogError(ex, "Thanh toán thất bại");
            return OrderResult.Failed("Thanh toán thất bại: " + ex.Message);
        }

        // 4. Tạo đơn giao hàng
        var orderId = Guid.NewGuid().ToString();
        var shipmentId = await _shipping.CreateShipmentAsync(new Order(orderId, command));

        // 5. Gửi xác nhận
        await _notification.SendOrderConfirmationAsync(command.CustomerEmail, orderId);

        // 6. Ghi audit log
        await _audit.LogOrderEventAsync(orderId, "ORDER_PLACED");

        return OrderResult.Success(orderId, transactionId, shipmentId);
    }
}

// Controller chỉ biết Facade, không biết gì về các subsystem
[ApiController, Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly OrderFacade _facade;

    public OrdersController(OrderFacade facade) => _facade = facade;

    [HttpPost]
    public async Task<IActionResult> PlaceOrder(PlaceOrderCommand command)
    {
        var result = await _facade.PlaceOrderAsync(command);
        return result.Success ? Ok(result) : BadRequest(result.ErrorMessage);
    }
}
```

---

## 4. Proxy Pattern — Mẫu Ủy Quyền

### Định Nghĩa

> Cung cấp một đối tượng thay thế hoặc placeholder — vật giữ chỗ — cho một đối tượng khác để kiểm soát việc truy cập vào đối tượng đó.

### Các Loại Proxy

| Loại | Mục Đích |
| ---- | -------- |
| **Virtual Proxy** | Lazy loading — khởi tạo chậm khi thực sự cần |
| **Protection Proxy** | Kiểm soát quyền truy cập (authorization) |
| **Remote Proxy** | Đại diện cho object ở nơi khác (RPC, gRPC) |
| **Caching Proxy** | Cache kết quả của object tốn kém |
| **Logging Proxy** | Ghi log mọi lần truy cập |

### ✅ Virtual Proxy — Lazy Loading

```csharp
public interface IReport
{
    Task<byte[]> GeneratePdfAsync();
    string GetTitle();
}

// Object thực sự — tốn kém để tạo (query database lớn, xử lý data)
public class SalesReport : IReport
{
    private readonly AppDbContext _db;
    private readonly byte[]? _cachedPdf;

    public SalesReport(AppDbContext db) => _db = db;

    public string GetTitle() => "Báo Cáo Doanh Thu";

    public async Task<byte[]> GeneratePdfAsync()
    {
        // Tốn nhiều tài nguyên: query lớn + render PDF
        var data = await _db.Orders
            .Include(o => o.Items)
            .ToListAsync();
        return await PdfGenerator.GenerateAsync(data);
    }
}

// Virtual Proxy — chỉ khởi tạo SalesReport khi thực sự cần PDF
public class LazyReportProxy : IReport
{
    private readonly Lazy<SalesReport> _report;

    public LazyReportProxy(AppDbContext db)
    {
        // Report chỉ được tạo khi GeneratePdfAsync() được gọi lần đầu
        _report = new Lazy<SalesReport>(() => new SalesReport(db));
    }

    public string GetTitle() => "Báo Cáo Doanh Thu"; // Trả về nhanh, không cần tạo report

    public async Task<byte[]> GeneratePdfAsync()
        => await _report.Value.GeneratePdfAsync(); // Khởi tạo ở đây mới cần
}
```

### ✅ Protection Proxy — Kiểm Soát Quyền Truy Cập

```csharp
public interface IDocumentService
{
    Task<Document> GetAsync(Guid id);
    Task UpdateAsync(Document document);
    Task DeleteAsync(Guid id);
}

public class DocumentService : IDocumentService
{
    private readonly IDocumentRepository _repo;
    public DocumentService(IDocumentRepository repo) => _repo = repo;

    public Task<Document> GetAsync(Guid id) => _repo.GetByIdAsync(id);
    public Task UpdateAsync(Document document) => _repo.SaveAsync(document);
    public Task DeleteAsync(Guid id) => _repo.DeleteAsync(id);
}

// Protection Proxy — kiểm tra quyền trước mỗi thao tác
public class AuthorizedDocumentService : IDocumentService
{
    private readonly IDocumentService _inner;
    private readonly ICurrentUser _currentUser;

    public AuthorizedDocumentService(IDocumentService inner, ICurrentUser currentUser)
    {
        _inner = inner;
        _currentUser = currentUser;
    }

    public async Task<Document> GetAsync(Guid id)
    {
        var doc = await _inner.GetAsync(id);
        if (!doc.IsPublic && doc.OwnerId != _currentUser.Id && !_currentUser.IsAdmin)
            throw new UnauthorizedAccessException("Bạn không có quyền xem tài liệu này");
        return doc;
    }

    public async Task UpdateAsync(Document document)
    {
        if (document.OwnerId != _currentUser.Id && !_currentUser.IsAdmin)
            throw new UnauthorizedAccessException("Bạn không có quyền sửa tài liệu này");
        await _inner.UpdateAsync(document);
    }

    public async Task DeleteAsync(Guid id)
    {
        if (!_currentUser.IsAdmin)
            throw new UnauthorizedAccessException("Chỉ admin mới có thể xóa tài liệu");
        await _inner.DeleteAsync(id);
    }
}
```

### Proxy vs Decorator — Sự Khác Nhau

| | Decorator | Proxy |
| - | --------- | ----- |
| **Mục đích** | Thêm behavior | Kiểm soát truy cập |
| **Tạo object** | Nhận đã khởi tạo | Có thể tự quản lý vòng đời |
| **Client biết không?** | Client không cần biết | Client thường không biết là proxy |
| **Ví dụ** | Thêm logging, caching | Auth proxy, lazy loading |

---

## 5. Composite Pattern — Mẫu Tổng Hợp

### Định Nghĩa

> Tổ chức các đối tượng thành cấu trúc **cây** để thể hiện mối quan hệ "phần-toàn". Composite — Tổng Hợp — cho phép client xử lý đối tượng đơn lẻ và nhóm đối tượng theo cách đồng nhất.

### Khi Nào Dùng

- File system: file và folder
- Menu hệ thống: menu item và submenu
- UI component tree: widget đơn lẻ và container
- Organizational hierarchy: nhân viên và phòng ban

### ✅ Composite Cho Permission System — Hệ Thống Phân Quyền

```csharp
// Component — thành phần: interface chung
public interface IPermission
{
    string Name { get; }
    bool Check(User user);
    IEnumerable<string> GetAllPermissions();
}

// Leaf — lá: permission đơn lẻ
public class SimplePermission : IPermission
{
    public string Name { get; }
    private readonly Func<User, bool> _check;

    public SimplePermission(string name, Func<User, bool> check)
    {
        Name = name;
        _check = check;
    }

    public bool Check(User user) => _check(user);
    public IEnumerable<string> GetAllPermissions() => [Name];
}

// Composite — nhóm permission
public class PermissionGroup : IPermission
{
    public string Name { get; }
    private readonly List<IPermission> _permissions = [];
    private readonly bool _requireAll; // true = AND logic, false = OR logic

    public PermissionGroup(string name, bool requireAll = true)
    {
        Name = name;
        _requireAll = requireAll;
    }

    public void Add(IPermission permission) => _permissions.Add(permission);

    public bool Check(User user) =>
        _requireAll
            ? _permissions.All(p => p.Check(user))   // AND: tất cả đều phải thỏa
            : _permissions.Any(p => p.Check(user));   // OR: ít nhất một thỏa

    public IEnumerable<string> GetAllPermissions() =>
        _permissions.SelectMany(p => p.GetAllPermissions());
}

// Xây dựng cây permission
var canViewOrder = new SimplePermission("view:orders", u => u.Roles.Contains("staff"));
var canEditOrder = new SimplePermission("edit:orders", u => u.Roles.Contains("manager"));
var canDeleteOrder = new SimplePermission("delete:orders", u => u.IsAdmin);

var orderManagement = new PermissionGroup("order:management");
orderManagement.Add(canViewOrder);
orderManagement.Add(canEditOrder);
orderManagement.Add(canDeleteOrder);

// Kiểm tra — code xử lý group và leaf đồng nhất
IPermission permission = orderManagement;
if (permission.Check(currentUser))
    Console.WriteLine($"User có đủ quyền: {string.Join(", ", permission.GetAllPermissions())}");
```

### ✅ Composite Cho File System

```csharp
// Component
public abstract class FileSystemItem
{
    public string Name { get; init; } = string.Empty;
    public abstract long GetSize();
    public abstract void Print(string indent = "");
}

// Leaf — file đơn lẻ
public class File : FileSystemItem
{
    public long SizeInBytes { get; init; }
    public override long GetSize() => SizeInBytes;
    public override void Print(string indent = "")
        => Console.WriteLine($"{indent}📄 {Name} ({SizeInBytes:N0} bytes)");
}

// Composite — thư mục chứa các item khác
public class Directory : FileSystemItem
{
    private readonly List<FileSystemItem> _items = [];

    public void Add(FileSystemItem item) => _items.Add(item);
    public void Remove(FileSystemItem item) => _items.Remove(item);

    public override long GetSize() => _items.Sum(item => item.GetSize());

    public override void Print(string indent = "")
    {
        Console.WriteLine($"{indent}📁 {Name} ({GetSize():N0} bytes)");
        foreach (var item in _items)
            item.Print(indent + "  ");
    }
}

// Xây dựng cây và xử lý đồng nhất
var root = new Directory { Name = "project" };
var src = new Directory { Name = "src" };
src.Add(new File { Name = "Program.cs", SizeInBytes = 2_048 });
src.Add(new File { Name = "Startup.cs", SizeInBytes = 4_096 });

var tests = new Directory { Name = "tests" };
tests.Add(new File { Name = "UnitTests.cs", SizeInBytes = 8_192 });

root.Add(src);
root.Add(tests);
root.Add(new File { Name = "README.md", SizeInBytes = 1_024 });

root.Print(); // in toàn bộ cây
Console.WriteLine($"Tổng dung lượng: {root.GetSize():N0} bytes");
```

---

## 🔄 Tổng Kết So Sánh

| Pattern | Câu Hỏi Gợi Nhớ |
| ------- | --------------- |
| **Adapter** | "Interface này không khớp, làm sao kết nối?" |
| **Decorator** | "Làm thế nào thêm chức năng mà không sửa class?" |
| **Facade** | "Hệ thống phức tạp quá, cần API đơn giản hơn?" |
| **Proxy** | "Cần kiểm soát truy cập vào object này?" |
| **Composite** | "Cần xử lý object đơn và nhóm object giống nhau?" |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: "Decorator và Inheritance — khi nào dùng cái nào?"**
→ Dùng Decorator khi cần thêm nhiều combination of behavior độc lập (logging + caching + validation). Inheritance tạo ra class explosion khi có nhiều combination. Decorator linh hoạt hơn và tuân thủ Open/Closed Principle.

**Q: "ASP.NET Core Middleware dùng pattern nào?"**
→ Decorator Pattern. Mỗi middleware bao quanh pipeline tiếp theo, thêm behavior trước và sau khi gọi `next()`.

**Q: "Proxy và Decorator khác nhau thế nào?"**
→ Decorator thêm behavior, Proxy kiểm soát truy cập. Decorator nhận object đã khởi tạo, Proxy có thể tự quản lý vòng đời. EF Core lazy loading dùng Virtual Proxy.

**Q: "Khi nào dùng Facade?"**
→ Khi client cần làm việc với nhiều subsystem phức tạp. Facade giảm coupling và cung cấp API đơn giản hơn. Service layer trong Clean Architecture thường là Facade.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
