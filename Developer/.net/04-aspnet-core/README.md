# ASP.NET Core — Framework Web Hiện Đại của Microsoft

> ASP.NET Core là framework web **cross-platform** (đa nền tảng), **open-source** (mã nguồn mở), **high-performance** (hiệu năng cao) để xây dựng ứng dụng web hiện đại, API, và microservices. Đây là nền tảng cốt lõi mà mọi .NET Backend Developer phải thành thạo.

---

## 📋 Tổng Quan Nhanh

| Khái Niệm | Ý Nghĩa | Khi Nào Dùng |
| --------- | ------- | ------------ |
| Middleware Pipeline | Chuỗi xử lý request/response | Mọi request qua đây |
| Dependency Injection | DI — Tiêm phụ thuộc tích hợp sẵn | Quản lý lifetime của service |
| Routing | Định tuyến URL đến handler | Map URL → method |
| Minimal APIs | API gọn nhẹ, ít boilerplate | API đơn giản, microservice |
| Model Binding | Ràng buộc dữ liệu từ request | Deserialize input tự động |
| Filters | Xử lý cross-cutting concerns | Auth, logging, exception |
| Configuration | Quản lý cấu hình ứng dụng | Settings, secrets |
| Health Checks | Kiểm tra tình trạng dịch vụ | Kubernetes liveness/readiness |

---

## 📁 Cấu Trúc Files

| File | Chủ Đề | Độ Khó |
| ---- | ------- | ------ |
| [1-middleware-pipeline.md](1-middleware-pipeline.md) | Request pipeline, custom middleware, short-circuit | ⭐⭐ |
| [2-dependency-injection.md](2-dependency-injection.md) | Singleton/Scoped/Transient, lifetime pitfalls | ⭐⭐ |
| [3-routing.md](3-routing.md) | Attribute routing, conventional routing, route constraints | ⭐⭐ |
| [4-minimal-apis.md](4-minimal-apis.md) | Minimal API vs Controllers, endpoint filters | ⭐⭐ |
| [5-model-binding-validation.md](5-model-binding-validation.md) | Model binding, Data Annotations, FluentValidation | ⭐⭐ |
| [6-filters.md](6-filters.md) | Action, Exception, Authorization, Resource filters | ⭐⭐⭐ |
| [7-configuration.md](7-configuration.md) | appsettings, IOptions\<T\>, environment variables, secrets | ⭐⭐ |
| [8-health-checks.md](8-health-checks.md) | Health endpoint, liveness, readiness, dependency checks | ⭐⭐ |

---

## 🎯 Kiến Trúc Tổng Thể

### Luồng Xử Lý Request trong ASP.NET Core

```
Client gửi HTTP Request
        ↓
[Kestrel / IIS / Nginx]  ← Web Server / Reverse Proxy
        ↓
[Middleware 1: Exception Handling]
        ↓
[Middleware 2: HTTPS Redirection]
        ↓
[Middleware 3: Static Files]
        ↓
[Middleware 4: Routing]
        ↓
[Middleware 5: Authentication]
        ↓
[Middleware 6: Authorization]
        ↓
[Middleware 7: Endpoint — Controller / Minimal API]
        ↓
Response trả về Client (ngược chiều qua các middleware)
```

### Hosting Model — Mô Hình Hosting

```
Program.cs (điểm khởi đầu)
    ↓
WebApplication.CreateBuilder(args)  ← IHostBuilder — Bộ xây dựng host
    ↓
builder.Services.Add*(...)          ← Đăng ký Dependency Injection
    ↓
var app = builder.Build()           ← Tạo WebApplication
    ↓
app.Use*(...)                       ← Cấu hình Middleware Pipeline
    ↓
app.Run()                           ← Khởi động server, lắng nghe request
```

### Ví Dụ Program.cs Đầy Đủ (ASP.NET Core 6+)

```csharp
var builder = WebApplication.CreateBuilder(args);

// === Đăng ký Services (Dependency Injection) ===
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Custom services
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();

// EF Core
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt => { /* cấu hình */ });

// Health Checks
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Default"));

var app = builder.Build();

// === Cấu hình Middleware Pipeline ===
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();        // Redirect HTTP → HTTPS
app.UseStaticFiles();             // Serve file tĩnh từ wwwroot/
app.UseRouting();                 // Kích hoạt routing
app.UseAuthentication();          // Xác thực (phải trước Authorization)
app.UseAuthorization();           // Phân quyền

app.MapControllers();             // Map Controller routes
app.MapHealthChecks("/health");   // Health check endpoint

app.Run();
```

---

## 🔑 Khái Niệm Cốt Lõi Cần Nắm

### 1. Middleware — Phần Mềm Trung Gian

**Middleware** là component xử lý HTTP request và response. Chúng được kết nối thành **pipeline** — chuỗi xử lý. Mỗi middleware có thể:
- Xử lý request và **chuyển tiếp** (call next)
- Xử lý request và **dừng lại** (short-circuit — ngắn mạch)
- Thay đổi request **trước khi** chuyển tiếp
- Thay đổi response **sau khi** nhận về từ middleware tiếp theo

```
Request  →  [MW1] → [MW2] → [MW3] → Handler
Response ←  [MW1] ← [MW2] ← [MW3] ← Handler
```

### 2. Dependency Injection — Tiêm Phụ Thuộc

ASP.NET Core có **IoC container** (Inversion of Control — Đảo Ngược Điều Khiển) tích hợp sẵn. Ba lifetime:
- **Singleton** — một instance duy nhất cho toàn bộ app
- **Scoped** — một instance cho mỗi HTTP request
- **Transient** — tạo mới mỗi lần resolve

### 3. Configuration System — Hệ Thống Cấu Hình

Đọc từ nhiều nguồn theo thứ tự ưu tiên:
```
appsettings.json
    ← appsettings.{Environment}.json
        ← Environment Variables (biến môi trường)
            ← Command Line Arguments
                ← User Secrets (development only)
```

---

## 💡 Điểm Khác Biệt ASP.NET Core vs ASP.NET (Classic)

| Điểm | ASP.NET (Classic) | ASP.NET Core |
| ---- | ----------------- | ------------ |
| Nền tảng | Windows only | Cross-platform |
| Hiệu năng | Trung bình | Rất cao (top web framework benchmarks) |
| Hosting | IIS | Kestrel, IIS, Docker, Nginx |
| Pipeline | HttpModule/HttpHandler | Middleware (linh hoạt hơn) |
| DI | Cần cài thêm (Unity, Ninject) | Tích hợp sẵn |
| Open Source | Không | Có (github.com/dotnet/aspnetcore) |

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

1. Middleware pipeline hoạt động như thế nào? Khi nào dùng `Use` vs `Run` vs `Map`?
2. Sự khác biệt giữa Singleton, Scoped, Transient trong DI?
3. Captive dependency là gì? Tại sao Singleton không nên inject Scoped service?
4. Minimal API vs Controller-based API — khi nào dùng cái nào?
5. Action Filter và Middleware khác nhau như thế nào? Khi nào dùng cái nào?
6. `IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>` khác nhau ra sao?
7. Health checks trong Kubernetes: liveness probe vs readiness probe là gì?

---

## 🔗 Liên Kết Nội Bộ

- Xem [01-fundamentals/](../01-fundamentals/) để hiểu C# nền tảng
- Xem [05-entity-framework/](../05-entity-framework/) để thêm data layer
- Xem [08-security/](../08-security/) cho JWT authentication trong ASP.NET Core
- Xem [06-testing/](../06-testing/) để test API với `WebApplicationFactory`
