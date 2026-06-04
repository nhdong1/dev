# Quản Lý Cấu Hình — Configuration Management, Secrets, Azure Key Vault

> Quản lý cấu hình là thách thức quan trọng trong các ứng dụng hiện đại: mỗi môi trường (Development, Staging, Production) cần cấu hình khác nhau, trong đó nhiều giá trị là nhạy cảm (passwords, API keys, connection strings). Bài này trình bày cách quản lý cấu hình an toàn trong .NET với ASP.NET Core Configuration, Azure Key Vault, Kubernetes Secrets và best practices cho từng môi trường.

---

## 1. ASP.NET Core Configuration System

### Thứ Tự Ưu Tiên Configuration Providers — Nguồn Cấu Hình

```
Ưu tiên cao nhất (override tất cả bên dưới)
    │
    │  6. Command-line arguments
    │  5. Environment variables
    │  4. User Secrets (Development only)
    │  3. appsettings.{Environment}.json
    │  2. appsettings.json
    │  1. Default values trong code
    │
Ưu tiên thấp nhất
```

```csharp
// Program.cs — thứ tự providers được tự động cấu hình
var builder = WebApplication.CreateBuilder(args);

// WebApplication.CreateBuilder đã thêm providers theo thứ tự:
// 1. appsettings.json
// 2. appsettings.{Environment}.json
// 3. User Secrets (khi Development)
// 4. Environment Variables
// 5. Command-line args

// Thêm Azure Key Vault (ưu tiên cao nhất)
if (builder.Environment.IsProduction())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://myapp-keyvault.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

---

## 2. appsettings.json — Cấu Hình Cơ Bản

### Phân Tầng Cấu Hình Theo Môi Trường

```
appsettings.json                  ← giá trị mặc định, không nhạy cảm
appsettings.Development.json      ← override cho Development
appsettings.Staging.json          ← override cho Staging
appsettings.Production.json       ← override cho Production (ít thông tin, dùng Key Vault)
```

```json
// appsettings.json — cấu hình chung, không chứa secrets
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "App": {
    "Name": "MyApp API",
    "Version": "1.0.0",
    "MaxPageSize": 100
  },
  "Cache": {
    "DefaultExpiryMinutes": 60,
    "SlidingExpiryMinutes": 10
  },
  "Email": {
    "FromAddress": "noreply@myapp.com",
    "FromName": "MyApp"
  }
}
```

```json
// appsettings.Development.json — cấu hình development (an toàn hơn để commit)
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=myapp_dev;Username=postgres;Password=devpassword"
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  },
  "Jwt": {
    "SecretKey": "dev-secret-key-not-used-in-production-must-be-at-least-32-chars",
    "ExpiryMinutes": 60,
    "Issuer": "https://localhost:5001",
    "Audience": "https://localhost:5001"
  }
}
```

```json
// appsettings.Production.json — chỉ chứa non-secret overrides
// Secrets lấy từ Azure Key Vault hoặc environment variables
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Error"
    }
  },
  "Jwt": {
    "ExpiryMinutes": 15,
    "Issuer": "https://api.myapp.com",
    "Audience": "https://myapp.com"
  }
}
```

---

## 3. Options Pattern — Strongly-Typed Configuration

Options pattern cho phép đọc cấu hình theo kiểu dữ liệu cụ thể thay vì string.

```csharp
// Options classes — lớp cấu hình strongly-typed
public class JwtOptions
{
    public const string SectionName = "Jwt";

    public string SecretKey { get; init; } = string.Empty;
    public int ExpiryMinutes { get; init; } = 60;
    public string Issuer { get; init; } = string.Empty;
    public string Audience { get; init; } = string.Empty;
}

public class EmailOptions
{
    public const string SectionName = "Email";

    public string SmtpHost { get; init; } = string.Empty;
    public int SmtpPort { get; init; } = 587;
    public string FromAddress { get; init; } = string.Empty;
    public string FromName { get; init; } = string.Empty;
    public string Username { get; init; } = string.Empty;
    public string Password { get; init; } = string.Empty;
}

public class CacheOptions
{
    public const string SectionName = "Cache";

    public int DefaultExpiryMinutes { get; init; } = 60;
    public int SlidingExpiryMinutes { get; init; } = 10;
}
```

```csharp
// Program.cs — đăng ký Options
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // validate ngay khi app khởi động, fail fast nếu thiếu config

