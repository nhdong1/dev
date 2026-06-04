# Deadlock Prevention — Phòng Tránh Bế Tắc

> Deadlock — Bế Tắc — xảy ra khi hai hoặc nhiều thread chờ nhau vô hạn, không thread nào có thể tiến lên. Đây là một trong những lỗi khó debug nhất vì không có exception, app chỉ đơn giản là "đóng băng" (hang) mà không báo lỗi gì.

---

## 📋 Tổng Quan

| Loại Deadlock | Nguyên Nhân | Giải Pháp |
| ------------- | ----------- | --------- |
| Classic lock deadlock | Lock theo thứ tự khác nhau | Lock ordering — thứ tự khóa nhất quán |
| Async deadlock | `.Result`/`.Wait()` + SynchronizationContext | Async all the way |
| Resource starvation | Tất cả thread bị block, không ai release | Timeout, bounded queue |
| Thread pool starvation | Thread pool cạn kiệt | Tránh blocking trong thread pool |

---

## 🔷 Deadlock Điều Kiện 4 Coffman

```
Deadlock xảy ra khi đồng thời có đủ 4 điều kiện:

1. Mutual Exclusion — Loại Trừ Lẫn Nhau:
   Tài nguyên chỉ có một thread dùng tại một thời điểm.
   (lock, semaphore)

2. Hold and Wait — Giữ Và Chờ:
   Thread đang giữ tài nguyên A, chờ tài nguyên B.

3. No Preemption — Không Giành Được:
   Tài nguyên không thể bị lấy cưỡng bức — phải chờ release tự nguyện.

4. Circular Wait — Chờ Vòng Tròn:
   Thread A chờ Thread B, Thread B chờ Thread A (hoặc chu trình dài hơn).

→ Phá vỡ BẤT KỲ điều kiện nào trong 4 → không có deadlock.
```

---

## 🔷 Classic Lock Deadlock — Bế Tắc Với Lock

### Ví Dụ Điển Hình

```csharp
// ❌ DEADLOCK: Hai thread lấy lock theo thứ tự ngược nhau
public class BankTransfer
{
    private readonly object _lockA = new();
    private readonly object _lockB = new();

    // Thread 1 gọi: Transfer(accountA, accountB, 100)
    // Thread 2 gọi: Transfer(accountB, accountA, 200)
    public void Transfer(object from, object to, decimal amount)
    {
        lock (from)   // Thread 1: lock A → Thread 2: lock B
        {
            Thread.Sleep(10);  // Giả lập processing
            lock (to)  // Thread 1: chờ B (đang bị T2 giữ)
            {          // Thread 2: chờ A (đang bị T1 giữ)
                       // DEADLOCK! Không ai nhường ai
                DoTransfer(from, to, amount);
            }
        }
    }
}
```

### Giải Pháp: Lock Ordering — Thứ Tự Khóa Nhất Quán

```csharp
// ✅ ĐÚNG: Luôn lock theo thứ tự đã định trước (theo ID)
public class BankTransfer
{
    public void Transfer(Account from, Account to, decimal amount)
    {
        // Quy tắc: luôn lock account có ID nhỏ hơn trước
        var first  = from.Id < to.Id ? from : to;
        var second = from.Id < to.Id ? to : from;

        lock (first._lock)   // Luôn theo thứ tự → không có circular wait
        {
            lock (second._lock)
            {
                from.Balance -= amount;
                to.Balance += amount;
            }
        }
    }
}
```

### Giải Pháp: Monitor.TryEnter Với Timeout

```csharp
// ✅ ĐÚNG: Thử lấy lock, từ bỏ nếu không được sau timeout
public bool TryTransfer(object from, object to, decimal amount)
{
    bool fromLocked = false, toLocked = false;

    try
    {
        Monitor.TryEnter(from, TimeSpan.FromSeconds(1), ref fromLocked);
        if (!fromLocked) return false;  // Không lấy được lock — từ bỏ

        Monitor.TryEnter(to, TimeSpan.FromSeconds(1), ref toLocked);
        if (!toLocked) return false;    // Không lấy được lock — từ bỏ

        DoTransfer(from, to, amount);
        return true;
    }
    finally
    {
        if (toLocked)   Monitor.Exit(to);
        if (fromLocked) Monitor.Exit(from);
    }
}
```

