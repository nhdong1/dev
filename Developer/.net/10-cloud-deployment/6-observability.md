# Observability — OpenTelemetry, Serilog, Metrics, Distributed Tracing

> Observability — Khả Năng Quan Sát là khả năng hiểu trạng thái nội bộ của hệ thống dựa trên dữ liệu đầu ra của nó. Ba trụ cột của observability là: Logs — Nhật Ký, Metrics — Chỉ Số và Traces — Vết Theo Dõi. Bài này trình bày cách triển khai observability toàn diện cho .NET với Serilog, OpenTelemetry, Prometheus, Grafana và Jaeger.

---

## 1. Ba Trụ Cột Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    BA TRỤ CỘT OBSERVABILITY                     │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │    LOGS      │  │   METRICS    │  │       TRACES         │  │
│  │  (Nhật Ký)  │  │  (Chỉ Số)   │  │  (Vết Theo Dõi)     │  │
│  │              │  │              │  │                      │  │
│  │ Gì xảy ra   │  │ Hệ thống    │  │ Request đi qua       │  │
│  │ khi nào     │  │ khỏe mạnh   │  │ những đâu            │  │
│  │ với ai      │  │ như thế nào │  │ mất bao lâu          │  │
│  │             │  │             │  │ ở từng bước          │  │
│  │ Serilog     │  │ Prometheus  │  │ OpenTelemetry        │  │
│  │ Seq         │  │ Grafana     │  │ Jaeger / Zipkin      │  │
│  │ Log Analytics│ │ Azure Monitor│  │ Azure App Insights   │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Khi Nào Dùng Cái Gì?

| Câu Hỏi | Dùng |
|---------|------|
| "Lỗi gì xảy ra với user X lúc 10:30?" | **Logs** |
| "API của tôi có bị chậm không?" | **Metrics** |
| "Request này chậm ở microservice nào?" | **Traces** |
| "Hệ thống có bình thường không?" | **Metrics + Alerts** |

---

## 2. Serilog — Structured Logging — Ghi Log Có Cấu Trúc

### Tại Sao Dùng Serilog?

```csharp
// ❌ Unstructured logging — khó query sau này
logger.LogInformation($"Order {orderId} processed by user {userId} in {elapsedMs}ms");
// Log ra: "Order 12345 processed by user 67890 in 234ms"
// Không thể query: "tìm tất cả orders > 200ms"

// ✅ Structured logging với Serilog — có thể query bất kỳ field nào
logger.LogInformation("Order {OrderId} processed by {UserId} in {ElapsedMs}ms",
    orderId, userId, elapsedMs);
// Log ra JSON: { "OrderId": 12345, "UserId": 67890, "ElapsedMs": 234, "Message": "..." }
// Có thể query: WHERE ElapsedMs > 200
```

### Cài Đặt Serilog

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq          # Seq log server
dotnet add package Serilog.Sinks.ApplicationInsights
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Process
dotnet add package Serilog.Enrichers.Thread
dotnet add package Serilog.Enrichers.Span     # OpenTelemetry TraceId/SpanId
```

### Cấu Hình Serilog trong Program.cs

```csharp
// Program.cs
using Serilog;
using Serilog.Events;

// Tạo bootstrap logger để catch lỗi trong startup
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // Thay thế default logging bằng Serilog
    builder.Host.UseSerilog((context, services, configuration) =>
        configuration
            .ReadFrom.Configuration(context.Configuration)   // đọc từ appsettings.json
            .ReadFrom.Services(services)
            .Enrich.FromLogContext()
            .Enrich.WithMachineName()
            .Enrich.WithEnvironmentName()
            .Enrich.WithProperty("Application", "MyApp.Api")
            .Enrich.WithProperty("Version", Assembly.GetExecutingAssembly()
                .GetName().Version?.ToString() ?? "unknown"));

    var app = builder.Build();

    // Ghi log mọi HTTP request
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate =
            "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        };
    });

    await app.RunAsync();
    return 0;
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
    return 1;
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

### Cấu Hình trong appsettings.json

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}",
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/myapp-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 14,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://seq:5341",
          "apiKey": "seq-api-key"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  }
}
```

### Correlation ID — ID Tương Quan

```csharp
// Middleware thêm CorrelationId vào mọi request
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Response.Headers[CorrelationIdHeader] = correlationId;

        // Thêm vào Serilog context để tất cả logs trong request có CorrelationId
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}
```

### Logging Best Practices

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Dùng LogContext để thêm properties vào toàn bộ scope
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = request.OrderId,
            ["CustomerId"] = request.CustomerId
        });

        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);

        try
        {
            var order = await _repository.CreateAsync(request);

            // ✅ Log business events quan trọng ở Information level
            _logger.LogInformation(
                "Order {OrderId} created successfully. Total: {Total:C}",
                order.Id, order.Total);

            return order;
        }
        catch (InsufficientStockException ex)
        {
            // ✅ Business exceptions ở Warning (không phải Error — không cần wake up on-call)
            _logger.LogWarning(ex,
                "Insufficient stock for order {OrderId}. Item: {ItemId}",
                request.OrderId, ex.ItemId);
            throw;
        }
        catch (Exception ex)
        {
            // ✅ Unexpected exceptions ở Error level
            _logger.LogError(ex,
                "Failed to create order {OrderId} for customer {CustomerId}",
                request.OrderId, request.CustomerId);
            throw;
        }
    }
}
```

