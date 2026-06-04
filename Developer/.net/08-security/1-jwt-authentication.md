# JWT Authentication — JSON Web Token — Xác Thực Bằng Token JSON

> JWT — JSON Web Token — là tiêu chuẩn mở (RFC 7519) để truyền thông tin an toàn giữa các bên dưới dạng JSON object được ký số. Đây là phương pháp xác thực phổ biến nhất cho REST API và microservices.

---

## 1. Cấu Trúc JWT

JWT gồm 3 phần, ngăn cách bởi dấu chấm (`.`):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ik5ndXllbiBWYW4gQSIsImlhdCI6MTcxNzI5NjAwMCwiZXhwIjoxNzE3Mjk5NjAwfQ
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

  └─────── Header ──────┘ └──────────────────── Payload ──────────────────────────────┘ └──── Signature ────┘
```

### Header — Tiêu Đề

```json
{
  "alg": "HS256",   // thuật toán ký: HS256, RS256, ES256
  "typ": "JWT"      // loại token
}
```

### Payload — Tải Trọng (Claims — Các Khẳng Định)

```json
{
  "sub": "1234567890",       // Subject — chủ thể (user ID)
  "name": "Nguyen Van A",    // claim tùy chỉnh
  "email": "a@example.com",
  "role": "Admin",
  "iat": 1717296000,         // Issued At — thời điểm phát hành (Unix timestamp)
  "exp": 1717299600,         // Expiration — thời điểm hết hạn
  "nbf": 1717296000,         // Not Before — hiệu lực từ thời điểm này
  "iss": "https://api.myapp.com",  // Issuer — bên phát hành
  "aud": "https://myapp.com"       // Audience — đối tượng nhận
}
```

**Registered Claims — Claims Đã Đăng Ký (RFC 7519):**

| Claim | Ý Nghĩa | Bắt Buộc? |
|-------|---------|-----------|
| `sub` | Subject — ID người dùng | Nên có |
| `iss` | Issuer — URL của auth server | Nên có |
| `aud` | Audience — URL của resource server | Nên có |
| `exp` | Expiration — Unix timestamp hết hạn | **Bắt buộc** |
| `iat` | Issued At — Unix timestamp tạo token | Nên có |
| `nbf` | Not Before — token chưa hợp lệ trước thời điểm này | Tùy chọn |
| `jti` | JWT ID — unique identifier, dùng để revoke | Tùy chọn |

### Signature — Chữ Ký

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secretKey
)
```

> ⚠️ **Lưu ý quan trọng:** Payload JWT chỉ được **encode** (base64), KHÔNG được **encrypt** (mã hóa). Bất kỳ ai cũng có thể đọc nội dung — chỉ có signature mới xác minh tính toàn vẹn.

---

## 2. JWT Flow — Luồng Xử Lý JWT

```
Client                    API Server                  Auth Server
  │                           │                            │
  │  POST /auth/login          │                            │
  │  { username, password }    │                            │
  │──────────────────────────►│                            │
  │                           │  Verify credentials        │
  │                           │───────────────────────────►│
  │                           │                            │
  │                           │◄─── JWT Token ─────────────│
  │◄─────── JWT Token ────────│                            │
  │                           │                            │
  │  GET /api/orders           │                            │
  │  Authorization: Bearer <token>                         │
  │──────────────────────────►│                            │
  │                           │ Validate signature         │
  │                           │ Check exp, iss, aud        │
  │                           │ Extract claims             │
  │◄─────── 200 OK ───────────│                            │
```

---

## 3. Cài Đặt JWT trong ASP.NET Core

### Package cần thiết

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

### Cấu hình trong Program.cs

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Đọc cấu hình JWT từ appsettings.json
var jwtSettings = builder.Configuration.GetSection("Jwt");
var secretKey = jwtSettings["SecretKey"]!;

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        // Xác thực chữ ký
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey)),

        // Xác thực Issuer — bên phát hành
        ValidateIssuer = true,
        ValidIssuer = jwtSettings["Issuer"],

        // Xác thực Audience — đối tượng nhận
        ValidateAudience = true,
        ValidAudience = jwtSettings["Audience"],

        // Xác thực thời hạn
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero  // Không cho phép sai lệch đồng hồ (mặc định 5 phút)
    };

    // Xử lý sự kiện (tùy chọn)
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.Headers.Append("Token-Expired", "true");
            }
            return Task.CompletedTask;
        }
    };
});

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();  // Phải trước UseAuthorization
app.UseAuthorization();

