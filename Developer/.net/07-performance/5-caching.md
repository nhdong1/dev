# Caching — Bộ Nhớ Đệm trong .NET

> Caching — Bộ Nhớ Đệm: lưu kết quả tính toán tốn kém để dùng lại, tránh tính lại mỗi lần.

---

## 1. Khi Nào Nên Cache?

```
Cache có lợi khi:
✅ Dữ liệu đọc nhiều hơn ghi (read-heavy)
✅ Tính toán tốn kém (query DB, gọi API ngoài)
✅ Dữ liệu không thay đổi thường xuyên
✅ Chấp nhận được dữ liệu đôi khi stale — cũ một chút

Cache KHÔNG nên dùng khi:
❌ Dữ liệu thay đổi realtime (giá chứng khoán tick-by-tick)
❌ Dữ liệu mang tính cá nhân cao (không muốn dùng chung)
❌ Kết quả phải luôn 100% mới nhất (tài khoản ngân hàng)
❌ Dữ liệu quá lớn và ít được truy cập
```

---

## 2. IMemoryCache — Cache Trong Bộ Nhớ Tiến Trình

`IMemoryCache` lưu dữ liệu trong RAM của tiến trình hiện tại — đơn giản, nhanh, không cần infrastructure.

### Đăng Ký và Dùng Cơ Bản

```csharp
// Program.cs
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;        // tổng "size units" tối đa
    options.CompactionPercentage = 0.25; // compact 25% khi đầy
});

// Inject và dùng
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;
    
    public ProductService(IMemoryCache cache, IProductRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }
    
    public async Task<Product?> GetProductAsync(int id)
    {
        string cacheKey = $"product:{id}";
        
        // GetOrCreateAsync — pattern phổ biến nhất
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            // Cấu hình entry options
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Size = 1; // phải khai báo khi cache có SizeLimit
            entry.Priority = CacheItemPriority.Normal;
            
            // Factory: chỉ gọi khi cache miss
            return await _repo.GetByIdAsync(id);
        });
    }
}
```

### Expiration — Hết Hạn Cache

```csharp
var options = new MemoryCacheEntryOptions();

// AbsoluteExpiration: hết hạn vào thời điểm cố định
options.AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1);

// AbsoluteExpirationRelativeToNow: hết hạn sau khoảng thời gian
options.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);

// SlidingExpiration: hết hạn nếu KHÔNG được truy cập trong khoảng thời gian
// Mỗi lần truy cập → reset countdown
options.SlidingExpiration = TimeSpan.FromMinutes(2);

// Kết hợp: absolute + sliding (sliding không thể vượt absolute)
options.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10); // max 10 phút
options.SlidingExpiration = TimeSpan.FromMinutes(2); // hết hạn nếu idle 2 phút

_cache.Set(key, value, options);
```

### Cache Invalidation — Xóa Cache

```csharp
// Xóa một entry
_cache.Remove("product:42");

// Xóa nhóm entry với CancellationTokenSource
private readonly CancellationTokenSource _categoryCts = new();

// Set với token
var options = new MemoryCacheEntryOptions()
    .AddExpirationToken(new CancellationChangeToken(_categoryCts.Token));
_cache.Set("category:electronics", categoryData, options);

// Invalidate toàn bộ group
_categoryCts.Cancel();
// Tạo token mới cho lần sau
// (cần cơ chế quản lý token phức tạp hơn cho production)
```

### PostEviction Callback — Hành Động Khi Xóa

```csharp
var options = new MemoryCacheEntryOptions();
options.RegisterPostEvictionCallback((key, value, reason, state) =>
{
    Console.WriteLine($"Cache evicted: {key}, reason: {reason}");
    
    // EvictionReason values:
    // None, Removed (manual), Replaced, Expired, Capacity, TokenExpired
});

_cache.Set("key", value, options);
```

---

## 3. IDistributedCache — Cache Phân Tán

`IDistributedCache` cho phép share cache giữa nhiều instance (scale-out). Lưu byte array, không phải object.

### Redis — Triển Khai Phổ Biến Nhất

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    // Redis connection string: "localhost:6379" hoặc "redis-server:6379,password=xxx"
    options.InstanceName = "MyApp:"; // prefix cho tất cả keys
});

