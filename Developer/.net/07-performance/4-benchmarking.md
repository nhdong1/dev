# Benchmarking — Đo Lường Hiệu Năng với BenchmarkDotNet

> "Measure, don't guess." — Đo đạc, đừng đoán mò.

---

## 1. Tại Sao Phải Đo Lường?

```
❌ Sai lầm phổ biến:
────────────────────────────────────────────────────────
"Tôi nghĩ for loop nhanh hơn LINQ, nên tôi không dùng LINQ"
"Struct chắc nhanh hơn class, dùng struct hết đi"
"Caching sẽ nhanh hơn, thêm cache vào đi"

Hậu quả: code phức tạp hơn, khó đọc hơn, nhưng
         đôi khi KHÔNG thực sự nhanh hơn (JIT tối ưu tốt lắm!)

✅ Đúng đắn:
────────────────────────────────────────────────────────
1. Phát hiện vấn đề thực sự (profiler, metrics, APM)
2. Tái tạo vấn đề trong benchmark
3. Thử nghiệm giải pháp
4. Đo → xác nhận cải thiện
5. Mới merge vào production
```

---

## 2. Cài Đặt BenchmarkDotNet

```bash
# Thêm vào project benchmark riêng
dotnet add package BenchmarkDotNet

# hoặc template
dotnet new console -n MyBenchmarks
cd MyBenchmarks
dotnet add package BenchmarkDotNet
```

```csharp
// Program.cs
using BenchmarkDotNet.Running;

BenchmarkRunner.Run<MyBenchmarks>();

// QUAN TRỌNG: chạy ở Release mode!
// dotnet run -c Release
// Không bao giờ benchmark ở Debug mode!
```

---

## 3. Cấu Trúc Benchmark Cơ Bản

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]          // hiển thị GC alloc, Gen0/1/2
[SimpleJob(RuntimeMoniker.Net80)] // chạy trên .NET 8
public class StringBenchmarks
{
    private const int N = 1000;
    private string[] _data = null!;
    
    // GlobalSetup: chạy 1 lần trước tất cả benchmarks
    [GlobalSetup]
    public void Setup()
    {
        _data = Enumerable.Range(0, N)
            .Select(i => $"item_{i}")
            .ToArray();
    }
    
    // Baseline — dùng làm mốc so sánh (Ratio column)
    [Benchmark(Baseline = true)]
    public string StringConcat()
    {
        string result = string.Empty;
        foreach (var s in _data)
            result += s;
        return result;
    }
    
    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var s in _data)
            sb.Append(s);
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringJoin()
    {
        return string.Join("", _data);
    }
}
```

### Chạy và Đọc Kết Quả

```
$ dotnet run -c Release

BenchmarkDotNet v0.13.x
.NET 8.0.x

| Method        | Mean       | Ratio | Gen0    | Gen1   | Allocated  |
|-------------- |------------|-------|---------|--------|------------|
| StringConcat  | 1,234.5 us | 1.00  | 234.375 | 3.9063 | 1,918 KB   |
| StringBuilder |   125.3 us | 0.10  |  63.477 |      - |   520 KB   |
| StringJoin    |    98.7 us | 0.08  |  31.250 |      - |   257 KB   |

Giải thích cột:
- Mean: thời gian trung bình (ns/us/ms)
- Ratio: so với Baseline (1.00 = bằng, 0.10 = 10x nhanh hơn)
- Gen0/Gen1/Gen2: số lần GC collection per 1000 ops
- Allocated: tổng bytes cấp phát per operation
```

---

## 4. Attributes Quan Trọng

### Job Attributes — Cấu Hình Môi Trường Chạy

```csharp
// Chạy trên nhiều runtime
[SimpleJob(RuntimeMoniker.Net80)]
[SimpleJob(RuntimeMoniker.Net90)]
public class MultiRuntimeBenchmark { }

// Điều chỉnh iteration count
[SimpleJob(launchCount: 1, warmupCount: 3, iterationCount: 10)]
public class QuickBenchmark { }

// Chạy trong process riêng biệt (mặc định)
[SimpleJob(RunStrategy.ColdStart, launchCount: 5)]
public class ColdStartBenchmark { }
```

### Parameter Benchmarks — Chạy Với Nhiều Tham Số

```csharp
public class SortBenchmarks
{
    // Chạy với từng giá trị N
    [Params(100, 1_000, 10_000, 100_000)]
    public int N;
    
    private int[] _data = null!;
    
    [GlobalSetup]
    public void Setup()
    {
        var random = new Random(42);
        _data = Enumerable.Range(0, N).Select(_ => random.Next()).ToArray();
    }
    
    [Benchmark]
    public void ArraySort()
    {
        var copy = (int[])_data.Clone();
        Array.Sort(copy);
    }
    
    [Benchmark]
    public void LinqOrderBy()
    {
        var sorted = _data.OrderBy(x => x).ToArray();
    }
}
```

### IterationSetup — Reset Giữa Các Iteration

```csharp
public class MutatingBenchmarks
{
    private List<int> _list = null!;
    
