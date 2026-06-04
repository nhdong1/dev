# SOLID Principles — Năm Nguyên Lý Thiết Kế Phần Mềm

> SOLID là tập hợp 5 nguyên lý thiết kế hướng đối tượng giúp code dễ bảo trì, dễ mở rộng và dễ kiểm thử. Được đặt tên bởi Robert C. Martin — "Uncle Bob".

---

## 📋 Tổng Quan Nhanh

| Chữ | Tên Đầy Đủ | Một Câu Tóm Tắt |
| --- | ---------- | --------------- |
| **S** | Single Responsibility Principle — Nguyên Lý Trách Nhiệm Đơn | Mỗi class chỉ có một lý do để thay đổi |
| **O** | Open/Closed Principle — Nguyên Lý Mở/Đóng | Mở rộng bằng code mới, không sửa code cũ |
| **L** | Liskov Substitution Principle — Nguyên Lý Thay Thế Liskov | Subclass phải thay thế được base class mà không phá vỡ behavior |
| **I** | Interface Segregation Principle — Nguyên Lý Tách Biệt Giao Diện | Nhiều interface nhỏ tốt hơn một interface lớn |
| **D** | Dependency Inversion Principle — Nguyên Lý Đảo Ngược Phụ Thuộc | Phụ thuộc vào abstraction, không vào concrete implementation |

---

## S — Single Responsibility Principle (SRP) — Nguyên Lý Trách Nhiệm Đơn

### Định Nghĩa

> "A class should have only one reason to change."
> — Một class chỉ nên có một lý do để thay đổi.

**Ý nghĩa thực tế:** Mỗi class chỉ chịu trách nhiệm về một chức năng nghiệp vụ. Nếu bạn phải mô tả class và dùng từ "và" — đó là dấu hiệu vi phạm SRP.

### ❌ Vi Phạm SRP

```csharp
// BAD: UserService làm quá nhiều thứ — xử lý user, gửi email, ghi log
public class UserService
{
    private readonly IDbConnection _db;

    public UserService(IDbConnection db) => _db = db;

    public async Task RegisterAsync(string email, string password)
    {
        // 1. Lưu user vào database
        var passwordHash = BCrypt.HashPassword(password);
        await _db.ExecuteAsync(
            "INSERT INTO Users (Email, PasswordHash) VALUES (@Email, @Hash)",
            new { Email = email, Hash = passwordHash });

        // 2. Gửi email chào mừng (lý do thứ 2 để thay đổi)
        var smtpClient = new SmtpClient("smtp.gmail.com");
        var message = new MailMessage("noreply@app.com", email,
            "Chào mừng!", "Tài khoản của bạn đã được tạo.");
        smtpClient.Send(message);

        // 3. Ghi log (lý do thứ 3 để thay đổi)
        File.AppendAllText("app.log", $"[{DateTime.Now}] User registered: {email}\n");
    }
}
```

**Vấn đề:** Thay đổi cách gửi email (sang SendGrid) buộc phải sửa `UserService`. Thay đổi cách log (sang Serilog) cũng buộc phải sửa `UserService`.

### ✅ Áp Dụng SRP

```csharp
// GOOD: Mỗi class chỉ có một trách nhiệm

public class UserRepository
{
    private readonly IDbConnection _db;
    public UserRepository(IDbConnection db) => _db = db;

    public async Task CreateAsync(string email, string passwordHash)
    {
        await _db.ExecuteAsync(
            "INSERT INTO Users (Email, PasswordHash) VALUES (@Email, @Hash)",
            new { Email = email, Hash = passwordHash });
    }
}

public class EmailService
{
    private readonly ISmtpClient _smtp;
    public EmailService(ISmtpClient smtp) => _smtp = smtp;

    public async Task SendWelcomeEmailAsync(string email)
    {
        await _smtp.SendAsync(new EmailMessage
        {
            To = email,
            Subject = "Chào mừng!",
            Body = "Tài khoản của bạn đã được tạo."
        });
    }
}

public class UserRegistrationService
{
    private readonly UserRepository _users;
    private readonly EmailService _email;
    private readonly ILogger<UserRegistrationService> _logger;

    public UserRegistrationService(
        UserRepository users,
        EmailService email,
        ILogger<UserRegistrationService> logger)
    {
        _users = users;
        _email = email;
        _logger = logger;
    }

    public async Task RegisterAsync(string email, string password)
    {
        var passwordHash = BCrypt.HashPassword(password);
        await _users.CreateAsync(email, passwordHash);
        await _email.SendWelcomeEmailAsync(email);
        _logger.LogInformation("User registered: {Email}", email);
    }
}
```

