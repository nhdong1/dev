# Clean Architecture — Kiến Trúc Sạch

> Clean Architecture (Kiến Trúc Sạch) là mô hình kiến trúc phần mềm do Robert C. Martin (Uncle Bob) đề xuất, tổ chức code thành các layer — tầng đồng tâm — với quy tắc phụ thuộc hướng vào trong (Dependency Rule). Mục tiêu: domain logic — logic nghiệp vụ — hoàn toàn độc lập với framework, database, và UI.

---

## 1. Nguyên Tắc Cốt Lõi — The Dependency Rule

```
┌─────────────────────────────────────────────┐
│           Presentation Layer                │  ← ASP.NET Controllers, gRPC, SignalR
│  ┌───────────────────────────────────────┐  │
│  │        Infrastructure Layer           │  │  ← EF Core, HTTP clients, file system
│  │  ┌─────────────────────────────────┐  │  │
│  │  │      Application Layer          │  │  │  ← Use Cases, Commands, Queries
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │      Domain Layer         │  │  │  │  ← Entities, Value Objects, Domain Events
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘

Dependency Rule: Phụ thuộc CHỈ hướng vào trong
Outer layers biết inner layers, KHÔNG chiều ngược lại
```

**Quy tắc vàng:** Domain layer **không import** bất kỳ thứ gì từ Infrastructure hoặc Presentation.

---

## 2. Bốn Tầng Kiến Trúc

### 2.1 Domain Layer — Tầng Miền (Trung Tâm)

- **Chứa:** Entities (Thực thể), Value Objects (Đối tượng giá trị), Domain Events (Sự kiện miền), Domain Services (Dịch vụ miền), Repository Interfaces (Giao diện kho dữ liệu)
- **Không phụ thuộc:** Không có NuGet package bên ngoài (ngoại trừ FluentValidation nếu cần)
- **Là trung tâm** của toàn bộ hệ thống

```csharp
// Domain Layer — không import gì từ bên ngoài
namespace Domain.Entities;

public class Order
{
    public Guid Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public Money TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    // Domain Events — Sự kiện miền
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    // Factory method — không dùng constructor public
    public static Order Create(CustomerId customerId)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            TotalAmount = Money.Zero
        };
        order._domainEvents.Add(new OrderCreatedEvent(order.Id));
        return order;
    }

    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể thêm sản phẩm vào đơn hàng ở trạng thái Draft.");

        var item = new OrderItem(productId, quantity, unitPrice);
        _items.Add(item);
        TotalAmount = TotalAmount.Add(item.Total);
    }

    public void Submit()
    {
        if (!_items.Any())
            throw new DomainException("Đơn hàng phải có ít nhất một sản phẩm.");

        Status = OrderStatus.Submitted;
        _domainEvents.Add(new OrderSubmittedEvent(Id, TotalAmount));
    }
}
```

```csharp
// Value Object — Đối tượng giá trị: bất biến, so sánh theo giá trị
public record Money(decimal Amount, string Currency)
{
    public static Money Zero => new(0, "VND");

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Không thể cộng tiền khác đơn vị: {Currency} và {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public bool IsPositive() => Amount > 0;
}

// Strongly-typed ID — ID có kiểu mạnh, tránh nhầm lẫn Guid
public record CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.NewGuid());
}
```

```csharp
// Repository Interface — trong Domain, Implementation — trong Infrastructure
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Order>> GetByCustomerIdAsync(CustomerId customerId, CancellationToken cancellationToken = default);
    void Add(Order order);
    void Update(Order order);
}
```

---

### 2.2 Application Layer — Tầng Ứng Dụng

- **Chứa:** Use Cases (Trường hợp sử dụng), Commands, Queries, DTOs (Data Transfer Objects — Đối tượng truyền dữ liệu), Application Services, Validation
- **Phụ thuộc:** Domain Layer (vào trong)
- **Không phụ thuộc:** Infrastructure, Presentation

