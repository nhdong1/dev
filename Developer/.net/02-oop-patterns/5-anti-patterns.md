# Anti-Patterns — Các Mẫu Thiết Kế Sai Cần Tránh

> Anti-pattern — Mẫu Thiết Kế Sai — là những giải pháp trông có vẻ hợp lý và được dùng phổ biến, nhưng thực tế tạo ra nhiều vấn đề hơn là giải quyết. Nhận biết chúng quan trọng không kém biết các design pattern tốt.

---

## 📋 Danh Sách Anti-Patterns

| Anti-Pattern | Triệu Chứng Chính | Vi Phạm SOLID | Mức Độ Nguy Hiểm |
| ------------ | ----------------- | ------------- | ---------------- |
| **God Object** | Class làm quá nhiều thứ | SRP | 🔴 Nghiêm trọng |
| **Tight Coupling** | Thay đổi một class phá vỡ nhiều class khác | DIP, OCP | 🔴 Nghiêm trọng |
| **Service Locator** | "Hộp ma thuật" tự tìm dependency | DIP | 🟠 Cao |
| **Anemic Domain Model** | Domain object chỉ có data, không có behavior | SRP | 🟠 Cao |
| **Shotgun Surgery** | Một thay đổi logic phải sửa nhiều file | SRP | 🟡 Trung bình |
| **Premature Optimization** | Tối ưu trước khi đo lường | - | 🟡 Trung bình |
| **Magic Numbers/Strings** | Số và chuỗi ma thuật không có tên | - | 🟡 Trung bình |

---

## 1. God Object — Đối Tượng Thần Thánh

### Mô Tả

Class quá lớn, biết quá nhiều, làm quá nhiều thứ. Thường được sinh ra từ việc "cứ thêm vào đây cho nhanh" dẫn đến mã spaghetti — spaghetti code.

### ❌ Nhận Biết God Object

```csharp
// BAD: UserManager làm tất cả mọi thứ liên quan đến user
public class UserManager // 2000+ dòng
{
    // Quản lý database
    private readonly SqlConnection _conn;

    // Gửi email
    private readonly SmtpClient _smtp;

    // Xử lý thanh toán
    private readonly PaymentGateway _payment;

    // Quản lý file
    private readonly FileStorage _storage;

    // Tất cả logic user dồn vào đây:
    public async Task RegisterAsync(RegisterRequest request) { /* ... */ }
    public async Task LoginAsync(string email, string password) { /* ... */ }
    public async Task UpdateProfileAsync(Guid userId, ProfileData data) { /* ... */ }
    public async Task SendVerificationEmailAsync(Guid userId) { /* ... */ }
    public async Task ResetPasswordAsync(string email) { /* ... */ }
    public async Task UploadAvatarAsync(Guid userId, byte[] imageData) { /* ... */ }
    public async Task SubscribePremiumAsync(Guid userId, CreditCard card) { /* ... */ }
    public async Task GenerateMonthlyReportAsync(Guid userId) { /* ... */ }
    public async Task SendNewsletterAsync() { /* ... */ }
    public async Task BanUserAsync(Guid userId, string reason) { /* ... */ }
    // ... 50 method khác
}
```

**Dấu Hiệu Nhận Biết:**
- Class có > 500–1000 dòng
- Constructor có > 7–8 dependency
- Class tên là `Manager`, `Handler`, `Helper`, `Util` với phạm vi quá rộng
- Mọi developer đều sửa file này khi thêm feature mới

### ✅ Cách Sửa: Tách Thành Các Class Có Trách Nhiệm Rõ Ràng

```csharp
// GOOD: Mỗi class một trách nhiệm

public class UserRegistrationService
{
    private readonly IUserRepository _users;
    private readonly IPasswordHasher _hasher;
    private readonly IEmailVerificationService _emailVerification;

    public async Task<User> RegisterAsync(RegisterRequest request)
    {
        var user = User.Create(request.Email, _hasher.Hash(request.Password));
        await _users.SaveAsync(user);
        await _emailVerification.SendVerificationAsync(user.Email, user.Id);
        return user;
    }
}

public class AuthenticationService
{
    private readonly IUserRepository _users;
    private readonly IPasswordHasher _hasher;
    private readonly ITokenService _tokens;

    public async Task<AuthResult> LoginAsync(string email, string password)
    {
        var user = await _users.FindByEmailAsync(email);
        if (user == null || !_hasher.Verify(password, user.PasswordHash))
            return AuthResult.Failed("Email hoặc mật khẩu không đúng");

        var token = _tokens.Generate(user);
        return AuthResult.Success(token);
    }
}

public class UserProfileService
{
    private readonly IUserRepository _users;
    private readonly IFileStorage _storage;
    private readonly IImageProcessor _images;

    public async Task UpdateProfileAsync(Guid userId, ProfileData data) { /* ... */ }
    public async Task UploadAvatarAsync(Guid userId, Stream image) { /* ... */ }
}

public class SubscriptionService
{
    private readonly IUserRepository _users;
    private readonly IPaymentProcessor _payment;
    private readonly IEmailService _email;

    public async Task SubscribePremiumAsync(Guid userId, CreditCard card) { /* ... */ }
}
```

