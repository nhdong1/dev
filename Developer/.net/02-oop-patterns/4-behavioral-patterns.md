# Behavioral Patterns — Mẫu Thiết Kế Hành Vi

> Behavioral Patterns — Mẫu Hành Vi — giải quyết vấn đề **giao tiếp và phân chia trách nhiệm** giữa các đối tượng, xác định cách các đối tượng tương tác và phân phối công việc.

---

## 📋 Tổng Quan

| Pattern | Mục Đích | Khi Nào Dùng | .NET Ví Dụ |
| ------- | -------- | ------------ | ----------- |
| **Strategy** | Hoán đổi thuật toán khi chạy | Nhiều cách làm cùng một việc | Sorting, validation, pricing |
| **Observer** | Thông báo nhiều subscriber khi state thay đổi | Event-driven system | `event`, SignalR, domain events |
| **Mediator** | Trung gian điều phối giao tiếp | Giảm coupling giữa nhiều object | MediatR, chat room |
| **Command** | Đóng gói yêu cầu thành object | Undo/Redo, queue, CQRS | ICommand, CQRS commands |
| **Chain of Responsibility** | Truyền request qua chuỗi handler | Pipeline xử lý tuần tự | ASP.NET Middleware, validation |

---

## 1. Strategy Pattern — Mẫu Chiến Lược

### Định Nghĩa

> Định nghĩa một nhóm các thuật toán, đóng gói mỗi cái, và làm cho chúng có thể hoán đổi cho nhau. Strategy — Chiến Lược — cho phép thuật toán thay đổi độc lập với client sử dụng nó.

**Đây là cơ chế triển khai chính của OCP — Open/Closed Principle.**

### Khi Nào Dùng

- Nhiều cách triển khai cùng một behavior
- Muốn chọn algorithm tại runtime
- Thay `if/else` hay `switch` trên type bằng polymorphism
- Cần unit test từng algorithm độc lập

### ✅ Strategy Cho Discount Calculation — Tính Giảm Giá

```csharp
// Strategy interface
public interface IDiscountStrategy
{
    decimal Calculate(decimal originalPrice, Customer customer);
    string Description { get; }
}

// Concrete strategies — các chiến lược cụ thể
public class NoDiscount : IDiscountStrategy
{
    public string Description => "Không giảm giá";
    public decimal Calculate(decimal price, Customer customer) => 0;
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percentage;
    public PercentageDiscount(decimal percentage) => _percentage = percentage;
    public string Description => $"Giảm {_percentage}%";
    public decimal Calculate(decimal price, Customer customer) => price * _percentage / 100;
}

public class MembershipDiscount : IDiscountStrategy
{
    public string Description => "Ưu đãi thành viên";
    public decimal Calculate(decimal price, Customer customer) =>
        customer.MembershipLevel switch
        {
            MembershipLevel.Silver => price * 0.05m,
            MembershipLevel.Gold   => price * 0.10m,
            MembershipLevel.Platinum => price * 0.15m,
            _ => 0
        };
}

public class SeasonalDiscount : IDiscountStrategy
{
    public string Description => "Khuyến mãi mùa hè";
    public decimal Calculate(decimal price, Customer customer)
    {
        var month = DateTime.Now.Month;
        return month is >= 6 and <= 8 ? price * 0.20m : 0; // Hè: tháng 6-8
    }
}

// Context — ngữ cảnh sử dụng strategy
public class PricingEngine
{
    private readonly List<IDiscountStrategy> _strategies = [];

    public PricingEngine AddStrategy(IDiscountStrategy strategy)
    {
        _strategies.Add(strategy);
        return this; // fluent API
    }

    public PriceBreakdown Calculate(decimal originalPrice, Customer customer)
    {
        var discounts = _strategies
            .Select(s => new DiscountLine(s.Description, s.Calculate(originalPrice, customer)))
            .Where(d => d.Amount > 0)
            .ToList();

        var totalDiscount = discounts.Sum(d => d.Amount);
        return new PriceBreakdown(originalPrice, discounts, originalPrice - totalDiscount);
    }
}

// Sử dụng — linh hoạt kết hợp strategies
var engine = new PricingEngine()
    .AddStrategy(new MembershipDiscount())
    .AddStrategy(new SeasonalDiscount());

var breakdown = engine.Calculate(1_000_000m, currentCustomer);
Console.WriteLine($"Giá gốc: {breakdown.OriginalPrice:C}");
Console.WriteLine($"Giá sau giảm: {breakdown.FinalPrice:C}");
```

