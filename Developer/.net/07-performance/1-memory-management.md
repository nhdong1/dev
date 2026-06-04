# Quản Lý Bộ Nhớ trong .NET — GC Internals

> Hiểu sâu GC — Garbage Collector — Bộ Thu Gom Rác là chìa khóa để viết code .NET hiệu quả.

---

## 1. Tổng Quan — Stack vs Heap

### Stack — Ngăn Xếp

```
┌─────────────────────────────────┐
│  Stack Frame (Method A)         │
│  ├── int x = 5      (4 bytes)   │
│  ├── bool flag = true (1 byte)  │
│  └── Point p = {1,2} (8 bytes)  │ ← struct, value type
└─────────────────────────────────┘
  ↑ Tự động giải phóng khi method kết thúc
  ↑ LIFO — Last In First Out — Vào sau ra trước
  ↑ Cực nhanh: chỉ cần dịch stack pointer
```

**Đặc điểm Stack:**
- **Tự động quản lý:** cấp phát/giải phóng khi vào/ra method
- **Tốc độ nhanh:** chỉ cần tăng/giảm stack pointer
- **Kích thước giới hạn:** mặc định 1MB (Windows), 8MB (Linux) — có thể gây `StackOverflowException`
- **Lưu trữ:** value types cục bộ, con trỏ đến heap, frame metadata

### Heap — Đống Bộ Nhớ Được Quản Lý

```
┌─────────────────────────────────────────────────┐
│  Managed Heap — Heap Được Quản Lý                │
│                                                   │
│  ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │   Gen 0    │ │   Gen 1    │ │    Gen 2     │  │
│  │ (ephemeral)│ │ (ephemeral)│ │  (long-live) │  │
│  │  ~256KB    │ │  ~2MB      │ │  unlimited   │  │
│  └────────────┘ └────────────┘ └──────────────┘  │
│                                                   │
│  ┌─────────────────────────────────────────────┐  │
│  │  LOH — Large Object Heap — Heap Đối Tượng  │  │
│  │  Lớn (objects ≥ 85,000 bytes)              │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**Đặc điểm Heap:**
- **GC quản lý:** không giải phóng thủ công, GC lo
- **Linh hoạt:** không giới hạn kích thước (chỉ giới hạn bởi RAM)
- **Chậm hơn Stack:** phải tìm chỗ trống, cập nhật header, thread-safe
- **Lưu trữ:** reference types (class, delegate, string, array), boxed value types

---

## 2. GC — Garbage Collector — Bộ Thu Gom Rác

### Cơ Chế Cơ Bản — Tri-Color Mark and Sweep

GC .NET dùng thuật toán **Mark-and-Compact** — Đánh Dấu và Nén:

```
Bước 1: Mark (Đánh Dấu)
─────────────────────────
  GC roots ──► tất cả object có thể truy cập từ:
    - Stack variables (biến cục bộ)
    - Static fields (trường tĩnh)
    - CPU registers (thanh ghi CPU)
    - GC handles (handle GC)

  Object có thể truy cập = "sống"
  Object không thể truy cập = "chết" → có thể thu gom

Bước 2: Compact (Nén — chỉ Gen 0, 1, 2; không nén LOH)
──────────────────────────────────────────────────────────
  ┌──┬──┬░░┬──┬░░┬░░┬──┐   Trước: có gaps (░░ = dead)
  └──┴──┴──┴──┴──┴──┴──┘
        ↓ compact
  ┌──┬──┬──┬──┬         ┐   Sau: objects dồn lại, cập nhật tất cả con trỏ
  └──┴──┴──┴──┴─────────┘
```

### GC Generations — Thế Hệ GC

**Giả thiết Generational GC:** đối tượng "trẻ" chết sớm, đối tượng "già" sống lâu.

```
Gen 0 — Thế Hệ 0 (Nursery — Vườn Trẻ)
────────────────────────────────────────
  - Mới nhất, nhỏ nhất (~256KB)
  - Hầu hết object chết ở đây
  - Collection rất thường xuyên (có thể vài trăm lần/giây)
  - Rất nhanh: chỉ xét 1 vùng nhỏ

