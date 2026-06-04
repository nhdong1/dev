# Microservices Basics — Nền Tảng Kiến Trúc Vi Dịch Vụ

> Microservices — Vi Dịch Vụ là phong cách kiến trúc chia ứng dụng thành tập hợp các service nhỏ, độc lập, mỗi service chạy trong tiến trình riêng và giao tiếp qua network. Mỗi service tập trung vào một business capability — khả năng nghiệp vụ — cụ thể.

---

## 1. Monolith vs Microservices

### Monolith — Đơn Khối

```
┌─────────────────────────────────────────────┐
│              Monolith Application           │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Users   │  │  Orders  │  │ Products │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │         Shared Database              │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
   Deploy toàn bộ cùng lúc
```

**Ưu điểm Monolith:**
- Đơn giản để phát triển ban đầu
- Dễ test, dễ debug
- Không có distributed systems complexity
- Team nhỏ, delivery nhanh

**Nhược điểm khi scale:**
- Deploy toàn bộ chỉ để thay đổi một module nhỏ
- Scale toàn bộ dù chỉ một phần cần thêm tài nguyên
- Tech stack bị lock vào một ngôn ngữ/framework
- Codebase lớn dần, khó maintain

### Microservices — Vi Dịch Vụ

```
        ┌─────────────┐
        │ API Gateway │  ← Entry point duy nhất cho client
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ User   │ │ Order  │ │Product │
│Service │ │Service │ │Service │
│  :5001 │ │  :5002 │ │  :5003 │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
┌───┴──┐  ┌───┴──┐  ┌───┴──┐
│ DB   │  │ DB   │  │ DB   │  ← Mỗi service có DB riêng
└──────┘  └──────┘  └──────┘
           │
    ┌──────┴──────┐
    │ Message Bus │  ← Giao tiếp bất đồng bộ
    │ RabbitMQ    │
    └─────────────┘
```

---

## 2. Decomposition — Cách Phân Tách Service

### 2.1 Phân Tách Theo Business Capability — Khả Năng Nghiệp Vụ

```
E-commerce Platform
├── User Service        — Quản lý người dùng, xác thực
├── Product Service     — Catalog sản phẩm, tìm kiếm
├── Order Service       — Quản lý đơn hàng, checkout
├── Inventory Service   — Quản lý kho hàng
├── Payment Service     — Thanh toán, hoàn tiền
├── Notification Service— Email, SMS, push notifications
└── Shipping Service    — Giao hàng, tracking
```

### 2.2 Phân Tách Theo Bounded Context — DDD

Mỗi Bounded Context trong DDD là một candidate cho một Microservice hoặc một module trong monolith.

```csharp
// Mỗi service là một .NET solution độc lập
OrderService/
├── src/
│   ├── OrderService.Domain/
│   ├── OrderService.Application/
│   ├── OrderService.Infrastructure/
│   └── OrderService.Api/          ← ASP.NET Core API
└── tests/

UserService/
├── src/
│   ├── UserService.Domain/
│   └── UserService.Api/
└── tests/
```

### 2.3 Strangler Fig Pattern — Tách Dần Từ Monolith

```
Giai đoạn 1: Monolith toàn bộ
┌─────────────────────┐
│      Monolith       │
│  Users + Orders + Products │
└─────────────────────┘

Giai đoạn 2: Tách dần từng service
┌──────────────┐    ┌──────────┐
│   Monolith   │    │  User    │
│ Orders +     │    │ Service  │ ← mới tách ra
│ Products     │    │(new)     │
└──────────────┘    └──────────┘

Giai đoạn 3: Hoàn tất
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Order   │ │  User    │ │ Product  │
│ Service  │ │ Service  │ │ Service  │
└──────────┘ └──────────┘ └──────────┘
```

---

## 3. Inter-Service Communication — Giao Tiếp Giữa Các Service

### 3.1 Synchronous — Đồng Bộ (HTTP/gRPC)