### Dấu Hiệu Vi Phạm SRP

- Class có quá 200–300 dòng
- Method có quá 30–50 dòng
- Class có quá 5–7 dependencies (constructor injection)
- Class import nhiều namespace không liên quan

---

## O — Open/Closed Principle (OCP) — Nguyên Lý Mở/Đóng

### Định Nghĩa

> "Software entities should be open for extension, but closed for modification."
> — Thực thể phần mềm nên mở để mở rộng, nhưng đóng để sửa đổi.

**Ý nghĩa thực tế:** Thêm tính năng mới bằng cách viết code mới, không sửa code đang hoạt động tốt.

### ❌ Vi Phạm OCP

```csharp
// BAD: Mỗi lần thêm discount type phải sửa DiscountCalculator
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        if (discountType == "student")
            return order.Total * 0.10m;

        if (discountType == "employee")
            return order.Total * 0.20m;

        if (discountType == "senior")    // thêm type mới → sửa class này
            return order.Total * 0.15m;

        // thêm "vip", "seasonal"... mãi phải sửa đây
        return 0;
    }
}
```

### ✅ Áp Dụng OCP

```csharp
// GOOD: Thêm discount type mới chỉ cần tạo class mới

public interface IDiscountStrategy
{
    decimal Calculate(Order order);
}

public class StudentDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.10m;
}

public class EmployeeDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.20m;
}

public class SeniorDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) => order.Total * 0.15m;
}

// Thêm VipDiscount → chỉ tạo class mới, không sửa gì cả
public class VipDiscount : IDiscountStrategy
{
    public decimal Calculate(Order order) =>
        order.Total > 1_000_000 ? order.Total * 0.25m : order.Total * 0.15m;
}

public class DiscountCalculator
{
    private readonly IDiscountStrategy _strategy;

    public DiscountCalculator(IDiscountStrategy strategy) => _strategy = strategy;

    public decimal Calculate(Order order) => _strategy.Calculate(order);
}
```

**Đây chính là Strategy Pattern — Mẫu Chiến Lược** — xem chi tiết trong [4-behavioral-patterns.md](./4-behavioral-patterns.md).

### Cơ Chế Triển Khai OCP

- **Polymorphism — Đa hình**: interface + override
- **Strategy Pattern**: hoán đổi algorithm khi chạy
- **Template Method Pattern**: định nghĩa skeleton, để subclass điền chi tiết
- **Decorator Pattern**: thêm behavior mà không sửa class gốc

---

## L — Liskov Substitution Principle (LSP) — Nguyên Lý Thay Thế Liskov

### Định Nghĩa

> "Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."
> — Đối tượng của superclass phải có thể được thay thế bằng đối tượng của subclass mà không làm hỏng chương trình.

**Phát biểu kỹ thuật:** Nếu `S` là subtype của `T`, thì mọi chỗ dùng `T` đều phải dùng được `S` mà không cần biết đó là `S`.

### ❌ Vi Phạm LSP — Ví Dụ Kinh Điển

```csharp
// BAD: Hình vuông (Square) kế thừa hình chữ nhật (Rectangle) — vi phạm LSP
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    // Hình vuông buộc Width == Height
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // thay đổi Height!
    }

    public override int Height
    {
        get => base.Height;
        set { base.Width = value; base.Height = value; } // thay đổi Width!
    }
}

// Code này hợp lý với Rectangle nhưng thất bại với Square
void SetDimensions(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    Console.WriteLine(rect.Area()); // Mong đợi: 50, Thực tế với Square: 100 (!)
}
```

