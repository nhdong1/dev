# Spring Data Redis — Caching và In-Memory Data Store

> Redis (Remote Dictionary Server — Máy Chủ Từ Điển Từ Xa) là in-memory data store (kho dữ liệu trong bộ nhớ)
> nhanh nhất hiện có. Spring Boot tích hợp Redis qua `spring-boot-starter-data-redis`,
> hỗ trợ caching, session store, message broker, và distributed locking.

---

## 📋 Mục Tiêu

- [ ] Cấu hình **Redis** với Spring Boot — Lettuce client vs Jedis client
- [ ] Dùng **`@Cacheable`**, **`@CacheEvict`**, **`@CachePut`** cho cache layer
- [ ] Thao tác trực tiếp với **`RedisTemplate`** — String, Hash, List, Set, ZSet
- [ ] Dùng **`RedisRepository`** lưu entity trong Redis
- [ ] Cấu hình **TTL** (Time-to-Live — Thời Gian Sống) và cache serialization
- [ ] Implement **Pub/Sub** (Publish/Subscribe — Phát/Đăng Ký) với Redis

---

## 1. Cấu Hình Redis

### Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- JSON serialization -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}        # Để trống nếu không có password
      timeout: 2000ms                     # Connection timeout
      lettuce:
        pool:
          max-active: 20                  # Số connection tối đa trong pool
          max-idle: 10                    # Số connection idle tối đa
          min-idle: 5                     # Số connection idle tối thiểu
          max-wait: -1ms                  # -1 = đợi vô hạn khi pool đầy

  cache:
    type: redis
    redis:
      time-to-live: 3600000ms             # TTL mặc định: 1 giờ
      cache-null-values: false            # Không cache kết quả null
      use-key-prefix: true
      key-prefix: "myapp:"               # Prefix cho tất cả cache keys
```

### Redis Configuration Bean

```java
@Configuration
@EnableCaching  // Bật Spring Cache abstraction
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key serializer — dùng String để key dễ đọc trong Redis CLI
        StringRedisSerializer stringSerializer = new StringRedisSerializer();
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);

        // Value serializer — dùng JSON để value dễ đọc và cross-language
        Jackson2JsonRedisSerializer<Object> jsonSerializer =
            new Jackson2JsonRedisSerializer<>(Object.class);

        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // Lưu class type trong JSON → cần thiết để deserialize đúng type
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        jsonSerializer.setObjectMapper(mapper);

        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // Cấu hình riêng cho từng cache
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();

        // Cache products — TTL 10 phút
        cacheConfigs.put("products", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues());

        // Cache categories — TTL 1 giờ (ít thay đổi)
        cacheConfigs.put("categories", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)));

        // Cache user sessions — TTL 30 phút
        cacheConfigs.put("user-sessions", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(30))    // Mặc định: 30 phút
                    .disableCachingNullValues()
                    .serializeKeysWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(
                            new StringRedisSerializer()))
                    .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(
                            new GenericJackson2JsonRedisSerializer()))
            )
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## 2. Spring Cache Abstraction — @Cacheable, @CacheEvict, @CachePut

### @Cacheable — Lưu Kết Quả Vào Cache

```java
@Service
public class ProductService {

    // Lần đầu gọi → thực thi method, lưu kết quả vào cache
    // Lần sau gọi với cùng productId → trả về từ cache, KHÔNG gọi DB
    @Cacheable(
        value = "products",                    // Cache name (tên nhóm cache)
        key = "#productId",                    // Cache key — SpEL expression
        condition = "#productId > 0",          // Chỉ cache khi condition đúng
        unless = "#result == null"             // Không cache khi result là null
    )
    public ProductDto findById(Long productId) {
        return productRepository.findById(productId)
            .map(mapper::toDto)
            .orElse(null);
    }

    // Cache key tùy chỉnh phức tạp
    @Cacheable(value = "products",
               key = "'category:' + #categoryId + ':page:' + #pageable.pageNumber")
    public Page<ProductDto> findByCategory(Long categoryId, Pageable pageable) {
        return productRepository.findByCategoryId(categoryId, pageable)
                                .map(mapper::toDto);
    }

    // Dùng custom KeyGenerator thay vì SpEL
    @Cacheable(value = "products", keyGenerator = "productKeyGenerator")
    public List<ProductDto> findAll() {
        return productRepository.findAll().stream()
                                .map(mapper::toDto)
                                .toList();
    }
}
```