---

## 🔷 Async Deadlock — Bế Tắc Bất Đồng Bộ

### Nguyên Nhân Phổ Biến Nhất Trong .NET

```csharp
// ❌ DEADLOCK trong ASP.NET Classic hoặc WPF:
// Kịch bản: Controller gọi async method bằng .Result

// Controller (chạy trên ASP.NET request thread với SynchronizationContext)
public ActionResult Index()
{
    // .Result block request thread
    var data = GetDataAsync().Result;  // Thread bị block ở đây!
    return View(data);
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(100);
    // Sau await: cần resume trên request thread (do SynchronizationContext)
    // Nhưng request thread đang bị block bởi .Result
    // → DEADLOCK vĩnh cửu
    return "data";
}
```

```
Phân Tích Deadlock:
┌─────────────────────────────────────────────────┐
│ Request Thread                                   │
│  → Gọi GetDataAsync().Result                     │
│  → Block (chờ task xong)                        │
│  ← Task cần request thread để resume            │
│  → Không thể resume (thread đang block)         │
│  → DEADLOCK                                      │
└─────────────────────────────────────────────────┘
```

### Giải Pháp 1: Async All the Way — Bất Đồng Bộ Từ Đầu

```csharp
// ✅ ĐÚNG: await xuyên suốt — không bao giờ dùng .Result hay .Wait()
public async Task<ActionResult> Index()
{
    var data = await GetDataAsync();  // Không block thread
    return View(data);
}
```

### Giải Pháp 2: ConfigureAwait(false) Trong Library

```csharp
// ✅ Nếu bắt buộc phải block (legacy code), dùng ConfigureAwait(false)
public async Task<string> GetDataAsync()
{
    await Task.Delay(100).ConfigureAwait(false);
    // ConfigureAwait(false): không cần resume trên original SyncContext
    // → Không bị block bởi .Result ở caller
    return "data";
}

// Caller (legacy):
var data = GetDataAsync().Result;  // Không deadlock vì ConfigureAwait(false)
// Nhưng: vẫn block thread → vẫn là anti-pattern, chỉ dùng khi không thể thay đổi
```

### Giải Pháp 3: Task.Run Để Thoát SynchronizationContext

```csharp
// ✅ Giải pháp tạm thời khi legacy code bắt buộc phải .Wait()
var data = Task.Run(() => GetDataAsync()).Result;
// Task.Run chạy trên ThreadPool (không có SyncContext)
// → Không cần resume trên original thread → không deadlock
// Nhưng: lãng phí thread, không phải giải pháp lý tưởng
```

---

## 🔷 Thread Pool Starvation — Cạn Kiệt Thread Pool

### Vấn Đề

```csharp
// ❌ STARVATION: Tất cả thread pool thread bị block — không còn thread xử lý I/O completions

// Thread pool mặc định: ~250 threads (4 threads/core)
// Mỗi request block một thread:
for (int i = 0; i < 300; i++)
{
    Task.Run(() =>
    {
        // Block thread pool thread vô hạn
        Thread.Sleep(Timeout.Infinite);
        // Hoặc:
        SomeAsyncMethod().Wait();  // Block thread chờ I/O
    });
}
// Thread pool cạn kiệt → async operations không thể hoàn thành
// → Toàn bộ app đóng băng (không phải deadlock thực sự nhưng triệu chứng giống)
```

### Phát Hiện Thread Pool Starvation

```csharp
// Monitoring thread pool:
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
ThreadPool.GetAvailableThreads(out int availWorker, out int availIO);

_logger.LogInformation(
    "ThreadPool: Available={Available}/{Max}, IO={IOAvail}/{IOMax}",
    availWorker, maxWorker, availIO, maxIO);

// Khi availWorker → 0: starvation đang xảy ra
```

