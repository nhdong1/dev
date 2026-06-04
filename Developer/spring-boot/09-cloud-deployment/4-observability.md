# Observability — Micrometer, Prometheus, Grafana, Zipkin & Jaeger

> Observability — Khả Năng Quan Sát — là khả năng hiểu trạng thái bên trong hệ thống dựa trên output bên ngoài (metrics, traces, logs). Hướng dẫn này bao gồm ba trụ cột observability cho Spring Boot: Metrics với Micrometer + Prometheus + Grafana, Distributed Tracing — Theo Dõi Phân Tán — với Zipkin/Jaeger, và Structured Logging — Ghi Nhật Ký Có Cấu Trúc.

---

## 1. Ba Trụ Cột Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Stack                          │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐  │
│  │   METRICS     │  │    TRACES     │  │      LOGS         │  │
│  │  (Chỉ Số)     │  │  (Dấu Vết)    │  │   (Nhật Ký)       │  │
│  │               │  │               │  │                   │  │
│  │ Micrometer    │  │  Micrometer   │  │  Logback          │  │
│  │     ↓         │  │   Tracing     │  │      ↓            │  │
│  │ Prometheus    │  │     ↓         │  │  ELK Stack        │  │
│  │     ↓         │  │ Zipkin/Jaeger │  │  hoặc Loki        │  │
│  │  Grafana      │  │               │  │                   │  │
│  └───────────────┘  └───────────────┘  └───────────────────┘  │
│                                                                 │
│  Hỏi: "Có bao nhiêu  Hỏi: "Request     Hỏi: "Tại sao         │
│  lỗi 500?"           này chậm ở đâu?"  lỗi xảy ra?"          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Micrometer — Facade Metrics

Micrometer — Facade Chỉ Số — là lớp trừu tượng cho metrics, tương tự như SLF4J cho logging. Spring Boot tự động cấu hình `MeterRegistry`.

```xml
<!-- pom.xml -->
<!-- Actuator đã bao gồm Micrometer core -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Registry cho Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```properties
# application.properties
# Expose Prometheus endpoint
management.endpoints.web.exposure.include=health,prometheus,metrics
management.metrics.export.prometheus.enabled=true

# Thêm common tags cho tất cả metrics
management.metrics.tags.application=${spring.application.name}
management.metrics.tags.environment=${spring.profiles.active:default}
management.metrics.tags.region=ap-southeast-1
```

### Các Loại Meter Trong Micrometer

```java
import io.micrometer.core.instrument.*;

@Component
public class MetricsDemoService {

    private final MeterRegistry registry;

    // Counter — Bộ Đếm: tăng đơn điệu, không giảm
    private final Counter requestCounter;

    // Timer — Đồng Hồ: đo duration và count
    private final Timer dbQueryTimer;

    // Gauge — Thước Đo: giá trị hiện tại (có thể tăng/giảm)
    private final AtomicInteger activeConnections = new AtomicInteger(0);

    // DistributionSummary — Tóm Tắt Phân Phối: đo giá trị (bytes, rows, ...)
    private final DistributionSummary payloadSizeSummary;

    public MetricsDemoService(MeterRegistry registry) {
        this.registry = registry;

        this.requestCounter = Counter.builder("api.requests.total")
            .description("Tổng số request API")
            .tag("service", "demo")
            .register(registry);

        this.dbQueryTimer = Timer.builder("db.query.duration")
            .description("Thời gian truy vấn database")
            .publishPercentiles(0.5, 0.95, 0.99)       // p50, p95, p99
            .publishPercentileHistogram()                 // Histogram cho Grafana
            .sla(Duration.ofMillis(100), Duration.ofMillis(500))  // SLA thresholds
            .register(registry);

        Gauge.builder("api.connections.active", activeConnections, AtomicInteger::get)
            .description("Số kết nối đang hoạt động")
            .register(registry);

        this.payloadSizeSummary = DistributionSummary.builder("http.request.size")
            .description("Kích thước request payload (bytes)")
            .baseUnit("bytes")
            .publishPercentiles(0.5, 0.95)
            .register(registry);
    }

    public void recordRequest(int payloadBytes) {
        requestCounter.increment();
        payloadSizeSummary.record(payloadBytes);
    }