---

## 2. Tight Coupling — Ghép Nối Chặt

### Mô Tả

Khi các class phụ thuộc trực tiếp vào implementation cụ thể của nhau, thay đổi một class kéo theo phải sửa nhiều class khác. Đây là vi phạm DIP — Dependency Inversion Principle.

### ❌ Nhận Biết Tight Coupling

```csharp
// BAD: OrderService biết quá nhiều về implementation cụ thể
public class OrderService
{
    // Phụ thuộc trực tiếp vào SQL Server repository — không thể đổi sang MongoDB
    public async Task<Order> PlaceOrderAsync(OrderRequest request)
    {
        // new trực tiếp → không thể mock trong test
        var repo = new SqlServerOrderRepository("Server=prod-db;Database=shop;...");
        var emailer = new GmailEmailService("smtp.gmail.com", 587);
        var inventory = new WarehouseInventoryChecker();

        if (!await inventory.CheckAsync(request.ProductId, request.Quantity))
            throw new InsufficientStockException();

        var order = new Order(request);
        await repo.SaveAsync(order);
        await emailer.SendConfirmationAsync(request.CustomerEmail, order.Id);

        return order;
    }
}

// Hệ quả:
// - Không thể viết unit test (cần SQL Server thật, Gmail thật, Warehouse thật)
// - Muốn đổi database → sửa OrderService
// - Muốn đổi email provider → sửa OrderService
// - Nhiều class khác cũng new SqlServerOrderRepository() → scattered coupling
```

### ✅ Cách Sửa: Dùng Interface + Dependency Injection

```csharp
// GOOD: Phụ thuộc vào abstraction, inject qua constructor

public class OrderService
{
    private readonly IOrderRepository _orders;     // abstraction
    private readonly IEmailService _email;          // abstraction
    private readonly IInventoryService _inventory;  // abstraction
    private readonly IEventPublisher _events;

    // Constructor injection — DI Container cung cấp implementation
    public OrderService(
        IOrderRepository orders,
        IEmailService email,
        IInventoryService inventory,
        IEventPublisher events)
    {
        _orders = orders;
        _email = email;
        _inventory = inventory;
        _events = events;
    }

    public async Task<Order> PlaceOrderAsync(OrderRequest request)
    {
        if (!await _inventory.CheckAsync(request.ProductId, request.Quantity))
            throw new InsufficientStockException();

        var order = Order.Create(request);
        await _orders.SaveAsync(order);
        await _email.SendConfirmationAsync(request.CustomerEmail, order.Id);
        await _events.PublishAsync(new OrderPlacedEvent(order.Id));

        return order;
    }
}

// Unit test dễ dàng vì tất cả dependency đều là interface
[Fact]
public async Task PlaceOrder_WhenStockSufficient_ShouldSaveAndSendEmail()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();
    var mockInventory = new Mock<IInventoryService>();
    var mockEvents = new Mock<IEventPublisher>();

    mockInventory.Setup(i => i.CheckAsync(It.IsAny<Guid>(), It.IsAny<int>()))
        .ReturnsAsync(true);

    var service = new OrderService(mockRepo.Object, mockEmail.Object,
        mockInventory.Object, mockEvents.Object);

    await service.PlaceOrderAsync(new OrderRequest());

    mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
    mockEmail.Verify(e => e.SendConfirmationAsync(It.IsAny<string>(), It.IsAny<Guid>()), Times.Once);
}
```

### Tight Coupling Khó Nhận Ra Hơn

