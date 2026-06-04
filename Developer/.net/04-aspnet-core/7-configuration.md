# 7 — Configuration & Options Pattern (Cấu Hình Ứng Dụng)

> ASP.NET Core có hệ thống cấu hình phân lớp linh hoạt: đọc từ nhiều nguồn (file JSON, biến môi trường, command-line, secrets), merge theo thứ tự ưu tiên, và inject vào app qua `IOptions<T>`. Hiểu đúng cách quản lý cấu hình giúp deploy an toàn và linh hoạt trên nhiều môi trường.

---

## 📋 Tổng Quan Nhanh

| Interface | Khi Nào Dùng | Reload Khi File Thay Đổi |
| --------- | ------------ | ------------------------ |
| `IOptions<T>` | Cấu hình tĩnh, không đổi | Không |
| `IOptionsSnapshot<T>` | Per-request reload (Scoped) | Có |
| `IOptionsMonitor<T>` | Real-time reload (Singleton) | Có (callback) |

---

## 1. Nguồn Cấu Hình và Thứ Tự Ưu Tiên

ASP.NET Core đọc cấu hình từ nhiều nguồn. **Nguồn sau ghi đè nguồn trước:**

```
Thứ tự ưu tiên (thấp → cao):

1. appsettings.json                    ← Cấu hình mặc định
2. appsettings.{Environment}.json      ← Override theo môi trường
3. User Secrets                        ← Development only, không commit lên git
4. Environment Variables               ← Production secrets, Docker, Kubernetes
5. Command Line Arguments              ← Override từ lệnh chạy
```

```csharp
// Default configuration setup trong WebApplication.CreateBuilder
var builder = WebApplication.CreateBuilder(args);
// ASP.NET Core tự động thêm tất cả nguồn trên theo thứ tự này
// Bạn có thể thêm thêm nguồn:
builder.Configuration
    .AddJsonFile("custom-settings.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables(prefix: "MYAPP_")
    .AddCommandLine(args);
```

---

## 2. appsettings.json — Cấu Hình Cơ Bản

```json
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;Trusted_Connection=True;",
    "Redis": "localhost:6379"
  },
  "Jwt": {
    "Secret": "dev-secret-key-not-for-production",
    "Issuer": "https://myapp.dev",
    "Audience": "myapp-users",
    "ExpiresInMinutes": 60
  },
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "FromAddress": "noreply@myapp.dev",
    "EnableSsl": true
  },
  "Features": {
    "EnableNewDashboard": false,
    "MaxUploadSizeMb": 10
  }
}
```

```json
// appsettings.Development.json — chỉ dùng khi ASPNETCORE_ENVIRONMENT=Development
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "MyApp": "Debug"
    }
  },
  "Features": {
    "EnableNewDashboard": true   // Bật feature trong dev
  }
}
```

```json
// appsettings.Production.json — ghi đè cho Production
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

---

## 3. IConfiguration — Đọc Cấu Hình Trực Tiếp

```csharp
// Inject IConfiguration
public class SomeService
{
    private readonly IConfiguration _config;

    public SomeService(IConfiguration config)
    {
        _config = config;
    }

    public void UseConfig()
    {
        // Đọc từng giá trị
        var connectionString = _config.GetConnectionString("Default");
        var secret = _config["Jwt:Secret"];               // Dùng : để đi sâu
        var port = _config.GetValue<int>("Email:SmtpPort"); // Typed read

        // Đọc section
        var jwtSection = _config.GetSection("Jwt");
        var issuer = jwtSection["Issuer"];

        // Kiểm tra tồn tại
        var exists = _config.GetSection("OptionalFeature").Exists();
    }
}
```

**Lưu ý:** Nên dùng `IOptions<T>` thay vì `IConfiguration` trực tiếp trong production code vì:
- Strongly-typed, IntelliSense hỗ trợ
- Validation tại startup
- Testable (mock dễ hơn)

---

## 4. Options Pattern — Strongly-Typed Configuration

### Bước 1: Định Nghĩa Options Class

```csharp
// Convention: tên class = tên section trong appsettings
public class JwtOptions
{
    // Dùng làm section name trong Configure<T>
    public const string SectionName = "Jwt";

    [Required]
    public string Secret { get; set; } = string.Empty;

    [Required]
    public string Issuer { get; set; } = string.Empty;

    public string Audience { get; set; } = string.Empty;

    [Range(1, 43200)]  // 1 phút đến 30 ngày
    public int ExpiresInMinutes { get; set; } = 60;
}

public class EmailOptions
{
    public const string SectionName = "Email";

