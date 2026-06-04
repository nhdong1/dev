# CancellationToken — Hủy Tác Vụ Bất Đồng Bộ

> `CancellationToken` là cơ chế để thông báo cho tác vụ biết rằng "nên dừng lại". Đây là nền tảng của cooperative cancellation — hủy hợp tác — trong .NET: không có cưỡng bức hủy thread, thay vào đó tác vụ tự kiểm tra và dừng đúng lúc.

---

## 📋 Tổng Quan

| Loại | Mô Tả | Dùng Khi |
| ---- | ------ | -------- |
| `CancellationToken` | Token đọc — tác vụ dùng để kiểm tra hủy | Trong method bất đồng bộ |
| `CancellationTokenSource` | Nguồn tạo token và kích hoạt hủy | Bên gọi (caller) |
| `CancellationTokenSource.CreateLinkedTokenSource` | Kết hợp nhiều token | Tổng hợp điều kiện hủy |
| Timeout Cancellation | Tự hủy sau thời gian nhất định | Request timeout |

---

## 🔷 Cơ Chế Hoạt Động

### CancellationTokenSource và CancellationToken

```csharp
// Bên gọi (caller) — người có quyền hủy:
using var cts = new CancellationTokenSource();
CancellationToken token = cts.Token;  // Token chỉ đọc — chỉ để kiểm tra

// Bắt đầu task với token
var task = ProcessDataAsync(token);

// Sau 3 giây, quyết định hủy
await Task.Delay(3000);
cts.Cancel();  // Báo hiệu: "hãy dừng lại"

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Task đã bị hủy");
}
```

### Method Nhận và Truyền Token

```csharp
// ✅ Pattern chuẩn: nhận token, truyền xuống tất cả calls
public async Task ProcessOrdersAsync(CancellationToken ct = default)
{
    var orders = await _repository.GetPendingOrdersAsync(ct);  // Truyền xuống

    foreach (var order in orders)
    {
        ct.ThrowIfCancellationRequested();  // Kiểm tra tại vòng lặp

        await _paymentService.ProcessAsync(order, ct);  // Truyền xuống
        await _emailService.SendConfirmationAsync(order.Email, ct);  // Truyền xuống
    }
}
```

---

## 🔷 Cooperative Cancellation — Hủy Hợp Tác

### Các Cách Kiểm Tra Trạng Thái Hủy

```csharp
public async Task LongRunningTaskAsync(CancellationToken ct)
{
    // Cách 1: ThrowIfCancellationRequested — ném exception nếu đã hủy
    // Dùng tại điểm kiểm tra trong vòng lặp
    for (int i = 0; i < 1_000_000; i++)
    {
        ct.ThrowIfCancellationRequested();  // Ném OperationCanceledException
        ProcessItem(i);
    }

    // Cách 2: IsCancellationRequested — kiểm tra không ném exception
    // Dùng khi cần cleanup trước khi return
    while (HasMoreWork())
    {
        if (ct.IsCancellationRequested)
        {
            await CleanupAsync();  // Dọn dẹp trước khi dừng
            ct.ThrowIfCancellationRequested();  // Ném sau khi cleanup
        }

        await DoWorkAsync(ct);
    }

    // Cách 3: await Task bên trong — tự động phát hiện cancellation
    // Khi await một Task có truyền CancellationToken, nó tự ném khi bị hủy
    await Task.Delay(5000, ct);  // Ném OperationCanceledException nếu ct bị hủy
    var data = await _http.GetStringAsync(url, ct);
}
```

### Xử Lý OperationCanceledException

```csharp
// OperationCanceledException là expected — không phải lỗi thực sự
public async Task RunWithCancellationAsync(CancellationToken ct)
{
    try
    {
        await DoWorkAsync(ct);
    }
    catch (OperationCanceledException) when (ct.IsCancellationRequested)
    {
        // ✅ Cancellation bình thường — log và trả về gracefully
        _logger.LogInformation("Operation was cancelled");
        // KHÔNG re-throw nếu muốn xử lý im lặng
    }
    catch (OperationCanceledException)
    {
        // ⚠️ Cancellation từ nguồn khác (linked token?) — re-throw
        throw;
    }
    catch (Exception ex)
    {
        // ❌ Lỗi thực sự — log và re-throw hoặc handle
        _logger.LogError(ex, "Operation failed");
        throw;
    }
}
```

---

## 🔷 Timeout — Tự Động Hủy Sau Thời Gian

### Timeout Với CancellationTokenSource

```csharp
// Cách 1: CancellationTokenSource với timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

try
{
    var result = await _service.GetDataAsync(cts.Token);
}
catch (OperationCanceledException)
{
    throw new TimeoutException("Request exceeded 10 seconds");
}

// Cách 2: CancelAfter — đặt timeout sau khi tạo
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(10));
```

