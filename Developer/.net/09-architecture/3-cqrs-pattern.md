# CQRS Pattern — Command Query Responsibility Segregation

> CQRS — Command Query Responsibility Segregation — Phân Tách Trách Nhiệm Lệnh và Truy Vấn là pattern tách biệt hoàn toàn operations **thay đổi dữ liệu** (Command — Lệnh) và operations **đọc dữ liệu** (Query — Truy vấn) thành hai model riêng biệt.

---

## 1. Vấn Đề CQRS Giải Quyết

### CRUD Truyền Thống — Vấn Đề

```
┌─────────────────────────────────────────────┐
│   Một Service / Model cho tất cả mọi thứ   │
│                                             │
│   GET    /orders → OrderDto                 │
│   POST   /orders → OrderDto                 │
│   PUT    /orders/{id} → OrderDto            │
│   DELETE /orders/{id} → void                │
│                                             │
│   Vấn đề:                                   │
│   - Read cần JOIN 10 bảng → model phức tạp │
│   - Write cần validate nghiêm ngặt          │
│   - Optimize read ảnh hưởng write model     │
│   - Scale không đồng đều (99% read, 1% write)│
└─────────────────────────────────────────────┘
```

### Sau Khi Áp Dụng CQRS

```
┌──────────────────────┐    ┌──────────────────────┐
│    Command Side      │    │    Query Side         │
│   (Write Model)      │    │   (Read Model)        │
│                      │    │                      │
│  SubmitOrderCmd      │    │  GetOrderByIdQuery   │
│  CancelOrderCmd      │    │  GetOrderListQuery   │
│  UpdateAddressCmd    │    │  GetOrderStatsQuery  │
│                      │    │                      │
│  → Validate nghiêm   │    │  → Optimize cho read │
│  → Domain logic      │    │  → Có thể dùng raw SQL│
│  → Raise events      │    │  → Flat DTO, nhanh   │
└──────────────────────┘    └──────────────────────┘
         ↓                           ↑
    Write DB                    Read DB (hoặc cùng DB)
```

---

## 2. CQRS Với MediatR — Triển Khai Thực Tế

### 2.1 Cài Đặt

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

### 2.2 Định Nghĩa Interfaces

```csharp
// Marker interfaces cho Commands và Queries
// Command trả về void hoặc result
public interface ICommand : IRequest { }
public interface ICommand<TResult> : IRequest<TResult> { }

// Query luôn trả về data, không thay đổi state
public interface IQuery<TResult> : IRequest<TResult> { }

// Handler interfaces
public interface ICommandHandler<TCommand> : IRequestHandler<TCommand>
    where TCommand : ICommand { }

public interface ICommandHandler<TCommand, TResult> : IRequestHandler<TCommand, TResult>
    where TCommand : ICommand<TResult> { }

public interface IQueryHandler<TQuery, TResult> : IRequestHandler<TQuery, TResult>
    where TQuery : IQuery<TResult> { }
```

### 2.3 Commands — Lệnh

```csharp
// Command: thay đổi trạng thái hệ thống, có thể fail
public record CreateOrderCommand(
    Guid CustomerId,
    List<OrderItemRequest> Items,
    AddressDto ShippingAddress) : ICommand<CreateOrderResult>;

public record OrderItemRequest(Guid ProductId, int Quantity);

public record CreateOrderResult(Guid OrderId, string Status);

// Command Handler — Xử lý lệnh
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand, CreateOrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderCommandHandler(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IUnitOfWork unitOfWork)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task<CreateOrderResult> Handle(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        // Validate và load data cần thiết
        var shippingAddress = new ShippingAddress(
            command.ShippingAddress.Street,
            command.ShippingAddress.City,
            command.ShippingAddress.Province);

        // Tạo Aggregate qua factory method
        var order = Order.Create(new CustomerId(command.CustomerId), shippingAddress);

        // Load products và thêm vào order
        foreach (var item in command.Items)
        {
            var product = await _productRepository.GetByIdAsync(
                new ProductId(item.ProductId), cancellationToken)
                ?? throw new NotFoundException($"Không tìm thấy sản phẩm {item.ProductId}");

            order.AddItem(product.Id, product.Name, item.Quantity, product.Price);
        }

        _orderRepository.Add(order);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        return new CreateOrderResult(order.Id.Value, order.Status.ToString());
    }
}
```

```csharp
// FluentValidation cho Command
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("CustomerId không được để trống.");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Đơn hàng phải có ít nhất một sản phẩm.")
            .Must(items => items.All(i => i.Quantity > 0))
            .WithMessage("Số lượng sản phẩm phải lớn hơn 0.");

        RuleFor(x => x.ShippingAddress.City)
            .NotEmpty().WithMessage("Thành phố giao hàng không được để trống.");
    }
}
```

### 2.4 Queries — Truy Vấn

