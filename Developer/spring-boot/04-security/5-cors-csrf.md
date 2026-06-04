# CORS & CSRF — Bảo Vệ API Khỏi Tấn Công Cross-Origin

> **CORS** (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Chéo Nguồn Gốc) và
> **CSRF** (Cross-Site Request Forgery — Giả Mạo Yêu Cầu Chéo Site) là hai khía cạnh bảo mật
> web quan trọng, thường bị hiểu nhầm và cấu hình sai.

---

## Phần 1: CORS (Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Chéo Nguồn Gốc)

### 1.1 CORS Là Gì?

CORS là cơ chế của **trình duyệt** (không phải server) kiểm soát việc trang web từ **origin** (nguồn gốc) này có được phép gọi API ở **origin** khác không.

**Origin** = scheme + host + port:
```
https://app.example.com:443  ← Origin A
https://api.example.com:443  ← Origin B (khác subdomain = khác origin)
http://app.example.com:80    ← Origin C (khác scheme)
https://app.example.com:8080 ← Origin D (khác port)
```

**Vấn đề:** Browser từ chối request cross-origin theo mặc định do **Same-Origin Policy** (Chính Sách Cùng Nguồn Gốc).

### 1.2 Preflight Request (Yêu Cầu Kiểm Tra Trước)

Với "non-simple" requests (PUT, DELETE, custom headers, JSON body), browser gửi **OPTIONS request** trước để hỏi server có cho phép không:

```
Browser                              API Server
   │                                      │
   │── OPTIONS /api/orders ──────────────►│
   │   Origin: https://app.example.com    │
   │   Access-Control-Request-Method: POST│
   │   Access-Control-Request-Headers: Authorization, Content-Type
   │                                      │
   │◄── 200 OK ───────────────────────────│
   │   Access-Control-Allow-Origin: https://app.example.com
   │   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   │   Access-Control-Allow-Headers: Authorization, Content-Type
   │   Access-Control-Max-Age: 3600
   │                                      │
   │── POST /api/orders ─────────────────►│  ← Actual request
```

---

### 1.3 Cấu Hình CORS Trong Spring Boot

#### Cách 1: `CorsConfigurationSource` trong SecurityFilterChain (Khuyến Nghị)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable)
            // ...
        ;
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        // Allowed origins — các origin được phép gọi API
        config.setAllowedOrigins(List.of(
            "https://app.example.com",
            "https://admin.example.com"
        ));

        // Hoặc dùng pattern (cho môi trường dev)
        // config.setAllowedOriginPatterns(List.of("http://localhost:*"));

        // Allowed methods
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));

        // Allowed headers
        config.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "Accept"
        ));

        // Expose headers — headers client có thể đọc từ response
        config.setExposedHeaders(List.of("X-Total-Count", "X-Page-Number"));

        // Allow credentials (cookies, authorization headers)
        config.setAllowCredentials(true);

        // Cache preflight response trong 1 giờ
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);   // Chỉ áp dụng cho /api/**

        return source;
    }
}
```

#### Cách 2: `@CrossOrigin` Trên Controller (Scope nhỏ hơn)

```java
@RestController
@RequestMapping("/api/products")
@CrossOrigin(
    origins = "https://app.example.com",
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = {"Authorization", "Content-Type"},
    maxAge = 3600
)
public class ProductController {

