# Performance Testing — Kiểm Thử Hiệu Năng

> BenchmarkDotNet, NBomber, k6 — đo lường micro-benchmarks, load testing và stress testing cho ứng dụng .NET.

---

## 🎯 Các Loại Kiểm Thử Hiệu Năng

| Loại                              | Mục Đích                                                       | Công Cụ                    |
| --------------------------------- | -------------------------------------------------------------- | -------------------------- |
| **Micro-benchmark** (So sánh vi mô) | Đo hiệu năng của method/algorithm cụ thể                    | BenchmarkDotNet            |
| **Load Test** (Kiểm thử tải)       | Hệ thống hoạt động như thế nào dưới tải bình thường           | k6, NBomber                |
| **Stress Test** (Kiểm thử căng thẳng) | Tìm điểm phá vỡ của hệ thống                             | k6, NBomber                |
| **Spike Test** (Kiểm thử đột biến) | Phản ứng với tải đột ngột tăng cao                            | k6, NBomber                |
| **Soak Test** (Kiểm thử ngâm)      | Tìm memory leak khi chạy tải liên tục dài hạn                 | k6, NBomber                |
| **Profiling** (Phân tích hiệu năng)| Tìm bottleneck trong code                                     | dotTrace, PerfView, VS     |

---

## ⚡ BenchmarkDotNet — Micro-Benchmark Chính Xác

**BenchmarkDotNet** là thư viện chuẩn để đo hiệu năng code .NET với độ chính xác cao — tự động warmup, loại bỏ nhiễu, thống kê chuyên sâu.

### Tại Sao Không Dùng `Stopwatch` Thủ Công?

```csharp
// ❌ BAD: Stopwatch không đáng tin cậy cho benchmarking
var sw = Stopwatch.StartNew();
for (int i = 0; i < 1000; i++) DoWork();
sw.Stop();
Console.WriteLine($"Avg: {sw.ElapsedMilliseconds / 1000.0}ms");
// Vấn đề: JIT warmup, GC interference, CPU cache effects, không có thống kê
```

BenchmarkDotNet giải quyết: warmup tự động, nhiều lần chạy, loại outlier, báo cáo đầy đủ.

### Cài Đặt

```bash
# Tạo project benchmark riêng (không đặt trong Test project)
dotnet new console -n MyApp.Benchmarks
dotnet add package BenchmarkDotNet
```

### Benchmark Cơ Bản

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

// Cấu hình benchmark
[MemoryDiagnoser]           // Đo memory allocation
[SimpleJob(RuntimeMoniker.Net80)]
public class StringConcatBenchmark
{
    private const int N = 1000;
    private readonly string[] _words;

    public StringConcatBenchmark()
    {
        _words = Enumerable.Range(0, N)
                           .Select(i => $"word{i}")
                           .ToArray();
    }

    [Benchmark(Baseline = true)]
    public string StringConcatOperator()
    {
        var result = "";
        foreach (var word in _words)
            result += word; // O(n²) allocations!
        return result;
    }

    [Benchmark]
    public string StringBuilderConcat()
    {
        var sb = new StringBuilder();
        foreach (var word in _words)
            sb.Append(word);
        return sb.ToString(); // O(n) với ít allocation hơn
    }

    [Benchmark]
    public string StringJoin()
    {
        return string.Join("", _words); // Tối ưu nhất
    }

    [Benchmark]
    public string SpanConcat()
    {
        return string.Concat(_words.AsSpan()); // Zero extra allocation
    }
}

// Entrypoint
BenchmarkRunner.Run<StringConcatBenchmark>();
```

```
# Kết quả mẫu:
| Method              | Mean        | Ratio | Allocated |
|---------------------|-------------|-------|-----------|
| StringConcatOperator| 4,521.3 μs  | 1.00  | 4,089 KB  |
| StringBuilderConcat |   125.4 μs  | 0.03  | 16 KB     |
| StringJoin          |    89.2 μs  | 0.02  | 8 KB      |
| SpanConcat          |    67.1 μs  | 0.01  | 4 KB      |
```

### Parameterized Benchmark — Tham Số Hóa

```csharp
[MemoryDiagnoser]
public class CollectionLookupBenchmark
{
    [Params(100, 1000, 10000)]  // Chạy với các kích thước khác nhau
    public int N { get; set; }

    private List<int> _list;
    private HashSet<int> _hashSet;
    private Dictionary<int, int> _dict;
    private int _searchTarget;

    [GlobalSetup]
    public void Setup()
    {
        var data = Enumerable.Range(0, N).ToArray();
        _list = new List<int>(data);
        _hashSet = new HashSet<int>(data);
        _dict = data.ToDictionary(x => x);
        _searchTarget = N / 2; // Tìm phần tử ở giữa
    }