```csharp
// BAD: Tight coupling thông qua static method — khó mock
public class ReportGenerator
{
    public Report Generate(Guid reportId)
    {
        // Static call → không thể mock trong test
        var data = DatabaseHelper.GetReportData(reportId);
        var formatted = CurrencyFormatter.Format(data.Total, "VND");
        Logger.Log($"Generated report {reportId}"); // static Logger
        return new Report(data, formatted);
    }
}

// GOOD: Inject thay vì dùng static
public class ReportGenerator
{
    private readonly IReportDataService _dataService;
    private readonly ICurrencyFormatter _formatter;
    private readonly ILogger<ReportGenerator> _logger;

    public ReportGenerator(
        IReportDataService dataService,
        ICurrencyFormatter formatter,
        ILogger<ReportGenerator> logger)
    {
        _dataService = dataService;
        _formatter = formatter;
        _logger = logger;
    }

    public async Task<Report> GenerateAsync(Guid reportId)
    {
        var data = await _dataService.GetAsync(reportId);
        var formatted = _formatter.Format(data.Total, "VND");
        _logger.LogInformation("Đã tạo report {ReportId}", reportId);
        return new Report(data, formatted);
    }
}
```

---

## 3. Service Locator Anti-Pattern — Bộ Định Vị Dịch Vụ (Cần Tránh)

### Mô Tả

Service Locator là một global registry (đăng ký toàn cục) cung cấp service theo yêu cầu. Trông giống Dependency Injection nhưng thực chất là Tight Coupling ẩn — hidden coupling và vi phạm DIP theo cách tinh vi hơn.

### ❌ Service Locator Anti-Pattern

```csharp
// BAD: Service Locator — "hộp ma thuật"
public static class ServiceLocator
{
    private static readonly Dictionary<Type, object> _services = [];

    public static void Register<T>(T service) where T : class
        => _services[typeof(T)] = service;

    public static T Resolve<T>() where T : class
        => (_services.TryGetValue(typeof(T), out var service) ? service : null) as T
           ?? throw new InvalidOperationException($"Service {typeof(T).Name} chưa được đăng ký");
}

// Đăng ký services
ServiceLocator.Register<IEmailService>(new SmtpEmailService());
ServiceLocator.Register<IOrderRepository>(new SqlOrderRepository(connection));

// Class dùng Service Locator — VẤN ĐỀ Ở ĐÂY
public class OrderService
{
    public async Task PlaceOrderAsync(OrderRequest request)
    {
        // Dependencies ẩn — không thấy trong constructor!
        var repo = ServiceLocator.Resolve<IOrderRepository>();
        var email = ServiceLocator.Resolve<IEmailService>();

        // Logic xử lý...
        await repo.SaveAsync(new Order(request));
        await email.SendConfirmationAsync(request.CustomerEmail);
    }
}
```

**Tại Sao Đây Là Anti-Pattern:**

```
1. Dependencies ẩn (hidden dependencies):
   - Nhìn vào OrderService không biết nó cần gì
   - Phải đọc toàn bộ code mới biết dependencies
   
2. Khó test:
   - Phải setup ServiceLocator trước mỗi test
   - Tests phụ thuộc vào global state → test order matters
   
3. Runtime errors thay vì compile-time errors:
   - Nếu quên đăng ký service → crash lúc chạy
   - DI Container báo lỗi ngay khi build

4. Tight coupling ẩn:
   - Code vẫn phụ thuộc vào ServiceLocator (một concrete class)
   - Thay đổi cách resolve service → sửa nhiều chỗ
```

### ✅ Thay Bằng Constructor Injection

```csharp
// GOOD: Dependencies rõ ràng, dễ hiểu, dễ test
public class OrderService
{
    private readonly IOrderRepository _orders;
    private readonly IEmailService _email;

    // Constructor injection — DI Container tự động resolve
    public OrderService(IOrderRepository orders, IEmailService email)
    {
        _orders = orders;
        _email = email;
    }

    public async Task PlaceOrderAsync(OrderRequest request)
    {
        await _orders.SaveAsync(new Order(request));
        await _email.SendConfirmationAsync(request.CustomerEmail);
    }
}

// Đăng ký trong DI Container một lần duy nhất
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddTransient<IEmailService, SendGridEmailService>();
builder.Services.AddScoped<OrderService>();
```

### Khi Nào Service Locator Có Thể Chấp Nhận Được

