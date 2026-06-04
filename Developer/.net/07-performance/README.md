# 07 — Hiệu Năng & Bộ Nhớ trong .NET

> Tối ưu hiệu năng đúng cách: đo trước, tối ưu sau — không bao giờ đoán mò.

---

## 🎯 Mục Tiêu Của Section Này

Sau khi hoàn thành section này, bạn sẽ có thể:

- Hiểu GC — Garbage Collector — Bộ Thu Gom Rác hoạt động ra sao và cách giảm áp lực lên GC
- Dùng `Span<T>` và `Memory<T>` để xử lý dữ liệu không cấp phát bộ nhớ heap
- Áp dụng các chiến lược pooling — tái sử dụng đối tượng để tránh cấp phát lặp lại
- Đo lường hiệu năng chính xác với BenchmarkDotNet
- Cài đặt caching — bộ nhớ đệm — đúng cách với nhiều tầng
- Dùng công cụ profiling — phân tích hiệu năng để tìm điểm nghẽn thực sự

---

## 📋 Danh Sách Files

| File | Chủ Đề | Độ Khó |
|------|--------|--------|
| [1-memory-management.md](1-memory-management.md) | GC internals, Gen 0/1/2, LOH — Large Object Heap | ⭐⭐⭐ |
| [2-span-and-memory.md](2-span-and-memory.md) | `Span<T>`, `Memory<T>`, `ReadOnlySpan<T>` — zero-allocation | ⭐⭐⭐ |
| [3-pooling-strategies.md](3-pooling-strategies.md) | `ArrayPool<T>`, `MemoryPool<T>`, `ObjectPool<T>` | ⭐⭐ |
| [4-benchmarking.md](4-benchmarking.md) | BenchmarkDotNet — đo lường hiệu năng chính xác | ⭐⭐ |
| [5-caching.md](5-caching.md) | `IMemoryCache`, `IDistributedCache`, Redis, cache-aside | ⭐⭐ |
| [6-profiling-tools.md](6-profiling-tools.md) | dotnet-trace, dotnet-dump, PerfView, dotMemory | ⭐⭐⭐ |

---

## 🧠 Kiến Thức Nền Cần Có

Trước khi học section này, hãy đảm bảo đã nắm:

- [01-fundamentals/2-clr-and-memory.md](../01-fundamentals/2-clr-and-memory.md) — Stack vs Heap, GC cơ bản
- [01-fundamentals/3-collections.md](../01-fundamentals/3-collections.md) — `List<T>`, `Dictionary<K,V>`, `Span<T>`
- [03-async-concurrency/](../03-async-concurrency/) — Async/Await, Thread safety
- [06-testing/6-performance-testing.md](../06-testing/6-performance-testing.md) — BenchmarkDotNet cơ bản

---

## 🗺️ Bản Đồ Tư Duy — Performance Mindset

```
Hiệu Năng .NET
├── Memory — Bộ Nhớ
│   ├── GC Generations — Thế hệ GC (Gen 0 → Gen 1 → Gen 2 → LOH)
│   ├── Stack allocation — Cấp phát trên stack (struct, stackalloc)
│   ├── Heap allocation — Cấp phát trên heap (class, new)
│   └── LOH — Large Object Heap (object > 85KB không được compact)
│
├── Zero-Allocation — Không cấp phát
│   ├── Span<T> — Lát cắt bộ nhớ contiguous (stack, heap, native)
│   ├── Memory<T> — Phiên bản async-safe của Span<T>
│   ├── ReadOnlySpan<T> — Span chỉ đọc, không cấp phát
│   └── stackalloc — Cấp phát mảng trên stack
│
├── Pooling — Tái Sử Dụng
│   ├── ArrayPool<T> — Pool mảng, tránh cấp phát mảng lớn
│   ├── MemoryPool<T> — Pool bộ nhớ dạng IMemoryOwner
│   └── ObjectPool<T> — Pool đối tượng tùy chỉnh
│
├── Caching — Bộ Nhớ Đệm
│   ├── IMemoryCache — Cache trong bộ nhớ tiến trình
│   ├── IDistributedCache — Cache phân tán (Redis, SQL Server)
│   └── ResponseCaching — Cache response HTTP
│
└── Profiling — Phân Tích
    ├── dotnet-trace — Trace CPU, GC, ThreadPool
    ├── dotnet-dump — Phân tích heap dump
    ├── PerfView — Microsoft tool phân tích ETW events
    └── JetBrains dotMemory — GUI-based memory profiler
```

---

## ⚡ Quy Tắc Vàng — Golden Rules

### 1. Đo Trước — Measure First

