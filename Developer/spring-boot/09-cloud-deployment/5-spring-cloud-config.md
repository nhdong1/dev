# Spring Cloud Config — Cấu Hình Tập Trung

> Spring Cloud Config cung cấp Config Server — Máy Chủ Cấu Hình — tập trung cho hệ thống microservices. Thay vì mỗi service quản lý file cấu hình riêng, tất cả properties được lưu trữ tập trung (thường trên Git) và các service fetch về tại startup. Kết hợp với `@RefreshScope`, cấu hình có thể cập nhật tại runtime mà không cần restart.

---

## 1. Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    Configuration Flow                           │
│                                                                 │
│  Git Repository (GitHub/GitLab)                                 │
│  ├── application.yml          ← Default cho tất cả services     │
│  ├── order-service.yml        ← Cho order-service               │
│  ├── order-service-prod.yml   ← Cho order-service, env prod     │
│  └── user-service.yml         ← Cho user-service                │
│                │                                                │
│                ▼ (pull)                                         │
│  ┌─────────────────────┐                                        │
│  │   Config Server     │  :8888                                 │
│  │  (Spring Cloud      │◄── GET /order-service/prod             │
│  │   Config Server)    │                                        │
│  └─────────────────────┘                                        │
│                │ HTTP                                           │
│                ▼ (push config)                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Order Service│  │ User Service │  │ Auth Service │         │
│  │  (client)    │  │  (client)    │  │  (client)    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Ưu Điểm

| Ưu Điểm | Mô Tả |
|---------|-------|
| **Centralized** | Quản lý cấu hình tại một nơi duy nhất |
| **Version controlled** | Mọi thay đổi config đều có lịch sử Git |
| **Environment-specific** | Tự động chọn config theo profile (dev/staging/prod) |
| **Dynamic refresh** | Cập nhật config không cần restart service |
| **Audit trail** | Biết ai thay đổi gì, khi nào |

---

## 2. Config Server — Máy Chủ Cấu Hình

### Tạo Config Server

```xml
<!-- config-server/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- Bảo mật Config Server -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

```java
// ConfigServerApplication.java
@SpringBootApplication
@EnableConfigServer  // Bật Config Server
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Cấu Hình Git Backend

```yaml
# config-server/src/main/resources/application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  
  cloud:
    config:
      server:
        git:
          # URL của Git repository chứa cấu hình
          uri: https://github.com/my-org/config-repo
          
          # Branch mặc định
          default-label: main
          
          # Timeout kết nối Git (milliseconds)
          timeout: 10
          
          # Clone on start — tải về ngay khi khởi động (không lazy)
          clone-on-start: true
          
          # Force pull — luôn pull từ Git, không dùng cache cũ
          force-pull: true
          
          # Credentials — thông tin xác thực
          username: ${GIT_USERNAME}
          password: ${GIT_TOKEN}
          
          # Nếu dùng SSH key
          # private-key: ${GIT_SSH_KEY}
          # known-hosts-file: known_hosts
          
          # Search paths — tìm config trong sub-folder theo {application}
          search-paths: '{application}'
          
          # Cấu hình cho nhiều repositories
          # repos:
          #   microservices:
          #     pattern: microservice-*
          #     uri: https://github.com/my-org/microservice-configs
          #   shared:
          #     uri: https://github.com/my-org/shared-configs

# Bảo mật Config Server với basic auth
spring:
  security:
    user:
      name: config-admin
      password: ${CONFIG_SERVER_PASSWORD}

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

### Cấu Hình Native Backend (Filesystem — Dùng Cho Development)

```yaml
# application-dev.yml (profile dev)
spring:
  cloud:
    config:
      server:
        native:
          # Đọc config từ local filesystem
          search-locations:
            - classpath:/config
            - file:./config-repo
  profiles:
    active: native
