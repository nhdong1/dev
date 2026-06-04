# 6 — Exception Handling (Xử Lý Ngoại Lệ)

> Xử lý exception đúng cách là dấu hiệu của developer có kinh nghiệm. Nuốt exception, bắt quá rộng, hay quên finally đều dẫn đến hệ thống không ổn định và khó debug.

---

## 📌 Exception Là Gì?

**Exception** (ngoại lệ) là một đối tượng biểu diễn lỗi xảy ra tại runtime (thời điểm chạy). Khi exception được **thrown** (ném ra), runtime tìm kiếm **catch block** (khối bắt) phù hợp trong call stack (ngăn xếp lời gọi).

```
Exception (gốc của tất cả exceptions)
├── SystemException
│   ├── NullReferenceException     — truy cập null object
│   ├── ArgumentException          — tham số không hợp lệ
│   │   ├── ArgumentNullException
│   │   └── ArgumentOutOfRangeException
│   ├── InvalidOperationException  — thao tác không hợp lệ với trạng thái hiện tại
│   ├── IndexOutOfRangeException   — truy cập index ngoài mảng
│   ├── InvalidCastException       — ép kiểu không hợp lệ
│   ├── OverflowException          — tràn số học
│   ├── StackOverflowException     — đệ quy vô hạn
│   ├── OutOfMemoryException       — hết bộ nhớ
│   └── IOException                — lỗi nhập/xuất
│       ├── FileNotFoundException
│       └── DirectoryNotFoundException
└── ApplicationException          — base cho custom app exceptions (ít dùng)
```

---

## 1. try / catch / finally — Cấu Trúc Cơ Bản

```csharp
try
{
    // Code có thể ném exception
    int result = int.Parse(userInput);
    int divided = 100 / result;
}
catch (FormatException ex)
{
    // Bắt lỗi cụ thể: userInput không phải số
    Console.WriteLine($"Nhập sai định dạng: {ex.Message}");
}
catch (DivideByZeroException ex)
{
    // Bắt lỗi cụ thể: chia cho 0
    Console.WriteLine("Không thể chia cho 0");
}
catch (Exception ex)
{
    // Bắt tất cả các loại còn lại — đặt cuối cùng
    Console.WriteLine($"Lỗi không xác định: {ex.Message}");
    throw; // Re-throw để không mất stack trace
}
finally
{
    // LUÔN chạy dù có exception hay không
    // Dùng để cleanup: đóng file, release resources
    Console.WriteLine("Kết thúc xử lý");
}
```

### Thứ tự bắt exception — Từ cụ thể đến tổng quát

```csharp
// ✅ Đúng: cụ thể trước, tổng quát sau
catch (FileNotFoundException ex) { }
catch (IOException ex) { }
catch (Exception ex) { }

// ❌ Sai: Exception bắt hết, FileNotFoundException không bao giờ được dùng
catch (Exception ex) { }
catch (FileNotFoundException ex) { } // Lỗi compile: unreachable code
```

---

## 2. `throw` và `throw ex` — Sự Khác Biệt Quan Trọng

```csharp
try
{
    RiskyOperation();
}
catch (Exception ex)
{
    // ✅ throw — re-throw giữ nguyên stack trace gốc
    throw;

    // ❌ throw ex — RESET stack trace, mất thông tin nguồn gốc lỗi
    throw ex;
}
```

Luôn dùng `throw;` (không có argument) khi muốn re-throw exception trong catch block.

### Throwing với context — Bọc exception với context mới

```csharp
try
{
    var data = File.ReadAllText(filePath);
}
catch (FileNotFoundException ex)
{
    // Bọc exception với thông tin ngữ cảnh, giữ nguyên inner exception
    throw new ApplicationException($"Không tìm thấy file cấu hình: {filePath}", ex);
}
```

---

## 3. Exception Filters — Lọc Ngoại Lệ (C# 6+)

```csharp
// when clause — chỉ bắt khi điều kiện đúng
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    Console.WriteLine("Tài nguyên không tồn tại (404)");
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
{
    Console.WriteLine("Chưa xác thực (401)");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Lỗi HTTP khác: {ex.StatusCode}");
}
```

Exception filters **không làm mất stack trace** vì stack không bị unwind (tháo gỡ) cho đến khi filter khớp.

```csharp
// Dùng filter để log mà không bắt exception
catch (Exception ex) when (LogException(ex))
{
    // LogException trả về false → exception không bị bắt nhưng được log
}

bool LogException(Exception ex)
{
    logger.LogError(ex, "Unhandled exception");
    return false; // Không bắt exception
}
```