### HttpClient Timeout vs CancellationToken Timeout

```csharp
// HttpClient.Timeout: áp dụng cho toàn bộ lifecycle của request
// CancellationToken: có thể hủy từ bên ngoài sớm hơn

public async Task<string> FetchDataAsync(
    string url,
    TimeSpan timeout,
    CancellationToken externalCt = default)
{
    // LinkedToken: hủy khi HOẶC timeout HOẶC external cancel
    using var timeoutCts = new CancellationTokenSource(timeout);
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        externalCt, timeoutCts.Token);

    try
    {
        return await _http.GetStringAsync(url, linkedCts.Token);
    }
    catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
    {
        throw new TimeoutException($"Request to {url} timed out after {timeout}");
    }
    // OperationCanceledException từ externalCt sẽ bubble up tự nhiên
}
```

---

## 🔷 Linked Token — Kết Hợp Nhiều Token

### CreateLinkedTokenSource

```csharp
// Kết hợp: hủy khi BẤT KỲ token nào bị hủy
public async Task ProcessWithLinkedCancellationAsync(
    CancellationToken userCt,       // Người dùng cancel
    CancellationToken shutdownCt)   // App shutdown
{
    // LinkedCts bị hủy khi userCt HOẶC shutdownCt bị hủy
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        userCt, shutdownCt);

    await DoWorkAsync(linkedCts.Token);
}
```

### Real-World: ASP.NET Core Request Cancellation

```csharp
// ASP.NET Core tự động truyền CancellationToken khi client disconnect
[HttpGet("data")]
public async Task<IActionResult> GetData(CancellationToken ct)
{
    // ct bị hủy khi: client đóng kết nối hoặc request timeout
    try
    {
        var data = await _service.GetExpensiveDataAsync(ct);
        return Ok(data);
    }
    catch (OperationCanceledException)
    {
        // Client đã disconnect — không cần trả kết quả
        return StatusCode(499);  // Client Closed Request (Nginx convention)
    }
}

// ✅ Lợi ích: nếu client đóng tab, DB query bị cancel → tiết kiệm tài nguyên
```

---

## 🔷 CancellationToken Với Các API Khác

### Task.Delay Với CancellationToken

```csharp
// Delay có thể bị cancel sớm
public async Task WaitOrCancelAsync(int seconds, CancellationToken ct)
{
    try
    {
        await Task.Delay(TimeSpan.FromSeconds(seconds), ct);
        Console.WriteLine("Đã chờ đủ thời gian");
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Chờ bị hủy sớm");
    }
}
```

### HttpClient Với CancellationToken

```csharp
// Hủy HTTP request đang chạy
public async Task<string> GetWithTimeoutAsync(string url, CancellationToken ct)
{
    using var response = await _httpClient.GetAsync(url, ct);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync(ct);
}
```

### Entity Framework Core Với CancellationToken

```csharp
// Hủy DB query
public async Task<List<User>> GetActiveUsersAsync(CancellationToken ct)
{
    return await _db.Users
        .Where(u => u.IsActive)
        .OrderBy(u => u.Name)
        .ToListAsync(ct);  // EF Core hỗ trợ cancellation
}
```

---

## 🔷 BackgroundService Và Graceful Shutdown

```csharp
// ✅ Pattern chuẩn trong BackgroundService (hosted service)
public class DataProcessingService : BackgroundService
{
    private readonly ILogger<DataProcessingService> _logger;

    public DataProcessingService(ILogger<DataProcessingService> logger)
        => _logger = logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // stoppingToken bị hủy khi app shutdown
        _logger.LogInformation("Service started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessNextBatchAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // App đang shutdown — dừng vòng lặp
                break;
            }
            catch (Exception ex)
            {
                // Lỗi thực sự — log và tiếp tục (không crash service)
                _logger.LogError(ex, "Error processing batch");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }

        _logger.LogInformation("Service stopped gracefully");
    }

    private async Task ProcessNextBatchAsync(CancellationToken ct)
    {
        var items = await _queue.DequeueAsync(ct);
        foreach (var item in items)
        {
            ct.ThrowIfCancellationRequested();
            await ProcessItemAsync(item, ct);
        }
    }
}
```

---

## 🔷 Manual Cancellation — Hủy Thủ Công