### @CacheEvict — Xóa Cache Khi Dữ Liệu Thay Đổi

```java
@Service
public class ProductService {

    // Xóa entry cụ thể sau khi update
    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public ProductDto update(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId).orElseThrow();
        mapper.updateEntity(request, product);
        return mapper.toDto(productRepository.save(product));
    }

    // Xóa toàn bộ cache của "products"
    @CacheEvict(value = "products", allEntries = true)
    @Transactional
    public void deleteAll() {
        productRepository.deleteAll();
    }

    // Xóa nhiều cache cùng lúc
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#productId"),
        @CacheEvict(value = "product-listings", allEntries = true),
        @CacheEvict(value = "categories", allEntries = true)
    })
    @Transactional
    public void delete(Long productId) {
        productRepository.deleteById(productId);
    }

    // beforeInvocation = true: Xóa cache TRƯỚC khi method chạy
    // (mặc định: sau khi method thành công)
    @CacheEvict(value = "products", key = "#productId", beforeInvocation = true)
    public void invalidateProductCache(Long productId) {
        // Method này chỉ để clear cache — không cần làm gì khác
    }
}
```

### @CachePut — Cập Nhật Cache

```java
// @CachePut: Luôn thực thi method VÀ cập nhật cache
// Khác @Cacheable: @Cacheable bỏ qua method nếu cache hit
@CachePut(value = "products", key = "#result.id")
@Transactional
public ProductDto create(ProductCreateRequest request) {
    Product product = mapper.toEntity(request);
    product = productRepository.save(product);
    return mapper.toDto(product);  // Kết quả được lưu vào cache
}
```

### @Caching — Kết Hợp Nhiều Annotations

```java
// Xóa cache cũ, cập nhật cache mới trong 1 method
@Caching(
    put = {
        @CachePut(value = "products", key = "#result.id")
    },
    evict = {
        @CacheEvict(value = "product-listings", allEntries = true),
        @CacheEvict(value = "categories", key = "#result.categoryId")
    }
)
@Transactional
public ProductDto update(Long id, ProductUpdateRequest request) {
    Product product = productRepository.findById(id).orElseThrow();
    mapper.updateEntity(request, product);
    return mapper.toDto(productRepository.save(product));
}
```

---

## 3. RedisTemplate — Thao Tác Trực Tiếp

Khi cần kiểm soát chi tiết hơn `@Cacheable`, dùng `RedisTemplate` trực tiếp.

### String Operations — Chuỗi Đơn Giản

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    // Lưu token vào blacklist với TTL
    public void blacklistToken(String token, Duration expiry) {
        String key = "blacklist:token:" + token;
        redisTemplate.opsForValue().set(key, "revoked", expiry);
    }

    // Kiểm tra token có trong blacklist không
    public boolean isBlacklisted(String token) {
        String key = "blacklist:token:" + token;
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

    // Đếm số lần request (rate limiting)
    public long incrementRequestCount(String clientId, Duration window) {
        String key = "rate:limit:" + clientId;
        Long count = redisTemplate.opsForValue().increment(key);
        // Đặt TTL khi key mới được tạo
        if (count != null && count == 1) {
            redisTemplate.expire(key, window);
        }
        return count != null ? count : 0;
    }
}
```

### Hash Operations — Cấu Trúc Key-Value Lồng Nhau

```java
@Service
@RequiredArgsConstructor
public class UserSessionService {

    private final RedisTemplate<String, Object> redisTemplate;

    // Lưu session dưới dạng Hash
    public void saveSession(String sessionId, Map<String, Object> sessionData) {
        String key = "session:" + sessionId;
        HashOperations<String, String, Object> hashOps = redisTemplate.opsForHash();

        hashOps.putAll(key, sessionData);
        redisTemplate.expire(key, Duration.ofMinutes(30));
    }

    // Lấy một field từ session
    public Object getSessionField(String sessionId, String field) {
        return redisTemplate.opsForHash().get("session:" + sessionId, field);
    }

    // Cập nhật một field mà không cần lấy toàn bộ session
    public void updateLastActivity(String sessionId) {
        redisTemplate.opsForHash().put(
            "session:" + sessionId,
            "lastActivity",
            Instant.now().toString()
        );
        redisTemplate.expire("session:" + sessionId, Duration.ofMinutes(30));
    }

