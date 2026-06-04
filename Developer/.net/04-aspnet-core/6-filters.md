# 6 — Filters (Bộ Lọc)

> Filters — Bộ Lọc — trong ASP.NET Core cho phép chạy code **trước hoặc sau** các giai đoạn cụ thể trong request processing pipeline. Chúng là công cụ để xử lý **cross-cutting concerns** — mối quan tâm cắt ngang — như logging, caching, authentication, authorization mà không cần lặp lại code trong mỗi action.

---

## 📋 Tổng Quan Nhanh

| Loại Filter | Khi Nào Chạy | Dùng Cho |
| ----------- | ------------ | -------- |
| **Authorization Filter** | Đầu tiên, trước tất cả | Kiểm tra quyền truy cập |
| **Resource Filter** | Sau Authorization, bao quanh Model Binding | Cache, short-circuit trước binding |
| **Action Filter** | Trước/sau Action Method | Logging, transform input/output |
| **Exception Filter** | Khi có exception chưa được xử lý | Xử lý ngoại lệ tập trung |
| **Result Filter** | Trước/sau Result execution | Transform response, add headers |

---

## 1. Vòng Đời Filter — Filter Pipeline

```
HTTP Request
    ↓
[Authorization Filters]         ← Chạy đầu tiên
    ↓
[Resource Filters — OnResourceExecuting]
    ↓
[Model Binding]
    ↓
[Action Filters — OnActionExecuting]
    ↓
[Action Method]
    ↓
[Action Filters — OnActionExecuted]
    ↓
[Exception Filters]             ← Bắt exception từ action
    ↓
[Result Filters — OnResultExecuting]
    ↓
[Result Execution — JSON serialization, view rendering...]
    ↓
[Result Filters — OnResultExecuted]
    ↓
[Resource Filters — OnResourceExecuted]
    ↓
HTTP Response
```

---

## 2. Action Filters — Bộ Lọc Action (Thường Dùng Nhất)

### Implement IActionFilter (Synchronous)

```csharp
public class LogActionFilter : IActionFilter
{
    private readonly ILogger<LogActionFilter> _logger;

    public LogActionFilter(ILogger<LogActionFilter> logger)
    {
        _logger = logger;
    }

    // Chạy TRƯỚC khi action method thực thi
    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation(
            "→ Executing: {Controller}.{Action} | Args: {@Args}",
            context.RouteData.Values["controller"],
            context.RouteData.Values["action"],
            context.ActionArguments);
    }

    // Chạy SAU khi action method thực thi
    public void OnActionExecuted(ActionExecutedContext context)
    {
        _logger.LogInformation(
            "← Executed: {Controller}.{Action} | Status: {Status}",
            context.RouteData.Values["controller"],
            context.RouteData.Values["action"],
            (context.Result as ObjectResult)?.StatusCode ?? 200);
    }
}
```

### Implement IAsyncActionFilter (Async — Khuyến Nghị)

```csharp
public class PerformanceFilter : IAsyncActionFilter
{
    private readonly ILogger<PerformanceFilter> _logger;
    private const int SlowRequestThresholdMs = 500;

    public PerformanceFilter(ILogger<PerformanceFilter> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)   // next = phần còn lại của pipeline
    {
        var sw = Stopwatch.StartNew();

        var executedContext = await next();  // Gọi action + các filter còn lại

        sw.Stop();

        if (sw.ElapsedMilliseconds > SlowRequestThresholdMs)
        {
            _logger.LogWarning(
                "⚠️ Slow request: {Controller}.{Action} took {ElapsedMs}ms",
                context.RouteData.Values["controller"],
                context.RouteData.Values["action"],
                sw.ElapsedMilliseconds);
        }

        // Kiểm tra exception xảy ra trong action
        if (executedContext.Exception is not null)
        {
            _logger.LogError(executedContext.Exception,
                "Action threw an exception after {ElapsedMs}ms",
                sw.ElapsedMilliseconds);
        }
    }
}
```

### Short-Circuit Trong Action Filter

