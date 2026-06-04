# API Gateway — Cổng API với Spring Cloud Gateway

> API Gateway (Cổng API) là điểm vào duy nhất của hệ thống microservices. Spring Cloud Gateway cung cấp routing (định tuyến), rate limiting (giới hạn tốc độ), authentication (xác thực), load balancing (cân bằng tải) và nhiều cross-cutting concerns khác.

---

## 📋 Mục Lục

1. [Tại Sao Cần API Gateway?](#tại-sao-cần-api-gateway)
2. [Spring Cloud Gateway Cơ Bản](#spring-cloud-gateway-cơ-bản)
3. [Routing Configuration](#routing-configuration)
4. [Built-in Filters](#built-in-filters)
5. [Custom Filters](#custom-filters)
6. [Rate Limiting](#rate-limiting)
7. [Authentication & Authorization](#authentication--authorization)
8. [Load Balancing](#load-balancing)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần API Gateway?

### Không Có API Gateway

```
Client ──────► Order Service :8081
Client ──────► Payment Service :8082     ← Client phải biết địa chỉ từng service
Client ──────► Inventory Service :8083   ← Authentication phải implement ở mỗi service
Client ──────► User Service :8084        ← CORS phải config ở mỗi service

Vấn Đề:
❌ Client coupled với internal topology (cấu trúc nội bộ)
❌ Cross-cutting concerns (CORS, Auth, Logging) bị lặp
❌ Không thể rate limit từng client
❌ Khó thay đổi internal services mà không ảnh hưởng client
```

### Có API Gateway

```
                   ┌──────────────────────────────────┐
                   │         API GATEWAY              │
Client ───────────►│  • Authentication (Xác Thực)     │──► Order Service
                   │  • Rate Limiting (Giới Hạn Tốc Độ│──► Payment Service
                   │  • Routing (Định Tuyến)           │──► Inventory Service
                   │  • Load Balancing (Cân Bằng Tải)  │──► User Service
                   │  • SSL Termination (Kết Thúc SSL) │
                   │  • Request/Response Transform     │
                   └──────────────────────────────────┘

Ưu Điểm:
✅ Single entry point — client chỉ cần biết gateway URL
✅ Cross-cutting concerns tập trung
✅ Loose coupling giữa client và services
✅ Dễ implement A/B testing, canary deployment
```

### API Gateway vs Load Balancer (Cân Bằng Tải)

| Tính Năng | API Gateway | Load Balancer |
|-----------|-------------|---------------|
| Routing | Layer 7 (HTTP path, header) | Layer 4 (IP, TCP) hoặc Layer 7 |
| Authentication | ✅ Có | ❌ Thường không |
| Rate Limiting | ✅ Có | ❌ Hạn chế |
| Request Transform | ✅ Có | ❌ Không |
| Circuit Breaking | ✅ Có | ❌ Không |
| Protocol Translation | ✅ HTTP ↔ gRPC | ❌ Không |

---

## Spring Cloud Gateway Cơ Bản

### Dependency

```xml
<!-- Spring Cloud Gateway (reactive — dựa trên WebFlux) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- Eureka Client — để dùng lb:// scheme -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!-- Redis — cho Rate Limiting -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### Kiến Trúc Spring Cloud Gateway

```
Incoming Request
       │
       ▼
  Gateway Handler Mapping (Xác Định Route)
       │
       ▼
  Gateway Web Handler
       │
       ▼
  Filter Chain (Chuỗi Bộ Lọc)
  ┌─────────────────────┐
  │ Pre-filters          │ ← Logging, Auth, Rate Limit
  │ (Trước khi forward) │
  └──────────┬──────────┘
             │
             ▼
      Downstream Service (Dịch Vụ Hạ Nguồn)
             │
  ┌──────────▼──────────┐
  │ Post-filters         │ ← Response modification, Logging
  │ (Sau khi response)  │
  └─────────────────────┘
       │
       ▼
  Response to Client
```

---

## Routing Configuration

### YAML Configuration (Cấu Hình YAML)

```yaml
# application.yml cho API Gateway
spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      routes:
        # Route 1: Order Service
        - id: order-service
          uri: lb://order-service          # lb:// = Load Balanced (qua Eureka)
          predicates:
            - Path=/api/v1/orders/**       # Match URL path
          filters:
            - StripPrefix=0               # Không bỏ prefix
            - AddRequestHeader=X-Gateway-Source, api-gateway

        # Route 2: Product Service — với path rewrite
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/v1/products/**
            - Method=GET,POST             # Chỉ GET và POST
          filters:
            - RewritePath=/api/v1/products/(?<segment>.*), /internal/products/${segment}

        # Route 3: User Service — với header predicate
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/v1/users/**
            - Header=X-Internal-Call, true  # Chỉ forward nếu có header này

        # Route 4: Payment Service — versioned routing
        - id: payment-service-v2
          uri: lb://payment-service-v2
          predicates:
            - Path=/api/v2/payments/**
            - Weight=payment-group, 80     # 80% traffic → v2

        - id: payment-service-v1
          uri: lb://payment-service-v1
          predicates:
            - Path=/api/v1/payments/**
            - Weight=payment-group, 20     # 20% traffic → v1 (canary)

      # Global CORS Configuration
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins:
              - "https://app.example.com"
              - "https://admin.example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders:
              - Authorization
              - Content-Type
              - X-Requested-With
            maxAge: 3600
```

### Java DSL Configuration (Cấu Hình Bằng Code Java)

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()

            // Order Service Route
            .route("order-service", r -> r
                .path("/api/v1/orders/**")
                .and()
                .method(HttpMethod.GET, HttpMethod.POST, HttpMethod.PUT, HttpMethod.DELETE)
                .filters(f -> f
                    .addRequestHeader("X-Gateway-Source", "api-gateway")
                    .addResponseHeader("X-Response-Time", LocalDateTime.now().toString())
                    .circuitBreaker(config -> config
                        .setName("order-service-cb")
                        .setFallbackUri("forward:/fallback/orders")
                    )
                )
                .uri("lb://order-service")
            )

            // Static asset route — trực tiếp không qua services
            .route("static-files", r -> r
                .path("/static/**")
                .uri("https://cdn.example.com")
            )

            .build();
    }
}
```

---

## Built-in Filters

### Commonly Used Gateway Filters (Các Filter Hay Dùng)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: example
          uri: lb://example-service
          filters:
            # 1. AddRequestHeader — thêm header vào request
            - AddRequestHeader=X-Request-Id, #{T(java.util.UUID).randomUUID()}

            # 2. AddResponseHeader — thêm header vào response
            - AddResponseHeader=X-Cache-Status, MISS

            # 3. RemoveRequestHeader — xóa header (ví dụ: bảo mật)
            - RemoveRequestHeader=Cookie

            # 4. RewritePath — thay đổi path
            - RewritePath=/api/v1/(?<path>.*), /${path}

            # 5. StripPrefix — bỏ prefix
            - StripPrefix=2   # /api/v1/orders → /orders

            # 6. PrefixPath — thêm prefix
            - PrefixPath=/internal

            # 7. RequestRateLimiter — giới hạn tốc độ (xem phần Rate Limiting)
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"

            # 8. Retry — thử lại khi lỗi
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 10ms
                  maxBackoff: 50ms
                  factor: 2

            # 9. CircuitBreaker — ngắt mạch
            - name: CircuitBreaker
              args:
                name: myCircuitBreaker
                fallbackUri: forward:/fallback

            # 10. RequestSize — giới hạn kích thước request
            - name: RequestSize
              args:
                maxSize: 5MB
```

---

## Custom Filters

### Global Filter (Filter Toàn Cục) — Logging

```java
@Component
@Slf4j
public class LoggingGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String requestId = UUID.randomUUID().toString();
        String path = request.getURI().getPath();
        HttpMethod method = request.getMethod();
        long startTime = System.currentTimeMillis();

        // Pre-processing (Tiền Xử Lý)
        log.info("Gateway Request: [{} {}] requestId={}", method, path, requestId);

        // Thêm request ID vào header để trace qua các services
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-Request-Id", requestId)
            .header("X-Gateway-Time", String.valueOf(startTime))
            .build();

        return chain.filter(exchange.mutate().request(modifiedRequest).build())
            .then(Mono.fromRunnable(() -> {
                // Post-processing (Hậu Xử Lý)
                long duration = System.currentTimeMillis() - startTime;
                int statusCode = exchange.getResponse().getStatusCode().value();
                log.info("Gateway Response: [{} {}] status={} duration={}ms requestId={}",
                    method, path, statusCode, duration, requestId);
            }));
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 1; // Chạy gần cuối nhất
    }
}
```

### Route-specific Filter (Filter Theo Route) — Request Transformation

```java
@Component
public class RequestTransformGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestTransformGatewayFilterFactory.Config> {

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // Đọc và transform request body
            return DataBufferUtils.join(request.getBody())
                .flatMap(dataBuffer -> {
                    byte[] bytes = new byte[dataBuffer.readableByteCount()];
                    dataBuffer.read(bytes);
                    DataBufferUtils.release(dataBuffer);

                    String body = new String(bytes, StandardCharsets.UTF_8);
                    // Transform body (ví dụ: thêm timestamp)
                    String transformedBody = addTimestamp(body);

                    ServerHttpRequest modifiedRequest = request.mutate()
                        .header(HttpHeaders.CONTENT_LENGTH,
                            String.valueOf(transformedBody.getBytes().length))
                        .build();

                    return chain.filter(exchange.mutate().request(modifiedRequest).build());
                });
        };
    }

    public static class Config {}
}
```

---

## Rate Limiting

### Token Bucket Algorithm với Redis

```
Token Bucket (Thuật Toán Xô Token):

[Token Bucket: capacity=20]
│ Tokens được thêm vào với tốc độ: 10 tokens/giây (replenishRate)
│ Tối đa 20 tokens (burstCapacity — tải đột biến)
│
├── Request đến → Lấy 1 token → Nếu có token: cho qua ✅
│                              Nếu không có token: từ chối 429 ❌
│
└── Tokens tích lũy khi không có request
```

### Cấu Hình Rate Limiting

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api-with-rate-limit
          uri: lb://backend-service
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                # Số token được thêm mỗi giây
                redis-rate-limiter.replenishRate: 10
                # Tối đa tokens tích lũy (xử lý burst traffic)
                redis-rate-limiter.burstCapacity: 20
                # Mỗi request tốn bao nhiêu token
                redis-rate-limiter.requestedTokens: 1
                # Bean xác định key để rate limit (theo user/IP/API key)
                key-resolver: "#{@userKeyResolver}"

  data:
    redis:
      host: localhost
      port: 6379
```

```java
@Configuration
public class RateLimitConfig {

    // Rate limit theo User ID (từ JWT)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id");
            return Mono.just(userId != null ? userId : "anonymous");
        };
    }

    // Rate limit theo IP Address
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                .getRemoteAddress()
                .getAddress()
                .getHostAddress()
        );
    }

    // Rate limit theo API Key
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                .getHeaders()
                .getFirst("X-API-Key")
        );
    }
}
```

### Different Rate Limits Per Route

```yaml
routes:
  # Public API — giới hạn 5 req/giây
  - id: public-api
    uri: lb://backend-service
    predicates:
      - Path=/api/public/**
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 5
          redis-rate-limiter.burstCapacity: 10
          key-resolver: "#{@ipKeyResolver}"

  # Authenticated API — giới hạn 100 req/giây per user
  - id: authenticated-api
    uri: lb://backend-service
    predicates:
      - Path=/api/v1/**
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 100
          redis-rate-limiter.burstCapacity: 200
          key-resolver: "#{@userKeyResolver}"
```

---

## Authentication & Authorization

### JWT Validation tại Gateway

```java
@Component
@Slf4j
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    private final JwtTokenValidator jwtValidator;

    // Các path không cần auth (public endpoints)
    private static final List<String> PUBLIC_PATHS = List.of(
        "/api/v1/auth/login",
        "/api/v1/auth/register",
        "/api/v1/auth/refresh",
        "/api/v1/products",         // GET products là public
        "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // Bỏ qua public paths
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }

        // Lấy JWT từ Authorization header
        String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorizedResponse(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        // Validate JWT
        return jwtValidator.validateToken(token)
            .flatMap(claims -> {
                // Truyền user info xuống downstream service qua header
                ServerHttpRequest modifiedRequest = request.mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Roles", String.join(",", claims.getRoles()))
                    .header("X-User-Email", claims.getEmail())
                    .build();
                return chain.filter(exchange.mutate().request(modifiedRequest).build());
            })
            .onErrorResume(e -> {
                log.warn("JWT validation failed: {}", e.getMessage());
                return unauthorizedResponse(exchange, "Invalid or expired token");
            });
    }

    private Mono<Void> unauthorizedResponse(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        String body = """
            {"error": "UNAUTHORIZED", "message": "%s"}
            """.formatted(message);
        DataBuffer buffer = response.bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));
        return response.writeWith(Mono.just(buffer));
    }

    private boolean isPublicPath(String path) {
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }

    @Override
    public int getOrder() {
        return -100; // Chạy trước tất cả các filters khác
    }
}
```

---

## Load Balancing

### Spring Cloud LoadBalancer

```java
// Thay thế Ribbon (deprecated) trong Spring Cloud 2021+
// Tự động load balance qua các instances đăng ký với Eureka

// Cấu hình load balancing strategy (chiến lược)
@Configuration
public class LoadBalancerConfig {

    // RoundRobin (Luân Phiên) — mặc định
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

### Health-aware Load Balancing

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        # Chỉ gửi traffic đến instances healthy
        enabled: true
        path:
          default: /actuator/health
        interval: 15s
      configurations: health-check  # Dùng HealthCheckServiceInstanceListSupplier
```

---

## Fallback Endpoint

```java
// Xử lý khi downstream service không khả dụng
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/orders")
    public ResponseEntity<Map<String, Object>> ordersFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", "Order service đang tạm thời không khả dụng. Vui lòng thử lại sau.",
                "timestamp", LocalDateTime.now().toString()
            ));
    }

    @GetMapping("/payments")
    public ResponseEntity<Map<String, Object>> paymentsFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "SERVICE_UNAVAILABLE",
                "message", "Payment service đang bảo trì. Vui lòng liên hệ support.",
                "supportContact", "support@example.com"
            ));
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: API Gateway khác BFF (Backend For Frontend — Hậu Phần Dành Cho Giao Diện) như thế nào?**

> API Gateway là một điểm vào chung cho nhiều clients (web, mobile, third-party). BFF là một Gateway được tùy chỉnh riêng cho một loại client cụ thể — ví dụ Mobile BFF chỉ expose những endpoints và data format phù hợp với mobile, Web BFF phục vụ SPA. BFF pattern giúp tránh over-fetching và under-fetching cho từng loại client.

**Q: Spring Cloud Gateway dựa trên gì, khác Zuul ở điểm nào?**

> Spring Cloud Gateway dựa trên Spring WebFlux (reactive, non-blocking), trong khi Zuul 1.x dựa trên Servlet (blocking). Gateway xử lý nhiều concurrent connections hiệu quả hơn với ít threads hơn. Spring Cloud Gateway cũng có Route Predicate factories phong phú hơn và tích hợp tốt với Spring Cloud ecosystem hiện đại.

**Q: Rate Limiting hoạt động thế nào với nhiều instances Gateway?**

> Dùng Redis làm shared store cho token bucket. Mọi Gateway instance đều đọc/ghi vào Redis — đảm bảo rate limit được áp dụng globally (toàn cục) dù có nhiều Gateway instances. Nếu không dùng Redis, mỗi instance sẽ có counter riêng → rate limit không chính xác.

**Q: Làm sao handle authentication hiệu quả ở Gateway?**

> Hai approach: (1) Validate JWT ở Gateway — nhanh, không cần gọi Auth Service cho mỗi request, nhưng không biết token bị revoke nhanh; (2) Introspect tại Auth Service — luôn up-to-date nhưng thêm latency cho mỗi request. Thực tế hay dùng hybrid: validate JWT signature ở Gateway, check revocation cho sensitive operations. Dùng short-lived access tokens (15 phút) giảm rủi ro khi không introspect.

---

## ✅ Checklist

- [ ] JWT validation thực hiện ở Gateway — không lặp ở từng service
- [ ] Rate Limiting cấu hình với Redis để hoạt động đúng với nhiều Gateway instances
- [ ] Fallback endpoints cho mọi downstream service
- [ ] Circuit Breaker tích hợp trên mỗi route
- [ ] Logging/tracing với Request ID qua toàn bộ chain
- [ ] CORS cấu hình tập trung ở Gateway
- [ ] Health check endpoint của Gateway expose cho load balancer

---

**Xem tiếp:** [5-service-discovery.md](5-service-discovery.md) — Eureka & Consul — Khám Phá Dịch Vụ Tự Động