### Strategy Cho Sorting — Sắp Xếp

```csharp
// Strategy interface cho sort
public interface ISortStrategy<T>
{
    IEnumerable<T> Sort(IEnumerable<T> items);
}

// Concrete strategies
public class AscendingNameSort : ISortStrategy<Product>
{
    public IEnumerable<Product> Sort(IEnumerable<Product> items)
        => items.OrderBy(p => p.Name);
}

public class PriceDescendingSort : ISortStrategy<Product>
{
    public IEnumerable<Product> Sort(IEnumerable<Product> items)
        => items.OrderByDescending(p => p.Price);
}

public class PopularitySort : ISortStrategy<Product>
{
    public IEnumerable<Product> Sort(IEnumerable<Product> items)
        => items.OrderByDescending(p => p.SalesCount).ThenBy(p => p.Name);
}

// Context
public class ProductCatalog
{
    private ISortStrategy<Product> _sortStrategy = new AscendingNameSort();

    public void SetSortStrategy(ISortStrategy<Product> strategy) => _sortStrategy = strategy;

    public IEnumerable<Product> GetProducts(IEnumerable<Product> products)
        => _sortStrategy.Sort(products);
}
```

---

## 2. Observer Pattern — Mẫu Quan Sát Viên

### Định Nghĩa

> Định nghĩa mối quan hệ one-to-many — một-đến-nhiều giữa các đối tượng, sao cho khi một đối tượng thay đổi trạng thái, tất cả các đối tượng phụ thuộc của nó sẽ được thông báo và cập nhật tự động.

**Subject (Publisher) — Chủ thể (Nhà xuất bản):** đối tượng được quan sát
**Observer (Subscriber) — Quan sát viên (Người đăng ký):** đối tượng được thông báo

### ✅ Observer Với C# Event

```csharp
// Subject — publisher
public class StockMarket
{
    // C# event là Observer Pattern tích hợp sẵn
    public event EventHandler<StockPriceChangedEventArgs>? StockPriceChanged;

    private readonly Dictionary<string, decimal> _prices = [];

    public void UpdatePrice(string symbol, decimal newPrice)
    {
        var oldPrice = _prices.GetValueOrDefault(symbol);
        _prices[symbol] = newPrice;

        // Thông báo tất cả subscribers
        StockPriceChanged?.Invoke(this, new StockPriceChangedEventArgs
        {
            Symbol = symbol,
            OldPrice = oldPrice,
            NewPrice = newPrice,
            ChangePercent = oldPrice > 0 ? (newPrice - oldPrice) / oldPrice * 100 : 0
        });
    }
}

public class StockPriceChangedEventArgs : EventArgs
{
    public string Symbol { get; init; } = string.Empty;
    public decimal OldPrice { get; init; }
    public decimal NewPrice { get; init; }
    public decimal ChangePercent { get; init; }
}

// Observers — subscribers
public class PriceAlertService
{
    private readonly Dictionary<string, decimal> _alerts = []; // symbol → threshold

    public void SetAlert(string symbol, decimal threshold) => _alerts[symbol] = threshold;

    // Handler — xử lý sự kiện
    public void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (_alerts.TryGetValue(e.Symbol, out var threshold) && e.NewPrice >= threshold)
            Console.WriteLine($"⚠️ ALERT: {e.Symbol} đạt {e.NewPrice:C} (ngưỡng: {threshold:C})");
    }
}

public class PortfolioTracker
{
    private readonly Dictionary<string, int> _holdings = [];

    public void AddHolding(string symbol, int quantity) => _holdings[symbol] = quantity;

    public void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (_holdings.TryGetValue(e.Symbol, out var qty))
        {
            var value = qty * e.NewPrice;
            Console.WriteLine($"📊 {e.Symbol}: {qty} cổ phiếu × {e.NewPrice:C} = {value:C} ({e.ChangePercent:+0.##;-0.##}%)");
        }
    }
}

// Wiring — kết nối subscriber với publisher
var market = new StockMarket();
var alertService = new PriceAlertService();
var portfolio = new PortfolioTracker();

alertService.SetAlert("VNM", 85_000m);
portfolio.AddHolding("VNM", 100);
portfolio.AddHolding("FPT", 50);

// Đăng ký (subscribe)
market.StockPriceChanged += alertService.OnPriceChanged;
market.StockPriceChanged += portfolio.OnPriceChanged;

market.UpdatePrice("VNM", 86_000m); // Kích hoạt alert và update portfolio
market.UpdatePrice("FPT", 120_000m); // Chỉ update portfolio

// Hủy đăng ký (unsubscribe) khi không cần nữa — quan trọng để tránh memory leak
market.StockPriceChanged -= alertService.OnPriceChanged;
```