app.Run();
```

### appsettings.json

```json
{
  "Jwt": {
    "SecretKey": "your-super-secret-key-must-be-at-least-256-bits-long",
    "Issuer": "https://api.myapp.com",
    "Audience": "https://myapp.com",
    "AccessTokenExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 30
  }
}
```

> ⚠️ **Không bao giờ** lưu `SecretKey` trực tiếp trong appsettings.json ở production. Dùng Azure Key Vault, AWS Secrets Manager, hoặc environment variables.

---

## 4. Tạo JWT Token

### TokenService — Dịch Vụ Tạo Token

```csharp
public interface ITokenService
{
    string GenerateAccessToken(User user);
    string GenerateRefreshToken();
    ClaimsPrincipal? GetPrincipalFromExpiredToken(string token);
}

public class TokenService : ITokenService
{
    private readonly IConfiguration _configuration;

    public TokenService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GenerateAccessToken(User user)
    {
        var jwtSettings = _configuration.GetSection("Jwt");
        var secretKey = jwtSettings["SecretKey"]!;
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        // Claims — các thông tin gắn vào token
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            new(ClaimTypes.Name, user.UserName),
            new(ClaimTypes.Role, user.Role)  // Role claim cho [Authorize(Roles = "Admin")]
        };

        var expiration = DateTime.UtcNow.AddMinutes(
            int.Parse(jwtSettings["AccessTokenExpirationMinutes"]!));

        var token = new JwtSecurityToken(
            issuer: jwtSettings["Issuer"],
            audience: jwtSettings["Audience"],
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: expiration,
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        // Refresh Token là random bytes, KHÔNG phải JWT
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }

    public ClaimsPrincipal? GetPrincipalFromExpiredToken(string token)
    {
        var jwtSettings = _configuration.GetSection("Jwt");
        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings["SecretKey"]!)),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings["Issuer"],
            ValidateAudience = true,
            ValidAudience = jwtSettings["Audience"],
            ValidateLifetime = false  // Bỏ qua check expiration khi refresh
        };

        var tokenHandler = new JwtSecurityTokenHandler();
        var principal = tokenHandler.ValidateToken(
            token, tokenValidationParameters, out var securityToken);

        if (securityToken is not JwtSecurityToken jwtToken ||
            !jwtToken.Header.Alg.Equals(
                SecurityAlgorithms.HmacSha256,
                StringComparison.InvariantCultureIgnoreCase))
        {
            return null;
        }

        return principal;
    }
}
```

### AuthController — Controller Xác Thực

```csharp
[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly ITokenService _tokenService;
    private readonly UserManager<User> _userManager;
    private readonly IRefreshTokenRepository _refreshTokenRepo;

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);
        if (user == null || !await _userManager.CheckPasswordAsync(user, request.Password))
        {
            return Unauthorized(new { message = "Email hoặc mật khẩu không đúng" });
        }

        var accessToken = _tokenService.GenerateAccessToken(user);
        var refreshToken = _tokenService.GenerateRefreshToken();

        // Lưu refresh token vào database
        await _refreshTokenRepo.SaveAsync(new RefreshToken
        {
            Token = refreshToken,
            UserId = user.Id,
            ExpiresAt = DateTime.UtcNow.AddDays(30),
            CreatedAt = DateTime.UtcNow
        });

        return Ok(new
        {
            accessToken,
            refreshToken,
            expiresIn = 3600  // giây
        });
    }

    [HttpPost("refresh")]
    public async Task<IActionResult> Refresh([FromBody] RefreshRequest request)
    {
        var principal = _tokenService.GetPrincipalFromExpiredToken(request.AccessToken);
        if (principal == null) return BadRequest("Token không hợp lệ");

        var userId = principal.FindFirstValue(JwtRegisteredClaimNames.Sub);
        var storedToken = await _refreshTokenRepo.FindAsync(request.RefreshToken);

        if (storedToken == null || storedToken.UserId != userId ||
            storedToken.ExpiresAt < DateTime.UtcNow || storedToken.IsRevoked)
        {
            return Unauthorized("Refresh token không hợp lệ hoặc đã hết hạn");
        }

        var user = await _userManager.FindByIdAsync(userId!);
        var newAccessToken = _tokenService.GenerateAccessToken(user!);
        var newRefreshToken = _tokenService.GenerateRefreshToken();

        // Rotate refresh token — xoay vòng để tăng bảo mật
        await _refreshTokenRepo.RevokeAsync(request.RefreshToken);
        await _refreshTokenRepo.SaveAsync(new RefreshToken
        {
            Token = newRefreshToken,
            UserId = user!.Id,
            ExpiresAt = DateTime.UtcNow.AddDays(30)
        });

        return Ok(new { accessToken = newAccessToken, refreshToken = newRefreshToken });
    }

    [Authorize]
    [HttpPost("logout")]
    public async Task<IActionResult> Logout([FromBody] LogoutRequest request)
    {
        await _refreshTokenRepo.RevokeAsync(request.RefreshToken);
        return NoContent();
    }
}
```

---

## 5. Access Token vs Refresh Token

```
Access Token — Token Truy Cập         Refresh Token — Token Làm Mới
─────────────────────────────         ─────────────────────────────────
Thời hạn: 15–60 phút (ngắn)          Thời hạn: 7–30 ngày (dài)
Định dạng: JWT (tự chứa)              Định dạng: Random opaque string
Gửi kèm: Mọi API request              Gửi khi: Access token hết hạn
Lưu ở: Memory / localStorage          Lưu ở: HttpOnly Cookie (an toàn hơn)
Invalidate: Hết hạn tự nhiên          Invalidate: Xóa khỏi database
```

### Token Rotation — Xoay Vòng Token

```
Lần 1: Login
  → Access Token A (60 phút)
  → Refresh Token 1

