# Thread Safety — An Toàn Luồng

> Thread Safety — An Toàn Luồng — là khả năng code chạy đúng khi nhiều thread cùng truy cập dữ liệu chia sẻ (shared data). Không có thread safety, có thể xảy ra race condition — điều kiện đua — dẫn đến kết quả không thể đoán trước, data corruption, và crash.

---

## 📋 Tổng Quan Các Primitive

| Primitive | Dùng Cho | Async-Safe | Chi Phí |
| --------- | -------- | ---------- | ------- |
| `lock` | Bảo vệ đoạn code ngắn, critical section | ❌ | Thấp |
| `Monitor` | Giống lock nhưng có TryEnter, Pulse | ❌ | Thấp |
| `Mutex` | Lock xuyên process (cross-process) | ❌ | Cao |
| `SemaphoreSlim` | Giới hạn số thread, có async | ✅ | Trung bình |
| `ReaderWriterLockSlim` | Nhiều đọc, ít ghi | ❌ | Trung bình |
| `Interlocked` | Atomic operation cho số nguyên | ✅ | Rất thấp |
| `volatile` | Visibility của biến giữa các thread | N/A | Rất thấp |
| `Concurrent collections` | Thread-safe collections sẵn | ✅ | Thấp |
| Immutable objects | Không cần lock — không thay đổi được | ✅ | N/A |

---

## 🔷 Race Condition — Điều Kiện Đua Là Gì?

```csharp
// ❌ VÍ DỤ RACE CONDITION ĐIỂN HÌNH:
public class BankAccount
{
    private decimal _balance = 1000;

    // Không thread-safe!
    public void Withdraw(decimal amount)
    {
        if (_balance >= amount)          // Thread A kiểm tra: balance = 1000 ✓
        {
            // Thread B chen vào ở đây! Cũng kiểm tra: balance = 1000 ✓
            _balance -= amount;          // Thread A: balance = 500
            // Thread B: balance = -500 ← CATASTROPHIC! Âm tài khoản!
        }
    }
}

// Chạy 2 thread cùng lúc:
var account = new BankAccount();
var t1 = Task.Run(() => account.Withdraw(800));
var t2 = Task.Run(() => account.Withdraw(800));
await Task.WhenAll(t1, t2);
// Kết quả: balance có thể là -600 dù chỉ có 1000 ban đầu!
```

---

## 🔷 lock — Khóa Độc Quyền

### Cú Pháp lock

```csharp
// lock đảm bảo chỉ một thread vào critical section — vùng tới hạn — cùng lúc
public class ThreadSafeBankAccount
{
    private decimal _balance = 1000;
    private readonly object _lock = new();  // Object dùng làm khóa

    public void Withdraw(decimal amount)
    {
        lock (_lock)  // Chỉ một thread được vào block này cùng lúc
        {
            if (_balance >= amount)
                _balance -= amount;
            else
                throw new InvalidOperationException("Số dư không đủ");
        }  // Tự động release lock khi ra khỏi block
    }

    public decimal Balance
    {
        get { lock (_lock) { return _balance; } }
    }
}
```

### Quy Tắc Khi Dùng lock

```csharp
// ✅ ĐÚNG: Dùng object private readonly làm lock object
private readonly object _lock = new();

// ❌ SAI: Lock trên this — người ngoài có thể lock cùng object
lock (this) { ... }

// ❌ SAI: Lock trên string literal — interned strings được chia sẻ toàn app
lock ("shared-string") { ... }

// ❌ SAI: Lock trên Type — có thể gây deadlock với code khác
lock (typeof(MyClass)) { ... }

// ❌ SAI: Lock trên value type (sẽ bị boxing, mỗi lần một object mới)
private int _lockObj = 0;
lock (_lockObj) { ... }  // Compiler warning — boxing mỗi lần → không lock được
```

### Giữ Lock Ngắn Nhất Có Thể

```csharp
// ❌ SAI: Giữ lock trong lúc gọi I/O
public async Task ProcessOrderAsync(Order order)
{
    lock (_lock)
    {
        var existing = _orders[order.Id];
        await _emailService.SendAsync(order);  // Compilation ERROR: await trong lock
        _orders[order.Id] = order;
    }
}

// ✅ ĐÚNG: Lấy dữ liệu cần với lock, làm I/O bên ngoài
public async Task ProcessOrderAsync(Order order)
{
    Order? existing;
    lock (_lock)
    {
        existing = _orders.GetValueOrDefault(order.Id);  // Nhanh, không I/O
    }

    await _emailService.SendAsync(order);  // I/O bên ngoài lock

    lock (_lock)
    {
        _orders[order.Id] = order;  // Update ngắn gọn
    }
}
```