### ✅ Domain Events — Sự Kiện Miền (Observer Trong DDD)

```csharp
// Domain Event — sự kiện trong domain
public record OrderPlacedEvent(Guid OrderId, Guid CustomerId, decimal Total, DateTime OccurredAt);
public record PaymentProcessedEvent(Guid OrderId, string TransactionId, decimal Amount);

// Event Publisher — nhà xuất bản sự kiện
public interface IDomainEventPublisher
{
    Task PublishAsync<T>(T domainEvent) where T : class;
}

// Event Handlers — các xử lý sự kiện
public class SendOrderConfirmationEmailHandler
{
    private readonly IEmailService _email;
    public SendOrderConfirmationEmailHandler(IEmailService email) => _email = email;

    public async Task HandleAsync(OrderPlacedEvent evt)
    {
        await _email.SendOrderConfirmationAsync(evt.CustomerId, evt.OrderId);
    }
}

public class UpdateInventoryHandler
{
    private readonly IInventoryService _inventory;
    public UpdateInventoryHandler(IInventoryService inventory) => _inventory = inventory;

    public async Task HandleAsync(OrderPlacedEvent evt)
    {
        await _inventory.DeductStockAsync(evt.OrderId);
    }
}

// Với MediatR — thư viện Mediator phổ biến trong .NET
// OrderPlacedEvent implement INotification
// Handler implement INotificationHandler<OrderPlacedEvent>
// MediatR tự động dispatch đến tất cả handlers đã đăng ký
```

---

## 3. Mediator Pattern — Mẫu Trung Gian

### Định Nghĩa

> Định nghĩa một đối tượng đóng gói cách một tập hợp các đối tượng tương tác. Mediator — Trung Gian — thúc đẩy loose coupling — ghép nối lỏng bằng cách ngăn các đối tượng tham chiếu trực tiếp lẫn nhau.

**Tương tự thực tế:** Kiểm soát không lưu — máy bay không giao tiếp trực tiếp với nhau, tất cả đi qua control tower.

### Khi Nào Dùng

- Nhiều object tương tác tạo ra spaghetti code — mã spaghetti
- Muốn tách rời business logic khỏi request/response plumbing
- CQRS — Command Query Responsibility Segregation — phân tách lệnh và truy vấn

### ✅ Mediator Đơn Giản

```csharp
// Mediator interface
public interface IMediator
{
    Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request);
    Task PublishAsync<TNotification>(TNotification notification) where TNotification : class;
}

// Request/Response — yêu cầu và phản hồi
public interface IRequest<TResponse> { }

// Handlers — các xử lý
public interface IRequestHandler<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct = default);
}

// Ví dụ: CQRS Command
public record CreateOrderCommand(Guid CustomerId, List<OrderItem> Items) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IMediator _mediator;

    public CreateOrderHandler(IOrderRepository orders, IMediator mediator)
    {
        _orders = orders;
        _mediator = mediator;
    }

    public async Task<Guid> HandleAsync(CreateOrderCommand cmd, CancellationToken ct = default)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);
        await _orders.SaveAsync(order);

        // Publish event — không cần biết ai sẽ xử lý
        await _mediator.PublishAsync(new OrderPlacedEvent(order.Id, cmd.CustomerId, order.Total, DateTime.UtcNow));

        return order.Id;
    }
}

// Query
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderRepository _orders;
    public GetOrderHandler(IOrderRepository orders) => _orders = orders;

    public async Task<OrderDto?> HandleAsync(GetOrderQuery query, CancellationToken ct = default)
    {
        var order = await _orders.GetByIdAsync(query.OrderId);
        return order == null ? null : OrderDto.FromEntity(order);
    }
}
```

