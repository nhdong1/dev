# System Design Scenarios — Kịch Bản Thiết Kế Hệ Thống

> 5 kịch bản thiết kế hệ thống thực tế cho phỏng vấn Senior/Mid .NET Developer. Mỗi kịch bản bao gồm: yêu cầu, kiến trúc, technology choices và trade-off.

---

## Framework Trả Lời System Design

```
Bước 1 (5 phút): Clarify Requirements
  • Scale: bao nhiêu users? bao nhiêu requests/second?
  • Consistency vs Availability: ưu tiên gì?
  • Read-heavy hay Write-heavy?
  • Latency requirement (P99)?
  • Budget constraint?

Bước 2 (10 phút): High-Level Design
  • Vẽ sơ đồ các thành phần chính
  • Xác định data flow
  • Chọn database sơ bộ

Bước 3 (20 phút): Deep Dive
  • API design
  • Database schema
  • Caching strategy
  • Async processing

Bước 4 (10 phút): Scale & Failure Handling
  • Bottleneck ở đâu?
  • Horizontal scaling plan
  • Failure scenarios
```

---

## Kịch Bản 1 — Thiết Kế URL Shortener (như bit.ly)

### Yêu Cầu

```
Functional Requirements — Yêu Cầu Chức Năng:
• Nhận long URL → trả về short URL (6-7 ký tự)
• Khi truy cập short URL → redirect về long URL
• Custom alias tùy chọn
• Expiry time tùy chọn
• Analytics: click count, geo, device

Non-Functional Requirements — Yêu Cầu Phi Chức Năng:
• 100M URLs tạo/ngày
• 10:1 read/write ratio → 1B redirects/ngày
• P99 latency < 10ms cho redirect
• Availability: 99.9%
```

### High-Level Architecture

```
┌─────────┐    ┌───────────┐    ┌──────────────┐
│ Client  │────│ API GW    │────│ Write Service│──→ PostgreSQL
└─────────┘    │ (nginx)   │    └──────────────┘
               │           │    ┌──────────────┐
               │           │────│ Read Service │──→ Redis Cache
               └───────────┘    └──────────────┘
```

### Deep Dive — Chi Tiết Thiết Kế

**Short URL Generation — Tạo Short URL:**

```csharp
// Approach 1: Base62 encoding từ auto-increment ID
// ID: 1,000,000 → base62 → "4c92"
// Ưu điểm: không collision, đơn giản
// Nhược điểm: sequential → đoán được URL tiếp theo

public string Encode(long id)
{
    const string chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    var sb = new StringBuilder();
    while (id > 0)
    {
        sb.Insert(0, chars[(int)(id % 62)]);
        id /= 62;
    }
    return sb.ToString().PadLeft(7, '0');
}

// Approach 2: Random 7-char string + collision check
// Ưu điểm: không đoán được, distributed generation
// Nhược điểm: có thể collision (dù ít)

public string GenerateAlias()
{
    const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    return new string(Enumerable.Repeat(chars, 7)
        .Select(s => s[RandomNumberGenerator.GetInt32(s.Length)])
        .ToArray());
}
```

**Database Schema:**

```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    short_alias VARCHAR(10) NOT NULL UNIQUE,
    long_url    TEXT NOT NULL,
    user_id     BIGINT REFERENCES users(id),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    expires_at  TIMESTAMPTZ,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_short_alias ON urls(short_alias);
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;
```

**Redirect Flow — Luồng Redirect:**

