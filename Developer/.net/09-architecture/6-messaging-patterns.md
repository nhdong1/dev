# Messaging Patterns — Mẫu Nhắn Tin Bất Đồng Bộ

> Messaging Patterns — Mẫu Nhắn Tin là tập hợp các pattern sử dụng message broker — bộ môi giới tin nhắn để giao tiếp bất đồng bộ giữa các services. Trong .NET ecosystem, MassTransit là abstraction layer phổ biến nhất, hỗ trợ RabbitMQ, Azure Service Bus, Amazon SQS, và Kafka.

---

## 1. Tổng Quan Message-Based Communication

```
┌──────────────┐    publish    ┌────────────────┐    deliver   ┌──────────────┐
│  Producer    │──────────────►│  Message Broker│─────────────►│  Consumer    │
│  (Publisher) │               │  (Queue/Topic) │              │  (Subscriber)│
└──────────────┘               └────────────────┘              └──────────────┘

Message Broker:
- RabbitMQ          — Open source, AMQP protocol, on-premise hoặc cloud
- Azure Service Bus — Azure managed service, enterprise features
- Amazon SQS/SNS    — AWS managed service
- Apache Kafka      — High-throughput event streaming platform
- Redis Streams     — Redis-based lightweight streaming
```

**Lợi ích của messaging:**
- **Decoupling** — Tách biệt: Producer không cần biết consumer là ai
- **Reliability** — Độ tin cậy: Message được lưu trong broker nếu consumer tạm thời không có
- **Scalability** — Mở rộng: Thêm consumer để xử lý tải tăng
- **Async** — Bất đồng bộ: Producer tiếp tục mà không chờ consumer

---

## 2. MassTransit — Message Bus Abstraction

### 2.1 Cài Đặt

```bash
# RabbitMQ transport
dotnet add package MassTransit.RabbitMQ

# Azure Service Bus transport
dotnet add package MassTransit.Azure.ServiceBus.Core

# Amazon SQS transport
dotnet add package MassTransit.AmazonSQS

# Entity Framework integration (Outbox Pattern)
dotnet add package MassTransit.EntityFrameworkCore
```

### 2.2 Cấu Hình MassTransit + RabbitMQ

```csharp
// Program.cs
builder.Services.AddMassTransit(x =>
{
    // Tự động tìm tất cả consumers trong assembly
    x.AddConsumers(typeof(Program).Assembly);

    // Đăng ký Saga
    x.AddSagaStateMachine<OrderSaga, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Pessimistic;
            r.AddDbContext<DbContext, SagaDbContext>((provider, options) =>
            {
                options.UseSqlServer(
                    builder.Configuration.GetConnectionString("DefaultConnection"));
            });
        });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        // Retry policy — chính sách thử lại
        cfg.UseMessageRetry(r => r.Incremental(3,
            initialInterval: TimeSpan.FromSeconds(1),
            intervalIncrement: TimeSpan.FromSeconds(2)));

        // Dead letter queue — hàng đợi thư chết (message không xử lý được)
        cfg.ReceiveEndpoint("order-submitted-queue", e =>
        {
            e.ConfigureConsumer<OrderSubmittedConsumer>(context);
            e.DeadLetterQueueSuffix = "_error";
        });

        // Auto-configure tất cả endpoints
        cfg.ConfigureEndpoints(context);
    });
});
```

### 2.3 Publishing Messages — Publish Tin Nhắn

