# 2 — CLR và Quản Lý Bộ Nhớ

> CLR (Common Language Runtime — Môi Trường Chạy Ngôn Ngữ Chung) là trái tim của .NET. Hiểu cách nó quản lý bộ nhớ giúp bạn viết code hiệu quả và tránh memory leak (rò rỉ bộ nhớ).

---

## 📌 CLR — Common Language Runtime

**CLR** là máy ảo thực thi code .NET. Khi bạn build một project C#:

```
Code C# (.cs)
    ↓ Biên dịch (Roslyn compiler)
IL Code — Intermediate Language (Ngôn ngữ trung gian) (.dll / .exe)
    ↓ Chạy lần đầu (JIT — Just-In-Time Compiler)
Native Machine Code (Mã máy thực thi)
    ↓
CPU thực thi
```

### Các trách nhiệm chính của CLR

| Chức năng | Mô tả |
|-----------|-------|
| **JIT Compilation** — Biên dịch kịp thời | Chuyển IL sang mã máy khi chạy |
| **Memory Management** — Quản lý bộ nhớ | Phân bổ và thu hồi bộ nhớ qua GC |
| **Type Safety** — An toàn kiểu | Đảm bảo không truy cập bộ nhớ trái phép |
| **Exception Handling** — Xử lý ngoại lệ | Cơ chế try/catch xuyên suốt |
| **Thread Management** — Quản lý luồng | Thread pool, synchronization |
| **Security** — Bảo mật | Code Access Security (đã giảm vai trò trong .NET hiện đại) |

### AOT — Ahead-Of-Time Compilation (Biên dịch Trước Khi Chạy)

.NET 7+ hỗ trợ **Native AOT**: biên dịch thẳng sang mã máy, không cần JIT.

```
Ưu điểm AOT: khởi động nhanh hơn, bộ nhớ nhỏ hơn, phù hợp serverless/cloud
Nhược điểm:  không hỗ trợ một số tính năng dynamic, build lâu hơn
```

---

## 1. Stack và Heap — Hai Vùng Bộ Nhớ Chính

### Stack — Ngăn Xếp

- **LIFO** (Last In First Out — Vào Sau Ra Trước) — cấu trúc chồng đĩa
- Lưu: **local variables** (biến cục bộ) kiểu giá trị, địa chỉ trả về, tham số hàm
- Phân bổ và giải phóng **cực nhanh** (chỉ di chuyển con trỏ stack)
- **Kích thước giới hạn** (~1MB mặc định mỗi thread) → `StackOverflowException` nếu tràn
- Tự động giải phóng khi method (phương thức) kết thúc

```csharp
void Method()
{
    int x = 10;         // x trên Stack
    double y = 3.14;    // y trên Stack
    // Khi Method() kết thúc: x và y tự động bị xóa
}
```

### Heap — Đống (Heap)

- Vùng nhớ lớn, tổ chức phức tạp hơn
- Lưu: tất cả **reference type objects** (đối tượng kiểu tham chiếu) — class, array, string
- Phân bổ chậm hơn Stack (cần tìm vùng trống, cập nhật GC metadata)
- **GC** quản lý thu hồi bộ nhớ không còn được dùng

```csharp
void Method()
{
    var person = new Person { Name = "Alice" }; // Person object trên Heap
    // person (biến) trên Stack, trỏ đến Heap
}
// person bị xóa khỏi Stack
// Person object trên Heap → GC sẽ thu hồi sau
```

### Sơ đồ Stack vs Heap

```
STACK                    HEAP
┌──────────────┐         ┌────────────────────────────┐
│ person (ref) │────────►│ Person { Name = "Alice" }  │
│ x = 10       │         │                            │
│ y = 3.14     │         │ string "Alice" (interned)  │
└──────────────┘         └────────────────────────────┘
  Tự động giải phóng        GC quản lý thu hồi
```

---

## 2. GC — Garbage Collector (Bộ Thu Gom Rác)

GC tự động thu hồi bộ nhớ của các đối tượng không còn được tham chiếu. Bạn **không cần** (và thường **không nên**) gọi thủ công.

### Nguyên tắc hoạt động

GC sử dụng thuật toán **Mark and Sweep** (đánh dấu và quét):