---

## 3. OpenTelemetry — Tiêu Chuẩn Observability Mở

OpenTelemetry — OTEL — là tiêu chuẩn mở cho observability, hỗ trợ Traces, Metrics và Logs.

### Cài Đặt OpenTelemetry

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Instrumentation.SqlClient
dotnet add package OpenTelemetry.Instrumentation.StackExchangeRedis
dotnet add package OpenTelemetry.Exporter.Otlp          # gửi đến Collector
dotnet add package OpenTelemetry.Exporter.Console       # debug local
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore  # Application Insights
```

### Cấu Hình OpenTelemetry

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "MyApp.Api",
            serviceVersion: Assembly.GetExecutingAssembly().GetName().Version?.ToString(),
            serviceInstanceId: Environment.MachineName))

    // ─── Distributed Tracing — Theo Dõi Phân Tán ──────────────────
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(options =>
        {
            options.RecordException = true;
            // Không trace health check endpoints
            options.Filter = context =>
                !context.Request.Path.StartsWithSegments("/health");
        })
        .AddHttpClientInstrumentation(options =>
        {
            options.RecordException = true;
        })
        .AddEntityFrameworkCoreInstrumentation(options =>
        {
            options.SetDbStatementForText = true;  // include SQL query trong trace
        })
        .AddRedisInstrumentation()
        .AddSource("MyApp.*")  // include custom ActivitySources
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(builder.Configuration["OpenTelemetry:CollectorEndpoint"]
                ?? "http://otel-collector:4317");
        })
        .AddConsoleExporter())  // chỉ development

    // ─── Metrics — Chỉ Số ─────────────────────────────────────────
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()    // GC, threadpool metrics
        .AddMeter("MyApp.*")            // custom meters
        .AddPrometheusExporter()        // expose /metrics endpoint
        .AddOtlpExporter());
```

### Custom Traces — Trace Tùy Chỉnh

```csharp
// Tạo ActivitySource cho custom instrumentation
public class OrderService
{
    // ActivitySource dùng để tạo spans
    private static readonly ActivitySource ActivitySource =
        new("MyApp.OrderService");

    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Tạo một span — khoảng thời gian đo lường
        using var activity = ActivitySource.StartActivity("CreateOrder");
        activity?.SetTag("order.customer_id", request.CustomerId);
        activity?.SetTag("order.item_count", request.Items.Count);

        try
        {
            // Child span cho payment processing
            using var paymentActivity = ActivitySource.StartActivity("ProcessPayment");
            var payment = await _paymentService.ProcessAsync(request.PaymentInfo);
            paymentActivity?.SetTag("payment.method", payment.Method);
            paymentActivity?.SetTag("payment.amount", payment.Amount);

            var order = await _repository.CreateAsync(request);

            activity?.SetTag("order.id", order.Id);
            activity?.SetStatus(ActivityStatusCode.Ok);

            return order;
        }
        catch (Exception ex)
        {
            // Ghi exception vào span
            activity?.RecordException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### Custom Metrics — Chỉ Số Tùy Chỉnh

```csharp
// OrderMetrics.cs — định nghĩa business metrics
public class OrderMetrics
{
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderProcessingTime;
    private readonly ObservableGauge<int> _pendingOrders;
    private readonly IOrderRepository _repository;

    public OrderMetrics(IMeterFactory meterFactory, IOrderRepository repository)
    {
        _repository = repository;
        var meter = meterFactory.Create("MyApp.Orders");

        // Counter — đếm số orders được tạo
        _ordersCreated = meter.CreateCounter<long>(
            name: "myapp.orders.created.total",
            unit: "{orders}",
            description: "Total number of orders created");

        // Histogram — phân phối thời gian xử lý
        _orderProcessingTime = meter.CreateHistogram<double>(
            name: "myapp.orders.processing.duration",
            unit: "ms",
            description: "Order processing duration in milliseconds");

        // Gauge — số orders đang chờ xử lý (real-time)
        _pendingOrders = meter.CreateObservableGauge<int>(
            name: "myapp.orders.pending.count",
            observeValue: () => _repository.GetPendingCountAsync().GetAwaiter().GetResult(),
            description: "Current number of pending orders");
    }