```csharp
// Integration Event — sự kiện tích hợp giữa các services
// Khác với Domain Event: Integration Event là public contract
public record OrderSubmittedIntegrationEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemData> Items { get; init; } = new();
    public decimal TotalAmount { get; init; }
    public string Currency { get; init; } = "VND";
    public DateTime SubmittedAt { get; init; } = DateTime.UtcNow;
}

// Publish từ Command Handler
public class SubmitOrderCommandHandler : ICommandHandler<SubmitOrderCommand, SubmitOrderResult>
{
    private readonly IOrderRepository _repository;
    private readonly IPublishEndpoint _publishEndpoint;  // MassTransit
    private readonly IUnitOfWork _unitOfWork;

    public SubmitOrderCommandHandler(
        IOrderRepository repository,
        IPublishEndpoint publishEndpoint,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _publishEndpoint = publishEndpoint;
        _unitOfWork = unitOfWork;
    }

    public async Task<SubmitOrderResult> Handle(
        SubmitOrderCommand command,
        CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(command.OrderId, cancellationToken)
            ?? throw new NotFoundException($"Đơn hàng {command.OrderId} không tìm thấy.");

        order.Submit();
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        // Publish integration event
        await _publishEndpoint.Publish(new OrderSubmittedIntegrationEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Items = order.Items.Select(i => new OrderItemData
            {
                ProductId = i.ProductId,
                Quantity = i.Quantity
            }).ToList(),
            TotalAmount = order.TotalAmount.Amount
        }, cancellationToken);

        return new SubmitOrderResult(order.Id, order.Status.ToString());
    }
}
```

### 2.4 Consuming Messages — Nhận Tin Nhắn

```csharp
// Consumer trong Inventory Service
public class OrderSubmittedConsumer : IConsumer<OrderSubmittedIntegrationEvent>
{
    private readonly IInventoryRepository _inventoryRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<OrderSubmittedConsumer> _logger;

    public OrderSubmittedConsumer(
        IInventoryRepository inventoryRepository,
        IUnitOfWork unitOfWork,
        ILogger<OrderSubmittedConsumer> logger)
    {
        _inventoryRepository = inventoryRepository;
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderSubmittedIntegrationEvent> context)
    {
        var message = context.Message;
        _logger.LogInformation("Nhận đơn hàng {OrderId} cần reserve hàng.", message.OrderId);

        foreach (var item in message.Items)
        {
            var inventory = await _inventoryRepository.GetByProductIdAsync(
                item.ProductId, context.CancellationToken);

            if (inventory is null)
            {
                _logger.LogWarning("Không tìm thấy tồn kho cho sản phẩm {ProductId}", item.ProductId);
                // Có thể throw để trigger retry
                throw new InvalidOperationException($"Sản phẩm {item.ProductId} không có trong kho.");
            }

            inventory.Reserve(item.Quantity);
        }

        await _unitOfWork.SaveChangesAsync(context.CancellationToken);
        _logger.LogInformation("Đã reserve hàng cho đơn hàng {OrderId}.", message.OrderId);
    }
}
```

---

## 3. Outbox Pattern — Mẫu Hộp Thư Đi

Giải quyết vấn đề: **Lưu DB thành công nhưng publish message thất bại** (hoặc ngược lại).

### Vấn Đề

```
┌─────────────────────────────────────────────────────────┐
│  Vấn đề: Không có Outbox Pattern                        │
│                                                         │
│  1. SaveChanges() thành công → DB cập nhật ✅           │
│  2. Publish message → Network error ❌                  │
│                                                         │
│  Kết quả: Order được submit trong DB,                   │
│  nhưng Inventory Service không bao giờ biết!            │
│  → Data inconsistency — Không nhất quán dữ liệu        │
└─────────────────────────────────────────────────────────┘
```

### Giải Pháp Outbox Pattern

```
┌─────────────────────────────────────────────────────────┐
│  Với Outbox Pattern                                     │
│                                                         │
│  1. SaveChanges() lưu cả:                               │
│     - Order entity ✅                                   │
│     - OutboxMessage (message chưa gửi) ✅              │
│     (cùng một transaction — giao dịch)                 │
│                                                         │
│  2. Background worker định kỳ:                          │
│     - Đọc OutboxMessages chưa xử lý                    │
│     - Publish lên message broker                        │
│     - Đánh dấu đã xử lý                                │
│                                                         │
│  Guarantee: At-least-once delivery — đảm bảo gửi ít   │
│  nhất một lần (consumer phải idempotent — chịu được    │
│  nhận trùng lặp)                                        │
└─────────────────────────────────────────────────────────┘
```