// Dùng IDistributedCache
public class UserService
{
    private readonly IDistributedCache _cache;
    private readonly IUserRepository _repo;
    
    public UserService(IDistributedCache cache, IUserRepository repo)
    {
        _cache = cache;
        _repo = repo;
    }
    
    public async Task<UserDto?> GetUserAsync(int userId, CancellationToken ct = default)
    {
        string key = $"user:{userId}";
        
        // GetAsync trả về byte[]?
        byte[]? cachedBytes = await _cache.GetAsync(key, ct);
        
        if (cachedBytes is not null)
        {
            // Deserialize từ bytes
            return JsonSerializer.Deserialize<UserDto>(cachedBytes);
        }
        
        // Cache miss: lấy từ DB
        var user = await _repo.GetByIdAsync(userId, ct);
        if (user is null) return null;
        
        // Serialize và cache
        byte[] serialized = JsonSerializer.SerializeToUtf8Bytes(user);
        await _cache.SetAsync(key, serialized, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
            SlidingExpiration = TimeSpan.FromMinutes(5)
        }, ct);
        
        return user;
    }
    
    public async Task InvalidateUserAsync(int userId, CancellationToken ct = default)
    {
        await _cache.RemoveAsync($"user:{userId}", ct);
    }
}
```

### Helper Extension Cho IDistributedCache

```csharp
// Extension để dùng như IMemoryCache (với GetOrCreate)
public static class DistributedCacheExtensions
{
    public static async Task<T?> GetOrCreateAsync<T>(
        this IDistributedCache cache,
        string key,
        Func<Task<T?>> factory,
        DistributedCacheEntryOptions? options = null,
        CancellationToken ct = default)
    {
        byte[]? cached = await cache.GetAsync(key, ct);
        
        if (cached is not null)
            return JsonSerializer.Deserialize<T>(cached);
        
        T? value = await factory();
        if (value is null) return default;
        
        byte[] bytes = JsonSerializer.SerializeToUtf8Bytes(value);
        await cache.SetAsync(key, bytes, options ?? DefaultOptions(), ct);
        
        return value;
    }
    
    private static DistributedCacheEntryOptions DefaultOptions() => new()
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    };
}

// Sử dụng
var user = await _cache.GetOrCreateAsync($"user:{id}",
    () => _repo.GetByIdAsync(id));
```

---

## 4. Cache-Aside Pattern — Mẫu Cache Bên Cạnh

Cache-Aside là pattern phổ biến nhất — application quản lý cache thủ công:

```
Cache-Aside Flow:
─────────────────────────────────────────────────────────────────
    App                    Cache                   Database
     │                       │                         │
     │──── Get(key) ────────►│                         │
     │◄─── null (miss) ──────│                         │
     │                       │                         │
     │──────────────────────────────── Query ─────────►│
     │◄──────────────────────────────── Data ──────────│
     │                       │                         │
     │──── Set(key, data) ──►│                         │
     │                       │                         │
     │  (Next request)       │                         │
     │──── Get(key) ────────►│                         │
     │◄─── Data (hit!) ──────│ ← không cần vào DB     │
```

```csharp
public async Task<Product?> GetProductCacheAsideAsync(int id)
{
    string key = $"product:{id}";
    
    // Step 1: Check cache
    if (_cache.TryGetValue(key, out Product? cached))
        return cached;
    
    // Step 2: Cache miss → query database
    var product = await _db.Products
        .AsNoTracking()
        .FirstOrDefaultAsync(p => p.Id == id);
    
    // Step 3: Set cache (kể cả null để tránh cache stampede một phần)
    var options = new MemoryCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = product is not null
            ? TimeSpan.FromMinutes(10)
            : TimeSpan.FromMinutes(1), // null cache ngắn hơn
        Size = 1
    };
    _cache.Set(key, product, options);
    
    return product;
}
```

---

## 5. Cache Stampede Prevention — Phòng Tránh Bão Cache

**Cache Stampede** (hay Thundering Herd — Bầy Sấm) xảy ra khi nhiều request đồng thời thấy cache miss và cùng lúc query DB:

```
Vấn đề:
────────────────────────────────────────────────────────
t=0: Cache expires cho "popular:product:1"
t=1ms: Request 1 → cache miss → query DB
t=2ms: Request 2 → cache miss → query DB   ← 100 request cùng lúc!
t=3ms: Request 3 → cache miss → query DB
...
→ 100 queries đến DB cùng lúc cho 1 key!