```csharp
public class MaintenanceModeFilter : IActionFilter
{
    private readonly IConfiguration _config;

    public MaintenanceModeFilter(IConfiguration config)
    {
        _config = config;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (_config.GetValue<bool>("App:MaintenanceMode"))
        {
            // Short-circuit: set Result để dừng pipeline, không gọi action
            context.Result = new ObjectResult(new
            {
                message = "Hệ thống đang bảo trì. Vui lòng thử lại sau.",
                estimatedResume = DateTime.UtcNow.AddHours(2)
            })
            {
                StatusCode = StatusCodes.Status503ServiceUnavailable
            };
        }
    }

    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

---

## 3. Exception Filters — Bộ Lọc Ngoại Lệ

```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ApiExceptionFilter> _logger;

    public ApiExceptionFilter(ILogger<ApiExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");

        var (statusCode, message) = context.Exception switch
        {
            NotFoundException ex => (404, ex.Message),
            ValidationException ex => (400, ex.Message),
            UnauthorizedException ex => (401, ex.Message),
            ForbiddenException ex => (403, ex.Message),
            ConflictException ex => (409, ex.Message),
            _ => (500, "Đã xảy ra lỗi. Vui lòng thử lại sau.")
        };

        context.Result = new ObjectResult(new ProblemDetails
        {
            Status = statusCode,
            Title = message,
            Instance = context.HttpContext.Request.Path,
            Extensions = { ["traceId"] = context.HttpContext.TraceIdentifier }
        })
        {
            StatusCode = statusCode
        };

        context.ExceptionHandled = true;  // Đánh dấu đã xử lý — không propagate lên
    }
}
```

---

## 4. Authorization Filters — Bộ Lọc Phân Quyền

```csharp
// Custom Authorization Filter — chạy trước mọi filter khác
public class ApiKeyAuthorizationFilter : IAuthorizationFilter
{
    private readonly IApiKeyService _apiKeyService;

    public ApiKeyAuthorizationFilter(IApiKeyService apiKeyService)
    {
        _apiKeyService = apiKeyService;
    }

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        // Nếu endpoint có [AllowAnonymous] → bỏ qua
        if (context.ActionDescriptor.EndpointMetadata
            .OfType<IAllowAnonymous>().Any())
            return;

        if (!context.HttpContext.Request.Headers.TryGetValue("X-Api-Key", out var apiKey)
            || !_apiKeyService.IsValidKey(apiKey!))
        {
            context.Result = new UnauthorizedObjectResult(new
            {
                error = "API key không hợp lệ hoặc thiếu"
            });
        }
    }
}
```

---

## 5. Resource Filters — Bộ Lọc Tài Nguyên

Resource Filters bao quanh cả Model Binding — hữu ích để **cache response** trước khi binding/action chạy:

```csharp
public class ResponseCacheFilter : IAsyncResourceFilter
{
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _duration;

    public ResponseCacheFilter(IMemoryCache cache, TimeSpan duration)
    {
        _cache = cache;
        _duration = duration;
    }

    public async Task OnResourceExecutionAsync(
        ResourceExecutingContext context,
        ResourceExecutionDelegate next)
    {
        var cacheKey = $"response:{context.HttpContext.Request.Path}:{context.HttpContext.Request.QueryString}";

        // Kiểm tra cache — nếu có, trả ngay không cần qua action
        if (_cache.TryGetValue(cacheKey, out IActionResult? cachedResult))
        {
            context.Result = cachedResult;  // Short-circuit — bỏ qua binding + action
            return;
        }

        var executedContext = await next();  // Chạy Model Binding + Action

        // Lưu vào cache nếu thành công
        if (executedContext.Result is OkObjectResult okResult)
        {
            _cache.Set(cacheKey, executedContext.Result, _duration);
        }
    }
}
```

---

## 6. Result Filters — Bộ Lọc Kết Quả

```csharp
// Thêm headers vào mọi response
public class SecurityHeadersFilter : IResultFilter
{
    public void OnResultExecuting(ResultExecutingContext context)
    {
        var headers = context.HttpContext.Response.Headers;
        headers.Append("X-Content-Type-Options", "nosniff");
        headers.Append("X-Frame-Options", "DENY");
        headers.Append("X-XSS-Protection", "1; mode=block");
        headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    }

