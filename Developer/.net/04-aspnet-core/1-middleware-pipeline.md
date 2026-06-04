# 1 — Middleware Pipeline (Chuỗi Xử Lý Request)

> Middleware pipeline là xương sống của ASP.NET Core. Mọi HTTP request đều đi qua chuỗi middleware trước khi đến handler và đi ngược lại khi trả về response. Hiểu pipeline giúp bạn kiểm soát toàn bộ vòng đời của request.

---

## 📋 Tổng Quan Nhanh

| Khái Niệm | Giải Thích |
| --------- | ---------- |
| Middleware | Component xử lý HTTP request/response trong pipeline |
| `Use` | Thêm middleware có thể gọi next — middleware tiếp theo |
| `Run` | Thêm terminal middleware — kết thúc pipeline, không gọi next |
| `Map` | Rẽ nhánh pipeline theo URL prefix |
| `MapWhen` | Rẽ nhánh theo điều kiện tùy chỉnh |
| Short-circuit | Ngắn mạch — dừng pipeline sớm, không gọi next |
| Request Delegate | `Func<HttpContext, Task>` — đơn vị xử lý cơ bản |

---

## 🔍 Cơ Chế Hoạt Động

### Mô Hình Pipeline

```
HTTP Request đến
        ↓
┌─────────────────────────────────────────┐
│  Middleware 1 (Exception Handler)        │
│    ↓ next()                             │
│  Middleware 2 (HTTPS Redirect)          │
│    ↓ next()                             │
│  Middleware 3 (Static Files)            │
│    ↓ next()                             │
│  Middleware 4 (Routing)                 │
│    ↓ next()                             │
│  Middleware 5 (Auth)                    │
│    ↓ next()                             │
│  Middleware 6 (Endpoint/Controller)     │
│    ↑ response trả ngược chiều          │
└─────────────────────────────────────────┘
        ↑
HTTP Response trả về
```

**Lưu ý quan trọng:** Request đi **xuống** qua pipeline, response đi **ngược lên**. Middleware có thể xử lý cả hai chiều.

### Ví Dụ: Request/Response Đi Qua Middleware

```csharp
// Middleware này xử lý cả hai chiều
app.Use(async (context, next) =>
{
    // === Xử lý REQUEST (trước khi gọi next) ===
    Console.WriteLine($"→ Request: {context.Request.Method} {context.Request.Path}");
    var stopwatch = Stopwatch.StartNew();

    await next(context);  // Gọi middleware tiếp theo

    // === Xử lý RESPONSE (sau khi next hoàn thành) ===
    stopwatch.Stop();
    Console.WriteLine($"← Response: {context.Response.StatusCode} | {stopwatch.ElapsedMilliseconds}ms");
});
```

---

## 1. Ba Phương Thức Cơ Bản: Use, Run, Map

### `Use` — Middleware Thông Thường

Có thể gọi `next` để chuyển sang middleware tiếp theo, hoặc không gọi để short-circuit.

```csharp
app.Use(async (context, next) =>
{
    // Xử lý trước
    await next(context);   // Gọi middleware tiếp theo
    // Xử lý sau (khi response quay lại)
});
```

### `Run` — Terminal Middleware (Điểm Cuối Pipeline)

Không có tham số `next`, luôn kết thúc pipeline. Đây là điểm cuối của chuỗi.

```csharp
app.Run(async context =>
{
    // Không có next — đây là điểm cuối
    await context.Response.WriteAsync("Hello from terminal middleware!");
});

// Mọi middleware sau app.Run() đều KHÔNG bao giờ được gọi
app.Use(/* này không chạy */);  // ← Dead code!
```

### `Map` — Rẽ Nhánh Theo URL

Tạo sub-pipeline cho các URL prefix cụ thể.

```csharp
// Mọi request có path bắt đầu bằng /api đi vào nhánh này
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        context.Response.Headers.Add("X-Api-Version", "1.0");
        await next(context);
    });

    apiApp.Run(async context =>
    {
        await context.Response.WriteAsync("API response");
    });
});

// Các request khác vẫn đi qua pipeline chính
app.Run(async context =>
{
    await context.Response.WriteAsync("Main pipeline");
});
```

### `MapWhen` — Rẽ Nhánh Theo Điều Kiện

```csharp
// Chỉ request từ mobile user agents đi vào nhánh này
app.MapWhen(
    context => context.Request.Headers["User-Agent"].ToString().Contains("Mobile"),
    mobileApp =>
    {
        mobileApp.Run(async context =>
        {
            await context.Response.WriteAsync("Mobile response");
        });
    }
);
```

---

## 2. Thứ Tự Middleware — Order Matters!

