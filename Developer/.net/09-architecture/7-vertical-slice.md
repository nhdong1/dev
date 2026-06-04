# Vertical Slice Architecture — Kiến Trúc Lát Cắt Dọc

> Vertical Slice Architecture — Kiến Trúc Lát Cắt Dọc là phương pháp tổ chức code theo **tính năng** (feature) thay vì theo **tầng kỹ thuật** (layer). Mỗi "lát cắt" — slice — chứa toàn bộ code từ UI đến database cho một tính năng cụ thể. Được phổ biến bởi Jimmy Bogard (tác giả AutoMapper, MediatR).

---

## 1. Layered vs Vertical Slice

### Layered Architecture — Kiến Trúc Phân Tầng (Truyền Thống)

```
Cấu trúc theo LAYER (kỹ thuật):

├── Controllers/
│   ├── OrdersController.cs
│   ├── ProductsController.cs
│   └── UsersController.cs
│
├── Services/
│   ├── OrderService.cs
│   ├── ProductService.cs
│   └── UserService.cs
│
├── Repositories/
│   ├── OrderRepository.cs
│   ├── ProductRepository.cs
│   └── UserRepository.cs
│
└── Models/
    ├── Order.cs
    ├── Product.cs
    └── User.cs

❌ Vấn đề: Thêm tính năng "Tạo đơn hàng" phải chạm vào 4 folder khác nhau
```

### Vertical Slice Architecture — Kiến Trúc Lát Cắt Dọc

```
Cấu trúc theo FEATURE (tính năng):

├── Features/
│   ├── Orders/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrderCommand.cs    ← command
│   │   │   ├── CreateOrderHandler.cs   ← handler + validator + logic
│   │   │   └── CreateOrderEndpoint.cs  ← API endpoint
│   │   ├── GetOrder/
│   │   │   ├── GetOrderQuery.cs
│   │   │   └── GetOrderHandler.cs
│   │   └── CancelOrder/
│   │       ├── CancelOrderCommand.cs
│   │       └── CancelOrderHandler.cs
│   │
│   ├── Products/
│   │   ├── GetProductList/
│   │   └── CreateProduct/
│   │
│   └── Users/
│       ├── Register/
│       └── Login/
│
└── Common/
    ├── Database/
    └── Behaviors/

✅ Thêm tính năng mới → tạo folder mới, không đụng code cũ
```

---

## 2. Triển Khai Với MediatR

### 2.1 Cấu Trúc Một Slice

```csharp
// Tất cả trong một file hoặc một folder — CreateOrder/
// Không cần phân chia ra nhiều layer

public static class CreateOrder
{
    // 1. Request/Command
    public record Command(
        Guid CustomerId,
        List<OrderItemRequest> Items,
        AddressDto ShippingAddress) : IRequest<Result>;

    public record OrderItemRequest(Guid ProductId, int Quantity);

    // 2. Response
    public record Result(Guid OrderId, string Status);

    // 3. Validator — FluentValidation
    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.CustomerId).NotEmpty();
            RuleFor(x => x.Items).NotEmpty()
                .WithMessage("Đơn hàng phải có ít nhất một sản phẩm.");
            RuleForEach(x => x.Items).ChildRules(item =>
            {
                item.RuleFor(x => x.Quantity).GreaterThan(0);
            });
        }
    }

    // 4. Handler — business logic ở đây
    public class Handler : IRequestHandler<Command, Result>
    {
        private readonly AppDbContext _db;

        public Handler(AppDbContext db) => _db = db;

        public async Task<Result> Handle(Command request, CancellationToken cancellationToken)
        {
            // Load data cần thiết
            var productIds = request.Items.Select(i => i.ProductId).ToList();
            var products = await _db.Products
                .Where(p => productIds.Contains(p.Id))
                .ToDictionaryAsync(p => p.Id, cancellationToken);

            // Validate products tồn tại
            var missingProducts = productIds.Except(products.Keys).ToList();
            if (missingProducts.Any())
                throw new NotFoundException($"Không tìm thấy sản phẩm: {string.Join(", ", missingProducts)}");

            // Tạo order
            var order = new Order
            {
                Id = Guid.NewGuid(),
                CustomerId = request.CustomerId,
                Status = "Draft",
                ShippingStreet = request.ShippingAddress.Street,
                ShippingCity = request.ShippingAddress.City,
                CreatedAt = DateTime.UtcNow
            };

            foreach (var item in request.Items)
            {
                var product = products[item.ProductId];
                order.Items.Add(new OrderItem
                {
                    Id = Guid.NewGuid(),
                    OrderId = order.Id,
                    ProductId = item.ProductId,
                    ProductName = product.Name,
                    Quantity = item.Quantity,
                    UnitPrice = product.Price
                });
            }

            order.TotalAmount = order.Items.Sum(i => i.Quantity * i.UnitPrice);

            _db.Orders.Add(order);
            await _db.SaveChangesAsync(cancellationToken);

            return new Result(order.Id, order.Status);
        }
    }

    // 5. Endpoint — ASP.NET Core Minimal API
    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (
            Command command,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var result = await sender.Send(command, cancellationToken);
            return Results.Created($"/api/orders/{result.OrderId}", result);
        })
        .WithName("CreateOrder")
        .WithTags("Orders")
        .Produces<Result>(StatusCodes.Status201Created)
        .ProducesValidationProblem();
    }
}
```