builder.Services
    .AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// Sử dụng trong service
public class JwtService
{
    private readonly JwtOptions _options;

    public JwtService(IOptions<JwtOptions> options)
    {
        _options = options.Value;
    }

    public string GenerateToken(string userId)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.SecretKey));
        // ...
    }
}
```

### Validate Options khi Khởi Động

```csharp
// Thêm DataAnnotations validation
public class JwtOptions
{
    [Required]
    [MinLength(32, ErrorMessage = "JWT SecretKey must be at least 32 characters")]
    public string SecretKey { get; init; } = string.Empty;

    [Range(1, 1440)]
    public int ExpiryMinutes { get; init; } = 60;

    [Required, Url]
    public string Issuer { get; init; } = string.Empty;
}

// Custom validation
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .Validate(options =>
    {
        if (options.ExpiryMinutes > 60 && !IsTestEnvironment())
            return false; // token không nên tồn tại > 1 giờ trên production
        return true;
    }, "JWT expiry too long for production")
    .ValidateOnStart();
```

---

## 4. User Secrets — Bí Mật Người Dùng (Development Only)

User Secrets lưu secrets cục bộ trên máy developer, KHÔNG vào source code.

```bash
# Khởi tạo User Secrets (thêm UserSecretsId vào .csproj)
dotnet user-secrets init --project src/MyApp.Api/

# Thêm secrets
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;..." \
    --project src/MyApp.Api/

dotnet user-secrets set "Jwt:SecretKey" "my-dev-secret-key-32-chars-minimum" \
    --project src/MyApp.Api/

dotnet user-secrets set "Email:Password" "dev-smtp-password" \
    --project src/MyApp.Api/

# Xem tất cả secrets
dotnet user-secrets list --project src/MyApp.Api/

# Xóa secret
dotnet user-secrets remove "Email:Password" --project src/MyApp.Api/
```

```xml
<!-- MyApp.Api.csproj — sau khi init -->
<PropertyGroup>
  <UserSecretsId>a1b2c3d4-e5f6-7890-abcd-ef1234567890</UserSecretsId>
</PropertyGroup>
```

> Secrets được lưu tại: `%APPDATA%\Microsoft\UserSecrets\<user-secrets-id>\secrets.json` (Windows)

---

## 5. Environment Variables — Biến Môi Trường

### Quy Tắc Mapping — Ánh Xạ

ASP.NET Core tự động map `__` (double underscore) trong env vars thành `:` (dấu hai chấm) trong configuration:

```bash
# ConnectionStrings:Default → ConnectionStrings__Default
export ConnectionStrings__Default="Server=prod-server;..."

# Jwt:SecretKey → Jwt__SecretKey
export Jwt__SecretKey="production-secret-key"

# Logging:LogLevel:Default → Logging__LogLevel__Default
export Logging__LogLevel__Default="Warning"
```

### Trong Docker Compose

```yaml
services:
  api:
    environment:
      # Dùng __ để tránh nhầm lẫn
      - ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=secret
      - Jwt__SecretKey=docker-compose-secret-key
      - ASPNETCORE_ENVIRONMENT=Development
    env_file:
      - .env.development  # load từ file .env
```

```bash
# .env.development — KHÔNG commit file này vào git!
# Thêm .env* vào .gitignore
ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=devpassword
Jwt__SecretKey=dev-secret-key-at-least-32-characters-long
Redis__ConnectionString=redis:6379
```

---

## 6. Azure Key Vault — Quản Lý Secrets Production

### Cấu Trúc Secrets trong Key Vault

Azure Key Vault dùng `--` (double dash) thay vì `:` hay `__`:

```
Key Vault Secret Name          ← ánh xạ thành →   .NET Config Key
ConnectionStrings--Default                          ConnectionStrings:Default
Jwt--SecretKey                                      Jwt:SecretKey
Email--Password                                     Email:Password
```

### Tích Hợp Key Vault trong .NET

```csharp
// Program.cs
using Azure.Identity;
using Azure.Extensions.AspNetCore.Configuration.Secrets;

var builder = WebApplication.CreateBuilder(args);

