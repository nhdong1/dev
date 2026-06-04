# Bảo Mật trong ASP.NET Core — Tổng Quan

> Bảo mật là lớp phòng vệ nhiều tầng. Không có giải pháp đơn lẻ nào bảo vệ toàn diện — bạn cần kết hợp Authentication, Authorization, HTTPS, Input Validation và nhiều cơ chế khác.

---

## 📚 Nội Dung Chương Này

| File | Chủ Đề | Độ Khó |
| ---- | ------- | ------ |
| [1-jwt-authentication.md](1-jwt-authentication.md) | JWT — JSON Web Token — cấu trúc, signing, validation | ⭐⭐ |
| [2-aspnet-identity.md](2-aspnet-identity.md) | ASP.NET Core Identity — quản lý user, role, claim | ⭐⭐ |
| [3-oauth2-openidconnect.md](3-oauth2-openidconnect.md) | OAuth 2.0, OpenID Connect, authorization code flow | ⭐⭐⭐ |
| [4-authorization-policies.md](4-authorization-policies.md) | Policy-based, claims-based, resource-based authorization | ⭐⭐ |
| [5-data-protection.md](5-data-protection.md) | ASP.NET Core Data Protection API, key rotation | ⭐⭐ |
| [6-https-and-tls.md](6-https-and-tls.md) | HTTPS enforcement, HSTS, certificate management | ⭐⭐ |
| [7-cors.md](7-cors.md) | CORS — Cross-Origin Resource Sharing — cấu hình đúng cách | ⭐ |
| [8-secure-coding.md](8-secure-coding.md) | Input validation, SQL Injection, XSS, CSRF prevention | ⭐⭐ |

---

## 🔐 Authentication vs Authorization

Hai khái niệm này thường bị nhầm lẫn:

```
Authentication — Xác Thực                 Authorization — Phân Quyền
─────────────────────────────             ────────────────────────────────
"Bạn là ai?"                              "Bạn được làm gì?"

Kiểm tra danh tính:                       Kiểm tra quyền hạn:
  - Username / Password                     - Role: "Admin", "User"
  - JWT Token                               - Policy: "CanDeletePost"
  - API Key                                 - Resource ownership
  - Certificate                             - Claim: "department=HR"

Xảy ra TRƯỚC                             Xảy ra SAU authentication
```

---

## 🏛️ Kiến Trúc Bảo Mật ASP.NET Core

```
Request đến
    │
    ▼
┌──────────────────────────────────────────────┐
│  HTTPS / TLS — Mã Hóa Truyền Tải            │  ← Tầng 1: Transport Security
│  (certificate, HSTS)                         │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│  CORS Middleware                             │  ← Tầng 2: Origin Control
│  (cho phép/từ chối cross-origin requests)   │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│  Authentication Middleware                   │  ← Tầng 3: Xác Thực
│  JWT Bearer / Cookie / OAuth2                │
│  → Gắn ClaimsPrincipal vào HttpContext.User  │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│  Authorization Middleware                    │  ← Tầng 4: Phân Quyền
│  [Authorize] / Policy / Resource-based       │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│  Input Validation & Sanitization             │  ← Tầng 5: Dữ liệu đầu vào
│  (SQL Injection, XSS, CSRF)                  │
└──────────────────────────────────────────────┘
    │
    ▼
  Controller / Handler
```

---

## 🔑 Các Cơ Chế Xác Thực Phổ Biến

### 1. JWT — JSON Web Token

```
Phù hợp: API stateless, microservices, mobile apps
Không phù hợp: Cần revoke token ngay lập tức

Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0In0.abc123
```

### 2. Cookie Authentication

```
Phù hợp: Web app truyền thống, SSR — Server-Side Rendering
Không phù hợp: Mobile app, cross-domain API

Set-Cookie: .AspNetCore.Auth=<encrypted>; HttpOnly; Secure; SameSite=Strict
```

### 3. OAuth 2.0 + OpenID Connect — OIDC

```
Phù hợp: Đăng nhập bằng Google/Microsoft/GitHub, SSO — Single Sign-On
Không phù hợp: Ứng dụng đơn giản không cần third-party auth

Authorization Server (Google/Azure AD) ──► Access Token + ID Token
```

