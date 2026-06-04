# Service Discovery — Khám Phá Dịch Vụ

> Service Discovery (Khám Phá Dịch Vụ) cho phép các microservices tìm thấy nhau mà không cần hard-code địa chỉ IP hay port. Eureka (Netflix OSS), Consul (HashiCorp) và Kubernetes Service Discovery là các giải pháp phổ biến trong hệ sinh thái Spring Boot.

---

## 📋 Mục Lục

1. [Vấn Đề Service Discovery Giải Quyết](#vấn-đề-service-discovery-giải-quyết)
2. [Client-side vs Server-side Discovery](#client-side-vs-server-side-discovery)
3. [Netflix Eureka](#netflix-eureka)
4. [Consul](#consul)
5. [Spring Cloud LoadBalancer](#spring-cloud-loadbalancer)
6. [Kubernetes Service Discovery](#kubernetes-service-discovery)
7. [Health Check & Self-preservation](#health-check--self-preservation)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Service Discovery Giải Quyết

### Không Có Service Discovery

```
Order Service cần gọi Inventory Service:

Cách cũ — Hard-code địa chỉ:
http://192.168.1.100:8083/api/inventory

Vấn Đề:
❌ IP thay đổi khi service restart (đặc biệt trong Docker/K8s)
❌ Scale out: thêm instance mới → phải cập nhật config thủ công
❌ Dead instances không bị loại khỏi danh sách
❌ Không biết service nào đang healthy
```

### Có Service Discovery

```
1. Service Start → Đăng ký với Service Registry (Sổ Đăng Ký Dịch Vụ):
   "Tôi là inventory-service, đang chạy ở 192.168.1.100:8083"

2. Client cần gọi → Tra cứu Registry:
   "inventory-service ở đâu?"
   → Registry trả về: [192.168.1.100:8083, 192.168.1.101:8083]

3. Client chọn instance (load balancing) và gọi trực tiếp

4. Service Stop → Registry loại khỏi danh sách (qua heartbeat)

Lợi Ích:
✅ Dynamic topology (cấu trúc mạng động) — thêm/bớt instance tự động
✅ Health-aware routing — chỉ gọi healthy instances
✅ Load balancing built-in
✅ Không cần cập nhật config khi scale
```

---

## Client-side vs Server-side Discovery

### Client-side Discovery (Khám Phá Phía Máy Khách)

```
Client ──────► Service Registry (truy vấn)
   │              │
   │     ◄────────┘ (danh sách instances)
   │
   └──────────────► Service Instance (gọi trực tiếp sau khi load balance ở client)

Ví Dụ: Eureka + Spring Cloud LoadBalancer

Ưu điểm: Ít network hop (bước mạng), client kiểm soát load balancing
Nhược điểm: Client phải implement discovery logic
```

### Server-side Discovery (Khám Phá Phía Máy Chủ)

```
Client ──────► Load Balancer / API Gateway
                     │
                     ├──────────► Service Instance A
                     │              (Load Balancer tự tra Registry)
                     └──────────► Service Instance B

Ví Dụ: AWS ALB + ECS Service Discovery, Kubernetes Service

Ưu điểm: Client đơn giản, không cần biết gì về discovery
Nhược điểm: Thêm network hop, Load Balancer là potential bottleneck
```

---

## Netflix Eureka

### Eureka Server (Máy Chủ Eureka)

```xml
<!-- pom.xml của Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer   // Kích hoạt Eureka Server
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml cho Eureka Server
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    # Eureka Server không cần tự đăng ký với chính nó
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

  server:
    # Thời gian chờ trước khi evict (loại bỏ) instance không gửi heartbeat
    eviction-interval-timer-in-ms: 5000
    # Self-preservation mode (Chế Độ Tự Bảo Vệ) — bật để tránh mass eviction
    enable-self-preservation: true
    # Ngưỡng để bật self-preservation (85% instances healthy)
    renewal-percent-threshold: 0.85
```

### Eureka Client (Máy Khách Eureka)

```xml
<!-- pom.xml của mỗi microservice -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
// Không cần annotation @EnableEurekaClient từ Spring Cloud 2021+
// Chỉ cần có dependency + config là tự động kích hoạt
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```yaml
# application.yml của Order Service
spring:
  application:
    name: order-service    # Tên đăng ký — PHẢI unique và nhất quán

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    # Interval đồng bộ registry từ server (giây)
    registry-fetch-interval-seconds: 30
    # Thời gian initial fetch khi startup
    initial-instance-info-replication-interval-seconds: 40

  instance:
    # Thay vì IP, dùng hostname để định danh
    prefer-ip-address: true
    # Gửi heartbeat mỗi 10 giây (mặc định 30)
    lease-renewal-interval-in-seconds: 10
    # Sau bao lâu không có heartbeat thì evict (mặc định 90)
    lease-expiration-duration-in-seconds: 30
    # Custom instance ID (có thể override mặc định)
    instance-id: ${spring.application.name}:${server.port}:${random.value}
    # Metadata tùy chỉnh
    metadata-map:
      version: "2.0"
      environment: "production"
```

### Eureka High Availability (Tính Sẵn Sàng Cao)

```yaml
# Peer-to-peer replication — các Eureka Server sync với nhau

# Eureka Server 1 (chạy với profile: eureka1)
spring:
  profiles: eureka1
eureka:
  instance:
    hostname: eureka1.example.com
  client:
    service-url:
      defaultZone: http://eureka2.example.com:8761/eureka/

---
# Eureka Server 2
spring:
  profiles: eureka2
eureka:
  instance:
    hostname: eureka2.example.com
  client:
    service-url:
      defaultZone: http://eureka1.example.com:8761/eureka/

# Client trỏ đến cả hai để fallback
eureka:
  client:
    service-url:
      defaultZone: http://eureka1.example.com:8761/eureka/,http://eureka2.example.com:8761/eureka/
```

---

## Consul

### Consul vs Eureka — So Sánh

| Tính Năng | Eureka | Consul |
|-----------|--------|--------|
| **Health Checking** | Heartbeat (client push) | Agent-based (active pull) |
| **Multi-datacenter** | Không hỗ trợ sẵn | ✅ Built-in |
| **Key-Value Store** | ❌ Không | ✅ Có (dùng cho config) |
| **Service Mesh** | ❌ Không | ✅ Consul Connect |
| **DNS Interface** | ❌ Không | ✅ Có |
| **Maintenance Mode** | ❌ Không | ✅ Có |
| **CP/AP** | AP (ưu tiên Availability) | CP (ưu tiên Consistency) |

### Tích Hợp Spring Boot với Consul

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<!-- Consul cũng là Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

```yaml
# application.yml với Consul
spring:
  application:
    name: order-service
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        enabled: true
        register: true
        # Health check endpoint
        health-check-path: /actuator/health
        health-check-interval: 10s
        health-check-timeout: 5s
        # Tags để filter instances
        tags:
          - "version=2.0"
          - "environment=production"
      config:
        enabled: true
        # Đọc config từ Consul KV store
        prefix: config
        default-context: application
        format: YAML
```

### Consul Health Check

```java
// Consul Agent thực hiện health check, không phải application push heartbeat

// Service cần expose health endpoint
// Spring Boot Actuator tự cung cấp /actuator/health

// Custom health indicator
@Component
public class DatabaseHealthIndicator extends AbstractHealthIndicator {

    private final DataSource dataSource;

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(2)) {
                builder.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected");
            } else {
                builder.down().withDetail("database", "Connection invalid");
            }
        }
    }
}
```

---

## Spring Cloud LoadBalancer

### Tự Động Load Balancing Qua Service Discovery

```java
// Cách 1: Dùng @LoadBalanced RestTemplate
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced  // Annotation này enable service discovery cho RestTemplate
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderService {

    private final RestTemplate restTemplate;

    public InventoryResponse checkInventory(String productId) {
        // Dùng service name thay vì IP:port
        // Spring Cloud tự resolve "inventory-service" qua Eureka/Consul
        return restTemplate.getForObject(
            "http://inventory-service/api/v1/inventory/check?productId=" + productId,
            InventoryResponse.class
        );
    }
}

// Cách 2: WebClient (Reactive)
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class OrderService {

    private final WebClient.Builder webClientBuilder;

    public Mono<InventoryResponse> checkInventory(String productId) {
        return webClientBuilder.build()
            .get()
            .uri("http://inventory-service/api/v1/inventory/check", uri -> uri
                .queryParam("productId", productId)
                .build())
            .retrieve()
            .bodyToMono(InventoryResponse.class);
    }
}

// Cách 3: OpenFeign (Khuyến Nghị — Declarative HTTP Client)
@FeignClient(name = "inventory-service")  // name = Eureka service name
public interface InventoryClient {

    @GetMapping("/api/v1/inventory/check")
    InventoryResponse checkStock(
        @RequestParam("productId") String productId,
        @RequestParam("quantity") int quantity
    );
}
```

### Custom Load Balancing Strategy

```java
// Dùng Zone Affinity (Ưu Tiên Cùng Vùng) — gọi instance cùng AZ trước
@Configuration
@LoadBalancerClient(name = "inventory-service", configuration = InventoryLoadBalancerConfig.class)
public class InventoryLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);

        // Dùng Random thay vì RoundRobin cho inventory service
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}

// Custom instance filtering — chỉ gọi instances với tag "environment=production"
@Bean
public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
        .withDiscoveryClient()
        .withHealthChecks()          // Lọc unhealthy instances
        .withSameInstancePreference()  // Ưu tiên cùng instance (giảm cold start)
        .build(context);
}
```

---

## Kubernetes Service Discovery

### Service Discovery trong Kubernetes (K8s)

```yaml
# Kubernetes Service — DNS-based discovery
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service      # Tự tìm Pods có label này
  ports:
    - port: 80
      targetPort: 8081
  type: ClusterIP           # Chỉ internal access

---
# Spring Boot app trong K8s gọi order-service:
# http://order-service.production.svc.cluster.local/api/v1/orders
# hoặc ngắn hơn (cùng namespace): http://order-service/api/v1/orders
```

### Spring Cloud Kubernetes

```xml
<!-- Tích hợp Spring Boot với Kubernetes Service Discovery -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-all</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: order-service
  cloud:
    kubernetes:
      discovery:
        enabled: true
        # Chỉ discover services trong cùng namespace
        all-namespaces: false
      config:
        enabled: true
        sources:
          # Đọc từ ConfigMap tên "order-service-config"
          - namespace: production
            name: order-service-config
```

---

## Health Check & Self-preservation

### Self-preservation Mode (Chế Độ Tự Bảo Vệ)

```
Vấn Đề: Eureka Server mất kết nối mạng tạm thời
→ Không nhận heartbeat từ clients
→ Nếu evict tất cả → clients không còn ai để gọi

Self-preservation Mode giải quyết:
- Eureka theo dõi tỉ lệ heartbeat nhận được trong 15 phút
- Nếu tỉ lệ < 85% → BẬT self-preservation
- Khi self-preservation bật: KHÔNG evict instance nào
- Khi mạng ổn lại → Tắt self-preservation, tiếp tục evict bình thường

Cảnh Báo Trong Eureka Dashboard:
"EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT."
→ Đây là self-preservation đang hoạt động, không phải lỗi nghiêm trọng
```

### Health Check Endpoint

```java
// Expose thông tin health chi tiết
@Component
public class OrderServiceHealthIndicator extends AbstractHealthIndicator {

    private final OrderRepository orderRepository;
    private final KafkaProducerHealthCheck kafkaHealthCheck;

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        Map<String, Object> details = new LinkedHashMap<>();

        // Kiểm tra DB
        try {
            long count = orderRepository.count();
            details.put("database", "UP");
            details.put("totalOrders", count);
        } catch (Exception e) {
            builder.down().withDetail("database", "DOWN: " + e.getMessage());
            return;
        }

        // Kiểm tra Kafka
        boolean kafkaUp = kafkaHealthCheck.isHealthy();
        details.put("kafka", kafkaUp ? "UP" : "DOWN");

        if (kafkaUp) {
            builder.up().withDetails(details);
        } else {
            // Kafka down nhưng app vẫn có thể hoạt động partial
            builder.status("DEGRADED").withDetails(details);
        }
    }
}
```

```yaml
management:
  endpoint:
    health:
      show-details: always        # Hiển thị chi tiết health
      show-components: always
  health:
    diskspace:
      enabled: true
      threshold: 10MB             # Cảnh báo nếu disk < 10MB
    circuitbreakers:
      enabled: true               # Hiển thị trạng thái Circuit Breaker