Lần 2: Access Token A hết hạn
  → Gửi Refresh Token 1
  → Server: Thu hồi Refresh Token 1
  → Cấp Access Token B (60 phút) + Refresh Token 2

Lần 3: Nếu kẻ tấn công dùng Refresh Token 1 (đã bị thu hồi)
  → Server phát hiện → Thu hồi toàn bộ tokens của user này
  → Logout buộc toàn bộ sessions
```

---

## 6. Thuật Toán Ký — Signing Algorithms

| Thuật Toán | Loại | Ưu Điểm | Nhược Điểm |
|-----------|------|---------|------------|
| HS256 | Symmetric — Đối Xứng | Đơn giản, nhanh | Phải chia sẻ secret key |
| RS256 | Asymmetric — Bất Đối Xứng | Public key có thể chia sẻ | Chậm hơn, phức tạp hơn |
| ES256 | Asymmetric (ECDSA) | Key ngắn hơn RS256, nhanh hơn | Ít phổ biến hơn |

**Khi nào dùng gì:**
- **HS256:** Monolith, khi chỉ có một service tạo và verify token
- **RS256:** Microservices, khi nhiều service cần verify nhưng chỉ một service tạo token — resource servers dùng public key để verify

```csharp
// RS256 — dùng RSA key pair
var rsa = RSA.Create(2048);
var rsaKey = new RsaSecurityKey(rsa);
var credentials = new SigningCredentials(rsaKey, SecurityAlgorithms.RsaSha256);
```

---

## 7. JWT Security Best Practices — Thực Hành Tốt Nhất

### ✅ Nên Làm

```
1. Dùng HTTPS — không bao giờ truyền JWT qua HTTP
2. Secret key ≥ 256-bit, ngẫu nhiên, không đoán được
3. Thiết lập exp (expiration) phù hợp — không quá dài
4. Validate iss, aud, exp trên mọi request
5. Dùng jti + blacklist để revoke ngay lập tức khi cần
6. Lưu Refresh Token trong HttpOnly Cookie, không localStorage
7. Implement Refresh Token Rotation
8. Log mọi failed authentication attempts
```

### ❌ Không Nên Làm

```
1. Không lưu sensitive data trong payload (mã hóa bằng JWE nếu cần)
2. Không dùng alg: none (tắt xác thực chữ ký)
3. Không hardcode secret key trong source code
4. Không đặt expiration quá dài (> 24 giờ cho access token)
5. Không bỏ qua validate claims (iss, aud)
6. Không tin tưởng payload mà không verify signature
```

---

## 8. JWT Revocation — Thu Hồi Token

JWT về bản chất là **stateless** — không thể revoke như session. Các giải pháp:

### Cách 1: Short Expiration + Refresh Token (Phổ biến nhất)
```
Access Token: 15 phút
→ Nếu cần revoke: chờ 15 phút hoặc blacklist jti
```

### Cách 2: JTI Blacklist với Redis

```csharp
public class JwtRevocationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;  // Redis

    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"]
            .ToString().Replace("Bearer ", "");

        if (!string.IsNullOrEmpty(token))
        {
            var handler = new JwtSecurityTokenHandler();
            var jwtToken = handler.ReadJwtToken(token);
            var jti = jwtToken.Claims
                .FirstOrDefault(c => c.Type == JwtRegisteredClaimNames.Jti)?.Value;

            // Kiểm tra token có trong blacklist không
            if (jti != null && await _cache.GetStringAsync($"revoked:{jti}") != null)
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsJsonAsync(new { error = "Token đã bị thu hồi" });
                return;
            }
        }

        await _next(context);
    }
}

