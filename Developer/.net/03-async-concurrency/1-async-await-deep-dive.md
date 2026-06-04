# Async/Await Deep Dive — Hiểu Sâu Lập Trình Bất Đồng Bộ

> `async`/`await` là cú pháp đường (syntactic sugar — cú pháp tiện lợi) được compiler C# biến đổi thành một state machine — máy trạng thái. Hiểu cơ chế bên dưới giúp bạn tránh deadlock, tối ưu hiệu năng, và debug lỗi bất đồng bộ hiệu quả.

---

## 📋 Tổng Quan Nhanh

| Khái Niệm | Giải Thích |
| --------- | ---------- |
| `async` | Modifier báo cho compiler tạo state machine |
| `await` | Điểm tạm dừng — suspension point — trả thread về pool |
| State Machine | Máy trạng thái được compiler sinh ra thay cho async method |
| `SynchronizationContext` | Ngữ cảnh đồng bộ hóa — quy định thread nào tiếp tục sau await |
| `ConfigureAwait(false)` | Bỏ qua SynchronizationContext — dùng trong library code |
| `TaskScheduler` | Bộ lập lịch tác vụ — quyết định thread nào chạy Task |

---

## 🔬 Async/Await Là Gì Bên Dưới?

### Ví Dụ Đơn Giản

```csharp
// Code bạn viết:
public async Task<string> GetUserNameAsync(int userId)
{
    var user = await _repository.GetByIdAsync(userId);
    return user.Name;
}
```

### Compiler Sinh Ra Gì?

Compiler C# biến đổi method trên thành một **state machine** — **máy trạng thái** tương đương với:

```csharp
// Đây là code TƯƠNG ĐƯƠNG được compiler sinh ra (đã đơn giản hóa)
public Task<string> GetUserNameAsync(int userId)
{
    var stateMachine = new GetUserNameAsyncStateMachine
    {
        _this = this,
        _userId = userId,
        _state = -1  // Trạng thái ban đầu
    };
    stateMachine.MoveNext();
    return stateMachine._builder.Task;
}

// State machine được compiler sinh ra
private struct GetUserNameAsyncStateMachine : IAsyncStateMachine
{
    public int _state;          // Trạng thái hiện tại: -1, 0, 1...
    public AsyncTaskMethodBuilder<string> _builder;
    public ExampleClass _this;
    public int _userId;
    private User _user;         // Biến cục bộ được "lưu" qua await point
    private TaskAwaiter<User> _awaiter;

    public void MoveNext()
    {
        switch (_state)
        {
            case -1:  // Lần chạy đầu tiên
                _awaiter = _this._repository.GetByIdAsync(_userId).GetAwaiter();

                if (!_awaiter.IsCompleted)
                {
                    _state = 0;  // Đặt state để biết phải resume ở đâu
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;      // Trả thread về ThreadPool — KHÔNG block!
                }
                goto case 0;

            case 0:  // Tiếp tục sau khi GetByIdAsync hoàn thành
                _user = _awaiter.GetResult();
                _builder.SetResult(_user.Name);  // Hoàn thành Task<string>
                return;
        }
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

### Giải Thích Quá Trình

```
1. Gọi GetUserNameAsync(42)
   → Compiler tạo state machine, gọi MoveNext() lần 1
   → state = -1: bắt đầu GetByIdAsync

2. GetByIdAsync chưa xong (I/O đang chạy)
   → state = 0 (ghi nhớ "nơi sẽ tiếp tục")
   → Trả thread về ThreadPool — thread KHÔNG bị block
   → Một Task<string> được trả về caller

3. DB trả kết quả → I/O completion callback kích hoạt
   → Thread từ pool (có thể khác thread ban đầu) gọi MoveNext() lần 2
   → state = 0: lấy kết quả từ _awaiter, trả về user.Name

4. Task<string> hoàn thành với giá trị "Alice"
   → Caller nhận được kết quả
