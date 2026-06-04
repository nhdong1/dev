# Event Sourcing — Nguồn Sự Kiện

> Event Sourcing — Nguồn Sự Kiện là pattern lưu trữ **lịch sử các sự kiện** (events) thay vì lưu **trạng thái hiện tại** của entity. State hiện tại được tái tạo bằng cách replay — phát lại tất cả events theo thứ tự thời gian.

---

## 1. Ý Tưởng Cốt Lõi

### Database Truyền Thống — Lưu State Hiện Tại

```
Orders Table
┌──────┬────────┬──────────┬────────────┐
│ Id   │ Status │ Total    │ UpdatedAt  │
├──────┼────────┼──────────┼────────────┤
│ #123 │ Shipped│ 500,000  │ 2026-06-02 │  ← chỉ lưu state cuối
└──────┴────────┴──────────┴────────────┘
Vấn đề: Không biết quá khứ — ai thay đổi gì lúc nào
```

### Event Sourcing — Lưu Lịch Sử Events

```
OrderEvents Table
┌──────┬───────────────────┬───────────────────────────────┬────────────┐
│ Seq  │ AggregateId       │ EventType                     │ OccurredAt │
├──────┼───────────────────┼───────────────────────────────┼────────────┤
│ 1    │ order-123         │ OrderCreated                  │ 09:00      │
│ 2    │ order-123         │ OrderItemAdded {prod: A, q:2} │ 09:01      │
│ 3    │ order-123         │ OrderItemAdded {prod: B, q:1} │ 09:02      │
│ 4    │ order-123         │ OrderSubmitted                │ 09:05      │
│ 5    │ order-123         │ PaymentReceived {500000 VND}  │ 09:10      │
│ 6    │ order-123         │ OrderShipped {trackingNo}     │ 10:00      │
└──────┴───────────────────┴───────────────────────────────┴────────────┘
State hiện tại = Replay tất cả events từ đầu
```

---

## 2. Triển Khai Event Sourcing Từ Đầu

### 2.1 Domain Events

```csharp
// Base event
public abstract record DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public int Version { get; init; }  // event sequence trong aggregate
}

// Các events cụ thể của Order
public record OrderCreatedEvent(
    Guid OrderId,
    Guid CustomerId,
    string Street,
    string City) : DomainEvent;

public record OrderItemAddedEvent(
    Guid OrderId,
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    string Currency) : DomainEvent;

public record OrderItemRemovedEvent(
    Guid OrderId,
    Guid ProductId) : DomainEvent;

public record OrderSubmittedEvent(
    Guid OrderId,
    decimal TotalAmount,
    string Currency) : DomainEvent;

public record OrderCancelledEvent(
    Guid OrderId,
    string Reason) : DomainEvent;

public record OrderShippedEvent(
    Guid OrderId,
    string TrackingNumber,
    string Carrier) : DomainEvent;
```

### 2.2 Event-Sourced Aggregate — Aggregate Được Cung Cấp Bởi Sự Kiện