```csharp
// Controller
[HttpGet("/{alias}")]
public async Task<IActionResult> Redirect(string alias)
{
    // 1. Check Redis cache trước
    var longUrl = await _cache.GetStringAsync($"url:{alias}");
    
    if (longUrl == null)
    {
        // 2. Cache miss — query DB
        var url = await _db.Urls
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.ShortAlias == alias 
                                   && (u.ExpiresAt == null || u.ExpiresAt > DateTime.UtcNow));
        
        if (url == null) return NotFound();
        
        longUrl = url.LongUrl;
        
        // 3. Populate cache (TTL 24h)
        await _cache.SetStringAsync($"url:{alias}", longUrl,
            new DistributedCacheEntryOptions 
            { 
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) 
            });
    }
    
    // 4. Fire-and-forget analytics (không block redirect)
    _ = Task.Run(() => _analyticsService.RecordClickAsync(alias));
    
    // 5. 301 Permanent vs 302 Temporary
    return RedirectPermanent(longUrl); // 301 — browser cache redirect
}
```

**Scaling Strategy — Chiến Lược Mở Rộng:**

```
Read Path (redirect — 1B/ngày ≈ 11,500 req/s):
  • Redis cache với 100M entries (≈ 10GB RAM)
  • CDN caching cho popular URLs
  • Read replicas PostgreSQL
  • Horizontal scale read service (K8s HPA)

Write Path (tạo URL — 100M/ngày ≈ 1,200 req/s):
  • Vertical scale DB đủ cho write
  • Nếu cần: message queue (Kafka) để async processing

Analytics:
  • Không write trực tiếp vào DB khi redirect (quá chậm)
  • Ghi vào Kafka → Flink/Spark processing → ClickHouse/BigQuery
```

---

## Kịch Bản 2 — Thiết Kế Hệ Thống E-Commerce Cart & Checkout

### Yêu Cầu

```
Functional:
• Thêm/xóa/cập nhật giỏ hàng
• Hiển thị giá real-time, kiểm tra tồn kho
• Checkout: payment + order creation + inventory deduction
• Flash sale: 10,000 users mua cùng lúc

Non-Functional:
• Cart operations: < 50ms
• Checkout: < 2 giây
• Inventory consistency: KHÔNG oversell
• 99.99% availability
```

### Kiến Trúc Tổng Quan

```
Client → API Gateway → ┌─ Cart Service (Redis)
                        ├─ Product Service (PostgreSQL + Redis)
                        ├─ Inventory Service (PostgreSQL, pessimistic locking)
                        ├─ Payment Service (Stripe/VNPay)
                        └─ Order Service (PostgreSQL + Outbox)
                               ↓
                          Message Bus (RabbitMQ)
                               ↓
                     ┌─ Email Service
                     ├─ Notification Service
                     └─ Analytics Service
```

### Cart Service — Dịch Vụ Giỏ Hàng

```csharp
// Lưu cart trong Redis (TTL 7 ngày)
public class CartService
{
    private readonly IDatabase _redis;
    
    private string CartKey(string userId) => $"cart:{userId}";
    
    public async Task AddItemAsync(string userId, CartItem item)
    {
        var key = CartKey(userId);
        
        // Redis Hash: field = productId, value = JSON(CartItem)
        await _redis.HashSetAsync(key,
            item.ProductId.ToString(),
            JsonSerializer.Serialize(item));
        
        await _redis.KeyExpireAsync(key, TimeSpan.FromDays(7));
    }
    
    public async Task<Cart> GetCartAsync(string userId)
    {
        var fields = await _redis.HashGetAllAsync(CartKey(userId));
        var items = fields
            .Select(f => JsonSerializer.Deserialize<CartItem>(f.Value!)!)
            .ToList();
        return new Cart(userId, items);
    }
}
```

### Checkout Flow — Luồng Thanh Toán

**Vấn đề quan trọng: Oversell Prevention — Phòng Chống Bán Quá Số Lượng**