    [Required]
    public string SmtpHost { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int SmtpPort { get; set; } = 587;

    [Required, EmailAddress]
    public string FromAddress { get; set; } = string.Empty;

    public bool EnableSsl { get; set; } = true;
}
```

### Bước 2: Đăng Ký

```csharp
// Program.cs — đăng ký với validation
builder.Services.AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)  // Bind từ appsettings section "Jwt"
    .ValidateDataAnnotations()                   // Validate bằng Data Annotations
    .ValidateOnStart();                          // Validate ngay khi app khởi động (fail fast)

builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .Validate(opt => !string.IsNullOrEmpty(opt.SmtpHost), "SmtpHost là bắt buộc")
    .ValidateOnStart();

// Hoặc cách cũ (không có validation)
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection(JwtOptions.SectionName));
```

### Bước 3: Inject và Sử Dụng

```csharp
// IOptions<T> — snapshot tại thời điểm khởi động, không reload
public class JwtTokenService
{
    private readonly JwtOptions _options;

    public JwtTokenService(IOptions<JwtOptions> options)
    {
        _options = options.Value;  // .Value lấy instance cấu hình
    }

    public string GenerateToken(string userId)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Secret));
        // ...
    }
}
```

---

## 5. Ba Biến Thể IOptions

### IOptions\<T\> — Singleton, Không Reload

```csharp
public class StaticConfigService
{
    private readonly MyOptions _options;

    // IOptions<T> là Singleton — .Value trả cùng instance mãi mãi
    public StaticConfigService(IOptions<MyOptions> options)
    {
        _options = options.Value;
    }
}
```

**Dùng khi:** Cấu hình không bao giờ thay đổi lúc runtime (connection strings, JWT secret).

### IOptionsSnapshot\<T\> — Scoped, Reload Mỗi Request

```csharp
public class DynamicPerRequestService
{
    private readonly MyOptions _options;

    // IOptionsSnapshot<T> là Scoped — snapshot mới mỗi HTTP request
    // Phản ánh thay đổi appsettings.json kể từ lần reload gần nhất
    public DynamicPerRequestService(IOptionsSnapshot<MyOptions> options)
    {
        _options = options.Value;
    }
}
```

**Dùng khi:** Cần cấu hình cập nhật theo từng request, nhưng ổn định trong suốt request đó.

### IOptionsMonitor\<T\> — Singleton, Real-Time Reload

```csharp
public class BackgroundRefreshService : BackgroundService
{
    private readonly IOptionsMonitor<FeatureFlags> _monitor;
    private readonly ILogger<BackgroundRefreshService> _logger;

