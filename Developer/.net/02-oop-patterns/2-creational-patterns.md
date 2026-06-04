# Creational Patterns — Mẫu Thiết Kế Khởi Tạo

> Creational Patterns — Mẫu Khởi Tạo — giải quyết vấn đề **tạo đối tượng** một cách linh hoạt, che giấu logic khởi tạo phức tạp và giảm phụ thuộc vào class cụ thể.

---

## 📋 Tổng Quan

| Pattern | Mục Đích | Khi Nào Dùng | .NET Ví Dụ |
| ------- | -------- | ------------ | ----------- |
| **Singleton** | Đảm bảo chỉ có 1 instance | Resource dùng chung toàn hệ thống | `IConfiguration`, connection pool |
| **Factory Method** | Để subclass quyết định tạo class nào | Cần override logic khởi tạo | `ILoggerFactory`, `DbProviderFactory` |
| **Abstract Factory** | Tạo nhóm đối tượng liên quan | Nhiều "gia đình" object cùng theme | UI theme, cross-platform |
| **Builder** | Xây dựng object phức tạp từng bước | Object có nhiều tham số optional | `IHostBuilder`, `SqlConnectionStringBuilder` |
| **Prototype** | Clone object thay vì tạo mới | Tạo object tốn kém | Deep copy, object template |

---

## 1. Singleton Pattern — Mẫu Đơn Thể

### Định Nghĩa

> Đảm bảo một class chỉ có **một instance duy nhất** trong toàn bộ vòng đời ứng dụng và cung cấp điểm truy cập toàn cục (global access point) đến instance đó.

### Khi Nào Dùng

- Configuration manager — Quản lý cấu hình ứng dụng
- Connection pool — Bể kết nối database
- Logger — Ghi log
- Cache — Bộ nhớ đệm toàn cục
- Event bus — Xe buýt sự kiện

### ❌ Singleton Không Thread-Safe

```csharp
// BAD: Trong môi trường đa luồng, có thể tạo nhiều instance
public class BadSingleton
{
    private static BadSingleton? _instance;

    private BadSingleton() { }

    public static BadSingleton Instance
    {
        get
        {
            if (_instance == null) // race condition — điều kiện tranh đua ở đây!
                _instance = new BadSingleton();
            return _instance;
        }
    }
}
```

### ✅ Singleton Thread-Safe — Cách 1: Lazy<T>

```csharp
// GOOD: Lazy<T> đảm bảo thread-safe và khởi tạo lười biếng (lazy initialization)
public sealed class AppConfiguration
{
    private static readonly Lazy<AppConfiguration> _lazy =
        new(() => new AppConfiguration());

    public static AppConfiguration Instance => _lazy.Value;

    public string DatabaseConnectionString { get; private set; }
    public int MaxRetryCount { get; private set; }

    private AppConfiguration()
    {
        DatabaseConnectionString = Environment.GetEnvironmentVariable("DB_CONNECTION")
            ?? throw new InvalidOperationException("DB_CONNECTION chưa được cấu hình");
        MaxRetryCount = 3;
    }
}

// Sử dụng
var config = AppConfiguration.Instance;
Console.WriteLine(config.DatabaseConnectionString);
```

### ✅ Singleton Thread-Safe — Cách 2: Static Constructor

```csharp
// Static constructor được CLR đảm bảo chỉ chạy một lần, thread-safe
public sealed class EventAggregator
{
    private static readonly EventAggregator _instance = new EventAggregator();

    // Ngăn CLR đánh dấu type là beforefieldinit
    static EventAggregator() { }

    public static EventAggregator Instance => _instance;

    private readonly List<object> _handlers = [];

    private EventAggregator() { }

    public void Subscribe<T>(Action<T> handler) => _handlers.Add(handler);
}
```

### Singleton Trong ASP.NET Core DI

```csharp
// GOOD: Trong .NET, nên dùng DI Container thay vì Singleton tự làm
// DI Container quản lý lifetime tốt hơn và dễ test hơn
builder.Services.AddSingleton<IConfiguration>(...); // tạo 1 lần dùng suốt app
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IEventBus, RabbitMqEventBus>();

// Scoped — tạo mới mỗi HTTP request
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();

// Transient — tạo mới mỗi lần inject
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
```

### Cảnh Báo: Singleton Pitfalls — Bẫy Singleton