```csharp
// Approach 1: Pessimistic Locking — Khóa Bi Quan (phù hợp tải vừa)
public async Task<CheckoutResult> CheckoutAsync(CheckoutRequest request)
{
    await using var transaction = await _db.Database.BeginTransactionAsync(
        IsolationLevel.Serializable); // Mức độ cô lập cao nhất
    
    try
    {
        foreach (var item in request.Items)
        {
            // SELECT ... FOR UPDATE — lock row
            var inventory = await _db.Inventories
                .FromSqlRaw("SELECT * FROM inventories WHERE product_id = {0} FOR UPDATE",
                    item.ProductId)
                .FirstOrDefaultAsync();
            
            if (inventory == null || inventory.Available < item.Quantity)
                throw new InsufficientStockException(item.ProductId);
            
            inventory.Available -= item.Quantity;
            inventory.Reserved += item.Quantity;
        }
        
        var order = CreateOrder(request);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        
        await transaction.CommitAsync();
        return CheckoutResult.Success(order.Id);
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}

// Approach 2: Redis Atomic Decrement — cho Flash Sale (tải rất cao)
public async Task<bool> ReserveInventoryAsync(int productId, int quantity)
{
    var key = $"inventory:{productId}";
    
    // Lua script — atomic check-and-decrement
    var script = @"
        local current = tonumber(redis.call('GET', KEYS[1]))
        if current == nil or current < tonumber(ARGV[1]) then
            return 0
        end
        redis.call('DECRBY', KEYS[1], ARGV[1])
        return 1";
    
    var result = (int)await _redis.ScriptEvaluateAsync(
        script, 
        new RedisKey[] { key }, 
        new RedisValue[] { quantity });
    
    return result == 1;
    // Sync về DB bất đồng bộ qua message queue
}
```

### Outbox Pattern — Đảm Bảo Consistency Giữa DB và Message Bus

```csharp
// Vấn đề: Save order thành công nhưng publish event thất bại → mất event
// Giải pháp: Outbox Pattern — lưu event vào DB cùng transaction

public class OrderService
{
    public async Task<Order> CreateOrderAsync(CreateOrderCommand cmd)
    {
        await using var tx = await _db.Database.BeginTransactionAsync();
        
        var order = new Order(cmd);
        _db.Orders.Add(order);
        
        // Lưu outbox event trong CÙNG transaction
        _db.OutboxMessages.Add(new OutboxMessage
        {
            EventType = nameof(OrderCreatedEvent),
            Payload = JsonSerializer.Serialize(new OrderCreatedEvent(order)),
            CreatedAt = DateTime.UtcNow
        });
        
        await _db.SaveChangesAsync(); // Atomic: order + outbox
        await tx.CommitAsync();
        
        return order;
    }
}

// Background service: polling outbox → publish to RabbitMQ
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(ct);
            
            foreach (var msg in messages)
            {
                await _bus.PublishAsync(msg.EventType, msg.Payload, ct);
                msg.ProcessedAt = DateTime.UtcNow;
            }
            
            await _db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

---

## Kịch Bản 3 — Thiết Kế Rate Limiter — Giới Hạn Tốc Độ

### Yêu Cầu

```
• Giới hạn API calls: 100 requests / minute / user
• Giới hạn theo IP cho public endpoints
• Distributed: nhiều server phải share limit
• Header trả về: X-RateLimit-Remaining, X-RateLimit-Reset
• Khi vượt limit: HTTP 429 Too Many Requests
```

### Thuật Toán So Sánh

```
Fixed Window   — Cửa Sổ Cố Định:
  • Đơn giản nhất
  • Vấn đề: burst ở ranh giới cửa sổ
  • Ví dụ: 100 req/min, nhưng 100 req cuối phút + 100 req đầu phút = 200 req trong 2 giây

Sliding Window — Cửa Sổ Trượt:
  • Chính xác hơn, không có burst problem
  • Tốn bộ nhớ hơn (Redis Sorted Set)

Token Bucket   — Thùng Token (phổ biến nhất):
  • Token được thêm vào theo rate (ví dụ: 1 token/600ms = 100/phút)
  • Mỗi request tiêu thụ 1 token
  • Cho phép burst trong giới hạn bucket size

Leaky Bucket   — Thùng Rỉ:
  • Queue request, process với rate cố định
  • Smooth output, không burst
```

### Triển Khai với Redis — Sliding Window

```csharp
public class RateLimiterService
{
    private readonly IDatabase _redis;
    