    public void OnResultExecuted(ResultExecutedContext context) { }
}
```

---

## 7. Áp Dụng Filter — 3 Mức

### Mức Global — Toàn Bộ App

```csharp
// Áp dụng cho tất cả controllers và actions
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ApiExceptionFilter>();       // Thêm qua type
    options.Filters.Add<SecurityHeadersFilter>();
    options.Filters.Add(new LogActionFilter(...));   // Thêm qua instance
});
```

### Mức Controller — Một Controller

```csharp
// Áp dụng cho tất cả actions trong controller này
[ApiController]
[Route("api/[controller]")]
[ServiceFilter(typeof(LogActionFilter))]     // Dùng khi filter cần DI
[TypeFilter(typeof(PerformanceFilter))]      // TypeFilter tạo instance mới mỗi lần
public class OrdersController : ControllerBase
{
    // Mọi action đều được log và đo performance
}
```

### Mức Action — Một Action Cụ Thể

```csharp
[HttpPost]
[ServiceFilter(typeof(ValidationFilter))]    // Chỉ action này mới validate
[TypeFilter(typeof(AuditFilter))]
public async Task<IActionResult> Create([FromBody] CreateOrderRequest request)
{
    return Ok();
}
```

---

## 8. ServiceFilter vs TypeFilter vs Attribute Filter

| | Cách Tạo | DI Support | Khi Nào Dùng |
| - | -------- | ---------- | ------------ |
| `[ServiceFilter(typeof(F))]` | DI container | Đầy đủ | Filter cần inject services |
| `[TypeFilter(typeof(F))]` | Activator + DI | Đầy đủ + có thể pass args | Filter cần args tùy chỉnh |
| Filter Attribute tự viết | New() | Không tự động | Filter đơn giản, không cần DI |
| Global `options.Filters.Add` | DI hoặc instance | Tùy | Áp dụng toàn app |

### Filter Attribute Có DI Qua IFilterFactory

```csharp
// Attribute kết hợp với IFilterFactory để hỗ trợ DI
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class CacheResponseAttribute : Attribute, IFilterFactory
{
    public int DurationSeconds { get; set; } = 60;
    public bool IsReusable => false;

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var cache = serviceProvider.GetRequiredService<IMemoryCache>();
        return new ResponseCacheFilter(cache, TimeSpan.FromSeconds(DurationSeconds));
    }
}

// Sử dụng như attribute thông thường
[HttpGet]
[CacheResponse(DurationSeconds = 300)]
public IActionResult GetExpensiveData()
    => Ok(ComputeExpensiveData());
```

---

## 9. Filters vs Middleware — Khi Nào Dùng Cái Nào?

| | Middleware | Filters |
| - | ---------- | ------- |
| **Phạm vi** | Toàn bộ request pipeline | Controller/Action phạm vi |
| **Truy cập action metadata** | Không | Có (`ActionDescriptor`, Route values) |
| **Truy cập ModelState** | Không | Có |
| **Short-circuit** | Có | Có |
| **DI** | Qua constructor (singleton) hoặc InvokeAsync (scoped) | Qua `ServiceFilter` / `TypeFilter` |
| **Dùng cho** | CORS, HTTPS redirect, static files, auth pipeline | Logging action, validate business rules, cache response |

**Quy tắc ngón tay cái:**
- Dùng **Middleware** khi logic không liên quan đến MVC (routing, static files, CORS)
- Dùng **Filters** khi cần truy cập thông tin MVC (action, controller, model state)

---

## ✅ Checklist

- [ ] Hiểu thứ tự chạy của các loại filter
- [ ] Viết được Action Filter bằng `IAsyncActionFilter`
- [ ] Implement Exception Filter để xử lý lỗi tập trung
- [ ] Hiểu sự khác biệt giữa `ServiceFilter` và `TypeFilter`
- [ ] Biết khi nào dùng Middleware vs Filter
- [ ] Dùng `IFilterFactory` để filter attribute hỗ trợ DI
- [ ] Đăng ký filter global, controller-level, action-level đúng cách
