# Span\<T\> và Memory\<T\> — Zero-Allocation Programming

> `Span<T>` là một trong những tính năng hiệu năng quan trọng nhất trong .NET hiện đại — xử lý dữ liệu mà không cấp phát thêm bộ nhớ.

---

## 1. Vấn Đề Cần Giải Quyết

```csharp
// ❌ Tình huống truyền thống: cần xử lý substring → tạo string mới
string data = "2026-06-02";
string year  = data.Substring(0, 4);  // allocation: "2026"
string month = data.Substring(5, 2);  // allocation: "06"
string day   = data.Substring(8, 2);  // allocation: "02"
// 3 string objects mới trên heap!

// ✅ Với Span<T>: không allocation nào cả
ReadOnlySpan<char> span = data.AsSpan();
ReadOnlySpan<char> yearSpan  = span.Slice(0, 4);  // chỉ là con trỏ + length
ReadOnlySpan<char> monthSpan = span.Slice(5, 2);  // không copy dữ liệu
ReadOnlySpan<char> daySpan   = span.Slice(8, 2);  // zero allocation!
```

---

## 2. Span\<T\> — Khái Niệm Cơ Bản

### Định Nghĩa

`Span<T>` là **ref struct** — kiểu tham chiếu chỉ tồn tại trên stack — đại diện cho một đoạn bộ nhớ **contiguous** — liên tục.

```
Span<T> = { pointer: IntPtr, length: int }
           ↓
     trỏ vào vùng bộ nhớ bất kỳ:
     ┌─────────────────────────────────┐
     │  Stack memory (stackalloc)      │
     │  Heap memory (array, string)    │
     │  Native memory (unmanaged)      │
     └─────────────────────────────────┘
```

### Tạo Span\<T\>

```csharp
// Từ array
int[] array = { 1, 2, 3, 4, 5 };
Span<int> span1 = array;                    // toàn bộ array
Span<int> span2 = array.AsSpan();           // giống trên
Span<int> span3 = array.AsSpan(1, 3);      // {2, 3, 4}
Span<int> span4 = new Span<int>(array, 2, 2); // {3, 4}

// Từ stackalloc — cấp phát trên stack, không có GC!
Span<int> stackSpan = stackalloc int[8];   // 32 bytes trên stack

// Từ string (ReadOnlySpan)
string s = "Hello World";
ReadOnlySpan<char> strSpan = s.AsSpan();
ReadOnlySpan<char> wordSpan = s.AsSpan(6, 5); // "World"

// Từ single value
int value = 42;
Span<int> singleSpan = MemoryMarshal.CreateSpan(ref value, 1);
```

### Đọc và Ghi

```csharp
Span<int> span = stackalloc int[5];

// Ghi
span[0] = 10;
span[1] = 20;

// Điền giá trị
span.Fill(0);           // điền tất cả = 0
span.Clear();           // giống Fill(default)

// Copy
int[] source = { 1, 2, 3, 4, 5 };
int[] dest = new int[5];
source.AsSpan().CopyTo(dest); // copy không allocation

// Slice — lấy đoạn con
Span<int> slice = span.Slice(start: 1, length: 3);

// Tìm kiếm
int idx = span.IndexOf(20);  // trả về index hoặc -1

// So sánh
bool equal = span.SequenceEqual(other);
```

---

## 3. ReadOnlySpan\<T\> — Span Chỉ Đọc

```csharp
// ReadOnlySpan<T>: không thể ghi vào, an toàn hơn
ReadOnlySpan<char> text = "Hello, World!".AsSpan();

// Xử lý string không allocation
bool startsWithHello = text.StartsWith("Hello".AsSpan());
int commaIdx = text.IndexOf(',');
ReadOnlySpan<char> greeting = text.Slice(0, commaIdx);

// Parsing số không allocation
ReadOnlySpan<char> numText = "12345".AsSpan();
int result = int.Parse(numText); // API Span-aware không tạo string mới

// Split không allocation (.NET 8+)
ReadOnlySpan<char> csv = "a,b,c,d".AsSpan();
foreach (var range in csv.Split(','))
{
    ReadOnlySpan<char> token = csv[range];
    Console.WriteLine(token.ToString()); // chỉ tạo string khi thực sự cần
}
```