```

---

## 🌐 SynchronizationContext — Ngữ Cảnh Đồng Bộ Hóa

### SynchronizationContext Là Gì?

`SynchronizationContext` — Ngữ cảnh đồng bộ hóa — là cơ chế quy định **thread nào sẽ chạy phần code sau `await`**.

```
Môi Trường              SynchronizationContext        Hành Vi Sau Await
─────────────────────────────────────────────────────────────────────
ASP.NET Core            null (không có)               Bất kỳ thread từ pool
WPF/WinForms            DispatcherSynchronizationCtx  Phải chạy trên UI thread
ASP.NET Classic         AspNetSynchronizationContext  Phải chạy trên request thread
Console App             null                          Bất kỳ thread từ pool
```

### Tại Sao ASP.NET Classic Có Thể Gây Deadlock?

```csharp
// Trong ASP.NET Classic (KHÔNG phải ASP.NET Core):
// Controller action (chạy trên request thread với SyncContext)
public ActionResult Index()
{
    // .Result BLOCK request thread
    var data = GetDataAsync().Result;  // ❌ DEADLOCK!
    return View(data);
}

public async Task<string> GetDataAsync()
{
    // await muốn resume trên request thread (do SyncContext)
    // nhưng request thread đang bị block bởi .Result
    // → DEADLOCK — bế tắc vĩnh cửu!
    await Task.Delay(100);
    return "data";
}
```

```
Luồng Deadlock:
1. Request Thread chạy Index()
2. Index() gọi GetDataAsync().Result → block Request Thread
3. GetDataAsync() await → cần Request Thread để resume
4. Request Thread đang bị block → không thể resume
5. DEADLOCK — không ai giải phóng ai được
```

### ASP.NET Core Không Có Vấn Đề Này

```csharp
// ASP.NET Core: không có SynchronizationContext
// Controller action
[HttpGet]
public async Task<IActionResult> Index()
{
    var data = await GetDataAsync();  // ✅ Không cần specific thread
    return Ok(data);
}
// Resume trên bất kỳ thread nào từ pool → không có deadlock
```

---

## ⚙️ ConfigureAwait(false) — Cấu Hình Await

### Vấn Đề ConfigureAwait Giải Quyết

```csharp
// Trong library code (thư viện):
public async Task<User> GetUserAsync(int id)
{
    // Mặc định: sau await, cố gắng resume trên SynchronizationContext gốc
    // Trong WPF/ASP.NET Classic, điều này có thể gây deadlock hoặc chậm
    var data = await _httpClient.GetStringAsync($"/users/{id}");
    return JsonSerializer.Deserialize<User>(data);
}
```

```csharp
// ✅ ĐÚNG trong library code:
public async Task<User> GetUserAsync(int id)
{
    // ConfigureAwait(false): KHÔNG cần resume trên SynchronizationContext gốc
    // Resume trên bất kỳ thread từ pool → nhanh hơn, không deadlock
    var data = await _httpClient.GetStringAsync($"/users/{id}")
        .ConfigureAwait(false);

    return JsonSerializer.Deserialize<User>(data);
}
```

### Khi Nào Dùng ConfigureAwait(false)?

```
✅ Nên dùng ConfigureAwait(false):
- Trong thư viện (NuGet package, class library)
- Trong code không cần truy cập UI context
- Trong background service, worker service

❌ Không nên dùng ConfigureAwait(false):
- Trong ASP.NET Core controller action (đã không có SyncContext)
- Trong code cần truy cập HttpContext sau await
- Trong WPF/WinForms code cần cập nhật UI sau await
```

### ConfigureAwait Trong ASP.NET Core

```csharp
// ASP.NET Core: ConfigureAwait(false) không hại nhưng cũng không cần thiết
// vì đã không có SynchronizationContext
[HttpGet]
public async Task<IActionResult> GetUser(int id)
{
    // Cả hai đều OK trong ASP.NET Core:
    var user = await _service.GetUserAsync(id);
    // var user = await _service.GetUserAsync(id).ConfigureAwait(false);

    return Ok(user);
}
```

---

## 🎯 Các Trường Hợp Đặc Biệt

### ValueTask — Tối Ưu Cho Hot Path

```csharp
// Task<T>: luôn allocate object trên heap — tốn bộ nhớ
public async Task<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return value;  // Vẫn tạo Task<int> object dù không await

    return await _db.GetValueAsync(key);
}

