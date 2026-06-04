# Task Parallel Library — TPL — Thư Viện Tác Vụ Song Song

> TPL — Task Parallel Library — Thư Viện Tác Vụ Song Song — là tập hợp API trong `System.Threading.Tasks` giúp viết code song song và bất đồng bộ đơn giản hơn. TPL quản lý ThreadPool, cân bằng tải tự động, và cung cấp các primitive cao cấp như `Task`, `Parallel`, và `PLINQ`.

---

## 📋 Tổng Quan TPL

| API | Dùng Cho | Loại |
| --- | -------- | ---- |
| `Task` / `Task<T>` | Đơn vị công việc bất đồng bộ | I/O + CPU |
| `Task.WhenAll` | Chờ nhiều task hoàn thành cùng lúc | Fan-out |
| `Task.WhenAny` | Chờ task đầu tiên hoàn thành | Racing |
| `Task.Run` | Chạy CPU-bound code trên ThreadPool | CPU-bound |
| `Parallel.For` | Vòng lặp song song | CPU-bound data |
| `Parallel.ForEach` | Duyệt collection song song | CPU-bound data |
| `PLINQ` | LINQ chạy song song | CPU-bound query |

---

## 🔷 Task — Tác Vụ Bất Đồng Bộ

### Tạo và Chạy Task

```csharp
// 1. Task từ async method (cách thường dùng nhất)
Task<User> userTask = _repository.GetUserAsync(42);
User user = await userTask;

// 2. Task.Run — đẩy CPU-bound code sang ThreadPool
Task<int> calcTask = Task.Run(() =>
{
    return HeavyCalculation(data);  // Chạy trên ThreadPool thread
});
int result = await calcTask;

// 3. Task.FromResult — Task đã hoàn thành với giá trị sẵn (không async)
Task<string> cachedTask = Task.FromResult("cached-value");

// 4. Task.CompletedTask — Task đã hoàn thành, không giá trị
Task noOpTask = Task.CompletedTask;

// 5. Task.FromException — Task thất bại với exception
Task<string> failedTask = Task.FromException<string>(
    new NotFoundException("User not found"));
```

### Continuation — Tiếp Nối Task

```csharp
// ContinueWith: thực hiện sau khi task hoàn thành
Task<string> result = _service.GetDataAsync()
    .ContinueWith(t =>
    {
        if (t.IsFaulted)
            return "Error: " + t.Exception?.Message;
        return t.Result.ToUpper();
    });

// ✅ Dùng await thay ContinueWith cho code rõ hơn:
try
{
    var data = await _service.GetDataAsync();
    return data.ToUpper();
}
catch (Exception ex)
{
    return "Error: " + ex.Message;
}
```

---

## 🔷 Task.WhenAll — Chạy Song Song

### Fan-out Pattern — Mẫu Khuếch Tán

```csharp
// ❌ CHẬM: Sequential — tuần tự
var user = await _userService.GetUserAsync(userId);       // chờ xong
var orders = await _orderService.GetOrdersAsync(userId);  // rồi mới gọi tiếp
var reviews = await _reviewService.GetReviewsAsync(userId); // lần lượt

// ✅ NHANH: Parallel — song song với Task.WhenAll
var userTask    = _userService.GetUserAsync(userId);
var ordersTask  = _orderService.GetOrdersAsync(userId);
var reviewsTask = _reviewService.GetReviewsAsync(userId);

await Task.WhenAll(userTask, ordersTask, reviewsTask);

var user    = userTask.Result;     // Lấy kết quả — không block vì đã await WhenAll
var orders  = ordersTask.Result;
var reviews = reviewsTask.Result;
```

### WhenAll Với Collection

```csharp
// Gọi API cho nhiều item song song (nhớ giới hạn concurrency!)
var userIds = new[] { 1, 2, 3, 4, 5 };

var tasks = userIds.Select(id => _service.GetUserAsync(id));
var users = await Task.WhenAll(tasks);

// ⚠️ CẢNH BÁO: Nếu có 1000 ids, tạo 1000 HTTP requests cùng lúc!
// → Dùng SemaphoreSlim để giới hạn (xem bên dưới)
```

### Exception Handling Với WhenAll

```csharp
// Task.WhenAll ném AggregateException nếu có task lỗi
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    // ex chỉ chứa exception đầu tiên!
    // Các exception khác bị nuốt
}

// ✅ ĐÚNG: Lấy tất cả exception
var tasks = new[] { task1, task2, task3 };

try
{
    await Task.WhenAll(tasks);
}
catch
{
    var exceptions = tasks
        .Where(t => t.IsFaulted)
        .Select(t => t.Exception!.InnerException!)
        .ToList();

    foreach (var ex in exceptions)
        _logger.LogError(ex, "Task failed");

    throw new AggregateException(exceptions);
}
```

