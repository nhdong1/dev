# HTTPS & TLS — Bảo Mật Truyền Tải

> HTTPS — HyperText Transfer Protocol Secure — là HTTP được mã hóa bằng TLS — Transport Layer Security. Đây là lớp bảo vệ đầu tiên và bắt buộc cho mọi ứng dụng web production.

---

## 1. HTTPS vs HTTP

```
HTTP (Không Mã Hóa)              HTTPS (Có Mã Hóa)
────────────────────             ─────────────────────────────
Client ──[plain text]──► Server  Client ──[encrypted]──► Server

Kẻ tấn công có thể đọc:         Kẻ tấn công chỉ thấy:
  - Username / Password            - Địa chỉ IP + Port
  - JWT Token                      - Hostname (SNI)
  - Nội dung request/response      - Không thấy URL path, headers, body
  - Cookie session
  - Credit card numbers
```

---

## 2. TLS Handshake — Quá Trình Bắt Tay TLS

```
Client                              Server
  │                                    │
  │ ── Client Hello ──────────────────►│
  │    - TLS version supported          │
  │    - Cipher suites supported        │
  │    - Random bytes (client random)   │
  │                                    │
  │◄── Server Hello ───────────────────│
  │    - TLS version chosen             │
  │    - Cipher suite chosen            │
  │    - Random bytes (server random)   │
  │    - Server Certificate             │
  │                                    │
  │ Verify certificate:                 │
  │   - Trusted CA signed it?           │
  │   - Not expired?                    │
  │   - Domain matches?                 │
  │                                    │
  │ ── Key Exchange ──────────────────►│
  │    (ECDH hoặc RSA)                 │
  │                                    │
  │ Both derive: Session Key           │
  │ (từ client random + server random  │
  │  + key exchange)                   │
  │                                    │
  │◄──── Encrypted Application Data ──►│
  │      (dùng Session Key)            │
```

---

## 3. Bật HTTPS trong ASP.NET Core

### Development — Chứng Chỉ Self-Signed

```bash
# Tạo và tin cậy certificate development
dotnet dev-certs https --trust

# Kiểm tra
dotnet dev-certs https --check
```

### Program.cs — Cấu Hình HTTPS

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// HTTPS Redirection — chuyển hướng HTTP sang HTTPS
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});

// HSTS — HTTP Strict Transport Security — bảo buộc dùng HTTPS
builder.Services.AddHsts(options =>
{
    options.Preload = true;           // Cho phép thêm vào HSTS preload list
    options.IncludeSubDomains = true; // Áp dụng cho tất cả subdomains
    options.MaxAge = TimeSpan.FromDays(365);  // Thời gian browser nhớ
});

var app = builder.Build();

// Middleware thứ tự quan trọng
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();  // HSTS chỉ bật ở production
}

app.UseHttpsRedirection();  // Redirect HTTP → HTTPS

// ... các middleware khác
```

---

## 4. HSTS — HTTP Strict Transport Security — Bảo Buộc HTTPS

HSTS là HTTP header bảo browser **không bao giờ** gửi request HTTP thuần — luôn dùng HTTPS.

```
Lần 1: User truy cập http://myapp.com
  → Server redirect 301/308 → https://myapp.com
  → Response có header: Strict-Transport-Security: max-age=31536000; includeSubDomains

Lần 2+: Browser tự động dùng HTTPS (không cần redirect)
  → Ngay cả khi user gõ "http://"
  → Không có bất kỳ HTTP request nào rời browser
```

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Giải thích các directive:**
- `max-age=31536000` — Browser nhớ HSTS trong 365 ngày (tính bằng giây)
- `includeSubDomains` — Áp dụng cho sub.myapp.com, api.myapp.com, v.v.
- `preload` — Đăng ký vào [HSTS Preload List](https://hstspreload.org/) — browser biết từ trước khi ghé thăm lần đầu

> ⚠️ **Cảnh báo:** Bật `preload` + `includeSubDomains` rất khó rút lui. Mọi subdomain **phải** hỗ trợ HTTPS. Kiểm tra kỹ trước khi bật.

---

## 5. Certificate — Chứng Chỉ TLS

### Các Loại Certificate

```
DV — Domain Validation:
  - Xác minh sở hữu domain
  - Phát hành nhanh (phút)
  - Let's Encrypt miễn phí
  - Phù hợp: hầu hết web apps

