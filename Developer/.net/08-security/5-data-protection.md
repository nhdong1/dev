# ASP.NET Core Data Protection API — Bảo Vệ Dữ Liệu

> ASP.NET Core Data Protection API — DPAPI — là hệ thống mã hóa tích hợp sẵn trong ASP.NET Core, được thiết kế để bảo vệ dữ liệu nhạy cảm ngắn hạn như cookie authentication, anti-forgery tokens, và các dữ liệu cần mã hóa tạm thời.

---

## 1. Khi Nào Dùng Data Protection API

```
Dùng Data Protection API cho:              Không phù hợp cho:
────────────────────────────               ──────────────────────────
✅ Cookie authentication values            ❌ Mật khẩu (dùng hash: BCrypt)
✅ Anti-forgery tokens (CSRF)              ❌ Dữ liệu cần lưu vĩnh viễn
✅ OAuth state parameters                  ❌ Dữ liệu cần decrypt ở hệ thống khác
✅ Reset password tokens                   ❌ Sensitive data at rest (dùng DB encryption)
✅ Email confirmation tokens
✅ Temporary sensitive URLs
✅ Encrypt query parameters
```

---

## 2. Cách Hoạt Động

```
Plain Text              Data Protector             Cipher Text
───────────             ──────────────             ───────────
"user:12345"    ──────► Encrypt + HMAC  ──────►   "CfDJ8M..."
                        (AES-256-CBC +              (base64url)
                         HMAC-SHA256)

Cipher Text             Data Protector             Plain Text
───────────             ──────────────             ───────────
"CfDJ8M..."    ──────►  Verify HMAC    ──────►    "user:12345"
                        Decrypt
                        Check expiry
```

**Data Protector tự động:**
- Tạo và quản lý key (default: 90 ngày)
- Rotate key khi đến hạn
- Giữ old keys để decrypt dữ liệu cũ
- Thêm purpose string để ngăn dữ liệu từ một context bị dùng ở context khác

---

## 3. Cài Đặt Cơ Bản

```csharp
// Program.cs — Data Protection đã có sẵn, nhưng cần cấu hình cho production
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")                    // Để chia sẻ keys giữa các apps
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));  // Key hết hạn sau 90 ngày

// Mặc định, keys lưu tại:
// Windows: %LOCALAPPDATA%\ASP.NET\DataProtection-Keys
// Linux: ~/.aspnet/DataProtection-Keys
// Docker: Mất khi container restart — cần persist!
```

---

## 4. Sử Dụng IDataProtector

```csharp
public class TokenService
{
    private readonly IDataProtector _protector;

    public TokenService(IDataProtectionProvider provider)
    {
        // "purpose" string phân biệt context sử dụng
        // Token tạo với purpose "email-confirm" không thể dùng cho "password-reset"
        _protector = provider.CreateProtector("EmailConfirmation");
    }

    public string CreateEmailConfirmationToken(string userId)
    {
        var payload = $"{userId}:{DateTime.UtcNow:O}";
        return _protector.Protect(payload);
    }

    public (string userId, DateTime createdAt)? ValidateToken(string token)
    {
        try
        {
            var payload = _protector.Unprotect(token);
            var parts = payload.Split(':');
            return (parts[0], DateTime.Parse(parts[1]));
        }
        catch (CryptographicException)
        {
            // Token bị tamper hoặc không hợp lệ
            return null;
        }
    }
}
```

---

## 5. ITimeLimitedDataProtector — Bảo Vệ Với Thời Hạn