### MediatR — Thư Viện Mediator Phổ Biến Nhất Trong .NET

```csharp
// Cài đặt: dotnet add package MediatR

// Command với MediatR
public record CreateProductCommand(string Name, decimal Price) : IRequest<Guid>;

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly AppDbContext _db;

    public CreateProductCommandHandler(AppDbContext db) => _db = db;

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product { Name = request.Name, Price = request.Price };
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
        return product.Id;
    }
}

// Pipeline Behavior — Hành Vi Pipeline (giống Middleware nhưng cho MediatR)
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request,
        RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        // Validate trước khi gọi handler
        var failures = _validators
            .SelectMany(v => v.Validate(request).Errors)
            .Where(e => e != null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next(); // gọi handler tiếp theo
    }
}

// Đăng ký trong DI
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

// Controller dùng MediatR — không biết gì về handler
[ApiController, Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    public ProductsController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create(CreateProductCommand command)
    {
        var id = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}
```

---

## 4. Command Pattern — Mẫu Lệnh

### Định Nghĩa

> Đóng gói một yêu cầu thành object, cho phép tham số hóa client với các yêu cầu khác nhau, hàng đợi (queue) hay ghi log các yêu cầu, và hỗ trợ các thao tác có thể hoàn tác (undo).

### ✅ Command Với Undo/Redo — Hoàn Tác/Làm Lại

```csharp
// Command interface
public interface ICommand
{
    void Execute();
    void Undo();
}

// Receiver — người nhận lệnh
public class TextEditor
{
    private readonly StringBuilder _content = new();
    public string Content => _content.ToString();

    public void InsertText(int position, string text) => _content.Insert(position, text);
    public void DeleteText(int position, int length) => _content.Remove(position, length);
}

// Concrete commands — lệnh cụ thể
public class InsertTextCommand : ICommand
{
    private readonly TextEditor _editor;
    private readonly int _position;
    private readonly string _text;

    public InsertTextCommand(TextEditor editor, int position, string text)
    {
        _editor = editor;
        _position = position;
        _text = text;
    }

    public void Execute() => _editor.InsertText(_position, _text);
    public void Undo() => _editor.DeleteText(_position, _text.Length);
}

public class DeleteTextCommand : ICommand
{
    private readonly TextEditor _editor;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText = string.Empty;

    public DeleteTextCommand(TextEditor editor, int position, int length)
    {
        _editor = editor;
        _position = position;
        _length = length;
    }

    public void Execute()
    {
        // Lưu lại text trước khi xóa để có thể undo
        _deletedText = _editor.Content.Substring(_position, _length);
        _editor.DeleteText(_position, _length);
    }

    public void Undo() => _editor.InsertText(_position, _deletedText);
}

// Invoker — người gọi lệnh, quản lý lịch sử
public class CommandHistory
{
    private readonly Stack<ICommand> _history = new();
    private readonly Stack<ICommand> _redoStack = new();

    public void Execute(ICommand command)
    {
        command.Execute();
        _history.Push(command);
        _redoStack.Clear(); // sau khi execute, redo stack bị xóa
    }

    public void Undo()
    {
        if (!_history.TryPop(out var command)) return;
        command.Undo();
        _redoStack.Push(command);
    }

    public void Redo()
    {
        if (!_redoStack.TryPop(out var command)) return;
        command.Execute();
        _history.Push(command);
    }
}

// Sử dụng
var editor = new TextEditor();
var history = new CommandHistory();

history.Execute(new InsertTextCommand(editor, 0, "Xin chào"));
history.Execute(new InsertTextCommand(editor, 8, " thế giới"));
Console.WriteLine(editor.Content); // "Xin chào thế giới"

history.Undo(); // Xóa " thế giới"
Console.WriteLine(editor.Content); // "Xin chào"

history.Redo(); // Chèn lại " thế giới"
Console.WriteLine(editor.Content); // "Xin chào thế giới"
```