    // Override cấu hình của class cho một method cụ thể
    @GetMapping("/public")
    @CrossOrigin(origins = "*")     // Public endpoint cho phép tất cả
    public List<Product> getPublicProducts() { ... }
}
```

#### Cách 3: `WebMvcConfigurer` (Không Dùng Spring Security)

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

> **Quan trọng:** Khi dùng Spring Security, CORS phải được cấu hình **qua Spring Security** (cách 1), không phải `WebMvcConfigurer` — vì `SecurityFilterChain` chạy trước `DispatcherServlet`.

---

### 1.4 CORS Cho Môi Trường Phát Triển (Development)

```yaml
# application-dev.yml
app:
  cors:
    allowed-origins: "http://localhost:3000,http://localhost:4200,http://localhost:5173"
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Value("${app.cors.allowed-origins}")
    private String[] allowedOrigins;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(allowedOrigins));
        config.setAllowedMethods(List.of("*"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

### 1.5 Lỗi CORS Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-------------|----------|
| `No 'Access-Control-Allow-Origin' header` | CORS chưa được cấu hình | Thêm `CorsConfigurationSource` |
| `Credential flag is 'true' but 'Access-Control-Allow-Origin' is '*'` | Không thể dùng `*` với credentials | Chỉ định origin cụ thể |
| CORS error dù đã cấu hình | `WebMvcConfigurer` bị Spring Security override | Cấu hình CORS qua `HttpSecurity.cors()` |
| Preflight 403 Forbidden | Spring Security block OPTIONS request | Đảm bảo `.cors()` được cấu hình trước authentication |

---

## Phần 2: CSRF (Cross-Site Request Forgery — Giả Mạo Yêu Cầu Chéo Site)

### 2.1 CSRF Tấn Công Thế Nào?

CSRF lợi dụng việc browser tự động gửi cookies khi request đến server:

```
1. Nạn nhân đăng nhập bank.com → nhận session cookie
2. Nạn nhân truy cập evil.com (trang độc hại)
3. evil.com chứa form ẩn:
   <form action="https://bank.com/transfer" method="POST">
     <input name="amount" value="1000000">
     <input name="to" value="attacker_account">
   </form>
   <script>document.forms[0].submit();</script>
4. Browser tự động gửi session cookie → bank.com tin đây là request hợp lệ
5. Tiền bị chuyển đi!
```

### 2.2 CSRF vs REST API

**CSRF không ảnh hưởng đến stateless REST API** khi:
- Dùng JWT trong `Authorization: Bearer` header (browser không tự gửi header)
- **Không** dùng session cookie để xác thực

```java
// Stateless REST API dùng JWT — có thể tắt CSRF
http.csrf(AbstractHttpConfigurer::disable)
    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

**CSRF CẦN được bật** khi:
- Dùng session-based authentication (cookie `JSESSIONID`)
- Có form HTML truyền thống
- Dùng `remember-me` cookie

### 2.3 CSRF Protection trong Spring Security

Spring Security mặc định bật CSRF protection với **Synchronizer Token Pattern** (Mẫu Token Đồng Bộ):

```java
// Bật CSRF (mặc định) với cấu hình tùy chỉnh
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // Bỏ qua CSRF cho một số endpoint
    .ignoringRequestMatchers("/api/webhooks/**", "/public/**")
);
```

### 2.4 CookieCsrfTokenRepository — Cho SPA (Single Page Application)

Khi frontend là SPA (React, Angular, Vue), dùng cookie-based CSRF:

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // Cookie có HttpOnly=false để JavaScript đọc được
);
```

Frontend đọc `XSRF-TOKEN` cookie và gửi lại trong header:
```javascript
// Angular tự động làm điều này
// React — cần tự làm:
const csrfToken = document.cookie
    .split('; ')
    .find(row => row.startsWith('XSRF-TOKEN='))
    ?.split('=')[1];

fetch('/api/orders', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': csrfToken      // ← Gửi token trong header
    },
    credentials: 'include',             // ← Gửi kèm cookie
    body: JSON.stringify(orderData)
});
```

### 2.5 CSRF Token Trong Thymeleaf

Spring Security tự động inject CSRF token vào form Thymeleaf:

```html
<!-- Thymeleaf tự thêm hidden input -->
<form th:action="@{/transfer}" method="post">
    <!-- Spring Security tự inject: <input type="hidden" name="_csrf" value="xxx"> -->
    <input name="amount" type="number">
    <button type="submit">Chuyển tiền</button>
</form>
```

---

### 2.6 SameSite Cookie — Cách Hiện Đại Chống CSRF

Thuộc tính `SameSite` của cookie ngăn browser gửi cookie trong cross-site requests:

```java
@Configuration
public class SessionConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setSameSite("Strict");   // Không gửi trong mọi cross-site request
        // serializer.setSameSite("Lax"); // Chỉ gửi khi navigate (click link), không gửi POST
        serializer.setHttpOnly(true);
        serializer.setSecure(true);        // Chỉ gửi qua HTTPS
        return serializer;
    }
}
```

| SameSite Value | Hành Vi | Phù Hợp Với |
|----------------|---------|-------------|
| `Strict` | Không gửi cookie trong bất kỳ cross-site request nào | App không cần cross-site navigation |
| `Lax` | Gửi khi top-level navigation (GET), không gửi POST | Cân bằng giữa bảo mật và UX |
| `None` | Luôn gửi (cần `Secure=true`) | API cho bên thứ ba |

---

## 3. Tổng Hợp: CORS vs CSRF

| Khía Cạnh | CORS | CSRF |
|-----------|------|------|
| **Mục đích** | Kiểm soát origin nào được gọi API | Ngăn request giả mạo từ site khác |
| **Thực thi tại** | Browser (Same-Origin Policy) | Server (token/cookie validation) |
| **Liên quan đến** | Đọc response cross-origin | Thực hiện action thay mặt user |
| **Bị khai thác khi** | Server cho phép origin không tin cậy | Dùng session cookie không có CSRF token |
| **Với JWT Bearer** | Cần cấu hình đúng | Không cần (browser không tự gửi header) |

---

## 4. Cấu Hình Hoàn Chỉnh

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // CORS — cấu hình trước authentication
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // CSRF — tắt cho REST API stateless dùng JWT
            .csrf(AbstractHttpConfigurer::disable)

            // Hoặc bật CSRF cho ứng dụng web truyền thống:
            // .csrf(csrf -> csrf
            //     .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            //     .ignoringRequestMatchers("/auth/**")
            // )

            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // Cho phép preflight
                .requestMatchers("/auth/**", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(s ->
                s.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "https://app.example.com",
            "https://admin.example.com"
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 5. Checklist CORS & CSRF

- [ ] Hiểu CORS là cơ chế của browser, không phải server-side security
- [ ] Cấu hình CORS qua `HttpSecurity.cors()` khi dùng Spring Security
- [ ] Không dùng `allowedOrigins("*")` với `allowCredentials(true)`
- [ ] Hiểu khi nào cần bật/tắt CSRF
- [ ] Biết dùng `CookieCsrfTokenRepository` cho SPA
- [ ] Cấu hình `SameSite` cookie attribute cho production
- [ ] Luôn permit `OPTIONS` requests (preflight) trong authorization rules

---

## 🔗 Bài Tiếp Theo

→ [6-security-best-practices.md](6-security-best-practices.md) — Security Best Practices & Production Hardening