Gen 1 — Thế Hệ 1 (Buffer)
───────────────────────────
  - Object sống qua 1 lần Gen 0 collection
  - Buffer giữa Gen 0 và Gen 2
  - Ít thường xuyên hơn Gen 0

Gen 2 — Thế Hệ 2 (Tenured — Kỳ Cựu)
─────────────────────────────────────
  - Object sống lâu (long-lived objects)
  - Collection ít thường xuyên nhất
  - ĐẮTSẤT: phải quét toàn bộ heap
  - Gây STW — Stop-The-World — Dừng Tất Cả Thread (giây đến vài giây)

LOH — Large Object Heap — Heap Đối Tượng Lớn
──────────────────────────────────────────────
  - Object ≥ 85,000 bytes (thường là array lớn, string dài)
  - Được xem là Gen 2 về mặt logic
  - KHÔNG được compact mặc định (tránh copy dữ liệu lớn)
  - Thu gom khi Gen 2 collection xảy ra
```

### Minh Họa Vòng Đời Object

```csharp
void ProcessRequest()
{
    // object mới → được đặt vào Gen 0
    var dto = new RequestDto { Name = "Alice" };          // Gen 0
    var items = new List<string>(100);                     // Gen 0

    // Nếu GC chạy và dto còn sống → promote lên Gen 1
    // Nếu GC chạy lại và dto vẫn còn sống → promote lên Gen 2

    DoSomething(dto, items);

    // Khi method kết thúc: dto và items không còn root
    // → chờ GC thu gom ở Gen 0 (rất nhanh)
}

// ❌ Vấn đề: giữ reference không cần thiết → object lên Gen 2
static List<User> _cachedUsers; // static → sống mãi → Gen 2 → LOH khi lớn
```

---

## 3. GC Modes — Chế Độ GC

### Workstation GC vs Server GC

```
Workstation GC — GC Trạm Làm Việc
────────────────────────────────────
  - Mặc định cho console và desktop apps
  - 1 heap, 1 background GC thread
  - Ưu tiên: độ trễ thấp, UI responsive

Server GC — GC Máy Chủ
────────────────────────
  - Mặc định cho ASP.NET Core
  - 1 heap per logical CPU core
  - GC chạy song song trên nhiều thread
  - Ưu tiên: throughput cao
  - Dùng nhiều RAM hơn

Cấu hình trong runtimeconfig.json:
{
  "configProperties": {
    "System.GC.Server": true,      // Server GC
    "System.GC.Concurrent": true   // Background GC
  }
}
```

### Background GC vs Blocking GC

```
Blocking GC (Foreground)
─────────────────────────
  - Dừng tất cả thread (Stop-The-World)
  - Gen 0 và Gen 1 luôn dùng blocking
  - Thời gian: microseconds đến milliseconds

Background GC
──────────────
  - Gen 2 chạy song song với app thread
  - Giảm STW pause time
  - Vẫn cần STW ngắn ở một số phase
```

---

## 4. LOH — Large Object Heap — Vấn Đề Thực Tế

```csharp
// Object đủ lớn để vào LOH (≥ 85,000 bytes):
byte[] bigArray = new byte[100_000];        // → LOH
string longString = new string('x', 100_000); // → LOH
double[] matrix = new double[11_000];       // 88,000 bytes → LOH

// ❌ Vấn đề LOH fragmentation — Phân mảnh LOH:
for (int i = 0; i < 1000; i++)
{
    byte[] temp = new byte[100_000]; // cấp phát LOH
    ProcessData(temp);
    // temp trở thành garbage, nhưng LOH KHÔNG compact
    // → LOH ngày càng phân mảnh
}

// ✅ Giải pháp: ArrayPool để tái sử dụng
byte[] buffer = ArrayPool<byte>.Shared.Rent(100_000);
try
{
    ProcessData(buffer);
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer); // trả lại pool, không phân mảnh
}