---

## 🔷 Task.WhenAny — Racing Pattern

```csharp
// WhenAny: hoàn thành khi task đầu tiên xong
var primaryTask = _primaryDb.GetUserAsync(userId);
var fallbackTask = _fallbackDb.GetUserAsync(userId);

// Chờ DB nào phản hồi trước
var firstCompleted = await Task.WhenAny(primaryTask, fallbackTask);
var user = await firstCompleted;

// ✅ Pattern: Timeout với WhenAny
var dataTask = _service.GetDataAsync();
var timeoutTask = Task.Delay(TimeSpan.FromSeconds(5));

var completedTask = await Task.WhenAny(dataTask, timeoutTask);

if (completedTask == timeoutTask)
    throw new TimeoutException("Request took too long");

var data = await dataTask;  // await lại để unwrap exception nếu có
```

---

## 🔷 Throttling — Giới Hạn Concurrency Với SemaphoreSlim

### Vấn Đề Khi Không Giới Hạn

```csharp
// ❌ NGUY HIỂM: 10,000 requests cùng lúc
var urls = GetThousandsOfUrls();
var tasks = urls.Select(url => _http.GetStringAsync(url));
await Task.WhenAll(tasks);  // Tạo 10,000 request đồng thời → server OOM hoặc throttle
```

### SemaphoreSlim Để Giới Hạn

```csharp
// ✅ ĐÚNG: Tối đa 10 request đồng thời
using var semaphore = new SemaphoreSlim(10, 10);  // initialCount=10, maxCount=10

var urls = GetThousandsOfUrls();
var tasks = urls.Select(async url =>
{
    await semaphore.WaitAsync();  // Chờ khi đã có 10 request đang chạy
    try
    {
        return await _http.GetStringAsync(url);
    }
    finally
    {
        semaphore.Release();  // Luôn release trong finally
    }
});

var results = await Task.WhenAll(tasks);
```

### Helper Method — Phương Thức Trợ Giúp

```csharp
// Utility để chạy collection với giới hạn concurrency
public static async Task<IEnumerable<TResult>> WhenAllThrottled<TItem, TResult>(
    IEnumerable<TItem> items,
    Func<TItem, Task<TResult>> func,
    int maxConcurrency)
{
    using var semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();
        try { return await func(item); }
        finally { semaphore.Release(); }
    });
    return await Task.WhenAll(tasks);
}

// Sử dụng:
var results = await WhenAllThrottled(userIds, id => _service.GetUserAsync(id), 10);
```

---

## 🔷 Parallel.For và Parallel.ForEach — Vòng Lặp Song Song

### Parallel.For — Dùng Với Mảng Index

```csharp
// Xử lý mảng số song song — chia đều cho các CPU core
var numbers = Enumerable.Range(1, 1_000_000).ToArray();
var results = new long[numbers.Length];

Parallel.For(0, numbers.Length, i =>
{
    results[i] = HeavyCalculation(numbers[i]);
});

// ✅ Với ParallelOptions để giới hạn degree
Parallel.For(0, numbers.Length,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    i => { results[i] = HeavyCalculation(numbers[i]); });
```

### Parallel.ForEach — Dùng Với Collection

```csharp
var images = LoadImages();

// Xử lý ảnh song song — CPU-bound task
Parallel.ForEach(images, image =>
{
    var compressed = CompressImage(image);
    SaveImage(compressed);
});

// ✅ Với thread-safe aggregation — tổng hợp kết quả an toàn
long totalSize = 0;

Parallel.ForEach(images,
    () => 0L,                              // localInit: giá trị ban đầu cho mỗi thread
    (image, state, localTotal) =>          // body: xử lý từng item
    {
        return localTotal + GetSize(image);
    },
    localTotal =>                          // localFinally: sau khi thread xong
    {
        Interlocked.Add(ref totalSize, localTotal);  // Thread-safe add
    });
```

### Parallel.ForEachAsync — Cho I/O-Bound Operations (.NET 6+)

```csharp
// Kết hợp parallel + async cho I/O-bound tasks
await Parallel.ForEachAsync(
    userIds,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 10,
        CancellationToken = cancellationToken
    },
    async (userId, ct) =>
    {
        var user = await _service.GetUserAsync(userId, ct);
        await _emailService.SendAsync(user.Email, ct);
    });
```

---

## 🔷 PLINQ — Parallel LINQ — LINQ Song Song

### Cú Pháp PLINQ