```csharp
// Query: chỉ đọc, không thay đổi state
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailDto>;

// DTO tối ưu cho UI, không cần follow domain model
public record OrderDetailDto(
    Guid Id,
    string CustomerName,
    string Status,
    decimal TotalAmount,
    string Currency,
    DateTime CreatedAt,
    List<OrderItemDto> Items);

public record OrderItemDto(
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal Total);

// Query Handler — có thể dùng raw SQL hoặc EF Core AsNoTracking
public class GetOrderByIdQueryHandler : IQueryHandler<GetOrderByIdQuery, OrderDetailDto>
{
    private readonly AppDbContext _context;

    public GetOrderByIdQueryHandler(AppDbContext context) => _context = context;

    public async Task<OrderDetailDto> Handle(
        GetOrderByIdQuery query,
        CancellationToken cancellationToken)
    {
        // AsNoTracking vì chỉ đọc
        var order = await _context.Orders
            .AsNoTracking()
            .Include(o => o.Items)
            .Select(o => new OrderDetailDto(
                o.Id.Value,
                o.Customer.FullName,   // JOIN với Customer qua navigation property
                o.Status.ToString(),
                o.TotalAmount.Amount,
                o.TotalAmount.Currency,
                o.CreatedAt,
                o.Items.Select(i => new OrderItemDto(
                    i.ProductId.Value,
                    i.ProductName,
                    i.Quantity,
                    i.UnitPrice.Amount,
                    i.Total.Amount)).ToList()))
            .FirstOrDefaultAsync(o => o.Id == query.OrderId, cancellationToken);

        return order ?? throw new NotFoundException($"Không tìm thấy đơn hàng {query.OrderId}");
    }
}
```

```csharp
// Query với phân trang — Pagination
public record GetOrderListQuery(
    Guid? CustomerId,
    OrderStatus? Status,
    int Page = 1,
    int PageSize = 20) : IQuery<PagedResult<OrderSummaryDto>>;

public record PagedResult<T>(
    List<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

public class GetOrderListQueryHandler : IQueryHandler<GetOrderListQuery, PagedResult<OrderSummaryDto>>
{
    private readonly AppDbContext _context;

    public GetOrderListQueryHandler(AppDbContext context) => _context = context;

    public async Task<PagedResult<OrderSummaryDto>> Handle(
        GetOrderListQuery query,
        CancellationToken cancellationToken)
    {
        var dbQuery = _context.Orders.AsNoTracking();

        if (query.CustomerId.HasValue)
            dbQuery = dbQuery.Where(o => o.CustomerId.Value == query.CustomerId.Value);

        if (query.Status.HasValue)
            dbQuery = dbQuery.Where(o => o.Status == query.Status.Value);

        var totalCount = await dbQuery.CountAsync(cancellationToken);

        var items = await dbQuery
            .OrderByDescending(o => o.CreatedAt)
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(o => new OrderSummaryDto(
                o.Id.Value,
                o.Status.ToString(),
                o.TotalAmount.Amount,
                o.CreatedAt))
            .ToListAsync(cancellationToken);

        return new PagedResult<OrderSummaryDto>(items, totalCount, query.Page, query.PageSize);
    }
}
```

---

## 3. Pipeline Behaviors — Hành Vi Pipeline

MediatR cho phép thêm cross-cutting concerns — mối quan tâm xuyên suốt qua pipeline behaviors, tương tự middleware.

```csharp
// 1. Validation Behavior — xác thực tự động
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();

        var failures = _validators
            .Select(v => v.Validate(new ValidationContext<TRequest>(request)))
            .SelectMany(r => r.Errors)
            .Where(e => e is not null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}

// 2. Logging Behavior — ghi log
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Bắt đầu xử lý {RequestName}", requestName);

        var stopwatch = Stopwatch.StartNew();
        try
        {
            var response = await next();
            stopwatch.Stop();
            _logger.LogInformation(
                "Hoàn thành {RequestName} sau {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "Lỗi trong {RequestName} sau {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// 3. Performance Behavior — cảnh báo khi chậm
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private const int SlowRequestThresholdMs = 500;
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;

    public PerformanceBehavior(ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();
        var response = await next();
        stopwatch.Stop();

        if (stopwatch.ElapsedMilliseconds > SlowRequestThresholdMs)
        {
            _logger.LogWarning(
                "Request chậm phát hiện: {RequestName} mất {ElapsedMs}ms. Request: {@Request}",
                typeof(TRequest).Name,
                stopwatch.ElapsedMilliseconds,
                request);
        }

        return response;
    }
}
```

```csharp
// Đăng ký tất cả behaviors
services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(ApplicationAssemblyMarker).Assembly));

// Thứ tự quan trọng: thực thi theo thứ tự đăng ký
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

---

## 4. CQRS Với Read/Write Database Riêng — Nâng Cao

Khi read/write workload chênh lệch lớn, dùng hai database riêng:

```
┌─────────────────────────────────────────────────────┐
│                   API Layer                         │
└─────┬────────────────────────────────┬──────────────┘
      │ Commands                       │ Queries
      ▼                                ▼