// ✅ ValueTask<T>: không allocate khi kết quả sẵn có (synchronous path)
public async ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return value;  // Không allocate! Trả về ValueTask trực tiếp trên stack

    return await _db.GetValueAsync(key);
}
```

```
Quy tắc chọn ValueTask vs Task:
- Task<T>:      Hầu hết trường hợp — đơn giản, an toàn
- ValueTask<T>: Hot path mà kết quả thường sẵn có đồng bộ
                (ví dụ: cache hit, in-memory lookup)
KHÔNG dùng ValueTask nếu:
- Kết quả thường không sẵn có đồng bộ
- Task được await nhiều lần
- Task được lưu và await sau
```

### async void — Khi Nào Chấp Nhận?

```csharp
// ❌ SAI: async void trong method thông thường
public async void SendEmailAsync()  // Exception không thể catch từ caller!
{
    await _emailService.SendAsync("...");
}

// Nếu exception xảy ra trong async void method:
try
{
    SendEmailAsync();  // Không thể await → không catch được exception!
}
catch (Exception ex)
{
    // KHÔNG bao giờ vào đây — exception crash process!
}

// ✅ ĐÚNG: async Task
public async Task SendEmailAsync()
{
    await _emailService.SendAsync("...");
}

// ✅ CHẤP NHẬN: Event handler — đây là use case duy nhất hợp lệ
button.Click += async (sender, e) =>
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex)
    {
        MessageBox.Show(ex.Message);  // Handle trong handler
    }
};
```

### Returning Task Without Async

```csharp
// Có thể bỏ async/await khi chỉ truyền tiếp Task:
public Task<string> GetNameAsync(int id)
{
    // KHÔNG dùng async/await: không có state machine, không allocate
    return _repository.GetNameAsync(id);
}

// SO SÁNH: dùng async/await không cần thiết
public async Task<string> GetNameAsync(int id)
{
    // Tạo state machine, tốn thêm resource mà không cần
    return await _repository.GetNameAsync(id);
}

// NHƯNG: phải dùng async/await nếu có try/catch hoặc using
public async Task<string> GetNameAsync(int id)
{
    try
    {
        return await _repository.GetNameAsync(id);  // PHẢI await để catch exception
    }
    catch (NotFoundException)
    {
        return "Unknown";
    }
}
```

---

## 🚫 Anti-Patterns Phổ Biến

### 1. .Result và .Wait() — Blocking Call

```csharp
// ❌ SAI: Blocking
var result = GetDataAsync().Result;    // Nguy hiểm
var result = GetDataAsync().GetAwaiter().GetResult();  // Ít nguy hiểm hơn nhưng vẫn block

// ✅ ĐÚNG:
var result = await GetDataAsync();
```

### 2. Fire-and-Forget Không Cẩn Thận

```csharp
// ❌ SAI: Exception bị nuốt, không biết task xong chưa
_ = SendEmailAsync();  // Fire-and-forget — bỏ qua exception

// ✅ TỐT HƠN: Nếu cần fire-and-forget, handle exception
_ = SendEmailAsync().ContinueWith(
    t => _logger.LogError(t.Exception, "Email failed"),
    TaskContinuationOptions.OnlyOnFaulted);

// ✅ TỐT NHẤT: Dùng BackgroundService hoặc queue
await _backgroundQueue.EnqueueAsync(() => SendEmailAsync());
```

### 3. Task.Run Trong ASP.NET Core

```csharp
// ❌ SAI: Task.Run trong controller — lãng phí thread
[HttpGet]
public async Task<IActionResult> Get()
{
    // Đang dùng một thread từ pool, rồi Task.Run lấy thêm thread khác
    var result = await Task.Run(() => _service.GetData());
    return Ok(result);
}