    public <T> T executeWithTimer(Supplier<T> operation) {
        return dbQueryTimer.record(operation);
    }
}
```

### Tùy Chỉnh Metrics Cho Business (Nghiệp Vụ)

```java
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
    // Counter với tags động
    registry.counter("orders.created",
        "product_category", event.getCategory(),
        "payment_method", event.getPaymentMethod(),
        "region", event.getRegion()
    ).increment();

    // Timer record với tags
    Timer.builder("order.processing.duration")
        .tag("order_type", event.getType())
        .register(registry)
        .record(event.getProcessingTime());
}
```

---

## 3. Prometheus — Thu Thập Metrics

### Cách Hoạt Động

```
Spring Boot App                  Prometheus Server
    │                                │
    │ /actuator/prometheus           │
    │ ◄──── SCRAPE mỗi 15s ─────────│
    │                                │
    │ counter, gauge, timer data     │
    │ ─────────────────────────────► │
    │                                │ Lưu trữ time-series data
                                     │ Query với PromQL
                                     │
                                     ▼
                                  Grafana
                                (Visualization)
```

### Cấu Hình `prometheus.yml`

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s      # Thu thập metrics mỗi 15 giây
  evaluation_interval: 15s  # Đánh giá alerting rules mỗi 15 giây

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"

scrape_configs:
  # Thu thập metrics từ Spring Boot app
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'myapp:8080'
        labels:
          application: 'myapp'
          environment: 'production'

  # Prometheus tự monitor chính nó
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Kubernetes service discovery — tự động phát hiện services trong K8s
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
```

### Alerting Rules — Quy Tắc Cảnh Báo

```yaml
# prometheus/alert_rules.yml
groups:
  - name: spring-boot-alerts
    rules:
      # Cảnh báo khi error rate (tỷ lệ lỗi) > 5%
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          /
          rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate cao trên {{ $labels.application }}"
          description: "Error rate là {{ $value | humanizePercentage }}"

      # Cảnh báo khi p99 latency > 1 giây
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latency (độ trễ) p99 cao trên {{ $labels.application }}"

      # Cảnh báo khi JVM heap usage > 85%
      - alert: JvmHeapHigh
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM Heap usage cao: {{ $value | humanizePercentage }}"

      # Cảnh báo khi HikariCP connection pool cạn
      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_pending > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Connection pool bị cạn trên {{ $labels.application }}"
```

### PromQL — Ngôn Ngữ Truy Vấn Prometheus

```promql
# ─── HTTP Metrics ───

# Request rate (số request/giây) trong 5 phút qua
rate(http_server_requests_seconds_count[5m])

# Error rate (tỷ lệ lỗi 5xx)
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
/
rate(http_server_requests_seconds_count[5m])

# P99 latency (độ trễ bách phân vị 99)
histogram_quantile(0.99,
  rate(http_server_requests_seconds_bucket[5m])
)

# ─── JVM Metrics ───

# Heap usage theo phần trăm
jvm_memory_used_bytes{area="heap"}
/
jvm_memory_max_bytes{area="heap"} * 100

# GC pause time rate
rate(jvm_gc_pause_seconds_sum[5m])

# ─── HikariCP Metrics ───

# Connection pool utilization (mức sử dụng bể kết nối)
hikaricp_connections_active
/
hikaricp_connections_max

# ─── Business Metrics ───

# Orders per minute (đơn hàng mỗi phút)
rate(orders_created_total[1m]) * 60

# Average order processing time (thời gian xử lý đơn hàng trung bình)
rate(orders_processing_duration_seconds_sum[5m])
/
rate(orders_processing_duration_seconds_count[5m])
```

---

## 4. Grafana — Visualization Metrics

### Docker Compose Setup

```yaml
services:
  grafana:
    image: grafana/grafana:10.x
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

volumes:
  grafana_data:
  prometheus_data:
```

### Provisioning Data Sources Tự Động

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    access: proxy
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: jaeger
```

### Dashboard Panels Quan Trọng

```json
// Ví dụ panel "Request Rate" trong Grafana JSON model
{
  "title": "HTTP Request Rate",
  "type": "graph",
  "targets": [
    {
      "expr": "rate(http_server_requests_seconds_count{application=\"$application\"}[5m])",
      "legendFormat": "{{method}} {{uri}} - {{status}}"
    }
  ]
}
```

**Dashboards cộng đồng miễn phí:**
- **Spring Boot 2.1+ Statistics** — Dashboard ID: `12900`
- **JVM (Micrometer)** — Dashboard ID: `4701`
- **HikariCP** — Dashboard ID: `6083`

---

## 5. Distributed Tracing — Theo Dõi Phân Tán

Distributed Tracing cho phép theo dõi một request xuyên suốt nhiều microservices bằng cách truyền `traceId` và `spanId` qua HTTP headers.

```
User Request
     │
     ▼
