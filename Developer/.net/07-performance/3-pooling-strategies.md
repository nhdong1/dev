# Pooling Strategies — Chiến Lược Tái Sử Dụng Bộ Nhớ

> Pooling — Gộp chung tài nguyên: thay vì cấp phát và giải phóng liên tục, hãy "thuê" và "trả lại" từ một pool chung.

---

## 1. Tại Sao Cần Pooling?

```
Vấn đề: New/GC cycle lặp lại liên tục
──────────────────────────────────────
Request 1:  new byte[4096] → dùng → GC thu gom → Gen 0 collection
Request 2:  new byte[4096] → dùng → GC thu gom → Gen 0 collection
Request 3:  new byte[4096] → dùng → GC thu gom → Gen 0 collection
...1000 requests/sec = 1000 allocations/sec, 1000 GC pressures/sec

Giải pháp: Pool
───────────────────────────────────────────────────────
             ┌────────────────────────┐
             │       Pool             │
             │  [buf][buf][buf][buf]  │
             └────────────────────────┘
Request 1:  Rent(4096) → dùng → Return()   ← zero GC!
Request 2:  Rent(4096) → dùng → Return()   ← zero GC!
Request 3:  Rent(4096) → dùng → Return()   ← zero GC!
```

**Lợi ích:**
- **Giảm GC pressure** — ít Gen 0 collection, không có Gen 2 collection từ pooled objects
- **Giảm LOH fragmentation** — array lớn không vào LOH khi được pool
- **Tăng throughput** — không cần khởi tạo object mới mỗi lần

---

## 2. ArrayPool\<T\> — Pool Mảng

`ArrayPool<T>` là pool mảng tích hợp sẵn trong .NET, thread-safe, cực kỳ hiệu quả.

### Cách Sử Dụng Cơ Bản

```csharp
// Shared pool — pool mặc định, dùng được ở mọi nơi
ArrayPool<byte> pool = ArrayPool<byte>.Shared;

// Rent — thuê mảng (size >= minLength)
byte[] buffer = pool.Rent(minimumLength: 4096);
// buffer.Length >= 4096 (có thể lớn hơn — pool làm tròn lên)

try
{
    // Dùng buffer
    int bytesRead = await stream.ReadAsync(buffer);
    ProcessData(buffer.AsSpan(0, bytesRead));
}
finally
{
    // Return — TRẢ LẠI BẮT BUỘC, dùng try/finally
    pool.Return(buffer, clearArray: false);
    // clearArray: true → điền 0 trước khi trả (bảo mật, nhưng chậm hơn)
}
```

### Pattern Đúng Với IDisposable

```csharp
// ✅ Tạo wrapper IDisposable để đảm bảo return
public readonly struct RentedArray<T> : IDisposable
{
    private readonly T[] _array;
    private readonly ArrayPool<T> _pool;
    public readonly int Length;
    
    public RentedArray(int minimumLength, ArrayPool<T>? pool = null)
    {
        _pool = pool ?? ArrayPool<T>.Shared;
        _array = _pool.Rent(minimumLength);
        Length = minimumLength;
    }
    
    public T this[int index] => _array[index];
    public Span<T> AsSpan() => _array.AsSpan(0, Length);
    public Memory<T> AsMemory() => _array.AsMemory(0, Length);
    
    public void Dispose() => _pool.Return(_array, clearArray: false);
}

// Sử dụng
using var buffer = new RentedArray<byte>(4096);
await stream.ReadAsync(buffer.AsMemory());
Process(buffer.AsSpan());
// Tự động Return khi ra khỏi using block
```

### ArrayPool Tùy Chỉnh

```csharp
// Tạo pool riêng với giới hạn kích thước
ArrayPool<byte> customPool = ArrayPool<byte>.Create(
    maxArrayLength: 1024 * 1024,  // tối đa 1MB per array
    maxArraysPerBucket: 50        // tối đa 50 array per size bucket
);

// Shared pool vs Custom pool
// Shared: dùng cho mọi trường hợp thông thường
// Custom: khi cần kiểm soát kích thước tối đa hoặc số lượng

// Lưu ý quan trọng về Shared pool limits:
// - Array > 1MB: KHÔNG được pool (cấp phát và GC bình thường)
// - Nếu pool hết array: Rent() tạo mới (không throw)
// - Return() array không từ pool này: undefined behavior (thực tế bị bỏ qua)
```