    // Chạy trước MỖI iteration (không tính vào thời gian benchmark)
    [IterationSetup]
    public void IterSetup() => _list = new List<int>(Enumerable.Range(0, 1000));
    
    [Benchmark]
    public void SortInPlace() => _list.Sort();
    
    [Benchmark]
    public void LinqSort() => _list = _list.OrderBy(x => x).ToList();
}
```

### Diagnosers — Công Cụ Phân Tích Bổ Sung

```csharp
[MemoryDiagnoser]       // GC stats, allocations (LUÔN DÙNG)
[ThreadingDiagnoser]    // threads, contentions
[ExceptionDiagnoser]    // exceptions thrown
[EtwProfiler]           // Windows: ETW profiling (cần elevated)
[NativeMemoryProfiler]  // native memory tracking
public class DiagnoserDemo { }
```

---

## 5. Anti-Patterns — Benchmark Sai Dẫn Đến Kết Quả Sai

### 5.1 Benchmark Ở Debug Mode

```
❌ Debug mode: JIT không optimize, kết quả không có ý nghĩa
✅ LUÔN chạy: dotnet run -c Release
```

### 5.2 Dead Code Elimination — JIT Loại Bỏ Code

```csharp
// ❌ JIT có thể eliminate toàn bộ nếu kết quả không được dùng
[Benchmark]
public void BadBenchmark()
{
    int result = HeavyComputation(); // JIT biết result không dùng → xóa luôn!
}

// ✅ Trả về kết quả để JIT không eliminate
[Benchmark]
public int GoodBenchmark()
{
    return HeavyComputation(); // BenchmarkDotNet đọc return value
}

// ✅ Hoặc dùng Consume helper
private readonly Consumer _consumer = new Consumer();

[Benchmark]
public void ConsumeResult()
{
    _consumer.Consume(HeavyComputation()); // đảm bảo không bị eliminate
}
```

### 5.3 Benchmark Quá Ngắn

```csharp
// ❌ Quá ngắn: overhead benchmark lớn hơn operation
[Benchmark]
public int AddInts() => 1 + 2; // 1-2 ns — noise lớn hơn signal

// ✅ Thêm nhiều lần lặp vào benchmark
[Benchmark]
public int AddManyInts()
{
    int sum = 0;
    for (int i = 0; i < 1000; i++) sum += i; // 1000 ops per call
    return sum;
}
// hoặc dùng [OperationsPerInvoke]
[Benchmark]
[OperationsPerInvoke(1000)] // nói cho BDN biết có 1000 ops trong 1 invocation
public void BulkOperation()
{
    for (int i = 0; i < 1000; i++) DoOneOp(i);
}
```

### 5.4 Shared State Giữa Benchmarks

```csharp
// ❌ Các benchmark ảnh hưởng lẫn nhau qua shared state
private List<int> _sharedList = new();

[Benchmark]
public void AddItems() => _sharedList.Add(1); // list ngày càng to!

[Benchmark]
public void SearchItems() => _sharedList.Contains(1); // context khác nhau mỗi lần

// ✅ Dùng GlobalSetup để khởi tạo, IterationSetup để reset
[IterationSetup]
public void ResetList() => _sharedList = new List<int>(1000);
```

---

## 6. So Sánh Thực Tế — Real-World Benchmarks

### 6.1 LINQ vs For Loop

```csharp
[MemoryDiagnoser]
public class IterationBenchmarks
{
    private int[] _data = Enumerable.Range(0, 10_000).ToArray();
    
    [Benchmark(Baseline = true)]
    public int Linq_Sum()
        => _data.Where(x => x % 2 == 0).Sum();
    
    [Benchmark]
    public int ForLoop_Sum()
    {
        int sum = 0;
        foreach (var x in _data)
            if (x % 2 == 0) sum += x;
        return sum;
    }
    
    [Benchmark]
    public int Span_Sum()
    {
        ReadOnlySpan<int> span = _data;
        int sum = 0;
        foreach (var x in span)
            if (x % 2 == 0) sum += x;
        return sum;
    }
}
// Kết quả điển hình:
// ForLoop ~= Span (gần như nhau sau JIT optimization)
// LINQ: chậm hơn ~20%, nhưng allocates iterator objects
// → Đối với hot path: for loop; còn lại: LINQ cho readability
```

### 6.2 Dictionary vs Switch

```csharp
public class LookupBenchmarks
{
    private static readonly Dictionary<string, int> _dict = new()
    {
        ["low"] = 1, ["medium"] = 2, ["high"] = 3, ["critical"] = 4
    };
    
    private readonly string[] _inputs = { "low", "medium", "high", "critical" };
    private readonly Random _rng = new(42);
    
    [Benchmark(Baseline = true)]
    public int DictionaryLookup()
    {
        string key = _inputs[_rng.Next(4)];
        return _dict.TryGetValue(key, out int val) ? val : 0;
    }
    