```csharp
// Base class cho Event-Sourced Aggregate
public abstract class EventSourcedAggregate
{
    private readonly List<DomainEvent> _uncommittedEvents = new();
    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents.AsReadOnly();

    public Guid Id { get; protected set; }
    public int Version { get; private set; } = -1;  // -1 = chưa lưu lần nào

    // Raise event mới: apply vào state và ghi nhớ để lưu
    protected void RaiseEvent(DomainEvent @event)
    {
        // Gán version
        var eventWithVersion = @event with { Version = Version + 1 };
        Apply(eventWithVersion);  // cập nhật state
        _uncommittedEvents.Add(eventWithVersion);
        Version++;
    }

    // Restore từ events đã lưu (replay)
    public void LoadFromHistory(IEnumerable<DomainEvent> history)
    {
        foreach (var @event in history)
        {
            Apply(@event);
            Version = @event.Version;
        }
    }

    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();

    // Subclass override để apply từng event type
    protected abstract void Apply(DomainEvent @event);
}

// Order Aggregate với Event Sourcing
public class Order : EventSourcedAggregate
{
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal TotalAmount { get; private set; }
    public string Currency { get; private set; } = "VND";
    public string ShippingStreet { get; private set; } = string.Empty;
    public string ShippingCity { get; private set; } = string.Empty;
    private readonly Dictionary<Guid, (string Name, int Qty, decimal Price)> _items = new();
    public IReadOnlyDictionary<Guid, (string Name, int Qty, decimal Price)> Items => _items;

    // Constructor riêng cho Load from history
    private Order() { }

    // Factory method — tạo mới
    public static Order Create(Guid customerId, string street, string city)
    {
        var order = new Order();
        order.RaiseEvent(new OrderCreatedEvent(
            Guid.NewGuid(), customerId, street, city));
        return order;
    }

    // Business methods — chỉ raise events, không mutate state trực tiếp
    public void AddItem(Guid productId, string productName, int quantity, decimal unitPrice, string currency)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể thêm sản phẩm vào đơn hàng Draft.");
        if (quantity <= 0)
            throw new DomainException("Số lượng phải lớn hơn 0.");

        RaiseEvent(new OrderItemAddedEvent(Id, productId, productName, quantity, unitPrice, currency));
    }

    public void Submit()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể submit đơn hàng Draft.");
        if (!_items.Any())
            throw new DomainException("Đơn hàng phải có ít nhất một sản phẩm.");

        RaiseEvent(new OrderSubmittedEvent(Id, TotalAmount, Currency));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new DomainException("Không thể hủy đơn hàng đã giao.");

        RaiseEvent(new OrderCancelledEvent(Id, reason));
    }

    public void Ship(string trackingNumber, string carrier)
    {
        if (Status != OrderStatus.Submitted)
            throw new DomainException("Chỉ có thể giao đơn hàng đã được xác nhận.");

        RaiseEvent(new OrderShippedEvent(Id, trackingNumber, carrier));
    }

    // Apply: thực sự cập nhật state từ event — đây là nguồn sự thật duy nhất
    protected override void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case OrderCreatedEvent e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = OrderStatus.Draft;
                ShippingStreet = e.Street;
                ShippingCity = e.City;
                break;

            case OrderItemAddedEvent e:
                if (_items.TryGetValue(e.ProductId, out var existing))
                    _items[e.ProductId] = (existing.Name, existing.Qty + e.Quantity, e.UnitPrice);
                else
                    _items[e.ProductId] = (e.ProductName, e.Quantity, e.UnitPrice);
                TotalAmount = _items.Sum(i => i.Value.Qty * i.Value.Price);
                break;

            case OrderItemRemovedEvent e:
                _items.Remove(e.ProductId);
                TotalAmount = _items.Sum(i => i.Value.Qty * i.Value.Price);
                break;

            case OrderSubmittedEvent:
                Status = OrderStatus.Submitted;
                break;

            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;

            case OrderShippedEvent:
                Status = OrderStatus.Shipped;
                break;
        }
    }

    // Load từ events đã lưu
    public static Order LoadFromHistory(IEnumerable<DomainEvent> events)
    {
        var order = new Order();
        order.LoadFromHistory(events);
        return order;
    }
}
```

---

### 2.3 Event Store — Kho Lưu Sự Kiện