```csharp
// NGUY HIỂM: Singleton chứa Scoped service → memory leak và data leak
public class MySingleton
{
    private readonly IDbContext _db; // Scoped service trong Singleton → SAI!

    public MySingleton(IDbContext db) => _db = db; // db bị "capture" mãi mãi
}

// Giải pháp: Inject IServiceScopeFactory thay vì Scoped service trực tiếp
public class MySingleton
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MySingleton(IServiceScopeFactory factory) => _scopeFactory = factory;

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<IDbContext>();
        // dùng db trong scope ngắn, không giữ mãi
    }
}
```

---

## 2. Factory Method Pattern — Mẫu Phương Thức Nhà Máy

### Định Nghĩa

> Định nghĩa interface để tạo đối tượng, nhưng để các subclass quyết định class nào sẽ được khởi tạo. Factory Method — Phương Thức Nhà Máy — cho phép class trì hoãn việc khởi tạo đến subclass.

### Khi Nào Dùng

- Khi class không biết trước loại object nào cần tạo
- Khi subclass cần kiểm soát logic khởi tạo
- Khi muốn tập trung logic khởi tạo phức tạp

### ✅ Triển Khai Factory Method

```csharp
// Product — sản phẩm: interface cho đối tượng được tạo
public interface INotification
{
    Task SendAsync(string recipient, string message);
}

// Concrete Products — sản phẩm cụ thể
public class EmailNotification : INotification
{
    private readonly string _smtpHost;
    public EmailNotification(string smtpHost) => _smtpHost = smtpHost;

    public async Task SendAsync(string recipient, string message)
    {
        Console.WriteLine($"[Email → {recipient}] {message} qua {_smtpHost}");
        await Task.Delay(100); // giả lập gửi email
    }
}

public class SmsNotification : INotification
{
    private readonly string _apiKey;
    public SmsNotification(string apiKey) => _apiKey = apiKey;

    public async Task SendAsync(string recipient, string message)
    {
        Console.WriteLine($"[SMS → {recipient}] {message}");
        await Task.Delay(50);
    }
}

public class SlackNotification : INotification
{
    private readonly string _webhookUrl;
    public SlackNotification(string webhookUrl) => _webhookUrl = webhookUrl;

    public async Task SendAsync(string recipient, string message)
    {
        Console.WriteLine($"[Slack → #{recipient}] {message}");
        await Task.Delay(80);
    }
}

// Creator — lớp khởi tạo: chứa factory method
public abstract class NotificationService
{
    // Factory Method — phương thức nhà máy: subclass override để tạo loại cụ thể
    protected abstract INotification CreateNotification();

    // Template method dùng factory method
    public async Task NotifyAsync(string recipient, string message)
    {
        var notification = CreateNotification(); // gọi factory method
        await notification.SendAsync(recipient, message);
    }
}

// Concrete Creators — lớp khởi tạo cụ thể
public class EmailNotificationService : NotificationService
{
    private readonly IConfiguration _config;
    public EmailNotificationService(IConfiguration config) => _config = config;

    protected override INotification CreateNotification()
        => new EmailNotification(_config["Smtp:Host"]!);
}

public class SmsNotificationService : NotificationService
{
    private readonly IConfiguration _config;
    public SmsNotificationService(IConfiguration config) => _config = config;

    protected override INotification CreateNotification()
        => new SmsNotification(_config["Sms:ApiKey"]!);
}

// Sử dụng
NotificationService service = new EmailNotificationService(config);
await service.NotifyAsync("user@example.com", "Đơn hàng của bạn đã được xác nhận");
```

### Simple Factory — Nhà Máy Đơn Giản (Không Phải GoF)

```csharp
// Simple Factory — không phải pattern GoF chính thức nhưng rất phổ biến
public static class NotificationFactory
{
    public static INotification Create(string channel, IConfiguration config) =>
        channel.ToLower() switch
        {
            "email" => new EmailNotification(config["Smtp:Host"]!),
            "sms"   => new SmsNotification(config["Sms:ApiKey"]!),
            "slack" => new SlackNotification(config["Slack:WebhookUrl"]!),
            _ => throw new ArgumentException($"Kênh thông báo không hỗ trợ: {channel}")
        };
}

// Sử dụng
var notification = NotificationFactory.Create("email", config);
await notification.SendAsync("user@example.com", "Xin chào!");
```

---

## 3. Abstract Factory Pattern — Mẫu Nhà Máy Trừu Tượng