```

---

## 3. Cấu Trúc Repository Config

```
config-repo/ (Git repository)
│
├── application.yml                  ← Mặc định cho TẤT CẢ services
├── application-production.yml       ← Mặc định cho env production
│
├── order-service.yml                ← Cho order-service (mọi env)
├── order-service-development.yml    ← Cho order-service, env development
├── order-service-production.yml     ← Cho order-service, env production
│
├── user-service.yml
├── user-service-production.yml
│
└── shared/
    ├── database.yml                 ← Cấu hình DB dùng chung
    └── security.yml                 ← Cấu hình security dùng chung
```

### Nội Dung File Config Mẫu

```yaml
# application.yml — Mặc định cho tất cả services
logging:
  level:
    root: INFO
    com.example: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: always

spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
```

```yaml
# order-service.yml — Cấu hình riêng cho order-service
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://postgres:5432/orders
    username: order_user
    # password inject từ Kubernetes Secret, không để trong Git

order:
  processing:
    timeout: 30s
    max-retry: 3
  kafka:
    topic: order-events
    consumer-group: order-processors
```

```yaml
# order-service-production.yml — Override cho production
logging:
  level:
    root: WARN
    com.example: INFO

spring:
  datasource:
    hikari:
      maximum-pool-size: 50    # Production cần pool lớn hơn

order:
  processing:
    timeout: 60s               # Production timeout dài hơn
```

### Quy Tắc Priority — Ưu Tiên Của Config

Config Server merge các file theo thứ tự ưu tiên **từ cao đến thấp**:

```
1. {application}-{profile}.yml    (VD: order-service-production.yml)
2. {application}.yml              (VD: order-service.yml)
3. application-{profile}.yml      (VD: application-production.yml)
4. application.yml                (default, ưu tiên thấp nhất)
```

---

## 4. Config Client — Service Kết Nối Đến Config Server

### Thêm Dependencies

```xml
<!-- service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <!-- Retry khi Config Server tạm thời không khả dụng -->
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <!-- Actuator để refresh config -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### Cấu Hình Client

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: order-service    # QUAN TRỌNG: phải khớp với tên file config
  
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:development}
  
  config:
    # Địa chỉ Config Server
    import: "optional:configserver:http://config-server:8888"
  
  cloud:
    config:
      # Username/password để xác thực với Config Server
      username: ${CONFIG_SERVER_USERNAME:config-admin}
      password: ${CONFIG_SERVER_PASSWORD}
      
      # Fail fast — thất bại ngay nếu không kết nối được
      fail-fast: true
      
      # Retry configuration — cấu hình thử lại
      retry:
        initial-interval: 1000     # Chờ 1s trước lần retry đầu
        multiplier: 1.5            # Tăng interval mỗi lần (1s, 1.5s, 2.25s, ...)
        max-attempts: 6            # Thử tối đa 6 lần
        max-interval: 2000         # Tối đa 2s giữa các lần retry

# Expose refresh endpoint
management:
  endpoints:
    web:
      exposure:
        include: health,refresh,busrefresh
```

---

## 5. @RefreshScope — Làm Mới Cấu Hình Tại Runtime

`@RefreshScope` — Phạm Vi Làm Mới — đánh dấu một bean sẽ được khởi tạo lại khi có lệnh refresh, nhận giá trị config mới mà không cần restart ứng dụng.

### Sử Dụng @RefreshScope

```java
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
@RefreshScope  // Bean này sẽ được tạo lại khi có refresh event
public class FeatureFlagService {

    @Value("${features.new-payment-flow.enabled:false}")
    private boolean newPaymentFlowEnabled;

    @Value("${features.max-order-amount:100000}")
    private int maxOrderAmount;

    @Value("${rate-limiting.requests-per-minute:60}")
    private int requestsPerMinute;

    public boolean isNewPaymentFlowEnabled() {
        return newPaymentFlowEnabled;
    }

    public int getMaxOrderAmount() {
        return maxOrderAmount;
    }
}
```

```java
// @ConfigurationProperties cũng hỗ trợ refresh
@Component
@RefreshScope
@ConfigurationProperties(prefix = "order")
public class OrderProperties {