    public async Task<RateLimitResult> CheckRateLimitAsync(
        string userId, 
        int limitPerMinute = 100)
    {
        var key = $"ratelimit:{userId}";
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowMs = 60_000; // 1 minute
        var windowStart = now - windowMs;
        
        // Sliding window với Redis Sorted Set
        var pipe = _redis.CreateBatch();
        
        // 1. Xóa entries cũ hơn 1 phút
        var removeTask = pipe.SortedSetRemoveRangeByScoreAsync(key, 0, windowStart);
        
        // 2. Thêm request hiện tại (score = timestamp)
        var addTask = pipe.SortedSetAddAsync(key, Guid.NewGuid().ToString(), now);
        
        // 3. Đếm requests trong window
        var countTask = pipe.SortedSetLengthAsync(key);
        
        // 4. Set TTL
        var expireTask = pipe.KeyExpireAsync(key, TimeSpan.FromMinutes(2));
        
        pipe.Execute();
        await Task.WhenAll(removeTask, addTask, countTask, expireTask);
        
        var count = await countTask;
        var remaining = Math.Max(0, limitPerMinute - count);
        var resetAt = DateTimeOffset.UtcNow.AddMinutes(1);
        
        return new RateLimitResult(
            IsAllowed: count <= limitPerMinute,
            Remaining: (int)remaining,
            ResetAt: resetAt,
            TotalRequests: (int)count
        );
    }
}

// ASP.NET Core Middleware
public class RateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly RateLimiterService _rateLimiter;

    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
                     ?? context.Connection.RemoteIpAddress?.ToString()
                     ?? "anonymous";
        
        var result = await _rateLimiter.CheckRateLimitAsync(userId);
        
        context.Response.Headers["X-RateLimit-Limit"] = "100";
        context.Response.Headers["X-RateLimit-Remaining"] = result.Remaining.ToString();
        context.Response.Headers["X-RateLimit-Reset"] = result.ResetAt.ToUnixTimeSeconds().ToString();
        
        if (!result.IsAllowed)
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["Retry-After"] = "60";
            await context.Response.WriteAsJsonAsync(new { error = "Too many requests" });
            return;
        }
        
        await _next(context);
    }
}
```

---

## Kịch Bản 4 — Thiết Kế Notification System — Hệ Thống Thông Báo

### Yêu Cầu

```
Functional:
• Gửi thông báo qua: Email, SMS, Push Notification, In-App
• Template với variable substitution
• Scheduled notifications — lên lịch gửi
• Batch notifications — gửi hàng loạt (newsletter)
• Retry khi thất bại

Non-Functional:
• 1M notifications/hour
• Email: delivery trong 5 phút
• SMS: delivery trong 30 giây
• Push: delivery trong 5 giây
• At-least-once delivery (không mất thông báo)
```

### Kiến Trúc

```
                                    ┌──────────────┐
Producer Services ────────────────→ │  API Gateway │
(Order, Auth, etc.)                 └──────┬───────┘
                                           │
                                    ┌──────▼───────┐
                                    │ Notification │
                                    │   Service    │ ← Template Engine
                                    └──────┬───────┘
                                           │
                              ┌────────────┼────────────┐
                              │            │            │
                        ┌─────▼─┐   ┌─────▼─┐   ┌─────▼─┐
                        │ Email │   │  SMS  │   │ Push  │
                        │ Queue │   │ Queue │   │ Queue │
                        └─────┬─┘   └─────┬─┘   └─────┬─┘
                              │            │            │
                        ┌─────▼─┐   ┌─────▼─┐   ┌─────▼─┐
                        │SendGrid│  │Twilio │   │ FCM/  │
                        │/SES   │   │       │   │ APNs  │
                        └───────┘   └───────┘   └───────┘