OV — Organization Validation:
  - Xác minh tổ chức
  - Phát hành 1–3 ngày
  - Phí $50–300/năm
  - Phù hợp: business websites

EV — Extended Validation:
  - Xác minh kỹ nhất (pháp lý)
  - Phát hành 1–2 tuần
  - Phí $150–500/năm
  - Phù hợp: tài chính, ngân hàng
  - Hiển thị tên tổ chức trong address bar (browser hỗ trợ ít dần)

Wildcard (*.example.com):
  - Bao phủ mọi subdomain
  - Không bao phủ sub-subdomains
  - Giá cao hơn
```

### Let's Encrypt — Chứng Chỉ Miễn Phí

```bash
# Dùng Certbot trên Linux
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Auto-renew (certificate hết hạn sau 90 ngày)
# Certbot tự động renew nếu cài đúng cách
sudo certbot renew --dry-run
```

### Certificate trong ASP.NET Core (Kestrel)

```json
// appsettings.json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:80"
      },
      "Https": {
        "Url": "https://0.0.0.0:443",
        "Certificate": {
          "Path": "/certs/myapp.pfx",
          "Password": "cert-password"   // ← Lưu trong secrets, không hardcode
        }
      }
    }
  }
}
```

```csharp
// Hoặc cấu hình trong code
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.Listen(IPAddress.Any, 80);
    serverOptions.Listen(IPAddress.Any, 443, listenOptions =>
    {
        listenOptions.UseHttps(httpsOptions =>
        {
            httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;

            // Tắt các cipher suite yếu (optional — OS thường đã làm)
            httpsOptions.OnAuthenticate = (context, options) =>
            {
                options.CipherSuitesPolicy = new CipherSuitesPolicy(
                    new[]
                    {
                        TlsCipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
                        TlsCipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
                        TlsCipherSuite.TLS_AES_256_GCM_SHA384,
                        TlsCipherSuite.TLS_AES_128_GCM_SHA256,
                    });
            };
        });
    });
});
```

---

## 6. TLS Versions — Phiên Bản TLS

```
TLS 1.0 (1999) — ❌ KHÔNG DÙNG — bị tấn công POODLE, BEAST
TLS 1.1 (2006) — ❌ KHÔNG DÙNG — deprecated RFC 8996 (2021)
TLS 1.2 (2008) — ✅ Minimum requirement (hỗ trợ rộng)
TLS 1.3 (2018) — ✅ Khuyến nghị (nhanh hơn, an toàn hơn)
```

```csharp
// Chỉ cho phép TLS 1.2 và 1.3
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
    });
});
```

---

## 7. Security Headers — HTTP Security Headers

Ngoài HSTS, có nhiều security headers quan trọng khác:

```csharp
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;

    // HSTS (nếu không dùng AddHsts)
    headers.Append("Strict-Transport-Security",
        "max-age=31536000; includeSubDomains");

    // CSP — Content Security Policy — Chính Sách Bảo Mật Nội Dung
    // Ngăn XSS bằng cách chỉ cho phép load script/style từ nguồn tin cậy
    headers.Append("Content-Security-Policy",
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'");

    // X-Content-Type-Options — Ngăn MIME sniffing
    headers.Append("X-Content-Type-Options", "nosniff");

    // X-Frame-Options — Ngăn Clickjacking (bị thay thế bởi CSP frame-ancestors)
    headers.Append("X-Frame-Options", "DENY");

    // X-XSS-Protection — Bảo vệ XSS (legacy, modern browsers dùng CSP)
    headers.Append("X-XSS-Protection", "1; mode=block");

    // Referrer-Policy — Kiểm soát Referer header
    headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

    // Permissions-Policy — Kiểm soát browser features
    headers.Append("Permissions-Policy",
        "camera=(), microphone=(), geolocation=(), payment=()");

    await next();
});
```

Hoặc dùng package **NWebSec** hoặc **NetEscapades.AspNetCore.SecurityHeaders**:

```bash
dotnet add package NetEscapades.AspNetCore.SecurityHeaders
```

```csharp
app.UseSecurityHeaders(policies =>
    policies
        .AddDefaultSecurityHeaders()
        .AddContentSecurityPolicy(builder =>
        {
            builder.AddDefaultSrc().Self();
            builder.AddScriptSrc().Self().UnsafeInline();
        }));
