# DDD Fundamentals — Nền Tảng Domain-Driven Design

> DDD — Domain-Driven Design — Thiết Kế Hướng Miền là phương pháp thiết kế phần mềm tập trung vào domain — miền nghiệp vụ — của bài toán, với ngôn ngữ chung giữa developer và domain expert — chuyên gia nghiệp vụ. Được đề xuất bởi Eric Evans trong cuốn "Domain-Driven Design" (2003).

---

## 1. Hai Loại DDD

### Strategic DDD — DDD Chiến Lược (Big Picture)
Tổ chức hệ thống ở cấp cao: chia domain thành các bounded context, xác định context map.

### Tactical DDD — DDD Chiến Thuật (Implementation)
Cách implement cụ thể trong từng bounded context: Aggregate, Entity, Value Object, Domain Events, Repository, Domain Service.

---

## 2. Strategic DDD — DDD Chiến Lược

### 2.1 Ubiquitous Language — Ngôn Ngữ Chung

Developers và business experts dùng **cùng một từ** cho cùng một khái niệm.

```
❌ Sai — Developer dùng từ kỹ thuật, business dùng từ nghiệp vụ:
  Developer: "UserRecord trong DB được đánh dấu isActive = false"
  Business: "Khách hàng bị khóa tài khoản"

✅ Đúng — Cùng ngôn ngữ:
  Cả hai: "Tài khoản khách hàng bị tạm khóa (suspended)"
  Code: customer.Suspend();   // method tên rõ ràng như ngôn ngữ business
```

### 2.2 Bounded Context — Ngữ Cảnh Giới Hạn

Ranh giới rõ ràng cho một domain model cụ thể. Cùng một từ có thể có nghĩa khác nhau trong các context khác nhau.

```
┌─────────────────────────┐    ┌─────────────────────────┐
│   Order Context         │    │   Catalog Context        │
│                         │    │                          │
│   Product = {           │    │   Product = {            │
│     orderedQuantity,    │    │     name, description,   │
│     priceAtOrder        │    │     images, category     │
│   }                     │    │   }                      │
└─────────────────────────┘    └─────────────────────────┘
       ↑ "Product" khác nhau trong mỗi context ↑
```

### 2.3 Context Map — Bản Đồ Ngữ Cảnh

```
┌──────────────┐  Anti-Corruption Layer   ┌──────────────────┐
│  Order       │ ◄─────────────────────── │  Legacy Payment  │
│  Context     │                          │  System          │
└──────────────┘                          └──────────────────┘
       │
       │ Shared Kernel (dùng chung Domain Events)
       ▼
┌──────────────┐  Customer/Supplier       ┌──────────────────┐
│  Shipping    │ ◄─────────────────────── │  Inventory       │
│  Context     │                          │  Context         │
└──────────────┘                          └──────────────────┘
```

**Các loại relationship giữa contexts:**
- **Shared Kernel** — Nhân chung: chia sẻ một phần domain model
- **Customer/Supplier** — Khách/Nhà cung cấp: upstream/downstream dependency
- **Anti-Corruption Layer — ACL — Lớp Chống Tham Nhũng**: isolate khỏi legacy system
- **Open Host Service** — Dịch vụ giao thức mở: public API cho nhiều consumers

---

## 3. Tactical DDD — Building Blocks

### 3.1 Entity — Thực Thể

Có **identity** — danh tính — duy nhất. Hai Entity cùng thuộc tính nhưng khác ID là **hai đối tượng khác nhau**.

```csharp
public class Customer
{
    public CustomerId Id { get; private set; }  // identity xác định Entity
    public string Name { get; private set; }
    public Email Email { get; private set; }
    public CustomerStatus Status { get; private set; }

    // Protected constructor — chỉ tạo qua factory method
    protected Customer() { }

    public static Customer Register(string name, Email email)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new DomainException("Tên khách hàng không được để trống.");

        return new Customer
        {
            Id = CustomerId.New(),
            Name = name,
            Email = email,
            Status = CustomerStatus.Active
        };
    }

    public void ChangeName(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new DomainException("Tên mới không hợp lệ.");
        Name = newName;
    }

    public void Suspend(string reason)
    {
        if (Status == CustomerStatus.Suspended)
            throw new DomainException("Tài khoản đã bị tạm khóa rồi.");
        Status = CustomerStatus.Suspended;
        // có thể raise domain event ở đây
    }
}
```