---

## 4. Memory\<T\> — Phiên Bản Async-Safe

### Tại Sao Cần Memory\<T\>?

`Span<T>` là **ref struct** → **không thể** dùng trong:
- `async` methods (stack thay đổi qua await)
- Fields của class (chỉ tồn tại trên stack)
- Lambdas và closures
- `IEnumerable<T>` và các interface thông thường

`Memory<T>` giải quyết vấn đề này:

```csharp
// ❌ Span<T> không hoạt động trong async
async Task ProcessAsync(Span<byte> data) // COMPILE ERROR!
{
    await SomeOperationAsync();
    Process(data); // stack đã thay đổi qua await!
}

// ✅ Memory<T> hoạt động trong async
async Task ProcessAsync(Memory<byte> memory)
{
    await SomeOperationAsync();
    Span<byte> span = memory.Span; // lấy Span chỉ khi cần (synchronous)
    Process(span);
}
```

### So Sánh Span\<T\> vs Memory\<T\>

| Tính Năng | `Span<T>` | `Memory<T>` |
|-----------|-----------|-------------|
| Loại | `ref struct` | `struct` |
| Stack only | ✅ | ❌ (có thể dùng field) |
| Async methods | ❌ | ✅ |
| Class fields | ❌ | ✅ |
| Hiệu năng | Cao nhất | Cao (một chút overhead) |
| Lấy Span | Trực tiếp | `.Span` property |

```csharp
// Memory<T> API
byte[] buffer = new byte[1024];
Memory<byte> memory = buffer;

// Chuyển về Span để xử lý synchronous
Span<byte> span = memory.Span;

// Slice
Memory<byte> first512 = memory.Slice(0, 512);
Memory<byte> last512  = memory.Slice(512);

// Pin memory (interop với native code)
using MemoryHandle handle = memory.Pin();
unsafe
{
    void* ptr = handle.Pointer;
    // dùng ptr với native API
}
```

---

## 5. IMemoryOwner\<T\> — Quản Lý Vòng Đời

```csharp
// IMemoryOwner<T>: kết hợp Memory<T> với IDisposable
// Dùng khi cần thuê bộ nhớ và trả lại sau

using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> memory = owner.Memory;

// Xử lý
await FillBufferAsync(memory);
await SendDataAsync(memory.Slice(0, bytesWritten));

// Khi Dispose: bộ nhớ được trả về pool
```

---

## 6. stackalloc — Cấp Phát Trên Stack

```csharp
// stackalloc: cấp phát array trên stack, zero GC overhead
// Giới hạn: không được quá lớn (stack overflow), chỉ trong unsafe trước .NET 7.2

// .NET Core 2.1+ với Span<T>: dùng stackalloc an toàn (không cần unsafe)
Span<byte> buffer = stackalloc byte[256]; // OK, 256 bytes trên stack

// Pattern phổ biến: stackalloc cho buffer nhỏ, heap cho buffer lớn
static void ProcessData(int size)
{
    const int StackThreshold = 512;
    
    // stackalloc nếu nhỏ, ArrayPool nếu lớn
    byte[]? rentedArray = null;
    Span<byte> buffer = size <= StackThreshold
        ? stackalloc byte[StackThreshold]
        : (rentedArray = ArrayPool<byte>.Shared.Rent(size));
    
    try
    {
        buffer = buffer.Slice(0, size);
        DoWork(buffer);
    }
    finally
    {
        if (rentedArray is not null)
            ArrayPool<byte>.Shared.Return(rentedArray);
    }
}
```

---

## 7. Ví Dụ Thực Tế — Practical Examples

### 7.1 Parse CSV Không Allocation