Giải pháp 1: SemaphoreSlim per key
```

```csharp
// Dùng SemaphoreSlim để chỉ 1 request query DB cùng lúc
public class StampedePreventingCache
{
    private readonly IMemoryCache _cache;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();
    
    public async Task<T?> GetOrCreateAsync<T>(
        string key,
        Func<Task<T?>> factory,
        TimeSpan duration)
    {
        // Fast path: cache hit (không cần lock)
        if (_cache.TryGetValue(key, out T? cached))
            return cached;
        
        // Slow path: cache miss → lấy lock
        var semaphore = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        
        await semaphore.WaitAsync();
        try
        {
            // Double-check sau khi có lock
            if (_cache.TryGetValue(key, out cached))
                return cached;
            
            // Chỉ 1 request chạy factory
            T? value = await factory();
            _cache.Set(key, value, duration);
            return value;
        }
        finally
        {
            semaphore.Release();
            _locks.TryRemove(key, out _); // dọn semaphore
        }
    }
}
```

---

## 6. Response Caching — Cache HTTP Response

```csharp
// Program.cs
builder.Services.AddResponseCaching();
// ...
app.UseResponseCaching(); // phải trước UseRouting

// Controller/Action
[HttpGet]
[ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "id" })]
public IActionResult GetProduct(int id)
{
    var product = _service.GetProduct(id);
    return Ok(product);
}

// Cache-Control headers tự động được thêm:
// Cache-Control: public, max-age=60
```

---

## 7. Output Caching — .NET 7+ (Hiện Đại Hơn)

```csharp
// Program.cs (.NET 7+)
builder.Services.AddOutputCache(options =>
{
    // Define cache policies
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(10)));
    
    options.AddPolicy("Products", b => b
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("products")
        .VaryByRouteValue("id"));
});

app.UseOutputCache();

// Endpoint
app.MapGet("/products/{id}", async (int id, IProductService svc) =>
    await svc.GetByIdAsync(id))
    .CacheOutput("Products");

// Invalidate by tag (khi data thay đổi)
app.MapPost("/products/{id}", async (
    int id,
    ProductDto dto,
    IOutputCacheStore cache,
    CancellationToken ct) =>
{
    await _service.UpdateAsync(id, dto);
    await cache.EvictByTagAsync("products", ct); // invalidate tất cả product cache
    return Results.Ok();
});
```

---

## 8. Hybrid Cache — .NET 9 (.NET 9+)

HybridCache kết hợp IMemoryCache (L1) và IDistributedCache (L2):

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

```csharp
// Program.cs
builder.Services.AddHybridCache(options =>
{
    options.MaximumPayloadBytes = 1024 * 1024; // 1MB
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),
        LocalCacheExpiration = TimeSpan.FromMinutes(1)
    };
});

// Sử dụng
public class ProductService
{
    private readonly HybridCache _cache;
    
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel => await _repo.GetByIdAsync(id, cancel),
            cancellationToken: ct
        );
    }
    
    public async Task InvalidateAsync(int id)
    {
        await _cache.RemoveAsync($"product:{id}");
    }
}
```

---

## 9. Chiến Lược Cache Key — Cache Key Design

```csharp
// ✅ Key rõ ràng, có namespace
string key = $"product:{id}";                    // entity:id
string key = $"user:{userId}:orders:{page}";     // entity:id:subresource:param
string key = $"search:category:{cat}:page:{p}";  // operation:params

// ✅ Versioning: khi schema thay đổi
string key = $"v2:product:{id}"; // thêm version prefix

// ❌ Key quá chung
string key = "products"; // nhiều request dùng chung, khó invalidate
string key = id.ToString(); // không có context, dễ collision