```csharp
// Event Store interface
public interface IEventStore
{
    Task AppendEventsAsync(
        Guid aggregateId,
        int expectedVersion,
        IEnumerable<DomainEvent> events,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<DomainEvent>> LoadEventsAsync(
        Guid aggregateId,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<DomainEvent>> LoadEventsAsync(
        Guid aggregateId,
        int fromVersion,
        CancellationToken cancellationToken = default);
}

// Event record lưu vào DB
public class StoredEvent
{
    public long GlobalSequence { get; set; }  // global ordering
    public Guid EventId { get; set; }
    public Guid AggregateId { get; set; }
    public string AggregateType { get; set; } = string.Empty;
    public string EventType { get; set; } = string.Empty;
    public string EventData { get; set; } = string.Empty;  // JSON serialized
    public int Version { get; set; }
    public DateTime OccurredAt { get; set; }
}

// SQL Server implementation
public class SqlEventStore : IEventStore
{
    private readonly EventStoreDbContext _context;
    private readonly JsonSerializerOptions _jsonOptions;

    public SqlEventStore(EventStoreDbContext context)
    {
        _context = context;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }

    public async Task AppendEventsAsync(
        Guid aggregateId,
        int expectedVersion,
        IEnumerable<DomainEvent> events,
        CancellationToken cancellationToken = default)
    {
        // Optimistic concurrency check — kiểm tra phiên bản để tránh conflict
        var currentVersion = await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .MaxAsync(e => (int?)e.Version, cancellationToken) ?? -1;

        if (currentVersion != expectedVersion)
            throw new OptimisticConcurrencyException(
                $"Xung đột phiên bản: expected {expectedVersion}, actual {currentVersion}");

        var storedEvents = events.Select(e => new StoredEvent
        {
            EventId = e.EventId,
            AggregateId = aggregateId,
            AggregateType = "Order",
            EventType = e.GetType().Name,
            EventData = JsonSerializer.Serialize(e, e.GetType(), _jsonOptions),
            Version = e.Version,
            OccurredAt = e.OccurredAt
        });

        _context.Events.AddRange(storedEvents);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task<IReadOnlyList<DomainEvent>> LoadEventsAsync(
        Guid aggregateId,
        CancellationToken cancellationToken = default)
    {
        var stored = await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync(cancellationToken);

        return stored.Select(Deserialize).ToList();
    }

    public async Task<IReadOnlyList<DomainEvent>> LoadEventsAsync(
        Guid aggregateId,
        int fromVersion,
        CancellationToken cancellationToken = default)
    {
        var stored = await _context.Events
            .Where(e => e.AggregateId == aggregateId && e.Version >= fromVersion)
            .OrderBy(e => e.Version)
            .ToListAsync(cancellationToken);

        return stored.Select(Deserialize).ToList();
    }

    private DomainEvent Deserialize(StoredEvent stored)
    {
        // Map event type name → .NET type
        var eventType = Type.GetType($"Domain.Events.{stored.EventType}, Domain")
            ?? throw new InvalidOperationException($"Không tìm thấy type cho event: {stored.EventType}");

        return (DomainEvent)JsonSerializer.Deserialize(stored.EventData, eventType, _jsonOptions)!;
    }
}
```

---

### 2.4 Event-Sourced Repository

```csharp
public class EventSourcedOrderRepository
{
    private readonly IEventStore _eventStore;

    public EventSourcedOrderRepository(IEventStore eventStore)
        => _eventStore = eventStore;

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        var events = await _eventStore.LoadEventsAsync(id, cancellationToken);
        if (!events.Any()) return null;

        return Order.LoadFromHistory(events);
    }

    public async Task SaveAsync(Order order, CancellationToken cancellationToken = default)
    {
        if (!order.UncommittedEvents.Any()) return;

        var expectedVersion = order.Version - order.UncommittedEvents.Count;

        await _eventStore.AppendEventsAsync(
            order.Id,
            expectedVersion,
            order.UncommittedEvents,
            cancellationToken);

        order.ClearUncommittedEvents();
    }
}
```

---

## 3. Snapshots — Ảnh Chụp Trạng Thái

Khi số lượng events lớn (vài trăm đến vài nghìn), replay từ đầu chậm. Snapshot lưu state tại một thời điểm để tăng tốc.

```csharp
public class OrderSnapshot
{
    public Guid AggregateId { get; set; }
    public int Version { get; set; }          // version tại lúc snapshot
    public string StateJson { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

public class SnapshotOrderRepository
{
    private readonly IEventStore _eventStore;
    private readonly ISnapshotStore _snapshotStore;
    private const int SnapshotThreshold = 50;  // snapshot sau mỗi 50 events

    public SnapshotOrderRepository(IEventStore eventStore, ISnapshotStore snapshotStore)
    {
        _eventStore = eventStore;
        _snapshotStore = snapshotStore;
    }

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        // 1. Thử load snapshot mới nhất
        var snapshot = await _snapshotStore.GetLatestAsync(id, cancellationToken);

        Order? order;
        if (snapshot is not null)
        {
            // 2. Restore từ snapshot
            order = JsonSerializer.Deserialize<Order>(snapshot.StateJson)!;

            // 3. Load chỉ events SAU snapshot version
            var recentEvents = await _eventStore.LoadEventsAsync(
                id, snapshot.Version + 1, cancellationToken);

            if (recentEvents.Any())
                order.LoadFromHistory(recentEvents);
        }
        else
        {
            // Không có snapshot — replay từ đầu
            var allEvents = await _eventStore.LoadEventsAsync(id, cancellationToken);
            if (!allEvents.Any()) return null;
            order = Order.LoadFromHistory(allEvents);
        }

        return order;
    }

    public async Task SaveAsync(Order order, CancellationToken cancellationToken = default)
    {
        await _eventStore.AppendEventsAsync(
            order.Id,
            order.Version - order.UncommittedEvents.Count,
            order.UncommittedEvents,
            cancellationToken);

        // Tạo snapshot nếu cần
        if (order.Version % SnapshotThreshold == 0)
        {
            await _snapshotStore.SaveAsync(new OrderSnapshot
            {
                AggregateId = order.Id,
                Version = order.Version,
                StateJson = JsonSerializer.Serialize(order),
                CreatedAt = DateTime.UtcNow
            }, cancellationToken);
        }

        order.ClearUncommittedEvents();
    }
}
```

