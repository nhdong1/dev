# Caching Strategies — Chiến Lược Bộ Nhớ Đệm

> Caching (Bộ Nhớ Đệm) là kỹ thuật lưu trữ kết quả tính toán tốn kém để tái sử dụng,
> giảm tải database và tăng tốc độ phản hồi. Spring Cache Abstraction (Tầng Trừu Tượng
> Bộ Nhớ Đệm) cho phép chuyển đổi giữa các provider (nhà cung cấp) mà không thay đổi code.

---

## 📋 Mục Tiêu

- [ ] Hiểu **Cache Abstraction** của Spring và các annotation cốt lõi
- [ ] Dùng **`@Cacheable`** để cache kết quả method
- [ ] Dùng **`@CacheEvict`** để xóa cache khi data thay đổi
- [ ] Dùng **`@CachePut`** để cập nhật cache sau write
- [ ] Cấu hình **Redis** làm distributed cache (Bộ Nhớ Đệm Phân Tán)
- [ ] Implement **Cache-Aside Pattern** (Mẫu Cache Phụ Trợ) đúng cách
- [ ] Xử lý **cache stampede** (Cơn Bão Cache) và **cache invalidation** (Thu Hồi Cache)

---

## 1. Spring Cache Abstraction

### Cách Hoạt Động

```
@Cacheable("products")          ← Khai báo cache name
public Product getById(Long id) {
    return repository.findById(id).orElseThrow();
}

Lần 1: cache miss → gọi method → lưu vào cache
Lần 2+: cache hit → trả về từ cache, KHÔNG gọi method
```

### Thêm Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Nếu dùng Redis làm cache provider -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### Bật Cache

```java
@SpringBootApplication
@EnableCaching  // ← BẮT BUỘC phải có annotation này
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 2. Ba Annotation Cốt Lõi

### 2.1 `@Cacheable` — Đọc Từ Cache

```java
@Service
public class ProductService {

    // Cache cơ bản — key mặc định là tham số method
    @Cacheable("products")
    public Product getById(Long id) {
        // Method này CHỈ chạy khi cache miss
        log.info("Fetching product {} from database", id);
        return repository.findById(id).orElseThrow();
    }

    // Cache với key tùy chỉnh — dùng SpEL (Spring Expression Language)
    @Cacheable(value = "products", key = "#id")
    public Product getByIdCustomKey(Long id) {
        return repository.findById(id).orElseThrow();
    }

    // Cache với điều kiện — chỉ cache khi id > 0
    @Cacheable(value = "products", condition = "#id > 0")
    public Product getByIdConditional(Long id) {
        return repository.findById(id).orElseThrow();
    }

    // Cache với unless — cache NGOẠI TRỪ khi kết quả null
    @Cacheable(value = "products", unless = "#result == null")
    public Product getByIdUnlessNull(Long id) {
        return repository.findById(id).orElse(null);
    }

    // Cache với composite key (Khóa Tổng Hợp)
    @Cacheable(value = "products", key = "#category + ':' + #page")
    public Page<Product> getByCategory(String category, int page) {
        return repository.findByCategory(category, PageRequest.of(page, 20));
    }
}
```

### 2.2 `@CacheEvict` — Xóa Cache

```java
@Service
public class ProductService {

    // Xóa 1 entry cụ thể khi cập nhật
    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public Product update(Product product) {
        return repository.save(product);
    }

    // Xóa toàn bộ cache sau khi delete
    @CacheEvict(value = "products", allEntries = true)
    @Transactional
    public void delete(Long id) {
        repository.deleteById(id);
    }

    // beforeInvocation = true: xóa cache TRƯỚC khi method chạy
    // Mặc định là false: xóa SAU khi method hoàn thành thành công
    @CacheEvict(value = "products", key = "#id", beforeInvocation = true)
    @Transactional
    public void deleteBeforeEvict(Long id) {
        repository.deleteById(id);
    }

    // Xóa nhiều cache cùng lúc với @Caching
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#product.id"),
        @CacheEvict(value = "productsByCategory", key = "#product.category")
    })
    @Transactional
    public Product updateWithMultipleEvict(Product product) {
        return repository.save(product);
    }
}
```

### 2.3 `@CachePut` — Cập Nhật Cache

```java
@Service
public class ProductService {

    // @CachePut: LUÔN chạy method VÀ cập nhật cache với kết quả mới
    // Khác @Cacheable: @Cacheable bỏ qua method khi cache hit
    @CachePut(value = "products", key = "#result.id")
    @Transactional
    public Product create(CreateProductRequest request) {
        Product product = mapper.toEntity(request);
        return repository.save(product);
    }