```csharp
// Command — Lệnh (thay đổi dữ liệu)
public record SubmitOrderCommand(Guid OrderId) : ICommand<SubmitOrderResult>;

// Command Handler — Xử lý lệnh
public class SubmitOrderCommandHandler : ICommandHandler<SubmitOrderCommand, SubmitOrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IDomainEventDispatcher _eventDispatcher;

    public SubmitOrderCommandHandler(
        IOrderRepository orderRepository,
        IUnitOfWork unitOfWork,
        IDomainEventDispatcher eventDispatcher)
    {
        _orderRepository = orderRepository;
        _unitOfWork = unitOfWork;
        _eventDispatcher = eventDispatcher;
    }

    public async Task<SubmitOrderResult> Handle(
        SubmitOrderCommand command,
        CancellationToken cancellationToken)
    {
        var order = await _orderRepository.GetByIdAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException($"Không tìm thấy đơn hàng {command.OrderId}");

        order.Submit();  // domain logic ở đây

        await _unitOfWork.SaveChangesAsync(cancellationToken);

        // Dispatch domain events sau khi save thành công
        await _eventDispatcher.DispatchAsync(order.DomainEvents, cancellationToken);

        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

```csharp
// Query — Truy vấn (chỉ đọc, không thay đổi)
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailDto>;

public class GetOrderByIdQueryHandler : IQueryHandler<GetOrderByIdQuery, OrderDetailDto>
{
    private readonly IOrderReadRepository _readRepository; // read-only repository

    public GetOrderByIdQueryHandler(IOrderReadRepository readRepository)
    {
        _readRepository = readRepository;
    }

    public async Task<OrderDetailDto> Handle(
        GetOrderByIdQuery query,
        CancellationToken cancellationToken)
    {
        return await _readRepository.GetOrderDetailAsync(query.OrderId, cancellationToken)
            ?? throw new NotFoundException($"Không tìm thấy đơn hàng {query.OrderId}");
    }
}
```

---

### 2.3 Infrastructure Layer — Tầng Hạ Tầng

- **Chứa:** EF Core implementations, HTTP clients, file I/O, email, 3rd-party services
- **Implement:** Các interface định nghĩa trong Domain/Application
- **Phụ thuộc:** Application Layer và Domain Layer

```csharp
// EF Core implementation của IOrderRepository
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<IReadOnlyList<Order>> GetByCustomerIdAsync(
        CustomerId customerId,
        CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Where(o => o.CustomerId == customerId)
            .ToListAsync(cancellationToken);
    }

    public void Add(Order order) => _context.Orders.Add(order);
    public void Update(Order order) => _context.Orders.Update(order);
}
```

```csharp
// EF Core DbContext configuration
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; } = null!;
    public DbSet<OrderItem> OrderItems { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Áp dụng tất cả IEntityTypeConfiguration trong assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Fluent configuration — cấu hình EF Core, tách khỏi domain entity
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);

        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount");
            money.Property(m => m.Currency).HasColumnName("Currency").HasMaxLength(3);
        });

        builder.HasMany(o => o.Items)
               .WithOne()
               .HasForeignKey("OrderId");

        // Ignore domain events — không lưu vào DB
        builder.Ignore(o => o.DomainEvents);
    }
}
```

---

### 2.4 Presentation Layer — Tầng Trình Bày

- **Chứa:** ASP.NET Controllers, Minimal API endpoints, SignalR hubs, gRPC services
- **Phụ thuộc:** Application Layer (gửi Commands và Queries)
- **Không chứa:** Business logic

```csharp
// ASP.NET Core Controller — chỉ điều phối, không có logic
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly ISender _sender; // MediatR sender

    public OrdersController(ISender sender)
    {
        _sender = sender;
    }

    [HttpPost("{id}/submit")]
    public async Task<IActionResult> SubmitOrder(Guid id, CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new SubmitOrderCommand(id), cancellationToken);
        return Ok(result);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var result = await _sender.Send(new GetOrderByIdQuery(id), cancellationToken);
        return Ok(result);
    }
}
```

---

## 3. Cấu Trúc Project .NET

```
MySolution/
├── src/
│   ├── MyApp.Domain/                ← Domain Layer
│   │   ├── Entities/
│   │   │   └── Order.cs
│   │   ├── ValueObjects/
│   │   │   └── Money.cs
│   │   ├── Events/
│   │   │   └── OrderSubmittedEvent.cs
│   │   ├── Repositories/
│   │   │   └── IOrderRepository.cs  ← interface (contract)
│   │   └── Exceptions/
│   │       └── DomainException.cs
│   │
│   ├── MyApp.Application/           ← Application Layer
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   └── SubmitOrder/
│   │   │   │       ├── SubmitOrderCommand.cs
│   │   │   │       └── SubmitOrderCommandHandler.cs
│   │   │   └── Queries/
│   │   │       └── GetOrderById/
│   │   │           ├── GetOrderByIdQuery.cs
│   │   │           └── GetOrderByIdQueryHandler.cs
│   │   ├── Common/
│   │   │   ├── ICommand.cs
│   │   │   ├── IQuery.cs
│   │   │   └── IUnitOfWork.cs
│   │   └── DependencyInjection.cs
│   │
│   ├── MyApp.Infrastructure/        ← Infrastructure Layer
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   │   └── OrderConfiguration.cs
│   │   │   └── Repositories/
│   │   │       └── OrderRepository.cs ← implements IOrderRepository
│   │   ├── Services/
│   │   │   └── EmailService.cs
│   │   └── DependencyInjection.cs
│   │
│   └── MyApp.Api/                   ← Presentation Layer
│       ├── Controllers/
│       │   └── OrdersController.cs
│       ├── Middleware/
│       │   └── ExceptionHandlingMiddleware.cs
│       └── Program.cs
│
└── tests/
    ├── MyApp.Domain.Tests/
    ├── MyApp.Application.Tests/
    └── MyApp.Integration.Tests/