```csharp
// HTTP Client với Polly cho retry và circuit breaker
// NuGet: Microsoft.Extensions.Http.Polly

// Program.cs
builder.Services.AddHttpClient<IProductServiceClient, ProductServiceClient>(client =>
{
    client.BaseAddress = new Uri("https://product-service");
})
.AddTransientHttpErrorPolicy(policy => policy
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (outcome, timespan, attempt, context) =>
        {
            // Log retry
        }))
.AddTransientHttpErrorPolicy(policy => policy
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30)));

// Client class
public class ProductServiceClient : IProductServiceClient
{
    private readonly HttpClient _httpClient;

    public ProductServiceClient(HttpClient httpClient)
        => _httpClient = httpClient;

    public async Task<ProductDto?> GetProductAsync(
        Guid productId,
        CancellationToken cancellationToken = default)
    {
        var response = await _httpClient.GetAsync(
            $"/api/products/{productId}", cancellationToken);

        if (response.StatusCode == HttpStatusCode.NotFound)
            return null;

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<ProductDto>(
            cancellationToken: cancellationToken);
    }
}
```

```csharp
// gRPC — Google Remote Procedure Call — Giao Thức Gọi Thủ Tục Từ Xa
// Nhanh hơn HTTP/JSON, dùng Protocol Buffers
// NuGet: Grpc.AspNetCore, Grpc.Net.Client

// product.proto
/*
syntax = "proto3";
package product;

service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
}

message GetProductRequest {
  string product_id = 1;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  double price = 3;
}
*/

// gRPC Server
public class ProductGrpcService : Product.ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;

    public ProductGrpcService(IProductRepository repository)
        => _repository = repository;

    public override async Task<ProductResponse> GetProduct(
        GetProductRequest request,
        ServerCallContext context)
    {
        var product = await _repository.GetByIdAsync(
            Guid.Parse(request.ProductId),
            context.CancellationToken)
            ?? throw new RpcException(new Status(StatusCode.NotFound,
                $"Không tìm thấy sản phẩm {request.ProductId}"));

        return new ProductResponse
        {
            Id = product.Id.ToString(),
            Name = product.Name,
            Price = (double)product.Price.Amount
        };
    }
}
```

### 3.2 Asynchronous — Bất Đồng Bộ (Message Bus)

```csharp
// Publish event khi order được submit
public class SubmitOrderCommandHandler : ICommandHandler<SubmitOrderCommand, SubmitOrderResult>
{
    private readonly IOrderRepository _repository;
    private readonly IPublisher _publisher;  // message publisher

    public async Task<SubmitOrderResult> Handle(
        SubmitOrderCommand command,
        CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException($"Đơn hàng {command.OrderId} không tìm thấy.");

        order.Submit();
        await _repository.SaveAsync(order, cancellationToken);

        // Publish integration event — sự kiện tích hợp sang các service khác
        await _publisher.PublishAsync(new OrderSubmittedIntegrationEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount.Amount,
            Items = order.Items.Select(i => new OrderItemIntegration
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity
            }).ToList()
        }, cancellationToken);

        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}

// Inventory Service nhận event và reserve hàng
public class OrderSubmittedIntegrationEventConsumer
    : IConsumer<OrderSubmittedIntegrationEvent>  // MassTransit consumer
{
    private readonly IInventoryRepository _inventoryRepository;
    private readonly IUnitOfWork _unitOfWork;

    public OrderSubmittedIntegrationEventConsumer(
        IInventoryRepository inventoryRepository,
        IUnitOfWork unitOfWork)
    {
        _inventoryRepository = inventoryRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task Consume(ConsumeContext<OrderSubmittedIntegrationEvent> context)
    {
        var message = context.Message;
        foreach (var item in message.Items)
        {
            var inventory = await _inventoryRepository.GetByProductIdAsync(
                item.ProductId, context.CancellationToken)
                ?? throw new NotFoundException($"Không tìm thấy tồn kho cho sản phẩm {item.ProductId}");

            inventory.Reserve(item.Quantity);
        }
        await _unitOfWork.SaveChangesAsync(context.CancellationToken);
    }
}
```

---

## 4. API Gateway — Cổng API

```
Clients (Browser, Mobile, 3rd Party)
         │
    ┌────┴────────────────────────────────────┐
    │              API Gateway                │
    │                                         │
    │  - Authentication/Authorization         │
    │  - Rate Limiting — Giới hạn tốc độ      │
    │  - Load Balancing — Cân bằng tải        │
    │  - SSL Termination — Kết thúc SSL       │
    │  - Request Routing — Định tuyến         │
    │  - Response Aggregation — Gộp phản hồi  │
    │  - Logging, Tracing                     │
    └────┬────────────┬─────────────┬─────────┘
         │            │             │
    ┌────┴───┐  ┌─────┴──┐  ┌──────┴──┐
    │ Order  │  │  User  │  │Product  │
    │Service │  │Service │  │Service  │
    └────────┘  └────────┘  └─────────┘
```