    // Xóa session
    public void deleteSession(String sessionId) {
        redisTemplate.delete("session:" + sessionId);
    }
}
```

### List Operations — Hàng Đợi và Stack

```java
@Service
@RequiredArgsConstructor
public class NotificationQueueService {

    private final RedisTemplate<String, String> redisTemplate;

    private static final String QUEUE_KEY = "notifications:queue";

    // Push vào cuối danh sách (queue behavior — hàng đợi FIFO)
    public void enqueue(String notification) {
        redisTemplate.opsForList().rightPush(QUEUE_KEY, notification);
    }

    // Pop từ đầu danh sách (queue)
    public String dequeue() {
        return redisTemplate.opsForList().leftPop(QUEUE_KEY);
    }

    // Blocking pop — đợi tối đa 5 giây nếu queue rỗng
    public String blockingDequeue() {
        return redisTemplate.opsForList().leftPop(QUEUE_KEY, Duration.ofSeconds(5));
    }

    // Xem kích thước queue
    public long queueSize() {
        Long size = redisTemplate.opsForList().size(QUEUE_KEY);
        return size != null ? size : 0;
    }
}
```

### Set Operations — Tập Hợp Không Trùng Lặp

```java
@Service
@RequiredArgsConstructor
public class OnlineUserService {

    private final RedisTemplate<String, String> redisTemplate;

    private static final String ONLINE_KEY = "users:online";

    // Thêm user vào tập online
    public void userOnline(String userId) {
        redisTemplate.opsForSet().add(ONLINE_KEY, userId);
        redisTemplate.expire(ONLINE_KEY, Duration.ofHours(1));
    }

    // Xóa user khỏi tập online
    public void userOffline(String userId) {
        redisTemplate.opsForSet().remove(ONLINE_KEY, userId);
    }

    // Đếm số user online
    public long countOnlineUsers() {
        Long count = redisTemplate.opsForSet().size(ONLINE_KEY);
        return count != null ? count : 0;
    }

    // Kiểm tra user có online không
    public boolean isOnline(String userId) {
        return Boolean.TRUE.equals(redisTemplate.opsForSet().isMember(ONLINE_KEY, userId));
    }
}
```

### ZSet (Sorted Set) — Bảng Xếp Hạng

```java
@Service
@RequiredArgsConstructor
public class LeaderboardService {

    private final RedisTemplate<String, String> redisTemplate;

    private static final String LEADERBOARD_KEY = "game:leaderboard";

    // Thêm/cập nhật điểm số
    public void updateScore(String userId, double score) {
        redisTemplate.opsForZSet().add(LEADERBOARD_KEY, userId, score);
    }

    // Tăng điểm
    public double incrementScore(String userId, double delta) {
        Double newScore = redisTemplate.opsForZSet().incrementScore(LEADERBOARD_KEY, userId, delta);
        return newScore != null ? newScore : 0;
    }

    // Lấy top 10 người chơi (điểm cao nhất trước)
    public Set<String> getTopPlayers(int count) {
        return redisTemplate.opsForZSet()
            .reverseRange(LEADERBOARD_KEY, 0, count - 1);
    }

    // Lấy xếp hạng của user (0-based, thấp nhất trước)
    public Long getUserRank(String userId) {
        Long rank = redisTemplate.opsForZSet().reverseRank(LEADERBOARD_KEY, userId);
        return rank != null ? rank + 1 : null; // Chuyển về 1-based
    }

    // Lấy top 10 kèm theo điểm số
    public Set<ZSetOperations.TypedTuple<String>> getTopPlayersWithScores(int count) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores(LEADERBOARD_KEY, 0, count - 1);
    }
}
```

---

## 4. RedisRepository — Lưu Entity Trong Redis

```java
// Cần @EnableRedisRepositories trong config
@Configuration
@EnableRedisRepositories
public class RedisConfig { ... }

// Entity lưu trong Redis
@RedisHash(value = "carts",  // Key prefix: "carts"
           timeToLive = 1800) // TTL: 1800 giây = 30 phút
public class ShoppingCart {

    @Id
    private String sessionId;  // Key: "carts:{sessionId}"

    @Indexed  // Tạo index để tìm kiếm theo field này
    private Long userId;