// Bật LOH compaction (có chi phí cao, chỉ dùng khi cần):
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(); // compact sẽ xảy ra ở lần collect tiếp theo
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.Default;
```

---

## 5. Kỹ Thuật Giảm GC Pressure — Reduce GC Pressure

### 5.1 Dùng Struct Thay Class Khi Phù Hợp

```csharp
// ❌ Class: cấp phát trên heap, GC phải dọn
public class Point2D
{
    public double X { get; set; }
    public double Y { get; set; }
}

// ✅ Struct: sống trên stack, không cần GC
public readonly struct Point2D
{
    public double X { get; init; }
    public double Y { get; init; }
}

// Khi nào dùng struct:
// ✅ Nhỏ (< 16-32 bytes guideline, thực tế tùy context)
// ✅ Immutable (readonly struct)
// ✅ Không cần identity (hai struct bằng nhau nếu field bằng nhau)
// ✅ Không cần inheritance
// ❌ Không nên: mutable struct qua nhiều method (copy bug)
```

### 5.2 StringBuilder Cho String Concatenation

```csharp
// ❌ Tạo nhiều string trung gian
string result = string.Empty;
foreach (var item in items)
    result += item + ", "; // mỗi += tạo string mới!

// ✅ StringBuilder — tái sử dụng buffer
var sb = new StringBuilder(capacity: items.Count * 20); // ước lượng size
foreach (var item in items)
    sb.Append(item).Append(", ");
string result = sb.ToString();

// ✅ Còn tốt hơn: string.Join
string result = string.Join(", ", items);

// ✅ .NET 6+ interpolated string handler (zero-allocation cho logging)
logger.LogInformation($"Processing {count} items"); // không cấp phát nếu log level tắt
```

### 5.3 Tránh LINQ Allocation Không Cần Thiết

```csharp
var numbers = new int[] { 1, 2, 3, 4, 5 };

// ❌ Nhiều allocation: mỗi LINQ operator tạo enumerator object
var result = numbers
    .Where(x => x > 2)
    .Select(x => x * 2)
    .ToList(); // ToList() thêm 1 allocation

// ✅ Dùng trực tiếp nếu biết kích thước
var result = new List<int>(capacity: numbers.Length);
foreach (var n in numbers)
    if (n > 2) result.Add(n * 2);

// ✅ Với .NET 8+: dùng Span + stackalloc cho array nhỏ
Span<int> result = stackalloc int[5];
int count = 0;
foreach (var n in numbers)
    if (n > 2) result[count++] = n * 2;
```

### 5.4 Object Reuse với IDisposable

```csharp
// ❌ Tạo HttpClient mới mỗi lần: socket exhaustion + GC pressure
using var client = new HttpClient(); // đừng làm thế này trong loop!

// ✅ IHttpClientFactory tái sử dụng connection
public class MyService
{
    private readonly HttpClient _client;
    
    public MyService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient("myApi");
    }
}

// ✅ Tái sử dụng StringBuilder nặng
[ThreadStatic]
private static StringBuilder? _cachedBuilder;

private static StringBuilder GetBuilder()
{
    var sb = _cachedBuilder ??= new StringBuilder();
    sb.Clear();
    return sb;
}
```

---

## 6. GC API — Kiểm Soát GC Lập Trình

```csharp
// Thông tin GC hiện tại
Console.WriteLine($"Gen 0 count: {GC.CollectionCount(0)}");
Console.WriteLine($"Gen 1 count: {GC.CollectionCount(1)}");
Console.WriteLine($"Gen 2 count: {GC.CollectionCount(2)}");
Console.WriteLine($"Total memory: {GC.GetTotalMemory(forceFullCollection: false):N0} bytes");
Console.WriteLine($"Total allocated: {GC.GetTotalAllocatedBytes():N0} bytes");