```csharp
// Outbox Message entity
public class OutboxMessage
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string MessageType { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;  // JSON serialized
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }  // null = chưa xử lý
    public string? Error { get; set; }
    public int RetryCount { get; set; }
}

// Lưu message vào outbox khi save
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; } = null!;
    public DbSet<OutboxMessage> OutboxMessages { get; set; } = null!;

    // Intercept SaveChanges để lưu domain events vào outbox
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // Lấy tất cả domain events trước khi save
        var outboxMessages = ChangeTracker
            .Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .SelectMany(a => a.DomainEvents)
            .Select(domainEvent => new OutboxMessage
            {
                MessageType = domainEvent.GetType().Name,
                Content = JsonSerializer.Serialize(domainEvent, domainEvent.GetType())
            })
            .ToList();

        // Xóa domain events khỏi aggregate
        ChangeTracker.Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .ToList()
            .ForEach(a => a.ClearDomainEvents());

        // Thêm outbox messages
        OutboxMessages.AddRange(outboxMessages);

        // Một transaction — cả order update VÀ outbox messages
        return await base.SaveChangesAsync(cancellationToken);
    }
}

// Background worker xử lý outbox
public class OutboxProcessorWorker : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OutboxProcessorWorker> _logger;

    public OutboxProcessorWorker(
        IServiceProvider serviceProvider,
        ILogger<OutboxProcessorWorker> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessOutboxMessagesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessOutboxMessagesAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publisher = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();

        // Lấy messages chưa xử lý (với for update lock để tránh duplicate processing)
        var messages = await dbContext.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 3)
            .OrderBy(m => m.CreatedAt)
            .Take(50)
            .ToListAsync(cancellationToken);

        foreach (var message in messages)
        {
            try
            {
                var messageType = Type.GetType(message.MessageType)
                    ?? throw new InvalidOperationException($"Không tìm thấy type: {message.MessageType}");

                var eventObject = JsonSerializer.Deserialize(message.Content, messageType)!;
                await publisher.Publish(eventObject, messageType, cancellationToken);

                message.ProcessedAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Lỗi khi xử lý outbox message {MessageId}", message.Id);
                message.RetryCount++;
                message.Error = ex.Message;
            }
        }

        await dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

### Outbox Pattern Với MassTransit

```csharp
// MassTransit có built-in Outbox Pattern hỗ trợ EF Core
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UseSqlServer();  // hoặc UsePostgres()
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.DisableInboxCleanupService();  // optional
    });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.UseMessageRetry(r => r.Incremental(3,
            TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(2)));

        cfg.UseEntityFrameworkOutbox<AppDbContext>(context);

        cfg.ConfigureEndpoints(context);
    });
});
```

---

## 4. Message Patterns — Các Mẫu Tin Nhắn

### 4.1 Point-to-Point — Điểm Đến Điểm (Queue)

Một producer, một consumer. Message chỉ được nhận một lần.

```csharp
// Send — gửi đến một endpoint cụ thể
public class OrderController : ControllerBase
{
    private readonly ISendEndpointProvider _sendEndpointProvider;

    public async Task<IActionResult> ProcessPayment([FromBody] ProcessPaymentRequest request)
    {
        var endpoint = await _sendEndpointProvider.GetSendEndpoint(
            new Uri("queue:payment-processing"));

        await endpoint.Send(new ProcessPaymentCommand
        {
            OrderId = request.OrderId,
            Amount = request.Amount
        });

        return Accepted();
    }
}
```

### 4.2 Publish/Subscribe — Xuất Bản/Đăng Ký (Topic)

Một producer, nhiều consumers. Mỗi subscriber nhận một bản copy của message.

```csharp
// Publish — gửi đến tất cả subscribers
await _publishEndpoint.Publish(new OrderShippedIntegrationEvent
{
    OrderId = orderId,
    TrackingNumber = trackingNumber
});