```csharp
// Cho phép user cancel từ UI
public class OrderProcessor
{
    private CancellationTokenSource? _cts;

    public async Task StartProcessingAsync()
    {
        _cts = new CancellationTokenSource();
        try
        {
            await ProcessAllOrdersAsync(_cts.Token);
        }
        finally
        {
            _cts.Dispose();
            _cts = null;
        }
    }

    public void Cancel()
    {
        _cts?.Cancel();  // Thread-safe: Cancel() có thể gọi từ thread khác
    }

    private async Task ProcessAllOrdersAsync(CancellationToken ct)
    {
        var orders = await _repo.GetPendingAsync(ct);
        foreach (var order in orders)
        {
            ct.ThrowIfCancellationRequested();
            await ProcessOrderAsync(order, ct);
        }
    }
}
```

---

## 🚫 Anti-Patterns Phổ Biến

### 1. Không Truyền CancellationToken Xuống

```csharp
// ❌ SAI: Nhận token nhưng không dùng
public async Task ProcessAsync(CancellationToken ct)
{
    var data = await _repository.GetAllAsync();  // Không truyền ct → không thể cancel
    foreach (var item in data)
        await _service.ProcessAsync(item);  // Không truyền ct → không thể cancel
}

// ✅ ĐÚNG: Truyền xuống tất cả
public async Task ProcessAsync(CancellationToken ct)
{
    var data = await _repository.GetAllAsync(ct);
    foreach (var item in data)
    {
        ct.ThrowIfCancellationRequested();
        await _service.ProcessAsync(item, ct);
    }
}
```

### 2. Bắt Exception Không Cẩn Thận

```csharp
// ❌ SAI: Nuốt OperationCanceledException — task không thể bị cancel
public async Task ProcessAsync(CancellationToken ct)
{
    try
    {
        await DoLongWorkAsync(ct);
    }
    catch (Exception ex)  // Bắt tất cả, kể cả OperationCanceledException!
    {
        _logger.LogError(ex, "Error");
        // OperationCanceledException bị nuốt → caller nghĩ task hoàn thành bình thường
    }
}

// ✅ ĐÚNG: Re-throw cancellation exception
public async Task ProcessAsync(CancellationToken ct)
{
    try
    {
        await DoLongWorkAsync(ct);
    }
    catch (OperationCanceledException)
    {
        throw;  // Re-throw để cancellation propagate đúng cách
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error");
        throw;
    }
}
```

### 3. Tạo CancellationTokenSource Không Dispose

```csharp
// ❌ SAI: Không dispose — memory leak
public async Task ProcessAsync()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    await DoWorkAsync(cts.Token);
    // cts không được dispose → leak timer handle
}

// ✅ ĐÚNG: Dùng using
public async Task ProcessAsync()
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    await DoWorkAsync(cts.Token);
}  // cts.Dispose() được gọi tự động
```

---

## 📊 Tóm Tắt: Khi Nào Dùng Gì

```
Tình Huống                    Giải Pháp
──────────────────────────────────────────────────────
Request có timeout           CancellationTokenSource(timeout)
User cancel (button)         CancellationTokenSource + Cancel()
App shutdown (graceful)      BackgroundService.stoppingToken
Vừa timeout vừa user cancel  CreateLinkedTokenSource
ASP.NET request cancel       HttpContext.RequestAborted
Kiểm tra trong vòng lặp      ThrowIfCancellationRequested()
Cleanup trước khi dừng       IsCancellationRequested + cleanup + throw
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: CancellationToken và CancellationTokenSource khác nhau thế nào?**
> `CancellationTokenSource` là người có quyền HỦY — tạo token và gọi `Cancel()`. `CancellationToken` là token chỉ ĐỌC — chỉ để kiểm tra xem đã bị hủy chưa. Separation of concerns: tác vụ chỉ được biết "đã bị hủy chưa", không được hủy chính nó.

**Q: Tại sao không nên dùng Thread.Abort() mà phải dùng CancellationToken?**
> `Thread.Abort()` đã bị xóa trong .NET Core vì nó ném `ThreadAbortException` tại điểm bất kỳ, khiến code dở dang và có thể corrupt state. `CancellationToken` là cooperative — tác vụ tự quyết định dừng tại điểm an toàn, đảm bảo cleanup đúng cách.

**Q: ASP.NET Core tự động truyền CancellationToken thế nào?**
> Khi Controller action có parameter `CancellationToken ct`, ASP.NET Core model binding tự động binding từ `HttpContext.RequestAborted`. Token này bị hủy khi client disconnect (đóng tab, timeout). Đây là lý do luôn nên nhận `CancellationToken` trong action method.

**Q: Sự khác nhau giữa IsCancellationRequested và ThrowIfCancellationRequested?**
> `IsCancellationRequested`: chỉ kiểm tra, trả về `bool`, không ném exception — dùng khi cần cleanup trước khi dừng. `ThrowIfCancellationRequested()`: kiểm tra và ném `OperationCanceledException` nếu đã cancel — dùng trong vòng lặp để dừng nhanh.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
