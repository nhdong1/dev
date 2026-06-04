# Security Best Practices — Bảo Mật Production

> Tổng hợp các best practices (thực hành tốt nhất) về bảo mật khi triển khai Spring Boot
> trong môi trường production: mã hóa mật khẩu, quản lý secret, HTTP security headers,
> và checklist hardening (tăng cường bảo mật).

---

## 1. Password Security (Bảo Mật Mật Khẩu)

### 1.1 BCrypt — Thuật Toán Mã Hóa Được Khuyến Nghị

BCrypt là thuật toán hash mật khẩu được thiết kế chậm có chủ đích (slow by design), khiến brute-force tốn kém hơn:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // Cost factor = 12
}
```

**Đặc điểm BCrypt:**
- **Adaptive** (Thích Ứng) — có thể tăng cost factor theo thời gian khi phần cứng mạnh lên
- **Salt tự động** — mỗi lần hash tạo ra salt ngẫu nhiên, chống rainbow table attack
- **One-way** — không giải mã ngược được

**Chọn cost factor:**
```
Cost 10 → ~100ms   → Minimum cho production
Cost 12 → ~300ms   → Khuyến nghị (cân bằng bảo mật/hiệu năng)
Cost 14 → ~1000ms  → Cho hệ thống yêu cầu bảo mật cao
Cost 16 → ~4000ms  → Rất cao (cân nhắc UX)
```

### 1.2 Argon2 — Thuật Toán Hiện Đại Hơn

Argon2 là người chiến thắng [Password Hashing Competition 2015](https://www.password-hashing.net/), được coi là tiêu chuẩn mới:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Argon2id — variant được khuyến nghị (kết hợp Argon2i và Argon2d)
    return new Argon2PasswordEncoder(
        16,    // salt length (bytes)
        32,    // hash length (bytes)
        1,     // parallelism
        65536, // memory (KB) — 64MB
        3      // iterations
    );
}
```

### 1.3 DelegatingPasswordEncoder — Hỗ Trợ Migration

Cho phép migration dần từ thuật toán cũ sang mới mà không reset mật khẩu tất cả user:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Default encoder là bcrypt, hỗ trợ đọc các format cũ
    PasswordEncoder encoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
    return encoder;
}
// Passwords được lưu: {bcrypt}$2a$10$..., {noop}plaintext, {sha256}...
```

### 1.4 Quy Tắc Mật Khẩu (Password Policy)

```java
@Component
public class PasswordPolicyValidator {

    private static final int MIN_LENGTH = 8;
    private static final Pattern HAS_UPPERCASE = Pattern.compile("[A-Z]");
    private static final Pattern HAS_DIGIT = Pattern.compile("[0-9]");
    private static final Pattern HAS_SPECIAL = Pattern.compile("[!@#$%^&*(),.?\":{}|<>]");

    public void validate(String password) {
        List<String> violations = new ArrayList<>();

        if (password.length() < MIN_LENGTH)
            violations.add("Mật khẩu phải có ít nhất " + MIN_LENGTH + " ký tự");
        if (!HAS_UPPERCASE.matcher(password).find())
            violations.add("Mật khẩu phải có ít nhất 1 chữ hoa");
        if (!HAS_DIGIT.matcher(password).find())
            violations.add("Mật khẩu phải có ít nhất 1 chữ số");
        if (!HAS_SPECIAL.matcher(password).find())
            violations.add("Mật khẩu phải có ít nhất 1 ký tự đặc biệt");

        if (!violations.isEmpty()) {
            throw new PasswordPolicyViolationException(violations);
        }
    }
}
```

---

## 2. Secrets Management (Quản Lý Bí Mật)

Secrets bao gồm: JWT secret key, database password, API keys, encryption keys.

### 2.1 ❌ Sai — Hardcode Trong Code

```java
// TUYỆT ĐỐI KHÔNG LÀM
private static final String JWT_SECRET = "my-super-secret-key";
private static final String DB_PASSWORD = "password123";
```

### 2.2 ✅ Đúng — Environment Variables (Biến Môi Trường)

```yaml
# application.yml — tham chiếu đến biến môi trường
app:
  jwt:
    secret: ${JWT_SECRET}   # Đọc từ env var JWT_SECRET