---

## 4. Projections — Chiếu Sự Kiện Sang Read Model

Event Sourcing thường đi kèm với CQRS: Write side dùng events, Read side là projections — chiếu từ events sang dạng dễ query.

```csharp
// Projection: build read model từ events
public class OrderProjection
{
    // In-memory read model (production dùng database)
    private readonly Dictionary<Guid, OrderReadModel> _orders = new();

    public void Apply(OrderCreatedEvent @event)
    {
        _orders[@event.OrderId] = new OrderReadModel
        {
            Id = @event.OrderId,
            CustomerId = @event.CustomerId,
            Status = "Draft",
            TotalAmount = 0,
            CreatedAt = @event.OccurredAt
        };
    }

    public void Apply(OrderItemAddedEvent @event)
    {
        if (_orders.TryGetValue(@event.OrderId, out var order))
        {
            order.ItemCount++;
            order.TotalAmount += @event.Quantity * @event.UnitPrice;
        }
    }

    public void Apply(OrderSubmittedEvent @event)
    {
        if (_orders.TryGetValue(@event.OrderId, out var order))
            order.Status = "Submitted";
    }

    public void Apply(OrderShippedEvent @event)
    {
        if (_orders.TryGetValue(@event.OrderId, out var order))
        {
            order.Status = "Shipped";
            order.TrackingNumber = @event.TrackingNumber;
        }
    }

    public OrderReadModel? GetById(Guid id)
        => _orders.TryGetValue(id, out var order) ? order : null;
}

public class OrderReadModel
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public int ItemCount { get; set; }
    public string? TrackingNumber { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

```csharp
// Event processor — subscribe và process events để update projection
public class OrderProjectionProcessor : IHostedService
{
    private readonly IEventStore _eventStore;
    private readonly OrderProjectionRepository _projectionRepository;
    private long _lastProcessedSequence = 0;
    private Timer? _timer;

    public OrderProjectionProcessor(
        IEventStore eventStore,
        OrderProjectionRepository projectionRepository)
    {
        _eventStore = eventStore;
        _projectionRepository = projectionRepository;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new Timer(ProcessNewEvents, null, TimeSpan.Zero, TimeSpan.FromSeconds(1));
        return Task.CompletedTask;
    }

    private async void ProcessNewEvents(object? state)
    {
        var newEvents = await _eventStore.GetEventsAfterSequenceAsync(_lastProcessedSequence);
        foreach (var @event in newEvents)
        {
            await _projectionRepository.ApplyAsync(@event);
            _lastProcessedSequence = @event.GlobalSequence;
        }
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _timer?.Dispose();
        return Task.CompletedTask;
    }
}
```

---

## 5. Time Travel — Xem Lại Trạng Thái Quá Khứ

```csharp
// Xem trạng thái của Order tại một thời điểm bất kỳ
public class OrderTimeTravel
{
    private readonly IEventStore _eventStore;

    public OrderTimeTravel(IEventStore eventStore) => _eventStore = eventStore;

    public async Task<Order?> GetStateAtAsync(
        Guid orderId,
        DateTime pointInTime,
        CancellationToken cancellationToken = default)
    {
        var allEvents = await _eventStore.LoadEventsAsync(orderId, cancellationToken);

        // Chỉ lấy events xảy ra trước thời điểm cần xem
        var eventsUntilPoint = allEvents
            .Where(e => e.OccurredAt <= pointInTime)
            .ToList();

        if (!eventsUntilPoint.Any()) return null;

        return Order.LoadFromHistory(eventsUntilPoint);
    }
}