    private List<CartItem> items = new ArrayList<>();
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Sub-object trong Redis entity
public class CartItem {
    private Long productId;
    private String productName;
    private int quantity;
    private BigDecimal price;
}

// Repository — tương tự JpaRepository nhưng cho Redis
public interface ShoppingCartRepository extends CrudRepository<ShoppingCart, String> {

    // Tìm theo @Indexed field
    Optional<ShoppingCart> findByUserId(Long userId);

    // Xóa cart cũ theo userId
    void deleteByUserId(Long userId);
}

// Service sử dụng
@Service
@RequiredArgsConstructor
public class CartService {

    private final ShoppingCartRepository cartRepository;

    public ShoppingCart getOrCreateCart(String sessionId, Long userId) {
        return cartRepository.findById(sessionId)
            .orElseGet(() -> {
                ShoppingCart cart = new ShoppingCart();
                cart.setSessionId(sessionId);
                cart.setUserId(userId);
                cart.setCreatedAt(LocalDateTime.now());
                return cartRepository.save(cart);
            });
    }

    public ShoppingCart addItem(String sessionId, CartItem item) {
        ShoppingCart cart = cartRepository.findById(sessionId)
            .orElseThrow(() -> new CartNotFoundException(sessionId));

        cart.getItems().add(item);
        cart.setUpdatedAt(LocalDateTime.now());
        return cartRepository.save(cart);
    }
}
```

---

## 5. Pub/Sub — Phát Và Đăng Ký Thông Điệp

```java
// Publisher (Người Phát Thông Điệp)
@Service
@RequiredArgsConstructor
public class EventPublisher {

    private final RedisTemplate<String, String> redisTemplate;

    public void publishOrderEvent(String orderId, String eventType) {
        String channel = "order-events";
        String message = String.format("{\"orderId\":\"%s\",\"event\":\"%s\"}", orderId, eventType);
        redisTemplate.convertAndSend(channel, message);
    }
}

// Subscriber (Người Đăng Ký Lắng Nghe)
@Component
public class OrderEventSubscriber implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String body = new String(message.getBody());
        log.info("Received message on channel {}: {}", channel, body);
        // Xử lý message...
    }
}

// Cấu hình MessageListenerContainer
@Bean
public RedisMessageListenerContainer messageListenerContainer(
    RedisConnectionFactory factory,
    OrderEventSubscriber subscriber
) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(factory);

    // Đăng ký subscriber lắng nghe channel
    container.addMessageListener(subscriber, new ChannelTopic("order-events"));

    // Lắng nghe theo pattern (wildcard)
    container.addMessageListener(subscriber, new PatternTopic("order-*"));

    return container;
}
```

---

## 6. Cache-Aside Pattern — Mẫu Cache Bên Cạnh

```java
// Cache-Aside: Application tự quản lý cache — không phụ thuộc @Cacheable
@Service
@RequiredArgsConstructor
public class ProductCacheService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;

    private static final String KEY_PREFIX = "product:";
    private static final Duration TTL = Duration.ofMinutes(30);

    public ProductDto getProduct(Long productId) {
        String key = KEY_PREFIX + productId;

        // 1. Kiểm tra cache
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return objectMapper.convertValue(cached, ProductDto.class);
        }

        // 2. Cache miss → đọc từ DB
        ProductDto product = productRepository.findById(productId)
            .map(mapper::toDto)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // 3. Lưu vào cache
        redisTemplate.opsForValue().set(key, product, TTL);

        return product;
    }

    public void invalidateProduct(Long productId) {
        redisTemplate.delete(KEY_PREFIX + productId);
    }
}
```

---

## 7. Distributed Lock — Khóa Phân Tán

```java
// Redis-based distributed lock để đồng bộ hóa giữa nhiều instance
@Service
@RequiredArgsConstructor
public class DistributedLockService {

    private final RedisTemplate<String, String> redisTemplate;

    public boolean acquireLock(String lockKey, String lockValue, Duration timeout) {
        // SET key value NX PX milliseconds — atomic operation
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, timeout);
        return Boolean.TRUE.equals(acquired);
    }

    public void releaseLock(String lockKey, String expectedValue) {
        // Chỉ release nếu value khớp — tránh release lock của thread khác
        String currentValue = (String) redisTemplate.opsForValue().get(lockKey);
        if (expectedValue.equals(currentValue)) {
            redisTemplate.delete(lockKey);
        }
    }
}

// Sử dụng trong service
@Service
public class InventoryService {

    @Autowired
    private DistributedLockService lockService;