```csharp
// ❌ Truyền thống: nhiều string allocation
static IEnumerable<(string Name, int Age)> ParseCsvOld(string line)
{
    var parts = line.Split(','); // 1 array + n strings
    return parts.Select(p =>
    {
        var kv = p.Split(':'); // n arrays + 2n strings
        return (kv[0], int.Parse(kv[1]));
    });
}

// ✅ Với Span: zero allocation
static void ParseCsvNew(ReadOnlySpan<char> line)
{
    while (!line.IsEmpty)
    {
        int commaIdx = line.IndexOf(',');
        ReadOnlySpan<char> token = commaIdx >= 0
            ? line.Slice(0, commaIdx)
            : line;

        int colonIdx = token.IndexOf(':');
        if (colonIdx > 0)
        {
            ReadOnlySpan<char> name = token.Slice(0, colonIdx);
            ReadOnlySpan<char> ageSpan = token.Slice(colonIdx + 1);
            int age = int.Parse(ageSpan);
            Console.WriteLine($"Name: {name}, Age: {age}");
        }

        line = commaIdx >= 0 ? line.Slice(commaIdx + 1) : default;
    }
}
```

### 7.2 Đọc Binary Protocol Không Allocation

```csharp
// Đọc header của network packet mà không cấp phát
static PacketHeader ReadHeader(ReadOnlySpan<byte> data)
{
    // Dùng MemoryMarshal để đọc struct trực tiếp từ bytes
    if (data.Length < PacketHeader.Size)
        throw new ArgumentException("Data too short");
    
    // BinaryPrimitives: đọc số không allocation
    int magic   = BinaryPrimitives.ReadInt32BigEndian(data.Slice(0, 4));
    short version = BinaryPrimitives.ReadInt16BigEndian(data.Slice(4, 2));
    int length  = BinaryPrimitives.ReadInt32BigEndian(data.Slice(6, 4));
    
    return new PacketHeader(magic, version, length);
}

// Ghi header vào buffer
static void WriteHeader(Span<byte> buffer, PacketHeader header)
{
    BinaryPrimitives.WriteInt32BigEndian(buffer.Slice(0, 4), header.Magic);
    BinaryPrimitives.WriteInt16BigEndian(buffer.Slice(4, 2), header.Version);
    BinaryPrimitives.WriteInt32BigEndian(buffer.Slice(6, 4), header.Length);
}
```

### 7.3 String Processing Hiệu Quả

```csharp
// Trim và lowercase không tạo intermediate string
static bool IsValidEmail(ReadOnlySpan<char> input)
{
    // Trim whitespace không allocation
    ReadOnlySpan<char> trimmed = input.Trim();
    
    if (trimmed.IsEmpty) return false;
    
    int atIdx = trimmed.IndexOf('@');
    if (atIdx <= 0) return false;
    
    ReadOnlySpan<char> local = trimmed.Slice(0, atIdx);
    ReadOnlySpan<char> domain = trimmed.Slice(atIdx + 1);
    
    return !domain.IsEmpty && domain.Contains('.'); // không tạo string!
}

// Xử lý text file line-by-line không allocation
static async Task ProcessLinesAsync(Stream stream)
{
    using var reader = new StreamReader(stream);
    
    while (await reader.ReadLineAsync() is string line)
    {
        // ReadOnlySpan từ string không copy
        ReadOnlySpan<char> span = line.AsSpan().Trim();
        
        if (span.StartsWith("//".AsSpan()))
            continue; // comment line, skip
        
        // xử lý span...
    }
}
```

### 7.4 MemoryMarshal — Truy Cập Bộ Nhớ Nâng Cao

```csharp
// MemoryMarshal: cast và reinterpret bytes
byte[] data = { 0x01, 0x00, 0x00, 0x00 }; // little-endian int = 1

// Cast span bytes thành span int (zero copy!)
ReadOnlySpan<int> ints = MemoryMarshal.Cast<byte, int>(data.AsSpan());
Console.WriteLine(ints[0]); // 1

// Đọc struct trực tiếp từ bytes
Span<byte> rawBytes = stackalloc byte[8];
rawBytes[0] = 42; rawBytes[4] = 10;
ref MyStruct s = ref MemoryMarshal.AsRef<MyStruct>(rawBytes);
Console.WriteLine(s.Field1); // 42

[StructLayout(LayoutKind.Sequential)]
struct MyStruct { public int Field1; public int Field2; }
```