    [Benchmark(Baseline = true)]
    public bool ListContains() => _list.Contains(_searchTarget); // O(n)

    [Benchmark]
    public bool HashSetContains() => _hashSet.Contains(_searchTarget); // O(1)

    [Benchmark]
    public bool DictionaryContainsKey() => _dict.ContainsKey(_searchTarget); // O(1)
}
```

### Các Attribute Quan Trọng

```csharp
[MemoryDiagnoser]          // Đo allocation (Gen0/1/2 GC, bytes)
[ThreadingDiagnoser]       // Đo context switches
[CpuUsageDiagnoser]        // Đo CPU usage
[GcForce]                  // Force GC trước mỗi benchmark
[InvocationCount(1000)]    // Số lần gọi mỗi iteration
[WarmupCount(5)]           // Số lần warmup
[IterationCount(10)]       // Số lần đo
[Benchmark(Baseline = true)] // Đặt làm baseline để so sánh ratio
[IterationSetup]           // Chạy trước mỗi iteration
[GlobalSetup]              // Chạy một lần trước tất cả benchmarks
[GlobalCleanup]            // Dọn dẹp sau tất cả
```

### So Sánh Serialization — Ví Dụ Thực Tế

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class SerializationBenchmark
{
    private readonly Product _product;
    private readonly string _json;

    public SerializationBenchmark()
    {
        _product = new Product
        {
            Id = Guid.NewGuid(),
            Name = "Benchmark Widget",
            Price = 99.99m,
            Tags = new[] { "electronics", "sale", "featured" }
        };
        _json = System.Text.Json.JsonSerializer.Serialize(_product);
    }

    [Benchmark]
    public string SystemTextJson_Serialize()
        => System.Text.Json.JsonSerializer.Serialize(_product);

    [Benchmark]
    public string NewtonsoftJson_Serialize()
        => Newtonsoft.Json.JsonConvert.SerializeObject(_product);

    [Benchmark]
    public Product? SystemTextJson_Deserialize()
        => System.Text.Json.JsonSerializer.Deserialize<Product>(_json);

    [Benchmark]
    public Product? NewtonsoftJson_Deserialize()
        => Newtonsoft.Json.JsonConvert.DeserializeObject<Product>(_json);
}
```

---

## 🔫 NBomber — Load Testing Trong .NET

**NBomber** — load testing framework native .NET, viết test bằng C#, không cần học ngôn ngữ khác.

### Cài Đặt

```bash
dotnet add package NBomber
dotnet add package NBomber.Http  # Cho HTTP testing
```

### Load Test Cơ Bản

```csharp
using NBomber.Contracts;
using NBomber.CSharp;
using NBomber.Http.CSharp;

var httpClient = new HttpClient();

// Định nghĩa scenario (kịch bản)
var scenario = Scenario.Create("get_products", async context =>
    {
        var request = Http.CreateRequest("GET", "https://localhost:5001/api/products")
                         .WithHeader("Accept", "application/json");

        var response = await Http.Send(httpClient, request);
        return response;
    })
    .WithWarmUpDuration(TimeSpan.FromSeconds(5))   // Warmup 5 giây
    .WithLoadSimulations(
        Simulation.Inject(                          // Inject: thêm users theo tốc độ
            rate: 10,                               // 10 request/giây
            interval: TimeSpan.FromSeconds(1),
            during: TimeSpan.FromSeconds(30)),      // Trong 30 giây
        Simulation.KeepConstant(                    // KeepConstant: giữ số users cố định
            copies: 50,                             // 50 concurrent users
            during: TimeSpan.FromSeconds(60))
    );

// Chạy test
NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFolder("./reports")
    .WithReportFormats(ReportFormat.Html, ReportFormat.Csv)
    .Run();
```

### Load Simulation Patterns — Mẫu Mô Phỏng Tải

```csharp
// Inject — tiêm request theo tốc độ cố định
Simulation.Inject(rate: 10, interval: TimeSpan.FromSeconds(1),
    during: TimeSpan.FromMinutes(5))

// InjectRandom — tiêm với tốc độ ngẫu nhiên
Simulation.InjectRandom(minRate: 5, maxRate: 20,
    interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))

// KeepConstant — giữ số lượng concurrent users
Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(10))

// RampingInject — tăng dần tốc độ (Ramp Up)
Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1),
    during: TimeSpan.FromMinutes(2))  // Tăng từ 0 → 100 req/s trong 2 phút

// RampingConstant — tăng dần số users
Simulation.RampingConstant(copies: 200, during: TimeSpan.FromMinutes(3))

// Spike — đột biến tải
Simulation.KeepConstant(copies: 10, during: TimeSpan.FromSeconds(30)),
Simulation.KeepConstant(copies: 500, during: TimeSpan.FromSeconds(5)), // Spike!
Simulation.KeepConstant(copies: 10, during: TimeSpan.FromSeconds(30))
```