    // Kết hợp @CachePut và @CacheEvict
    @Caching(
        put = @CachePut(value = "products", key = "#result.id"),
        evict = @CacheEvict(value = "productList", allEntries = true)
    )
    @Transactional
    public Product updateAndRefreshList(Long id, UpdateProductRequest request) {
        Product product = repository.findById(id).orElseThrow();
        mapper.updateEntity(product, request);
        return repository.save(product);
    }
}
```

---

## 3. Cấu Hình Redis Cache

### `application.yml`

```yaml
spring:
  cache:
    type: redis                # Dùng Redis làm cache provider
    redis:
      time-to-live: 600000     # TTL (Time To Live — Thời Gian Sống) mặc định: 10 phút (ms)
      cache-null-values: false # Không cache giá trị null
      use-key-prefix: true     # Thêm cache name vào trước key

  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms          # Connection timeout
      lettuce:
        pool:
          max-active: 10       # Số kết nối tối đa trong pool
          max-idle: 5          # Số kết nối idle tối đa
          min-idle: 2          # Số kết nối idle tối thiểu
```

### Cấu Hình Redis Cache Manager (Tùy Chỉnh TTL Theo Cache)

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // Cấu hình mặc định cho tất cả caches
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))              // TTL mặc định 10 phút
            .disableCachingNullValues()                    // Không cache null
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())  // Serialize thành JSON
            );

        // Cấu hình riêng cho từng cache — TTL khác nhau
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();

        cacheConfigurations.put("products",
            defaultConfig.entryTtl(Duration.ofHours(1)));      // Products: cache 1 giờ

        cacheConfigurations.put("users",
            defaultConfig.entryTtl(Duration.ofMinutes(30)));   // Users: cache 30 phút

        cacheConfigurations.put("shortLived",
            defaultConfig.entryTtl(Duration.ofMinutes(5)));    // Short-lived: 5 phút

        cacheConfigurations.put("categories",
            defaultConfig.entryTtl(Duration.ofDays(1)));       // Categories: 1 ngày

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

---

## 4. Cache Patterns (Mẫu Cache)

### 4.1 Cache-Aside Pattern (Mẫu Cache Phụ Trợ) — Phổ Biến Nhất

```
Application quản lý cache thủ công:

READ:
  1. App kiểm tra cache
  2. Cache HIT → trả về ngay
  3. Cache MISS → đọc từ DB → lưu vào cache → trả về

WRITE:
  1. App ghi vào DB
  2. App xóa cache entry tương ứng (invalidate)
  3. Lần đọc tiếp → cache miss → reload từ DB
```

```java
// Spring @Cacheable = Cache-Aside pattern tự động
@Cacheable(value = "products", key = "#id")
public Product getById(Long id) {
    return repository.findById(id).orElseThrow();
}

@CacheEvict(value = "products", key = "#product.id")
public Product update(Product product) {
    return repository.save(product);
}
```

### 4.2 Write-Through Pattern (Mẫu Ghi Xuyên)

```
Mỗi write → cập nhật DB VÀ cache đồng thời

Ưu điểm: Cache luôn đồng bộ với DB
Nhược điểm: Tốn tài nguyên hơn (2 writes mỗi lần)
```

```java
// @CachePut = Write-Through pattern
@CachePut(value = "products", key = "#result.id")
public Product update(Long id, UpdateProductRequest request) {
    Product product = repository.findById(id).orElseThrow();
    mapper.updateEntity(product, request);
    return repository.save(product);
}
```

### 4.3 Read-Through Pattern (Mẫu Đọc Xuyên)

```
Cache tự động load từ DB khi miss (cache là nguồn sự thật chính)
Spring @Cacheable = Read-Through khi kết hợp với CacheLoader
```

### 4.4 Write-Behind Pattern (Mẫu Ghi Sau) — Bất Đồng Bộ

```
Write → vào cache trước → DB được cập nhật bất đồng bộ sau
Rủi ro: Mất data nếu cache crash trước khi sync sang DB
Phù hợp: Counter updates, event streaming
```

---

## 5. Cache Key Design (Thiết Kế Khóa Cache)

### SpEL (Spring Expression Language) trong Cache Key

```java
// Các biểu thức SpEL phổ biến:

@Cacheable(value = "cache", key = "#id")
// key = giá trị tham số id

@Cacheable(value = "cache", key = "#user.id")
// key = trường id của object user

@Cacheable(value = "cache", key = "'prefix:' + #id")
// key = "prefix:123" (thêm prefix cố định)

@Cacheable(value = "cache", key = "#p0")
// key = tham số đầu tiên (positional)

@Cacheable(value = "cache", key = "#root.methodName + ':' + #id")
// key = "getById:123" (thêm tên method)

