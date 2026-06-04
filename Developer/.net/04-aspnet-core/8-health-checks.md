# 8 — Health Checks (Kiểm Tra Tình Trạng Dịch Vụ)

> Health Checks — Kiểm Tra Tình Trạng — là endpoint chuẩn để hệ thống điều phối (Kubernetes, load balancer) biết ứng dụng có đang hoạt động bình thường không. ASP.NET Core tích hợp sẵn health check framework với khả năng kiểm tra database, cache, external APIs và bất kỳ dependency nào.

---

## 📋 Tổng Quan Nhanh

| Loại Probe | Mục Đích | Kubernetes Dùng Cho |
| ---------- | -------- | ------------------- |
| **Liveness** — Sống sót | App có còn sống không? Nếu fail → restart pod | `livenessProbe` |
| **Readiness** — Sẵn sàng | App có sẵn sàng nhận traffic không? Nếu fail → ngừng routing | `readinessProbe` |
| **Startup** — Khởi động | App đã khởi động xong chưa? | `startupProbe` |

---

## 1. Setup Cơ Bản

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy("App đang chạy bình thường"))
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default")!,
        name: "database",
        tags: new[] { "db", "ready" })
    .AddRedis(
        redisConnectionString: builder.Configuration["Redis:ConnectionString"]!,
        name: "redis",
        tags: new[] { "cache", "ready" })
    .AddUrlGroup(
        uri: new Uri("https://api.external-service.com/health"),
        name: "external-api",
        tags: new[] { "external", "ready" });

var app = builder.Build();

// Endpoint đơn giản — trả Healthy/Unhealthy
app.MapHealthChecks("/health");

// Endpoint chi tiết với JSON response
app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Liveness — chỉ kiểm tra app còn sống (không include db, redis)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live") || !check.Tags.Any()
});

// Readiness — kiểm tra tất cả dependencies
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready") || check.Tags.Contains("db")
});
```

---

## 2. Custom Health Check

### Implement IHealthCheck

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly AppDbContext _db;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(AppDbContext db, ILogger<DatabaseHealthCheck> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Thử kết nối database
            var canConnect = await _db.Database.CanConnectAsync(cancellationToken);

            if (!canConnect)
                return HealthCheckResult.Unhealthy("Không thể kết nối database");

            // Kiểm tra thêm: query đơn giản
            var count = await _db.Users.CountAsync(cancellationToken);

            return HealthCheckResult.Healthy("Database hoạt động bình thường", new Dictionary<string, object>
            {
                ["userCount"] = count,
                ["timestamp"] = DateTime.UtcNow
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");

            return HealthCheckResult.Unhealthy(
                "Database health check thất bại",
                exception: ex,
                data: new Dictionary<string, object>
                {
                    ["error"] = ex.Message,
                    ["timestamp"] = DateTime.UtcNow
                });
        }
    }
}
```

### Đăng Ký Custom Health Check

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { "db", "ready" })
    .AddCheck<RedisHealthCheck>("redis", tags: new[] { "cache", "ready" })
    .AddCheck<ExternalApiHealthCheck>("payment-gateway",
        failureStatus: HealthStatus.Degraded,  // Fail → Degraded thay vì Unhealthy
        tags: new[] { "external" });
```

### Health Check Với Timeout

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>(
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db" },
        timeout: TimeSpan.FromSeconds(5));  // Timeout sau 5 giây
```

---

## 3. Các Health Check Thư Viện Có Sẵn

```bash
# Cài package
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.Redis
dotnet add package AspNetCore.HealthChecks.NpgSql
dotnet add package AspNetCore.HealthChecks.Rabbitmq
dotnet add package AspNetCore.HealthChecks.Uris
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
```

```csharp
builder.Services.AddHealthChecks()
    // SQL Server
    .AddSqlServer(
        connectionString: config.GetConnectionString("Default")!,
        healthQuery: "SELECT 1",
        name: "sql-server",
        tags: new[] { "db", "ready" })

    // PostgreSQL
    .AddNpgSql(
        npgsqlConnectionString: config.GetConnectionString("Postgres")!,
        name: "postgresql",
        tags: new[] { "db", "ready" })

    // Redis
    .AddRedis(
        redisConnectionString: config["Redis"]!,
        name: "redis",
        tags: new[] { "cache", "ready" })

    // RabbitMQ
    .AddRabbitMQ(
        rabbitConnectionString: config["RabbitMQ:ConnectionString"]!,
        name: "rabbitmq",
        tags: new[] { "messaging", "ready" })

    // HTTP endpoint check
    .AddUrlGroup(
        new Uri("https://api.stripe.com/v1"),
        name: "stripe",
        tags: new[] { "external" })

    // Disk space check
    .AddDiskStorageHealthCheck(setup =>
        setup.AddDrive("C:\\", minimumFreeMegabytes: 512),
        name: "disk-storage");
```

---

## 4. Health Check Response Format

### Response Đơn Giản (Mặc Định)

```
GET /health
200 OK

Healthy
```

### Response JSON Chi Tiết

```csharp
app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";

        var result = new
        {
            status = report.Status.ToString(),
            duration = report.TotalDuration,
            timestamp = DateTime.UtcNow,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration,
                data = e.Value.Data,
                exception = e.Value.Exception?.Message
            })
        };

        await context.Response.WriteAsJsonAsync(result);
    }
});
```

Response mẫu:

