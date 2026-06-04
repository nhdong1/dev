# Spring Boot Actuator — Health, Metrics & Custom Endpoints

> Spring Boot Actuator cung cấp các endpoint sẵn sàng cho production để giám sát và quản lý ứng dụng: health checks (kiểm tra sức khỏe), metrics (chỉ số), info, và khả năng tạo custom endpoints phù hợp với nghiệp vụ.

---

## 1. Giới Thiệu Actuator

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Sau khi thêm dependency, Actuator tự động expose các endpoint tại `/actuator/`:

```bash
curl http://localhost:8080/actuator
# Trả về danh sách các endpoint đang active
```

---

## 2. Cấu Hình Expose Endpoints

```properties
# application.properties

# Expose tất cả endpoints qua HTTP
management.endpoints.web.exposure.include=*

# Chỉ expose một số endpoints cần thiết (khuyến nghị cho production)
management.endpoints.web.exposure.include=health,info,metrics,prometheus,loggers

# Loại trừ endpoint nhạy cảm
management.endpoints.web.exposure.exclude=shutdown,env,beans

# Thay đổi base path của actuator (mặc định: /actuator)
management.endpoints.web.base-path=/management

# Chạy Actuator trên port riêng (không expose ra internet)
management.server.port=8081
management.server.address=127.0.0.1  # Chỉ localhost
```

### Danh Sách Endpoints Quan Trọng

| Endpoint | Mô Tả | Nhạy Cảm |
|---------|--------|-----------|
| `/actuator/health` | Trạng thái sức khỏe ứng dụng | Thấp |
| `/actuator/health/liveness` | Liveness probe cho K8s | Không |
| `/actuator/health/readiness` | Readiness probe cho K8s | Không |
| `/actuator/info` | Thông tin ứng dụng | Không |
| `/actuator/metrics` | Danh sách metrics | Trung bình |
| `/actuator/prometheus` | Metrics định dạng Prometheus | Trung bình |
| `/actuator/loggers` | Xem và thay đổi log level | Cao |
| `/actuator/env` | Biến môi trường | **Rất Cao** |
| `/actuator/beans` | Danh sách Spring beans | Cao |
| `/actuator/threaddump` | Thread dump | Trung bình |
| `/actuator/heapdump` | Heap dump | **Rất Cao** |
| `/actuator/shutdown` | Tắt ứng dụng | **Tối Cao** |

---

## 3. Health Endpoint — Endpoint Sức Khỏe

### Cấu Hình Chi Tiết

```properties
# Hiển thị chi tiết cho mọi request (development)
management.endpoint.health.show-details=always

# Chỉ hiển thị khi authenticated (production)
management.endpoint.health.show-details=when-authorized

# Bật health probes cho K8s
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
management.endpoint.health.probes.enabled=true

# Thêm thông tin group
management.endpoint.health.group.custom.include=db,redis,kafka
```

### Response Health Check

```json
// GET /actuator/health
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.11"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 107374182400,
        "free": 53687091200,
        "threshold": 10485760,
        "path": "/"
      }
    }
  }
}
```

```json
// GET /actuator/health/liveness
{ "status": "UP" }

// GET /actuator/health/readiness
{
  "status": "UP",
  "components": {
    "readinessState": { "status": "UP" }
  }
}
```

### Các Health Indicator Tích Hợp Sẵn

Spring Boot tự động tạo health indicator cho các component:

| Component | Health Indicator |
|-----------|-----------------|
| DataSource / JPA | `DataSourceHealthIndicator` |
| Redis | `RedisHealthIndicator` |
| Kafka | `KafkaHealthIndicator` |
| RabbitMQ | `RabbitHealthIndicator` |
| MongoDB | `MongoHealthIndicator` |
| Elasticsearch | `ElasticsearchHealthIndicator` |
| Disk Space | `DiskSpaceHealthIndicator` |

---