### 4. API Key

```
Phù hợp: Machine-to-machine, B2B integrations, developer APIs
Không phù hợp: User-facing apps

X-Api-Key: sk-prod-abc123...
```

---

## 🛡️ Defense in Depth — Bảo Vệ Theo Chiều Sâu

Nguyên tắc: **không dựa vào một lớp bảo vệ duy nhất**.

```
Lớp 1: Network          Firewall, VPC, private subnet
Lớp 2: Transport        HTTPS/TLS, certificate pinning
Lớp 3: Application      Authentication + Authorization
Lớp 4: Data             Encryption at rest, hashing passwords
Lớp 5: Validation       Input sanitization, parameterized queries
Lớp 6: Monitoring       Audit logs, anomaly detection, alerting
```

---

## 🚨 OWASP Top 10 — Mười Lỗ Hổng Bảo Mật Phổ Biến Nhất

| # | Lỗ Hổng | Ví Dụ .NET |
|---|---------|-----------|
| A01 | Broken Access Control — Kiểm soát truy cập bị phá vỡ | Thiếu `[Authorize]`, IDOR |
| A02 | Cryptographic Failures — Lỗi mã hóa | Lưu password plain text, HTTP thay HTTPS |
| A03 | Injection — Tấn công tiêm mã | SQL Injection, Command Injection |
| A04 | Insecure Design — Thiết kế không an toàn | Thiếu rate limiting, không có audit log |
| A05 | Security Misconfiguration — Cấu hình sai | Debug mode trên production, exposed endpoints |
| A06 | Vulnerable Components — Thư viện lỗi thời | NuGet packages chưa vá lỗ hổng |
| A07 | Auth Failures — Lỗi xác thực | Session fixation, weak JWT secret |
| A08 | Integrity Failures — Lỗi toàn vẹn | Deserialization attacks, CI/CD pipeline attacks |
| A09 | Logging Failures — Thiếu ghi log | Không log failed login, không monitor |
| A10 | SSRF — Server-Side Request Forgery | Fetch URL từ input người dùng |

---

## 📋 Security Checklist — Danh Sách Kiểm Tra Bảo Mật

### Trước Khi Deploy

```
Authentication & Authorization
  ✅ Tất cả endpoint nhạy cảm có [Authorize]
  ✅ JWT có expiration time hợp lý (15–60 phút)
  ✅ Secret key đủ mạnh (≥ 256-bit), không hardcode
  ✅ Password được hash với BCrypt / Argon2 / PBKDF2
  ✅ Role và Policy được kiểm tra đúng chỗ

Transport Security
  ✅ HTTPS enforced, redirect HTTP → HTTPS
  ✅ HSTS header được bật
  ✅ TLS 1.2+ (không dùng TLS 1.0/1.1, SSL)

Input & Output
  ✅ Parameterized queries (không string concatenation SQL)
  ✅ HTML encoding output để chống XSS
  ✅ CSRF token trên mọi form POST
  ✅ File upload: validate type, size, scan virus

Configuration
  ✅ Secrets trong Azure Key Vault / environment variables
  ✅ CORS chỉ cho phép origin cụ thể (không wildcard *)
  ✅ Error messages không expose stack trace
  ✅ Swagger/OpenAPI disable trên production hoặc protect

Logging & Monitoring
  ✅ Log mọi failed authentication
  ✅ Không log sensitive data (password, token, credit card)
  ✅ Có alert khi có nhiều failed login (brute force)
```

---

## 🔗 Liên Kết Liên Quan

| Chủ Đề | File |
|--------|------|
| ASP.NET Core Middleware Pipeline | [../04-aspnet-core/1-middleware-pipeline.md](../04-aspnet-core/1-middleware-pipeline.md) |
| Dependency Injection | [../04-aspnet-core/2-dependency-injection.md](../04-aspnet-core/2-dependency-injection.md) |
| Configuration & Secrets | [../04-aspnet-core/7-configuration.md](../04-aspnet-core/7-configuration.md) |
| Filters (Authorization Filter) | [../04-aspnet-core/6-filters.md](../04-aspnet-core/6-filters.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-02