---

## 🔷 Monitor — Kiểm Soát Chi Tiết Hơn lock

```csharp
// Monitor là nền tảng của lock — lock chỉ là syntactic sugar
// Dùng Monitor khi cần TryEnter (không muốn block mãi)

public bool TryUpdateData(Data data, TimeSpan timeout)
{
    bool acquired = false;
    try
    {
        Monitor.TryEnter(_lock, timeout, ref acquired);
        if (!acquired)
            return false;  // Không thể lấy lock trong timeout

        _data = data;
        return true;
    }
    finally
    {
        if (acquired)
            Monitor.Exit(_lock);
    }
}

// Monitor.Wait / Pulse — Báo hiệu giữa các thread (producer-consumer)
public class BlockingQueue<T>
{
    private readonly Queue<T> _queue = new();
    private readonly object _lock = new();

    public void Enqueue(T item)
    {
        lock (_lock)
        {
            _queue.Enqueue(item);
            Monitor.Pulse(_lock);  // Đánh thức một thread đang Wait
        }
    }

    public T Dequeue()
    {
        lock (_lock)
        {
            while (_queue.Count == 0)
                Monitor.Wait(_lock);  // Nhả lock, ngủ, chờ Pulse

            return _queue.Dequeue();
        }
    }
}
```

---

## 🔷 SemaphoreSlim — Giới Hạn Đồng Thời (Async-Safe)

```csharp
// SemaphoreSlim: cho phép tối đa N thread/task vào cùng lúc
// Có WaitAsync() — an toàn cho async context (khác Semaphore cũ)

public class RateLimitedApiClient
{
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(5, 5);  // Tối đa 5

    public async Task<string> CallApiAsync(string url, CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);  // Chờ slot trống (async — không block thread)
        try
        {
            return await _http.GetStringAsync(url, ct);
        }
        finally
        {
            _semaphore.Release();  // Trả slot — LUÔN trong finally
        }
    }
}

// ✅ Dùng SemaphoreSlim(1,1) thay lock khi cần async:
private readonly SemaphoreSlim _asyncLock = new SemaphoreSlim(1, 1);

public async Task UpdateAsync(Data data)
{
    await _asyncLock.WaitAsync();
    try
    {
        await _repository.SaveAsync(data);  // await trong "lock" được!
    }
    finally
    {
        _asyncLock.Release();
    }
}
```

---

## 🔷 ReaderWriterLockSlim — Tối Ưu Cho Read-Heavy Workload

```csharp
// Cho phép nhiều reader (đọc) cùng lúc, nhưng chỉ một writer (ghi)
// Tốt khi: đọc nhiều, ghi ít — ví dụ: config cache, in-memory dictionary

public class ConfigurationCache
{
    private readonly Dictionary<string, string> _cache = new();
    private readonly ReaderWriterLockSlim _rwLock = new();

    public string? Get(string key)
    {
        _rwLock.EnterReadLock();  // Nhiều reader vào cùng lúc được
        try
        {
            return _cache.GetValueOrDefault(key);
        }
        finally
        {
            _rwLock.ExitReadLock();
        }
    }

    public void Set(string key, string value)
    {
        _rwLock.EnterWriteLock();  // Chỉ một writer, block tất cả reader
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _rwLock.ExitWriteLock();
        }
    }

    // Không quên Dispose!
    public void Dispose() => _rwLock.Dispose();
}
```

---

## 🔷 Interlocked — Atomic Operations — Thao Tác Nguyên Tử

```csharp
// Interlocked: atomic — không thể bị ngắt giữa chừng bởi thread khác
// Rất nhanh, không cần lock cho các thao tác đơn giản

public class RequestCounter
{
    private long _totalRequests = 0;
    private long _failedRequests = 0;

    public void RecordRequest(bool success)
    {
        Interlocked.Increment(ref _totalRequests);  // Thread-safe ++
        if (!success)
            Interlocked.Increment(ref _failedRequests);
    }

    public void Reset()
    {
        Interlocked.Exchange(ref _totalRequests, 0);   // Atomic set
        Interlocked.Exchange(ref _failedRequests, 0);
    }

    // CompareExchange — Compare-And-Swap (CAS) — So Sánh Và Hoán Đổi
    public bool TrySetStatus(int expectedStatus, int newStatus)
    {
        // Chỉ set newStatus nếu giá trị hiện tại == expectedStatus
        // Trả về giá trị CŨ (trước khi exchange)
        int original = Interlocked.CompareExchange(
            ref _status, newStatus, expectedStatus);
        return original == expectedStatus;
    }

    private int _status = 0;
    public long TotalRequests => Interlocked.Read(ref _totalRequests);
}
```