┌───────────────┐              ┌───────────────────┐
│ Write Side    │              │ Read Side          │
│               │   Event      │                   │
│ SQL Server    │─────────────►│ Read Replica /    │
│ (normalized)  │   Sync       │ Elasticsearch /   │
│               │              │ Redis Cache        │
└───────────────┘              └───────────────────┘
```

```csharp
// Read-only DbContext cho Query side
public class ReadOnlyDbContext : DbContext
{
    public ReadOnlyDbContext(DbContextOptions<ReadOnlyDbContext> options) : base(options) { }

    // Không track thay đổi — tất cả query mặc định AsNoTracking
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }
}

// Hoặc dùng raw SQL/Dapper cho Query side — hiệu năng cao hơn
public class OrderQueryRepository
{
    private readonly string _connectionString;

    public OrderQueryRepository(IConfiguration configuration)
        => _connectionString = configuration.GetConnectionString("ReadReplica")!;

    public async Task<OrderDetailDto?> GetOrderDetailAsync(Guid orderId, CancellationToken cancellationToken)
    {
        using var connection = new SqlConnection(_connectionString);
        var sql = """
            SELECT o.Id, c.FullName AS CustomerName, o.Status,
                   o.TotalAmount, o.Currency, o.CreatedAt,
                   oi.ProductId, oi.ProductName, oi.Quantity, oi.UnitPrice
            FROM Orders o
            JOIN Customers c ON o.CustomerId = c.Id
            JOIN OrderItems oi ON oi.OrderId = o.Id
            WHERE o.Id = @OrderId
            """;

        var result = await connection.QueryAsync<OrderDetailFlat>(sql, new { OrderId = orderId });
        return MapToDto(result);
    }
}
```

---

## 5. Controller — Sử Dụng CQRS

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly ISender _sender;

    public OrdersController(ISender sender) => _sender = sender;

    // Command endpoint — POST thay đổi dữ liệu
    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var command = new CreateOrderCommand(
            request.CustomerId,
            request.Items.Select(i => new OrderItemRequest(i.ProductId, i.Quantity)).ToList(),
            request.ShippingAddress);

        var result = await _sender.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetOrder), new { id = result.OrderId }, result);
    }

    [HttpPost("{id}/submit")]
    public async Task<IActionResult> SubmitOrder(Guid id, CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new SubmitOrderCommand(id), cancellationToken);
        return Ok(result);
    }

    // Query endpoint — GET chỉ đọc
    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new GetOrderByIdQuery(id), cancellationToken);
        return Ok(result);
    }

    [HttpGet]
    public async Task<IActionResult> GetOrders(
        [FromQuery] Guid? customerId,
        [FromQuery] OrderStatus? status,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken cancellationToken = default)
    {
        var result = await _sender.Send(
            new GetOrderListQuery(customerId, status, page, pageSize),
            cancellationToken);
        return Ok(result);
    }
}
```

---

## 6. Ưu Điểm & Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Chi Tiết |
|---------|---------|
| **Optimize riêng** | Read model tối ưu cho display, Write model tối ưu cho consistency |
| **Scale độc lập** | Read side và Write side scale theo nhu cầu riêng |
| **Rõ intent** | Nhìn vào Command/Query biết ngay operation làm gì |
| **Dễ test** | Command handler và Query handler độc lập, dễ mock |
| **Pipeline behaviors** | Logging, validation, caching thêm vào một chỗ |

### ❌ Nhược Điểm

| Nhược Điểm | Chi Tiết |
|-----------|---------|
| **Nhiều file** | Mỗi operation cần ít nhất 2 files (command + handler) |
| **Eventual consistency** | Nếu dùng read replica — database đọc riêng, có độ trễ đồng bộ |
| **Overkill cho CRUD** | CRUD đơn giản không cần đến CQRS |
| **Learning curve** | Team cần học MediatR và pattern |

---

## 7. Câu Hỏi Phỏng Vấn

**Q: CQRS giải quyết vấn đề gì?**
> A: Tách biệt read và write operations thành hai model riêng, giúp optimize từng side độc lập. Write side tập trung vào business logic và consistency; Read side tập trung vào performance và flexible projection.

**Q: CQRS có bắt buộc phải dùng hai database không?**
> A: Không. CQRS là pattern về separation of concerns — phân tách mối quan tâm, không bắt buộc database riêng. Cùng database nhưng tách model đã có lợi ích. Database riêng là bước nâng cao khi cần scale.

**Q: MediatR có thực sự cần thiết không?**
> A: Không bắt buộc. MediatR là implementation detail cho CQRS, giúp dispatch commands/queries qua handler tương ứng. Có thể implement CQRS bằng tay (manual dispatch) hoặc dùng library khác. MediatR phổ biến vì đơn giản và có pipeline behavior.

**Q: CQRS và Event Sourcing có liên quan không?**
> A: Thường đi chung nhưng độc lập nhau. CQRS là về separation of read/write model. Event Sourcing là về cách lưu state qua events. Dùng cả hai cho phép Write side lưu events, Read side subscribe và build read model.

---

**Cập Nhật Lần Cuối:** 2026-06-02