### 2.2 GetOrder Slice — Slice Truy Vấn

```csharp
public static class GetOrderById
{
    // Query chỉ đọc — không cần validate nhiều
    public record Query(Guid OrderId) : IRequest<Result>;

    public record Result(
        Guid Id,
        Guid CustomerId,
        string Status,
        decimal TotalAmount,
        DateTime CreatedAt,
        List<ItemResult> Items);

    public record ItemResult(
        Guid ProductId,
        string ProductName,
        int Quantity,
        decimal UnitPrice,
        decimal Total);

    public class Handler : IRequestHandler<Query, Result>
    {
        private readonly AppDbContext _db;

        public Handler(AppDbContext db) => _db = db;

        public async Task<Result> Handle(Query request, CancellationToken cancellationToken)
        {
            // Có thể dùng raw SQL hoặc Dapper ở đây cho query phức tạp
            var result = await _db.Orders
                .AsNoTracking()
                .Where(o => o.Id == request.OrderId)
                .Select(o => new Result(
                    o.Id,
                    o.CustomerId,
                    o.Status,
                    o.TotalAmount,
                    o.CreatedAt,
                    o.Items.Select(i => new ItemResult(
                        i.ProductId,
                        i.ProductName,
                        i.Quantity,
                        i.UnitPrice,
                        i.Quantity * i.UnitPrice)).ToList()))
                .FirstOrDefaultAsync(cancellationToken);

            return result ?? throw new NotFoundException($"Không tìm thấy đơn hàng {request.OrderId}");
        }
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/orders/{id:guid}", async (
            Guid id,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var result = await sender.Send(new Query(id), cancellationToken);
            return Results.Ok(result);
        })
        .WithName("GetOrderById")
        .WithTags("Orders");
    }
}
```

---

## 3. Auto-Registration Endpoints — Đăng Ký Endpoint Tự Động