spring:
  datasource:
    password: ${DB_PASSWORD}
```

```bash
# Đặt biến môi trường khi khởi động
export JWT_SECRET="$(openssl rand -hex 32)"
export DB_PASSWORD="your-db-password"
java -jar app.jar
```

### 2.3 ✅ Đúng — Spring Cloud Vault (Cho Production)

HashiCorp Vault là giải pháp enterprise-grade để quản lý secrets:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  cloud:
    vault:
      host: vault.internal.company.com
      port: 8200
      scheme: https
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        default-context: spring-boot-app
```

### 2.4 ✅ Đúng — AWS Secrets Manager / Azure Key Vault / GCP Secret Manager

```xml
<!-- AWS Secrets Manager -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-secrets-manager-config</artifactId>
</dependency>
```

```yaml
spring:
  config:
    import: "aws-secretsmanager:/myapp/prod/secrets"
```

### 2.5 Tạo JWT Secret Key An Toàn

```bash
# Tạo 256-bit secret key (32 bytes = 256 bits)
openssl rand -hex 32

# Hoặc dùng Java
import java.security.SecureRandom;
byte[] key = new byte[32];
new SecureRandom().nextBytes(key);
String hexKey = HexFormat.of().formatHex(key);
```

---

## 3. HTTP Security Headers (HTTP Header Bảo Mật)

Spring Security tự động thêm nhiều security headers, nhưng cần cấu hình thêm cho production:

### 3.1 Cấu Hình Headers

```java
http.headers(headers -> headers
    // X-Content-Type-Options — ngăn MIME type sniffing
    .contentTypeOptions(ContentTypeOptionsConfig::disable) // Hoặc không cần vì mặc định đã bật

    // X-Frame-Options — ngăn clickjacking (tấn công click giả mạo)
    .frameOptions(frame -> frame.sameOrigin())  // Chỉ cho phép frame từ cùng origin

    // Content-Security-Policy — CSP — kiểm soát nguồn tài nguyên được tải
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
    )

    // HTTP Strict Transport Security — HSTS — buộc dùng HTTPS
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000)  // 1 năm
        .includeSubDomains(true)
        .preload(true)
    )

    // Referrer Policy — kiểm soát thông tin referrer
    .referrerPolicy(referrer ->
        referrer.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
    )

    // Permissions Policy (thay thế Feature-Policy) — tắt browser features không cần
    .permissionsPolicy(permissions ->
        permissions.policy("camera=(), microphone=(), geolocation=()")
    )
);
```

### 3.2 Headers Quan Trọng

| Header | Mục Đích | Giá Trị Khuyến Nghị |
|--------|---------|---------------------|
| `X-Content-Type-Options` | Ngăn MIME sniffing | `nosniff` |
| `X-Frame-Options` | Ngăn clickjacking | `SAMEORIGIN` hoặc `DENY` |
| `Strict-Transport-Security` | Buộc HTTPS | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | Kiểm soát resource loading | Tùy theo app |
| `X-XSS-Protection` | XSS filter (deprecated, thay bằng CSP) | `0` (tắt, dùng CSP thay) |
| `Referrer-Policy` | Kiểm soát referrer header | `strict-origin-when-cross-origin` |

---

## 4. Rate Limiting (Giới Hạn Tốc Độ)

Ngăn brute force attack và API abuse:

### 4.1 Bucket4j — Rate Limiting Trong Spring Boot

```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-redis</artifactId>
    <version>8.10.1</version>
</dependency>
```