### Định Nghĩa

> Cung cấp interface để tạo **gia đình các đối tượng liên quan** mà không chỉ định các class cụ thể. Abstract Factory — Nhà Máy Trừu Tượng — đảm bảo các đối tượng được tạo ra tương thích với nhau.

### Sự Khác Nhau Với Factory Method

| | Factory Method | Abstract Factory |
| - | - | - |
| **Tạo** | Một loại product | Gia đình các product liên quan |
| **Cách** | Subclass override method | Interface nhà máy riêng |
| **Dùng khi** | Không biết loại cụ thể | Cần đảm bảo tính tương thích |

### ✅ Triển Khai Abstract Factory

```csharp
// Ví dụ: Tạo UI components cho nhiều nền tảng (Windows, macOS, Web)

// Abstract Products — sản phẩm trừu tượng
public interface IButton
{
    string Render();
    void Click();
}

public interface ITextBox
{
    string Render();
    string GetValue();
}

public interface IDialog
{
    string Render();
}

// Concrete Products — Windows
public class WindowsButton : IButton
{
    public string Render() => "<button class='win-btn'>Win Button</button>";
    public void Click() => Console.WriteLine("Windows button clicked");
}

public class WindowsTextBox : ITextBox
{
    private string _value = string.Empty;
    public string Render() => $"<input type='text' class='win-input' value='{_value}'/>";
    public string GetValue() => _value;
}

// Concrete Products — macOS
public class MacButton : IButton
{
    public string Render() => "<button class='mac-btn'>Mac Button</button>";
    public void Click() => Console.WriteLine("macOS button clicked");
}

public class MacTextBox : ITextBox
{
    private string _value = string.Empty;
    public string Render() => $"<input type='text' class='mac-input' value='{_value}'/>";
    public string GetValue() => _value;
}

// Abstract Factory — nhà máy trừu tượng
public interface IUiFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
}

// Concrete Factories — nhà máy cụ thể
public class WindowsUiFactory : IUiFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
}

public class MacUiFactory : IUiFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
}

// Client — sử dụng factory mà không biết implementation cụ thể
public class LoginForm
{
    private readonly IButton _loginButton;
    private readonly ITextBox _usernameInput;
    private readonly ITextBox _passwordInput;

    public LoginForm(IUiFactory factory)
    {
        _loginButton = factory.CreateButton();
        _usernameInput = factory.CreateTextBox();
        _passwordInput = factory.CreateTextBox();
    }

    public string Render()
    {
        return $"""
            <form>
                {_usernameInput.Render()}
                {_passwordInput.Render()}
                {_loginButton.Render()}
            </form>
            """;
    }
}

// Sử dụng: chỉ cần đổi factory để thay đổi toàn bộ look & feel
IUiFactory factory = RuntimeInformation.IsOSPlatform(OSPlatform.OSX)
    ? new MacUiFactory()
    : new WindowsUiFactory();

var form = new LoginForm(factory);
Console.WriteLine(form.Render());
```

---

## 4. Builder Pattern — Mẫu Xây Dựng

### Định Nghĩa

> Tách rời quá trình xây dựng một đối tượng phức tạp khỏi biểu diễn của nó, cho phép cùng một quá trình xây dựng tạo ra các biểu diễn khác nhau.

### Khi Nào Dùng

- Object có quá nhiều tham số constructor (Telescoping Constructor — Constructor Ống Kính Thiên Văn)
- Có nhiều bước xây dựng tùy chọn
- Cần tạo nhiều biến thể của cùng một object
- Muốn đảm bảo object hợp lệ trước khi trả về

### ❌ Telescoping Constructor — Vấn Đề Constructor Nhiều Tham Số

```csharp
// BAD: Khó đọc, dễ nhầm thứ tự tham số
public class Email
{
    public Email(string to, string from, string subject, string body,
                 string? cc = null, string? bcc = null, bool isHtml = false,
                 int priority = 0, List<string>? attachments = null) { }
}

// Rất khó đọc và dễ nhầm
var email = new Email(
    "user@example.com", "noreply@app.com",
    "Xác nhận đơn hàng", "<h1>Cảm ơn!</h1>",
    null, null, true, 1, null);
```

### ✅ Builder Pattern — Fluent API