### Scenario Phức Tạp — Complete User Journey

```csharp
var httpClient = new HttpClient { BaseAddress = new Uri("https://localhost:5001") };

var loginScenario = Scenario.Create("user_login", async context =>
    {
        var request = Http.CreateRequest("POST", "/api/auth/login")
            .WithJsonBody(new { email = "user@test.com", password = "Test123!" });

        var response = await Http.Send(httpClient, request);

        // Kiểm tra response
        if (!response.IsOk)
            return Response.Fail(statusCode: (int)response.StatusCode);

        return Response.Ok(
            statusCode: (int)response.StatusCode,
            sizeBytes: response.Payload?.SizeBytes ?? 0);
    })
    .WithWarmUpDuration(TimeSpan.FromSeconds(10))
    .WithLoadSimulations(
        Simulation.RampingConstant(copies: 50, during: TimeSpan.FromMinutes(2)),
        Simulation.KeepConstant(copies: 50, during: TimeSpan.FromMinutes(5)),
        Simulation.RampingConstant(copies: 0, during: TimeSpan.FromMinutes(1))
    );

NBomberRunner
    .RegisterScenarios(loginScenario)
    .WithTestSuite("Authentication Load Tests")
    .WithTestName("Login Endpoint")
    .Run();
```

### Đặt Threshold — Ngưỡng Thất Bại

```csharp
var scenario = Scenario.Create("api_test", async ctx => { ... })
    .WithLoadSimulations(Simulation.Inject(50, TimeSpan.FromSeconds(1),
        TimeSpan.FromMinutes(5)));

NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFolder("./reports")
    .Run();

// Hoặc kiểm tra trong assertion
// P99 latency < 500ms, error rate < 1%
```

---

## 🌊 k6 — Load Testing Với JavaScript

**k6** là công cụ load testing open-source viết script bằng JavaScript, rất phổ biến cho REST API testing.

### Cài Đặt

```bash
# Windows (Chocolatey)
choco install k6

# macOS
brew install k6

# Linux
sudo apt-get install k6
```

### Script k6 Cơ Bản

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Custom metric — tỷ lệ lỗi
const errorRate = new Rate('errors');

// Cấu hình load
export const options = {
    stages: [
        { duration: '2m', target: 50 },   // Ramp up: 0→50 users trong 2 phút
        { duration: '5m', target: 50 },   // Giữ 50 users trong 5 phút
        { duration: '2m', target: 100 },  // Ramp up tiếp: 50→100 users
        { duration: '5m', target: 100 },  // Giữ 100 users
        { duration: '2m', target: 0 },    // Ramp down về 0
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
        errors: ['rate<0.01'],              // Error rate < 1%
        http_req_failed: ['rate<0.05'],    // HTTP failure rate < 5%
    },
};

export default function () {
    // Test GET /api/products
    const res = http.get('https://localhost:5001/api/products', {
        headers: { 'Accept': 'application/json' },
    });

    // Kiểm tra response
    const success = check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
        'has products array': (r) => JSON.parse(r.body).length > 0,
    });

    errorRate.add(!success);
    sleep(1); // Nghỉ 1 giây giữa các requests
}
```

### Test Authentication Flow

```javascript
// auth-flow-test.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';

export const options = {
    vus: 20,            // VUs — Virtual Users — Người dùng ảo
    duration: '5m',
    thresholds: {
        'http_req_duration{name:login}': ['p(99)<1000'],   // Login < 1s (p99)
        'http_req_duration{name:get_orders}': ['p(99)<500'],
    },
};

export default function () {
    let authToken;

    group('Authentication', () => {
        const loginRes = http.post(
            'https://localhost:5001/api/auth/login',
            JSON.stringify({ email: 'user@test.com', password: 'Test123!' }),
            {
                headers: { 'Content-Type': 'application/json' },
                tags: { name: 'login' },  // Tag để theo dõi riêng
            }
        );

        check(loginRes, {
            'login successful': (r) => r.status === 200,
        });

        authToken = JSON.parse(loginRes.body).token;
    });

    group('Orders', () => {
        const ordersRes = http.get(
            'https://localhost:5001/api/orders',
            {
                headers: {
                    'Authorization': `Bearer ${authToken}`,
                },
                tags: { name: 'get_orders' },
            }
        );

        check(ordersRes, {
            'orders returned': (r) => r.status === 200,
            'has data': (r) => JSON.parse(r.body).length >= 0,
        });
    });

    sleep(Math.random() * 3 + 1); // Random sleep 1-4s (realistic user behavior)
}
```

### Chạy k6

```bash
# Chạy load test
k6 run load-test.js

# Chạy với output vào InfluxDB (visualize với Grafana)
k6 run --out influxdb=http://localhost:8086/k6 load-test.js