// Chỉ dùng Key Vault khi Production
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = new Uri(builder.Configuration["KeyVault:Uri"]
        ?? throw new InvalidOperationException("KeyVault:Uri not configured"));

    builder.Configuration.AddAzureKeyVault(
        keyVaultUri,
        new DefaultAzureCredential(),
        new AzureKeyVaultConfigurationOptions
        {
            // Tự reload secrets sau 5 phút
            ReloadInterval = TimeSpan.FromMinutes(5)
        });
}
```

### Tạo Secrets trong Azure Key Vault

```bash
# Tạo secrets
az keyvault secret set \
    --vault-name myapp-keyvault \
    --name "ConnectionStrings--Default" \
    --value "Server=prod-server.database.windows.net;Database=myappdb;..."

az keyvault secret set \
    --vault-name myapp-keyvault \
    --name "Jwt--SecretKey" \
    --value "production-jwt-secret-key-at-least-32-characters"

az keyvault secret set \
    --vault-name myapp-keyvault \
    --name "Email--Password" \
    --value "smtp-production-password"

# Xem danh sách secrets (chỉ tên, không thấy value)
az keyvault secret list \
    --vault-name myapp-keyvault \
    --query "[].{Name:name, Updated:attributes.updated}" \
    --output table
```

### Cấp Quyền Truy Cập Key Vault

```bash
# Cấp quyền cho Managed Identity của App Service
az keyvault set-policy \
    --name myapp-keyvault \
    --object-id $(az webapp identity show \
        --resource-group myapp-rg \
        --name myapp-api \
        --query principalId -o tsv) \
    --secret-permissions get list

# Hoặc dùng RBAC (được khuyến nghị hơn)
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $(az webapp identity show --resource-group myapp-rg --name myapp-api --query principalId -o tsv) \
    --scope /subscriptions/.../resourceGroups/myapp-rg/providers/Microsoft.KeyVault/vaults/myapp-keyvault
```

---

## 7. Kubernetes Secrets và ConfigMaps

### Sealed Secrets — Secrets Được Mã Hóa để Commit vào Git

```bash
# Cài kubeseal CLI
brew install kubeseal

# Tạo SealedSecret từ Secret thông thường
kubectl create secret generic myapp-secrets \
    --from-literal=db-connection="Server=prod..." \
    --dry-run=client \
    -o yaml | \
    kubeseal \
    --controller-namespace kube-system \
    --controller-name sealed-secrets \
    --format yaml > k8s/sealed-secrets.yaml

# File sealed-secrets.yaml AN TOÀN để commit vào git
# Chỉ giải mã được trong cluster có private key
```

### External Secrets Operator — Đồng Bộ Từ Key Vault Tự Động

```yaml
# external-secret.yaml — tự động đồng bộ từ Azure Key Vault vào K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 5m   # đồng bộ mỗi 5 phút
  secretStoreRef:
    name: azure-keyvault-store
    kind: ClusterSecretStore
  target:
    name: myapp-secrets           # tên K8s Secret được tạo ra
    creationPolicy: Owner
  data:
    - secretKey: db-connection    # key trong K8s Secret
      remoteRef:
        key: ConnectionStrings--Default  # key trong Key Vault
    - secretKey: jwt-secret
      remoteRef:
        key: Jwt--SecretKey
```

---

## 8. Feature Flags — Cờ Tính Năng

Feature flags cho phép bật/tắt tính năng mà không cần deploy lại.

```csharp
// FeatureFlags.cs
public static class FeatureFlags
{
    public const string NewCheckoutFlow = "NewCheckoutFlow";
    public const string BetaSearch = "BetaSearch";
    public const string DarkMode = "DarkMode";
}
```

```csharp
// Program.cs — dùng Microsoft.FeatureManagement
builder.Services.AddFeatureManagement(
    builder.Configuration.GetSection("FeatureManagement"));

// Controller
[ApiController]
[Route("api/[controller]")]
public class CheckoutController : ControllerBase
{
    private readonly IFeatureManager _featureManager;