```csharp
// EXCEPTION: Trong một số tình huống hạn chế, Service Locator có thể hợp lý

// 1. Legacy code migration — di chuyển code cũ dần dần
// 2. Infrastructure code không thể inject (e.g., static factory methods)
// 3. Plugin/extension points cần dynamic resolution

// Nếu phải dùng, hãy giới hạn ở composition root — điểm khởi tạo
// và KHÔNG truyền IServiceProvider sâu vào business logic
public class Program
{
    static void Main(string[] args)
    {
        var host = Host.CreateDefaultBuilder(args).Build();
        // OK: Resolve ở composition root (Program.cs / Startup.cs)
        var legacyService = host.Services.GetRequiredService<ILegacyService>();
        legacyService.InitializeSystem();
        host.Run();
    }
}

// NEVER DO: Truyền IServiceProvider vào business logic
public class BadOrderService
{
    private readonly IServiceProvider _sp; // Service Locator disguised!

    public BadOrderService(IServiceProvider sp) => _sp = sp;

    public async Task PlaceOrderAsync(OrderRequest request)
    {
        var repo = _sp.GetRequiredService<IOrderRepository>(); // Service Locator!
    }
}
```

---

## 4. Anemic Domain Model — Mô Hình Miền Thiếu Máu

### Mô Tả

Domain object chỉ chứa data (properties) nhưng không có behavior (business logic). Logic nghiệp vụ bị phân tán vào Service layer, vi phạm OOP và dẫn đến procedural programming trong vỏ OOP.

### ❌ Anemic Domain Model

```csharp
// BAD: Order chỉ là data bag — túi dữ liệu
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = [];
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public string? CancellationReason { get; set; }
}

// Logic nghiệp vụ bị đổ vào OrderService — không phải chỗ của nó
public class OrderService
{
    public void ConfirmOrder(Order order)
    {
        if (order.Status != OrderStatus.Pending)
            throw new InvalidOperationException("Chỉ xác nhận được đơn hàng đang chờ");

        order.Total = order.Items.Sum(i => i.Price * i.Quantity);
        order.Status = OrderStatus.Confirmed;
    }

    public void CancelOrder(Order order, string reason)
    {
        if (order.Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Không thể hủy đơn đã giao");

        order.Status = OrderStatus.Cancelled;
        order.CancellationReason = reason;
    }

    public void ShipOrder(Order order)
    {
        if (order.Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("Chỉ giao được đơn đã xác nhận");

        order.Status = OrderStatus.Shipped;
        order.ShippedAt = DateTime.UtcNow;
    }
    // Logic về Order nằm rải rác ở đây, còn có thể ở OrderValidator, OrderHelper...
}
```

### ✅ Rich Domain Model — Mô Hình Miền Phong Phú

```csharp
// GOOD: Domain object chứa cả data lẫn behavior
public class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ShippedAt { get; private set; }
    public string? CancellationReason { get; private set; }

    // Domain events được raise bên trong domain object
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    // Factory method — tạo Order hợp lệ ngay từ đầu
    public static Order Create(Guid customerId, IEnumerable<OrderItem> items)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        foreach (var item in items) order.AddItem(item);
        return order;
    }

    public void AddItem(OrderItem item)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Không thể thêm sản phẩm vào đơn hàng đã xác nhận");

        _items.Add(item);
        RecalculateTotal();
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Chỉ xác nhận được đơn hàng đang chờ");
        if (!_items.Any())
            throw new DomainException("Đơn hàng phải có ít nhất một sản phẩm");

        Status = OrderStatus.Confirmed;
        _domainEvents.Add(new OrderConfirmedEvent(Id, CustomerId, Total));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Không thể hủy đơn hàng đã giao");

        Status = OrderStatus.Cancelled;
        CancellationReason = reason;
        _domainEvents.Add(new OrderCancelledEvent(Id, reason));
    }

    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
            throw new DomainException("Chỉ giao được đơn hàng đã xác nhận");

        Status = OrderStatus.Shipped;
        ShippedAt = DateTime.UtcNow;
        _domainEvents.Add(new OrderShippedEvent(Id, ShippedAt.Value));
    }

    private void RecalculateTotal()
        => Total = _items.Sum(i => i.Price * i.Quantity);
}

// OrderService bây giờ chỉ orchestrate — điều phối, không chứa business logic
public class OrderService
{
    private readonly IOrderRepository _orders;
    private readonly IDomainEventPublisher _events;

    public async Task ConfirmOrderAsync(Guid orderId)
    {
        var order = await _orders.GetByIdAsync(orderId)
            ?? throw new NotFoundException($"Đơn hàng {orderId} không tồn tại");

        order.Confirm(); // Business logic ở trong domain object
        await _orders.SaveAsync(order);
        await _events.PublishAsync(order.DomainEvents); // publish events
    }
}
```