### 3.2 Value Object — Đối Tượng Giá Trị

**Không có identity**. So sánh theo **giá trị**, bất biến (immutable — không thể thay đổi sau khi tạo).

```csharp
// Value Object dùng C# record (tự so sánh theo giá trị)
public record Email(string Value)
{
    public static Email Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Email không được để trống.");
        if (!value.Contains('@'))
            throw new DomainException("Định dạng email không hợp lệ.");
        return new Email(value.Trim().ToLowerInvariant());
    }
}

public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency = "VND") => new(0, currency);

    public Money Add(Money other)
    {
        EnsureSameCurrency(other);
        return this with { Amount = Amount + other.Amount };
    }

    public Money Subtract(Money other)
    {
        EnsureSameCurrency(other);
        if (Amount < other.Amount)
            throw new DomainException("Số tiền không đủ để trừ.");
        return this with { Amount = Amount - other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };

    private void EnsureSameCurrency(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Không thể thao tác giữa {Currency} và {other.Currency}.");
    }
}

public record Address(string Street, string City, string Province, string PostalCode)
{
    public string FullAddress => $"{Street}, {City}, {Province} {PostalCode}";
}
```

### 3.3 Aggregate — Tập Hợp (Quan Trọng Nhất)

Nhóm các Entity và Value Objects liên quan thành **một đơn vị nhất quán** (consistency boundary — ranh giới nhất quán). Mỗi Aggregate có một **Aggregate Root** — Gốc Tập Hợp — là entry point duy nhất.

```
Aggregate Root
┌─────────────────────────────┐
│  Order (Aggregate Root)     │
│  ┌───────────────────────┐  │
│  │  OrderItem (Entity)   │  │
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │  Money (Value Object) │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

**Quy tắc Aggregate:**
1. Chỉ truy cập thông qua Aggregate Root
2. Lưu toàn bộ Aggregate trong một transaction — giao dịch
3. Chỉ tham chiếu Aggregate khác qua ID, không qua object reference
4. Aggregate phải luôn ở trạng thái nhất quán

```csharp
// Aggregate Root
public class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }  // chỉ lưu ID, không lưu object
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }
    public ShippingAddress ShippingAddress { get; private set; }  // Value Object

    private readonly List<OrderItem> _items = new();
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected Order() { }

    public static Order Create(CustomerId customerId, ShippingAddress shippingAddress)
    {
        var order = new Order
        {
            Id = new OrderId(Guid.NewGuid()),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            TotalAmount = Money.Zero("VND"),
            ShippingAddress = shippingAddress
        };
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    // Tất cả operations đi qua Aggregate Root
    public void AddItem(ProductId productId, string productName, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể thêm sản phẩm vào đơn hàng ở trạng thái Draft.");

        if (quantity <= 0)
            throw new DomainException("Số lượng phải lớn hơn 0.");

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(productId, productName, quantity, unitPrice));
        }

        RecalculateTotal();
    }

    public void RemoveItem(ProductId productId)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể xóa sản phẩm khỏi đơn hàng Draft.");

        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new DomainException($"Sản phẩm {productId.Value} không có trong đơn hàng.");

        _items.Remove(item);
        RecalculateTotal();
    }

    public void Submit()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Chỉ có thể xác nhận đơn hàng Draft.");
        if (!_items.Any())
            throw new DomainException("Đơn hàng phải có ít nhất một sản phẩm.");

        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(Id, CustomerId, TotalAmount));
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            throw new DomainException("Không thể hủy đơn hàng đã giao.");

        Status = OrderStatus.Cancelled;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    private void RecalculateTotal()
    {
        TotalAmount = _items.Aggregate(
            Money.Zero("VND"),
            (acc, item) => acc.Add(item.Total));
    }

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Entity bên trong Aggregate — không thể tạo độc lập từ bên ngoài
public class OrderItem
{
    public OrderItemId Id { get; private set; }
    public ProductId ProductId { get; private set; }
    public string ProductName { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money Total => UnitPrice.Multiply(Quantity);

    internal OrderItem(ProductId productId, string productName, int quantity, Money unitPrice)
    {
        Id = new OrderItemId(Guid.NewGuid());
        ProductId = productId;
        ProductName = productName;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }

    internal void IncreaseQuantity(int amount)
    {
        if (amount <= 0) throw new DomainException("Số lượng tăng phải lớn hơn 0.");
        Quantity += amount;
    }
}
```

---

### 3.4 Domain Events — Sự Kiện Miền

Thông báo rằng **điều gì đó đã xảy ra** trong domain. Kết nối các Aggregates mà không tạo coupling trực tiếp.

```csharp
// Marker interface — interface đánh dấu
public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

// Domain Event cụ thể
public record OrderSubmittedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Money TotalAmount) : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