    private Processing processing = new Processing();
    private Kafka kafka = new Kafka();

    @Data
    public static class Processing {
        private Duration timeout = Duration.ofSeconds(30);
        private int maxRetry = 3;
    }

    @Data
    public static class Kafka {
        private String topic = "order-events";
        private String consumerGroup = "order-processors";
    }

    // getters/setters ...
}
```

### Kích Hoạt Refresh

```bash
# Refresh một service cụ thể
POST http://order-service:8080/actuator/refresh
Content-Type: application/json

# Response: danh sách config keys đã thay đổi
# ["order.processing.timeout", "features.new-payment-flow.enabled"]
```

---

## 6. Spring Cloud Bus — Refresh Tất Cả Instances

Khi có nhiều instances của một service, cần dùng Spring Cloud Bus để broadcast refresh event đến tất cả instances qua message broker.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    <!-- Hoặc dùng RabbitMQ: spring-cloud-starter-bus-amqp -->
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    bus:
      enabled: true
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: busrefresh
```

```bash
# Refresh TẤT CẢ instances của TẤT CẢ services
POST http://config-server:8888/actuator/busrefresh

# Refresh chỉ một service cụ thể
POST http://config-server:8888/actuator/busrefresh/order-service:**

# Hoặc từ một instance bất kỳ
POST http://order-service:8080/actuator/busrefresh
```

---

## 7. Git Webhook Tự Động Refresh

Kết hợp Git webhook với Spring Cloud Bus để tự động refresh khi có commit mới vào config repository:

```
Developer push config to Git
         │
         ▼
Git Repository sends webhook
         │
         ▼
Config Server receives webhook
         │
         ▼ POST /actuator/busrefresh
Spring Cloud Bus (Kafka/RabbitMQ)
         │
         ├──► order-service (instance 1) → refresh
         ├──► order-service (instance 2) → refresh
         └──► user-service (all instances) → refresh
```

```java
// Config Server — nhận webhook từ GitHub
@RestController
@RequestMapping("/webhook")
public class GitWebhookController {

    private final ApplicationEventPublisher eventPublisher;

    // GitHub sẽ POST đến đây khi có push event
    @PostMapping("/github")
    public ResponseEntity<Void> handleGitHubWebhook(
            @RequestHeader("X-Hub-Signature-256") String signature,
            @RequestBody String payload) {
        
        if (!verifySignature(signature, payload)) {
            return ResponseEntity.status(403).build();
        }
        
        // Trigger bus refresh
        eventPublisher.publishEvent(new RefreshRemoteApplicationEvent(
            this, UUID.randomUUID().toString(), null
        ));
        
        return ResponseEntity.ok().build();
    }
}
```

---

## 8. Mã Hóa Secrets Trong Config

Config Server hỗ trợ mã hóa/giải mã giá trị để lưu trữ secrets an toàn trong Git.

### Cấu Hình Encryption Key

```yaml
# config-server application.yml
encrypt:
  # Symmetric key — Khóa Đối Xứng (đơn giản nhưng kém bảo mật hơn)
  key: ${ENCRYPT_KEY}
  
  # Hoặc dùng Keystore — Kho Khóa (khuyến nghị cho production)
  # key-store:
  #   location: classpath:keystore.jks
  #   password: ${KEYSTORE_PASSWORD}
  #   alias: config-key
  #   secret: ${KEY_SECRET}
```

```bash
# Mã hóa giá trị
curl -X POST http://config-server:8888/encrypt \
  -H "Authorization: Basic ..." \
  -d "mysecretpassword"
# Output: AQC8KxW9aBcD...

# Giải mã giá trị
curl -X POST http://config-server:8888/decrypt \
  -H "Authorization: Basic ..." \
  -d "AQC8KxW9aBcD..."
# Output: mysecretpassword
```

### Sử Dụng Giá Trị Đã Mã Hóa Trong Config File