// GCMemoryInfo — thông tin chi tiết
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Heap size: {info.HeapSizeBytes:N0}");
Console.WriteLine($"Fragmented bytes: {info.FragmentedBytes:N0}");
Console.WriteLine($"Memory load: {info.MemoryLoadBytes:N0}");

// Ép GC collect (HIẾM KHI NÊN DÙNG — chỉ cho testing/benchmarking)
GC.Collect(generation: 2, GCCollectionMode.Forced, blocking: true);
GC.WaitForPendingFinalizers();

// Suppress finalization — khi IDisposable đã dọn
public class MyResource : IDisposable
{
    private bool _disposed;
    
    public void Dispose()
    {
        if (_disposed) return;
        // dọn resource
        _disposed = true;
        GC.SuppressFinalize(this); // không cần finalizer nữa
    }
    
    ~MyResource() // finalizer — chỉ là safety net
    {
        Dispose();
    }
}
```

---

## 7. Phân Tích GC — Đọc Metrics

### dotnet-counters — Theo Dõi Trực Tiếp

```bash
# Cài
dotnet tool install -g dotnet-counters

# Theo dõi GC metrics của process
dotnet-counters monitor --process-id <PID> --counters System.Runtime

# Output mẫu:
# [System.Runtime]
#   GC Heap Size (MB)                                64
#   Gen 0 GC Count (Count / 1 sec)                    5   ← nhiều = OK (Gen 0 rẻ)
#   Gen 1 GC Count (Count / 1 sec)                    0
#   Gen 2 GC Count (Count / 1 sec)                    0   ← nhiều = VẤN ĐỀ
#   GC Committed Bytes (MB)                          128
#   LOH Allocated Bytes (Count / 1 sec)          102400   ← LOH allocation = chú ý
#   Allocation Rate (B / 1 sec)                 5000000
```

### EventSource — Programmatic Monitoring

```csharp
// Theo dõi GC events trong code
using var listener = new GCEventListener();

public class GCEventListener : EventListener
{
    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name == "Microsoft-Windows-DotNETRuntime")
        {
            EnableEvents(eventSource, EventLevel.Informational, 
                (EventKeywords)0x1); // GC keyword
        }
    }
    
    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        if (eventData.EventName == "GCStart_V2")
        {
            var gen = (int)eventData.Payload![1]!;
            var reason = eventData.Payload[3]!.ToString();
            Console.WriteLine($"GC Gen {gen} started. Reason: {reason}");
        }
    }
}
```

---

## 8. Memory Leak Patterns — Các Mẫu Rò Rỉ Bộ Nhớ

### 8.1 Event Handler Không Unsubscribe

```csharp
// ❌ Memory leak: publisher giữ reference đến subscriber
public class Publisher
{
    public event EventHandler? DataReady;
}

public class Subscriber
{
    public Subscriber(Publisher publisher)
    {
        publisher.DataReady += OnDataReady; // subscribe
        // Nếu không unsubscribe, Publisher giữ Subscriber sống mãi
    }
    
    private void OnDataReady(object? sender, EventArgs e) { }
}

// ✅ Giải pháp: implement IDisposable
public class Subscriber : IDisposable
{
    private readonly Publisher _publisher;
    
    public Subscriber(Publisher publisher)
    {
        _publisher = publisher;
        _publisher.DataReady += OnDataReady;
    }
    
    public void Dispose()
    {
        _publisher.DataReady -= OnDataReady; // unsubscribe!
    }
}

// ✅ Tốt hơn: dùng WeakReference hoặc WeakEventManager
```

### 8.2 Static Collections Phình To

```csharp
// ❌ Static list accumulates mãi mãi
public static class RequestLog
{
    private static readonly List<string> _log = new(); // NGUY HIỂM
    
    public static void Add(string entry)
    {
        _log.Add(entry); // không bao giờ được dọn!
    }
}

// ✅ Dùng size-bounded collection
public static class RequestLog
{
    private static readonly ConcurrentQueue<string> _log = new();
    private const int MaxEntries = 1000;
    