// ❌ Key quá cụ thể
string key = $"product:{id}:{DateTime.Now:HHmmss}"; // không bao giờ hit!
```

---

## 10. Cache Monitoring — Theo Dõi Cache

```csharp
// Theo dõi cache hit rate
public class CachedProductService
{
    private long _hits;
    private long _misses;
    
    public double HitRate => _hits + _misses == 0 ? 0 
        : (double)_hits / (_hits + _misses);
    
    public async Task<Product?> GetProductAsync(int id)
    {
        string key = $"product:{id}";
        
        if (_cache.TryGetValue(key, out Product? product))
        {
            Interlocked.Increment(ref _hits);
            return product;
        }
        
        Interlocked.Increment(ref _misses);
        product = await _repo.GetByIdAsync(id);
        _cache.Set(key, product, TimeSpan.FromMinutes(10));
        return product;
    }
}

// Metrics với OpenTelemetry / Prometheus
_meterProvider.CreateCounter<long>("cache.hits")
    .Add(1, new TagList { ["cache_name"] = "products" });
```

---

## 11. So Sánh Các Tùy Chọn Cache

| Tùy Chọn | Phạm Vi | Phù Hợp | Nhược Điểm |
|----------|---------|----------|------------|
| `IMemoryCache` | Process-local | Single instance, nhanh nhất | Mất cache khi restart, không share giữa instances |
| `IDistributedCache` (Redis) | Cross-process | Multi-instance (scale-out) | Network latency, Redis infrastructure |
| `IDistributedCache` (SQL Server) | Cross-process | Không có Redis | Chậm hơn Redis đáng kể |
| Response Caching | HTTP response | GET requests công khai | Không linh hoạt bằng code |
| Output Caching (.NET 7+) | Endpoint output | Hiện đại, dễ invalidate | Cần .NET 7+ |
| HybridCache (.NET 9+) | L1 + L2 combined | Tốt nhất cả hai | Cần .NET 9+ |

---

## 12. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác nhau giữa `IMemoryCache` và `IDistributedCache`?**

> - `IMemoryCache`: lưu trong RAM của tiến trình hiện tại. Nhanh, đơn giản, nhưng mỗi instance server có cache riêng — không phù hợp khi scale-out (nhiều server).
> - `IDistributedCache`: lưu ở storage ngoài (Redis, SQL). Chậm hơn (network roundtrip), nhưng tất cả instance dùng chung một cache — phù hợp cho cloud deployment.

**Q: Cache Stampede là gì và cách phòng tránh?**

> Cache Stampede xảy ra khi cache key hết hạn, nhiều request đồng thời thấy cache miss và cùng lúc query DB. Giải pháp: (1) SemaphoreSlim per key để chỉ 1 request query DB; (2) Probabilistic early expiration (staggered expiry); (3) Background refresh trước khi hết hạn.

**Q: Khi nào nên dùng `AbsoluteExpiration` vs `SlidingExpiration`?**

> - `AbsoluteExpiration`: khi dữ liệu phải fresh sau khoảng thời gian nhất định (dữ liệu có TTL — Time-To-Live — Thời Gian Tồn Tại rõ ràng).
> - `SlidingExpiration`: khi muốn cache "hoạt động" — thường xuyên dùng thì giữ lâu, không dùng thì tự xóa (session data, hot items).
> - Kết hợp cả hai: sliding cho inactive eviction, absolute làm ceiling.

---

## ✅ Checklist

- [ ] Luôn đặt expiration khi cache (`AbsoluteExpirationRelativeToNow`)
- [ ] Đặt `SizeLimit` và `Size` cho `IMemoryCache` để tránh phình to
- [ ] Dùng `GetOrCreateAsync` thay vì `Get`/`Set` riêng lẻ
- [ ] Xử lý cache stampede với double-check locking hoặc semaphore
- [ ] Thiết kế cache key có namespace, tránh collision
- [ ] Implement cache invalidation khi data thay đổi
- [ ] Monitor cache hit rate trong production

---

**Xem Tiếp:** [6-profiling-tools.md](6-profiling-tools.md) — dotnet-trace, dotnet-dump, PerfView, dotMemory