### So Sánh Hiệu Năng

```csharp
// BenchmarkDotNet kết quả điển hình:
// | Method          | Mean      | Allocated  |
// |---------------- |-----------|------------|
// | NewArray        | 2,450 ns  | 4,096 B    |  ← cấp phát mới
// | ArrayPool_Rent  |    45 ns  |          - |  ← 54x nhanh hơn!
```

---

## 3. MemoryPool\<T\> — Pool Bộ Nhớ

`MemoryPool<T>` cấp phát `IMemoryOwner<T>` — kết hợp `Memory<T>` với `IDisposable`.

```csharp
// MemoryPool trả về IMemoryOwner — quản lý vòng đời tự động
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(minBufferSize: 4096);
Memory<byte> memory = owner.Memory;

// Dùng memory
await stream.ReadAsync(memory);
ProcessData(memory.Span);

// Khi Dispose: tự động trả về pool
```

### MemoryPool vs ArrayPool

| Tính Năng | `ArrayPool<T>` | `MemoryPool<T>` |
|-----------|----------------|-----------------|
| Trả về | `T[]` (array) | `IMemoryOwner<T>` |
| Quản lý vòng đời | Manual (try/finally) | `IDisposable` tự động |
| Async-safe | Cần wrap `Memory<T>` | Có sẵn `Memory<T>` |
| Hiệu năng | Cao hơn một chút | Một chút overhead |
| Trường hợp dùng | Synchronous, đơn giản | Async, cần ownership rõ ràng |

```csharp
// MemoryPool hữu ích khi cần pass ownership sang method khác
async Task<ProcessResult> ProcessWithOwnershipAsync(IMemoryOwner<byte> owner)
{
    using (owner) // caller transfer ownership về cho callee
    {
        await DoHeavyWorkAsync(owner.Memory);
        return new ProcessResult(owner.Memory.Length);
    }
}
```

---

## 4. ObjectPool\<T\> — Pool Đối Tượng Tùy Chỉnh

`ObjectPool<T>` từ `Microsoft.Extensions.ObjectPool` dùng để pool **bất kỳ đối tượng** nào tốn kém khởi tạo.

### Cài Đặt

```bash
dotnet add package Microsoft.Extensions.ObjectPool
```

### Sử Dụng Với StringBuilder

```csharp
// Pool StringBuilder — đặc biệt hữu ích vì StringBuilder.Clear() rẻ
public class StringBuilderPooledObjectPolicy : PooledObjectPolicy<StringBuilder>
{
    public override StringBuilder Create() => new StringBuilder(256);
    
    public override bool Return(StringBuilder sb)
    {
        if (sb.Capacity > 65536) // quá lớn → không pool
            return false;
        
        sb.Clear(); // reset trước khi trả về pool
        return true;
    }
}

// Đăng ký trong DI
services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
services.AddSingleton(sp =>
{
    var provider = sp.GetRequiredService<ObjectPoolProvider>();
    return provider.Create(new StringBuilderPooledObjectPolicy());
});

// Sử dụng
public class TextFormatter
{
    private readonly ObjectPool<StringBuilder> _pool;
    
    public TextFormatter(ObjectPool<StringBuilder> pool) => _pool = pool;
    
    public string Format(IEnumerable<string> items)
    {
        StringBuilder sb = _pool.Get();
        try
        {
            foreach (var item in items)
                sb.Append(item).Append(';');
            return sb.ToString();
        }
        finally
        {
            _pool.Return(sb); // trả về pool
        }
    }
}
```

### Pool Đối Tượng Tùy Chỉnh

```csharp
// Pool cho đối tượng tốn kém khởi tạo (vd: parser, encoder)
public class JsonSerializerPolicy : PooledObjectPolicy<Utf8JsonWriter>
{
    public override Utf8JsonWriter Create()
    {
        // Tốn kém: khởi tạo writer với stream nội bộ
        return new Utf8JsonWriter(new MemoryStream(), new JsonWriterOptions
        {
            Indented = false,
            SkipValidation = true
        });
    }
    
    public override bool Return(Utf8JsonWriter writer)
    {
        writer.Reset(); // reset state
        return true;
    }
}

// Sử dụng DefaultObjectPool trực tiếp
var policy = new DefaultPooledObjectPolicy<MyExpensiveObject>();
var pool = new DefaultObjectPool<MyExpensiveObject>(policy, maximumRetained: 8);

MyExpensiveObject obj = pool.Get();
try
{
    obj.Process(data);
}
finally
{
    pool.Return(obj);
}
```