1. **Mark** — Đánh dấu: bắt đầu từ các **GC Roots** (biến tĩnh, biến stack, xử lý native), duyệt qua tất cả tham chiếu, đánh dấu object còn dùng
2. **Sweep/Collect** — Quét/Thu hồi: mọi object không được đánh dấu → bộ nhớ được giải phóng
3. **Compact** — Nén: di chuyển các object còn sống về một đầu Heap để tránh phân mảnh

### Generational GC — GC Theo Thế Hệ

.NET GC chia Heap thành 3 thế hệ (generations) dựa trên quan sát: **hầu hết objects sống ngắn**.

```
Generation 0 (Gen 0) — Thế Hệ 0
├── Mới phân bổ, sống ngắn
├── Thu hồi thường xuyên nhất, nhanh nhất
└── Kích thước nhỏ (~256KB–2MB)

Generation 1 (Gen 1) — Thế Hệ 1
├── Objects sống sót qua 1 lần GC Gen 0
├── Buffer giữa Gen 0 và Gen 2
└── Thu hồi ít hơn Gen 0

Generation 2 (Gen 2) — Thế Hệ 2
├── Objects sống lâu (singleton, cache, static data)
├── Thu hồi ít nhất, tốn kém nhất (full GC — toàn bộ GC)
└── Kích thước lớn, không giới hạn
```

### LOH — Large Object Heap (Heap Đối Tượng Lớn)

```
Objects ≥ 85,000 bytes → LOH
├── Không được compact mặc định (di chuyển tốn kém)
├── Thu hồi cùng lúc với Gen 2
└── Dễ gây phân mảnh bộ nhớ — cần dùng ArrayPool<T>
```

### Vòng đời của một Object

```csharp
void Example()
{
    // 1. Phân bổ: vào Gen 0
    var obj = new MyClass();

    // 2. Nếu sống qua GC lần 1: thăng lên Gen 1
    // 3. Nếu sống qua GC lần 2: thăng lên Gen 2
    // 4. Khi không còn tham chiếu: bị thu hồi ở lần GC tiếp theo
}
```

---

## 3. IDisposable và `using` — Quản Lý Tài Nguyên Không Được Quản Lý

GC chỉ quản lý **managed resources** (tài nguyên được quản lý — bộ nhớ .NET). Các **unmanaged resources** (tài nguyên không được quản lý) như file handle, database connection, network socket cần giải phóng thủ công.

### Pattern IDisposable

```csharp
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;
    private bool _disposed = false;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // Ngăn GC gọi finalizer — không cần nữa
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _connection?.Dispose(); // Giải phóng managed resources
            }
            // Giải phóng unmanaged resources ở đây nếu có
            _disposed = true;
        }
    }
}
```

### `using` Statement — Tự Động Gọi Dispose

```csharp
// Cú pháp using block cũ
using (var conn = new DatabaseConnection(connStr))
{
    // Dùng conn
} // Dispose() được gọi tự động, kể cả khi có exception

// Cú pháp using declaration mới (C# 8+) — gọn hơn
using var conn = new DatabaseConnection(connStr);
// conn.Dispose() được gọi khi ra khỏi scope (phạm vi)
```

### Finalizer — Phương thức Hủy (Dự Phòng)

```csharp
public class ResourceHolder
{
    ~ResourceHolder() // Finalizer (destructor syntax)
    {
        // Chỉ dùng làm backup khi lập trình viên quên gọi Dispose
        // GC gọi tự động — không đảm bảo thời điểm
        // Làm chậm GC: objects có finalizer cần thêm 1 cycle GC
    }
}
```

**Quy tắc:** Nếu implement `IDisposable`, luôn dùng `using` khi tạo instance.

---

## 4. Memory Leak — Rò Rỉ Bộ Nhớ Trong .NET

.NET có GC nhưng vẫn có thể bị memory leak nếu giữ tham chiếu không cần thiết.

### Nguyên nhân phổ biến

#### Event handlers không unsubscribe (không hủy đăng ký sự kiện)

```csharp
public class Publisher
{
    public event EventHandler DataChanged;
}

public class Subscriber
{
    private Publisher _publisher;

    public Subscriber(Publisher publisher)
    {
        _publisher = publisher;
        _publisher.DataChanged += OnDataChanged; // Đăng ký sự kiện
    }

    private void OnDataChanged(object sender, EventArgs e) { }

    // ❌ Nếu không unsubscribe, Publisher giữ tham chiếu đến Subscriber
    // Subscriber không bao giờ được GC thu hồi dù không dùng nữa
}

// ✅ Fix: implement IDisposable và unsubscribe
public void Dispose()
{
    _publisher.DataChanged -= OnDataChanged;
}
```

