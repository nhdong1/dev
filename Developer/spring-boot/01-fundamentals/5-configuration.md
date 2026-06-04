# Cấu Hình Spring Boot — @Value, @ConfigurationProperties, Profiles

> Quản lý cấu hình tốt là yếu tố quyết định ứng dụng có thể chạy trơn tru trên nhiều môi trường
> (dev, staging, production) mà không cần thay đổi code.

---

## 📋 Mục Tiêu

- [ ] Đọc giá trị cấu hình bằng `@Value`
- [ ] Nhóm cấu hình thành class với `@ConfigurationProperties`
- [ ] Dùng **Profiles** (Môi Trường) để cấu hình theo từng môi trường
- [ ] Biết thứ tự ưu tiên của các nguồn cấu hình
- [ ] Validate cấu hình bằng **Bean Validation** khi khởi động
- [ ] Quản lý secrets (bí mật) an toàn — không hardcode

---

## 1. application.properties vs application.yml

Spring Boot hỗ trợ hai format cấu hình:

```properties
# application.properties — format key=value
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=postgres
spring.datasource.password=secret
app.jwt.secret=my-secret-key
app.jwt.expiration=3600
```

```yaml
# application.yml — format YAML, dễ đọc hơn khi có cấu hình lồng nhau
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret

app:
  jwt:
    secret: my-secret-key
    expiration: 3600
```

**Khuyến nghị:** Dùng YAML cho project mới — dễ đọc hơn khi cấu hình phức tạp.

---

## 2. @Value — Inject Giá Trị Đơn Lẻ

`@Value` dùng để inject một giá trị cấu hình đơn lẻ vào field hoặc constructor parameter.

### Cú Pháp Cơ Bản

```java
@Service
public class JwtService {

    // Inject từ application.yml: app.jwt.secret
    @Value("${app.jwt.secret}")
    private String jwtSecret;

    // Inject với giá trị mặc định nếu không tìm thấy property
    @Value("${app.jwt.expiration:3600}")
    private int jwtExpiration; // Mặc định 3600 giây nếu không cấu hình

    // Inject giá trị cố định (ít dùng)
    @Value("Bearer")
    private String tokenPrefix;

    // Inject danh sách — tự động parse từ "admin,user,guest"
    @Value("${app.allowed.roles:admin,user}")
    private List<String> allowedRoles;
}
```

### @Value Với SpEL (Spring Expression Language — Ngôn Ngữ Biểu Thức Spring)

```java
@Service
public class FeatureService {

    // Tính toán từ property
    @Value("${app.cache.ttl:300000}")       // milliseconds
    private long cacheTtlMs;

    @Value("#{${app.cache.ttl:300000} / 1000}")  // chuyển sang giây
    private long cacheTtlSeconds;

    // Inject System property
    @Value("#{systemProperties['user.home']}")
    private String userHome;

    // Inject từ Environment variable (Biến Môi Trường)
    @Value("${DATABASE_URL:#{null}}")
    private String databaseUrl; // null nếu không có

    // Điều kiện đơn giản
    @Value("#{${app.feature.enabled:false} ? 'ENABLED' : 'DISABLED'}")
    private String featureStatus;
}
```

### Hạn Chế Của @Value

```java
// ❌ @Value KHÔNG hoạt động trong constructor của @Configuration class
// khi class dùng @Bean method
@Configuration
public class AppConfig {
    @Value("${app.name}")  // ❌ Có thể bị null do thứ tự khởi tạo
    private String appName;
}

// ✅ Dùng @ConfigurationProperties thay thế cho cấu hình phức tạp
```

---

## 3. @ConfigurationProperties — Nhóm Cấu Hình Thành Class

`@ConfigurationProperties` cho phép map toàn bộ một nhóm properties vào một Java class — **được khuyến nghị hơn @Value** cho cấu hình phức tạp.

### Cơ Bản

```yaml
# application.yml
app:
  jwt:
    secret: my-super-secret-key-256-bits
    expiration: 3600        # giây
    refresh-expiration: 86400  # giây — 24 giờ
    issuer: my-api

  mail:
    host: smtp.gmail.com
    port: 587
    username: noreply@myapp.com
    password: mail-password
    from: "My App <noreply@myapp.com>"
```

```java
// JwtProperties.java
@ConfigurationProperties(prefix = "app.jwt")
public record JwtProperties(
    String secret,
    int expiration,
    int refreshExpiration,
    String issuer
) {}
// Spring tự động map:
// app.jwt.secret → secret
// app.jwt.expiration → expiration
// app.jwt.refresh-expiration → refreshExpiration (tự convert kebab-case → camelCase)

// MailProperties.java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    String host,
    int port,
    String username,
    String password,
    String from
) {}
```