    public void decreaseStock(Long productId, int quantity) {
        String lockKey = "lock:product:" + productId;
        String lockValue = UUID.randomUUID().toString();

        if (!lockService.acquireLock(lockKey, lockValue, Duration.ofSeconds(10))) {
            throw new ConcurrentModificationException("Product " + productId + " is being updated");
        }

        try {
            // Critical section — chỉ một instance được thực thi tại một thời điểm
            Product product = productRepository.findById(productId).orElseThrow();
            product.setStock(product.getStock() - quantity);
            productRepository.save(product);
        } finally {
            lockService.releaseLock(lockKey, lockValue);
        }
    }
}
```

---

## 8. Rate Limiting — Giới Hạn Tần Suất Request

```java
// Token Bucket Algorithm với Redis
@Component
@RequiredArgsConstructor
public class RateLimiter {

    private final RedisTemplate<String, String> redisTemplate;

    // Giới hạn: maxRequests mỗi windowSeconds giây
    public boolean isAllowed(String clientId, int maxRequests, int windowSeconds) {
        String key = "rate:" + clientId;

        // Sliding window với sorted set
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000L);

        redisTemplate.multi(); // Bắt đầu pipeline

        ZSetOperations<String, String> zOps = redisTemplate.opsForZSet();

        // Xóa requests cũ hơn window
        zOps.removeRangeByScore(key, 0, windowStart);

        // Đếm requests trong window
        Long requestCount = zOps.zCard(key);

        if (requestCount != null && requestCount < maxRequests) {
            // Thêm request hiện tại
            zOps.add(key, String.valueOf(now), now);
            redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
            return true;
        }

        return false;
    }
}

// Dùng trong Filter hoặc Interceptor
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String clientIp = request.getRemoteAddr();

        if (!rateLimiter.isAllowed(clientIp, 100, 60)) { // 100 requests/phút
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

## 9. Cấu Hình Redis Cluster (Production)

```yaml
# Cấu hình Redis Cluster (Cụm Redis) — production
spring:
  data:
    redis:
      cluster:
        nodes:
          - redis-node-1:6379
          - redis-node-2:6379
          - redis-node-3:6379
        max-redirects: 3
      lettuce:
        cluster:
          refresh:
            adaptive: true           # Tự động detect topology thay đổi
            period: 30s

# Sentinel (Người Canh Gác) — High Availability
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - sentinel-1:26379
          - sentinel-2:26379
          - sentinel-3:26379
        password: sentinel-password
```

---

## ✅ Checklist Redis Integration

- [ ] Cấu hình connection pool hợp lý (max-active, min-idle)
- [ ] Đặt TTL cho mọi cache key — không để cache sống mãi
- [ ] Serialize value thành JSON (không dùng Java serialization mặc định)
- [ ] Xử lý RedisConnectionException khi Redis down (circuit breaker)
- [ ] Monitor cache hit rate — nên đạt > 80%
- [ ] Dùng key prefix để tránh conflict giữa các service
- [ ] Test cache eviction logic kỹ lưỡng

---

## 💡 Anti-Patterns Phổ Biến

```java
// ❌ Cache không có TTL — cache sống mãi, dữ liệu cũ
redisTemplate.opsForValue().set(key, value); // Không có TTL!
// ✅ Luôn đặt TTL
redisTemplate.opsForValue().set(key, value, Duration.ofHours(1));

// ❌ Cache key quá ngắn — conflict giữa các service
redisTemplate.opsForValue().set("user", userDto);
// ✅ Key có namespace rõ ràng
redisTemplate.opsForValue().set("myapp:v1:user:" + userId, userDto);

// ❌ Cache toàn bộ List lớn
redisTemplate.opsForValue().set("all-products", productRepository.findAll());
// → Cache thay đổi liên tục, vô dụng, tốn memory
// ✅ Cache từng entity riêng lẻ
productRepository.findAll().forEach(p ->
    redisTemplate.opsForValue().set("product:" + p.getId(), p, TTL));
```

---

## 🔗 Liên Kết

- [4-n-plus-one-problem.md](4-n-plus-one-problem.md) — Kết hợp cache với JPA
- [07-performance/1-caching-strategies.md](../07-performance/1-caching-strategies.md) — Chiến lược cache toàn diện
- [Spring Data Redis Reference](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Redis Documentation](https://redis.io/documentation)

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