    public void RecordOrderCreated(string paymentMethod, string region)
    {
        _ordersCreated.Add(1, new TagList
        {
            { "payment_method", paymentMethod },
            { "region", region }
        });
    }

    public void RecordProcessingTime(double milliseconds, string status)
    {
        _orderProcessingTime.Record(milliseconds, new TagList
        {
            { "status", status }
        });
    }
}
```

---

## 4. Prometheus và Grafana — Thu Thập và Hiển Thị Metrics

### Expose Prometheus Endpoint

```csharp
// Program.cs
app.MapPrometheusScrapingEndpoint("/metrics");
// Prometheus sẽ scrape http://myapp:8080/metrics mỗi 15s
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'myapp-api'
    static_configs:
      - targets: ['myapp-api:8080']
    metrics_path: '/metrics'

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Grafana Dashboard JSON

Các metrics quan trọng cần monitor — theo dõi:

```
# Request rate — tốc độ request
rate(http_server_request_duration_seconds_count[5m])

# Error rate — tỷ lệ lỗi
rate(http_server_request_duration_seconds_count{http_status_code=~"5.."}[5m])
/ rate(http_server_request_duration_seconds_count[5m])

# P99 latency — độ trễ phần vị 99
histogram_quantile(0.99, rate(http_server_request_duration_seconds_bucket[5m]))

# GC pause time — thời gian tạm dừng GC
rate(dotnet_gc_pause_ratio[5m])

# Thread pool queue — hàng đợi thread pool
dotnet_threadpool_queue_length

# Custom business metric
rate(myapp_orders_created_total[5m])
```

---

## 5. Distributed Tracing — Theo Dõi Phân Tán

### Trace Flow qua Microservices

```
HTTP Request
    │
    ▼ TraceId: abc123, SpanId: span1
┌───────────┐
│  API GW   │──────────────────────────────► TraceId: abc123
└───────────┘
    │ gRPC
    ▼ SpanId: span2 (parent: span1)
┌───────────┐
│ Order Svc │──────────► DB Query (span3)
└───────────┘──────────► Redis (span4)
    │ HTTP
    ▼ SpanId: span5 (parent: span2)
┌───────────┐
│ Payment   │──────────► External API (span6)
└───────────┘

Jaeger/Zipkin hiển thị waterfall chart:
  span1 |────────────────────────────────────────| 250ms
  span2   |──────────────────────────────────| 200ms
  span3     |──────| 30ms (SQL query)
  span4          |────| 20ms (Redis)
  span5               |────────────────────| 120ms
  span6                 |──────────────| 100ms (Stripe API)
```

### W3C Trace Context — Tiêu Chuẩn Truyền Context

```
traceparent: 00-abc123def456...(TraceId)-span001...(SpanId)-01(sampled)
tracestate:  myapp=specific-vendor-data
```

```csharp
// HttpClient tự động propagate trace context khi dùng OpenTelemetry
// AddHttpClientInstrumentation() inject traceparent header

// Kiểm tra current trace trong code
var activity = Activity.Current;
var traceId = activity?.TraceId.ToString();
var spanId = activity?.SpanId.ToString();

// Thêm vào response header để client biết
Response.Headers["X-Trace-Id"] = traceId;
```

---

## 6. Application Insights — Azure Managed Observability

```bash
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore
```

```csharp
// Program.cs — tất cả trong một lệnh
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor(options =>
    {
        options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    });
```

### Kusto Query — KQL cho Log Analytics

```kusto
// Tìm requests chậm nhất trong 1 giờ qua
requests
| where timestamp > ago(1h)
| where success == false or duration > 1000  // chậm hơn 1 giây
| order by duration desc
| take 20
| project timestamp, name, duration, resultCode, url

// Tỷ lệ lỗi theo endpoint
requests
| where timestamp > ago(24h)
| summarize
    total = count(),
    failed = countif(success == false),
    error_rate = round(100.0 * countif(success == false) / count(), 2)
  by name
| order by error_rate desc

// Top exceptions
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| order by count_ desc
| take 10
```

---

## 7. Health Checks Nâng Cao

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: connectionString,
        healthQuery: "SELECT 1",
        name: "database",
        failureStatus: HealthStatus.Degraded,  // degraded thay vì unhealthy
        tags: ["ready", "db"])
    .AddRedis(
        redisConnectionString,
        name: "redis",
        tags: ["ready", "cache"])
    .AddCheck<ExternalApiHealthCheck>("payment-gateway",
        failureStatus: HealthStatus.Degraded,
        tags: ["ready", "external"])
    // Disk space check
    .AddDiskStorageHealthCheck(options =>
        options.AddDrive("/", minimumFreeMegabytes: 500),
        name: "disk-space",
        tags: ["live"]);