### Đăng Ký @ConfigurationProperties

```java
// Cách 1: @EnableConfigurationProperties trong @Configuration class
@Configuration
@EnableConfigurationProperties({JwtProperties.class, MailProperties.class})
public class AppConfig {}

// Cách 2: @ConfigurationPropertiesScan ở main class (scan toàn bộ package)
@SpringBootApplication
@ConfigurationPropertiesScan
public class MyApplication {}

// Cách 3: Thêm @Component vào class (đơn giản nhất)
@ConfigurationProperties(prefix = "app.jwt")
@Component
public record JwtProperties(String secret, int expiration) {}
```

### Sử Dụng Trong Service

```java
@Service
public class JwtService {

    private final JwtProperties jwtProperties;

    public JwtService(JwtProperties jwtProperties) {
        this.jwtProperties = jwtProperties;
    }

    public String generateToken(String username) {
        return Jwts.builder()
            .subject(username)
            .issuer(jwtProperties.issuer())
            .expiration(Date.from(Instant.now().plusSeconds(jwtProperties.expiration())))
            .signWith(Keys.hmacShaKeyFor(jwtProperties.secret().getBytes()))
            .compact();
    }
}
```

### Validation Cấu Hình — Fail-fast

```java
@ConfigurationProperties(prefix = "app.jwt")
@Validated // Kích hoạt Bean Validation
public record JwtProperties(
    @NotBlank(message = "JWT secret không được để trống")
    @Size(min = 32, message = "JWT secret phải có ít nhất 32 ký tự")
    String secret,

    @Min(value = 60, message = "JWT expiration tối thiểu 60 giây")
    @Max(value = 86400, message = "JWT expiration tối đa 1 ngày")
    int expiration,

    @NotBlank
    String issuer
) {}
// Nếu cấu hình không hợp lệ → ứng dụng từ chối khởi động với thông báo rõ ràng
// Thay vì lỗi bí ẩn lúc runtime!
```

### Cấu Hình Dạng List và Map

```yaml
# application.yml
app:
  cors:
    allowed-origins:
      - https://myapp.com
      - https://admin.myapp.com
      - http://localhost:3000
    allowed-methods:
      - GET
      - POST
      - PUT
      - DELETE

  rate-limit:
    rules:
      /api/auth/login: 5       # max 5 request/phút cho login
      /api/auth/register: 3    # max 3 request/phút cho register
      /api/: 100               # max 100 request/phút cho API chung
```

```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(
    CorsProperties cors,
    RateLimitProperties rateLimit
) {
    public record CorsProperties(
        List<String> allowedOrigins,
        List<String> allowedMethods
    ) {}

    public record RateLimitProperties(
        Map<String, Integer> rules
    ) {}
}
```

---

## 4. Profiles — Cấu Hình Theo Môi Trường

**Profiles** cho phép có cấu hình khác nhau cho từng môi trường (development, staging, production).

### Cấu Trúc File Cấu Hình Theo Profile

```
src/main/resources/
├── application.yml                 ← Cấu hình chung (áp dụng cho mọi môi trường)
├── application-local.yml           ← Môi trường local development
├── application-dev.yml             ← Môi trường development server
├── application-staging.yml         ← Môi trường staging
└── application-prod.yml            ← Môi trường production
```

```yaml
# application.yml — cấu hình base chung
spring:
  application:
    name: my-api
  jpa:
    open-in-view: false

server:
  port: 8080

app:
  name: My Application
```

```yaml
# application-local.yml — môi trường local
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb_local
    username: postgres
    password: postgres
  jpa:
    show-sql: true              # Hiện SQL query — hữu ích khi dev
    hibernate:
      ddl-auto: create-drop     # Tạo và xóa schema mỗi lần chạy

logging:
  level:
    com.example.myapp: DEBUG    # Log chi tiết khi dev
    org.springframework.web: DEBUG
```

```yaml
# application-prod.yml — môi trường production
spring:
  datasource:
    url: ${DATABASE_URL}        # Đọc từ Environment Variable
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
  jpa:
    show-sql: false             # Không log SQL trên production
    hibernate:
      ddl-auto: validate        # Chỉ validate schema, không tự động sửa

logging:
  level:
    root: WARN                  # Chỉ log warning và error trên production
    com.example.myapp: INFO
```

### Kích Hoạt Profile

```bash
# Khi chạy JAR
java -jar myapp.jar --spring.profiles.active=prod

# Biến môi trường
export SPRING_PROFILES_ACTIVE=prod
java -jar myapp.jar

# Docker
docker run -e SPRING_PROFILES_ACTIVE=prod myapp:latest
```