### ✅ Áp Dụng LSP

```csharp
// GOOD: Dùng interface chung, không dùng inheritance sai ngữ nghĩa
public interface IShape
{
    double Area();
}

public class Rectangle : IShape
{
    public int Width { get; init; }
    public int Height { get; init; }
    public double Area() => Width * Height;
}

public class Square : IShape
{
    public int Side { get; init; }
    public double Area() => Side * Side;
}

// Hoạt động đúng với mọi IShape
void PrintArea(IShape shape)
{
    Console.WriteLine($"Diện tích: {shape.Area()}");
}
```

### Vi Phạm LSP Thường Gặp Trong .NET

```csharp
// BAD: Subclass ném exception cho method mà base class không ném
public abstract class FileStorage
{
    public abstract void Save(byte[] data);
    public abstract byte[] Load(string path);
}

public class ReadOnlyStorage : FileStorage
{
    public override void Save(byte[] data)
        => throw new NotSupportedException("Không thể ghi vào storage chỉ đọc"); // Vi phạm!

    public override byte[] Load(string path) => File.ReadAllBytes(path);
}

// Code dùng FileStorage bị bất ngờ với ReadOnlyStorage
void ProcessFiles(FileStorage storage)
{
    var data = storage.Load("input.bin");
    // xử lý...
    storage.Save(data); // crash runtime với ReadOnlyStorage!
}
```

**Cách nhận biết vi phạm LSP:**
- Subclass có method ném `NotSupportedException` hoặc `NotImplementedException`
- Code dùng `if (obj is SubClass)` để xử lý khác nhau
- Subclass override method nhưng "làm yếu" điều kiện trả về (precondition / postcondition)

---

## I — Interface Segregation Principle (ISP) — Nguyên Lý Tách Biệt Giao Diện

### Định Nghĩa

> "Clients should not be forced to depend on interfaces they do not use."
> — Client không nên bị ép phụ thuộc vào interface mà nó không dùng.

**Ý nghĩa:** Chia interface lớn thành nhiều interface nhỏ, mỗi cái phục vụ một mục đích cụ thể. Client chỉ implement những gì nó thực sự cần.

### ❌ Vi Phạm ISP

```csharp
// BAD: Interface quá "béo" — Fat Interface
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void TakeBreak();
}

// Robot không cần Eat, Sleep, TakeBreak nhưng vẫn phải implement
public class Robot : IWorker
{
    public void Work() { /* làm việc */ }
    public void Eat() => throw new NotImplementedException(); // vô nghĩa!
    public void Sleep() => throw new NotImplementedException(); // vô nghĩa!
    public void TakeBreak() => throw new NotImplementedException(); // vô nghĩa!
}
```

### ✅ Áp Dụng ISP

```csharp
// GOOD: Tách thành interface nhỏ, có mục đích rõ ràng
public interface IWorkable
{
    void Work();
}

public interface IHuman
{
    void Eat();
    void Sleep();
    void TakeBreak();
}

public class HumanWorker : IWorkable, IHuman
{
    public void Work() { /* làm việc */ }
    public void Eat() { /* ăn cơm */ }
    public void Sleep() { /* ngủ */ }
    public void TakeBreak() { /* nghỉ ngơi */ }
}

public class Robot : IWorkable
{
    public void Work() { /* làm việc liên tục */ }
    // không bị ép implement Eat/Sleep/TakeBreak
}
```

### Ví Dụ Thực Tế Trong .NET

```csharp
// ASP.NET Core tách IHostedService thành các interface nhỏ
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}

// IHostedLifecycleService thêm các hook lifecycle chi tiết hơn
public interface IHostedLifecycleService : IHostedService
{
    Task StartingAsync(CancellationToken cancellationToken);
    Task StartedAsync(CancellationToken cancellationToken);
    Task StoppingAsync(CancellationToken cancellationToken);
    Task StoppedAsync(CancellationToken cancellationToken);
}

// Service đơn giản chỉ cần IHostedService
public class SimpleBackgroundService : IHostedService
{
    public Task StartAsync(CancellationToken ct) { /* bắt đầu */ return Task.CompletedTask; }
    public Task StopAsync(CancellationToken ct) { /* dừng */ return Task.CompletedTask; }
}

// Service cần control lifecycle chi tiết mới implement IHostedLifecycleService
```