public record OrderCancelledEvent(
    OrderId OrderId,
    string Reason) : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}
```

```csharp
// Domain Event Handler — xử lý sau khi event xảy ra
// Thường đặt trong Application layer
public class OrderSubmittedEventHandler : INotificationHandler<OrderSubmittedEvent>
{
    private readonly IEmailService _emailService;
    private readonly IInventoryService _inventoryService;

    public OrderSubmittedEventHandler(
        IEmailService emailService,
        IInventoryService inventoryService)
    {
        _emailService = emailService;
        _inventoryService = inventoryService;
    }

    public async Task Handle(
        OrderSubmittedEvent notification,
        CancellationToken cancellationToken)
    {
        // Gửi email xác nhận
        await _emailService.SendOrderConfirmationAsync(
            notification.CustomerId,
            notification.OrderId,
            cancellationToken);

        // Reserve hàng trong kho
        await _inventoryService.ReserveItemsAsync(
            notification.OrderId,
            cancellationToken);
    }
}
```

---

### 3.5 Repository — Kho Dữ Liệu

Interface trong Domain, Implementation trong Infrastructure. Ẩn chi tiết persistence — lưu trữ — khỏi domain.

```csharp
// Repository interface — trong Domain Layer
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken cancellationToken = default);
    Task<List<Order>> GetByCustomerIdAsync(CustomerId customerId, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(OrderId id, CancellationToken cancellationToken = default);
    void Add(Order order);
    void Remove(Order order);
}

// Implementation — trong Infrastructure Layer
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken cancellationToken = default)
        => await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);

    public async Task<List<Order>> GetByCustomerIdAsync(
        CustomerId customerId,
        CancellationToken cancellationToken = default)
        => await _context.Orders
            .Where(o => o.CustomerId == customerId)
            .Include(o => o.Items)
            .ToListAsync(cancellationToken);

    public async Task<bool> ExistsAsync(OrderId id, CancellationToken cancellationToken = default)
        => await _context.Orders.AnyAsync(o => o.Id == id, cancellationToken);

    public void Add(Order order) => _context.Orders.Add(order);
    public void Remove(Order order) => _context.Orders.Remove(order);
}
```

---

### 3.6 Domain Service — Dịch Vụ Miền

Logic nghiệp vụ **không thuộc về một Entity hay Value Object** cụ thể nào. Thường là operations liên quan nhiều Aggregate.

```csharp
// Domain Service — logic nghiệp vụ phức tạp liên quan nhiều entity
public class OrderPricingService
{
    private readonly IDiscountRepository _discountRepository;

    public OrderPricingService(IDiscountRepository discountRepository)
    {
        _discountRepository = discountRepository;
    }

    public async Task<Money> CalculateFinalPriceAsync(
        Order order,
        CustomerId customerId,
        CancellationToken cancellationToken = default)
    {
        var basePrice = order.TotalAmount;
        var discounts = await _discountRepository.GetActiveDiscountsForCustomerAsync(
            customerId, cancellationToken);

        var totalDiscount = discounts
            .Where(d => d.IsApplicable(order))
            .Aggregate(Money.Zero("VND"), (acc, d) => acc.Add(d.CalculateDiscount(basePrice)));

        return basePrice.Subtract(totalDiscount);
    }
}
```

---

## 4. Aggregate Design Guidelines — Hướng Dẫn Thiết Kế Aggregate

### 4.1 Nguyên Tắc Kích Thước Aggregate

```
❌ Aggregate quá lớn (God Aggregate):
Customer
├── Orders (1000+)
│   ├── OrderItems
│   └── Payments
├── Addresses (50+)
└── ReviewHistory (500+)

Vấn đề: Load toàn bộ để update một field, concurrency conflicts cao