### Giải Pháp: Không Block Thread Pool Thread

```csharp
// ❌ SAI: Block thread trong thread pool
Task.Run(() =>
{
    var result = GetDataAsync().Result;  // Block!
    Process(result);
});

// ✅ ĐÚNG: Async trong Task.Run
Task.Run(async () =>
{
    var result = await GetDataAsync();  // Không block
    await ProcessAsync(result);
});
```

---

## 🔷 Phát Hiện Deadlock

### Sử Dụng Timeout

```csharp
// Wrap mọi thao tác với timeout — không chờ vô hạn
public async Task<T> WithTimeout<T>(
    Func<CancellationToken, Task<T>> operation,
    TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    try
    {
        return await operation(cts.Token);
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        throw new TimeoutException($"Operation timed out after {timeout}");
    }
}

// Sử dụng:
var result = await WithTimeout(
    ct => _service.GetDataAsync(ct),
    TimeSpan.FromSeconds(30));
```

### Logging Và Diagnostics

```csharp
// Phát hiện lock hold time — thời gian giữ lock
public class DiagnosticLock
{
    private readonly object _lock = new();
    private readonly ILogger _logger;
    private readonly TimeSpan _warnThreshold;

    public T Execute<T>(Func<T> action)
    {
        var sw = Stopwatch.StartNew();
        lock (_lock)
        {
            var waitTime = sw.Elapsed;
            if (waitTime > _warnThreshold)
                _logger.LogWarning("Lock wait time: {Time}ms — potential contention", waitTime.TotalMilliseconds);

            sw.Restart();
            var result = action();

            if (sw.Elapsed > _warnThreshold)
                _logger.LogWarning("Lock hold time: {Time}ms — too long!", sw.Elapsed.TotalMilliseconds);

            return result;
        }
    }
}
```

### Sử Dụng dotnet-dump Và Windbg

```bash
# Lấy process dump khi app bị hang
dotnet-dump collect --process-id <PID>

# Phân tích dump
dotnet-dump analyze ./core_20260602.dmp

# Xem tất cả thread stacks
> threads
> clrstack  # Xem stack của thread hiện tại
> ~*e clrstack  # Xem stack của tất cả threads

# Tìm thread đang giữ lock
> syncblk  # Xem SyncBlock table — bảng sync block
```

---

## 🔷 Nguyên Tắc Phòng Tránh Deadlock

### 1. Async All the Way — Không Bao Giờ Block

```csharp
// Quy tắc vàng: KHÔNG BAO GIỜ gọi .Result, .Wait(), GetAwaiter().GetResult()
// trong async context hoặc code có SynchronizationContext

// Ngoại lệ chấp nhận:
// - Trong Main() của Console app (khi không thể async Main)
// - Trong static constructor
// - Trong property getter không thể async
// → Trong những trường hợp đó, dùng .GetAwaiter().GetResult() + ConfigureAwait(false)
```

### 2. Lock Ordering — Thứ Tự Khóa Nhất Quán

```csharp
// Tài liệu hóa thứ tự lock trong team
// Ví dụ: Lock A trước B trước C — không bao giờ ngược lại
// Dùng ID/hashcode để xác định thứ tự khi không có thứ tự tự nhiên
```

### 3. Giữ Lock Ngắn Nhất Có Thể

```csharp
// ❌ SAI: Tính toán phức tạp trong lock
lock (_lock)
{
    var result = DoHeavyComputation(data);  // 500ms trong lock!
    _cache[key] = result;
}

// ✅ ĐÚNG: Chỉ lock operation thực sự cần bảo vệ
var result = DoHeavyComputation(data);  // Tính toán ngoài lock
lock (_lock)
{
    _cache[key] = result;  // Chỉ write cần lock — microseconds
}
```

### 4. Tránh Nested Locks — Khóa Lồng Nhau