```yaml
# application.yml — set profile mặc định cho development
spring:
  profiles:
    default: local  # Profile mặc định nếu không chỉ định
```

### @Profile — Bean Chỉ Tạo Theo Profile

```java
// Chỉ tạo Bean này trong môi trường local
@Configuration
@Profile("local")
public class MockDataConfig {

    @Bean
    public DataInitializer mockDataInitializer(UserRepository repo) {
        return new MockDataInitializer(repo);
    }
}

// Chỉ tạo Bean này trong môi trường production
@Configuration
@Profile("prod")
public class ProductionConfig {

    @Bean
    public AuditLogger auditLogger() {
        return new CloudWatchAuditLogger();
    }
}

// Dùng cho Service — multiple profiles
@Service
@Profile({"dev", "local"})
public class MockEmailService implements EmailService {
    @Override
    public void send(String to, String subject, String body) {
        log.info("Mock email đến {}: {}", to, subject); // Không gửi thực
    }
}

@Service
@Profile("prod")
public class SmtpEmailService implements EmailService {
    @Override
    public void send(String to, String subject, String body) {
        // Gửi email thực qua SMTP
    }
}
```

### Profile Groups (Spring Boot 2.4+) — Nhóm Profiles

```yaml
# application.yml
spring:
  profiles:
    group:
      local: local-db, local-mail, local-kafka  # profile "local" kích hoạt 3 sub-profiles
      prod: prod-db, prod-mail, prod-kafka
```

---

## 5. Thứ Tự Ưu Tiên Nguồn Cấu Hình

Spring Boot đọc cấu hình từ nhiều nguồn, **nguồn sau ghi đè nguồn trước** (số nhỏ = ưu tiên thấp, số lớn = ưu tiên cao):

```
1.  Default properties (SpringApplication.setDefaultProperties)
2.  @PropertySource trong @Configuration class
3.  application.properties / application.yml
4.  application-{profile}.properties / application-{profile}.yml
5.  OS environment variables (SPRING_DATASOURCE_URL)
6.  Java System properties (-Dspring.datasource.url=...)
7.  Command line arguments (--spring.datasource.url=...)
```

```
Ví dụ:
  application.yml:         server.port=8080
  application-prod.yml:    server.port=443
  OS env:                  SERVER_PORT=9090
  Command line:            --server.port=8443

  Kết quả: 8443 (command line thắng)
```

**Quy tắc quan trọng:** Environment variables và command line arguments luôn ghi đè file cấu hình — rất hữu ích cho container deployments.

### Relaxed Binding — Đặt Tên Linh Hoạt

Spring Boot tự động chuyển đổi các format tên:

```yaml
# Tất cả đều map vào field: maximumPoolSize
spring.datasource.hikari.maximum-pool-size=50    # kebab-case (khuyến nghị)
spring.datasource.hikari.maximumPoolSize=50      # camelCase
spring.datasource.hikari.maximum_pool_size=50    # snake_case
SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=50    # SCREAMING_SNAKE_CASE (env var)
```

---

## 6. Quản Lý Secrets — Bí Mật An Toàn

### ❌ KHÔNG Bao Giờ Làm

```yaml
# application-prod.yml — ĐỪNG COMMIT FILE NÀY VÀO GIT!
spring:
  datasource:
    password: my-super-secret-db-password   # ❌ Hardcode password trong file

app:
  jwt:
    secret: jwt-secret-key-should-not-be-here  # ❌ Secret trong source code
```

### ✅ Cách Đúng — Environment Variables

```yaml
# application-prod.yml — AN TOÀN — có thể commit
spring:
  datasource:
    url: ${DATABASE_URL}                # Đọc từ env var
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}            # Không có giá trị default

app:
  jwt:
    secret: ${JWT_SECRET}               # Phải set trong môi trường chạy
    expiration: ${JWT_EXPIRATION:3600}  # Có default nếu không set
```

```bash
# Thiết lập trong môi trường production
export DATABASE_URL="jdbc:postgresql://prod-db:5432/mydb"
export DB_USERNAME="app_user"
export DB_PASSWORD="$(vault read -field=password secret/db)"
export JWT_SECRET="$(vault read -field=key secret/jwt)"
```

### Với Docker / Kubernetes

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=jdbc:postgresql://db:5432/mydb
      - DB_USERNAME=app_user
      - DB_PASSWORD_FILE=/run/secrets/db_password  # Docker Secret