```csharp
public class PasswordResetService
{
    private readonly ITimeLimitedDataProtector _protector;

    public PasswordResetService(IDataProtectionProvider provider)
    {
        _protector = provider
            .CreateProtector("PasswordReset")
            .ToTimeLimitedDataProtector();  // Chuyển thành time-limited
    }

    public string CreateResetToken(string email)
    {
        // Token tự động hết hạn sau 1 giờ
        return _protector.Protect(email, lifetime: TimeSpan.FromHours(1));
    }

    public string? ValidateResetToken(string token)
    {
        try
        {
            // Tự động kiểm tra thời hạn
            return _protector.Unprotect(token, out var expiry);
        }
        catch (SecurityTokenExpiredException)
        {
            // Token hết hạn
            return null;
        }
        catch (CryptographicException)
        {
            // Token bị tamper
            return null;
        }
    }
}
```

---

## 6. Purpose Hierarchy — Phân Cấp Mục Đích

```csharp
// Purpose là hierarchical — phân cấp
var rootProvider = provider.CreateProtector("MyApp");
var emailProtector = rootProvider.CreateProtector("EmailConfirmation");
var resetProtector = rootProvider.CreateProtector("PasswordReset");

// Tương đương với:
var emailProtector2 = provider.CreateProtector("MyApp", "EmailConfirmation");

// Token từ emailProtector KHÔNG thể decrypt bởi resetProtector
// → Ngăn chặn token reuse attacks
var token = emailProtector.Protect("user@example.com");
resetProtector.Unprotect(token);  // Ném CryptographicException
```

---

## 7. Key Storage — Lưu Trữ Keys

### Development (Mặc Định)

```csharp
// Keys lưu trong file system
// Không cần cấu hình thêm cho development
builder.Services.AddDataProtection();
```

### Production — Azure Key Vault + Azure Blob Storage

```bash
dotnet add package Azure.Extensions.AspNetCore.DataProtection.Blobs
dotnet add package Azure.Extensions.AspNetCore.DataProtection.Keys
```

```csharp
// Lưu keys vào Azure Blob Storage, mã hóa bằng Azure Key Vault
builder.Services.AddDataProtection()
    .SetApplicationName("MyProductionApp")
    .PersistKeysToAzureBlobStorage(
        new Uri("https://mystorageaccount.blob.core.windows.net/dataprotection/keys.xml"),
        new DefaultAzureCredential())  // Dùng Managed Identity
    .ProtectKeysWithAzureKeyVault(
        new Uri("https://mykeyvault.vault.azure.net/keys/DataProtectionKey"),
        new DefaultAzureCredential());
```

### Production — Redis

```bash
dotnet add package Microsoft.AspNetCore.DataProtection.StackExchangeRedis
```

```csharp
// Lưu keys vào Redis — phù hợp cho multi-instance deployment
var redis = ConnectionMultiplexer.Connect("localhost:6379");

builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToStackExchangeRedis(redis, "DataProtection-Keys");
```

### Production — Database (EF Core)

```bash
dotnet add package Microsoft.AspNetCore.DataProtection.EntityFrameworkCore
```

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToDbContext<ApplicationDbContext>();

// DbContext cần implement IDataProtectionKeyContext
public class ApplicationDbContext : DbContext, IDataProtectionKeyContext
{
    public DbSet<DataProtectionKey> DataProtectionKeys { get; set; } = null!;
}
```

---

## 8. Key Rotation — Xoay Vòng Key

```
Vòng Đời Key:
──────────────────────────────────────────────────────
    [Tạo]    [Active - 90 ngày]    [Expired]    [Retired]
      │              │                  │             │
      └──────────────┘                  │             │
              ↓                         │             │
      Mã hóa dữ liệu mới               │             │
                                        │             │
              Vẫn decrypt được          │             │
              dữ liệu cũ ──────────────►│             │
                                                      │
              Keys quá cũ bị xóa ──────────────────► │
```

```csharp
// Tùy chỉnh key rotation
builder.Services.AddDataProtection()
    .SetDefaultKeyLifetime(TimeSpan.FromDays(14))  // Key mới sau 14 ngày

    // Thêm key revocation (xóa key cụ thể)
    // Dùng khi key bị lộ
    ;