# Chạy với số VUs và duration cụ thể (override options)
k6 run --vus 100 --duration 30s load-test.js

# Chạy stress test (tăng dần đến giới hạn)
k6 run --stage 0s:0,2m:500,2m:500,1m:0 load-test.js
```

---

## 📊 Phân Tích Kết Quả

### Chỉ Số Quan Trọng

| Chỉ Số                   | Ký Hiệu  | Mô Tả                                      | Mục Tiêu Thông Thường |
| ------------------------ | -------- | ------------------------------------------ | --------------------- |
| **Response Time P50**    | p50      | 50% requests hoàn thành trong thời gian này | < 100ms               |
| **Response Time P95**    | p95      | 95% requests hoàn thành trong thời gian này | < 500ms               |
| **Response Time P99**    | p99      | 99% requests hoàn thành trong thời gian này | < 1000ms              |
| **Throughput**           | RPS      | Requests per second — Số request/giây       | Tùy nghiệp vụ         |
| **Error Rate**           | err%     | Tỷ lệ request thất bại                     | < 1%                  |
| **Concurrent Users**     | VUs      | Số người dùng ảo đồng thời                  | Theo yêu cầu          |

### Đọc Output NBomber

```
Scenario: get_products
  Duration: 1 min
  Requests count: 3,421
  OK count:       3,400 (99.4%)
  Fail count:        21 (0.6%)

  Latency:
    p50:    45ms
    p75:    78ms
    p95:   142ms
    p99:   312ms
    Max:  1,240ms   ← Outlier đáng chú ý

  Throughput: 57 RPS (requests per second)
```

---

## 🔧 Tích Hợp Performance Test Vào CI/CD

### GitHub Actions — Regression Test Hiệu Năng

```yaml
# .github/workflows/perf.yml
name: Performance Tests

on:
  pull_request:
    branches: [main]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Run Benchmarks
        run: |
          dotnet run -c Release --project ./MyApp.Benchmarks \
            -- --job short --runtimes net8.0 \
               --exporters json \
               --artifacts ./benchmark-results

      - name: Compare with Baseline
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'benchmarkdotnet'
          output-file-path: './benchmark-results/BenchmarkDotNet.Artifacts/results/**/*.json'
          fail-on-alert: true          # Fail nếu hiệu năng giảm > threshold
          alert-threshold: '120%'      # Alert nếu chậm hơn 20%
          github-token: ${{ secrets.GITHUB_TOKEN }}
          auto-push: true
```

---

## 🏗️ Best Practices — Thực Hành Tốt Nhất

### BenchmarkDotNet

```
✅ Chạy ở Release configuration, không Debug
✅ Dùng [GlobalSetup] để chuẩn bị data, không tính vào thời gian benchmark
✅ Dùng [MemoryDiagnoser] để phát hiện allocation không mong muốn
✅ Dùng [Params] để test với kích thước dữ liệu khác nhau
✅ Đặt Baseline để so sánh tương đối
❌ Không dùng Stopwatch thủ công
❌ Không chạy trong Debug mode
❌ Không kết hợp benchmark và unit test trong cùng project
```

### Load Testing

```
✅ Test môi trường staging/production-like, không test localhost đơn giản
✅ Bắt đầu với load nhỏ, tăng dần để tìm breaking point
✅ Monitor server metrics (CPU, RAM, DB connections) cùng lúc
✅ Test realistic user journeys, không chỉ một endpoint
✅ Warm up trước khi đo chính thức
❌ Không load test production trực tiếp (trừ khi có kế hoạch)
❌ Không bỏ qua error rate — 0.1% error với 1000 RPS = 1 error/giây
```

---

## 📋 Checklist Performance Testing

```
□ Benchmark code trước khi optimize ("measure, don't guess")
□ Tạo benchmark project riêng, không mix với test project
□ Cấu hình MemoryDiagnoser để phát hiện allocation
□ Đặt baseline để track regression
□ Tích hợp benchmark vào CI để phát hiện performance regression
□ Load test với realistic data và traffic patterns
□ Test P95/P99, không chỉ P50 (average che giấu outlier)
□ Monitor server resources trong khi load test
□ Document performance targets trước khi test
```

---

## 🔗 Liên Quan

- [1-unit-testing.md](./1-unit-testing.md) — Tổng quan testing
- [5-test-coverage.md](./5-test-coverage.md) — Coverage khác loại
- [07-performance/4-benchmarking.md](../07-performance/4-benchmarking.md) — BenchmarkDotNet chi tiết
- [07-performance/5-caching.md](../07-performance/5-caching.md) — Tối ưu sau khi phát hiện bottleneck

---

*Cập nhật: 2026-06-02 | BenchmarkDotNet 0.14.x | NBomber 5.x | k6 0.52.x | .NET 8*