### Repository Pattern và ISP

```csharp
// GOOD: Tách read và write — chuẩn bị cho CQRS
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
}

public interface IWriteRepository<T>
{
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(Guid id);
}

// Read-only cache repository chỉ cần IReadRepository
public class CachedProductRepository : IReadRepository<Product>
{
    private readonly IMemoryCache _cache;
    private readonly IReadRepository<Product> _inner;

    public async Task<Product?> GetByIdAsync(Guid id)
    {
        return await _cache.GetOrCreateAsync(id, _ => _inner.GetByIdAsync(id));
    }

    public Task<IEnumerable<Product>> GetAllAsync() => _inner.GetAllAsync();
}
```

---

## D — Dependency Inversion Principle (DIP) — Nguyên Lý Đảo Ngược Phụ Thuộc

### Định Nghĩa

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."
> — Module cấp cao không nên phụ thuộc vào module cấp thấp. Cả hai nên phụ thuộc vào abstraction.

> "Abstractions should not depend on details. Details should depend on abstractions."
> — Abstraction không nên phụ thuộc vào chi tiết. Chi tiết nên phụ thuộc vào abstraction.

### ❌ Vi Phạm DIP

```csharp
// BAD: OrderService phụ thuộc trực tiếp vào SqlOrderRepository (cụ thể)
public class SqlOrderRepository
{
    public void Save(Order order) { /* lưu vào SQL Server */ }
}

public class OrderService
{
    // Phụ thuộc vào implementation cụ thể — vi phạm DIP
    private readonly SqlOrderRepository _repository = new SqlOrderRepository();

    public void PlaceOrder(Order order)
    {
        // validate...
        _repository.Save(order);
    }
}

// Hệ quả: Muốn đổi sang PostgreSQL hay MongoDB → phải sửa OrderService
// Không thể viết unit test không cần database thật
```

### ✅ Áp Dụng DIP

```csharp
// GOOD: Cả hai phụ thuộc vào abstraction (interface)

// Abstraction — định nghĩa contract
public interface IOrderRepository
{
    Task SaveAsync(Order order);
    Task<Order?> GetByIdAsync(Guid id);
}

// Low-level module — chi tiết cài đặt
public class SqlOrderRepository : IOrderRepository
{
    private readonly DbContext _db;
    public SqlOrderRepository(DbContext db) => _db = db;

    public async Task SaveAsync(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
    }

    public async Task<Order?> GetByIdAsync(Guid id)
        => await _db.Orders.FindAsync(id);
}

// High-level module — nghiệp vụ
public class OrderService
{
    private readonly IOrderRepository _repository; // phụ thuộc vào abstraction
    private readonly IEventPublisher _events;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        IEventPublisher events,
        ILogger<OrderService> logger)
    {
        _repository = repository;
        _events = events;
        _logger = logger;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        order.Validate();
        await _repository.SaveAsync(order);
        await _events.PublishAsync(new OrderPlacedEvent(order.Id));
        _logger.LogInformation("Order placed: {OrderId}", order.Id);
    }
}
```

**DIP chính là nền tảng của Dependency Injection — DI — Tiêm Phụ Thuộc** trong ASP.NET Core:

```csharp
// Program.cs — ASP.NET Core DI Container đăng ký implementation
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IEventPublisher, RabbitMqEventPublisher>();

// OrderService nhận implementation qua constructor injection
// Có thể swap SqlOrderRepository → MongoOrderRepository không cần sửa OrderService
```

### DIP và Testability — Khả Năng Kiểm Thử