```csharp
// YARP — Yet Another Reverse Proxy — Reverse Proxy Tích Hợp .NET
// NuGet: Yarp.ReverseProxy

// appsettings.json
/*
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" },
        "Transforms": [{ "PathRemovePrefix": "/api" }]
      },
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": { "Path": "/api/products/{**catch-all}" }
      }
    },
    "Clusters": {
      "orders-cluster": {
        "Destinations": {
          "dest1": { "Address": "http://order-service:5002/" }
        }
      },
      "products-cluster": {
        "Destinations": {
          "dest1": { "Address": "http://product-service:5003/" }
        }
      }
    }
  }
}
*/

// Program.cs cho API Gateway
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// Thêm authentication middleware tại gateway
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity-server";
        options.Audience = "api";
    });

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
```

---

## 5. Service Discovery — Khám Phá Dịch Vụ

```csharp
// Consul — service discovery solution
// NuGet: Winton.Extensions.Configuration.Consul, Consul

// Service đăng ký với Consul khi khởi động
public class ConsulHostedService : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _config;
    private string _registrationId = string.Empty;

    public ConsulHostedService(IConsulClient consulClient, IConfiguration config)
    {
        _consulClient = consulClient;
        _config = config;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var serviceName = _config["Service:Name"]!;
        var servicePort = int.Parse(_config["Service:Port"]!);

        _registrationId = $"{serviceName}-{servicePort}";

        var registration = new AgentServiceRegistration
        {
            ID = _registrationId,
            Name = serviceName,
            Port = servicePort,
            Address = "order-service",
            Check = new AgentServiceCheck
            {
                HTTP = $"http://order-service:{servicePort}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5)
            }
        };

        await _consulClient.Agent.ServiceRegister(registration, cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await _consulClient.Agent.ServiceDeregister(_registrationId, cancellationToken);
    }
}
```

---

## 6. Distributed Tracing — Theo Dõi Phân Tán

Khi request đi qua nhiều services, cần trace để debug và monitor.

```csharp
// OpenTelemetry — tiêu chuẩn theo dõi phân tán
// NuGet: OpenTelemetry.Extensions.Hosting, OpenTelemetry.Instrumentation.AspNetCore
// NuGet: OpenTelemetry.Exporter.Jaeger (hoặc Zipkin, OTLP)

builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProvider =>
    {
        tracerProvider
            .AddSource("OrderService")
            .SetResourceBuilder(
                ResourceBuilder.CreateDefault()
                    .AddService("OrderService", serviceVersion: "1.0.0"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddJaegerExporter(options =>
            {
                options.AgentHost = "jaeger";
                options.AgentPort = 6831;
            });
    })
    .WithMetrics(metricsProvider =>
    {
        metricsProvider
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddPrometheusExporter();  // Expose metrics cho Prometheus
    });

// Tạo custom span trong code
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand, CreateOrderResult>
{
    private static readonly ActivitySource _activitySource = new("OrderService");

    public async Task<CreateOrderResult> Handle(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        using var activity = _activitySource.StartActivity("CreateOrder");
        activity?.SetTag("customer.id", command.CustomerId.ToString());

        // ... xử lý command ...

        activity?.SetTag("order.id", order.Id.ToString());
        activity?.SetStatus(ActivityStatusCode.Ok);

        return result;
    }
}
```

---

## 7. Health Checks — Kiểm Tra Sức Khỏe Service

```csharp
// Mỗi service expose health endpoint cho Kubernetes/load balancer
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "db", "sql" })
    .AddRabbitMQ(
        rabbitConnectionString: builder.Configuration["RabbitMQ:ConnectionString"]!,
        name: "rabbitmq",
        tags: new[] { "messaging" })
    .AddCheck<CustomHealthCheck>("custom", tags: new[] { "custom" });

// Liveness probe — Kubernetes gọi để biết service có đang chạy không
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // chỉ kiểm tra service đang chạy, không check dependencies
});

// Readiness probe — Kubernetes gọi để biết service có sẵn sàng nhận traffic không
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db") || check.Tags.Contains("messaging")
});
```

---

## 8. Saga Pattern — Quản Lý Distributed Transactions

Khi một business operation span qua nhiều services, cần Saga để đảm bảo consistency.