// Multiple consumers nhận cùng một event:
// NotificationConsumer → gửi email cho khách hàng
// AnalyticsConsumer → ghi lại thống kê
// LoyaltyConsumer → cộng điểm thưởng
```

### 4.3 Request/Response — Yêu Cầu/Phản Hồi

```csharp
// Request từ Order Service sang Inventory Service
public class CheckInventoryConsumer : IConsumer<CheckInventoryRequest>
{
    private readonly IInventoryRepository _repository;

    public async Task Consume(ConsumeContext<CheckInventoryRequest> context)
    {
        var isAvailable = await _repository.IsAvailableAsync(
            context.Message.ProductId,
            context.Message.Quantity,
            context.CancellationToken);

        await context.RespondAsync(new CheckInventoryResponse
        {
            ProductId = context.Message.ProductId,
            IsAvailable = isAvailable
        });
    }
}

// Order Service gọi và chờ response
public class CreateOrderCommandHandler
{
    private readonly IRequestClient<CheckInventoryRequest> _inventoryClient;

    public async Task<CreateOrderResult> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        // Request/Response qua message broker
        var response = await _inventoryClient.GetResponse<CheckInventoryResponse>(
            new CheckInventoryRequest
            {
                ProductId = command.ProductId,
                Quantity = command.Quantity
            }, ct);

        if (!response.Message.IsAvailable)
            throw new DomainException("Sản phẩm không đủ hàng.");

        // ... tiếp tục tạo order
    }
}
```

---

## 5. Idempotency — Tính Bất Biến Khi Nhận Trùng

Consumer phải xử lý được message trùng lặp (at-least-once delivery không đảm bảo exactly-once).

```csharp
// Idempotent consumer — có thể nhận cùng message nhiều lần mà kết quả không thay đổi
public class OrderSubmittedConsumer : IConsumer<OrderSubmittedIntegrationEvent>
{
    private readonly IInventoryRepository _inventoryRepository;
    private readonly IProcessedMessageRepository _processedMessages;
    private readonly IUnitOfWork _unitOfWork;

    public async Task Consume(ConsumeContext<OrderSubmittedIntegrationEvent> context)
    {
        var messageId = context.MessageId ?? context.Message.OrderId;

        // Kiểm tra đã xử lý message này chưa
        if (await _processedMessages.ExistsAsync(messageId, context.CancellationToken))
        {
            // Message đã xử lý → bỏ qua
            return;
        }

        // Xử lý message
        foreach (var item in context.Message.Items)
        {
            var inventory = await _inventoryRepository.GetByProductIdAsync(
                item.ProductId, context.CancellationToken)!;
            inventory.Reserve(item.Quantity);
        }

        // Đánh dấu đã xử lý (trong cùng transaction)
        _processedMessages.Add(new ProcessedMessage { Id = messageId });
        await _unitOfWork.SaveChangesAsync(context.CancellationToken);
    }
}
```

---

## 6. Dead Letter Queue — Hàng Đợi Thư Chết

Messages không xử lý được sau nhiều lần retry sẽ vào DLQ — Dead Letter Queue.

```csharp
// Cấu hình retry và DLQ
x.UsingRabbitMq((context, cfg) =>
{
    cfg.ReceiveEndpoint("order-events", e =>
    {
        // Retry 3 lần với exponential backoff — thời gian chờ tăng dần
        e.UseMessageRetry(r => r.Exponential(
            retryLimit: 3,
            minInterval: TimeSpan.FromSeconds(1),
            maxInterval: TimeSpan.FromMinutes(1),
            intervalDelta: TimeSpan.FromSeconds(2)));

        // Sau khi hết retry → DLQ
        e.DiscardFaultedMessages();  // hoặc để vào _error queue

        e.ConfigureConsumer<OrderSubmittedConsumer>(context);
    });
});