```

---

## 4. Dependency Injection Setup

```csharp
// Infrastructure/DependencyInjection.cs
public static class InfrastructureServiceExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        return services;
    }
}

// Application/DependencyInjection.cs
public static class ApplicationServiceExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // MediatR tự tìm tất cả handlers trong assembly
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(ApplicationServiceExtensions).Assembly));

        // FluentValidation — Validation tự động qua pipeline behavior
        services.AddValidatorsFromAssembly(typeof(ApplicationServiceExtensions).Assembly);
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

        return services;
    }
}

// Api/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddControllers();
```

---

## 5. Validation Behavior — Xác Thực Qua Pipeline

```csharp
// MediatR Pipeline Behavior — chạy validation trước khi handler xử lý
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var errors = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(e => e != null)
            .ToList();

        if (errors.Any())
            throw new ValidationException(errors);

        return await next();
    }
}

// FluentValidation validator cho Command
public class SubmitOrderCommandValidator : AbstractValidator<SubmitOrderCommand>
{
    public SubmitOrderCommandValidator()
    {
        RuleFor(x => x.OrderId)
            .NotEmpty()
            .WithMessage("OrderId không được để trống.");
    }
}
```

---

## 6. Unit of Work — Đơn Vị Công Việc

```csharp
// Interface trong Application Layer
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

// Implementation trong Infrastructure Layer
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }
}
```

---

## 7. Exception Handling — Xử Lý Lỗi Xuyên Tầng

```csharp
// Domain exceptions — lỗi nghiệp vụ
public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}

public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// Global exception handler middleware
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (DomainException ex)
        {
            context.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new
            {
                errors = ex.Errors.Select(e => new { e.PropertyName, e.ErrorMessage })
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Lỗi không mong muốn xảy ra.");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new { error = "Đã xảy ra lỗi hệ thống." });
        }
    }
}
```

---

## 8. Testing Strategy — Chiến Lược Kiểm Thử

### Domain Layer Tests — Test Hoàn Toàn Không Cần Mock

```csharp
public class OrderTests
{
    [Fact]
    public void Submit_WhenOrderHasItems_ShouldChangeStatusToSubmitted()
    {
        // Arrange
        var order = Order.Create(CustomerId.New());
        order.AddItem(new ProductId(Guid.NewGuid()), 2, new Money(50_000, "VND"));

        // Act
        order.Submit();

        // Assert
        order.Status.Should().Be(OrderStatus.Submitted);
        order.DomainEvents.Should().ContainSingle(e => e is OrderSubmittedEvent);
    }

