# Profiling Tools — Công Cụ Phân Tích Hiệu Năng .NET

> Profiling — Phân Tích Hiệu Năng: tìm đúng điểm nghẽn trước khi tối ưu.

---

## 1. Tổng Quan Công Cụ

```
Khi nào dùng gì:
─────────────────────────────────────────────────────────────────
"App chậm nhưng không biết ở đâu"
    → dotnet-counters (live metrics) → xác định vùng vấn đề
    → dotnet-trace (CPU flame graph) → tìm function chậm

"App dùng nhiều memory"
    → dotnet-gcdump (GC heap snapshot) → xem object nào chiếm nhiều nhất
    → dotnet-dump (full dump) → phân tích chi tiết

"Trên Windows, cần phân tích sâu ETW events"
    → PerfView → mọi thứ (complex nhưng powerful)

"Muốn GUI dễ dùng, sẵn sàng trả tiền"
    → JetBrains dotMemory / dotTrace
    → Visual Studio Diagnostic Tools (miễn phí với VS)
```

---

## 2. dotnet-counters — Theo Dõi Metrics Trực Tiếp

`dotnet-counters` cho phép theo dõi performance counters realtime — không cần restart app.

### Cài Đặt và Sử Dụng

```bash
# Cài dotnet-counters
dotnet tool install -g dotnet-counters

# Liệt kê process .NET đang chạy
dotnet-counters ps

# Theo dõi process theo PID
dotnet-counters monitor --process-id 12345

# Theo dõi process theo tên
dotnet-counters monitor --name MyWebApi

# Chỉ theo dõi counters cụ thể
dotnet-counters monitor --process-id 12345 \
    --counters System.Runtime,Microsoft.AspNetCore.Hosting

# Ghi ra file (CSV)
dotnet-counters collect --process-id 12345 --output metrics.csv
```

### Đọc Kết Quả

```
[System.Runtime]
  % Time in GC (since last GC)                           2.1    ← GC overhead
  Allocation Rate (B / 1 sec)                      1234567      ← bytes/sec cấp phát
  CPU Usage (%)                                         45.2
  Exception Count (Count / 1 sec)                          0
  GC Committed Bytes (MB)                               128
  GC Fragmentation (%)                                    3.2
  GC Heap Size (MB)                                      64      ← tổng heap
  Gen 0 GC Count (Count / 1 sec)                          8      ← bình thường
  Gen 1 GC Count (Count / 1 sec)                          1
  Gen 2 GC Count (Count / 1 sec)                          0      ← tốt!
  LOH Allocated Bytes (Count / 1 sec)                102400      ← chú ý nếu cao
  Number of Active Timers                                  3
  ThreadPool Completed Work Item Count (Count / 1 sec)   500
  ThreadPool Queue Length                                  0      ← tốt, không đợi
  ThreadPool Thread Count                                 10
  Working Set (MB)                                      256

[Microsoft.AspNetCore.Hosting]
  Current Requests                                        5
  Failed Requests                                         0
  Request Rate (Count / 1 sec)                         150      ← requests/sec
  Total Requests                                      15000
```

**Dấu Hiệu Cần Chú Ý:**
- `% Time in GC` > 10% → GC pressure cao
- `Gen 2 GC Count` > 1/phút → long-lived objects bị promote không cần thiết
- `LOH Allocated Bytes` tăng liên tục → LOH leak
- `ThreadPool Queue Length` > 0 liên tục → CPU bound hoặc async issue
- `Exception Count` > 0 → xem lại exception handling

---

## 3. dotnet-trace — CPU và Event Tracing

`dotnet-trace` thu thập .NET runtime events và CPU sampling — tạo trace file để phân tích.

### Thu Thập Trace

```bash
# Cài
dotnet tool install -g dotnet-trace

# Thu thập trace trong 30 giây
dotnet-trace collect --process-id 12345 --duration 00:00:30

# Trace với CPU profiling (nhiều info hơn)
dotnet-trace collect \
    --process-id 12345 \
    --providers Microsoft-DotNETCore-SampleProfiler \
    --duration 00:00:30

# Trace cho performance investigation
dotnet-trace collect \
    --process-id 12345 \
    --profile cpu-sampling \
    --duration 00:00:30 \
    --output trace.nettrace

# Trace GC events chi tiết
dotnet-trace collect \
    --process-id 12345 \
    --providers "Microsoft-Windows-DotNETRuntime:0x1:4" \
    --duration 00:00:30
```