    public BackgroundRefreshService(
        IOptionsMonitor<FeatureFlags> monitor,
        ILogger<BackgroundRefreshService> logger)
    {
        _monitor = monitor;

        // Đăng ký callback khi config thay đổi
        _monitor.OnChange(newOptions =>
        {
            logger.LogInformation("Feature flags updated: {@Options}", newOptions);
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // CurrentValue luôn trả giá trị mới nhất
            var flags = _monitor.CurrentValue;

            if (flags.EnableAggressiveRefresh)
                await DoAggressiveRefreshAsync();

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

**Dùng khi:** Background service, Singleton service cần cấu hình thay đổi real-time (feature flags).

---

## 6. Validation Options Khi Startup

```csharp
// Tùy chỉnh validation logic
builder.Services.AddOptions<DatabaseOptions>()
    .BindConfiguration("Database")
    .Validate(options =>
    {
        if (options.MaxPoolSize < options.MinPoolSize)
            return false;

        if (options.CommandTimeout < 1 || options.CommandTimeout > 300)
            return false;

        return true;
    }, "Database pool size và timeout không hợp lệ")
    .ValidateOnStart();

// Implement IValidateOptions<T> cho validation phức tạp
public class DatabaseOptionsValidator : IValidateOptions<DatabaseOptions>
{
    public ValidateOptionsResult Validate(string? name, DatabaseOptions options)
    {
        var failures = new List<string>();

        if (string.IsNullOrEmpty(options.ConnectionString))
            failures.Add("ConnectionString là bắt buộc");

        if (options.MaxPoolSize < 1)
            failures.Add("MaxPoolSize phải >= 1");

        if (options.MaxPoolSize < options.MinPoolSize)
            failures.Add("MaxPoolSize phải >= MinPoolSize");

        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

// Đăng ký validator
builder.Services.AddSingleton<IValidateOptions<DatabaseOptions>, DatabaseOptionsValidator>();
```

---

## 7. User Secrets — Bảo Mật Cấu Hình Dev

User Secrets lưu thông tin nhạy cảm **ngoài source code**, chỉ dùng trong Development:

```bash
# Khởi tạo user secrets cho project
dotnet user-secrets init

# Thêm secret
dotnet user-secrets set "Jwt:Secret" "super-secret-key-for-dev-only"
dotnet user-secrets set "Email:Password" "dev-email-password"
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;..."

# Xem tất cả secrets
dotnet user-secrets list

# Xóa secret
dotnet user-secrets remove "Email:Password"
```

Secrets lưu tại:
- Windows: `%APPDATA%\Microsoft\UserSecrets\{userSecretsId}\secrets.json`
- Linux/Mac: `~/.microsoft/usersecrets/{userSecretsId}/secrets.json`

```json
// secrets.json (KHÔNG commit file này lên git)
{
  "Jwt:Secret": "super-secret-key-for-dev-only",
  "Email:Password": "dev-email-password"
}
```

---

## 8. Environment Variables — Biến Môi Trường

Dùng cho **production secrets** và cấu hình trong Docker/Kubernetes:

```bash
# Convention: dùng __ (double underscore) thay cho : trong nested keys
# VD: "Jwt:Secret" → "Jwt__Secret" (environment variable)

# Linux/Mac
export Jwt__Secret="production-secret"
export ConnectionStrings__Default="Server=prod-db;..."
export ASPNETCORE_ENVIRONMENT="Production"

# Windows PowerShell
$env:Jwt__Secret = "production-secret"
$env:ASPNETCORE_ENVIRONMENT = "Production"
```

```yaml
# Docker Compose
services:
  api:
    image: myapp:latest
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - Jwt__Secret=${JWT_SECRET}          # Từ .env file
      - ConnectionStrings__Default=${DB_CONNECTION}
```

```yaml
# Kubernetes ConfigMap + Secret
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Jwt__ExpiresInMinutes: "60"

---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:
  Jwt__Secret: "production-jwt-secret"
  ConnectionStrings__Default: "Server=prod-db;..."
```

---

## 9. PostConfigure — Override Sau Khi Configure

```csharp
// Configure trước
builder.Services.Configure<FeatureOptions>(
    builder.Configuration.GetSection("Features"));

// PostConfigure chạy sau tất cả Configure calls
// Hữu ích cho: thêm giá trị computed, validate, override một phần
builder.Services.PostConfigure<FeatureOptions>(options =>
{
    // Tự động bật feature nếu không phải production
    if (builder.Environment.IsDevelopment())
    {
        options.EnableDebugPanel = true;
        options.ShowDetailedErrors = true;
    }
});
```

---

## 10. Named Options — Options Có Tên

Khi cần nhiều instance cấu hình của cùng type:

```csharp
public class SmtpOptions
{
    public string Host { get; set; } = string.Empty;
    public int Port { get; set; }
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}

// Đăng ký nhiều named instances
builder.Services.Configure<SmtpOptions>("Primary",
    builder.Configuration.GetSection("Email:Primary"));
builder.Services.Configure<SmtpOptions>("Backup",
    builder.Configuration.GetSection("Email:Backup"));

// Inject và dùng
public class EmailService
{
    private readonly SmtpOptions _primary;
    private readonly SmtpOptions _backup;

    public EmailService(IOptionsMonitor<SmtpOptions> monitor)
    {
        _primary = monitor.Get("Primary");
        _backup = monitor.Get("Backup");
    }
}
```

---

## ⚠️ Quy Tắc Bảo Mật Cấu Hình

| Môi Trường | Cách Lưu Secret |
| ---------- | --------------- |
| Development | User Secrets |
| Staging/Production | Environment Variables, Azure Key Vault, AWS Secrets Manager |
| Docker | Docker Secrets hoặc Environment Variables |
| Kubernetes | Kubernetes Secrets (+ External Secrets Operator) |

**KHÔNG BAO GIỜ:**
- Commit `appsettings.Production.json` chứa real secrets lên git
- Hardcode secret trong source code
- Để password trong connection string trong appsettings.json

---

## ✅ Checklist

- [ ] Hiểu thứ tự ưu tiên của các nguồn cấu hình
- [ ] Tạo Options class và đăng ký với `ValidateDataAnnotations()` + `ValidateOnStart()`
- [ ] Hiểu sự khác biệt giữa `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`
- [ ] Dùng User Secrets cho development thay vì hardcode
- [ ] Biết cách dùng environment variables với double underscore (`__`)
- [ ] Validate Options tại startup để fail fast
- [ ] Tránh inject `IConfiguration` trực tiếp — dùng Options Pattern