```csharp
// Interface đánh dấu endpoint
public interface IEndpoint
{
    static abstract void MapEndpoint(IEndpointRouteBuilder app);
}

// Mỗi slice implement
public static class CreateOrder
{
    // ... Command, Handler như trên ...

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (Command command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(command, ct);
            return Results.Created($"/api/orders/{result.OrderId}", result);
        }).WithTags("Orders");
    }
}

// Extension method để đăng ký tất cả endpoints tự động
public static class EndpointExtensions
{
    public static IServiceCollection AddEndpoints(
        this IServiceCollection services,
        Assembly assembly)
    {
        // Tìm tất cả classes có method MapEndpoint
        var endpoints = assembly.GetTypes()
            .Where(t => t.IsAbstract && t.IsSealed)  // static classes
            .SelectMany(t => t.GetMethods(BindingFlags.Public | BindingFlags.Static))
            .Where(m => m.Name == "MapEndpoint" &&
                        m.GetParameters().Length == 1 &&
                        m.GetParameters()[0].ParameterType == typeof(IEndpointRouteBuilder))
            .ToList();

        // Lưu để map sau
        services.AddSingleton<IEnumerable<MethodInfo>>(endpoints);
        return services;
    }

    public static IEndpointRouteBuilder MapEndpoints(this IEndpointRouteBuilder app)
    {
        var endpoints = app.ServiceProvider
            .GetRequiredService<IEnumerable<MethodInfo>>();

        foreach (var endpoint in endpoints)
            endpoint.Invoke(null, new object[] { app });

        return app;
    }
}

// Program.cs — đăng ký tất cả cùng lúc
builder.Services.AddEndpoints(typeof(Program).Assembly);
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

var app = builder.Build();
app.MapEndpoints();  // tự động map tất cả slice endpoints
```

---

## 4. Cross-Cutting Concerns — Mối Quan Tâm Xuyên Suốt

Những thứ dùng chung đặt trong `Common/`:

```csharp
// Common/Behaviors/ValidationBehavior.cs
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

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .SelectMany(v => v.Validate(context).Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}

// Common/Exceptions/
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

// Common/Database/AppDbContext.cs — dùng chung
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; } = null!;
    public DbSet<OrderItem> OrderItems { get; set; } = null!;
    public DbSet<Product> Products { get; set; } = null!;

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
}
```

---

## 5. So Sánh: Vertical Slice vs Clean Architecture

| Tiêu Chí | Vertical Slice | Clean Architecture |
|----------|---------------|-------------------|
| **Tổ chức code** | Theo feature | Theo layer kỹ thuật |
| **Coupling** | Low coupling giữa features | Low coupling giữa layers |
| **Boilerplate** | Ít hơn (no interfaces required) | Nhiều (interfaces + implementations) |
| **Phù hợp** | CRUD + moderate complexity | Complex domain logic |
| **Testability** | Test từng slice độc lập | Test từng layer |
| **Team scalability** | Tốt (mỗi team sở hữu features) | Tốt (mỗi team có thể sở hữu layer) |
| **Reuse** | Ít reuse hơn | Nhiều reuse hơn qua layers |
| **Learning curve** | Thấp | Cao |

---

## 6. Khi Nào Nên Dùng Vertical Slice

### ✅ Nên Dùng Khi

- **CRUD-heavy applications** — Ứng dụng nhiều thao tác CRUD, ít domain logic phức tạp
- **API-first applications** — Mỗi endpoint gần như là một slice hoàn chỉnh
- **Team feature-oriented** — Team chia theo tính năng, không theo kỹ thuật
- **Rapid development** — Cần ship features nhanh với ít overhead
- **Microservices** — Mỗi service nhỏ, Vertical Slice phù hợp hơn là full Clean Architecture

### ❌ Không Nên Dùng Khi

- **Phức tạp domain logic** — Logic nghiệp vụ phức tạp, nhiều business rules → dùng DDD + Clean Architecture
- **Cần nhiều reuse** — Khi business logic cần share giữa nhiều features, khó tránh duplication
- **Large monolith** — Khi monolith lớn, Vertical Slice có thể dẫn đến inconsistency

---

## 7. Hybrid Approach — Cách Tiếp Cận Kết Hợp

Thực tế, có thể kết hợp cả hai:

```
Features/
├── Orders/                  ← Simple CRUD features → Vertical Slice
│   ├── CreateOrder/
│   └── GetOrder/
│
├── Pricing/                 ← Complex domain → Mini Clean Architecture
│   ├── Domain/
│   │   ├── PricingRule.cs
│   │   └── DiscountCalculator.cs
│   ├── Application/
│   │   ├── CalculatePrice/
│   │   └── ApplyDiscount/
│   └── Infrastructure/
│       └── PricingRepository.cs
│
└── Common/
    ├── Database/
    └── Auth/
```