### Command Pattern Trong Queue — Hàng Đợi Tác Vụ

```csharp
// Command interface cho background jobs
public interface IJob
{
    Task ExecuteAsync(CancellationToken ct);
    string JobName { get; }
}

// Concrete jobs — lệnh cụ thể
public class SendEmailJob : IJob
{
    public string JobName => "SendEmail";
    private readonly string _to;
    private readonly string _subject;
    private readonly IEmailService _emailService;

    public SendEmailJob(string to, string subject, IEmailService emailService)
    {
        _to = to;
        _subject = subject;
        _emailService = emailService;
    }

    public async Task ExecuteAsync(CancellationToken ct)
        => await _emailService.SendAsync(_to, _subject, ct);
}

public class GenerateReportJob : IJob
{
    public string JobName => "GenerateReport";
    private readonly Guid _reportId;

    public GenerateReportJob(Guid reportId) => _reportId = reportId;

    public async Task ExecuteAsync(CancellationToken ct)
    {
        Console.WriteLine($"Tạo report {_reportId}...");
        await Task.Delay(5000, ct); // simulate heavy work
    }
}

// Job Queue — hàng đợi lệnh
public class BackgroundJobQueue
{
    private readonly Channel<IJob> _channel = Channel.CreateUnbounded<IJob>();

    public ValueTask EnqueueAsync(IJob job) => _channel.Writer.WriteAsync(job);

    public IAsyncEnumerable<IJob> ReadAllAsync(CancellationToken ct)
        => _channel.Reader.ReadAllAsync(ct);
}
```

---

## 5. Chain of Responsibility — Mẫu Chuỗi Trách Nhiệm

### Định Nghĩa

> Cho phép truyền request qua một chuỗi các handler — xử lý viên. Mỗi handler quyết định xử lý request hoặc chuyển tiếp cho handler kế tiếp trong chuỗi.

**Tương tự thực tế:** Bộ phận hỗ trợ khách hàng — Level 1 → Level 2 → Level 3 → Manager.

### ✅ Chain of Responsibility Cho Validation Pipeline

```csharp
// Handler interface
public abstract class ValidationHandler<T>
{
    private ValidationHandler<T>? _next;

    public ValidationHandler<T> SetNext(ValidationHandler<T> next)
    {
        _next = next;
        return next; // cho phép chain: handler1.SetNext(handler2).SetNext(handler3)
    }

    public virtual ValidationResult Handle(T request)
    {
        if (_next != null)
            return _next.Handle(request);

        return ValidationResult.Success; // cuối chuỗi — không ai từ chối
    }
}

// Concrete handlers — xử lý viên cụ thể
public class RequiredFieldsHandler : ValidationHandler<CreateOrderRequest>
{
    public override ValidationResult Handle(CreateOrderRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.CustomerEmail))
            return ValidationResult.Fail("Email khách hàng là bắt buộc");

        if (request.Items == null || !request.Items.Any())
            return ValidationResult.Fail("Đơn hàng phải có ít nhất một sản phẩm");

        return base.Handle(request); // chuyển sang handler kế tiếp
    }
}

public class BusinessRulesHandler : ValidationHandler<CreateOrderRequest>
{
    private const decimal MaxOrderAmount = 100_000_000m;

    public override ValidationResult Handle(CreateOrderRequest request)
    {
        var total = request.Items.Sum(i => i.Price * i.Quantity);

        if (total > MaxOrderAmount)
            return ValidationResult.Fail($"Giá trị đơn hàng không vượt quá {MaxOrderAmount:C}");

        if (request.Items.Any(i => i.Quantity > 100))
            return ValidationResult.Fail("Số lượng mỗi sản phẩm không vượt quá 100");

        return base.Handle(request);
    }
}

public class FraudDetectionHandler : ValidationHandler<CreateOrderRequest>
{
    private readonly IFraudService _fraudService;
    public FraudDetectionHandler(IFraudService fraudService) => _fraudService = fraudService;

    public override ValidationResult Handle(CreateOrderRequest request)
    {
        if (_fraudService.IsSuspicious(request.CustomerEmail, request.IpAddress))
            return ValidationResult.Fail("Yêu cầu bị từ chối vì lý do bảo mật");

        return base.Handle(request);
    }
}

// Xây dựng chuỗi và sử dụng
var required = new RequiredFieldsHandler();
var business = new BusinessRulesHandler();
var fraud = new FraudDetectionHandler(fraudService);

// Kết nối chuỗi: required → business → fraud
required.SetNext(business).SetNext(fraud);

var request = new CreateOrderRequest { /* ... */ };
var result = required.Handle(request); // bắt đầu từ đầu chuỗi

if (!result.IsValid)
    return BadRequest(result.ErrorMessage);
```