```csharp
// Nhờ DIP, có thể mock (giả lập) dependency trong unit test
[Fact]
public async Task PlaceOrder_ShouldPublishEvent()
{
    // Arrange
    var mockRepo = new Mock<IOrderRepository>();
    var mockEvents = new Mock<IEventPublisher>();
    var mockLogger = new Mock<ILogger<OrderService>>();

    var service = new OrderService(mockRepo.Object, mockEvents.Object, mockLogger.Object);
    var order = new Order { Id = Guid.NewGuid() };

    // Act
    await service.PlaceOrderAsync(order);

    // Assert
    mockEvents.Verify(e => e.PublishAsync(It.IsAny<OrderPlacedEvent>()), Times.Once);
    // Không cần database thật, không cần RabbitMQ thật!
}
```

---

## 🔄 SOLID Trong Thực Tế — Các Tình Huống Hay Gặp

### Tình Huống 1: Thêm Phương Thức Thanh Toán Mới

```
Vi phạm OCP nếu: phải sửa PaymentService mỗi khi thêm Stripe, VNPay, MoMo...
Giải pháp: IPaymentProvider interface + Strategy Pattern
→ Mỗi provider là một class riêng biệt
```

### Tình Huống 2: Gửi Thông Báo Qua Nhiều Kênh

```
Vi phạm ISP nếu: INotificationService có SendEmail + SendSms + SendPush và class nào cũng phải implement hết
Giải pháp: IEmailNotification, ISmsNotification, IPushNotification riêng
```

### Tình Huống 3: Thay Đổi Database

```
Vi phạm DIP nếu: Service gọi trực tiếp new SqlConnection(...)
Giải pháp: IDbRepository interface + DI Container quản lý lifetime
```

### Tình Huống 4: Class "Thần Thánh" Làm Mọi Thứ

```
Vi phạm SRP nếu: UserManager xử lý đăng nhập, gửi email, ghi log, tính phí...
Giải pháp: Tách thành AuthService, EmailService, AuditLogger, BillingService
```

---

## 📝 Checklist SOLID — Tự Đánh Giá

### SRP — Trách Nhiệm Đơn

- [ ] Class này có thể mô tả trong một câu không dùng từ "và"?
- [ ] Class có ít hơn 5–7 dependencies không?
- [ ] Method ngắn hơn 30 dòng?

### OCP — Mở/Đóng

- [ ] Thêm tính năng mới có cần sửa class hiện tại không? (Không = tốt)
- [ ] Dùng interface/abstract class thay vì if/else cho behavior khác nhau?

### LSP — Thay Thế Liskov

- [ ] Subclass có method ném `NotSupportedException` không? (Có = vi phạm)
- [ ] Code dùng `is`/`as` để xử lý khác nhau theo type? (Có = vi phạm)

### ISP — Tách Biệt Giao Diện

- [ ] Interface có method nào mà implementor không dùng không? (Có = vi phạm)
- [ ] Interface nhỏ hơn 5–7 method?

### DIP — Đảo Ngược Phụ Thuộc

- [ ] Constructor nhận interface thay vì concrete class?
- [ ] Không có `new ConcreteClass()` trong business logic?
- [ ] Có thể mock tất cả dependency trong unit test?

---

## 🎯 Câu Hỏi Phỏng Vấn & Gợi Ý Trả Lời

**Q: "Nguyên lý SOLID nào bạn áp dụng nhiều nhất và trong tình huống nào?"**

Gợi ý: Chọn DIP (vì liên quan đến DI Container mà mọi .NET project đều dùng), kể câu chuyện STAR:
- Situation: Project ban đầu new SqlConnection() trực tiếp trong service
- Task: Cần viết unit test nhưng không thể mock database
- Action: Tạo IRepository interface, inject qua constructor, dùng Moq trong test
- Result: Test coverage tăng 40%, refactor database layer không ảnh hưởng business logic

**Q: "Cho đoạn code này — vi phạm nguyên lý gì?"**

Checklist nhận biết:
1. Class làm nhiều thứ → SRP
2. `switch`/`if` trên type → OCP
3. `throw NotSupportedException()` trong override → LSP
4. Interface implement phải có method không dùng → ISP
5. `new ConcreteClass()` trong business logic → DIP

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