### Phân Tích Trace File

```bash
# Chuyển đổi sang SpeedScope format để xem Flame Graph online
dotnet-trace convert trace.nettrace --format Speedscope
# → trace.speedscope.json

# Upload lên https://www.speedscope.app/
# Hoặc mở bằng PerfView

# Chuyển sang Chromium format (Chrome DevTools)
dotnet-trace convert trace.nettrace --format Chromium
# Mở Chrome → chrome://tracing → load file
```

### Đọc Flame Graph

```
Flame Graph — Biểu Đồ Lửa
───────────────────────────────────────────────────────────────
  ┌──────────────────────────────────────────────────────────┐
  │ Program.Main                                              │  ← root (đỉnh)
  │  ┌────────────────────────────────────────────────────┐  │
  │  │ Controller.Handle                                  │  │
  │  │  ┌────────────────────────┐  ┌──────────────────┐  │  │
  │  │  │ ProductService.GetAll  │  │ OrderService.Get  │  │  │
  │  │  │  ┌──────────────────┐  │  │                  │  │  │
  │  │  │  │ DbContext.Query   │  │  └──────────────────┘  │  │
  │  │  │  │ (RỘNG = CHẬM!)   │  │                        │  │  ← bottleneck!
  │  │  │  └──────────────────┘  │                        │  │
  │  │  └────────────────────────┘                        │  │
  └──────────────────────────────────────────────────────────┘

Đọc: rộng = nhiều CPU time → điểm nghẽn thực sự
      hẹp = ít CPU time → không phải vấn đề
```

---

## 4. dotnet-dump — Phân Tích Heap Dump

`dotnet-dump` tạo và phân tích memory dump — dùng khi app dùng quá nhiều memory.

### Thu Thập Dump

```bash
# Cài
dotnet tool install -g dotnet-dump

# Tạo dump (process vẫn chạy)
dotnet-dump collect --process-id 12345

# Tạo full dump (lớn hơn, nhiều info hơn)
dotnet-dump collect --process-id 12345 --type Full

# Trigger dump khi memory vượt ngưỡng (production)
# (dùng procdump.exe trên Windows)
```

### Phân Tích Dump Với LLDB-like Commands

```bash
# Mở dump để phân tích
dotnet-dump analyze core_20260602_120000.dmp

# Xem tổng quan heap
> dumpheap -stat

# Output:
# MT             Count    TotalSize Class Name
# 00007f...     102400   102400000 System.Byte[]    ← 100MB byte arrays!
# 00007f...      50000     8000000 System.String
# 00007f...      10000     5000000 MyApp.Product

# Tìm tất cả instance của type cụ thể
> dumpheap -type MyApp.Product

# MT             Count    TotalSize Class Name
# 00007f...      10000     5000000 MyApp.Product

# Xem chi tiết 1 object
> dumpobj 00007f123456  

# Tìm GC roots của object (tại sao nó không được GC?)
> gcroot 00007f123456

# Xem finalizer queue (pending finalizers)
> finalizequeue

# Xem thread stacks
> threads
> clrstack

# Exit
> exit
```

---

## 5. dotnet-gcdump — GC Heap Snapshot

`dotnet-gcdump` tạo snapshot nhẹ hơn full dump — chỉ chụp GC heap, nhanh hơn nhiều:

```bash
# Cài
dotnet tool install -g dotnet-gcdump

# Tạo GC dump
dotnet-gcdump collect --process-id 12345

# Phân tích bằng Visual Studio, dotnet-heapview, hoặc PerfView
# Mở file .gcdump bằng Visual Studio → Object Graph View

# Cũng có thể convert và phân tích với dotnet-dump analyze
```

### Phân Tích Với Visual Studio