[API Gateway]  traceId=abc123, spanId=001
     │
     ├──► [Auth Service]  traceId=abc123, spanId=002
     │
     ├──► [User Service]  traceId=abc123, spanId=003
     │         │
     │         └──► [Database]  traceId=abc123, spanId=004
     │
     └──► [Order Service]  traceId=abc123, spanId=005
               │
               └──► [Kafka Producer]  traceId=abc123, spanId=006
```

---

## 6. Micrometer Tracing + Zipkin

### Dependencies

```xml
<!-- pom.xml -->
<!-- Micrometer Tracing Bridge cho Brave (Zipkin) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Zipkin Reporter -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<!-- Zipkin Sender qua HTTP -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-sender-okhttp3</artifactId>
</dependency>
```

### Cấu Hình

```properties
# application.properties
spring.application.name=order-service

# Zipkin server URL
management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans

# Sampling rate — Tỷ Lệ Lấy Mẫu
# 1.0 = 100% requests được trace (development)
# 0.1 = 10% requests được trace (production)
management.tracing.sampling.probability=0.1

# Propagation type — W3C (chuẩn OpenTelemetry) hoặc B3 (Zipkin)
management.tracing.propagation.type=b3
```

### Zipkin Docker Compose

```yaml
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      STORAGE_TYPE: mem  # Development; dùng elasticsearch cho production
```

---

## 7. Micrometer Tracing + Jaeger (OpenTelemetry)

### Dependencies

```xml
<!-- pom.xml -->
<!-- Micrometer Tracing Bridge cho OpenTelemetry -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<!-- OpenTelemetry Exporter cho Jaeger/OTLP -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Cấu Hình

```properties
# application.properties
spring.application.name=user-service

# OTLP endpoint (Jaeger hỗ trợ OTLP từ v1.35)
management.otlp.tracing.endpoint=http://jaeger:4318/v1/traces

# Sampling rate
management.tracing.sampling.probability=0.1
```

```yaml
# Jaeger Docker Compose
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4318:4318"    # OTLP HTTP receiver
      - "4317:4317"    # OTLP gRPC receiver
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

---

## 8. Custom Spans — Khoảng Thời Gian Tùy Chỉnh

```java
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.Span;

@Service
public class PaymentService {

    private final Tracer tracer;
    private final ExternalPaymentClient paymentClient;