    [Benchmark]
    public int SwitchExpression()
    {
        string key = _inputs[_rng.Next(4)];
        return key switch
        {
            "low"      => 1,
            "medium"   => 2,
            "high"     => 3,
            "critical" => 4,
            _          => 0
        };
    }
}
// Switch thường nhanh hơn Dictionary cho ít cases (< 10)
// Dictionary tốt hơn khi có nhiều cases hoặc runtime-configurable
```

### 6.3 Allocation Benchmark

```csharp
[MemoryDiagnoser]
public class AllocationBenchmarks
{
    [Benchmark(Baseline = true)]
    public byte[] AllocateArray() => new byte[4096];
    
    [Benchmark]
    public byte[] ArrayPoolRent()
    {
        var arr = ArrayPool<byte>.Shared.Rent(4096);
        ArrayPool<byte>.Shared.Return(arr);
        return arr; // không thực sự dùng sau return, chỉ để đo
    }
    
    [Benchmark]
    public void StackAlloc()
    {
        Span<byte> span = stackalloc byte[256]; // nhỏ thôi
        span[0] = 1; // đảm bảo không bị eliminate
    }
}
```

---

## 7. Exporter — Xuất Kết Quả

```csharp
// Xuất sang nhiều định dạng
[HtmlExporter]        // file HTML đẹp
[MarkdownExporter]    // GitHub markdown
[CsvExporter]         // CSV cho Excel
[JsonExporter]        // JSON cho tooling
[RPlotExporter]       // biểu đồ R (cần R cài sẵn)
public class ExportedBenchmarks { }

// File kết quả ở: BenchmarkDotNet.Artifacts/results/
```

---

## 8. Micro-Benchmark vs Macro-Benchmark

```
Micro-Benchmark — Đo lường nhỏ
────────────────────────────────
  Đo: 1 function, 1 algorithm cụ thể
  Công cụ: BenchmarkDotNet
  Dùng khi: so sánh 2 implementation cụ thể
  Cẩn thận: JIT warm-up, dead code elimination, cache effects

Macro-Benchmark — Đo lường lớn
─────────────────────────────────
  Đo: end-to-end performance (API throughput, latency)
  Công cụ: k6, NBomber, Apache JMeter
  Dùng khi: kiểm tra performance dưới tải thực tế
  Cẩn thận: môi trường test, data realistic
```

---

## 9. Tích Hợp Với CI/CD

```csharp
// Chạy benchmark trong CI để detect regression
// Dùng BenchmarkDotNet.Artifacts để so sánh kết quả

// csproj: benchmark project riêng
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net8.0</TargetFramework>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  <Optimize>true</Optimize>
</PropertyGroup>

// GitHub Actions:
// - name: Run benchmarks
//   run: dotnet run -c Release --project benchmarks/ -- --exporters json
// - name: Compare with baseline
//   uses: benchmark-action/github-action-benchmark@v1
```

---

## 10. Quick Profiling Không Cần BenchmarkDotNet

```csharp
// Đo thời gian nhanh với Stopwatch
var sw = Stopwatch.StartNew();
DoHeavyWork();
sw.Stop();
Console.WriteLine($"Elapsed: {sw.ElapsedMilliseconds}ms");

// Đo allocation nhanh
long before = GC.GetAllocatedBytesForCurrentThread();
DoSomething();
long after = GC.GetAllocatedBytesForCurrentThread();
Console.WriteLine($"Allocated: {after - before} bytes");

// ActivitySource cho distributed tracing
using var activity = ActivitySource.StartActivity("my-operation");
activity?.SetTag("input.size", data.Length);
DoWork(data);
// → đo được trong OpenTelemetry / Application Insights
```

---

## 11. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao phải chạy benchmark ở Release mode?**

> Debug mode vô hiệu hóa nhiều JIT optimization: inlining, dead code elimination, loop unrolling, SIMD. Kết quả benchmark Debug mode có thể khác Release mode 5-10x và không phản ánh thực tế production.

**Q: Dead code elimination là gì và tại sao ảnh hưởng benchmark?**

> JIT có thể nhận ra rằng kết quả của một computation không được sử dụng → xóa toàn bộ computation đó. Ví dụ: `void Benchmark() { int x = HeavyCalc(); }` — JIT thấy `x` không được đọc → bỏ `HeavyCalc()`. Giải pháp: trả về kết quả hoặc dùng `Consumer.Consume()`.

**Q: Khi nào LINQ chậm hơn for loop đáng kể?**

> LINQ chậm hơn khi: (1) allocate nhiều intermediate objects (Where+Select+ToList), (2) hot path với triệu lần gọi/giây, (3) dùng trên `Span<T>` (LINQ không hỗ trợ). Trong hầu hết trường hợp không phải hot path, LINQ readability > performance trade-off.

---

## ✅ Checklist

- [ ] Luôn benchmark ở Release mode (`dotnet run -c Release`)
- [ ] Dùng `[MemoryDiagnoser]` để thấy allocation
- [ ] Trả về giá trị từ benchmark để tránh dead code elimination
- [ ] Dùng `[Params]` để test với nhiều kích thước dữ liệu
- [ ] So sánh với `[Benchmark(Baseline = true)]`
- [ ] Không benchmark code quá ngắn (< 10ns) trực tiếp

---

**Xem Tiếp:** [5-caching.md](5-caching.md) — Caching với `IMemoryCache`, `IDistributedCache`, Redis