    public static void Add(string entry)
    {
        _log.Enqueue(entry);
        while (_log.Count > MaxEntries)
            _log.TryDequeue(out _); // bỏ entry cũ nhất
    }
}
```

### 8.3 MemoryCache Không Có Expiry

```csharp
// ❌ Cache không có expiry → tích lũy vô hạn
_cache.Set("key", value); // không bao giờ hết hạn!

// ✅ Luôn đặt expiry và size
var options = new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
    SlidingExpiration = TimeSpan.FromMinutes(2),
    Size = 1 // cần khai báo khi cache có SizeLimit
};
_cache.Set("key", value, options);
```

---

## 9. Finalization — Xử Lý Finalizer

```csharp
// Finalizer tốn kém: object có finalizer sống thêm ít nhất 1 GC cycle
// và chuyển lên Gen 1 trước khi được thu gom

// ❌ Finalizer không cần thiết
public class MyClass
{
    ~MyClass() { } // finalizer trống → tốn kém vô lý!
}

// ✅ Dispose Pattern đúng chuẩn
public class ManagedResource : IDisposable
{
    private IntPtr _nativeHandle; // unmanaged resource
    private bool _disposed = false;
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Giải phóng managed resources
                // (gọi Dispose() trên IDisposable khác)
            }
            
            // Giải phóng unmanaged resources
            if (_nativeHandle != IntPtr.Zero)
            {
                NativeLibrary.Free(_nativeHandle);
                _nativeHandle = IntPtr.Zero;
            }
            
            _disposed = true;
        }
    }
    
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // tối ưu: bỏ qua finalizer
    }
    
    ~ManagedResource() // safety net khi quên Dispose()
    {
        Dispose(disposing: false);
    }
}
```

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa Gen 0, Gen 1, Gen 2 trong GC?**

> - **Gen 0:** đối tượng mới nhất, collection nhanh và thường xuyên (~vài trăm lần/giây). Phần lớn object chết ở đây (ephemeral objects).
> - **Gen 1:** buffer giữa Gen 0 và Gen 2, object sống qua ít nhất 1 lần Gen 0 collection.
> - **Gen 2:** đối tượng long-lived, collection hiếm nhưng đắt (quét toàn bộ heap, có thể gây pause vài giây).

**Q: LOH là gì và tại sao nó vấn đề?**

> LOH — Large Object Heap — chứa object ≥ 85,000 bytes. Vấn đề: LOH **không được compact** mặc định, dẫn đến phân mảnh (fragmentation) theo thời gian. Giải pháp: dùng `ArrayPool<T>` để tái sử dụng array lớn.

**Q: Khi nào nên gọi `GC.Collect()`?**

> Hầu như **không bao giờ** trong production code. Gọi GC.Collect() phá vỡ generational hypothesis, gây Gen 2 collection không cần thiết. Chỉ dùng trong: tests/benchmarks, sau khi unload large data một lần (có justification rõ ràng).

**Q: `IDisposable` vs Finalizer — dùng khi nào?**

> - **`IDisposable`**: luôn implement khi có unmanaged resources hoặc dùng IDisposable khác. Caller phải gọi `Dispose()` (hoặc `using`).
> - **Finalizer (~ClassName)**: chỉ là safety net cho unmanaged resources khi caller quên Dispose. Tốn kém, luôn gọi `GC.SuppressFinalize(this)` trong Dispose().

---

## ✅ Checklist

- [ ] Giải thích được vì sao Gen 2 collection đắt hơn Gen 0
- [ ] Biết LOH threshold (85,000 bytes) và hệ quả của không compact
- [ ] Implement Dispose Pattern đúng chuẩn với `GC.SuppressFinalize`
- [ ] Nhận biết memory leak qua event handler không unsubscribe
- [ ] Dùng `GC.GetTotalAllocatedBytes()` để đo allocation trong test
- [ ] Biết khi nào nên dùng struct thay class

---

**Xem Tiếp:** [2-span-and-memory.md](2-span-and-memory.md) — `Span<T>` và zero-allocation programming