```

```yaml
# Kubernetes Secret + ConfigMap
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DB_PASSWORD: "supersecret"
  JWT_SECRET: "jwt-key-here"

---
# Trong Deployment
envFrom:
  - secretRef:
      name: app-secrets
  - configMapRef:
      name: app-config
```

---

## 7. @PropertySource — Nạp File Properties Tùy Chỉnh

```java
@Configuration
@PropertySource("classpath:aws.properties")
@PropertySource(value = "file:/etc/myapp/override.properties", ignoreResourceNotFound = true)
public class AwsConfig {

    @Value("${aws.region}")
    private String awsRegion;

    @Value("${aws.s3.bucket}")
    private String s3Bucket;
}
```

```properties
# src/main/resources/aws.properties
aws.region=ap-southeast-1
aws.s3.bucket=my-app-storage
aws.cloudfront.domain=cdn.myapp.com
```

---

## 8. Actuator — Xem Cấu Hình Đang Chạy

```yaml
management:
  endpoints:
    web:
      exposure:
        include: env, configprops
  endpoint:
    env:
      show-values: WHEN_AUTHORIZED  # Che giấu secrets khi hiển thị
```

```bash
# Xem tất cả properties đang áp dụng
GET /actuator/env

# Xem giá trị một property cụ thể
GET /actuator/env/server.port

# Xem tất cả @ConfigurationProperties
GET /actuator/configprops
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Q: @Value vs @ConfigurationProperties — khi nào dùng cái nào?

**Trả lời:**

| Tiêu Chí | @Value | @ConfigurationProperties |
|----------|--------|--------------------------|
| Số lượng properties | 1–2 properties đơn lẻ | Nhiều properties liên quan |
| Validation | Khó | Dễ — kết hợp @Validated |
| IDE support | Giới hạn | Đầy đủ (với annotation processor) |
| Test | Cần @TestPropertySource | Dễ — tạo instance trực tiếp |
| Tái sử dụng | Không | Có thể inject vào nhiều nơi |
| Relaxed binding | ❌ | ✅ |

**Quy tắc:** Nếu có hơn 2 properties liên quan → dùng `@ConfigurationProperties`.

### Q: Profiles hoạt động thế nào khi deploy?

**Trả lời:** Spring Boot đọc `spring.profiles.active` từ nhiều nguồn theo thứ tự ưu tiên. Trong production:
1. Set environment variable `SPRING_PROFILES_ACTIVE=prod`
2. Hoặc command line `--spring.profiles.active=prod`
3. File `application-prod.yml` sẽ được load và **ghi đè** lên `application.yml`
4. Sensitive values như passwords không ở trong file yml mà ở environment variables hoặc secret manager

### Q: Nếu có cùng property trong application.yml và command line, giá trị nào thắng?

**Trả lời:** Command line argument (`--server.port=9090`) luôn có **ưu tiên cao nhất** trong số các nguồn thông thường. Chỉ có một số ít nguồn đặc biệt có ưu tiên cao hơn (như `@TestPropertySource` trong test). Đây là lý do tại sao có thể override bất kỳ cấu hình nào khi deploy mà không cần sửa file.

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Tạo `@ConfigurationProperties` class với validation cho JWT config
- [ ] Thiết lập 3 profiles: local, dev, prod — mỗi profile có DataSource riêng
- [ ] Dùng environment variables cho passwords thay vì hardcode
- [ ] Giải thích thứ tự ưu tiên khi có cùng property ở nhiều nơi
- [ ] Dùng Actuator `/actuator/env` để kiểm tra cấu hình đang áp dụng

---

## 🔗 Tóm Tắt Module 01-fundamentals

Bạn đã hoàn thành toàn bộ nền tảng:

| File | Nội Dung | Trạng Thái |
|------|----------|-----------|
| [1-java-core.md](1-java-core.md) | Records, Sealed Classes, Streams, Optional, Virtual Threads | ✅ |
| [2-spring-core.md](2-spring-core.md) | IoC, DI, ApplicationContext, Component Scanning, AOP | ✅ |
| [3-spring-boot-basics.md](3-spring-boot-basics.md) | Auto-configuration, Starters, SpringApplication | ✅ |
| [4-bean-lifecycle.md](4-bean-lifecycle.md) | Lifecycle phases, @PostConstruct, @PreDestroy, Scopes | ✅ |
| [5-configuration.md](5-configuration.md) | @Value, @ConfigurationProperties, Profiles, Secrets | ✅ |

**Tiếp Theo:** [../02-web-layer/](../02-web-layer/) — Xây Dựng REST API

---

**Cập Nhật Lần Cuối:** 2026-06-02