### Lazy Initialization Với Interlocked

```csharp
// Double-checked locking pattern với Interlocked
public class Singleton
{
    private static volatile Singleton? _instance;
    private static readonly object _lock = new();

    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                        _instance = new Singleton();
                }
            }
            return _instance;
        }
    }
}

// ✅ Cách đơn giản hơn: Lazy<T> — thread-safe by default
private static readonly Lazy<Singleton> _lazy = new(() => new Singleton());
public static Singleton Instance => _lazy.Value;
```

---

## 🔷 volatile — Visibility Giữa Các Thread

```csharp
// volatile: đảm bảo thread luôn đọc giá trị mới nhất từ bộ nhớ chính
// Không dùng CPU cache — tránh instruction reordering

public class StopFlag
{
    private volatile bool _shouldStop = false;

    public void Stop() => _shouldStop = true;  // Các thread khác thấy ngay

    public async Task RunLoopAsync()
    {
        while (!_shouldStop)  // Luôn đọc giá trị thực từ memory
        {
            await DoWorkAsync();
        }
    }
}

// volatile KHÔNG đảm bảo atomicity — chỉ đảm bảo visibility
// Không dùng volatile cho compound operations (read-modify-write)
// Dùng Interlocked cho compound operations
```

---

## 🔷 Thread-Safe Collections — Collections An Toàn Luồng

```csharp
// System.Collections.Concurrent cung cấp collections thread-safe
using System.Collections.Concurrent;

// ConcurrentDictionary — Dictionary thread-safe
var dict = new ConcurrentDictionary<string, int>();

// Thread-safe operations:
dict.TryAdd("key", 1);
dict.TryGetValue("key", out var value);
dict.TryRemove("key", out _);
dict.AddOrUpdate("key", 1, (k, old) => old + 1);  // Atomic read-modify-write
dict.GetOrAdd("key", k => ExpensiveCompute(k));    // Chỉ tính nếu chưa có

// ConcurrentQueue — Queue thread-safe (FIFO — First In, First Out)
var queue = new ConcurrentQueue<Task>();
queue.Enqueue(myTask);
if (queue.TryDequeue(out var task))
    await task;

// ConcurrentBag — Collection không có thứ tự, thread-safe
var bag = new ConcurrentBag<int>();
bag.Add(42);
bag.TryTake(out var item);

// BlockingCollection<T> — Queue với chặn (blocking) — producer-consumer
var blocking = new BlockingCollection<WorkItem>(boundedCapacity: 100);
// Producer:
blocking.Add(new WorkItem());          // Block nếu đầy (100 items)
blocking.CompleteAdding();             // Báo hiệu không thêm nữa
// Consumer:
foreach (var item in blocking.GetConsumingEnumerable())
    Process(item);  // Tự block khi rỗng, thoát khi CompleteAdding được gọi
```

---

## 🔷 Immutability — Bất Biến: Giải Pháp Tốt Nhất

```csharp
// Immutable objects không cần lock — không thể thay đổi sau khi tạo
// Thread-safe by design — an toàn theo thiết kế

// record struct (C# 10+) — bất biến, value type
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(decimal amount) => this with { Amount = Amount + amount };
    // Không thay đổi object gốc — tạo object mới
}

// ✅ Immutable class với init-only properties
public sealed class UserSettings
{
    public string Theme { get; init; }       // Chỉ set khi khởi tạo
    public int PageSize { get; init; }
    public bool DarkMode { get; init; }

    public UserSettings(string theme, int pageSize, bool darkMode)
    {
        Theme = theme;
        PageSize = pageSize;
        DarkMode = darkMode;
    }

    // "Thay đổi" bằng cách tạo object mới (with expression)
    public UserSettings WithTheme(string newTheme)
        => new(newTheme, PageSize, DarkMode);
}

// ✅ ImmutableList, ImmutableDictionary (System.Collections.Immutable)
using System.Collections.Immutable;

var list = ImmutableList.Create(1, 2, 3);
var newList = list.Add(4);  // Không thay đổi list gốc, trả về list mới
// list vẫn là [1, 2, 3] — an toàn để chia sẻ giữa các thread
```

