# Filters & Interceptors — OncePerRequestFilter & HandlerInterceptor

> Bài này giải thích sự khác biệt giữa **Filter** (Bộ Lọc) và **Interceptor** (Bộ Chặn),
> thứ tự thực thi, và khi nào nên dùng cái nào trong Spring Boot.

---

## 📋 Mục Tiêu

- [ ] Phân biệt **Filter** (Servlet Filter) và **Interceptor** (Spring MVC Interceptor)
- [ ] Implement `OncePerRequestFilter` — Filter chỉ chạy một lần mỗi request
- [ ] Implement `HandlerInterceptor` với `preHandle`, `postHandle`, `afterCompletion`
- [ ] Hiểu thứ tự thực thi: Filter → Interceptor → Controller
- [ ] Biết khi nào dùng Filter vs Interceptor

---

## 1. Tổng Quan: Filter vs Interceptor

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Servlet Filter Chain (Chuỗi Bộ Lọc Servlet)                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                         │
│  │ Filter 1 │ → │ Filter 2 │ → │ Filter 3 │  ...                    │
│  └──────────┘   └──────────┘   └──────────┘                         │
│   (logging)      (cors)         (security)                           │
│                                                                       │
│  Chạy ở tầng Servlet — trước Spring MVC                              │
│  Có thể dùng với mọi Servlet (không chỉ Spring)                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  DispatcherServlet (Servlet Điều Phối Trung Tâm của Spring MVC)      │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  HandlerInterceptor Chain (Chuỗi Bộ Chặn Handler)            │    │
│  │  ┌─────────────┐   ┌─────────────┐                           │    │
│  │  │Interceptor 1│ → │Interceptor 2│  ...                      │    │
│  │  └─────────────┘   └─────────────┘                           │    │
│  │   (auth check)      (rate limit)                              │    │
│  │                                                               │    │
│  │  Có access vào Spring context — Handler, ModelAndView        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌──────────────────────────────────────┐                            │
│  │         @RestController              │                            │
│  │    @GetMapping / @PostMapping ...    │                            │
│  └──────────────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────┘
```

### So Sánh Filter vs Interceptor

| Tiêu Chí | Filter (Servlet Filter) | Interceptor (Spring MVC) |
|----------|------------------------|--------------------------|
| **Tầng hoạt động** | Tầng Servlet (trước Spring) | Tầng Spring MVC |
| **Tiêu chuẩn** | Jakarta EE (Servlet API) | Spring Framework riêng |
| **Access Spring Context** | ❌ Hạn chế | ✅ Đầy đủ |
| **Access Handler Info** | ❌ Không biết controller | ✅ Biết method, annotations |
| **Phạm vi áp dụng** | URL pattern (/*) | Cụ thể per-controller |
| **Exception Handling** | Tự xử lý | @ControllerAdvice bắt được |
| **Thứ tự** | Chạy trước | Chạy sau Filter |
| **Dùng cho** | CORS, logging, compression, security | Auth check, logging, rate limit |

---

## 2. Filter — OncePerRequestFilter

`OncePerRequestFilter` đảm bảo filter chỉ chạy **đúng một lần** mỗi request, ngay cả khi có forward/include.

### Ví Dụ 1: Request Logging Filter

```java
@Component
@Order(1)  // Thứ tự thực thi — số nhỏ hơn chạy trước
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain  // FilterChain — chuỗi filter còn lại
    ) throws ServletException, IOException {

        String requestId = UUID.randomUUID().toString().substring(0, 8);
        long startTime = System.currentTimeMillis();

        // Gắn requestId vào MDC (Mapped Diagnostic Context) để log tương quan
        MDC.put("requestId", requestId);
        response.addHeader("X-Request-Id", requestId);

        try {
            log.info("→ {} {} | IP: {} | User-Agent: {}",
                request.getMethod(),
                request.getRequestURI(),
                getClientIp(request),
                request.getHeader("User-Agent")
            );

            filterChain.doFilter(request, response);  // Chuyển sang filter/handler tiếp theo

        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("← {} {} | Status: {} | Duration: {}ms",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration
            );
            MDC.clear();  // Luôn clear MDC để tránh memory leak
        }
    }

    private String getClientIp(HttpServletRequest request) {
        // Xử lý khi đứng sau proxy/load balancer
        String forwardedFor = request.getHeader("X-Forwarded-For");
        if (forwardedFor != null && !forwardedFor.isBlank()) {
            return forwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }

    // Bỏ qua một số paths không cần log
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        return path.startsWith("/actuator") || path.startsWith("/swagger");
    }
}
```

### Ví Dụ 2: JWT Authentication Filter

```java
@Component
@Order(2)
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    private static final String BEARER_PREFIX = "Bearer ";

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.isValid(token)) {
            try {
                String username = jwtTokenProvider.extractUsername(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // Gắn Authentication vào SecurityContext
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (JwtException e) {
                log.warn("Invalid JWT token: {}", e.getMessage());
                // Không throw — để SecurityConfig xử lý (trả 401)
            }
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith(BEARER_PREFIX)) {
            return header.substring(BEARER_PREFIX.length());
        }
        return null;
    }
}
```

### Ví Dụ 3: CORS Filter Tùy Chỉnh

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)  // CORS phải chạy đầu tiên
public class CorsFilter extends OncePerRequestFilter {

    @Value("${app.cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        String origin = request.getHeader("Origin");

        if (origin != null && allowedOrigins.contains(origin)) {
            response.setHeader("Access-Control-Allow-Origin", origin);
            response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS");
            response.setHeader("Access-Control-Allow-Headers", "Authorization, Content-Type, X-Tenant-Id");
            response.setHeader("Access-Control-Max-Age", "3600");
            response.setHeader("Access-Control-Allow-Credentials", "true");
        }

        // Preflight request (OPTIONS) — trả 200 ngay, không cần tiếp tục
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 3. HandlerInterceptor — Bộ Chặn Spring MVC

`HandlerInterceptor` có 3 hook:

```
Request → preHandle() → Controller Method → postHandle() → afterCompletion()
              │                                  │                │
           Nếu false →                     Trước khi       Luôn chạy
           dừng chain                    render response    (kể cả lỗi)