---

## 8. Giới Hạn Của Span\<T\>

```
ref struct Restrictions — Ràng Buộc của ref struct:
─────────────────────────────────────────────────────

❌ Không thể là field của class thông thường:
   class MyClass { Span<int> _data; } // COMPILE ERROR

❌ Không thể dùng trong async methods:
   async Task Foo(Span<int> s) { await ...; } // COMPILE ERROR

❌ Không thể dùng trong lambdas capture:
   Span<int> s = ...; Action a = () => s[0]; // COMPILE ERROR

❌ Không thể là type parameter của IEnumerable<T>:
   IEnumerable<Span<int>> // không hợp lệ

✅ Giải pháp: dùng Memory<T> cho những trường hợp này
```

---

## 9. System.IO.Pipelines — Pipeline Không Allocation

`System.IO.Pipelines` là API cấp cao hơn xây dựng trên `Memory<T>`:

```csharp
// Đọc dữ liệu mạng hiệu quả với Pipe
var pipe = new Pipe();

// Writer side (network reader)
async Task FillPipeAsync(Socket socket, PipeWriter writer)
{
    while (true)
    {
        // Lấy buffer từ pipe (không allocation mới)
        Memory<byte> memory = writer.GetMemory(512);
        
        int bytesRead = await socket.ReceiveAsync(memory, SocketFlags.None);
        if (bytesRead == 0) break;
        
        writer.Advance(bytesRead); // báo đã ghi bao nhiêu
        
        FlushResult result = await writer.FlushAsync();
        if (result.IsCompleted) break;
    }
    await writer.CompleteAsync();
}

// Reader side (protocol parser)
async Task ReadPipeAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // Parse protocol từ buffer
        while (TryParseMessage(ref buffer, out Message message))
        {
            ProcessMessage(message);
        }
        
        reader.AdvanceTo(buffer.Start, buffer.End);
        
        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}
```

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: `Span<T>` khác `Memory<T>` như thế nào?**

> - `Span<T>` là `ref struct` — chỉ tồn tại trên stack, không thể dùng trong `async`, fields, lambdas. Hiệu năng cao nhất.
> - `Memory<T>` là `struct` thông thường — có thể dùng trong `async`, fields, lambdas. Có một chút overhead so với `Span<T>`.

**Q: Khi nào dùng `Span<T>` thay `string`?**

> Khi cần xử lý substrings, parsing, text manipulation không muốn tạo string mới (allocation). Ví dụ: parse HTTP request headers, đọc CSV, protocol parsing. Trade-off: phức tạp hơn, không thể serialize trực tiếp.

**Q: `stackalloc` có nguy hiểm không?**

> Với `Span<T>`, `stackalloc` **an toàn** (không cần `unsafe`). Giới hạn: không cấp phát quá lớn (thường < 1KB là ổn) vì stack có giới hạn. Dùng pattern kiểm tra kích thước: nếu nhỏ dùng `stackalloc`, nếu lớn dùng `ArrayPool<byte>`.

**Q: Tại sao `Span<T>` không thể dùng trong `async` methods?**

> `async` method được compiler compile thành state machine — một class. State machine lưu trữ local variables như fields của class. Vì `Span<T>` là `ref struct`, nó không thể là field của class → compile error.

---

## ✅ Checklist

- [ ] Dùng `AsSpan()` và `Slice()` để xử lý string không allocation
- [ ] Phân biệt khi nào dùng `Span<T>` vs `Memory<T>`
- [ ] Dùng `stackalloc` với Span cho buffer tạm nhỏ
- [ ] Parse số với `int.Parse(ReadOnlySpan<char>)` thay vì `int.Parse(string)`
- [ ] Dùng `BinaryPrimitives` để đọc/ghi binary data không allocation
- [ ] Biết giới hạn của `ref struct` và cách circumvent

---

**Xem Tiếp:** [3-pooling-strategies.md](3-pooling-strategies.md) — `ArrayPool<T>`, `MemoryPool<T>`, `ObjectPool<T>`