    [Fact]
    public void Submit_WhenOrderHasNoItems_ShouldThrowDomainException()
    {
        // Arrange
        var order = Order.Create(CustomerId.New());

        // Act & Assert
        order.Invoking(o => o.Submit())
             .Should().Throw<DomainException>()
             .WithMessage("*ít nhất một sản phẩm*");
    }
}
```

### Application Layer Tests — Mock Infrastructure

```csharp
public class SubmitOrderCommandHandlerTests
{
    private readonly Mock<IOrderRepository> _repositoryMock = new();
    private readonly Mock<IUnitOfWork> _unitOfWorkMock = new();
    private readonly Mock<IDomainEventDispatcher> _eventDispatcherMock = new();

    [Fact]
    public async Task Handle_WhenOrderExists_ShouldSubmitAndSave()
    {
        // Arrange
        var order = Order.Create(CustomerId.New());
        order.AddItem(new ProductId(Guid.NewGuid()), 1, new Money(100_000, "VND"));

        _repositoryMock
            .Setup(r => r.GetByIdAsync(order.Id, default))
            .ReturnsAsync(order);

        var handler = new SubmitOrderCommandHandler(
            _repositoryMock.Object,
            _unitOfWorkMock.Object,
            _eventDispatcherMock.Object);

        // Act
        var result = await handler.Handle(new SubmitOrderCommand(order.Id), default);

        // Assert
        result.Status.Should().Be("Submitted");
        _unitOfWorkMock.Verify(u => u.SaveChangesAsync(default), Times.Once);
    }
}
```

---

## 9. Ưu Điểm & Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Testability cao** | Domain và Application test được hoàn toàn không cần database |
| **Thay đổi Infrastructure dễ** | Đổi từ SQL Server sang PostgreSQL chỉ ảnh hưởng Infrastructure |
| **Domain logic rõ ràng** | Logic nghiệp vụ tập trung, dễ tìm, dễ hiểu |
| **Scalable theo team** | Mỗi layer có thể phát triển song song |
| **Framework-agnostic** | Domain không phụ thuộc ASP.NET, EF Core |

### ❌ Nhược Điểm

| Nhược Điểm | Giải Thích |
|-----------|-----------|
| **Boilerplate nhiều** | Nhiều file, nhiều interface cho tính năng đơn giản |
| **Overkill cho CRUD thuần** | Quá phức tạp nếu chỉ là form với database |
| **Learning curve cao** | Team cần thời gian để quen pattern |
| **Over-engineering risk** | Dễ tạo abstraction không cần thiết |

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Tại sao Domain layer không được phụ thuộc Infrastructure?**
> A: Để domain logic có thể test hoàn toàn không cần database, và dễ thay đổi implementation detail mà không ảnh hưởng business logic. Đây là nguyên tắc Dependency Inversion — đảo ngược phụ thuộc.

**Q: Clean Architecture khác gì với MVC thông thường?**
> A: MVC (Model-View-Controller) chỉ là presentation pattern. Clean Architecture định nghĩa cách organize toàn bộ codebase, MVC có thể là Presentation layer trong Clean Architecture.

**Q: Khi nào KHÔNG dùng Clean Architecture?**
> A: CRUD thuần, prototype nhanh, team nhỏ chưa quen pattern. Overhead setup ban đầu không xứng với lợi ích cho project nhỏ.

**Q: Repository Pattern trong Clean Architecture có thể bỏ không?**
> A: Có thể, nhưng Repository giúp domain không biết về EF Core, và giúp test dễ hơn bằng cách mock. Nếu team chấp nhận domain biết về DbContext, có thể bỏ.

---

## 📚 Tài Liệu Tham Khảo

- [Clean Architecture — Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [NuGet: MediatR](https://github.com/jbogard/MediatR)
- [NuGet: FluentValidation](https://fluentvalidation.net/)
- [Ardalis — Clean Architecture template](https://github.com/ardalis/CleanArchitecture)

---

**Cập Nhật Lần Cuối:** 2026-06-02