✅ Aggregate đúng kích thước:
Customer     Order         Payment
├── Name     ├── Items     ├── Amount
├── Email    ├── Status    ├── Status
└── Status   └── Address   └── CustomerId
```

### 4.2 Quy Tắc Tham Chiếu Giữa Aggregates

```csharp
// ❌ Sai — tham chiếu trực tiếp object
public class Order
{
    public Customer Customer { get; set; }  // coupling chặt
}

// ✅ Đúng — chỉ lưu ID
public class Order
{
    public CustomerId CustomerId { get; private set; }  // loose coupling
}

// Khi cần Customer data, load riêng hoặc dùng Application Service
```

---

## 5. Specification Pattern — Mẫu Đặc Tả

Đóng gói business rule — quy tắc nghiệp vụ — thành object tái sử dụng.

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    Expression<Func<T, bool>> ToExpression(); // cho EF Core query
}

public class ActiveCustomerSpecification : ISpecification<Customer>
{
    public bool IsSatisfiedBy(Customer customer)
        => customer.Status == CustomerStatus.Active;

    public Expression<Func<Customer, bool>> ToExpression()
        => customer => customer.Status == CustomerStatus.Active;
}

public class HighValueOrderSpecification : ISpecification<Order>
{
    private readonly Money _threshold;

    public HighValueOrderSpecification(Money threshold)
        => _threshold = threshold;

    public bool IsSatisfiedBy(Order order)
        => order.TotalAmount.Amount >= _threshold.Amount;

    public Expression<Func<Order, bool>> ToExpression()
        => order => order.TotalAmount.Amount >= _threshold.Amount;
}

// Sử dụng trong Repository
public async Task<List<Order>> GetHighValueOrdersAsync(CancellationToken cancellationToken)
{
    var spec = new HighValueOrderSpecification(new Money(10_000_000, "VND"));
    return await _context.Orders
        .Where(spec.ToExpression())
        .ToListAsync(cancellationToken);
}
```

---

## 6. Dispatch Domain Events — Phân Phối Sự Kiện Miền

```csharp
// Cách 1: Dispatch sau SaveChanges trong Infrastructure
public class AppDbContext : DbContext
{
    private readonly IMediator _mediator;

    public AppDbContext(DbContextOptions options, IMediator mediator) : base(options)
    {
        _mediator = mediator;
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var result = await base.SaveChangesAsync(cancellationToken);

        // Lấy tất cả domain events từ entities đã thay đổi
        var domainEvents = ChangeTracker.Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .SelectMany(a => a.DomainEvents)
            .ToList();

        // Xóa events khỏi entities
        ChangeTracker.Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .ToList()
            .ForEach(a => a.ClearDomainEvents());

        // Dispatch events
        foreach (var domainEvent in domainEvents)
        {
            await _mediator.Publish(domainEvent, cancellationToken);
        }

        return result;
    }
}
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Entity khác Value Object như thế nào?**
> A: Entity có identity duy nhất (ID), tồn tại xuyên suốt lifecycle. Value Object được so sánh theo giá trị, bất biến. Ví dụ: Customer là Entity (cùng tên nhưng khác ID = khác người). Money là Value Object (100_000 VND = 100_000 VND dù là hai object khác nhau).

**Q: Aggregate là gì và tại sao cần nó?**
> A: Aggregate là cluster các objects liên quan, với Aggregate Root làm entry point duy nhất và ranh giới nhất quán. Cần để: (1) đảm bảo business invariants — bất biến nghiệp vụ, (2) làm rõ transaction boundary, (3) tránh concurrent conflicts.

**Q: Tại sao Aggregates chỉ tham chiếu nhau qua ID?**
> A: Tránh load toàn bộ object graph khi chỉ cần một aggregate. Mỗi aggregate là independent — độc lập — có thể scale riêng, test riêng. Reference by ID cũng chuẩn bị cho việc tách microservices sau này.

**Q: Bounded Context khác gì với Microservice?**
> A: Bounded Context là khái niệm logical — logic thuần, không liên quan đến deployment. Microservice là đơn vị deployment — triển khai. Một Bounded Context có thể chạy trong một Microservice, hoặc nhiều Bounded Contexts có thể trong một Microservice (monolith).

---

## 📚 Tài Liệu Tham Khảo

- "Domain-Driven Design" — Eric Evans (Blue Book)
- "Implementing Domain-Driven Design" — Vaughn Vernon (Red Book)
- [Microsoft — DDD with Clean Architecture](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)

---

**Cập Nhật Lần Cuối:** 2026-06-02
