# Bảo Mật Node.js API — Tổng Quan

> Chủ đề bảo mật cho Backend Node.js: Authentication (Xác Thực), Authorization (Phân Quyền), OWASP Top 10, security headers, rate limiting, input validation, và secrets management (quản lý bí mật).

## Mục Lục

1. [Tại Sao Bảo Mật Quan Trọng](#tại-sao-bảo-mật-quan-trọng)
2. [Defense in Depth — Phòng Thủ Nhiều Lớp](#defense-in-depth--phòng-thủ-nhiều-lớp)
3. [Kiến Trúc Bảo Mật Tổng Quan](#kiến-trúc-bảo-mật-tổng-quan)
4. [Authentication vs Authorization](#authentication-vs-authorization)
5. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
6. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
7. [Bài Tập Thực Hành](#bài-tập-thực-hành)
8. [Security Checklist Production](#security-checklist-production)
9. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Bảo Mật Quan Trọng

API Backend là **attack surface (bề mặt tấn công)** trực tiếp — mọi endpoint public đều có thể bị khai thác. Một lỗ hổng nhỏ có thể dẫn đến:

| Rủi Ro | Hậu Quả |
| ------ | ------- |
| **Broken Authentication** | Account takeover, session hijacking |
| **SQL Injection** | Toàn bộ database bị dump |
| **Missing Authorization** | User A truy cập data của User B |
| **Secrets in Code** | API keys, DB passwords lộ trên GitHub |
| **No Rate Limiting** | Brute-force password, DoS (Denial of Service — Từ Chối Dịch Vụ) |
| **XSS / CSRF** | Steal tokens, thực hiện hành động thay user |

**Nguyên tắc cốt lõi:** Không tin tưởng client input. Mọi request đều phải được authenticate, authorize, validate, và log.

---

## Defense in Depth — Phòng Thủ Nhiều Lớp

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                                 │
│                                                                 │
│  Internet ──► WAF / CDN ──► Rate Limiter ──► HTTPS (TLS)       │
│                                      │                          │
│                                      ▼                          │
│                              Helmet (Headers)                   │
│                                      │                          │
│                                      ▼                          │
│                         Input Validation (Zod)                  │
│                                      │                          │
│                                      ▼                          │
│                    Authentication (JWT / Session)               │
│                                      │                          │
│                                      ▼                          │
│                    Authorization (RBAC / Permissions)           │
│                                      │                          │
│                                      ▼                          │
│              Business Logic + Parameterized Queries             │
│                                      │                          │
│                                      ▼                          │
│                    Secrets Manager + Encrypted Storage          │
└─────────────────────────────────────────────────────────────────┘
```

Mỗi lớp bảo vệ độc lập — nếu một lớp fail, lớp khác vẫn giữ an toàn.

---

## Kiến Trúc Bảo Mật Tổng Quan

### Luồng JWT Authentication Điển Hình

```
┌────────┐    POST /login     ┌─────────────┐
│ Client │ ─────────────────► │  Auth API   │
└────────┘                    └──────┬──────┘
     ▲                               │ Verify credentials (bcrypt)
     │                               ▼
     │                        ┌─────────────┐
     │  access_token (15m)    │  Issue JWT  │
     │  refresh_token (7d)    └──────┬──────┘
     └──────────────────────────────┘

┌────────┐  GET /api/users  Authorization: Bearer <token>
│ Client │ ─────────────────────────────────────────────► Protected API
└────────┘                              │
                                        ▼
                              Verify JWT signature + exp
                                        │
                                        ▼
                              Check RBAC permissions
                                        │
                                        ▼
                                   Return data
```

---

## Authentication vs Authorization

| Khái Niệm | Câu Hỏi Trả Lời | Ví Dụ |
| --------- | --------------- | ----- |
| **Authentication (Xác Thực)** | *Bạn là ai?* | Login với email/password, verify JWT |
| **Authorization (Phân Quyền)** | *Bạn được phép làm gì?* | Admin mới xóa user, user chỉ xem data của mình |

**Lỗi phổ biến:** Chỉ check "đã login" mà không check "có quyền thao tác resource này không" → Broken Access Control (OWASP A01).

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-authentication-jwt.md](./1-authentication-jwt.md) | JWT access/refresh tokens, token rotation, bcrypt | 1.5 giờ |
| 2 | [2-passport-strategies.md](./2-passport-strategies.md) | Passport.js — Local, JWT, OAuth2 strategies | 1 giờ |
| 3 | [3-authorization-rbac.md](./3-authorization-rbac.md) | RBAC, permissions, resource-level access | 1 giờ |
| 4 | [4-owasp-top10.md](./4-owasp-top10.md) | OWASP Top 10 áp dụng với Node.js | 1.5 giờ |
| 5 | [5-helmet-rate-limiting.md](./5-helmet-rate-limiting.md) | Helmet headers, CORS, rate limiting algorithms | 1 giờ |
| 6 | [6-input-validation.md](./6-input-validation.md) | Sanitization, whitelist, SQL parameterization | 1 giờ |
| 7 | [7-secrets-management.md](./7-secrets-management.md) | env vars, Vault, AWS Secrets Manager | 1 giờ |

**Thứ tự học khuyến nghị:** 1 → 3 → 4 → 5 → 6 → 2 → 7. Nắm JWT và RBAC trước; Passport khi cần OAuth2/social login.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-authentication-jwt.md](./1-authentication-jwt.md) | JWT structure, RS256 vs HS256, refresh token rotation, HttpOnly cookies |
| [2-passport-strategies.md](./2-passport-strategies.md) | `passport-local`, `passport-jwt`, OAuth2 Google/GitHub |
| [3-authorization-rbac.md](./3-authorization-rbac.md) | Roles, permissions matrix, middleware guards, ABAC |
| [4-owasp-top10.md](./4-owasp-top10.md) | SQL Injection, XSS, CSRF, SSRF prevention trong Node.js |
| [5-helmet-rate-limiting.md](./5-helmet-rate-limiting.md) | `helmet`, `express-rate-limit`, token bucket, sliding window |
| [6-input-validation.md](./6-input-validation.md) | Zod whitelist, `express-mongo-sanitize`, NoSQL injection |
| [7-secrets-management.md](./7-secrets-management.md) | `dotenv`, 12-factor app, HashiCorp Vault, rotation |

---

## Bài Tập Thực Hành

### Lab 1: JWT Auth Flow Hoàn Chỉnh (2 giờ)

```bash
mkdir auth-lab && cd auth-lab && npm init -y
npm install express jsonwebtoken bcryptjs zod dotenv
# Implement: register, login, refresh, logout, protected routes
```

### Lab 2: RBAC Middleware (1 giờ)

```typescript
// roles: admin, editor, viewer
// Middleware requireRole('admin') cho DELETE /users/:id
// User chỉ GET /users/:id nếu id === req.user.id hoặc là admin
```

### Lab 3: Security Hardening Express API (1.5 giờ)

```bash
npm install helmet express-rate-limit cors express-mongo-sanitize
# Thêm security headers, rate limit /login, CORS whitelist
```

### Lab 4: Secrets với dotenv + Validation (30 phút)

```typescript
// Zod schema validate process.env at startup
// Fail fast nếu thiếu JWT_SECRET, DATABASE_URL
```

---

## Security Checklist Production

- [ ] HTTPS everywhere — TLS 1.2+ (không HTTP trong production)
- [ ] JWT access token ngắn hạn (15 phút), refresh token rotation
- [ ] Password hash với bcrypt (cost ≥ 12) hoặc Argon2
- [ ] Parameterized queries — không concatenate SQL
- [ ] Input validation với Zod/Joi ở mọi endpoint
- [ ] Helmet security headers enabled
- [ ] Rate limiting trên `/login`, `/register`, public APIs
- [ ] CORS whitelist — không dùng `origin: *` cho authenticated API
- [ ] Secrets trong env vars hoặc secrets manager — không commit `.env`
- [ ] Dependency scanning (`npm audit`, Dependabot)
- [ ] Security logging — auth failures, access denied, validation errors
- [ ] Error messages generic trong production — không leak stack trace

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: JWT vs Session-based auth — trade-offs?

**Gợi ý trả lời:** JWT stateless — scale horizontal dễ, không cần session store; nhưng khó revoke token trước expiry (cần blacklist hoặc short TTL + refresh). Session server-side — revoke dễ, nhưng cần Redis/session store khi scale. JWT phù hợp microservices/API; session phù hợp monolith truyền thống.

### Câu 2: Làm sao prevent SQL Injection trong Node.js?

**Gợi ý trả lời:** Parameterized queries (`$1`, `$2` với `pg`), ORM prepared statements, validate input với Zod. Không bao giờ `query(\`SELECT * FROM users WHERE id = ${id}\`)`.

### Câu 3: Access token lưu ở đâu an toàn?

**Gợi ý trả lời:** HttpOnly + Secure + SameSite cookie (chống XSS steal token) hoặc memory (SPA). Tránh localStorage nếu có XSS risk. Refresh token chỉ HttpOnly cookie.

### Câu 4: RBAC vs ABAC?

**Gợi ý trả lời:** RBAC (Role-Based Access Control — Phân Quyền Theo Vai Trò) gán permission theo role (`admin`, `user`). ABAC (Attribute-Based Access Control — Phân Quyền Theo Thuộc Tính) dựa trên attributes động (owner, department, time). RBAC đơn giản hơn; ABAC linh hoạt hơn cho enterprise.

### Câu 5: Rate limiting algorithm nào phổ biến?

**Gợi ý trả lời:** Token bucket và sliding window log. Fixed window đơn giản nhưng có burst ở boundary. Redis `INCR` + `EXPIRE` cho distributed rate limiting.

---

**Xem tiếp:** [1-authentication-jwt.md](./1-authentication-jwt.md) — bắt đầu với JWT authentication flow.