---

## 5. RecyclableMemoryStream — Pool cho MemoryStream

`Microsoft.IO.RecyclableMemoryStream` là drop-in replacement cho `MemoryStream` với pooling:

```bash
dotnet add package Microsoft.IO.RecyclableMemoryStream
```

```csharp
// RecyclableMemoryStreamManager — singleton, cấu hình 1 lần
private static readonly RecyclableMemoryStreamManager _manager = new(
    new RecyclableMemoryStreamManager.Options
    {
        BlockSize = 4096,          // kích thước block nhỏ
        LargeBufferMultiple = 1024 * 1024, // bội số buffer lớn
        MaximumBufferSize = 128 * 1024 * 1024, // tối đa 128MB
        GenerateCallStacks = false // true để debug memory leak
    }
);

// Dùng như MemoryStream bình thường
using RecyclableMemoryStream stream = _manager.GetStream("tag-for-debugging");

// hoặc với initial data
byte[] data = GetData();
using RecyclableMemoryStream stream2 = _manager.GetStream("deserialize", data, 0, data.Length);

// Serialize JSON vào stream không allocation
using var writer = new Utf8JsonWriter(stream);
JsonSerializer.Serialize(writer, myObject);
byte[] result = stream.ToArray(); // hoặc stream.GetBuffer()
```

**Tại Sao Dùng `RecyclableMemoryStream`?**

```
MemoryStream thông thường:
- Khi dữ liệu vượt capacity → tạo mảng mới (gấp đôi) → copy → GC mảng cũ
- Mảng lớn → LOH → fragmentation

RecyclableMemoryStream:
- Dùng nhiều block nhỏ từ pool → không copy khi grow
- Không vào LOH (block nhỏ)
- Tự động trả về pool khi Dispose
```

---

## 6. Channel\<T\> — Pool Cho Producer-Consumer

```csharp
// Channel<T> có pooling ngầm cho các node trong queue
// Dùng thay cho ConcurrentQueue<T> khi cần async producer-consumer

var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false,
    SingleWriter = false
});

// Producer
await channel.Writer.WriteAsync(new WorkItem { Data = "..." });

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}
```

---

## 7. Anti-Patterns — Những Điều Cần Tránh

### 7.1 Quên Return Về Pool

```csharp
// ❌ Không return → pool cạn dần, GC phải tạo array mới
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
ProcessData(buffer);
// QUÊN Return! → memory leak khái niệm (pool object bị GC)

// ❌ Return nhưng không dùng try/finally
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
DoWork(buffer); // nếu throw exception
ArrayPool<byte>.Shared.Return(buffer); // không chạy!

// ✅ Luôn dùng try/finally
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try { DoWork(buffer); }
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

### 7.2 Dùng Buffer Sau Khi Return

```csharp
// ❌ Nguy hiểm: array được trả về pool nhưng vẫn giữ reference
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
DoWork(buffer);
ArrayPool<byte>.Shared.Return(buffer);

// Pool có thể cho buffer này cho thread khác!
buffer[0] = 99; // race condition / data corruption!
```

### 7.3 Pool Object Mutable Không Reset

```csharp
// ❌ Trả về pool khi object có state bẩn
public class BadPolicy : PooledObjectPolicy<StringBuilder>
{
    public override bool Return(StringBuilder sb)
    {
        return true; // THIẾU sb.Clear() → state bẩn!
    }
}

// ✅ Reset state trước khi trả về
public override bool Return(StringBuilder sb)
{
    if (sb.Capacity > 65536) return false;
    sb.Clear(); // bắt buộc!
    return true;
}
```

### 7.4 Pool Object Quá Lớn

```csharp
// ❌ Pool object dùng ít, nhưng chiếm nhiều memory
var pool = new DefaultObjectPool<HeavyObject>(policy, maximumRetained: 100);
// 100 HeavyObject × 10MB = 1GB RAM ngay cả khi idle!