@Cacheable(value = "cache", key = "T(java.util.Objects).hash(#category, #page)")
// key = hash của nhiều tham số
```

### Custom KeyGenerator (Bộ Tạo Khóa Tùy Chỉnh)

```java
@Component("customKeyGenerator")
public class CustomCacheKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        StringBuilder key = new StringBuilder();
        key.append(target.getClass().getSimpleName())
           .append(":")
           .append(method.getName())
           .append(":");

        for (Object param : params) {
            if (param != null) {
                key.append(param.toString()).append(":");
            }
        }

        return key.toString();
    }
}

// Sử dụng custom key generator
@Cacheable(value = "products", keyGenerator = "customKeyGenerator")
public Product getById(Long id) {
    return repository.findById(id).orElseThrow();
}
```

---

## 6. Cache Stampede Prevention (Ngăn Cơn Bão Cache)

### Vấn Đề Cache Stampede

```
Cache stampede (cơn bão cache) xảy ra khi:
1. Cache entry hết hạn (expired)
2. Nhiều requests đồng thời gặp cache miss
3. TẤT CẢ requests cùng gọi vào DB
4. DB bị quá tải đột ngột

Ví dụ: 1000 users cùng request 1 product → cache expired
       → 1000 concurrent queries đến DB
       → DB timeout, toàn bộ service down
```

### Giải Pháp 1 — Mutex Lock (Khóa Độc Quyền)

```java
@Service
public class ProductService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository repository;

    // Tránh stampede bằng distributed lock (Khóa Phân Tán)
    public Product getByIdWithLock(Long id) {
        String cacheKey = "product:" + id;
        String lockKey = "lock:product:" + id;

        // Thử lấy từ cache trước
        Product cached = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) return cached;

        // Cache miss — thử acquire lock
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

        if (Boolean.TRUE.equals(acquired)) {
            try {
                // Double-check sau khi acquire lock
                cached = (Product) redisTemplate.opsForValue().get(cacheKey);
                if (cached != null) return cached;

                // Load từ DB và lưu vào cache
                Product product = repository.findById(id).orElseThrow();
                redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(10));
                return product;
            } finally {
                redisTemplate.delete(lockKey);
            }
        } else {
            // Không acquire được lock — chờ và retry
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return getByIdWithLock(id);  // Recursive retry
        }
    }
}
```

### Giải Pháp 2 — Jitter TTL (TTL Ngẫu Nhiên)

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // Thêm jitter (độ nhiễu) vào TTL để tránh nhiều cache hết hạn cùng lúc
        Duration baseTtl = Duration.ofMinutes(10);
        Duration jitter = Duration.ofSeconds(ThreadLocalRandom.current().nextLong(0, 120));

        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(baseTtl.plus(jitter));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

## 7. Multi-Level Caching (Bộ Nhớ Đệm Đa Tầng)

```
Tầng 1: Local Cache (Bộ Nhớ Đệm Cục Bộ) — Caffeine/Ehcache
  - Cực nhanh (nanoseconds)
  - Giới hạn bởi JVM heap
  - Không chia sẻ giữa các instances

Tầng 2: Distributed Cache (Bộ Nhớ Đệm Phân Tán) — Redis
  - Nhanh (milliseconds)
  - Chia sẻ giữa tất cả instances
  - Persistence (Bền Vững) tùy chọn

Tầng 3: Database
  - Chậm (10–100ms+)
  - Nguồn sự thật duy nhất
```

```java
// Cấu hình Caffeine làm L1 cache
@Configuration
@EnableCaching
public class MultiLevelCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisFactory) {
        // L1: Caffeine — local in-memory cache
        CaffeineCacheManager caffeineCacheManager = new CaffeineCacheManager();
        caffeineCacheManager.setCaffeine(
            Caffeine.newBuilder()
                .maximumSize(500)              // Tối đa 500 entries
                .expireAfterWrite(1, TimeUnit.MINUTES)  // Expire sau 1 phút
                .recordStats()                  // Bật thống kê
        );

        return caffeineCacheManager;
    }

    // L2: Redis cache manager (cấu hình riêng)
    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
    }
}
```

---

## 8. Cache Monitoring (Giám Sát Cache)

### Metrics với Micrometer

```yaml
# application.yml — bật cache metrics
management:
  endpoints:
    web:
      exposure:
        include: metrics, caches
  metrics:
    cache:
      instrument:
        redis: true  # Bật metrics cho Redis cache
```

```java
// Xem cache statistics tại runtime
@Autowired
private CacheManager cacheManager;