---

## 4. Custom Exceptions — Ngoại Lệ Tùy Chỉnh

```csharp
// Custom exception chuẩn
public class OrderNotFoundException : Exception
{
    public int OrderId { get; }

    public OrderNotFoundException(int orderId)
        : base($"Không tìm thấy đơn hàng với ID: {orderId}")
    {
        OrderId = orderId;
    }

    public OrderNotFoundException(int orderId, Exception innerException)
        : base($"Không tìm thấy đơn hàng với ID: {orderId}", innerException)
    {
        OrderId = orderId;
    }
}

// Ném và bắt custom exception
public Order GetOrder(int orderId)
{
    var order = _repository.Find(orderId);
    if (order == null)
        throw new OrderNotFoundException(orderId);
    return order;
}

try
{
    var order = GetOrder(999);
}
catch (OrderNotFoundException ex)
{
    Console.WriteLine($"OrderId: {ex.OrderId}, Message: {ex.Message}");
}
```

### Phân cấp Custom Exceptions

```csharp
// Base cho domain exceptions
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception inner) : base(message, inner) { }
}

public class InsufficientFundsException : DomainException
{
    public decimal Required { get; }
    public decimal Available { get; }

    public InsufficientFundsException(decimal required, decimal available)
        : base($"Không đủ tiền. Cần {required:C}, có {available:C}")
    {
        Required = required;
        Available = available;
    }
}
```

---

## 5. `AggregateException` — Ngoại Lệ Tổng Hợp

`AggregateException` bọc nhiều exceptions lại, thường gặp khi dùng **Task Parallel Library** (TPL — Thư Viện Tác Vụ Song Song) hoặc `Task.WhenAll`.

```csharp
var tasks = new[]
{
    Task.Run(() => throw new InvalidOperationException("Lỗi task 1")),
    Task.Run(() => throw new ArgumentException("Lỗi task 2")),
    Task.Run(() => { /* thành công */ })
};

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    // await unwrap (mở gói) AggregateException: chỉ throw exception đầu tiên
    Console.WriteLine(ex.Message); // "Lỗi task 1"
}

// Để lấy TẤT CẢ exceptions:
try
{
    Task.WhenAll(tasks).Wait(); // Không dùng await — giữ AggregateException
}
catch (AggregateException ae)
{
    foreach (var ex in ae.Flatten().InnerExceptions)
    {
        Console.WriteLine($"- {ex.GetType().Name}: {ex.Message}");
    }
}
```

### Flatten() — Làm Phẳng AggregateException Lồng Nhau

```csharp
// AggregateException có thể lồng nhau
// ae.Flatten() gộp tất cả InnerExceptions về một tầng
AggregateException flat = ae.Flatten();
foreach (var inner in flat.InnerExceptions) { }
```

---

## 6. Anti-patterns — Những Cách Xử Lý Sai

### ❌ Nuốt Exception (Swallowing Exception)

```csharp
// ĐỪNG LÀM NÀY
try
{
    ProcessOrder(order);
}
catch (Exception)
{
    // Im lặng — bug bị giấu, không ai biết có lỗi
}

// ✅ Ít nhất phải log
catch (Exception ex)
{
    _logger.LogError(ex, "Lỗi khi xử lý đơn hàng {OrderId}", order.Id);
    throw; // Re-throw để caller biết có lỗi
}
```

### ❌ Bắt Exception Quá Rộng Khi Không Cần

```csharp
// ❌ Bắt tất cả nhưng không biết phải làm gì
catch (Exception ex)
{
    return null; // Caller không biết có lỗi!
}

// ✅ Chỉ bắt những gì bạn thực sự xử lý được
catch (FileNotFoundException)
{
    return DefaultConfig(); // Xử lý cụ thể: dùng config mặc định
}
```

### ❌ Dùng Exception cho Flow Control (Điều Khiển Luồng)

```csharp
// ❌ Exception không phải để kiểm soát luồng bình thường — rất chậm
try
{
    var value = dictionary["key"];
}
catch (KeyNotFoundException)
{
    value = defaultValue;
}

// ✅ Dùng TryGetValue
if (!dictionary.TryGetValue("key", out var value))
{
    value = defaultValue;
}
```

### ❌ `async void` — Mất Exception