```csharp
// GOOD: Builder với Fluent API — Giao diện lưu loát
public class Email
{
    public string To { get; private set; } = string.Empty;
    public string From { get; private set; } = string.Empty;
    public string Subject { get; private set; } = string.Empty;
    public string Body { get; private set; } = string.Empty;
    public string? Cc { get; private set; }
    public string? Bcc { get; private set; }
    public bool IsHtml { get; private set; }
    public int Priority { get; private set; }
    public List<string> Attachments { get; private set; } = [];

    private Email() { }

    public class Builder
    {
        private readonly Email _email = new();

        public Builder To(string to)
        {
            if (string.IsNullOrWhiteSpace(to))
                throw new ArgumentException("Địa chỉ người nhận không được rỗng");
            _email.To = to;
            return this;
        }

        public Builder From(string from)
        {
            _email.From = from;
            return this;
        }

        public Builder WithSubject(string subject)
        {
            _email.Subject = subject;
            return this;
        }

        public Builder WithBody(string body, bool isHtml = false)
        {
            _email.Body = body;
            _email.IsHtml = isHtml;
            return this;
        }

        public Builder Cc(string cc) { _email.Cc = cc; return this; }
        public Builder Bcc(string bcc) { _email.Bcc = bcc; return this; }
        public Builder WithPriority(int priority) { _email.Priority = priority; return this; }

        public Builder AddAttachment(string filePath)
        {
            _email.Attachments.Add(filePath);
            return this;
        }

        public Email Build()
        {
            // Validation tập trung — đảm bảo object hợp lệ
            if (string.IsNullOrWhiteSpace(_email.To))
                throw new InvalidOperationException("Phải có địa chỉ người nhận");
            if (string.IsNullOrWhiteSpace(_email.Subject))
                throw new InvalidOperationException("Phải có tiêu đề email");

            return _email;
        }
    }
}

// Sử dụng — rất rõ ràng, dễ đọc
var email = new Email.Builder()
    .To("user@example.com")
    .From("noreply@app.com")
    .WithSubject("Xác nhận đơn hàng #12345")
    .WithBody("<h1>Cảm ơn bạn đã đặt hàng!</h1>", isHtml: true)
    .Cc("manager@app.com")
    .WithPriority(1)
    .AddAttachment("/invoices/12345.pdf")
    .Build();
```

### Builder Trong .NET — IHostBuilder và WebApplicationBuilder

```csharp
// ASP.NET Core dùng Builder Pattern cho toàn bộ configuration
var builder = WebApplication.CreateBuilder(args);

// Thêm từng thành phần một cách rõ ràng
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* cấu hình JWT */ });

// Build — tạo app với tất cả configuration đã đăng ký
var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

### Builder Với Record — Immutable Builder

```csharp
// C# 9+ record + with expression là lightweight alternative
public record QueryOptions
{
    public int PageNumber { get; init; } = 1;
    public int PageSize { get; init; } = 20;
    public string? SortBy { get; init; }
    public bool SortDescending { get; init; } = false;
    public string? SearchTerm { get; init; }
}

// Sử dụng with expression — tạo bản sao với thay đổi nhỏ
var defaultQuery = new QueryOptions();
var sortedQuery = defaultQuery with { SortBy = "CreatedAt", SortDescending = true };
var pagedQuery = sortedQuery with { PageNumber = 2, PageSize = 10 };
```

---

## 5. Prototype Pattern — Mẫu Nguyên Mẫu

### Định Nghĩa

> Tạo đối tượng mới bằng cách **sao chép (clone)** đối tượng hiện có thay vì tạo từ đầu. Hữu ích khi quá trình khởi tạo tốn kém hoặc phức tạp.

### Shallow Copy vs Deep Copy — Sao Chép Nông vs Sao Chép Sâu

```csharp
// Shallow Copy — Sao Chép Nông: chỉ copy reference, không copy object bên trong
public class ShallowExample
{
    public List<string> Tags { get; set; } = [];
    public string Name { get; set; } = string.Empty;

    public ShallowExample ShallowClone() => (ShallowExample)MemberwiseClone();
}

var original = new ShallowExample { Name = "Original", Tags = ["A", "B"] };
var copy = original.ShallowClone();

copy.Name = "Copy";        // OK: string là immutable
copy.Tags.Add("C");        // NGUY HIỂM: thay đổi Tags của original luôn!

Console.WriteLine(original.Tags.Count); // 3 — bị ảnh hưởng!
```

```csharp
// Deep Copy — Sao Chép Sâu: tạo bản sao hoàn toàn độc lập
public interface IPrototype<T>
{
    T DeepClone();
}