```yaml
# order-service.yml trong Git repository
spring:
  datasource:
    password: '{cipher}AQC8KxW9aBcDeFgH...'  # Tiền tố {cipher}
```

Config Server tự động giải mã khi client fetch về.

---

## 9. Vault Backend — Tích Hợp HashiCorp Vault

Cho production, nên dùng HashiCorp Vault thay vì Git để lưu secrets:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  cloud:
    vault:
      host: vault.example.com
      port: 8200
      scheme: https
      authentication: APPROLE
      app-role:
        role-id: ${VAULT_ROLE_ID}
        secret-id: ${VAULT_SECRET_ID}
      kv:
        enabled: true
        backend: secret
        default-context: ${spring.application.name}
```

---

## 10. Config Server Với Kubernetes

Trong môi trường Kubernetes, có thể dùng ConfigMap/Secret trực tiếp thay vì Config Server, hoặc kết hợp cả hai:

```yaml
# Kubernetes ConfigMap cho Spring Boot
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  # Spring Boot tự động đọc từ /config/application.yaml
  application.yaml: |
    server:
      port: 8080
    order:
      processing:
        timeout: 30s
```

```yaml
# Deployment mount ConfigMap
spec:
  containers:
    - name: order-service
      volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: order-service-config
```

```properties
# Kích hoạt Kubernetes Config watcher
spring.cloud.kubernetes.config.enabled=true
spring.cloud.kubernetes.config.sources[0].name=order-service-config
spring.cloud.kubernetes.reload.enabled=true
spring.cloud.kubernetes.reload.strategy=refresh  # Refresh beans thay vì restart
```

---

## 11. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao cần Config Server thay vì để config trong từng service?**

A: Với microservices, cùng một thay đổi config (ví dụ: database URL, feature flag) phải cập nhật ở nhiều services và nhiều instances. Config Server tập trung tất cả ở một nơi, có lịch sử thay đổi qua Git, hỗ trợ phân tách theo environment, và cho phép cập nhật runtime không cần deploy lại.

**Q: @RefreshScope hoạt động thế nào?**

A: `@RefreshScope` là một Spring scope đặc biệt. Bean được wrapped trong proxy. Khi nhận `RefreshEvent` (từ `/actuator/refresh`), Spring phá hủy bean instance hiện tại và tạo mới với config values mới nhất từ `Environment`. Lần gọi tiếp theo sẽ dùng bean mới.

**Q: Spring Cloud Bus khác gì với /actuator/refresh?**

A: `/actuator/refresh` chỉ refresh một instance đơn lẻ. Spring Cloud Bus broadcast refresh event qua message broker (Kafka/RabbitMQ) đến tất cả instances đang subscribe. Hữu ích khi scale horizontally — mở rộng theo chiều ngang — với nhiều instances cần đồng bộ cấu hình.

**Q: Làm sao bảo vệ Config Server khỏi unauthorized access?**

A: (1) Bảo mật Config Server API với Spring Security (basic auth hoặc OAuth2), (2) Chạy Config Server trong internal network, không expose ra internet, (3) Mã hóa sensitive values trong Git repository bằng `{cipher}` syntax, (4) Dùng HashiCorp Vault làm backend cho secrets thay vì Git.

---

## ✅ Checklist

- [ ] Config Server chạy với Git backend được bảo mật
- [ ] Config repository có cấu trúc `{application}-{profile}.yml`
- [ ] Client kết nối với `fail-fast=true` và retry config
- [ ] Sensitive values được mã hóa bằng `{cipher}` hoặc Vault
- [ ] `@RefreshScope` trên beans sử dụng dynamic config
- [ ] `/actuator/refresh` endpoint được bảo vệ
- [ ] Spring Cloud Bus cho môi trường multi-instance
- [ ] Git webhook tự động trigger refresh
- [ ] Config Server có health check và monitoring
- [ ] Test refresh hoạt động đúng mà không restart
