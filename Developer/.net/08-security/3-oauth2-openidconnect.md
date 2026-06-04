# OAuth 2.0 & OpenID Connect — Ủy Quyền và Xác Thực Liên Kết

> OAuth 2.0 là framework **ủy quyền** — authorization — cho phép ứng dụng truy cập tài nguyên thay mặt người dùng mà không cần mật khẩu. OpenID Connect — OIDC — là lớp **xác thực** — authentication — xây dựng trên OAuth 2.0.

---

## 1. OAuth 2.0 vs OpenID Connect

```
OAuth 2.0                              OpenID Connect (OIDC)
─────────────────────────────          ──────────────────────────────────
"Ủy quyền" — Authorization            "Xác thực" — Authentication
"App được làm GÌ?"                    "Người dùng là AI?"

Trả về: Access Token                   Trả về: Access Token + ID Token
Dùng để: Gọi API thay mặt user        Dùng để: Đăng nhập, lấy user info

Ví dụ:                                 Ví dụ:
  - App đọc Google Drive của bạn         - "Đăng nhập bằng Google"
  - App post lên Twitter của bạn         - SSO — Single Sign-On
  - App xem calendar Outlook của bạn     - Liên kết tài khoản
```

---

## 2. Các Roles trong OAuth 2.0

```
┌──────────────────────┐
│  Resource Owner      │  — Người Dùng: sở hữu tài nguyên (ảnh, email, ...)
│  (End User)          │
└──────────────────────┘
           │ cấp quyền
           ▼
┌──────────────────────┐
│  Client              │  — Ứng Dụng: muốn truy cập tài nguyên
│  (Application)       │     (web app, mobile app, SPA)
└──────────────────────┘
           │ lấy token
           ▼
┌──────────────────────┐
│  Authorization       │  — Máy Chủ Ủy Quyền: phát token
│  Server              │     (Google, Azure AD, Auth0, Keycloak)
└──────────────────────┘
           │ token
           ▼
┌──────────────────────┐
│  Resource Server     │  — Máy Chủ Tài Nguyên: API chứa dữ liệu
│  (API)               │     (Google API, Microsoft Graph, API của bạn)
└──────────────────────┘
```

---

## 3. Authorization Code Flow — Luồng Mã Ủy Quyền (Khuyến Nghị)

Đây là flow an toàn nhất, phù hợp cho web apps và native apps.

```
User     Client App           Auth Server         Resource Server
  │          │                     │                     │
  │ Click    │                     │                     │
  │ "Login   │                     │                     │
  │ with     │                     │                     │
  │ Google"  │                     │                     │
  │─────────►│                     │                     │
  │          │ Redirect to         │                     │
  │          │ /authorize?         │                     │
  │          │ client_id=xxx       │                     │
  │          │ redirect_uri=xxx    │                     │
  │          │ scope=openid+email  │                     │
  │          │ state=random        │                     │
  │          │ code_challenge=xxx  │ ← PKCE              │
  │          │────────────────────►│                     │
  │          │                     │                     │
  │◄────────────────────────────── │ Login Page          │
  │ Nhập     │                     │                     │
  │ email/pw │                     │                     │
  │─────────────────────────────── │                     │
  │          │                     │                     │
  │◄────────────────────────────── │ Consent Screen      │
  │ "App muốn│                     │ (nếu chưa cho phép) │
  │ đọc email│                     │                     │
  │ của bạn" │                     │                     │
  │─────────────────────────────── │ User approves       │
  │          │                     │                     │
  │          │◄────────────────────│ Redirect với        │
  │          │ ?code=AUTH_CODE     │ Authorization Code  │
  │          │ &state=random       │                     │
  │          │                     │                     │
  │          │ POST /token         │                     │
  │          │ code=AUTH_CODE      │                     │
  │          │ code_verifier=xxx   │ ← PKCE verify       │
  │          │────────────────────►│                     │
  │          │                     │                     │
  │          │◄────────────────────│ Access Token        │
  │          │ + ID Token          │ + Refresh Token     │
  │          │ + Refresh Token     │                     │
  │          │                     │                     │
  │          │ GET /userinfo       │                     │
  │          │ Authorization:      │                     │
  │          │ Bearer <token>      │────────────────────►│
  │          │                     │                     │
  │          │◄─────────────────────────────────────────│
  │          │ { email, name, ... }│                     │
```

### PKCE — Proof Key for Code Exchange — Bằng Chứng Trao Đổi Mã

PKCE — phát âm "pixy" — ngăn Authorization Code Interception Attack:

```csharp
// Bước 1: Tạo code_verifier (ngẫu nhiên)
var codeVerifier = GenerateRandomString(128);

// Bước 2: Tạo code_challenge = SHA256(code_verifier)
using var sha256 = SHA256.Create();
var challengeBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(codeVerifier));
var codeChallenge = Base64UrlEncode(challengeBytes);

// Bước 3: Gửi code_challenge khi authorize
// /authorize?code_challenge=xxx&code_challenge_method=S256

// Bước 4: Gửi code_verifier khi đổi token
// POST /token body: code_verifier=xxx
// Server tự verify: SHA256(code_verifier) == code_challenge
```