---

## 5. Magic Numbers & Magic Strings — Số Và Chuỗi Ma Thuật

### Mô Tả

Sử dụng literal values (số hoặc chuỗi trực tiếp) trong code mà không có tên giải thích ý nghĩa.

### ❌ Magic Numbers / Magic Strings

```csharp
// BAD: Các con số và chuỗi không có ngữ nghĩa rõ ràng
public class OrderProcessor
{
    public decimal CalculateFee(Order order)
    {
        if (order.Total > 500000)
            return order.Total * 0.02m; // 2% là gì? Phí gì?

        if (order.Type == 3) // 3 là loại đơn hàng nào?
            return 15000;

        if (order.CustomerLevel == "G") // "G" nghĩa là gì? Gold? Guest?
            return 0;

        return 25000;
    }

    public bool CanCancel(Order order)
        => order.Status != 4 && order.Status != 5; // 4 và 5 là status nào?
}
```

### ✅ Dùng Constants, Enums, và Named Values

```csharp
// GOOD: Tên rõ ràng, tự documenting

// Constants — hằng số có tên
public static class OrderFees
{
    public const decimal FreeShippingThreshold = 500_000m;
    public const decimal StandardShippingFee = 25_000m;
    public const decimal ExpressShippingFee = 50_000m;
    public const decimal PremiumFeeRate = 0.02m; // 2% phí dịch vụ cao cấp
    public const decimal PremiumFeeThreshold = 500_000m;
}

// Enums — liệt kê có tên
public enum OrderType
{
    Standard = 1,
    Express = 2,
    SameDay = 3,
    International = 4
}

public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Processing = 3,
    Shipped = 4,
    Delivered = 5,
    Cancelled = 6,
    Refunded = 7
}

public enum CustomerLevel
{
    Standard,
    Silver,
    Gold,     // trước là "G"
    Platinum
}

// Code bây giờ tự giải thích
public class OrderProcessor
{
    public decimal CalculateFee(Order order)
    {
        if (order.CustomerLevel == CustomerLevel.Gold)
            return 0; // Khách Gold không mất phí vận chuyển

        if (order.Total > OrderFees.PremiumFeeThreshold)
            return order.Total * OrderFees.PremiumFeeRate;

        if (order.Type == OrderType.SameDay)
            return OrderFees.ExpressShippingFee;

        return OrderFees.StandardShippingFee;
    }

    public bool CanCancel(Order order)
        => order.Status != OrderStatus.Shipped
        && order.Status != OrderStatus.Delivered;
}
```

---

## 6. Premature Optimization — Tối Ưu Sớm

### Mô Tả

> "Premature optimization is the root of all evil." — Donald Knuth
> "Tối ưu sớm là gốc rễ của mọi điều xấu."

Tối ưu hiệu năng trước khi đo lường, dựa trên giả định thay vì số liệu thực tế.

### ❌ Premature Optimization

```csharp
// BAD: Tối ưu không cần thiết, làm code khó đọc
public class ProductService
{
    // "Tối ưu" bằng tay thay vì dùng LINQ — khó đọc, lỗi dễ xảy ra
    public List<Product> GetActiveProducts(List<Product> products)
    {
        var result = new List<Product>(products.Count); // pre-allocate
        var len = products.Count; // tránh gọi .Count nhiều lần (không cần thiết với List<T>)

        for (var i = 0; i < len; i++) // tránh foreach vì "overhead" (thực tế không đáng kể)
        {
            ref var p = ref System.Runtime.InteropServices.CollectionsMarshal
                .AsSpan(products)[i]; // unsafe optimization không cần thiết

            if (p.IsActive && p.Stock > 0)
                result.Add(p);
        }

        return result;
    }
}
```

### ✅ Viết Rõ Ràng Trước, Tối Ưu Sau Khi Có Benchmark