// Monitor DLQ và alert
public class DeadLetterQueueMonitor : BackgroundService
{
    private readonly IRabbitMqManagementClient _managementClient;
    private readonly IAlertService _alertService;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var dlqCount = await _managementClient.GetQueueMessageCountAsync(
                "order-events_error");

            if (dlqCount > 10)
            {
                await _alertService.SendAlertAsync(
                    $"DLQ có {dlqCount} messages chưa xử lý!");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

---

## 7. Azure Service Bus — Dịch Vụ Bus Azure

```csharp
// MassTransit với Azure Service Bus
builder.Services.AddMassTransit(x =>
{
    x.AddConsumers(typeof(Program).Assembly);

    x.UsingAzureServiceBus((context, cfg) =>
    {
        cfg.Host(builder.Configuration["AzureServiceBus:ConnectionString"]);

        // Topic/Subscription model (Publish/Subscribe)
        cfg.Message<OrderSubmittedIntegrationEvent>(m =>
        {
            m.SetEntityName("order-submitted");  // topic name
        });

        cfg.SubscriptionEndpoint<OrderSubmittedIntegrationEvent>(
            subscriptionName: "inventory-service",
            e =>
            {
                e.ConfigureConsumer<OrderSubmittedConsumer>(context);

                // Azure Service Bus có built-in dead letter
                e.DeadLetterOnMessageExpiration = true;
                e.MaxDeliveryCount = 3;
            });

        cfg.ConfigureEndpoints(context);
    });
});
```

---

## 8. So Sánh Message Brokers

| Tiêu Chí | RabbitMQ | Azure Service Bus | Apache Kafka |
|----------|----------|-------------------|--------------|
| **Protocol** | AMQP | AMQP, HTTP | Custom TCP |
| **Throughput** | Cao | Cao | Rất cao |
| **Ordering** | Per queue | Sessions | Per partition |
| **Retention** | Khi consume xong | Configurable | Configurable (ngày/GB) |
| **Use case** | Task queuing, RPC | Enterprise integration | Event streaming, log aggregation |
| **Managed** | Self-hosted hoặc CloudAMQP | Azure managed | MSK (AWS), Confluent |
| **.NET support** | Xuất sắc | Xuất sắc | Tốt (Confluent.Kafka) |

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Outbox Pattern giải quyết vấn đề gì?**
> A: Dual-write problem — vấn đề ghi đôi. Khi cần vừa save vào DB vừa publish message, không có cơ chế đảm bảo cả hai thành công hoặc cả hai fail. Outbox Pattern lưu message vào cùng DB trong cùng transaction, sau đó background worker gửi lên broker. Đảm bảo at-least-once delivery.

**Q: At-least-once delivery khác exactly-once như thế nào?**
> A: At-least-once: message được nhận ít nhất một lần, có thể bị duplicate — trùng lặp. Consumer phải idempotent. Exactly-once: message được nhận đúng một lần, cần distributed transaction hoặc deduplication phức tạp. Hầu hết systems dùng at-least-once + idempotent consumers.

**Q: Khi nào dùng Queue, khi nào dùng Topic?**
> A: Queue (Point-to-Point): một message chỉ một consumer xử lý — dùng cho task distribution, work queue. Topic (Publish-Subscribe): một message nhiều consumers nhận — dùng khi cần notify nhiều services về cùng event.

**Q: Làm thế nào để xử lý message ordering — thứ tự tin nhắn?**
> A: RabbitMQ: single queue single consumer đảm bảo ordering. Azure Service Bus: dùng Sessions để group messages theo session key. Kafka: partition key đảm bảo ordering trong partition. Tổng quát: nếu ordering quan trọng, giới hạn parallelism hoặc dùng partition/session.

---

**Cập Nhật Lần Cuối:** 2026-06-02