---

## 4. Các Grant Types — Loại Cấp Phép

| Grant Type | Khi Nào Dùng | Lưu Ý |
|-----------|-------------|-------|
| **Authorization Code + PKCE** | Web app, SPA, Mobile app | ✅ Khuyến nghị nhất |
| **Client Credentials** | Machine-to-machine (M2M), không có user | ✅ Backend service |
| **Device Code** | Smart TV, IoT không có browser | ✅ Thiết bị giới hạn |
| ~~Implicit~~ | ~~SPA~~ | ❌ Deprecated — không dùng |
| ~~Resource Owner Password~~ | ~~Legacy~~ | ❌ Deprecated — không dùng |

---

## 5. Tích Hợp Google Login trong ASP.NET Core

### Package

```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
```

### Cấu Hình

```csharp
// Program.cs
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
})
.AddCookie(options =>
{
    options.LoginPath = "/auth/login";
    options.ExpireTimeSpan = TimeSpan.FromHours(1);
})
.AddGoogle(options =>
{
    options.ClientId = builder.Configuration["Authentication:Google:ClientId"]!;
    options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"]!;

    // Yêu cầu thêm scopes — phạm vi quyền
    options.Scope.Add("email");
    options.Scope.Add("profile");

    // Lưu token để dùng sau
    options.SaveTokens = true;

    // Map claims từ Google
    options.ClaimActions.MapJsonKey(ClaimTypes.Email, "email");
    options.ClaimActions.MapJsonKey(ClaimTypes.Name, "name");
    options.ClaimActions.MapJsonKey("picture", "picture");

    options.Events.OnCreatingTicket = async context =>
    {
        // Xử lý khi user đăng nhập thành công
        var email = context.Principal?.FindFirstValue(ClaimTypes.Email);
        // Tạo hoặc cập nhật user trong database
    };
});
```

### Controller

```csharp
[Route("auth")]
public class AuthController : Controller
{
    [HttpGet("login")]
    public IActionResult Login(string? returnUrl = null)
    {
        var properties = new AuthenticationProperties
        {
            RedirectUri = Url.Action("Callback", new { returnUrl })
        };
        return Challenge(properties, GoogleDefaults.AuthenticationScheme);
    }

    [HttpGet("callback")]
    public async Task<IActionResult> Callback(string? returnUrl = null)
    {
        var result = await HttpContext.AuthenticateAsync(
            CookieAuthenticationDefaults.AuthenticationScheme);

        if (!result.Succeeded) return Unauthorized();

        var email = result.Principal?.FindFirstValue(ClaimTypes.Email);
        // Lấy hoặc tạo user trong DB

        return LocalRedirect(returnUrl ?? "/");
    }

    [HttpPost("logout")]
    public async Task<IActionResult> Logout()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        return RedirectToAction("Login");
    }
}
```

---

## 6. Client Credentials Flow — Luồng Thông Tin Xác Thực Máy Khách

Dùng cho service-to-service communication, không có user:

```
Service A              Auth Server          Service B (API)
    │                      │                     │
    │ POST /token           │                     │
    │ client_id=serviceA    │                     │
    │ client_secret=xxx     │                     │
    │ grant_type=           │                     │
    │   client_credentials  │                     │
    │ scope=api.read        │                     │
    │──────────────────────►│                     │
    │                       │                     │
    │◄──────────────────────│ Access Token        │
    │                       │ (no refresh token)  │
    │                       │                     │
    │ GET /api/data         │                     │
    │ Authorization:        │                     │
    │ Bearer <token>        │────────────────────►│
    │                       │                     │
    │◄──────────────────────────────────────────── data
```

```csharp
// Gọi API khác với Client Credentials
// Cài package: Microsoft.Extensions.Http.Resilience
// hoặc dùng HttpClient với IdentityModel

builder.Services.AddHttpClient("ServiceB")
    .AddClientCredentialsTokenHandler(tokenService =>
    {
        tokenService.TokenEndpoint = "https://auth.example.com/connect/token";
        tokenService.ClientId = "service-a";
        tokenService.ClientSecret = "secret";
        tokenService.Scope = "api.read";
    });
```

---

## 7. Scopes — Phạm Vi Quyền

Scopes định nghĩa những gì Access Token được phép làm:

```
openid      — Bắt buộc cho OIDC, trả về ID Token
profile     — Thông tin cơ bản: name, picture, locale
email       — Địa chỉ email
address     — Địa chỉ vật lý
phone       — Số điện thoại
offline_access — Cho phép nhận Refresh Token

Custom scopes (API của bạn):
  api.read    — Đọc dữ liệu
  api.write   — Ghi dữ liệu
  admin       — Quyền admin
```

```csharp
// Validate scope trong API
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireReadScope", policy =>
        policy.RequireClaim("scope", "api.read"));

    options.AddPolicy("RequireWriteScope", policy =>
        policy.RequireClaim("scope", "api.write"));
});

[Authorize(Policy = "RequireReadScope")]
[HttpGet("data")]
public IActionResult GetData() => Ok(data);
```