```java
@Component
@RequiredArgsConstructor
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Cache<String, Bucket> cache = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.HOURS)
        .maximumSize(10_000)
        .build();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // Rate limit theo IP cho endpoint login
        if (request.getServletPath().startsWith("/auth/login")) {
            String clientIp = getClientIp(request);
            Bucket bucket = cache.get(clientIp, k -> createBucket());

            if (!bucket.tryConsume(1)) {
                response.setStatus(429); // Too Many Requests
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.getWriter().write("""
                    {"error": "Quá nhiều lần đăng nhập. Vui lòng thử lại sau."}
                    """);
                return;
            }
        }

        filterChain.doFilter(request, response);
    }

    private Bucket createBucket() {
        // Cho phép 5 request/phút cho login endpoint
        return Bucket.builder()
            .addLimit(Bandwidth.classic(5, Refill.greedy(5, Duration.ofMinutes(1))))
            .build();
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## 5. Input Validation & SQL Injection Prevention (Ngăn Chặn SQL Injection)

### 5.1 Bean Validation

```java
public record CreateUserRequest(
    @NotBlank(message = "Username không được để trống")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_-]+$", message = "Username chỉ được chứa chữ cái, số, dấu gạch dưới")
    String username,

    @NotBlank
    @Size(min = 8, max = 100)
    String password,

    @Email(message = "Email không hợp lệ")
    @NotBlank
    String email
) {}
```

### 5.2 Parameterized Queries (Câu Truy Vấn Tham Số) — Ngăn SQL Injection

```java
// ❌ Nguy hiểm — SQL Injection
@Query("SELECT u FROM User u WHERE u.username = '" + username + "'")  // KHÔNG LÀM
public Optional<User> findByUsername(String username);

// ✅ An toàn — Spring Data tự động parameterize
public Optional<User> findByUsername(String username);  // Tự động dùng prepared statement

// ✅ An toàn — JPQL với named parameter
@Query("SELECT u FROM User u WHERE u.username = :username")
public Optional<User> findByUsername(@Param("username") String username);

// ✅ An toàn — Native query với named parameter
@Query(value = "SELECT * FROM users WHERE username = :username", nativeQuery = true)
public Optional<User> findByUsernameNative(@Param("username") String username);
```

---

## 6. Logging & Audit Trail (Nhật Ký & Dấu Vết Kiểm Tra)

### 6.1 Security Event Logging

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SecurityEventListener {

    private final AuditLogRepository auditLogRepository;

    // Ghi log khi đăng nhập thành công
    @EventListener
    public void onAuthenticationSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        log.info("Đăng nhập thành công: user={}", username);
        auditLogRepository.save(new AuditLog("LOGIN_SUCCESS", username));
    }

    // Ghi log khi đăng nhập thất bại
    @EventListener
    public void onAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        String reason = event.getException().getClass().getSimpleName();
        log.warn("Đăng nhập thất bại: user={}, reason={}", username, reason);
        auditLogRepository.save(new AuditLog("LOGIN_FAILURE", username, reason));
    }

    // Ghi log khi bị từ chối truy cập
    @EventListener
    public void onAuthorizationDenied(AuthorizationDeniedEvent<?> event) {
        String username = event.getAuthentication().get().getName();
        log.warn("Truy cập bị từ chối: user={}, resource={}", username,
                 event.getAuthorizationDecision());
    }
}
```

### 6.2 Những Thứ KHÔNG Được Log

```java
// ❌ KHÔNG log thông tin nhạy cảm
log.info("User login with password: {}", password);           // KHÔNG BAO GIỜ
log.debug("JWT token: {}", token);                            // Không nên
log.info("Credit card: {}", creditCard);                      // TUYỆT ĐỐI KHÔNG

// ✅ Log thông tin an toàn
log.info("User {} logged in from IP {}", username, clientIp);
log.info("Token issued for user {} with expiry {}", username, expiryTime);
```

---

## 7. HTTPS & TLS Configuration

### 7.1 Buộc HTTPS Trong Spring Boot

```yaml
# application.yml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: springboot
  port: 8443
```

### 7.2 Redirect HTTP → HTTPS

```java
@Configuration
public class HttpsConfig {

    @Bean
    public TomcatServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };

        // HTTP port — chỉ để redirect
        Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        tomcat.addAdditionalTomcatConnectors(connector);

        return tomcat;
    }
}
```

---

## 8. Security Testing Checklist (Danh Sách Kiểm Tra Bảo Mật)

### 8.1 Authentication & Authorization

- [ ] Tất cả endpoints yêu cầu xác thực (trừ public endpoints)
- [ ] Không thể bypass authorization bằng cách thay đổi URL
- [ ] JWT token được validate đầy đủ (signature, expiry, type)
- [ ] Refresh token được lưu và revoke đúng cách
- [ ] Password được hash bằng BCrypt (không MD5/SHA1)
- [ ] Rate limiting trên endpoint đăng nhập
- [ ] Account lockout sau N lần sai mật khẩu