---

## 8. Ví Dụ Thực Tế Đầy Đủ

```csharp
// Features/Products/GetProductList/GetProductList.cs
// Tất cả trong một file
public static class GetProductList
{
    public record Query(
        string? SearchTerm,
        string? Category,
        decimal? MinPrice,
        decimal? MaxPrice,
        int Page = 1,
        int PageSize = 20) : IRequest<PagedResult<ProductSummary>>;

    public record ProductSummary(
        Guid Id,
        string Name,
        string Category,
        decimal Price,
        int StockQuantity);

    public record PagedResult<T>(
        List<T> Items,
        int TotalCount,
        int Page,
        int PageSize)
    {
        public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    }

    public class Handler : IRequestHandler<Query, PagedResult<ProductSummary>>
    {
        private readonly AppDbContext _db;

        public Handler(AppDbContext db) => _db = db;

        public async Task<PagedResult<ProductSummary>> Handle(
            Query request,
            CancellationToken cancellationToken)
        {
            var query = _db.Products.AsNoTracking();

            if (!string.IsNullOrEmpty(request.SearchTerm))
                query = query.Where(p =>
                    p.Name.Contains(request.SearchTerm) ||
                    p.Description.Contains(request.SearchTerm));

            if (!string.IsNullOrEmpty(request.Category))
                query = query.Where(p => p.Category == request.Category);

            if (request.MinPrice.HasValue)
                query = query.Where(p => p.Price >= request.MinPrice.Value);

            if (request.MaxPrice.HasValue)
                query = query.Where(p => p.Price <= request.MaxPrice.Value);

            var totalCount = await query.CountAsync(cancellationToken);

            var items = await query
                .OrderBy(p => p.Name)
                .Skip((request.Page - 1) * request.PageSize)
                .Take(request.PageSize)
                .Select(p => new ProductSummary(
                    p.Id, p.Name, p.Category, p.Price, p.StockQuantity))
                .ToListAsync(cancellationToken);

            return new PagedResult<ProductSummary>(
                items, totalCount, request.Page, request.PageSize);
        }
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/products", async (
            [AsParameters] Query query,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var result = await sender.Send(query, cancellationToken);
            return Results.Ok(result);
        })
        .WithName("GetProductList")
        .WithTags("Products")
        .WithOpenApi();
    }
}
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Vertical Slice Architecture là gì và khác gì với Layered Architecture?**
> A: Vertical Slice tổ chức code theo feature (tính năng), mỗi slice chứa tất cả code cần thiết từ endpoint đến database. Layered Architecture tổ chức theo kỹ thuật (Controllers, Services, Repositories). Vertical Slice giảm coupling giữa features và dễ thêm tính năng mới mà không ảnh hưởng tính năng khác.

**Q: Vertical Slice có dẫn đến code duplication không?**
> A: Có thể, đặc biệt khi nhiều slices cần logic tương tự. Giải pháp: (1) Đặt shared logic vào `Common/`, (2) Dùng private methods, (3) Chấp nhận một chút duplication nếu slices thực ra có business logic khác nhau ("coupling thông qua DRY" đôi khi tệ hơn duplication).

**Q: Bạn có thể kết hợp Vertical Slice với DDD không?**
> A: Có. Vertical Slice là cách tổ chức code, DDD là cách model domain. Các feature phức tạp trong slice có thể dùng DDD concepts như Aggregates, Domain Events. Các feature CRUD đơn giản có thể trực tiếp thao tác với DB.

**Q: Khi nào nên chọn Vertical Slice thay vì Clean Architecture?**
> A: Vertical Slice phù hợp khi: ứng dụng CRUD-heavy, team cần ship nhanh, domain không quá phức tạp. Clean Architecture phù hợp khi: domain logic phức tạp, cần testability cao cho domain, cần độc lập hoàn toàn với framework.

---

**Cập Nhật Lần Cuối:** 2026-06-02