// Expose detailed health endpoint (chỉ cho internal monitoring)
app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    AllowCachingResponses = false
}).RequireAuthorization("HealthCheckPolicy");  // chỉ cho internal IPs
```

---

## 8. Alerting — Cảnh Báo

### Azure Monitor Alerts

```bash
# Alert khi error rate > 5%
az monitor metrics alert create \
    --name "high-error-rate" \
    --resource-group myapp-rg \
    --scopes /subscriptions/.../Microsoft.Web/sites/myapp-api \
    --condition "avg requests/Failed > 5" \
    --window-size 5m \
    --evaluation-frequency 1m \
    --action-group myapp-alerts-ag

# Alert khi P99 latency > 2 giây
az monitor metrics alert create \
    --name "high-latency-p99" \
    --resource-group myapp-rg \
    --scopes /subscriptions/.../Microsoft.Web/sites/myapp-api \
    --condition "avg requests/Duration > 2000" \
    --window-size 5m \
    --evaluation-frequency 1m \
    --action-group myapp-alerts-ag
```

### SLA — Service Level Agreement — Thỏa Thuận Mức Dịch Vụ Targets

```
SLI — Service Level Indicator — Chỉ Số Mức Dịch Vụ:
  - Availability: requests thành công / tổng requests
  - Latency: % requests < 200ms
  - Error rate: requests lỗi / tổng requests

SLO — Service Level Objective — Mục Tiêu Mức Dịch Vụ:
  - Availability: 99.9% (cho phép ~44 phút downtime/tháng)
  - P95 latency < 500ms
  - Error rate < 0.1%

SLA — Service Level Agreement — cam kết với khách hàng:
  - Thường thấp hơn SLO (buffer)
  - 99.5% availability trong SLA nếu SLO là 99.9%

Error Budget — Ngân Sách Lỗi:
  - 99.9% SLO = 0.1% error budget = 43.8 phút/tháng
  - Khi hết error budget → freeze new deployments
```

---

## 9. OpenTelemetry Collector — Bộ Thu Thập Trung Tâm

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 400
    spike_limit_mib: 100

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  prometheus:
    endpoint: "0.0.0.0:8889"
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

---

## 10. Tổng Hợp: Stack Observability Production

```yaml
# docker-compose.observability.yml — local development stack
version: '3.9'

services:
  # ─── Logs ────────────────────────────────────────────
  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"      # Seq UI
    environment:
      ACCEPT_EULA: Y

  # ─── Traces ──────────────────────────────────────────
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"

  # ─── Metrics ─────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"    # Grafana UI (admin/admin)
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  # ─── OpenTelemetry Collector ─────────────────────────
  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    ports:
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    depends_on:
      - jaeger
      - prometheus
```

---

## Checklist Observability .NET

- [ ] Serilog với structured logging — không dùng string interpolation trong log messages
- [ ] CorrelationId middleware được thêm vào request pipeline
- [ ] OpenTelemetry traces cho HTTP requests, DB calls, Redis, external APIs
- [ ] Custom business metrics (orders/min, revenue/min, error rates)
- [ ] Health checks với liveness và readiness endpoints
- [ ] Alerts được cài cho: error rate > threshold, P99 latency > threshold, disk space
- [ ] Logs không chứa PII — Personally Identifiable Information — thông tin nhận dạng cá nhân
- [ ] Log retention policy — chính sách lưu giữ log được cấu hình (30-90 ngày)
- [ ] SLO được định nghĩa và đo lường
- [ ] Runbook cho mỗi alert (hướng dẫn xử lý)

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa Logs, Metrics và Traces?**
> **Logs**: sự kiện rời rạc với timestamp, context đầy đủ. Tốt cho debugging chi tiết. **Metrics**: số liệu tổng hợp theo thời gian (counters, gauges, histograms). Tốt cho alerting và trending. **Traces**: theo dõi một request qua nhiều services, hiển thị latency từng bước. Tốt cho performance debugging trong microservices.

**Q: Tại sao structured logging quan trọng hơn string concatenation?**
> Structured logging lưu properties như các trường dữ liệu riêng biệt, cho phép query hiệu quả ("tìm tất cả requests của user X", "tìm tất cả orders > 500ms"). String concatenation tạo ra một blob text không thể parse. Khi có hàng triệu log entries, chỉ structured logging mới cho phép tìm kiếm và phân tích nhanh.

**Q: OpenTelemetry là gì và tại sao nên dùng thay vì vendor SDK trực tiếp?**
> OpenTelemetry là tiêu chuẩn mở (CNCF project) cho observability instrumentation. Dùng OTEL thay vì vendor SDK (Jaeger SDK, Datadog SDK) vì: vendor-agnostic — không bị lock-in vào một nhà cung cấp, standard API — code không thay đổi khi đổi backend, ecosystem phong phú — hỗ trợ mọi frameworks và libraries phổ biến.

**Cập Nhật Lần Cuối:** 2026-06-02