```
Visual Studio → File → Open → .gcdump file
→ Heap Usage view:
   ┌─────────────────────────────────────────────┐
   │ Type                  Count   TotalBytes  % │
   │ System.Byte[]         1000    50,000,000 45 │ ← nhiều nhất
   │ System.String         5000    10,000,000  9 │
   │ MyApp.CacheEntry       500     5,000,000  5 │
   └─────────────────────────────────────────────┘
→ Click vào type → xem GC roots → tìm nguyên nhân
```

---

## 6. PerfView — Công Cụ Phân Tích Toàn Diện (Windows)

PerfView là công cụ Microsoft, miễn phí, mạnh mẽ nhất cho Windows .NET analysis.

### Download và Chạy

```
Download: https://github.com/microsoft/perfview/releases
Không cần cài đặt — chạy trực tiếp PerfView.exe
Cần Administrator để thu thập kernel events
```

### Thu Thập Data

```
PerfView → Collect → Collect...
  ✅ CPU Stacks
  ✅ .NET Alloc (allocation stacks — EXPENSIVE, dùng ngắn thôi)
  ✅ .NET GC Heap
  Duration: 30 seconds
→ Start Collection
→ Reproduce vấn đề
→ Stop Collection
```

### Phân Tích CPU

```
PerfView.etl.zip → CPU Stacks
  → Filter by process
  → Grouping: Method name / Module
  
  Flame Graph view:
  - Rộng = hot path
  - "BROKEN" = JIT không thể optimize
  - "!" = GC sampling (có thể skew results)
```

### Phân Tích GC

```
PerfView.etl.zip → GC Events → GCStats
  → Xem per-generation pause times
  → LOH events
  → Finalization queue

GCStats table:
  Gen  Count  PauseTime(ms)  Reason
  0    1200   0.5 avg        AllocSmall
  1     120   2.3 avg        AllocSmall
  2       5   150.2 avg      AllocLarge ← GEN 2 VÌ LOH!
```

---

## 7. JetBrains dotMemory — GUI Memory Profiler

dotMemory là commercial tool, có giao diện đẹp, dễ hiểu hơn PerfView.

### Workflow Tiêu Biểu

```
1. Profiler → Run → Attach to Process → MyWebApi
2. Gửi requests vào app (load generator)
3. Take Snapshot 1
4. Tiếp tục gửi requests
5. Take Snapshot 2
6. Compare snapshots → xem object nào tăng

Compare view:
  ┌──────────────────────────────────────────────────────────┐
  │ Class               +Count    +Bytes    Survived Bytes   │
  │ MyApp.Session       +10000  +1,000,000    500,000       │ ← session leak!
  │ System.String       +5000    +500,000     200,000       │
  └──────────────────────────────────────────────────────────┘
  → Click MyApp.Session → xem retention path
  → tìm nguyên nhân tại sao session không được GC
```

---

## 8. Visual Studio Diagnostic Tools — Miễn Phí

Tích hợp trong Visual Studio, không cần tools riêng:

```
Debug → Performance Profiler (Alt+F2)
  ✅ CPU Usage
  ✅ Memory Usage
  ✅ .NET Async
  ✅ .NET Counters

Hoặc trong Debug session:
  Debug → Windows → Diagnostic Tools
  → Theo dõi memory và CPU realtime trong debug session
```

---

## 9. Application Insights — Production Monitoring

Trong production, dùng APM — Application Performance Monitoring — Giám Sát Hiệu Năng Ứng Dụng:

```csharp
// Cài Application Insights
dotnet add package Microsoft.ApplicationInsights.AspNetCore

// Program.cs
builder.Services.AddApplicationInsightsTelemetry();
// Trong appsettings.json:
// "ApplicationInsights": { "InstrumentationKey": "..." }

// Custom metrics
public class ProductService
{
    private readonly TelemetryClient _telemetry;
    
    public async Task<Product?> GetProductAsync(int id)
    {
        using var op = _telemetry.StartOperation<RequestTelemetry>("GetProduct");
        op.Telemetry.Properties["productId"] = id.ToString();
        
        var sw = Stopwatch.StartNew();
        try
        {
            var result = await _repo.GetByIdAsync(id);
            _telemetry.TrackMetric("ProductQuery.Duration", sw.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            op.Telemetry.Success = false;
            throw;
        }
    }
}
```