```

### Notification Service Implementation

```csharp
// Domain Model
public class NotificationRequest
{
    public string UserId { get; set; } = "";
    public NotificationType Type { get; set; }
    public string TemplateId { get; set; } = "";
    public Dictionary<string, string> Variables { get; set; } = new();
    public DateTime? ScheduledAt { get; set; }
    public NotificationChannel[] Channels { get; set; } = [];
}

public enum NotificationChannel { Email, Sms, Push, InApp }

// Template Service
public class TemplateService
{
    public string Render(string template, Dictionary<string, string> variables)
    {
        // Thay thế {{variable}} bằng giá trị thực
        foreach (var (key, value) in variables)
            template = template.Replace($"{{{{{key}}}}}", value);
        return template;
    }
}

// Channel Router
public class NotificationDispatcher
{
    private readonly Dictionary<NotificationChannel, INotificationSender> _senders;
    
    public NotificationDispatcher(IEnumerable<INotificationSender> senders)
    {
        _senders = senders.ToDictionary(s => s.Channel);
    }
    
    public async Task DispatchAsync(NotificationMessage message)
    {
        var tasks = message.Channels
            .Where(ch => _senders.ContainsKey(ch))
            .Select(ch => SendWithRetryAsync(_senders[ch], message));
        
        await Task.WhenAll(tasks);
    }
    
    private async Task SendWithRetryAsync(INotificationSender sender, 
        NotificationMessage message, int maxRetries = 3)
    {
        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                await sender.SendAsync(message);
                return;
            }
            catch (Exception ex) when (attempt < maxRetries)
            {
                // Exponential backoff — chờ lũy thừa
                var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt));
                await Task.Delay(delay);
            }
        }
    }
}

// MassTransit Consumer — xử lý từ queue
public class SendEmailConsumer : IConsumer<SendEmailMessage>
{
    private readonly IEmailSender _emailSender;
    private readonly ILogger _logger;
    
    public async Task Consume(ConsumeContext<SendEmailMessage> context)
    {
        var msg = context.Message;
        
        try
        {
            await _emailSender.SendAsync(new EmailMessage
            {
                To = msg.ToEmail,
                Subject = msg.Subject,
                HtmlBody = msg.HtmlBody
            });
            
            _logger.LogInformation("Email sent to {Email}", msg.ToEmail);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send email to {Email}", msg.ToEmail);
            throw; // MassTransit sẽ retry theo cấu hình
        }
    }
}
```

---

## Kịch Bản 5 — Thiết Kế Distributed Cache Layer

### Yêu Cầu

```
• Cache data từ nhiều services: user profile, product, config
• Cache invalidation khi data thay đổi
• Cache stampede prevention — tránh thundering herd
• Multi-region caching
• Memory limit: 16GB Redis cluster
```

### Cache Strategies — Chiến Lược Cache

```
1. Cache-Aside (Lazy Loading) — Tải Lười:
   Read: Check cache → miss → read DB → populate cache → return
   Write: Write DB → invalidate cache (hoặc update cache)
   Phù hợp: Read-heavy, data thay đổi không thường xuyên

2. Write-Through — Ghi Xuyên:
   Write: Write cache → write DB (đồng bộ)
   Read: Luôn có trong cache
   Phù hợp: Write không nhiều, cần consistency cao

3. Write-Behind (Write-Back) — Ghi Trễ:
   Write: Write cache → async write DB
   Phù hợp: Write-heavy, có thể chịu mất data nếu crash

4. Read-Through — Đọc Xuyên:
   Cache tự động load từ DB khi miss
   App chỉ giao tiếp với cache
```

### Cache Stampede Prevention — Chống Thundering Herd

```csharp
// Thundering Herd: 1000 requests cùng lúc hit cache miss → 1000 DB queries!