**Thứ tự cực kỳ quan trọng.** Microsoft khuyến nghị thứ tự chuẩn:

```csharp
var app = builder.Build();

// 1. Exception handling — phải đầu tiên để bắt mọi lỗi
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/error");

// 2. HSTS — HTTP Strict Transport Security (chỉ production)
app.UseHsts();

// 3. HTTPS Redirect
app.UseHttpsRedirection();

// 4. Static Files — serve trước routing để tránh overhead
app.UseStaticFiles();

// 5. Cookie Policy
app.UseCookiePolicy();

// 6. Routing — bật endpoint routing
app.UseRouting();

// 7. CORS — Cross-Origin Resource Sharing (sau Routing, trước Auth)
app.UseCors();

// 8. Authentication — xác thực (sau Routing)
app.UseAuthentication();

// 9. Authorization — phân quyền (sau Authentication)
app.UseAuthorization();

// 10. Custom middleware (nếu có)
app.UseRateLimiting();

// 11. Endpoint mapping — cuối cùng
app.MapControllers();
app.MapRazorPages();
app.MapHealthChecks("/health");

app.Run();
```

**Sai thứ tự sẽ gây lỗi phổ biến:**
```csharp
// ❌ SAI: Authorization trước Authentication — auth luôn fail
app.UseAuthorization();
app.UseAuthentication();

// ✅ ĐÚNG:
app.UseAuthentication();
app.UseAuthorization();
```

---

## 3. Custom Middleware — Tự Viết Middleware

### Cách 1: Inline Middleware (nhanh, đơn giản)

```csharp
app.Use(async (context, next) =>
{
    var requestId = Guid.NewGuid().ToString("N")[..8];
    context.Items["RequestId"] = requestId;
    context.Response.Headers.Add("X-Request-Id", requestId);

    await next(context);
});
```

### Cách 2: Middleware Class (được khuyến nghị cho production)

```csharp
// Middleware class phải có constructor nhận RequestDelegate
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    // Constructor Injection — inject qua constructor
    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // InvokeAsync — bắt buộc phải có method này (hoặc Invoke)
    public async Task InvokeAsync(HttpContext context)
    {
        // Trước request
        _logger.LogInformation(
            "→ {Method} {Path} | IP: {IP}",
            context.Request.Method,
            context.Request.Path,
            context.Connection.RemoteIpAddress);

        var sw = Stopwatch.StartNew();

        try
        {
            await _next(context);  // Gọi middleware tiếp theo
        }
        finally
        {
            sw.Stop();
            // Sau response
            _logger.LogInformation(
                "← {StatusCode} | {ElapsedMs}ms",
                context.Response.StatusCode,
                sw.ElapsedMilliseconds);
        }
    }
}

// Extension method để đăng ký gọn gàng
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder app)
        => app.UseMiddleware<RequestLoggingMiddleware>();
}

// Sử dụng trong Program.cs
app.UseRequestLogging();
```

### Cách 3: IMiddleware Interface (Strongly-Typed DI)

Khi middleware cần **Scoped service** (ví dụ: DbContext), dùng `IMiddleware` để DI inject đúng lifetime:

```csharp
// IMiddleware được DI container quản lý lifetime
public class AuditMiddleware : IMiddleware
{
    private readonly AuditDbContext _db;  // Scoped service!

    public AuditMiddleware(AuditDbContext db)
    {
        _db = db;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        await next(context);

        _db.AuditLogs.Add(new AuditLog
        {
            Path = context.Request.Path,
            StatusCode = context.Response.StatusCode,
            Timestamp = DateTime.UtcNow
        });
        await _db.SaveChangesAsync();
    }
}

// Đăng ký — phải đăng ký như một service
builder.Services.AddScoped<AuditMiddleware>();

// Sử dụng
app.UseMiddleware<AuditMiddleware>();
```

**Điểm khác biệt quan trọng:**

| | Conventional Middleware | IMiddleware |
| - | ----------------------- | ----------- |
| DI lifetime | Singleton (constructor) + Transient (InvokeAsync method params) | Scoped (mặc định) |
| Đăng ký service | Không cần | Bắt buộc `services.AddScoped<T>()` |
| Inject Scoped service | Chỉ qua method params, không qua constructor | Qua constructor OK |

---

## 4. Short-Circuit — Ngắn Mạch Pipeline

Short-circuit là khi middleware **không gọi `next`**, dừng pipeline sớm. Thường dùng cho:
- Trả cache response
- Reject request không hợp lệ
- Health check nhanh