```csharp
// Choreography-based Saga — Saga Điều Phối Qua Events
// Không có central coordinator, mỗi service tự biết phải làm gì

/*
OrderService:     Order.Create() → publish OrderPlaced
InventoryService: nhận OrderPlaced → reserve items → publish InventoryReserved
PaymentService:   nhận InventoryReserved → process payment → publish PaymentProcessed
OrderService:     nhận PaymentProcessed → confirm order → publish OrderConfirmed
ShippingService:  nhận OrderConfirmed → start shipping

Rollback (khi payment fail):
PaymentService:   publish PaymentFailed
InventoryService: nhận PaymentFailed → release items → publish InventoryReleased
OrderService:     nhận InventoryReleased → cancel order
*/

// Orchestration-based Saga — Saga Điều Phối Tập Trung (dùng MassTransit Saga)
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    public State InventoryReserving { get; private set; } = null!;
    public State PaymentProcessing { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<OrderPlacedEvent> OrderPlaced { get; private set; } = null!;
    public Event<InventoryReservedEvent> InventoryReserved { get; private set; } = null!;
    public Event<PaymentProcessedEvent> PaymentProcessed { get; private set; } = null!;
    public Event<PaymentFailedEvent> PaymentFailed { get; private set; } = null!;

    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderPlaced, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderPlaced)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .PublishAsync(ctx => ctx.Init<ReserveInventoryCommand>(new
                {
                    ctx.Message.OrderId,
                    ctx.Message.Items
                }))
                .TransitionTo(InventoryReserving));

        During(InventoryReserving,
            When(InventoryReserved)
                .PublishAsync(ctx => ctx.Init<ProcessPaymentCommand>(new
                {
                    ctx.Message.OrderId,
                    ctx.Saga.TotalAmount
                }))
                .TransitionTo(PaymentProcessing));

        During(PaymentProcessing,
            When(PaymentProcessed)
                .TransitionTo(Completed),
            When(PaymentFailed)
                .PublishAsync(ctx => ctx.Init<ReleaseInventoryCommand>(new
                {
                    ctx.Saga.OrderId
                }))
                .TransitionTo(Failed));
    }
}
```

---

## 9. Khi Nào NÊN và KHÔNG NÊN Dùng Microservices

### ✅ Nên Dùng Khi

| Điều Kiện | Lý Do |
|----------|-------|
| Team > 20 người, nhiều nhóm | Tránh merge conflicts, deploy độc lập |
| Scale requirements khác nhau | Order service cần 10x CPU, User service cần 2x |
| Tech stack diversity cần thiết | AI service dùng Python, API service dùng .NET |
| Different deployment cycles | Payment service deploy thận trọng, Notification service deploy nhanh |

### ❌ Không Nên Dùng Khi

| Điều Kiện | Lý Do |
|----------|-------|
| Team < 10 người | Distributed systems overhead quá lớn cho team nhỏ |
| Chưa có monolith ổn định | "Don't go microservices first" — Martin Fowler |
| Domain chưa rõ ràng | Sai boundary = service sẽ phải refactor tốn kém |
| Không có DevOps capability | Cần infrastructure cho container, orchestration |

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Microservices giải quyết vấn đề gì mà Monolith không thể?**
> A: (1) Scale độc lập từng service theo nhu cầu. (2) Deploy độc lập, giảm risk. (3) Team tự chủ với service của mình. (4) Fault isolation — lỗi một service không ảnh hưởng toàn bộ. (5) Tech diversity — mỗi service có thể dùng ngôn ngữ/DB phù hợp.

**Q: Nhược điểm lớn nhất của Microservices là gì?**
> A: Distributed systems complexity: network failures, distributed transactions, data consistency, tracing, monitoring. Cần đầu tư vào DevOps infrastructure. Operational overhead cao hơn nhiều so với Monolith.

**Q: Làm thế nào để tách Monolith sang Microservices?**
> A: Dùng Strangler Fig Pattern — tách dần từng bounded context. Bắt đầu từ những phần ít phụ thuộc nhất. Đảm bảo có observability tốt trước khi tách. Không cố tách tất cả cùng lúc.

**Q: Database per service — tại sao cần thiết?**
> A: Để đảm bảo loose coupling — mỗi service có thể thay đổi schema DB của mình mà không ảnh hưởng service khác. Tránh tình trạng một service lock table của service khác. Trade-off: không thể dùng SQL JOIN giữa các services, phải dùng API hoặc message.

---

**Cập Nhật Lần Cuối:** 2026-06-02