public void printCacheStats() {
    Cache cache = cacheManager.getCache("products");
    if (cache instanceof CaffeineCache caffeineCache) {
        CacheStats stats = caffeineCache.getNativeCache().stats();
        log.info("Cache hit rate: {}", stats.hitRate());
        log.info("Cache miss count: {}", stats.missCount());
        log.info("Cache eviction count: {}", stats.evictionCount());
    }
}
```

### Prometheus Metrics Quan Trọng

```
# Cache hit ratio — tỷ lệ cache hit (mục tiêu: > 80%)
cache_gets_total{result="hit"}  /  cache_gets_total

# Cache size
cache_size

# Redis memory usage
redis_memory_used_bytes

# Redis connected clients
redis_connected_clients
```

---

## 9. Anti-Patterns (Mẫu Chống Chỉ Định)

### ❌ Chớ Làm

```java
// ANTI-PATTERN 1: Cache trong @Transactional — dễ gây inconsistency
@Transactional
@Cacheable("products")   // ← SAI: cache có thể load data chưa commit
public Product getById(Long id) {
    return repository.findById(id).orElseThrow();
}

// ANTI-PATTERN 2: Cache mutable objects
@Cacheable("products")
public List<Product> getAll() {
    return repository.findAll();  // ← SAI: List có thể bị mutate bên ngoài
}

// ANTI-PATTERN 3: Cache method không có tham số ổn định
@Cacheable("currentUser")
public User getCurrentUser() {
    return SecurityContextHolder.getContext()...  // ← SAI: key là "" cho mọi user!
}

// ANTI-PATTERN 4: TTL quá dài cho data thay đổi thường xuyên
@Cacheable(value = "stockPrice", cacheManager = "redisCacheManager")  // TTL 1 giờ
public BigDecimal getStockPrice(String symbol) {
    return stockService.getPrice(symbol);  // ← SAI: giá cổ phiếu thay đổi theo giây
}
```

### ✅ Nên Làm

```java
// ĐÚNG 1: Tách @Cacheable ra khỏi @Transactional
// Cache ở service layer, @Transactional ở một method khác
@Cacheable("products")
public Product getById(Long id) {
    return productLoader.loadFromDb(id);  // loadFromDb có @Transactional
}

// ĐÚNG 2: Cache immutable hoặc defensive copy
@Cacheable("products")
public List<Product> getAll() {
    return Collections.unmodifiableList(repository.findAll());
}

// ĐÙNG 3: Cache với user-specific key
@Cacheable(value = "userDashboard", key = "#userId")
public Dashboard getDashboard(Long userId) {
    return dashboardService.build(userId);
}

// ĐÚNG 4: TTL phù hợp với business logic
@Cacheable(value = "productCatalog")  // TTL 1 giờ — catalog ít thay đổi
public List<Category> getCategories() {
    return categoryRepository.findAll();
}
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa `@Cacheable`, `@CachePut`, và `@CacheEvict`?**

```
@Cacheable: Cache-Aside READ — trả về từ cache nếu hit, gọi method nếu miss
@CachePut:  Write-Through WRITE — luôn gọi method VÀ cập nhật cache
@CacheEvict: Invalidate (Thu Hồi) — xóa entry hoặc toàn bộ cache
```

**Q: Khi nào dùng local cache (Caffeine) vs distributed cache (Redis)?**

```
Local cache (Caffeine):
  ✓ Data ít thay đổi, read-heavy (nhiều đọc)
  ✓ Không cần chia sẻ giữa các instances
  ✓ Cần latency cực thấp (nanoseconds)
  ✗ Không phù hợp cho multi-instance deployment

Distributed cache (Redis):
  ✓ Multi-instance deployment (K8s, microservices)
  ✓ Session data, token blacklist
  ✓ Data cần đồng nhất giữa các instances
  ✗ Thêm network latency (~1–5ms)
```

**Q: Cache invalidation strategy (Chiến Lược Thu Hồi Cache) nào phổ biến nhất?**

```
1. TTL-based (Dựa Trên Thời Gian Sống): đơn giản, eventual consistency
2. Event-driven (Hướng Sự Kiện): invalidate khi data thay đổi — dùng @CacheEvict
3. Versioned cache (Cache Có Phiên Bản): thêm version vào key — không bao giờ delete
```

---

## ✅ Checklist

- [ ] `@EnableCaching` được thêm vào `@SpringBootApplication`
- [ ] Redis dependency và `spring.cache.type=redis` được cấu hình
- [ ] TTL phù hợp cho mỗi cache (không quá dài, không quá ngắn)
- [ ] `@CacheEvict` được gọi đúng chỗ khi data thay đổi
- [ ] Không cache mutable objects
- [ ] Không đặt `@Cacheable` bên trong `@Transactional`
- [ ] Cache key đủ phân biệt — tránh collision (Va Chạm Khóa) giữa users
- [ ] Monitoring cache hit rate > 80%

---

**Xem tiếp:** [2-connection-pooling.md](2-connection-pooling.md) — HikariCP Configuration