---

## 8. ID Token — Token Định Danh (OIDC)

ID Token là JWT chứa thông tin về người dùng đã xác thực:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",      // Google User ID (unique, không đổi)
  "aud": "your-client-id.apps.googleusercontent.com",
  "exp": 1717299600,
  "iat": 1717296000,
  "email": "user@example.com",
  "email_verified": true,
  "name": "Nguyen Van A",
  "picture": "https://lh3.googleusercontent.com/...",
  "locale": "vi",
  "at_hash": "HK6E_P6Dh8Y93mRNtsDB1Q"  // Hash của Access Token để bind
}
```

**Lưu ý quan trọng:**
- ID Token dùng để **xác thực người dùng**, KHÔNG dùng để gọi API
- Access Token dùng để **gọi API**
- Luôn validate `iss`, `aud`, `exp`, `iat` của ID Token

---

## 9. Tích Hợp với Duende IdentityServer / Keycloak

Nếu bạn cần Authorization Server riêng:

```csharp
// Protect API với Resource Server configuration
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Authority — URL của Authorization Server
        options.Authority = "https://your-identityserver.com";

        // Audience — tên API resource
        options.Audience = "my-api";

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = true,
            ValidateIssuer = true,
            ValidateLifetime = true
        };
    });

// Validate scope
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ApiScope", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "my-api");
    });
});
```

---

## 10. Token Storage — Lưu Token An Toàn

```
Loại Ứng Dụng        Access Token            Refresh Token
─────────────────    ──────────────────      ───────────────────
Web App (SSR)        HttpOnly Cookie         HttpOnly Cookie
                     (server-side)

SPA (React/Vue)      Memory (JS variable)    HttpOnly Cookie
                     ❌ localStorage          hoặc BFF pattern

Native App           Secure storage          Secure storage
(iOS/Android)        (Keychain / Keystore)   (Keychain / Keystore)

Backend Service      Environment variable    Không cần (Client Credentials)
                     / Key Vault
```

### BFF Pattern — Backend For Frontend — Backend Phục Vụ Frontend

```
Browser/SPA          BFF Server              Auth Server + APIs
    │                    │                          │
    │ Đăng nhập          │                          │
    │───────────────────►│                          │
    │                    │ Authorization Code Flow   │
    │                    │◄─────────────────────────►│
    │                    │ Nhận và lưu token         │
    │◄─────────────────── Session Cookie (HttpOnly)  │
    │                    │                           │
    │ Gọi API            │                           │
    │───────────────────►│ BFF dùng token gọi API    │
    │                    │──────────────────────────►│
    │◄────────────────────────────────────────────── Data
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: OAuth 2.0 khác OpenID Connect như thế nào?**

```
OAuth 2.0: Framework ủy quyền
  - Trả lời: "App X được phép làm gì với tài nguyên của User Y?"
  - Trả về: Access Token (opaque hoặc JWT)
  - Không có thông tin về identity của user

OpenID Connect: Lớp xác thực trên OAuth 2.0
  - Trả lời: "Người dùng đang đăng nhập là ai?"
  - Trả về: Access Token + ID Token (JWT)
  - ID Token chứa claims về người dùng (email, name, ...)
  - Có UserInfo endpoint để lấy thêm thông tin
```

**Q: Tại sao không dùng Implicit Flow nữa?**

```
Implicit Flow (cũ):
  - Access Token trả về trực tiếp trong URL fragment (#access_token=...)
  - Dễ bị lộ qua browser history, Referer header, server logs
  - Không hỗ trợ Refresh Token

Authorization Code + PKCE (mới):
  - Chỉ trả về Authorization Code trong URL (ngắn hạn, dùng 1 lần)
  - Access Token được trao đổi qua POST request (không lộ trong URL)
  - PKCE ngăn code interception attack
  - Hỗ trợ Refresh Token
```

**Q: `state` parameter dùng để làm gì?**

```
state là random value do client tạo ra:
1. Gửi trong authorization request
2. Auth server trả lại trong callback
3. Client verify state match → ngăn CSRF attack trên OAuth flow

Ví dụ:
  state = crypto.randomBytes(32).toString('hex')
  // Lưu vào session
  // Sau khi callback: verify state từ URL == state trong session
```

---

## ✅ Checklist OAuth 2.0 / OIDC

```
✅ Dùng Authorization Code + PKCE (không dùng Implicit)
✅ Validate state parameter để chống CSRF
✅ Validate ID Token: iss, aud, exp, nonce
✅ Lưu tokens an toàn (HttpOnly Cookie hoặc Secure Storage)
✅ Dùng HTTPS cho tất cả OAuth endpoints
✅ Scope tối thiểu cần thiết (principle of least privilege)
✅ Refresh Token Rotation bật
✅ Không hardcode client_secret trong frontend code
```

---

**Xem Tiếp:** [4-authorization-policies.md](4-authorization-policies.md) — Policy-based Authorization