```csharp
// ❌ async void: exception không thể catch được bên ngoài
async void LoadDataAsync()
{
    throw new Exception("Lỗi nghiêm trọng"); // App crash!
}

// ✅ Dùng async Task
async Task LoadDataAsync()
{
    throw new Exception("Lỗi"); // Caller có thể await và catch
}
```

---

## 7. Best Practices — Thực Hành Tốt

### Fail Fast — Thất Bại Nhanh

```csharp
// Kiểm tra ngay ở đầu method, không chờ đến khi dùng
public void ProcessOrder(Order? order)
{
    ArgumentNullException.ThrowIfNull(order);           // C# 10+
    ArgumentException.ThrowIfNullOrEmpty(order.ItemId); // C# 10+

    if (order.Amount <= 0)
        throw new ArgumentOutOfRangeException(nameof(order.Amount),
            "Số tiền phải lớn hơn 0");

    // Logic thực sự bắt đầu từ đây — biết chắc order hợp lệ
}
```

### Global Exception Handler — Xử Lý Ngoại Lệ Toàn Cục

```csharp
// Trong Program.cs (ASP.NET Core)
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandler = context.Features.Get<IExceptionHandlerFeature>();
        var ex = exceptionHandler?.Error;

        context.Response.StatusCode = ex switch
        {
            NotFoundException => 404,
            ValidationException => 400,
            UnauthorizedException => 401,
            _ => 500
        };

        await context.Response.WriteAsJsonAsync(new
        {
            Error = ex?.Message,
            TraceId = Activity.Current?.Id
        });
    });
});

// Hoặc dùng middleware ProblemDetails (ASP.NET Core 7+)
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

### ExceptionDispatchInfo — Preserve Stack Trace Khi Re-throw Muộn

```csharp
// Khi cần lưu exception và throw ở nơi khác
ExceptionDispatchInfo? capturedException = null;

try
{
    RiskyOperation();
}
catch (Exception ex)
{
    capturedException = ExceptionDispatchInfo.Capture(ex);
}

// ... sau đó ở nơi khác ...
capturedException?.Throw(); // Re-throw với stack trace gốc được bảo toàn
```

---

## 8. Tóm Tắt: Quy Tắc Vàng

| Quy Tắc | Mô Tả |
|---------|-------|
| **Catch cụ thể** | Bắt đúng kiểu exception bạn biết xử lý |
| **Log trước khi nuốt** | Nếu buộc phải nuốt, ít nhất phải log |
| **`throw;` không `throw ex;`** | Giữ nguyên stack trace gốc |
| **`finally` cho cleanup** | Giải phóng resources, không phụ thuộc try/catch |
| **Dùng `using`** | Thay vì try/finally thủ công cho IDisposable |
| **Fail fast** | Validate ngay đầu method, không chờ đến khi crash sâu |
| **Custom exceptions** | Khi cần mang thêm context (dữ liệu) theo exception |
| **Không dùng exception cho flow control** | TryParse, TryGetValue luôn tốt hơn try/catch |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa `throw` và `throw ex`?**  
A: `throw;` re-throw exception và **giữ nguyên** original stack trace. `throw ex;` tạo lại exception từ điểm đó, làm mất thông tin nguồn gốc lỗi ban đầu — rất khó debug.

**Q: `finally` block luôn chạy không?**  
A: Gần như luôn — kể cả khi có exception chưa được bắt, hoặc khi `return` trong try. Ngoại lệ: `StackOverflowException`, `ExecutionEngineException`, hoặc khi process bị kill.

**Q: `AggregateException` là gì?**  
A: Bọc nhiều exceptions từ parallel tasks. Khi `await Task.WhenAll(...)`, C# tự unwrap và throw exception đầu tiên. Để lấy tất cả, dùng `.Wait()` + `AggregateException.Flatten()`.

**Q: Khi nào nên tạo custom exception?**  
A: Khi cần mang thêm dữ liệu theo exception (như OrderId, UserId), hoặc khi caller cần bắt và xử lý loại lỗi đặc thù của domain (miền nghiệp vụ) của bạn.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích sự khác biệt `throw` vs `throw ex`
- [ ] Viết custom exception với additional properties
- [ ] Dùng exception filter (`when`) đúng cách
- [ ] Xử lý `AggregateException` từ `Task.WhenAll`
- [ ] Biết 3 anti-patterns exception handling phổ biến
- [ ] Viết global exception handler cho ASP.NET Core

---

**Hoàn thành:** `01-fundamentals/` — Quay lại [README.md](./README.md) để review  
**Tiếp theo:** [02-oop-patterns/](../02-oop-patterns/) — SOLID & Design Patterns