// ✅ ĐÚNG: Nếu GetData() là I/O-bound, dùng async trực tiếp
[HttpGet]
public async Task<IActionResult> Get()
{
    var result = await _service.GetDataAsync();
    return Ok(result);
}

// ✅ CHẤP NHẬN: Nếu GetData() là CPU-bound thực sự (tính toán nặng)
[HttpGet]
public async Task<IActionResult> Calculate()
{
    var result = await Task.Run(() => HeavyCalculation());
    return Ok(result);
}
```

---

## 🔍 IAsyncEnumerable — Async Stream

```csharp
// Trả về dữ liệu theo luồng bất đồng bộ — không cần load hết vào memory
public async IAsyncEnumerable<User> GetUsersStreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var batch in _db.GetUserBatchesAsync(ct))
    {
        foreach (var user in batch)
        {
            yield return user;
        }
    }
}

// Tiêu thụ async stream:
await foreach (var user in GetUsersStreamAsync(cancellationToken))
{
    await ProcessUserAsync(user);
}

// Dùng trong ASP.NET Core endpoint:
[HttpGet("stream")]
public async IAsyncEnumerable<User> StreamUsers(
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var user in _service.GetUsersStreamAsync(ct))
    {
        yield return user;
    }
}
```

---

## 📊 Tóm Tắt: Checklist Async/Await

```
✅ Trước khi viết async method:
   □ Đây có phải I/O-bound không? (DB, HTTP, file)
     → Dùng async/await
   □ Đây có phải CPU-bound không? (tính toán nặng)
     → Dùng Task.Run từ caller (không tự bọc trong method)

✅ Khi viết async method:
   □ Trả về Task/Task<T>/ValueTask<T>, không trả void
   □ Tên method kết thúc bằng Async (quy ước)
   □ Nhận CancellationToken, truyền xuống tất cả calls
   □ Dùng ConfigureAwait(false) nếu là library code
   □ Không dùng .Result / .Wait() — dùng await

✅ Sau khi viết:
   □ Test trường hợp exception propagation
   □ Test cancellation behavior
   □ Check không có blocking call ẩn trong hot path
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: `async`/`await` có tạo ra thread mới không?**
> Không. `await` trả thread về ThreadPool để làm việc khác. Khi I/O xong, một thread (có thể là thread khác) từ pool tiếp tục chạy phần sau `await`. Không tạo thread mới.

**Q: Sự khác nhau giữa `Task.Run` và `async`/`await`?**
> `Task.Run` đẩy công việc sang thread từ ThreadPool (dùng cho CPU-bound). `async`/`await` là cơ chế không block thread trong lúc chờ I/O (dùng cho I/O-bound). Có thể kết hợp: `await Task.Run(() => CpuWork())`.

**Q: Tại sao không nên dùng `.Result` hoặc `.Wait()`?**
> Trong môi trường có SynchronizationContext (ASP.NET Classic, WPF), `.Result` block thread hiện tại trong khi `await` cần thread đó để resume → deadlock. Trong ASP.NET Core không bị deadlock nhưng vẫn lãng phí thread — đánh mất lợi ích của async.

**Q: `ConfigureAwait(false)` có tác dụng gì?**
> Nói với runtime "sau khi await hoàn thành, không cần resume trên SynchronizationContext gốc — bất kỳ thread từ pool đều được". Giúp tránh deadlock trong ASP.NET Classic và WPF, cải thiện hiệu năng trong library code.

**Q: Khi nào dùng `ValueTask<T>` thay vì `Task<T>`?**
> Khi kết quả thường có sẵn đồng bộ (cache hit, in-memory lookup) — tránh allocate Task object. Không dùng khi result thường là async, không await nhiều lần, không store lại.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Thuộc:** Developer/.net/03-async-concurrency/