### 8.2 Data Security

- [ ] Không có sensitive data trong JWT payload (password, credit card, SSN)
- [ ] SQL injection được ngăn chặn (prepared statements)
- [ ] XSS được ngăn chặn (output encoding)
- [ ] Sensitive data được mã hóa trong DB (PII — Personally Identifiable Information)
- [ ] Log không chứa password, token, sensitive data

### 8.3 Infrastructure

- [ ] HTTPS được bật và HTTP redirect sang HTTPS
- [ ] HSTS header được cấu hình
- [ ] Secrets không được hardcode, lưu trong Vault/env vars
- [ ] Không expose stack trace trong error response
- [ ] Security headers đầy đủ
- [ ] Dependencies không có known vulnerabilities (CVE — Common Vulnerabilities and Exposures)

### 8.4 Kiểm Tra Với OWASP ZAP hoặc Snyk

```bash
# Kiểm tra dependencies có CVE không
mvn org.owasp:dependency-check-maven:check

# Hoặc dùng Snyk
snyk test
```

---

## 9. Cấu Hình Application Properties Bảo Mật

```yaml
# application-prod.yml
spring:
  # Tắt H2 console trong production
  h2:
    console:
      enabled: false

  # Không expose sensitive info trong error
  mvc:
    problemdetails:
      enabled: true

  # Tắt DevTools trong production
  devtools:
    restart:
      enabled: false

management:
  # Chỉ expose health endpoint ra ngoài
  endpoints:
    web:
      exposure:
        include: health, info
  # Ẩn details về health check
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
```

---

## 10. Security Hardening Summary (Tóm Tắt Tăng Cường Bảo Mật)

```
Production Security Checklist
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AUTHENTICATION
  ✅ BCrypt (cost ≥ 12) cho password
  ✅ JWT secret ≥ 256 bits, lưu trong env/Vault
  ✅ Access token TTL ≤ 15 phút
  ✅ Refresh token rotation + revocation
  ✅ Rate limiting trên /auth/login (5 req/min)

AUTHORIZATION
  ✅ Deny by default — .anyRequest().authenticated()
  ✅ @PreAuthorize ở service layer
  ✅ Không expose admin endpoints công khai

TRANSPORT
  ✅ HTTPS only — redirect HTTP → HTTPS
  ✅ HSTS: max-age=31536000; includeSubDomains
  ✅ TLS 1.2+ (tắt TLS 1.0/1.1)

HTTP HEADERS
  ✅ X-Content-Type-Options: nosniff
  ✅ X-Frame-Options: SAMEORIGIN
  ✅ Content-Security-Policy
  ✅ Referrer-Policy

CORS
  ✅ Chỉ whitelist origins đã biết
  ✅ Không dùng wildcard '*' với credentials

SECRETS
  ✅ Không hardcode trong code/properties
  ✅ Dùng environment variables hoặc Vault
  ✅ Rotate secrets định kỳ

LOGGING
  ✅ Log authentication events
  ✅ Không log passwords, tokens, PII
  ✅ Centralized logging (ELK/Loki)

DEPENDENCIES
  ✅ Chạy OWASP Dependency Check trong CI/CD
  ✅ Update dependencies định kỳ
  ✅ Không dùng EOL (End-of-Life) libraries
```

---

## 11. Checklist Best Practices

- [ ] Dùng BCrypt/Argon2 với cost factor phù hợp
- [ ] Secrets được quản lý qua env vars hoặc Vault
- [ ] HTTP security headers đầy đủ
- [ ] Rate limiting trên authentication endpoints
- [ ] SQL injection được ngăn bằng parameterized queries
- [ ] Không log sensitive data
- [ ] HTTPS bắt buộc với HSTS
- [ ] OWASP Dependency Check trong CI/CD pipeline
- [ ] Security event logging đầy đủ
- [ ] Production endpoints không expose stack trace

---

## 🔗 Quay Lại

← [README.md](README.md) — Tổng Quan Module Bảo Mật

← [1-spring-security-basics.md](1-spring-security-basics.md) — Spring Security Basics

← [2-jwt-implementation.md](2-jwt-implementation.md) — JWT Implementation