```

---

## 8. HttpClient với HTTPS

```csharp
// Gọi external HTTPS API
builder.Services.AddHttpClient("ExternalApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
{
    // ❌ KHÔNG làm điều này trong production
    // ServerCertificateCustomValidationCallback = HttpClientHandler.DangerousAcceptAnyServerCertificateValidator

    // ✅ Validate certificate theo cách thông thường (mặc định)
    // Không cần cấu hình thêm
});

// Development: tắt SSL validation chỉ khi development với self-signed cert
if (app.Environment.IsDevelopment())
{
    builder.Services.AddHttpClient("LocalService")
        .ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
        {
            ServerCertificateCustomValidationCallback = (message, cert, chain, errors) =>
            {
                // Chỉ chấp nhận certificate từ localhost
                return message.RequestUri?.Host == "localhost";
            }
        });
}
```

---

## 9. Certificate Pinning — Ghim Chứng Chỉ (Nâng Cao)

```csharp
// Chỉ chấp nhận certificate cụ thể (dùng cho high-security apps)
builder.Services.AddHttpClient("SecureService")
    .ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
    {
        ServerCertificateCustomValidationCallback = (message, cert, chain, errors) =>
        {
            if (errors != SslPolicyErrors.None) return false;

            // Kiểm tra thumbprint của certificate
            var expectedThumbprint = "AA:BB:CC:...";  // SHA-256 của certificate
            var actualThumbprint = cert?.GetCertHashString(HashAlgorithmName.SHA256);

            return actualThumbprint == expectedThumbprint.Replace(":", "");
        }
    });
```

> ⚠️ Certificate Pinning rất khó maintain vì khi certificate expire/renew, phải update code.

---

## 10. HTTPS trong Docker

```dockerfile
# Dockerfile — Production
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .

# Certificate mount từ bên ngoài
# Không bao giờ COPY certificate vào image
EXPOSE 80 443
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```yaml
# docker-compose.yml
services:
  myapp:
    image: myapp
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./certs:/certs:ro  # Mount certificates
    environment:
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/certs/myapp.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=${CERT_PASSWORD}
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: HSTS là gì và tại sao quan trọng?**

```
HSTS — HTTP Strict Transport Security:
  - HTTP response header: Strict-Transport-Security: max-age=31536000
  - Báo browser: "Trong X giây tới, LUÔN dùng HTTPS với domain này"
  - Browser lưu vào bộ nhớ, không gửi bất kỳ HTTP request nào

Quan trọng vì:
  - Ngăn SSL stripping attacks (Moxie Marlinspike's sslstrip)
  - Tấn công: Kẻ tấn công chặn HTTP request trước khi redirect HTTPS
  - HSTS: Không có HTTP request nào rời browser
  - HSTS Preload: Bảo vệ ngay cả lần ghé thăm đầu tiên
```

**Q: TLS 1.2 vs TLS 1.3 — sự khác biệt?**

```
TLS 1.2:
  - Handshake: 2 round-trips (2 RTT)
  - Hỗ trợ nhiều cipher suites (kể cả yếu)
  - Forward Secrecy tùy chọn

TLS 1.3:
  - Handshake: 1 round-trip (1 RTT) — nhanh hơn
  - 0-RTT Resumption cho connections được khôi phục
  - Chỉ có cipher suites mạnh (loại bỏ RC4, 3DES, v.v.)
  - Forward Secrecy bắt buộc (ECDHE)
  - Mã hóa nhiều hơn (kể cả server certificate trong handshake)
```

---

## ✅ Checklist HTTPS & TLS

```
✅ HTTPS bắt buộc (redirect HTTP → HTTPS)
✅ HSTS bật với max-age ≥ 1 năm
✅ TLS 1.2 minimum, TLS 1.3 khuyến nghị
✅ TLS 1.0, 1.1 tắt
✅ Certificate hợp lệ từ trusted CA
✅ Auto-renewal certificate (Let's Encrypt / cert-manager)
✅ Security headers: X-Content-Type-Options, X-Frame-Options, CSP
✅ Không tắt certificate validation trong production
✅ Không lưu certificate password trong source code
✅ Dùng công cụ kiểm tra: SSL Labs (ssllabs.com)
```

---

**Xem Tiếp:** [7-cors.md](7-cors.md) — CORS Configuration