---

## 10. OpenTelemetry — Standard Observability

```csharp
// Cài packages
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MyApp")
        .AddOtlpExporter()) // xuất sang Jaeger/Zipkin/Tempo
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter()); // xuất sang Prometheus

// Custom spans
private static readonly ActivitySource _source = new("MyApp");

public async Task ProcessOrderAsync(Order order)
{
    using var activity = _source.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", order.Id);
    activity?.SetTag("order.amount", order.Total);
    
    // xử lý...
    
    activity?.SetStatus(ActivityStatusCode.Ok);
}
```

---

## 11. Workflow Điều Tra Hiệu Năng — Investigation Playbook

```
Step 1: Xác định triệu chứng
─────────────────────────────
  □ P99 latency tăng?         → CPU hoặc I/O bound
  □ Memory tăng không dừng?   → memory leak
  □ CPU cao không giải thích? → hot loop hoặc spin wait
  □ Gen 2 GC nhiều?           → long-lived objects
  □ ThreadPool queue depth > 0? → thread starvation

Step 2: Thu thập metrics (dotnet-counters)
────────────────────────────────────────────
  □ GC metrics (gen counts, heap size, allocation rate)
  □ ThreadPool (queue depth, thread count)
  □ HTTP metrics (request rate, errors)
  □ Exception count

Step 3: Thu thập trace (dotnet-trace)
──────────────────────────────────────
  □ 30-second trace trong lúc reproduce
  □ Mở SpeedScope → tìm hot functions

Step 4: Thu thập heap dump nếu cần (dotnet-gcdump)
────────────────────────────────────────────────────
  □ Snapshot trước
  □ Reproduce vài phút
  □ Snapshot sau
  □ So sánh → tìm class nào tăng

Step 5: Fix và đo lại
──────────────────────
  □ Fix điểm nghẽn
  □ Chạy BenchmarkDotNet trước/sau
  □ Monitor metrics sau deploy
```

---

## 12. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Bạn sẽ làm gì khi phát hiện .NET app đang dùng quá nhiều memory?**

> Quy trình: (1) `dotnet-counters` để xem GC heap size và allocation rate, xác định liệu là leak hay overuse; (2) `dotnet-gcdump` để lấy snapshot, so sánh 2 snapshot cách nhau vài phút; (3) Phân tích type nào tăng nhiều nhất; (4) Tìm GC roots của những object đó để hiểu tại sao chúng không được GC; (5) Fix: remove static reference, unsubscribe event, implement IDisposable đúng.

**Q: Sự khác nhau giữa `dotnet-trace` và `dotnet-dump`?**

> - `dotnet-trace`: thu thập events theo thời gian (CPU sampling, GC events, method calls) → tạo trace file để xem timeline, flame graph → dùng để phân tích performance (CPU, latency).
> - `dotnet-dump`: chụp snapshot toàn bộ memory tại một thời điểm → dùng để phân tích memory (object count, retention paths, memory leak).

**Q: Flame graph cho thấy điều gì?**

> Mỗi thanh ngang là 1 function call. Chiều rộng = % CPU time. Xếp chồng = call stack. Đọc từ dưới lên: root → callee. Frame rộng nhất = hot path — điểm cần tối ưu. Frames ở đỉnh (leaf) mà rộng = nơi CPU thực sự làm việc.

---

## ✅ Checklist

- [ ] Biết cách cài và chạy `dotnet-counters` để xem live metrics
- [ ] Biết cách thu thập trace với `dotnet-trace` và đọc SpeedScope
- [ ] Biết cách tạo GC dump với `dotnet-gcdump`
- [ ] Biết các metrics quan trọng: GC gen counts, heap size, allocation rate
- [ ] Hiểu flame graph: rộng = hot path = cần tối ưu
- [ ] Có workflow điều tra rõ ràng: triệu chứng → metrics → trace → fix

---

**Xem Lại Section:** [README.md](README.md) — Tổng quan 07-performance  
**Section Tiếp Theo:** [08-security/](../08-security/) — Bảo mật: JWT, OAuth2, Authorization