#### Static collections giữ references

```csharp
// ❌ Cache tĩnh không có cơ chế xóa
public static class Cache
{
    private static Dictionary<string, object> _cache = new();

    public static void Add(string key, object value)
    {
        _cache[key] = value; // Objects không bao giờ được giải phóng
    }
}

// ✅ Dùng WeakReference hoặc MemoryCache với expiry
```

---

## 5. `GC.Collect()` — Khi Nào Gọi Thủ Công?

```csharp
// Gọi thủ công — hiếm khi nên làm
GC.Collect();                           // Full GC tất cả generations
GC.Collect(0);                          // Chỉ Gen 0
GC.Collect(2, GCCollectionMode.Forced); // Ép buộc full GC

GC.WaitForPendingFinalizers(); // Chờ tất cả finalizer chạy xong
```

**Khi nào hợp lý để gọi `GC.Collect()`:**

1. Sau khi load dữ liệu lớn xong (ví dụ: giải nén file lớn)
2. Trước khi đo benchmark để đảm bảo clean state
3. Trong unit test để kiểm tra memory leak

**Không nên gọi trong production code thông thường** — làm gián đoạn hiệu năng.

---

## 6. GC Modes — Chế Độ GC

.NET hỗ trợ nhiều chế độ GC cho các tình huống khác nhau:

| Chế Độ | Mô Tả | Phù Hợp |
|--------|-------|---------|
| **Workstation GC** | Tối ưu độ trễ thấp | Desktop apps |
| **Server GC** | Nhiều thread GC, throughput cao | Web servers, ASP.NET Core |
| **Background GC** | Mặc định, GC chạy song song với app | Hầu hết trường hợp |

Cấu hình trong `runtimeconfig.json`:
```json
{
  "configProperties": {
    "System.GC.Server": true,
    "System.GC.Concurrent": true
  }
}
```

---

## 7. Công Cụ Phân Tích Bộ Nhớ

| Công Cụ | Mục Đích |
|---------|---------|
| **dotnet-trace** | Thu thập trace (dấu vết) để phân tích |
| **dotnet-dump** | Tạo và phân tích memory dump (ảnh chụp bộ nhớ) |
| **PerfView** | Phân tích GC events, allocations |
| **JetBrains dotMemory** | Profiler bộ nhớ GUI trực quan |
| **Visual Studio Diagnostic Tools** | Tích hợp sẵn trong VS |

```bash
# Lấy memory dump của process đang chạy
dotnet-dump collect --process-id <PID>

# Phân tích dump
dotnet-dump analyze core_20231015_123456
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: CLR là gì? Khác gì JVM của Java?**  
A: CLR là môi trường chạy của .NET: biên dịch JIT, quản lý bộ nhớ, exception handling. Tương tự JVM nhưng hỗ trợ nhiều ngôn ngữ (C#, F#, VB.NET). Từ .NET Core, CLR cross-platform (đa nền tảng).

**Q: Giải thích GC generations (thế hệ GC)?**  
A: Gen 0 cho objects mới (thu hồi nhanh, thường xuyên), Gen 1 là buffer trung gian, Gen 2 cho objects sống lâu (full GC, tốn kém nhất). Phân loại này tối ưu dựa trên quan sát "hầu hết objects sống ngắn".

**Q: `IDisposable` dùng để làm gì?**  
A: Giải phóng unmanaged resources (file, database connection, network socket) mà GC không tự quản lý được. Luôn dùng `using` để đảm bảo Dispose() được gọi kể cả khi có exception.

**Q: Có thể có memory leak trong .NET không?**  
A: Có. Nguyên nhân phổ biến: event handlers không unsubscribe, static collections giữ references, unmanaged resources không được Dispose. GC chỉ thu hồi objects không còn tham chiếu.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Giải thích được Stack vs Heap và khi nào object ở đâu
- [ ] Mô tả Gen 0, Gen 1, Gen 2 và LOH
- [ ] Viết đúng IDisposable pattern
- [ ] Dùng `using` statement đúng cách
- [ ] Liệt kê 3 nguyên nhân gây memory leak trong .NET
- [ ] Biết khi nào (không) nên gọi `GC.Collect()`

---

**Tiếp theo:** [3-collections.md](./3-collections.md) — Collections và Generic