```csharp
// ❌ NGUY HIỂM: Nested locks — dễ gây circular wait
lock (_lockA)
{
    lock (_lockB)   // Nguy hiểm nếu code khác lock B rồi A
    {
        DoWork();
    }
}

// ✅ TỐT HƠN: Tái cấu trúc để tránh nested locks
// Hoặc dùng một lock object duy nhất cho cả hai resource
// Hoặc dùng lock ordering cố định
```

### 5. Immutability — Không Cần Lock Nếu Không Thay Đổi

```csharp
// Immutable objects: chia sẻ giữa các thread không cần lock
public sealed record UserSnapshot(int Id, string Name, DateTime Timestamp);

// Không cần lock khi đọc UserSnapshot từ nhiều thread
// vì object không thể thay đổi sau khi tạo
```

---

## 🔷 Deadlock Checklist — Danh Sách Kiểm Tra

```
❌ Các Dấu Hiệu Nguy Hiểm (Code Review Red Flags):
   □ .Result hoặc .Wait() trong controller/service với SyncContext
   □ lock chứa await bên trong
   □ Nested lock (lock trong lock)
   □ Lock không có timeout
   □ Thread.Sleep trong lock
   □ Gọi method có thể lock trong khi đang lock object đó

✅ Các Giải Pháp:
   □ Async all the way — không bao giờ block
   □ ConfigureAwait(false) trong library code
   □ Lock ordering nhất quán và được tài liệu hóa
   □ Timeout cho mọi blocking operation
   □ SemaphoreSlim thay lock cho async code
   □ Immutable data structures cho shared state
   □ Monitor.TryEnter với timeout thay lock cho lock phức tạp
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Deadlock là gì? Bốn điều kiện cần để xảy ra?**
> Deadlock là trạng thái hai hoặc nhiều thread chờ nhau vô hạn, không ai tiến lên được. Bốn điều kiện Coffman: (1) Mutual Exclusion — tài nguyên không chia sẻ, (2) Hold and Wait — giữ một tài nguyên, chờ tài nguyên khác, (3) No Preemption — không thể lấy cưỡng bức, (4) Circular Wait — chờ vòng tròn. Phá vỡ bất kỳ điều kiện nào → không deadlock.

**Q: Tại sao .Result gây deadlock trong ASP.NET Classic?**
> ASP.NET Classic có `AspNetSynchronizationContext`. Khi `.Result` block request thread, và async method cần resume trên chính thread đó (do SynchronizationContext), nhưng thread đó đang bị block bởi `.Result` → circular wait → deadlock. Giải pháp: async all the way, hoặc ConfigureAwait(false) trong async method.

**Q: Làm sao phòng tránh deadlock khi cần lock nhiều tài nguyên cùng lúc?**
> Lock ordering — luôn lock theo thứ tự đã xác định trước (ví dụ: theo ID), không bao giờ đảo ngược thứ tự. Điều này phá vỡ điều kiện Circular Wait. Hoặc dùng Monitor.TryEnter với timeout — nếu không lấy được lock trong thời gian quy định, từ bỏ và thử lại.

**Q: Thread pool starvation khác deadlock thế nào?**
> Deadlock: các thread chờ nhau vô hạn — không thể thoát được. Thread pool starvation: tất cả thread bị block (thường do sync-over-async), không còn thread để xử lý I/O completion → mọi thứ đóng băng. Starvation có thể tự giải quyết nếu có thread nào đó release. Deadlock không bao giờ tự giải quyết.

**Q: Cách debug deadlock trong production?**
> (1) Lấy process dump khi app hang: `dotnet-dump collect --process-id <PID>`. (2) Phân tích bằng `dotnet-dump analyze`: xem thread stacks với `clrstack`, xem lock state với `syncblk`. (3) Tìm thread đang giữ lock mà thread khác đang chờ. (4) Trong code: thêm timeout cho mọi blocking operation, log khi lock wait time cao.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