```json
{
  "status": "Healthy",
  "duration": "00:00:00.1234567",
  "timestamp": "2026-06-02T03:00:00Z",
  "checks": [
    {
      "name": "database",
      "status": "Healthy",
      "description": "Database hoạt động bình thường",
      "duration": "00:00:00.0500000",
      "data": {
        "userCount": 1523,
        "timestamp": "2026-06-02T03:00:00Z"
      }
    },
    {
      "name": "redis",
      "status": "Healthy",
      "description": null,
      "duration": "00:00:00.0020000",
      "data": {}
    },
    {
      "name": "stripe",
      "status": "Degraded",
      "description": "Response time > 3000ms",
      "duration": "00:00:03.1000000",
      "data": {}
    }
  ]
}
```

---

## 5. Health Checks trong Kubernetes

### Cấu Hình Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: myapp:latest
          ports:
            - containerPort: 8080

          # Startup Probe — chờ app khởi động xong
          # Kubernetes không gửi traffic và không restart cho đến khi startup pass
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10  # Chờ 10s trước khi bắt đầu probe
            periodSeconds: 10        # Kiểm tra mỗi 10s
            failureThreshold: 30     # Cho phép fail 30 lần = 5 phút để khởi động

          # Liveness Probe — app còn sống không?
          # Nếu fail → Kubernetes restart pod
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3      # Fail 3 lần liên tiếp → restart
            timeoutSeconds: 5

          # Readiness Probe — app sẵn sàng nhận traffic không?
          # Nếu fail → Kubernetes ngừng routing traffic đến pod này
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 10
```

### Phân Biệt Liveness vs Readiness

```
Kịch bản 1: Database connection pool cạn kiệt
- Liveness:  HEALTHY (app còn sống, process đang chạy)
- Readiness: UNHEALTHY (app không thể xử lý request)
- Kubernetes: ngừng routing đến pod này, nhưng KHÔNG restart

Kịch bản 2: Deadlock trong code, app bị treo
- Liveness:  UNHEALTHY (app không respond)
- Readiness: UNHEALTHY
- Kubernetes: restart pod

Kịch bản 3: App đang warm up (load cache, init connections)
- Startup:   UNHEALTHY → HEALTHY sau khi xong
- Kubernetes: chờ, không gửi traffic, không restart
```

---

## 6. Health Check UI — Giao Diện Trực Quan

```csharp
// Cài package
// dotnet add package AspNetCore.HealthChecks.UI
// dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
// dotnet add package AspNetCore.HealthChecks.UI.Client

builder.Services.AddHealthChecks()
    .AddSqlServer(config.GetConnectionString("Default")!)
    .AddRedis(config["Redis"]!);

builder.Services.AddHealthChecksUI(opt =>
{
    opt.SetEvaluationTimeInSeconds(30);  // Poll mỗi 30 giây
    opt.MaximumHistoryEntriesPerEndpoint(50);
    opt.AddHealthCheckEndpoint("API", "/health/detail");
})
.AddInMemoryStorage();

// ...

app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";  // Dashboard tại /health-ui
});
```

---

## 7. Bảo Mật Health Check Endpoint

Health check thường chứa thông tin nhạy cảm (connection string, version). Cần bảo vệ:

```csharp
// Chỉ cho phép từ internal network
app.MapHealthChecks("/health/detail")
    .RequireHost("*.internal.company.com", "localhost")
    .AllowAnonymous();  // Cho phép anonymous nhưng restrict host

// Hoặc yêu cầu auth
app.MapHealthChecks("/health/admin")
    .RequireAuthorization("HealthCheckPolicy");

// Đặt health check trên port riêng (tốt nhất cho production)
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenLocalhost(8080);  // Public traffic
    options.ListenLocalhost(8081);  // Health check + internal
});

app.MapHealthChecks("/health").RequireHost("*:8081");
```

---

## 8. Health Check Trong Background Service

```csharp
// Background service tự báo cáo trạng thái
public class DataSyncService : BackgroundService, IHealthCheck
{
    private volatile bool _isHealthy = true;
    private volatile string _statusMessage = "Service đang chạy";

    // IHealthCheck implementation
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        return Task.FromResult(_isHealthy
            ? HealthCheckResult.Healthy(_statusMessage)
            : HealthCheckResult.Unhealthy(_statusMessage));
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await SyncDataAsync(stoppingToken);
                _isHealthy = true;
                _statusMessage = $"Last sync: {DateTime.UtcNow:HH:mm:ss}";
            }
            catch (Exception ex)
            {
                _isHealthy = false;
                _statusMessage = $"Sync failed: {ex.Message}";
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

// Đăng ký
builder.Services.AddSingleton<DataSyncService>();
builder.Services.AddHostedService(sp => sp.GetRequiredService<DataSyncService>());
builder.Services.AddHealthChecks()
    .AddCheck<DataSyncService>("data-sync", tags: new[] { "background" });
```

---

## ✅ Checklist

- [ ] Phân biệt liveness, readiness, startup probe và khi nào dùng mỗi loại
- [ ] Cài đặt health checks cho database, redis, external APIs
- [ ] Viết custom `IHealthCheck` với proper error handling
- [ ] Cấu hình tags để tách liveness và readiness endpoints
- [ ] Trả JSON chi tiết cho `/health/detail`
- [ ] Bảo vệ health check endpoint khỏi truy cập public
- [ ] Cấu hình Kubernetes probes đúng `initialDelaySeconds`, `periodSeconds`, `failureThreshold`