public class DocumentTemplate : IPrototype<DocumentTemplate>
{
    public string Title { get; set; } = string.Empty;
    public List<string> Sections { get; set; } = [];
    public Dictionary<string, string> Metadata { get; set; } = [];

    public DocumentTemplate DeepClone()
    {
        return new DocumentTemplate
        {
            Title = Title, // string immutable — ok
            Sections = new List<string>(Sections), // tạo list mới
            Metadata = new Dictionary<string, string>(Metadata) // tạo dict mới
        };
    }
}

// Sử dụng
var template = new DocumentTemplate
{
    Title = "Hợp đồng Mẫu",
    Sections = ["Điều 1", "Điều 2", "Điều 3"],
    Metadata = { ["Version"] = "1.0", ["Author"] = "Legal Dept" }
};

var contract = template.DeepClone();
contract.Title = "Hợp đồng Thuê Nhà #001";
contract.Sections.Add("Điều 4");

Console.WriteLine(template.Title);          // "Hợp đồng Mẫu" — không đổi
Console.WriteLine(template.Sections.Count); // 3 — không đổi
```

### Prototype Với JSON Serialization

```csharp
// Deep clone bằng JSON serialization — cách phổ biến trong .NET
public static class CloneExtensions
{
    public static T DeepClone<T>(this T obj)
    {
        var json = JsonSerializer.Serialize(obj);
        return JsonSerializer.Deserialize<T>(json)!;
    }
}

// Sử dụng
var original = new ComplexOrder { /* ... nhiều nested object */ };
var clonedOrder = original.DeepClone();
clonedOrder.OrderNumber = "ORD-2024-002";
// original không bị thay đổi
```

### Ứng Dụng Thực Tế: Report Template

```csharp
public class ReportConfig
{
    public string ReportName { get; set; } = string.Empty;
    public List<string> Columns { get; set; } = [];
    public FilterConfig Filters { get; set; } = new();
    public PaginationConfig Pagination { get; set; } = new();

    // Tạo biến thể từ template
    public static ReportConfig CreateMonthlyReport(ReportConfig baseTemplate)
    {
        var monthly = baseTemplate.DeepClone();
        monthly.ReportName = $"Monthly_{DateTime.Now:yyyy_MM}";
        monthly.Filters.DateRange = DateRange.ThisMonth;
        return monthly;
    }
}

var baseTemplate = new ReportConfig
{
    ReportName = "Sales Report",
    Columns = ["Date", "Customer", "Amount", "Status"],
    Pagination = new PaginationConfig { PageSize = 100 }
};

var januaryReport = ReportConfig.CreateMonthlyReport(baseTemplate);
var februaryReport = ReportConfig.CreateMonthlyReport(baseTemplate);
```

---

## 🔄 So Sánh Các Creational Patterns

| Câu Hỏi | Pattern Phù Hợp |
| -------- | --------------- |
| "Cần đảm bảo chỉ có 1 instance?" | **Singleton** |
| "Cần tạo object nhưng không biết loại cụ thể?" | **Factory Method** |
| "Cần tạo nhóm object liên quan, tương thích nhau?" | **Abstract Factory** |
| "Object có quá nhiều tham số tùy chọn?" | **Builder** |
| "Cần copy object phức tạp nhanh?" | **Prototype** |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: "Singleton trong multi-threaded environment — làm thế nào để thread-safe?"**
→ Dùng `Lazy<T>` (khuyến nghị) hoặc static constructor. Tránh double-checked locking thủ công.

**Q: "Factory Method vs Abstract Factory — khác nhau chỗ nào?"**
→ Factory Method: một method tạo một product. Abstract Factory: một interface tạo gia đình products liên quan, đảm bảo tương thích.

**Q: "Builder Pattern giải quyết vấn đề gì?"**
→ Telescoping Constructor Problem — khi constructor có quá nhiều tham số. Builder làm code rõ ràng hơn và đảm bảo object hợp lệ trước khi trả về.

**Q: "Singleton có phải anti-pattern không?"**
→ Singleton dùng đúng chỗ không phải anti-pattern. Vấn đề là khi dùng Singleton thay vì DI — làm code khó test. Trong .NET, nên dùng `AddSingleton()` của DI Container thay vì implement Singleton thủ công.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