// Xem danh sách keys hiện tại
var keyManager = app.Services.GetRequiredService<IKeyManager>();
var allKeys = keyManager.GetAllKeys();
foreach (var key in allKeys)
{
    Console.WriteLine($"Key {key.KeyId}: Created={key.CreationDate}, " +
                      $"Expires={key.ExpirationDate}, Revoked={key.IsRevoked}");
}

// Thu hồi key khi bị lộ
keyManager.RevokeKey(compromisedKeyId,
    reason: "Key compromised — phát hiện rò rỉ");
```

---

## 9. Mã Hóa Dữ Liệu Nhạy Cảm Trong Database

Data Protection API có thể dùng để mã hóa dữ liệu lưu trong DB:

```csharp
public class PersonalDataService
{
    private readonly IDataProtector _protector;

    public PersonalDataService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("PersonalData.v1");
    }

    // Mã hóa trước khi lưu DB
    public string EncryptPhoneNumber(string phoneNumber)
    {
        return _protector.Protect(phoneNumber);
    }

    // Giải mã khi đọc từ DB
    public string DecryptPhoneNumber(string encryptedPhone)
    {
        try
        {
            return _protector.Unprotect(encryptedPhone);
        }
        catch (CryptographicException)
        {
            return "[Không thể giải mã]";
        }
    }
}

// Entity với personal data được mã hóa
public class Patient
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string EncryptedPhoneNumber { get; set; } = string.Empty;  // Mã hóa trong DB
    public string EncryptedDateOfBirth { get; set; } = string.Empty;  // Mã hóa trong DB
}
```

---

## 10. Encryption vs Hashing — Khi Nào Dùng Gì

```
Encryption — Mã Hóa (có thể giải mã)    Hashing — Băm (không thể giải mã)
──────────────────────────────────        ──────────────────────────────────
Data Protection API                       BCrypt / Argon2 / PBKDF2

Dùng khi cần đọc lại giá trị gốc:        Dùng khi chỉ cần verify:
  - Số điện thoại cần hiển thị            - Mật khẩu
  - Email cần gửi                         - API keys
  - Số thẻ tín dụng (hiển thị 4 số cuối)  - Security tokens
  - Dữ liệu cá nhân cần truy cập          - File checksums

Symmetric encryption                      One-way function
Có key → có thể decrypt                  Không có cách nào reverse
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: Data Protection API khác `IDataProtectionProvider` trong Identity như thế nào?**

```
ASP.NET Core Identity dùng Data Protection API bên dưới để tạo:
  - Email confirmation tokens
  - Password reset tokens
  - Change email tokens

Bạn KHÔNG cần cài thêm gì — Identity đã tích hợp sẵn.
Chỉ cần đảm bảo Data Protection được cấu hình đúng
(keys persist trong production) để tokens không mất khi restart.
```

**Q: Tại sao phải persist Data Protection keys trong production?**

```
Mặc định, keys lưu trong memory hoặc file system cục bộ.

Vấn đề:
1. Restart container/server → mất keys → mọi encrypted data (cookies, tokens) hết hạn
2. Load balancing → Server A mã hóa → Server B không giải mã được
3. Blue-green deployment → rolling restart → users bị logout

Giải pháp:
→ Persist keys trong shared storage: Redis, Database, Azure Blob
→ Encrypt keys với Key Management Service: Azure Key Vault, AWS KMS
```

---

## ✅ Checklist Data Protection

```
✅ Keys được persist trong shared storage (production)
✅ Keys được mã hóa với Key Management Service
✅ SetApplicationName nếu có nhiều apps chia sẻ keys
✅ Time-limited tokens cho password reset (≤ 24 giờ)
✅ Purpose strings cụ thể và mô tả rõ ràng
✅ Handle CryptographicException khi unprotect
✅ Không dùng Data Protection cho password (dùng hash)
✅ Monitor key expiration và rotate định kỳ
```

---

**Xem Tiếp:** [6-https-and-tls.md](6-https-and-tls.md) — HTTPS & TLS Configuration