```csharp
// GOOD: Rõ ràng, dễ đọc — tối ưu khi benchmark chứng minh cần thiết
public class ProductService
{
    public IEnumerable<Product> GetActiveProducts(IEnumerable<Product> products)
        => products.Where(p => p.IsActive && p.Stock > 0);
}

// CHỈ khi BenchmarkDotNet chứng minh cần tối ưu:
public class OptimizedProductService
{
    public List<Product> GetActiveProducts(List<Product> products)
    {
        var result = new List<Product>();
        var span = CollectionsMarshal.AsSpan(products);
        foreach (ref readonly var product in span)
        {
            if (product.IsActive && product.Stock > 0)
                result.Add(product);
        }
        return result;
    }
}
```

**Quy Trình Đúng:**
```
1. Viết code rõ ràng, đúng đắn
2. Đo lường (profiler, BenchmarkDotNet)
3. Xác định bottleneck thực sự (80% thời gian ở 20% code)
4. Tối ưu chỗ đó — không phải toàn bộ codebase
5. Đo lại để xác nhận cải thiện
```

---

## 7. Shotgun Surgery — Phẫu Thuật Súng Săn

### Mô Tả

Một thay đổi logic nghiệp vụ đòi hỏi sửa nhiều class/file không liên quan. Đây là dấu hiệu logic bị phân tán không đúng chỗ.

### ❌ Nhận Biết Shotgun Surgery

```
Tình huống: Thêm trường "DiscountCode" vào order

Phải sửa:
- OrderRequest (DTO — Data Transfer Object)
- OrderValidator
- OrderMapper
- OrderCalculator
- OrderRepository
- OrderService
- OrderController
- OrderSummaryDto
- CreateOrderCommand
- OrderCreatedEvent
- ... 15 file

Nếu phải sửa > 5 file cho một thay đổi logic đơn giản → Shotgun Surgery
```

### ✅ Cách Khắc Phục

```csharp
// Tập trung logic vào domain object — thay đổi chỉ ở một chỗ
public class Order
{
    public string? DiscountCode { get; private set; }
    private decimal _discountAmount;

    public void ApplyDiscountCode(string code, IDiscountService discountService)
    {
        var discount = discountService.Calculate(code, Total);
        DiscountCode = code;
        _discountAmount = discount;
        RecalculateTotal();
    }
}
// Chỉ sửa Order.cs — domain logic tập trung
```

---

## 🔍 Code Smell Checklist — Danh Sách Kiểm Tra Mùi Code

Khi review code, hỏi những câu này:

### Về Class

- [ ] Class này có thể mô tả mà không dùng từ "và"?
- [ ] Constructor có > 7 parameters?
- [ ] Class có > 500 dòng?
- [ ] Class có `static` methods và mutable static state?

### Về Dependencies

- [ ] Có `new ConcreteClass()` trong business logic không?
- [ ] Có dùng `ServiceLocator.Resolve<T>()` trong business logic không?
- [ ] Có gọi static method của class khác không?

### Về Domain Logic

- [ ] Domain object có methods hay chỉ có properties?
- [ ] Business logic có nằm trong Service layer thay vì domain không?
- [ ] Có magic numbers/strings không?

### Về Coupling

- [ ] Thay đổi Class A có buộc sửa Class B, C, D không?
- [ ] Có `if (obj is ConcreteType)` trong code không?
- [ ] Có circular dependency (A phụ thuộc B, B phụ thuộc A) không?

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: "Service Locator vs Dependency Injection — khác nhau thế nào?"**
→ DI: dependency được inject từ bên ngoài, rõ ràng trong constructor. Service Locator: class tự tìm dependency từ global registry — dependency ẩn. DI dễ test, dễ đọc, dễ thay thế hơn. Service Locator là anti-pattern vì hidden coupling.

**Q: "God Object có ảnh hưởng gì đến team làm việc?"**
→ Mọi developer đều sửa cùng một file → merge conflict liên tục. Khó test riêng từng chức năng. Khó onboard người mới. God Object thường là nguyên nhân chính của "sợ sửa code".

**Q: "Anemic Domain Model có gì sai không? Nhiều team vẫn dùng."**
→ Không sai hoàn toàn — đối với CRUD đơn giản, Anemic là đủ và nhanh hơn. Nhưng khi business logic phức tạp, logic bị phân tán vào nhiều Service → khó theo dõi, dễ bỏ sót validation. Rich Domain Model phù hợp cho bounded context phức tạp, DDD.

**Q: "Làm thế nào phát hiện anti-pattern trong code review?"**
→ Tìm dấu hiệu: class > 500 dòng, constructor > 7 params, `new ConcreteClass()` trong logic, magic numbers, method tên có "And" hoặc "Or", test phải setup global state.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