```csharp
var numbers = Enumerable.Range(1, 1_000_000);

// Sequential LINQ — LINQ tuần tự
var result = numbers
    .Where(n => n % 2 == 0)
    .Select(n => HeavyCalculation(n))
    .Sum();

// ✅ PLINQ — chỉ thêm .AsParallel()
var result = numbers
    .AsParallel()                    // Bật song song
    .Where(n => n % 2 == 0)
    .Select(n => HeavyCalculation(n))
    .Sum();
```

### PLINQ Options

```csharp
var result = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4)           // Tối đa 4 thread
    .WithCancellation(cancellationToken)   // Hỗ trợ hủy
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)  // Bắt buộc song song
    .WithMergeOptions(ParallelMergeOptions.NotBuffered)         // Stream kết quả ngay
    .Where(n => n % 2 == 0)
    .Select(n => Calculate(n))
    .ToList();

// AsOrdered(): giữ thứ tự (chậm hơn)
var ordered = numbers.AsParallel().AsOrdered().Select(n => n * 2).ToList();
```

### Khi Nào PLINQ Không Nên Dùng?

```
PLINQ CÓ LỢI khi:
✅ CPU-bound (tính toán nặng)
✅ Collection lớn (>1000 items)
✅ Workload mỗi item đủ nặng để offset overhead
✅ Không có shared mutable state

PLINQ KHÔNG NÊN dùng khi:
❌ I/O-bound (gọi API, DB) → dùng async/await
❌ Collection nhỏ → overhead lớn hơn lợi ích
❌ Cần thứ tự kết quả (hoặc dùng AsOrdered nhưng chậm)
❌ Có side effects phụ thuộc thứ tự
❌ UI thread (sẽ throw exception)
```

---

## 🔷 Task Continuations — Chuỗi Tiếp Nối

```csharp
// Pattern: xử lý kết quả dựa trên trạng thái task
var task = _service.GetDataAsync();

await task.ContinueWith(t =>
{
    if (t.IsCompletedSuccessfully)
        _logger.LogInformation("Success: {Result}", t.Result);
},
TaskContinuationOptions.OnlyOnRanToCompletion);

await task.ContinueWith(t =>
{
    _logger.LogError(t.Exception, "Failed");
},
TaskContinuationOptions.OnlyOnFaulted);

// ✅ Equivalent với await + try/catch — rõ ràng hơn:
try
{
    var data = await _service.GetDataAsync();
    _logger.LogInformation("Success: {Result}", data);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
}
```

---

## 📊 Khi Nào Dùng Gì: Quyết Định Nhanh

```
Loại Tác Vụ           API Phù Hợp              Lý Do
─────────────────────────────────────────────────────────────────
I/O (API/DB/file)     async/await + Task        Không block thread
Một task CPU nặng     Task.Run                  Đẩy sang background
Nhiều I/O task        Task.WhenAll              Fan-out song song
Nhiều I/O (throttle)  WhenAll + SemaphoreSlim   Kiểm soát concurrency
Mảng lớn CPU         Parallel.For / PLINQ       Data parallelism
Stream liên tục       Channel<T>                Producer-consumer
Kết quả đầu tiên      Task.WhenAny              Racing / timeout
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Task.WhenAll vs Task.WhenAny khác nhau thế nào?**
> `WhenAll`: đợi TẤT CẢ task hoàn thành. `WhenAny`: đợi task ĐẦU TIÊN hoàn thành. Dùng `WhenAll` cho fan-out (cần tất cả kết quả), `WhenAny` cho racing (lấy kết quả nhanh nhất) hoặc timeout.

**Q: Parallel.For vs Task.WhenAll — khi nào dùng gì?**
> `Parallel.For`: CPU-bound, dữ liệu lớn, tính toán nặng — tận dụng đa core. `Task.WhenAll`: I/O-bound, gọi API/DB — không block thread. KHÔNG dùng `Parallel.For` cho I/O-bound operations.

**Q: Tại sao phải giới hạn concurrency khi dùng WhenAll?**
> Tạo quá nhiều task cùng lúc có thể: (1) Overwhelm server phía sau (rate limiting), (2) Tốn quá nhiều memory (mỗi request cần buffer), (3) Exhaust connection pool của DB/HTTP client. Dùng `SemaphoreSlim` để giới hạn số request chạy đồng thời.

**Q: PLINQ có phải luôn nhanh hơn LINQ không?**
> Không. PLINQ có overhead khởi tạo (tạo thread, partition data). Với collection nhỏ hoặc workload nhẹ, PLINQ có thể CHẬM hơn LINQ. Chỉ dùng PLINQ khi đã benchmark xác nhận cải thiện.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