public class CacheService
{
    private readonly IDistributedCache _cache;
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    // Approach 1: Mutex Lock (đơn giản, single instance)
    public async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan ttl)
    {
        // Check cache mà không lock
        var cached = await _cache.GetStringAsync(key);
        if (cached != null)
            return JsonSerializer.Deserialize<T>(cached);
        
        // Chỉ cho 1 thread vào để populate cache
        await _semaphore.WaitAsync();
        try
        {
            // Double-check sau khi vào lock
            cached = await _cache.GetStringAsync(key);
            if (cached != null)
                return JsonSerializer.Deserialize<T>(cached);
            
            var value = await factory();
            
            await _cache.SetStringAsync(key, 
                JsonSerializer.Serialize(value),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = ttl
                });
            
            return value;
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    // Approach 2: Probabilistic Early Expiration (phân tán tốt hơn)
    // Khi cache gần hết hạn, một số requests bắt đầu refresh sớm
    // Tránh tất cả expire cùng lúc
    public async Task<T?> GetWithEarlyRefreshAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan ttl,
        double beta = 1.0) // Higher beta = refresh sớm hơn
    {
        var (value, remainingTtl, delta) = await GetWithMetadataAsync<T>(key);
        
        if (value != null)
        {
            // Xác suất refresh = exp(-delta * beta * remainingTtl)
            var shouldRefreshEarly = -delta * beta * Math.Log(Random.Shared.NextDouble()) 
                                     >= remainingTtl.TotalSeconds;
            
            if (!shouldRefreshEarly)
                return value;
        }
        
        // Refresh cache
        var freshValue = await factory();
        await SetWithMetadataAsync(key, freshValue, ttl);
        return freshValue;
    }
}

// Cache Invalidation — Xóa Cache Khi Data Thay Đổi
public class ProductService
{
    public async Task UpdateProductAsync(int id, UpdateProductDto dto)
    {
        // 1. Update DB
        var product = await _db.Products.FindAsync(id);
        product!.Name = dto.Name;
        product.Price = dto.Price;
        await _db.SaveChangesAsync();
        
        // 2. Invalidate cache
        await _cache.RemoveAsync($"product:{id}");
        
        // 3. Publish event để các service khác invalidate cache của họ
        await _eventBus.PublishAsync(new ProductUpdatedEvent(id));
    }
}
```

### Redis Cluster cho Multi-Region

```
Region: Asia (Primary Write)
  Redis Cluster: 3 master + 3 replica
  Write → Master Asia

Region: Europe (Read Replica)
  Redis Sentinel: replica của Asia cluster
  Read → Local replica (low latency)
  Replication lag: < 100ms

Vấn đề Eventual Consistency — Nhất Quán Cuối Cùng:
  • Trong 100ms, EU đọc data cũ
  • Chấp nhận được cho: product info, catalog
  • Không chấp nhận: user balance, inventory count
    → Những thứ này cần read từ primary hoặc dùng strong consistency
```

---

## 🎯 Checklist System Design Interview

### Trước Khi Bắt Đầu

- [ ] Hỏi scale requirements (users, RPS, data size)
- [ ] Hỏi read vs write ratio
- [ ] Hỏi consistency vs availability preference
- [ ] Hỏi latency requirements
- [ ] Xác nhận trong phạm vi 45 phút sẽ design phần nào

### Khi Đang Làm

- [ ] Luôn giải thích lý do chọn technology
- [ ] Đề cập trade-off khi chọn approach
- [ ] Vẽ sơ đồ rõ ràng với các thành phần và connection
- [ ] Hỏi interviewer muốn deep dive vào phần nào

### Trade-off Quan Trọng Hay Được Hỏi

```
SQL vs NoSQL:
  SQL: ACID transactions, complex queries, known schema
  NoSQL: flexible schema, horizontal scale, eventual consistency

Cache vs Database:
  Cache: fast reads, bounded memory, risk of stale data
  Database: persistent, consistent, slower

Synchronous vs Asynchronous:
  Sync: simple, immediate feedback, coupled
  Async: decoupled, resilient, eventual consistency, complex

Monolith vs Microservices:
  Monolith: simple, fast dev, single deploy
  Microservices: independent scale, complex ops, network overhead
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