```

---

## Câu Hỏi Phỏng Vấn

**Q: Client-side Discovery khác Server-side Discovery thế nào? Spring Boot dùng cái nào?**

> Client-side Discovery: client tự hỏi Service Registry (ví dụ Eureka) để lấy danh sách instances, rồi tự chọn instance qua load balancing. Spring Boot + Eureka + Spring Cloud LoadBalancer là client-side discovery. Server-side Discovery: client gọi Load Balancer/API Gateway, thành phần này tra Registry và forward request — Kubernetes Service và AWS ALB là ví dụ. Spring Boot cũng hỗ trợ server-side khi chạy trên Kubernetes.

**Q: Eureka Self-preservation Mode là gì? Có nên tắt đi không?**

> Self-preservation giúp Eureka tránh evict hàng loạt instances khi xảy ra network partition (phân mảnh mạng) — situation khi server không nhận heartbeat nhưng clients vẫn healthy. Nếu tắt (`enable-self-preservation: false`), Eureka sẽ evict instances ngay khi heartbeat bị miss — phù hợp cho môi trường dev/test nhưng nguy hiểm cho production vì có thể gây evict nhầm khi network chậm tạm thời.

**Q: Tại sao Consul được ưa thích hơn Eureka trong một số trường hợp?**

> Consul cung cấp: (1) Active health checking — agent chủ động kiểm tra thay vì đợi heartbeat; (2) Multi-datacenter support (hỗ trợ đa trung tâm dữ liệu) built-in; (3) KV store cho dynamic configuration; (4) Service Mesh với Consul Connect. Eureka đơn giản hơn và đủ dùng cho single-region deployments. Consul phù hợp khi cần cross-datacenter discovery hoặc muốn giảm số lượng infrastructure components.

**Q: Làm sao microservice biết địa chỉ của Eureka Server?**

> Được cấu hình qua `eureka.client.service-url.defaultZone` trong `application.yml`. Trong Kubernetes, thường dùng DNS name của Eureka Service. Để high availability, trỏ đến nhiều Eureka instances (comma-separated). Trong production, thường dùng environment variable để inject URL thay vì hard-code.

---

## ✅ Checklist

- [ ] Mỗi service có `spring.application.name` unique và nhất quán
- [ ] Eureka Server có ít nhất 2 instances cho HA (High Availability — Tính Sẵn Sàng Cao)
- [ ] Health check endpoint expose và được Eureka/Consul kiểm tra
- [ ] Heartbeat interval phù hợp (không quá thấp gây flood, không quá cao gây chậm detect lỗi)
- [ ] Self-preservation bật ở production
- [ ] Fallback khi Service Discovery không khả dụng

---

**Xem tiếp:** [6-event-driven.md](6-event-driven.md) — CQRS, Event Sourcing & Saga Pattern