    public CheckoutController(IFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    [HttpPost]
    public async Task<IActionResult> Checkout([FromBody] CheckoutRequest request)
    {
        if (await _featureManager.IsEnabledAsync(FeatureFlags.NewCheckoutFlow))
        {
            return Ok(await _newCheckoutService.ProcessAsync(request));
        }

        return Ok(await _legacyCheckoutService.ProcessAsync(request));
    }
}
```

```json
// appsettings.json — cấu hình feature flags
{
  "FeatureManagement": {
    "NewCheckoutFlow": false,
    "BetaSearch": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 10
          }
        }
      ]
    }
  }
}
```

---

## 9. Tổng Hợp: Chiến Lược Theo Môi Trường

```
┌─────────────────────────────────────────────────────────────────┐
│                     CHIẾN LƯỢC CONFIGURATION                    │
│                                                                 │
│  Development                                                    │
│  ├── appsettings.json (defaults)                               │
│  ├── appsettings.Development.json (dev overrides)              │
│  └── User Secrets (sensitive dev values, không commit)         │
│                                                                 │
│  CI/CD Pipeline (Test)                                          │
│  ├── appsettings.json (defaults)                               │
│  └── Environment Variables (set trong CI runner)               │
│                                                                 │
│  Staging                                                        │
│  ├── appsettings.json (defaults)                               │
│  ├── appsettings.Staging.json (staging overrides)             │
│  ├── Environment Variables (non-sensitive)                     │
│  └── Azure Key Vault / K8s Secrets (sensitive)                │
│                                                                 │
│  Production                                                     │
│  ├── appsettings.json (defaults)                               │
│  ├── appsettings.Production.json (prod overrides, non-secret) │
│  ├── Environment Variables (non-sensitive runtime config)     │
│  └── Azure Key Vault (ALL secrets, auto-rotated)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Secret Rotation — Xoay Vòng Bí Mật

```bash
# Tạo phiên bản mới cho secret trong Key Vault
az keyvault secret set \
    --vault-name myapp-keyvault \
    --name "Jwt--SecretKey" \
    --value "new-rotated-secret-key-$(date +%Y%m%d)"

# Đặt expiry cho secret cũ
az keyvault secret set-attributes \
    --vault-name myapp-keyvault \
    --name "Jwt--SecretKey" \
    --expires "2026-12-31T00:00:00Z"
```

```csharp
// Trong .NET: IOptionsMonitor để nhận updates khi secret được reload
public class JwtService
{
    private readonly IOptionsMonitor<JwtOptions> _optionsMonitor;

    public JwtService(IOptionsMonitor<JwtOptions> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
    }

    public string GenerateToken(string userId)
    {
        // Luôn lấy giá trị mới nhất — IOptionsMonitor.CurrentValue
        var currentOptions = _optionsMonitor.CurrentValue;
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(currentOptions.SecretKey));
        // ...
    }
}
```

---

## Checklist Configuration Management

- [ ] Không có secrets trong source code hoặc appsettings.json commit vào git
- [ ] `.env` files được thêm vào `.gitignore`
- [ ] User Secrets dùng cho development
- [ ] Azure Key Vault dùng cho mọi production secrets
- [ ] Options pattern với ValidateOnStart để fail fast khi thiếu config
- [ ] IOptionsMonitor (thay vì IOptions) cho config có thể thay đổi khi runtime
- [ ] Feature flags cho phép bật/tắt tính năng không cần deploy
- [ ] Secret rotation được lên lịch định kỳ
- [ ] Least privilege — quyền tối thiểu cho Managed Identity truy cập Key Vault
- [ ] Audit logs được enable trên Key Vault

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa IOptions, IOptionsMonitor và IOptionsSnapshot?**
> `IOptions<T>`: Singleton, giá trị đọc một lần khi khởi động, không reload. `IOptionsMonitor<T>`: Singleton, reload khi config thay đổi (real-time), dùng cho long-lived services. `IOptionsSnapshot<T>`: Scoped, giá trị mới mỗi request, dùng cho services cần config nhất quán trong một request.

**Q: Tại sao không nên đưa secrets vào environment variables thuần túy?**
> Environment variables dễ bị lộ qua: process listing (`ps aux`), application error dumps, container inspect, log injection. Production secrets nên dùng dedicated secret management như Azure Key Vault, HashiCorp Vault, hoặc K8s Secrets (kết hợp với RBAC và encryption at rest).

**Q: ValidateOnStart làm gì?**
> `ValidateOnStart()` trigger validation của Options ngay khi ứng dụng khởi động thay vì khi Options được dùng lần đầu. Điều này giúp phát hiện misconfiguration — cấu hình sai sớm (fail fast), tránh ứng dụng khởi động thành công nhưng crash khi gặp config thiếu ở runtime.

**Cập Nhật Lần Cuối:** 2026-06-02