### ASP.NET Core Middleware Là Chain of Responsibility

```csharp
// Mỗi middleware là một handler trong chuỗi
app.Use(async (context, next) =>
{
    // Xử lý trước (hoặc từ chối)
    if (!context.Request.Headers.ContainsKey("X-Api-Key"))
    {
        context.Response.StatusCode = 401;
        return; // Không gọi next → dừng chuỗi
    }

    await next(context); // Chuyển sang middleware kế tiếp

    // Xử lý sau khi response về
    context.Response.Headers.Append("X-Processed-By", "ApiGateway");
});

app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);
    context.Response.Headers.Append("X-Response-Time", $"{sw.ElapsedMilliseconds}ms");
});
```

---

## 🔄 Tổng Kết So Sánh

| Pattern | Câu Hỏi Gợi Nhớ | Quan Hệ Quan Trọng |
| ------- | --------------- | ------------------ |
| **Strategy** | "Cần đổi algorithm lúc runtime?" | Thay `if/else` kiểm tra type |
| **Observer** | "Cần thông báo nhiều bên khi có sự kiện?" | Publisher/Subscriber |
| **Mediator** | "Các object giao tiếp quá phức tạp?" | Tập trung giao tiếp vào một chỗ |
| **Command** | "Cần queue, undo/redo, hay log actions?" | Đóng gói request thành object |
| **Chain of Responsibility** | "Nhiều bước xử lý tuần tự, mỗi bước có thể dừng?" | Pipeline xử lý |

---

## 🔗 Liên Hệ Thực Tế Trong .NET Ecosystem

| Pattern | Thể Hiện Trong .NET |
| ------- | ------------------- |
| Strategy | `IComparer<T>`, `IEqualityComparer<T>`, `HttpMessageHandler` |
| Observer | `event`, `IObservable<T>`, `INotifyPropertyChanged`, SignalR |
| Mediator | **MediatR** library, `IMediator` |
| Command | **MediatR** `IRequest`, CQRS Command objects |
| Chain | ASP.NET Middleware, **MediatR** `IPipelineBehavior<T>` |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: "MediatR sử dụng pattern nào?"**
→ Mediator Pattern (chính) + Command Pattern (cho IRequest). Pipeline Behavior dùng Chain of Responsibility.

**Q: "Strategy vs if/else — khi nào chuyển sang Strategy?"**
→ Khi có 3+ nhánh, khi nhánh có thể thêm mới (OCP), khi muốn test riêng từng algorithm. If/else ổn cho logic đơn giản, ít thay đổi.

**Q: "Observer và Event trong C# — liên quan gì?"**
→ `event` trong C# là Observer Pattern tích hợp sẵn. `event` cung cấp subscribe/unsubscribe, thread-safe invocation. Cần chú ý unsubscribe để tránh memory leak.

**Q: "CQRS dùng pattern nào?"**
→ Command Pattern (tách Command vs Query), Mediator Pattern (dispatch qua MediatR), Strategy Pattern (nhiều handler cho cùng loại command).

**Q: "Tại sao Chain of Responsibility tốt hơn nested if/else?"**
→ Dễ thêm/bỏ handler mà không sửa code hiện tại (OCP). Mỗi handler có một trách nhiệm (SRP). Dễ unit test từng handler riêng lẻ.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Trạng Thái:** ✅ Hoàn thành
