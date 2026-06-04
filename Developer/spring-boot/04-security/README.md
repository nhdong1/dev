# 04 — Bảo Mật (Security)

> Module này bao gồm toàn bộ kiến thức về bảo mật trong Spring Boot — từ Spring Security Filter Chain,
> xác thực JWT, tích hợp OAuth2/OIDC, method-level security, cấu hình CORS/CSRF, đến các best practices
> về bảo mật trong môi trường production thực tế.

---

## 📋 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Cấu hình **Spring Security** — `SecurityFilterChain` (Chuỗi Bộ Lọc Bảo Mật), `AuthenticationManager` (Trình Quản Lý Xác Thực)
- [ ] Implement **JWT** (JSON Web Token) — Access Token, Refresh Token, `JwtFilter` từ đầu
- [ ] Tích hợp **OAuth2** (Open Authorization — Ủy Quyền Mở) & **OIDC** (OpenID Connect) với Resource Server
- [ ] Áp dụng **Method Security** (Bảo Mật Cấp Phương Thức) — `@PreAuthorize`, `@PostAuthorize`, `@Secured`
- [ ] Cấu hình **CORS** (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Chéo Nguồn Gốc) và **CSRF** (Cross-Site Request Forgery — Giả Mạo Yêu Cầu Chéo Site)
- [ ] Áp dụng security best practices — BCrypt, secrets management, HTTP security headers

---

## 🗂️ Danh Sách Bài Học

| File | Chủ Đề | Thời Gian | Độ Khó |
|------|--------|-----------|--------|
| [1-spring-security-basics.md](1-spring-security-basics.md) | SecurityFilterChain, AuthenticationManager, UserDetailsService, PasswordEncoder | 120 phút | ⭐⭐ |
| [2-jwt-implementation.md](2-jwt-implementation.md) | JWT Access/Refresh Token, JwtFilter, Token Rotation, Blacklist | 120 phút | ⭐⭐⭐ |
| [3-oauth2-oidc.md](3-oauth2-oidc.md) | OAuth2 Flows, Resource Server, Spring Authorization Server, OIDC | 90 phút | ⭐⭐⭐ |
| [4-method-security.md](4-method-security.md) | @PreAuthorize, @PostAuthorize, @Secured, SpEL expressions | 60 phút | ⭐⭐ |
| [5-cors-csrf.md](5-cors-csrf.md) | CORS configuration, CSRF protection, SameSite cookies | 60 phút | ⭐⭐ |
| [6-security-best-practices.md](6-security-best-practices.md) | BCrypt, secrets management, security headers, hardening checklist | 90 phút | ⭐⭐⭐ |

**Tổng thời gian ước tính: 8–10 giờ**

---

## 🔁 Thứ Tự Học Khuyến Nghị

```
1-spring-security-basics.md    ← Nền tảng — học trước tiên
      ↓
2-jwt-implementation.md        ← Xác thực stateless phổ biến nhất
      ↓
3-oauth2-oidc.md               ← Delegated auth — bước tiếp theo
      ↓
4-method-security.md           ← Fine-grained authorization
      ↓
5-cors-csrf.md                 ← Cấu hình cho REST API & SPA
      ↓
6-security-best-practices.md   ← Hardening production-ready
```

---

## 🧠 Kiến Trúc Bảo Mật Spring Boot

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────┐
│           Security Filter Chain                  │
│  ┌──────────────────────────────────────────┐   │
│  │ SecurityContextPersistenceFilter          │   │
│  │ UsernamePasswordAuthenticationFilter      │   │
│  │ JwtAuthenticationFilter (custom)          │   │
│  │ ExceptionTranslationFilter                │   │
│  │ FilterSecurityInterceptor                 │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────┐
│  DispatcherServlet  │
│  @RestController    │
│  Method Security    │   ← @PreAuthorize / @PostAuthorize
└─────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### Authentication (Xác Thực) vs Authorization (Phân Quyền)

| Khái Niệm | Câu Hỏi | Ví Dụ |
|-----------|---------|-------|
| **Authentication** (Xác Thực) | *"Bạn là ai?"* | Đăng nhập với username/password |
| **Authorization** (Phân Quyền) | *"Bạn được phép làm gì?"* | User có ROLE_ADMIN mới xóa được |

### Principal (Chủ Thể) & Credentials (Thông Tin Xác Thực)

```
Authentication Object
├── Principal   → UserDetails (ai đang đăng nhập)
├── Credentials → password (thường xóa sau xác thực)
└── Authorities → [ROLE_USER, ROLE_ADMIN] (quyền hạn)
```

### Security Context (Ngữ Cảnh Bảo Mật)

```java
// Lấy thông tin người dùng hiện tại
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
```

---

## 🔐 Luồng Xác Thực JWT Điển Hình

```
Client                    Spring Boot Server
  │                              │
  │── POST /auth/login ─────────►│
  │   {username, password}       │ AuthenticationManager.authenticate()
  │                              │ UserDetailsService.loadUserByUsername()
  │                              │ PasswordEncoder.matches()
  │◄─ 200 OK ───────────────────│
  │   {accessToken, refreshToken}│
  │                              │
  │── GET /api/orders ──────────►│
  │   Authorization: Bearer xxx  │ JwtFilter.doFilterInternal()
  │                              │ JwtUtil.validateToken()
  │                              │ SecurityContext.setAuthentication()
  │◄─ 200 OK [data] ────────────│
```

---

## ⚠️ Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-------------|----------|
| 403 Forbidden dù đã đăng nhập | Thiếu ROLE_ prefix trong authority | Dùng `ROLE_USER` hoặc `hasAuthority('USER')` |
| JWT token bị reject | Clock skew giữa các server | Thêm `clockSkewSeconds` vào validator |
| CORS error từ browser | Thiếu cấu hình CORS | Thêm `@CrossOrigin` hoặc `CorsConfigurationSource` |
| `@PreAuthorize` không hoạt động | Thiếu `@EnableMethodSecurity` | Thêm annotation vào `@Configuration` class |
| Session không persist | Dùng stateless JWT nhưng còn `SessionManagementFilter` | Set `SessionCreationPolicy.STATELESS` |

---

## 📊 So Sánh Chiến Lược Xác Thực

| Chiến Lược | Stateful/Stateless | Phù Hợp Với | Nhược Điểm |
|-----------|-------------------|-------------|------------|
| **Session** (Phiên) | Stateful | Monolith, server-side render | Khó scale horizontally |
| **JWT** | Stateless | REST API, microservices, mobile | Không revoke được ngay |
| **OAuth2** | Stateless | Đăng nhập qua bên thứ ba | Phức tạp hơn |
| **API Key** | Stateless | Machine-to-machine | Không có expiry tự động |

---

## 📚 Tài Liệu Tham Khảo

- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [OAuth2 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [JWT RFC 7519](https://tools.ietf.org/html/rfc7519)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture/)

---

## 🎯 Câu Hỏi Phỏng Vấn Quan Trọng

- [ ] Spring Security Filter Chain hoạt động thế nào? Thứ tự các filter?
- [ ] Sự khác biệt giữa Authentication và Authorization?
- [ ] JWT Refresh Token strategy — cách implement token rotation?
- [ ] OAuth2 Authorization Code Flow (Luồng Mã Ủy Quyền) hoạt động thế nào?
- [ ] Cách prevent CSRF trong REST API?
- [ ] `@PreAuthorize` vs `@Secured` — khi nào dùng cái nào?
- [ ] Cách lưu trữ secret trong production an toàn?
- [ ] BCrypt cost factor ảnh hưởng thế nào đến hiệu năng?
