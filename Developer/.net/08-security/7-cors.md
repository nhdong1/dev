# CORS — Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Chéo Nguồn Gốc

> CORS là cơ chế bảo mật của browser ngăn chặn các request từ một origin (nguồn gốc) truy cập tài nguyên của origin khác — trừ khi server cho phép rõ ràng.

---

## 1. Same-Origin Policy — Chính Sách Cùng Nguồn Gốc

Browser áp dụng Same-Origin Policy — SOP: JavaScript chỉ được gọi API cùng origin.

```
Origin = scheme + hostname + port

https://myapp.com:443/api/users
  ↑          ↑         ↑
scheme    hostname    port

Hai request cùng origin? (Phải khớp CẢ BA)
┌─────────────────────────────────────────────────────────────────┐
│ FROM                    │ TO                     │ Kết Quả       │
├─────────────────────────┼────────────────────────┼───────────────┤
│ https://myapp.com       │ https://myapp.com/api  │ ✅ Cùng origin │
│ https://myapp.com       │ http://myapp.com       │ ❌ Khác scheme │
│ https://myapp.com       │ https://api.myapp.com  │ ❌ Khác host   │
│ https://myapp.com       │ https://myapp.com:8080 │ ❌ Khác port   │
│ https://myapp.com:443   │ https://myapp.com      │ ✅ Port 443 mặc định │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. CORS Hoạt Động Như Thế Nào

```
React App (https://app.com)          API Server (https://api.com)
         │                                      │
         │ GET /data                            │
         │ Origin: https://app.com              │
         │─────────────────────────────────────►│
         │                                      │
         │                                      │ Kiểm tra Origin
         │                                      │ trong whitelist?
         │                                      │
         │◄─────────────────────────────────────│
         │ Access-Control-Allow-Origin: https://app.com
         │ (Hoặc không có header → browser block)
         │                                      │
         │ Browser nhận response                │
         │ Nếu không có ACAO header → bị block  │
         │ Nếu có ACAO header phù hợp → ✅      │
```

### Preflight Request — Yêu Cầu Kiểm Tra Trước

Với "complex requests" (có custom headers, body JSON, v.v.), browser gửi OPTIONS request trước:

```
Browser                              API Server
   │                                    │
   │ OPTIONS /api/users                 │
   │ Origin: https://app.com            │
   │ Access-Control-Request-Method: POST│
   │ Access-Control-Request-Headers: Content-Type, Authorization
   │───────────────────────────────────►│
   │                                    │
   │◄───────────────────────────────────│
   │ 204 No Content                     │
   │ Access-Control-Allow-Origin: https://app.com
   │ Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   │ Access-Control-Allow-Headers: Content-Type, Authorization
   │ Access-Control-Max-Age: 86400 (cache preflight 24 giờ)
   │                                    │
   │ POST /api/users (actual request)   │
   │───────────────────────────────────►│
```

**Simple requests** (không cần preflight):
- Method: GET, HEAD, POST
- Content-Type: text/plain, multipart/form-data, application/x-www-form-urlencoded
- Không có custom headers

---

## 3. Cấu Hình CORS trong ASP.NET Core

### Named Policy — Chính Sách Có Tên (Khuyến Nghị)

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    // Policy cho development
    options.AddPolicy("DevelopmentPolicy", policy =>
    {
        policy
            .WithOrigins("http://localhost:3000", "http://localhost:5173")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });

    // Policy cho production
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy
            .WithOrigins(
                "https://myapp.com",
                "https://www.myapp.com",
                "https://admin.myapp.com")
            .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .WithHeaders(
                HeaderNames.ContentType,
                HeaderNames.Authorization,
                "X-Request-Id")
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromHours(24));  // Cache preflight
    });
});

var app = builder.Build();

app.UseCors(app.Environment.IsDevelopment()
    ? "DevelopmentPolicy"
    : "ProductionPolicy");
```

### Áp Dụng Policy Khác Nhau Cho Từng Controller

```csharp
// Global policy
app.UseCors("DefaultPolicy");

// Override cho controller cụ thể
[EnableCors("PublicApiPolicy")]
[ApiController]
[Route("api/public")]
public class PublicController : ControllerBase { }

// Tắt CORS cho controller cụ thể
[DisableCors]
[ApiController]
[Route("api/internal")]
public class InternalController : ControllerBase { }
```

---

## 4. CORS Headers Quan Trọng

### Response Headers từ Server

```http
Access-Control-Allow-Origin: https://app.com
  → Hoặc * (wildcard — không dùng với credentials)
  → Chỉ một origin duy nhất (không phải danh sách)

Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
  → Các HTTP method được cho phép

Access-Control-Allow-Headers: Content-Type, Authorization, X-Custom-Header
  → Các request headers được cho phép

Access-Control-Allow-Credentials: true
  → Cho phép gửi cookies/auth headers với cross-origin request

Access-Control-Expose-Headers: X-Total-Count, X-Request-Id
  → Các response headers mà JS được phép đọc (ngoài safe headers)

Access-Control-Max-Age: 86400
  → Browser cache preflight response bao lâu (giây)
```

---

## 5. Lỗi Thường Gặp

### Lỗi 1: Wildcard với Credentials

```csharp
// ❌ KHÔNG HỢP LỆ — Không thể dùng wildcard khi AllowCredentials
policy
    .AllowAnyOrigin()     // origin: *
    .AllowCredentials();  // → Browser từ chối (CORS spec không cho phép)

// ✅ Đúng
policy
    .WithOrigins("https://app.com")  // Phải specify origin cụ thể
    .AllowCredentials();
```

### Lỗi 2: Dynamic Origin (Nhiều Subdomain)

```csharp
// ✅ Cách validate origin động
options.AddPolicy("DynamicOriginPolicy", policy =>
{
    policy
        .SetIsOriginAllowed(origin =>
        {
            var uri = new Uri(origin);

            // Cho phép tất cả subdomains của myapp.com
            return uri.Host.EndsWith(".myapp.com") ||
                   uri.Host == "myapp.com";
        })
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials();
});
```

### Lỗi 3: CORS Không Phải Security Toàn Diện

```
CORS chỉ hoạt động trên BROWSER.
Không ngăn được:
  ❌ Postman / curl / server-to-server requests
  ❌ Custom HTTP clients không tuân thủ CORS
  ❌ Mobile apps (native)

→ CORS chỉ là UX protection, không phải server-side security
→ Vẫn cần Authentication + Authorization trên server
```

---

## 6. Cấu Hình Đúng Cho Từng Scenario

### SPA + API (Cùng Domain)

```csharp
// Nếu SPA và API cùng domain → không cần CORS
// Ví dụ: SPA serve từ https://myapp.com/
//         API expose tại https://myapp.com/api/

// Không cần AddCors!
```

### SPA + API (Khác Domain)

```csharp
options.AddPolicy("SpaPolicy", policy =>
{
    policy
        .WithOrigins("https://spa.myapp.com")
        .WithMethods("GET", "POST", "PUT", "DELETE")
        .WithHeaders(HeaderNames.ContentType, HeaderNames.Authorization)
        .AllowCredentials();
});
```

### Public API (Cho Phép Mọi Origin)

```csharp
options.AddPolicy("PublicApiPolicy", policy =>
{
    policy
        .AllowAnyOrigin()  // Wildcard OK vì không cần credentials
        .WithMethods("GET")
        .WithHeaders(HeaderNames.Accept, HeaderNames.ContentType);
    // KHÔNG AllowCredentials() với AllowAnyOrigin()
});
```

### Microservices Internal API

```csharp
// Internal service-to-service: không cần CORS
// CORS chỉ liên quan đến browser requests
// Server-to-server không bị ràng buộc bởi Same-Origin Policy
```

---

## 7. CORS trong Minimal APIs

```csharp
// Áp dụng policy cho nhóm routes
var publicGroup = app.MapGroup("/api/public")
    .RequireCors("PublicApiPolicy");

var protectedGroup = app.MapGroup("/api/protected")
    .RequireCors("ProductionPolicy")
    .RequireAuthorization();

// Áp dụng cho endpoint cụ thể
app.MapGet("/api/data", () => "data")
    .RequireCors("DefaultPolicy");
```

---

## 8. Kiểm Tra CORS

```bash
# Test CORS bằng curl
curl -v -H "Origin: https://app.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -X OPTIONS https://api.myapp.com/api/users

# Expected response:
# HTTP/1.1 204 No Content
# Access-Control-Allow-Origin: https://app.com
# Access-Control-Allow-Methods: GET, POST, ...
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: CORS là gì và tại sao cần nó?**

```
CORS — Cross-Origin Resource Sharing:
  - Cơ chế cho phép server khai báo những origin nào được phép truy cập
  - Giải quyết Same-Origin Policy restriction của browser
  - Chỉ áp dụng cho browser-based requests (không áp dụng cho server/curl/Postman)

Tại sao cần: SPA (React/Vue) chạy ở https://app.com muốn gọi
             API ở https://api.com → cần server API cho phép origin app.com
```

**Q: Tại sao không dùng `AllowAnyOrigin()` cho mọi API?**

```
AllowAnyOrigin() = Access-Control-Allow-Origin: *
Vấn đề:
  - Không thể dùng với AllowCredentials() (cookies, auth headers)
  - Bất kỳ website nào cũng có thể gọi API của bạn từ browser
  - Kẻ tấn công tạo website độc hại, dùng JS gọi API của bạn với session user

Giải pháp:
  - Chỉ whitelist origins cụ thể bạn tin tưởng
  - Dùng SetIsOriginAllowed() nếu cần dynamic origins
```

**Q: Preflight request là gì?**

```
Preflight request = OPTIONS request browser gửi TRƯỚC actual request
để hỏi server: "Tôi có được phép gửi POST với Content-Type: application/json không?"

Browser tự động gửi preflight cho:
  - Method: PUT, DELETE, PATCH
  - Content-Type không phải simple types
  - Custom headers (như Authorization)

Tối ưu preflight:
  - Access-Control-Max-Age: 86400 → cache 24 giờ, không preflight lại
```

---

## ✅ Checklist CORS

```
✅ Không dùng AllowAnyOrigin() cho API có authentication
✅ Không hardcode origins — đọc từ configuration
✅ Chỉ AllowCredentials() khi thực sự cần cookies
✅ SetPreflightMaxAge để giảm preflight requests
✅ Expose chỉ các headers cần thiết
✅ Different policies cho dev/staging/production
✅ Test CORS với curl hoặc browser DevTools
✅ Validate origins theo pattern nếu có nhiều subdomains
```

---

**Xem Tiếp:** [8-secure-coding.md](8-secure-coding.md) — Secure Coding Practices