```
Đừng bao giờ tối ưu mà không đo lường.
"Premature optimization is the root of all evil" — Donald Knuth

Quy trình:
1. Phát hiện vấn đề (user complain / alerting)
2. Tái tạo vấn đề trong môi trường có thể đo
3. Đo bằng BenchmarkDotNet hoặc profiler
4. Tìm điểm nghẽn (bottleneck) thực sự
5. Tối ưu điểm đó
6. Đo lại để xác nhận cải thiện
```

### 2. Hiểu Chi Phí Cấp Phát — Allocation Cost

```csharp
// ❌ Tốn kém: cấp phát string mới mỗi lần
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString(); // 1000 string objects trên heap!

// ✅ Hiệu quả: StringBuilder tái sử dụng buffer
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString(); // chỉ 1 string cuối cùng
```

### 3. Chọn Đúng Kiểu — Value vs Reference

```csharp
// struct — value type — sống trên stack, copy-by-value
// Dùng khi: nhỏ (< 16 bytes), immutable, ngắn hạn
readonly struct Point { public int X; public int Y; }

// class — reference type — sống trên heap, copy-by-reference
// Dùng khi: lớn, mutable, cần identity, dài hạn
class Customer { public string Name; public List<Order> Orders; }

// record struct — value type với value semantics
record struct Rectangle(double Width, double Height);
```

### 4. Tránh Boxing — Avoid Boxing

```csharp
// ❌ Boxing: int → object → GC pressure
object boxed = 42;
int unboxed = (int)boxed;

// ✅ Generics tránh boxing hoàn toàn
List<int> list = new List<int>(); // không có boxing
Dictionary<string, int> dict = new();
```

---

## 📊 Chỉ Số Hiệu Năng Quan Trọng

| Chỉ Số | Mô Tả | Ngưỡng Tốt |
|--------|--------|------------|
| **GC Gen 0 collections/sec** | Tần suất thu gom Gen 0 | < 10/sec trong production |
| **GC Gen 2 collections/sec** | Tần suất thu gom Gen 2 (đắt nhất) | < 1/giờ nếu có thể |
| **LOH allocations** | Cấp phát trên Large Object Heap | Tối thiểu hóa |
| **Allocations/request** | Byte cấp phát mỗi request | < 100KB cho API thông thường |
| **P99 latency** | Độ trễ phần trăm thứ 99 | Tùy SLA, thường < 200ms |
| **ThreadPool queue depth** | Số task chờ trong queue | Luôn gần 0 |

---

## 🛠️ Công Cụ Chính

| Công Cụ | Mục Đích | Cách Cài |
|---------|----------|----------|
| **BenchmarkDotNet** | Micro-benchmark chính xác | `dotnet add package BenchmarkDotNet` |
| **dotnet-trace** | Trace CPU, GC, runtime events | `dotnet tool install -g dotnet-trace` |
| **dotnet-counters** | Live metrics monitoring | `dotnet tool install -g dotnet-counters` |
| **dotnet-dump** | Heap dump analysis | `dotnet tool install -g dotnet-dump` |
| **dotnet-gcdump** | GC heap snapshot | `dotnet tool install -g dotnet-gcdump` |
| **PerfView** | ETW event analysis (Windows) | Download từ GitHub Microsoft |
| **JetBrains dotMemory** | GUI memory profiler | JetBrains Toolbox |

---

## 🔗 Liên Kết Với Các Section Khác

| Vấn Đề | Xem Tại |
|--------|---------|
| GC cơ bản | [01-fundamentals/2-clr-and-memory.md](../01-fundamentals/2-clr-and-memory.md) |
| Async và ThreadPool | [03-async-concurrency/2-task-parallel-library.md](../03-async-concurrency/2-task-parallel-library.md) |
| N+1 trong EF Core | [05-entity-framework/5-n-plus-one-problem.md](../05-entity-framework/5-n-plus-one-problem.md) |
| EF Core performance | [05-entity-framework/7-performance-tips.md](../05-entity-framework/7-performance-tips.md) |
| Performance testing | [06-testing/6-performance-testing.md](../06-testing/6-performance-testing.md) |

---

## ✅ Checklist Hoàn Thành Section

- [ ] Giải thích được GC generations và tại sao Gen 2 collection đắt
- [ ] Dùng được `Span<T>` để tránh substring allocation
- [ ] Áp dụng `ArrayPool<T>` cho mảng tạm thời
- [ ] Viết benchmark với BenchmarkDotNet đúng cách
- [ ] Cài `IMemoryCache` với expiration và size limit
- [ ] Cài `IDistributedCache` với Redis
- [ ] Chạy được `dotnet-trace` và đọc được kết quả
- [ ] Phân tích được GC dump với `dotnet-gcdump`

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Phiên Bản .NET:** .NET 8 / .NET 9