// Khi logout: thêm jti vào blacklist
await _cache.SetStringAsync(
    $"revoked:{jti}",
    "1",
    new DistributedCacheEntryOptions
    {
        AbsoluteExpiration = tokenExpiration  // Hết hạn khi token hết hạn
    });
```

---

## 9. Đọc Claims Trong Controller

```csharp
[Authorize]
[ApiController]
[Route("api/profile")]
public class ProfileController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProfile()
    {
        // Đọc claims từ JWT đã được xác thực
        var userId = User.FindFirstValue(JwtRegisteredClaimNames.Sub);
        var email = User.FindFirstValue(ClaimTypes.Email);
        var role = User.FindFirstValue(ClaimTypes.Role);
        var userName = User.Identity?.Name;

        // Kiểm tra role
        var isAdmin = User.IsInRole("Admin");

        return Ok(new { userId, email, role, isAdmin });
    }
}
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: JWT khác Session-based authentication như thế nào?**

```
Session-based:
  - Server lưu session trong memory/database
  - Client nhận session ID (cookie)
  - Stateful — server phải nhớ state
  - Dễ revoke ngay lập tức
  - Khó scale (cần shared session store)

JWT-based:
  - Server KHÔNG lưu trạng thái (stateless)
  - Client nhận JWT chứa toàn bộ claims
  - Stateless — server chỉ cần verify chữ ký
  - Khó revoke ngay (phải dùng blacklist)
  - Dễ scale (mọi server đều verify được)
```

**Q: Làm sao lưu JWT an toàn ở client?**

```
localStorage:
  ❌ Dễ bị XSS đọc (JavaScript có thể truy cập)

sessionStorage:
  ⚠️ Tốt hơn localStorage nhưng vẫn bị XSS

HttpOnly Cookie:
  ✅ JavaScript không thể đọc → chống XSS
  ⚠️ Cần thêm CSRF protection (SameSite=Strict hoặc CSRF token)

Memory (biến JavaScript):
  ✅ Chống XSS (không persist)
  ❌ Mất khi refresh trang → cần silent refresh flow
```

**Q: Tại sao không nên dùng `alg: none`?**

```
Kẻ tấn công có thể tạo token tùy ý mà không cần secret:
1. Decode payload JWT hợp lệ
2. Sửa claims (ví dụ role: "Admin")
3. Set alg: none trong header
4. Tạo token không có signature
5. Nếu server chấp nhận alg: none → bị tấn công

→ Luôn enforce thuật toán cụ thể khi validate
```

---

## ✅ Checklist JWT

```
✅ Validate iss, aud, exp, nbf
✅ Secret key ≥ 256-bit, lưu trong Key Vault / Secrets Manager
✅ Access token expiration ≤ 60 phút
✅ Refresh Token lưu trong HttpOnly Cookie
✅ Implement Token Rotation khi refresh
✅ Log failed authentication
✅ Không lưu sensitive data trong payload
✅ Dùng HTTPS
✅ Không chấp nhận alg: none
```

---

**Xem Tiếp:** [2-aspnet-identity.md](2-aspnet-identity.md) — ASP.NET Core Identity