---

## 🚫 Anti-Patterns Phổ Biến

### 1. Lock Granularity Quá Lớn

```csharp
// ❌ SAI: Một lock cho tất cả — bottleneck
private readonly object _globalLock = new();

public void UpdateUser(User user) { lock (_globalLock) { ... } }
public void UpdateOrder(Order order) { lock (_globalLock) { ... } }  // Không liên quan nhau!
public void UpdateProduct(Product p) { lock (_globalLock) { ... } }  // Nhưng phải chờ nhau

// ✅ ĐÚNG: Lock riêng cho từng resource
private readonly object _userLock = new();
private readonly object _orderLock = new();
private readonly object _productLock = new();
```

### 2. Double-Checked Locking Sai Cách

```csharp
// ❌ SAI: Không có volatile → compiler có thể reorder
private static Singleton _instance;
public static Singleton Instance
{
    get
    {
        if (_instance == null)           // Check 1: có thể thấy giá trị cũ
        {
            lock (_lock)
            {
                if (_instance == null)
                    _instance = new Singleton();  // Có thể bị reorder!
            }
        }
        return _instance;
    }
}

// ✅ ĐÚNG: volatile hoặc dùng Lazy<T>
private static volatile Singleton _instance;  // Thêm volatile
// HOẶC:
private static readonly Lazy<Singleton> _lazy = new(() => new Singleton());
```

### 3. Await Trong lock

```csharp
// ❌ KHÔNG COMPILE: lock không hỗ trợ await
lock (_lock)
{
    await SomeAsyncMethod();  // Compiler error!
}

// ✅ ĐÚNG: SemaphoreSlim(1,1) thay thế lock cho async
await _asyncLock.WaitAsync();
try
{
    await SomeAsyncMethod();
}
finally
{
    _asyncLock.Release();
}
```

---

## 📊 Quyết Định Nhanh: Dùng Gì?

```
Tình Huống                                 Giải Pháp Phù Hợp
───────────────────────────────────────────────────────────────────
Đếm request (increment/decrement)          Interlocked
Bảo vệ code đồng bộ ngắn                  lock
Bảo vệ code async                          SemaphoreSlim(1,1)
Nhiều reader, ít writer                    ReaderWriterLockSlim
Giới hạn N task đồng thời                 SemaphoreSlim(N,N)
Queue giữa producer và consumer            Channel<T> hoặc ConcurrentQueue
Dictionary với nhiều thread                ConcurrentDictionary
Không muốn lock gì cả                      Immutable objects + functional style
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Race condition là gì? Cho ví dụ?**
> Race condition xảy ra khi kết quả phụ thuộc vào thứ tự thực hiện của nhiều thread không xác định. Ví dụ: hai thread cùng đọc `balance = 1000`, cùng trừ 800, cùng ghi lại → balance thành 200 thay vì -600 (hoặc exception). Xử lý bằng `lock`, `Interlocked`, hoặc immutable objects.

**Q: Sự khác nhau giữa lock và SemaphoreSlim?**
> `lock` chỉ cho một thread vào, không hỗ trợ `await`. `SemaphoreSlim` cho N thread vào (N cấu hình được), có `WaitAsync()` cho async code. Dùng `SemaphoreSlim(1,1)` như `lock` nhưng async-friendly.

**Q: volatile giải quyết vấn đề gì?**
> `volatile` đảm bảo thread luôn đọc giá trị từ bộ nhớ chính, không từ CPU cache. Giải quyết visibility problem — vấn đề hiển thị. KHÔNG giải quyết atomicity — nguyên tử tính. Không dùng `volatile` cho compound operations (read-modify-write) — dùng `Interlocked` thay.

**Q: Khi nào dùng ConcurrentDictionary vs Dictionary + lock?**
> `ConcurrentDictionary`: khi nhiều thread read/write thường xuyên — tối ưu cho concurrent access. `Dictionary + lock`: khi cần atomic operation trên nhiều keys cùng lúc (không thể làm với ConcurrentDictionary), hoặc khi đọc nhiều hơn ghi (kết hợp với ReaderWriterLockSlim tốt hơn).

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