```csharp
app.Use(async (context, next) =>
{
    // Short-circuit: kiểm tra API key
    if (!context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey)
        || apiKey != "secret-key")
    {
        context.Response.StatusCode = StatusCodes.Status401Unauthorized;
        await context.Response.WriteAsJsonAsync(new { error = "Invalid API key" });
        return;  // KHÔNG gọi next — pipeline dừng tại đây
    }

    await next(context);  // Chỉ gọi next khi API key hợp lệ
});
```

---

## 5. Dependency Injection Trong Middleware

### Inject vào Constructor (Singleton dependencies)

```csharp
public class FeatureFlagMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _config;       // Singleton — OK trong constructor
    private readonly IMemoryCache _cache;          // Singleton — OK trong constructor

    public FeatureFlagMiddleware(
        RequestDelegate next,
        IConfiguration config,
        IMemoryCache cache)
    {
        _next = next;
        _config = config;
        _cache = cache;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        await _next(context);
    }
}
```

### Inject vào InvokeAsync (Scoped dependencies)

```csharp
public class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public TenantMiddleware(RequestDelegate next) => _next = next;

    // Scoped services được inject vào InvokeAsync, KHÔNG phải constructor
    public async Task InvokeAsync(
        HttpContext context,
        ITenantService tenantService,   // Scoped — inject qua method
        ILogger<TenantMiddleware> logger)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();
        if (tenantId is not null)
        {
            var tenant = await tenantService.GetTenantAsync(tenantId);
            context.Items["Tenant"] = tenant;
        }

        await _next(context);
    }
}
```

---

## 6. Exception Handling Middleware

### Middleware Bắt Ngoại Lệ Tùy Chỉnh

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Resource not found");
            await WriteErrorResponse(context, StatusCodes.Status404NotFound, ex.Message);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation failed");
            await WriteErrorResponse(context, StatusCodes.Status400BadRequest, ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error");
            await WriteErrorResponse(context, StatusCodes.Status500InternalServerError,
                "An unexpected error occurred");
        }
    }

    private static async Task WriteErrorResponse(HttpContext context, int statusCode, string message)
    {
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new
        {
            error = message,
            timestamp = DateTime.UtcNow,
            traceId = context.TraceIdentifier
        });
    }
}
```

### Sử dụng `UseExceptionHandler` Tích Hợp Sẵn

```csharp
// Cách đơn giản hơn: dùng built-in exception handler
app.UseExceptionHandler(exApp =>
{
    exApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionFeature?.Error;

        context.Response.StatusCode = exception switch
        {
            NotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedException => StatusCodes.Status401Unauthorized,
            _ => StatusCodes.Status500InternalServerError
        };

        await context.Response.WriteAsJsonAsync(new
        {
            error = exception?.Message ?? "Internal server error"
        });
    });
});
```

---

## 7. Đọc/Ghi Request & Response Body

**Cẩn thận:** `Request.Body` và `Response.Body` là stream — chỉ đọc được một lần!

```csharp
// Đọc Request Body trong middleware (cần enable buffering)
app.Use(async (context, next) =>
{
    // EnableBuffering — cho phép đọc body nhiều lần
    context.Request.EnableBuffering();

    var body = await new StreamReader(context.Request.Body).ReadToEndAsync();
    context.Request.Body.Position = 0;  // Reset về đầu để các handler khác đọc được

    // Log hoặc xử lý body
    if (body.Length > 0)
        Console.WriteLine($"Request body: {body[..Math.Min(100, body.Length)]}...");

    await next(context);
});
```

---

## ⚠️ Các Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
| --- | ----------- | --------- |
| Response đã bắt đầu nhưng vẫn ghi thêm | Gọi `WriteAsync` sau khi status code đã được gửi | Kiểm tra `context.Response.HasStarted` |
| Middleware không chạy | Sai thứ tự hoặc đặt sau `app.Run()` | Đảm bảo thứ tự đúng |
| Scoped service inject vào constructor | Middleware có lifetime Singleton, gây captive dependency | Inject qua `InvokeAsync` hoặc dùng `IMiddleware` |
| Đọc body bị lỗi "stream already consumed" | Đọc body không reset position | Dùng `EnableBuffering()` và reset `Position = 0` |

---

## ✅ Checklist

- [ ] Hiểu sự khác biệt giữa `Use`, `Run`, `Map`
- [ ] Biết thứ tự chuẩn của middleware pipeline
- [ ] Viết được custom middleware class với DI
- [ ] Hiểu khi nào dùng conventional middleware vs `IMiddleware`
- [ ] Biết cách short-circuit pipeline đúng cách
- [ ] Xử lý exception tập trung qua middleware
- [ ] Hiểu cách inject Scoped service vào middleware đúng cách