    public PaymentResult processPayment(PaymentRequest request) {
        // Tạo custom span để trace bước xử lý cụ thể
        Span span = tracer.nextSpan()
            .name("payment.process")
            .tag("payment.method", request.getMethod())
            .tag("payment.amount", String.valueOf(request.getAmount()))
            .start();

        try (Tracer.SpanInScope scope = tracer.withSpan(span)) {
            // Validation span con
            Span validationSpan = tracer.nextSpan().name("payment.validate").start();
            try (Tracer.SpanInScope vs = tracer.withSpan(validationSpan)) {
                validatePayment(request);
            } finally {
                validationSpan.end();
            }

            // Gọi external service
            return paymentClient.charge(request);

        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### @Observed Annotation — Tự Động Tạo Span

```java
import io.micrometer.observation.annotation.Observed;

@Service
public class UserService {

    // Tự động tạo span "find-user-by-id" với metrics và tracing
    @Observed(
        name = "user.find",
        contextualName = "find-user-by-id",
        lowCardinalityKeyValues = {"service", "user"}
    )
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

> Cần thêm `ObservedAspect` bean:

```java
@Configuration
public class ObservabilityConfig {
    @Bean
    public ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
}
```

---

## 9. Structured Logging — Ghi Nhật Ký Có Cấu Trúc

### Cấu Hình Logback JSON

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- JSON appender cho production (dễ parse bởi ELK/Loki) -->
    <springProfile name="production,staging">
        <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <!-- Tự động thêm traceId và spanId từ MDC -->
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <!-- Custom fields -->
                <customFields>{"service":"${spring.application.name}","env":"${spring.profiles.active}"}</customFields>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>

    <!-- Text appender cho development -->
    <springProfile name="dev,local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <!-- %X{traceId} lấy từ MDC — Mapped Diagnostic Context -->
                <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{traceId},%X{spanId}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

### Dependency Logstash Encoder

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

### Log Output Có Cấu Trúc

```json
{
  "@timestamp": "2026-06-02T08:30:15.123Z",
  "level": "INFO",
  "logger_name": "com.example.service.OrderService",
  "message": "Order created successfully",
  "service": "order-service",
  "env": "production",
  "traceId": "abc123def456",
  "spanId": "001002003004",
  "userId": "user-789",
  "orderId": "order-456",
  "amount": 150000
}
```

### Log Correlation — Liên Kết Log

Spring Boot + Micrometer Tracing tự động inject `traceId` và `spanId` vào MDC (Mapped Diagnostic Context — Ngữ Cảnh Chẩn Đoán Ánh Xạ), giúp filter logs theo một request cụ thể:

```bash
# Tìm tất cả logs của một request trong Elasticsearch/Kibana
traceId: "abc123def456"

# Hoặc trong Grafana Loki
{service="order-service"} |= "abc123def456"
```

---

## 10. Loki + Promtail — Log Aggregation

Loki — Aggregation Platform — tích hợp tốt với Grafana, dùng cùng query language (LogQL).

```yaml
# docker-compose.yml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./loki/promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
```

---

## 11. Observability Stack Hoàn Chỉnh

```yaml
# docker-compose.observability.yml
version: '3.8'
services:
  # Spring Boot App
  app:
    image: myapp:latest
    environment:
      MANAGEMENT_ZIPKIN_TRACING_ENDPOINT: http://zipkin:9411/api/v2/spans
      MANAGEMENT_TRACING_SAMPLING_PROBABILITY: "1.0"
    depends_on: [prometheus, zipkin]

  # Metrics Pipeline
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./monitoring/grafana:/etc/grafana/provisioning

  # Tracing
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"

  # Logging
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
```

---

## 12. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Ba trụ cột Observability là gì?**

A: (1) **Metrics** — Chỉ Số: dữ liệu số đo trạng thái hệ thống theo thời gian (request rate, error rate, latency, CPU, memory) — dùng để alert và dashboard; (2) **Traces** — Dấu Vết: theo dõi luồng của một request qua nhiều services với traceId/spanId — dùng để debug latency và bottleneck; (3) **Logs** — Nhật Ký: sự kiện văn bản ghi lại những gì đã xảy ra — dùng để điều tra nguyên nhân lỗi cụ thể.

**Q: Micrometer là gì? Tại sao không dùng trực tiếp Prometheus client?**

A: Micrometer là vendor-neutral metrics facade (lớp trừu tượng trung lập với vendor), tương tự SLF4J cho logging. Viết code một lần với Micrometer API, có thể export sang Prometheus, Datadog, CloudWatch, InfluxDB, ... chỉ bằng cách thêm/đổi dependency. Nếu dùng Prometheus client trực tiếp, code bị khóa cứng vào Prometheus.

**Q: Distributed Tracing hoạt động thế nào?**

A: Khi request đến Service A, Micrometer Tracing tạo `traceId` (unique cho toàn bộ luồng) và `spanId` (unique cho mỗi service). Khi A gọi Service B, `traceId` được truyền qua HTTP header (`traceparent` theo W3C hoặc `X-B3-TraceId` theo B3). Mỗi service tạo `spanId` mới nhưng giữ nguyên `traceId`. Jaeger/Zipkin thu thập tất cả spans và hiển thị waterfall diagram.

**Q: Sampling rate bao nhiêu là phù hợp cho production?**

A: Phụ thuộc vào traffic. Với 1000 req/s, sampling 10% (0.1) đã đủ để phân tích. Với traffic thấp hơn 100 req/s, có thể dùng 100% (1.0). Cần cân bằng giữa data đầy đủ và overhead lưu trữ/network. Một số hệ thống dùng adaptive sampling — lấy mẫu thích nghi — tăng rate khi phát hiện anomalies.

---

## ✅ Checklist

- [ ] Thêm `micrometer-registry-prometheus` dependency
- [ ] Expose `/actuator/prometheus` endpoint
- [ ] Thêm common tags (application, environment) cho tất cả metrics
- [ ] Tạo business metrics với Counter/Timer/Gauge
- [ ] Cấu hình Prometheus scrape config đúng
- [ ] Viết alerting rules cho error rate và latency
- [ ] Setup Grafana với Spring Boot dashboard
- [ ] Thêm Micrometer Tracing dependency (Brave hoặc OTel)
- [ ] Cấu hình Zipkin/Jaeger endpoint và sampling rate
- [ ] Dùng `@Observed` hoặc manual spans cho business operations
- [ ] Cấu hình structured logging (JSON) với traceId trong log
- [ ] Test tracing end-to-end: một request → thấy đầy đủ trace trong UI