## 4. Custom Health Indicator — Chỉ Số Sức Khỏe Tùy Chỉnh

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

// Kiểm tra sức khỏe external payment service
@Component("paymentService")  // Tên trong /actuator/health
public class PaymentServiceHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient paymentClient;

    public PaymentServiceHealthIndicator(PaymentGatewayClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @Override
    public Health health() {
        try {
            boolean isAvailable = paymentClient.ping();
            
            if (isAvailable) {
                return Health.up()
                    .withDetail("url", paymentClient.getBaseUrl())
                    .withDetail("responseTime", paymentClient.getLastResponseTime() + "ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("reason", "Payment gateway không phản hồi")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

```json
// Kết quả trong /actuator/health
{
  "status": "UP",
  "components": {
    "paymentService": {
      "status": "UP",
      "details": {
        "url": "https://payment.example.com",
        "responseTime": "45ms"
      }
    }
  }
}
```

### Reactive Health Indicator (Spring WebFlux)

```java
@Component
public class ExternalApiHealthIndicator implements ReactiveHealthIndicator {

    private final WebClient webClient;

    @Override
    public Mono<Health> health() {
        return webClient.get()
            .uri("/ping")
            .retrieve()
            .toBodilessEntity()
            .map(response -> Health.up().build())
            .onErrorReturn(Health.down().build())
            .timeout(Duration.ofSeconds(3));
    }
}
```

---

## 5. Info Endpoint — Thông Tin Ứng Dụng

### Cấu Hình Thông Tin Tĩnh

```properties
# application.properties
management.info.env.enabled=true
management.info.java.enabled=true
management.info.os.enabled=true
management.info.build.enabled=true
management.info.git.enabled=true

# Thông tin tùy chỉnh
info.app.name=My Spring Boot App
info.app.description=Backend API service
info.app.version=@project.version@
info.app.team=backend-team
info.contact.email=backend@example.com
```

### Tích Hợp Build Info Từ Maven

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <!-- Tạo build-info.properties tự động -->
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Tích Hợp Git Info

```xml
<!-- pom.xml -->
<plugin>
    <groupId>io.github.git-commit-id</groupId>
    <artifactId>git-commit-id-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>revision</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!-- Tạo git.properties -->
        <generateGitPropertiesFile>true</generateGitPropertiesFile>
        <includeOnlyProperties>
            <includeOnlyProperty>git.branch</includeOnlyProperty>
            <includeOnlyProperty>git.commit.id.abbrev</includeOnlyProperty>
            <includeOnlyProperty>git.commit.time</includeOnlyProperty>
        </includeOnlyProperties>
    </configuration>
</plugin>
```

```json
// GET /actuator/info
{
  "app": {
    "name": "My Spring Boot App",
    "version": "1.2.0",
    "team": "backend-team"
  },
  "build": {
    "artifact": "myapp",
    "version": "1.2.0",
    "time": "2026-06-02T08:30:00.000Z"
  },
  "git": {
    "branch": "main",
    "commit": {
      "id": { "abbrev": "a1b2c3d" },
      "time": "2026-06-01T15:00:00.000Z"
    }
  },
  "java": {
    "version": "21.0.3",
    "vendor": { "name": "Eclipse Adoptium" }
  }
}
```

---

## 6. Metrics Endpoint — Endpoint Chỉ Số

```bash
# Liệt kê tất cả metric names
GET /actuator/metrics

# Xem chi tiết một metric
GET /actuator/metrics/http.server.requests
GET /actuator/metrics/jvm.memory.used
GET /actuator/metrics/hikaricp.connections.active

# Filter theo tag
GET /actuator/metrics/http.server.requests?tag=uri:/api/users&tag=status:200
```

```json
// GET /actuator/metrics/http.server.requests
{
  "name": "http.server.requests",
  "measurements": [
    { "statistic": "COUNT", "value": 1250 },
    { "statistic": "TOTAL_TIME", "value": 12.5 },
    { "statistic": "MAX", "value": 0.345 }
  ],
  "availableTags": [
    { "tag": "uri", "values": ["/api/users", "/api/orders"] },
    { "tag": "method", "values": ["GET", "POST"] },
    { "tag": "status", "values": ["200", "404", "500"] }
  ]
}
```

### Metrics JVM Quan Trọng

```bash
# Heap memory usage — Sử dụng bộ nhớ heap
GET /actuator/metrics/jvm.memory.used?tag=area:heap
GET /actuator/metrics/jvm.memory.max?tag=area:heap

# GC (Garbage Collector — Bộ Thu Gom Rác) pause time
GET /actuator/metrics/jvm.gc.pause

# Thread count
GET /actuator/metrics/jvm.threads.live
GET /actuator/metrics/jvm.threads.daemon

# Database connection pool (HikariCP)
GET /actuator/metrics/hikaricp.connections.active
GET /actuator/metrics/hikaricp.connections.idle
GET /actuator/metrics/hikaricp.connections.pending

# Cache hit rate
GET /actuator/metrics/cache.gets?tag=result:hit
GET /actuator/metrics/cache.gets?tag=result:miss
```

---

## 7. Custom Metrics Với Micrometer

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    private final AtomicInteger pendingOrders = new AtomicInteger(0);

    public OrderService(MeterRegistry registry) {
        // Counter — Bộ Đếm: chỉ tăng, không giảm
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Tổng số đơn hàng đã tạo")
            .tag("service", "order")
            .register(registry);

        this.orderFailedCounter = Counter.builder("orders.failed")
            .description("Tổng số đơn hàng thất bại")
            .tag("service", "order")
            .register(registry);

        // Timer — Bộ Đếm Thời Gian: đo duration và count
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Thời gian xử lý đơn hàng")
            .tag("service", "order")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .register(registry);

        // Gauge — Thước Đo: giá trị có thể tăng/giảm
        Gauge.builder("orders.pending", pendingOrders, AtomicInteger::get)
            .description("Số đơn hàng đang chờ xử lý")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                pendingOrders.incrementAndGet();
                Order order = processOrder(request);
                orderCreatedCounter.increment();
                return order;
            } catch (Exception e) {
                orderFailedCounter.increment();
                throw e;
            } finally {
                pendingOrders.decrementAndGet();
            }
        });
    }
}
```

### @Timed Annotation — Annotation Đo Thời Gian

```java
import io.micrometer.core.annotation.Timed;
import io.micrometer.core.annotation.Counted;

@Service
@Timed("service.method.time")  // Áp dụng cho tất cả public methods
public class UserService {

    @Timed(
        value = "users.find.time",
        description = "Thời gian tìm kiếm user",
        percentiles = {0.5, 0.95, 0.99},
        extraTags = {"entity", "user"}
    )
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

> Cần thêm `MicrometerTimedAspect` bean:

```java
@Configuration
public class MetricsConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

---

## 8. Custom Actuator Endpoint — Endpoint Tùy Chỉnh

```java
import org.springframework.boot.actuate.endpoint.annotation.*;
import org.springframework.stereotype.Component;

@Component
@Endpoint(id = "cache")  // Accessible tại /actuator/cache
public class CacheActuatorEndpoint {

    private final CacheManager cacheManager;

    public CacheActuatorEndpoint(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    // GET /actuator/cache
    @ReadOperation
    public Map<String, Object> cacheInfo() {
        Map<String, Object> info = new HashMap<>();
        cacheManager.getCacheNames().forEach(name -> {
            Cache cache = cacheManager.getCache(name);
            info.put(name, Map.of(
                "size", getCacheSize(cache),
                "type", cache.getClass().getSimpleName()
            ));
        });
        return info;
    }

    // DELETE /actuator/cache/{cacheName}
    @DeleteOperation
    public void clearCache(@Selector String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.clear();
        }
    }

    // POST /actuator/cache/{cacheName}
    @WriteOperation
    public void evictKey(@Selector String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
        }
    }

    private long getCacheSize(Cache cache) {
        // Logic tùy thuộc vào implementation của cache
        return -1L;
    }
}
```

---

## 9. Loggers Endpoint — Thay Đổi Log Level Tại Runtime

```bash
# Xem log level hiện tại
GET /actuator/loggers
GET /actuator/loggers/com.example.service

# Thay đổi log level không cần restart (rất hữu ích khi debug production)
POST /actuator/loggers/com.example.service
Content-Type: application/json
{ "configuredLevel": "DEBUG" }

# Reset về mặc định
POST /actuator/loggers/com.example.service
Content-Type: application/json
{ "configuredLevel": null }
```

---

## 10. Bảo Mật Actuator Endpoints

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)  // Ưu tiên cao hơn security config chính
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                // Health và info cho K8s probes — không cần auth
                .requestMatchers(
                    "/actuator/health",
                    "/actuator/health/liveness",
                    "/actuator/health/readiness",
                    "/actuator/info"
                ).permitAll()
                // Metrics chỉ cho monitoring system (internal network)
                .requestMatchers("/actuator/prometheus").hasIpAddress("10.0.0.0/8")
                // Các endpoint nhạy cảm — yêu cầu ADMIN role
                .requestMatchers("/actuator/**").hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

```properties
# Credentials cho basic auth của actuator
spring.security.user.name=actuator
spring.security.user.password=${ACTUATOR_PASSWORD}
spring.security.user.roles=ACTUATOR_ADMIN
```

---

## 11. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Spring Boot Actuator là gì và tại sao cần nó?**

A: Actuator là module cung cấp các endpoint production-ready để giám sát và quản lý ứng dụng mà không cần code thêm. Quan trọng nhất là `/health` cho Kubernetes probes, `/metrics` và `/prometheus` cho monitoring stack, và `/loggers` để thay đổi log level tại runtime không cần restart.

**Q: Sự khác biệt giữa `/actuator/health/liveness` và `/actuator/health/readiness`?**

A: `liveness` — còn sống — cho biết ứng dụng có đang chạy hay bị deadlock/crash không. Nếu fail, K8s restart Pod. `readiness` — sẵn sàng — cho biết ứng dụng có thể phục vụ traffic không (DB kết nối được chưa, cache khởi tạo xong chưa). Nếu fail, K8s không route traffic vào Pod nhưng không restart.

**Q: Làm sao tạo custom health indicator?**

A: Implement interface `HealthIndicator` (hoặc `ReactiveHealthIndicator` cho WebFlux), annotate với `@Component`. Spring Boot tự động discover và thêm vào `/actuator/health`. Trả về `Health.up()` hoặc `Health.down()` kèm details tùy ý.

**Q: Cách bảo vệ Actuator endpoints trong production?**

A: (1) Expose trên port management riêng không public ra internet, (2) Chỉ expose endpoints cần thiết qua `management.endpoints.web.exposure.include`, (3) Dùng Spring Security với IP whitelist cho metrics endpoints, (4) Health và info có thể public cho K8s probes.

---

## ✅ Checklist

- [ ] Thêm `spring-boot-starter-actuator` dependency
- [ ] Chỉ expose endpoints cần thiết (không expose `*` ở production)
- [ ] Bật liveness và readiness probes cho K8s
- [ ] Cấu hình `server.shutdown=graceful` kết hợp với probes
- [ ] Chạy Actuator trên management port riêng (8081)
- [ ] Bảo vệ endpoint nhạy cảm (env, beans, heapdump, shutdown)
- [ ] Cấu hình build-info và git-info cho `/actuator/info`
- [ ] Thêm custom health indicator cho external dependencies
- [ ] Tạo custom metrics với Micrometer Counter/Timer/Gauge
- [ ] Test health endpoint trả về 200 cho healthy state