// ✅ Điều chỉnh maximumRetained phù hợp
var pool = new DefaultObjectPool<HeavyObject>(policy,
    maximumRetained: Environment.ProcessorCount * 2); // hợp lý hơn
```

---

## 8. Khi Nào Dùng Gì

```
Decision Tree — Cây Quyết Định Pooling:
─────────────────────────────────────────

Cần buffer/array tạm?
  ├── Nhỏ (< 512 bytes) và synchronous → stackalloc
  ├── Bất kỳ kích thước, synchronous → ArrayPool<byte>
  └── Bất kỳ kích thước, async → MemoryPool<byte>

Cần pool đối tượng?
  ├── StringBuilder cụ thể → StringBuilderPoolHelper / ObjectPool<StringBuilder>
  ├── Đối tượng custom tốn khởi tạo → ObjectPool<T>
  └── MemoryStream → RecyclableMemoryStream

Không cần pool (cấp phát bình thường):
  ├── Object chỉ tạo 1 lần (singleton, one-time init)
  ├── Object rất nhỏ (< 64 bytes) và tạo ít
  └── Object có lifetime dài (sống suốt vòng đời request)
```

---

## 9. Đo Lường Hiệu Quả

```csharp
// Dùng BenchmarkDotNet để đo
[MemoryDiagnoser] // hiện thị allocation
public class PoolingBenchmarks
{
    [Benchmark(Baseline = true)]
    public void WithNewArray()
    {
        byte[] buffer = new byte[4096];
        FillBuffer(buffer);
    }
    
    [Benchmark]
    public void WithArrayPool()
    {
        byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
        try { FillBuffer(buffer); }
        finally { ArrayPool<byte>.Shared.Return(buffer); }
    }
    
    [Benchmark]
    public void WithStackAlloc()
    {
        Span<byte> buffer = stackalloc byte[256]; // chỉ cho buffer nhỏ
        FillBuffer(buffer);
    }
}

// Kết quả điển hình:
// | Method          | Mean    | Gen0   | Allocated |
// |---------------- |---------|--------|-----------|
// | WithNewArray    | 380 ns  | 0.0610 | 4096 B    |
// | WithArrayPool   |  15 ns  |      - |       -   | ← 25x nhanh, 0 allocation!
// | WithStackAlloc  |   5 ns  |      - |       -   | ← 76x nhanh, 0 allocation!
```

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao không phải lúc nào cũng dùng ArrayPool?**

> ArrayPool có overhead: cần Rent/Return, có thể cần synchronization (thread-safe). Với array nhỏ (< 1KB) dùng ít, overhead đôi khi lớn hơn lợi ích. Quy tắc: dùng khi array ≥ vài KB, được cấp phát thường xuyên (per-request).

**Q: Điều gì xảy ra nếu quên Return về ArrayPool?**

> Array đó không được pool nữa — về mặt logic là "mất tích". GC sẽ thu gom nó bình thường. Pool không bị crash, nhưng mất đi lợi ích pooling. Nếu xảy ra thường xuyên, pool sẽ cạn dần và phải cấp phát mới liên tục.

**Q: `ObjectPool<T>` vs `new T()` — khi nào dùng ObjectPool?**

> Dùng ObjectPool khi: (1) khởi tạo T tốn kém (constructor phức tạp, allocate nhiều), (2) T được tạo và hủy thường xuyên (per-request), (3) T có thể reset về trạng thái ban đầu. Ví dụ điển hình: StringBuilder, parser objects, encoder/decoder.

---

## ✅ Checklist

- [ ] Dùng `ArrayPool<byte>.Shared` cho buffer I/O tạm thời
- [ ] Luôn wrap trong `try/finally` hoặc helper `IDisposable`
- [ ] Không dùng buffer sau khi đã Return về pool
- [ ] Pool `StringBuilder` trong service xử lý text thường xuyên
- [ ] Cân nhắc `RecyclableMemoryStream` cho serialization nặng
- [ ] Đo allocation trước và sau pooling với `[MemoryDiagnoser]`

---

**Xem Tiếp:** [4-benchmarking.md](4-benchmarking.md) — Đo lường hiệu năng với BenchmarkDotNet