// Sử dụng: "Trạng thái đơn hàng 123 lúc 9:05 sáng là gì?"
var orderAt9Am = await timeTravel.GetStateAtAsync(
    orderId: Guid.Parse("..."),
    pointInTime: new DateTime(2026, 6, 2, 9, 5, 0));
```

---

## 6. EventStoreDB — Database Chuyên Dụng

```csharp
// Dùng EventStoreDB — database được thiết kế riêng cho Event Sourcing
// NuGet: EventStore.Client.Grpc.Streams

public class EventStoreDbEventStore : IEventStore
{
    private readonly EventStoreClient _client;

    public EventStoreDbEventStore(EventStoreClient client) => _client = client;

    public async Task AppendEventsAsync(
        Guid aggregateId,
        int expectedVersion,
        IEnumerable<DomainEvent> events,
        CancellationToken cancellationToken = default)
    {
        var streamName = $"order-{aggregateId}";
        var revision = expectedVersion == -1
            ? StreamRevision.None
            : StreamRevision.FromInt64(expectedVersion);

        var eventData = events.Select(e => new EventData(
            Uuid.NewUuid(),
            e.GetType().Name,
            JsonSerializer.SerializeToUtf8Bytes(e, e.GetType())));

        await _client.AppendToStreamAsync(
            streamName,
            revision,
            eventData,
            cancellationToken: cancellationToken);
    }

    public async Task<IReadOnlyList<DomainEvent>> LoadEventsAsync(
        Guid aggregateId,
        CancellationToken cancellationToken = default)
    {
        var streamName = $"order-{aggregateId}";
        var result = _client.ReadStreamAsync(
            Direction.Forwards,
            streamName,
            StreamPosition.Start,
            cancellationToken: cancellationToken);

        var events = new List<DomainEvent>();
        await foreach (var resolvedEvent in result)
        {
            var eventType = Type.GetType($"Domain.Events.{resolvedEvent.Event.EventType}, Domain")!;
            var domainEvent = (DomainEvent)JsonSerializer.Deserialize(
                resolvedEvent.Event.Data.Span, eventType)!;
            events.Add(domainEvent);
        }

        return events;
    }
}
```

---

## 7. Ưu Điểm & Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Audit trail đầy đủ** | Toàn bộ lịch sử thay đổi được lưu, không mất dữ liệu |
| **Time travel** | Xem lại trạng thái tại bất kỳ thời điểm nào |
| **Debugging dễ hơn** | Biết chính xác sequence of events dẫn đến state hiện tại |
| **Event-driven** | Events là nguồn sự thật, dễ tích hợp với message bus |
| **Không mất dữ liệu** | Không xóa — append-only store |

### ❌ Nhược Điểm

| Nhược Điểm | Giải Thích |
|-----------|-----------|
| **Complexity cao** | Khó implement đúng, nhiều edge cases |
| **Eventual consistency** | Projection có độ trễ so với write side |
| **Schema evolution** | Khi event schema thay đổi, cần xử lý backward compatibility |
| **Replay chậm** | Khi events nhiều, snapshot cần thiết nhưng tăng thêm phức tạp |
| **Query khó** | Không thể query trực tiếp state, cần build projection |

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Event Sourcing lưu gì vào database?**
> A: Chỉ lưu events — các sự kiện đã xảy ra — theo thứ tự thời gian. Không lưu state hiện tại. State được tái tạo bằng cách replay tất cả events từ đầu.

**Q: Tại sao cần Snapshot?**
> A: Khi số lượng events tăng lên (hàng trăm, hàng nghìn), replay từ đầu mỗi lần load aggregate rất chậm. Snapshot lưu state tại một version nhất định, chỉ cần replay events sau snapshot đó.

**Q: Event Sourcing và CQRS có bắt buộc phải dùng chung không?**
> A: Không bắt buộc nhưng thường đi kèm. Event Sourcing là cách lưu state. CQRS giúp build read model từ events (projection). Dùng chung cho phép query hiệu quả từ read model trong khi write model dùng event store.

**Q: Làm thế nào khi event schema cần thay đổi?**
> A: Có nhiều cách: (1) Upcasting — chuyển đổi event cũ khi load, (2) Tạo event type mới và deprecate cái cũ, (3) Migration script chuyển đổi toàn bộ events, (4) Dùng weak schema (JSON) để linh hoạt hơn.

---

**Cập Nhật Lần Cuối:** 2026-06-02