```

### Interface HandlerInterceptor

```java
public interface HandlerInterceptor {

    // Chạy TRƯỚC khi controller method được gọi
    // Trả false → dừng xử lý request
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) throws Exception {
        return true;
    }

    // Chạy SAU controller method, TRƯỚC khi render response
    // handler: controller method vừa thực thi
    // modelAndView: có thể null với REST API
    default void postHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler, @Nullable ModelAndView modelAndView) throws Exception {}

    // Chạy sau khi response hoàn tất — LUÔN LUÔN chạy, kể cả khi có exception
    // Dùng để cleanup resources
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, @Nullable Exception ex) throws Exception {}
}
```

### Ví Dụ 1: Rate Limiting Interceptor — Giới Hạn Tốc Độ

```java
@Component
@RequiredArgsConstructor
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RedisTemplate<String, String> redisTemplate;

    private static final int MAX_REQUESTS_PER_MINUTE = 60;

    @Override
    public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception {

        String clientIp = request.getRemoteAddr();
        String key = "rate_limit:" + clientIp;

        // Tăng counter và đặt TTL (Time to Live — Thời Gian Sống) 60 giây
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, Duration.ofMinutes(1));
        }

        if (count > MAX_REQUESTS_PER_MINUTE) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {
                  "status": 429,
                  "title": "Too Many Requests",
                  "detail": "Vượt quá %d requests/phút. Vui lòng thử lại sau."
                }
                """.formatted(MAX_REQUESTS_PER_MINUTE));

            // Thêm Retry-After header
            response.addHeader("Retry-After", "60");
            return false; // Dừng xử lý
        }

        // Thêm header thông tin rate limit
        response.addHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS_PER_MINUTE));
        response.addHeader("X-RateLimit-Remaining", String.valueOf(MAX_REQUESTS_PER_MINUTE - count));
        return true;
    }
}
```

### Ví Dụ 2: Audit Logging Interceptor — Ghi Log Kiểm Toán

```java
@Component
@RequiredArgsConstructor
public class AuditLoggingInterceptor implements HandlerInterceptor {

    private final AuditLogRepository auditLogRepository;

    private static final ThreadLocal<Long> startTimeHolder = new ThreadLocal<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        startTimeHolder.set(System.currentTimeMillis());

        // Lấy thông tin handler (controller method)
        if (handler instanceof HandlerMethod handlerMethod) {
            Audit auditAnnotation = handlerMethod.getMethodAnnotation(Audit.class);
            if (auditAnnotation != null) {
                request.setAttribute("auditAction", auditAnnotation.action());
            }
        }
        return true;
    }

    @Override
    public void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        @Nullable Exception ex
    ) {
        Long startTime = startTimeHolder.get();
        startTimeHolder.remove(); // Tránh memory leak với ThreadLocal

        String auditAction = (String) request.getAttribute("auditAction");
        if (auditAction == null) return; // Chỉ log các action có @Audit annotation

        String username = Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName)
            .orElse("anonymous");

        AuditLog log = AuditLog.builder()
            .action(auditAction)
            .username(username)
            .method(request.getMethod())
            .path(request.getRequestURI())
            .statusCode(response.getStatus())
            .durationMs(startTime != null ? System.currentTimeMillis() - startTime : -1)
            .success(ex == null && response.getStatus() < 400)
            .errorMessage(ex != null ? ex.getMessage() : null)
            .timestamp(Instant.now())
            .build();

        auditLogRepository.save(log);
    }
}

// Custom annotation đánh dấu action cần audit
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audit {
    String action();
}

// Dùng trong controller
@DeleteMapping("/{id}")
@Audit(action = "DELETE_USER")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}
```

---

## 4. Đăng Ký Interceptor — WebMvcConfigurer

```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final RateLimitInterceptor rateLimitInterceptor;
    private final AuditLoggingInterceptor auditLoggingInterceptor;
    private final LocaleInterceptor localeInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        // Rate limit — áp dụng cho tất cả API endpoints
        registry.addInterceptor(rateLimitInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/v1/health", "/api/v1/public/**");

        // Audit logging — áp dụng rộng nhưng loại trừ read-only operations
        registry.addInterceptor(auditLoggingInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/**/list", "/api/**/search");

        // Locale — áp dụng toàn bộ
        registry.addInterceptor(localeInterceptor);
    }

    // Cấu hình CORS qua Spring MVC (thay thế cho CorsFilter)
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOriginPatterns("https://*.example.com", "http://localhost:[3000-3999]")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

---

## 5. Thứ Tự Thực Thi Đầy Đủ

```
HTTP Request
     │
     ▼
Filter 1 (doFilter - before)     ← Order(1) — RequestLoggingFilter
     │
     ▼
Filter 2 (doFilter - before)     ← Order(2) — JwtAuthenticationFilter
     │
     ▼
DispatcherServlet
     │
     ▼
Interceptor 1 preHandle()        ← RateLimitInterceptor
     │
     ▼
Interceptor 2 preHandle()        ← AuditLoggingInterceptor
     │
     ▼
@RestController method()         ← Business logic thực thi
     │
     ▼
Interceptor 2 postHandle()       ← Ngược lại với preHandle
     │
     ▼
Interceptor 1 postHandle()
     │
     ▼
Response rendering (JSON)
     │
     ▼
Interceptor 2 afterCompletion()  ← Ngược lại, luôn chạy
     │
     ▼
Interceptor 1 afterCompletion()
     │
     ▼
Filter 2 (doFilter - after)      ← Sau filterChain.doFilter()
     │
     ▼
Filter 1 (doFilter - after)
     │
     ▼
HTTP Response
```

---

## 6. Khi Nào Dùng Filter vs Interceptor

| Tình Huống | Dùng | Lý Do |
|------------|------|-------|
| Logging request/response body | Filter | Cần wrap HttpServletRequest để đọc body nhiều lần |
| CORS headers | Filter | Phải chạy trước Spring MVC, kể cả preflight OPTIONS |
| JWT Authentication | Filter | Gắn SecurityContext trước Spring Security processing |
| Compression (GZIP) | Filter | Xử lý ở tầng Servlet, độc lập với Spring |
| Rate limiting | Interceptor | Cần Spring context (Redis Bean, UserDetails...) |
| Authorization (role check) | Interceptor | Cần biết controller/method đang được gọi |
| Audit logging | Interceptor | Cần `afterCompletion` để biết kết quả, có exception không |
| Request tracing | Filter | Phải gắn trace ID trước tất cả processing |
| Cache response | Interceptor | Cần biết handler method, có annotation hay không |
| Locale/Timezone | Interceptor | Cần Spring context để resolve message |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Filter vs Interceptor — khác nhau chính là gì?**

> **Filter** chạy ở tầng Servlet (ngoài Spring MVC) — không có access vào Spring Application Context, không biết controller nào sẽ xử lý. **Interceptor** chạy trong Spring MVC — có đầy đủ access vào Spring Beans, biết Handler method, annotation. Filter dùng cho cross-cutting concerns ở tầng thấp (CORS, auth, compression). Interceptor dùng cho logic liên quan đến Spring như audit, rate limit, locale.

**Q: Tại sao dùng `OncePerRequestFilter` thay vì implement `Filter` trực tiếp?**

> `Filter` thông thường có thể được gọi nhiều lần trong một request khi có forward/include (ví dụ: error handling forward). `OncePerRequestFilter` đảm bảo `doFilterInternal()` chỉ chạy đúng một lần, tránh duplicate processing (logging hai lần, auth check hai lần...).

**Q: `preHandle` trả về `false` thì điều gì xảy ra?**

> Khi `preHandle` trả `false`, Spring MVC dừng xử lý request và không gọi các interceptors tiếp theo cũng như controller. Response đã được set trong `preHandle` (status code, body) sẽ được gửi về client. `postHandle` và `afterCompletion` của interceptor đó và các interceptor sau đó **không được gọi**, nhưng `afterCompletion` của các interceptor trước đó **vẫn được gọi** để cleanup.

---

## ✅ Checklist

- [ ] Hiểu thứ tự: Filter → Interceptor → Controller
- [ ] `OncePerRequestFilter` với doFilterInternal() và shouldNotFilter()
- [ ] `HandlerInterceptor` với preHandle, postHandle, afterCompletion
- [ ] Đăng ký interceptor qua WebMvcConfigurer với path patterns
- [ ] Biết khi nào dùng Filter vs Interceptor
- [ ] Xử lý ThreadLocal cleanup trong afterCompletion

---

**Cập Nhật Lần Cuối:** 2026-06-02  
**Tiếp Theo:** [5-openapi-swagger.md](5-openapi-swagger.md) — OpenAPI & Swagger UI
