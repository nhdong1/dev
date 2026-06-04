# 🏗️ Bài Toán Thiết Kế Hệ Thống — System Design Scenarios

> Hướng dẫn thiết kế hệ thống thực tế cho phỏng vấn Backend Spring Boot. Mỗi scenario bao gồm: yêu cầu, kiến trúc đề xuất, lựa chọn công nghệ và trade-offs cần thảo luận.

---

## 🎯 Cách Tiếp Cận System Design Interview

### Framework RESHADED

```
R — Requirements (Yêu cầu): Functional + Non-functional
E — Estimate (Ước tính): Scale, throughput, storage
S — Storage (Lưu trữ): Database selection, schema design
H — High-level Design (Thiết kế cấp cao): Kiến trúc tổng thể
A — APIs: Endpoint design
D — Detailed Design (Thiết kế chi tiết): Deep dive vào components
E — Edge Cases (Trường hợp biên): Failure scenarios, bottlenecks
D — Delivery (Phân phối): Caching, CDN, deployment
```

### Thứ tự trình bày (45 phút)

```
5 phút:  Clarify requirements — hỏi trước khi thiết kế
10 phút: High-level architecture — vẽ sơ đồ tổng thể
15 phút: Deep dive vào 2–3 components quan trọng nhất
10 phút: Scaling & optimization
5 phút:  Trade-offs và những gì sẽ làm khác đi
```

---

## 🔗 Scenario 1: URL Shortener (Dịch Vụ Rút Gọn URL)

**Tương tự:** bit.ly, TinyURL, t.co

### Yêu Cầu

**Functional (Chức Năng):**
- Tạo URL ngắn từ URL dài
- Redirect từ URL ngắn về URL gốc
- URL ngắn hết hạn sau 1 năm (configurable)
- Tùy chọn custom alias (bí danh tùy chỉnh)
- Analytics (phân tích): số lần click, referrer, location

**Non-functional (Phi Chức Năng):**
- 100M URLs mới/ngày, 10B redirects/ngày
- Redirect latency (độ trễ) < 10ms (P99)
- High availability (tính sẵn sàng cao) — 99.99% uptime
- URL shortcode: 7 ký tự, case-sensitive [a-zA-Z0-9]

### Ước Tính

```
Write QPS (Query Per Second — Truy Vấn Mỗi Giây):
  100M / 86400s ≈ 1,160 writes/s

Read QPS (redirects):
  10B / 86400s ≈ 115,740 reads/s → read:write ratio ≈ 100:1

Storage (1 năm):
  100M URLs/ngày × 365 ngày = 36.5B URLs
  500 bytes/URL → 36.5B × 500 = ~18 TB

Bandwidth (Băng Thông):
  Read: 115,740 × 500 bytes = ~55 MB/s
```

### Kiến Trúc Đề Xuất

```
Client
  ↓
CDN (CloudFront / Cloudflare)
  ↓ (cache miss)
Spring Cloud Gateway (Rate Limiting, Auth)
  ↓
  ├── URL Shortener Service (Spring Boot)
  │     ├── POST /api/urls       → tạo short URL
  │     └── GET /{shortCode}     → redirect
  │
  ├── Analytics Service (Spring Boot)
  │     └── Kafka Consumer — xử lý click events
  │
  └── Data Layer
        ├── Redis Cluster (cache redirect mappings)
        ├── PostgreSQL (persistent store)
        └── Kafka (click events stream)
```

### Thiết Kế Chi Tiết

**Tạo Short Code (7 ký tự):**

```java
// Option 1: Base62 encoding của auto-increment ID
// ID 1000000 → base62 → "4c92"
// 62^7 ≈ 3.5 tỷ unique URLs → đủ dùng

// Option 2: MD5/SHA1 hash, lấy 7 ký tự đầu
// Collision risk → cần check và retry

// Option 3: UUID v4 + truncate → collision risk cao hơn

@Service
public class UrlShortenerService {
    
    public String createShortUrl(String longUrl, String customAlias) {
        String shortCode = customAlias != null 
            ? validateAndReserveAlias(customAlias)
            : generateShortCode(); // Base62 từ Snowflake ID
        
        UrlMapping mapping = new UrlMapping(shortCode, longUrl, Instant.now().plus(1, YEARS));
        urlRepository.save(mapping);
        
        // Cache ngay lập tức để redirect nhanh
        redisTemplate.opsForValue().set(
            "url:" + shortCode, longUrl, Duration.ofDays(365));
        
        return "https://short.ly/" + shortCode;
    }
    
    public String redirect(String shortCode) {
        // 1. Kiểm tra Redis cache (fast path)
        String longUrl = redisTemplate.opsForValue().get("url:" + shortCode);
        
        if (longUrl == null) {
            // 2. Cache miss → query DB
            longUrl = urlRepository.findByShortCode(shortCode)
                .map(UrlMapping::getLongUrl)
                .orElseThrow(() -> new UrlNotFoundException(shortCode));
            
            // 3. Warm cache
            redisTemplate.opsForValue().set("url:" + shortCode, longUrl, Duration.ofHours(24));
        }
        
        // 4. Publish click event cho analytics (async)
        kafkaTemplate.send("url.clicks", new ClickEvent(shortCode, getClientInfo()));
        
        return longUrl;
    }
}
```

**Database Schema:**

```sql
CREATE TABLE url_mappings (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    user_id     BIGINT REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMPTZ NOT NULL,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_short_code ON url_mappings(short_code); -- lookup by short code
CREATE INDEX idx_expires_at ON url_mappings(expires_at); -- cleanup job
```

### Trade-offs Để Thảo Luận

1. **ID Generation (Tạo ID):** Snowflake ID (phân tán) vs DB auto-increment → Snowflake scale tốt hơn
2. **Cache Eviction (Xóa Cache):** LRU vs TTL-based — hot URLs nên cache lâu hơn
3. **Collision handling:** Base62 không có collision; hash-based cần retry logic
4. **Analytics:** Real-time (streaming) vs batch — trade-off latency vs cost

---

## 🛒 Scenario 2: E-Commerce Order System (Hệ Thống Đặt Hàng Thương Mại Điện Tử)

**Tương tự:** Shopee, Lazada, Tiki order management

### Yêu Cầu

**Functional:**
- Đặt hàng: chọn sản phẩm, thanh toán, xác nhận
- Kiểm tra và giữ inventory (tồn kho) trong khi thanh toán
- Theo dõi trạng thái đơn hàng real-time
- Hỗ trợ nhiều payment methods (phương thức thanh toán)
- Notification (thông báo) qua email/SMS/push

**Non-functional:**
- 50K orders/ngày, peak 5K orders/phút (flash sale)
- Inventory consistency (nhất quán tồn kho) — không oversell
- Order completion < 3 giây
- 99.9% availability

### Kiến Trúc Microservices

```
┌──────────────────────────────────────────────────────┐
│                    API Gateway                        │
│         (Authentication, Rate Limiting, Routing)     │
└──────────┬───────────────────────────────────────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
Order Service  Product Service
    │             │
    ▼             ▼
Payment Service  Inventory Service
    │             │
    ▼             ▼
Notification   Analytics
Service        Service
    │
    ▼ (Event Bus — Kafka)
    └── Tất cả services pub/sub qua Kafka
```

### Thiết Kế Order Flow với Saga Pattern

**Orchestration Saga cho Order Placement:**

```java
@Service
public class OrderSagaOrchestrator {
    
    @Transactional
    public OrderResult placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command); // trạng thái PENDING
        orderRepo.save(order);
        
        // Bước 1: Reserve inventory
        try {
            inventoryClient.reserve(order.getItems()); // sync gRPC call
        } catch (InsufficientStockException e) {
            order.fail("Insufficient stock");
            return OrderResult.failed(e.getMessage());
        }
        
        // Bước 2: Process payment
        try {
            PaymentResult payment = paymentClient.charge(order.getPaymentInfo());
            order.confirm(payment.getTransactionId());
        } catch (PaymentException e) {
            // Compensating transaction (giao dịch bù trừ) — hoàn tồn kho
            inventoryClient.release(order.getItems());
            order.fail("Payment failed: " + e.getMessage());
            return OrderResult.failed(e.getMessage());
        }
        
        // Bước 3: Publish OrderConfirmed event
        eventPublisher.publishEvent(new OrderConfirmedEvent(order));
        return OrderResult.success(order.getId());
    }
}
```

**Inventory Service với Optimistic Locking:**

```java
@Service
public class InventoryService {
    
    @Transactional
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    public void reserve(List<OrderItem> items) {
        for (OrderItem item : items) {
            // Optimistic locking — @Version field trong Product entity
            Product product = productRepo.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
            
            if (product.getStock() < item.getQuantity()) {
                throw new InsufficientStockException(item.getProductId());
            }
            
            product.decrementStock(item.getQuantity()); // @Version tăng lên
            productRepo.save(product);
            // Nếu concurrent update xảy ra → OptimisticLockException → @Retryable
        }
    }
}
```

### Handling Flash Sale (Giảm Giá Sốc — Lưu Lượng Cao)

```java
// Dùng Redis Atomic Operations để giảm tải DB
@Service
public class FlashSaleInventoryService {
    
    // Pre-load inventory vào Redis trước khi flash sale bắt đầu
    public void preloadInventory(Long productId, int quantity) {
        String key = "flash:inventory:" + productId;
        redisTemplate.opsForValue().set(key, quantity);
    }
    
    // DECR là atomic operation trong Redis — thread-safe
    public boolean reserveInRedis(Long productId, int quantity) {
        String key = "flash:inventory:" + productId;
        Long remaining = redisTemplate.opsForValue().decrement(key, quantity);
        
        if (remaining < 0) {
            // Hoàn lại — oversold
            redisTemplate.opsForValue().increment(key, quantity);
            return false; // Sold out (Hết hàng)
        }
        
        // Async sync Redis → DB
        kafkaTemplate.send("inventory.reserved", new InventoryReservedEvent(productId, quantity));
        return true;
    }
}
```

### Trade-offs Để Thảo Luận

1. **Consistency vs Availability:** Inventory reservation — CP (consistency priority) hay AP (availability priority)?
2. **Saga Orchestration vs Choreography:** Orchestration dễ debug hơn nhưng coupling cao hơn
3. **Distributed Transaction vs Eventual Consistency (Nhất Quán Cuối Cùng):** 2PC quá chậm; Saga + compensating transactions thực tế hơn
4. **Flash Sale Strategy:** Queue-based (xếp hàng) vs token-bucket → trade-off UX vs simplicity

---

## 💬 Scenario 3: Chat Application (Ứng Dụng Nhắn Tin)

**Tương tự:** Zalo, Telegram, Slack messaging backend

### Yêu Cầu

**Functional:**
- Tin nhắn 1-1 và nhóm (group chat)
- Real-time delivery (giao tin nhắn thời gian thực)
- Message persistence (lưu lịch sử)
- Online/offline status
- Message read receipts (xác nhận đã đọc)
- File upload/sharing

**Non-functional:**
- 100M active users, 50M concurrent
- Message delivery latency < 100ms
- Messages stored 1 năm
- 99.99% uptime

### Kiến Trúc Real-time

```
Client (Mobile/Web)
  ↓ WebSocket / Server-Sent Events
Connection Service (Spring WebFlux)
  ├── Maintain long-lived connections
  ├── Route messages đến đúng recipient
  └── Publish to Kafka khi user offline
        ↓
    Kafka "messages" topic
        ↓
    ├── Message Service (Spring Boot) → PostgreSQL (persist)
    ├── Push Notification Service → Firebase / APNs
    └── Search Service → Elasticsearch (full-text search)
```

### WebSocket Implementation với Spring WebFlux

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatWebSocketHandler(), "/ws/chat")
                .setAllowedOrigins("*");
    }
}

@Component
public class ChatWebSocketHandler implements WebSocketHandler {
    
    // Map userId → WebSocket session
    private final ConcurrentHashMap<Long, WebSocketSession> activeSessions 
        = new ConcurrentHashMap<>();
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Long userId = extractUserId(session); // từ JWT trong handshake
        activeSessions.put(userId, session);
        
        // Xử lý messages từ client
        Mono<Void> input = session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .flatMap(this::processMessage)
            .then();
        
        // Cleanup khi disconnect
        return input.doFinally(signal -> {
            activeSessions.remove(userId);
            presenceService.setOffline(userId);
        });
    }
    
    public Mono<Void> deliverMessage(Long recipientId, ChatMessage message) {
        WebSocketSession session = activeSessions.get(recipientId);
        
        if (session != null && session.isOpen()) {
            // Online → gửi trực tiếp qua WebSocket
            return session.send(Mono.just(session.textMessage(toJson(message))));
        } else {
            // Offline → push notification qua Kafka
            kafkaTemplate.send("offline.notifications", recipientId, message);
            return Mono.empty();
        }
    }
}
```

### Message Storage Strategy

**Fan-out on Write (Ghi Trước, Đọc Sau) vs Fan-out on Read (Đọc Mới Ghi):**

```
Fan-out on Write (phù hợp cho user ít followers):
  User A gửi → copy message vào inbox của mỗi recipient ngay
  Read: O(1) — chỉ đọc inbox của mình
  Write: O(N) — N = số members trong nhóm

Fan-out on Read (phù hợp cho celebrity/large groups):
  User A gửi → lưu 1 bản
  Read: O(N) — merge timelines khi đọc
  Write: O(1)

→ Hybrid: small groups = fan-out write; large groups (>1000) = fan-out read
```

**Database Schema cho Chat:**

```sql
-- Conversations (chats/groups)
CREATE TABLE conversations (
    id          UUID PRIMARY KEY,
    type        VARCHAR(10) NOT NULL, -- 'DIRECT' or 'GROUP'
    name        VARCHAR(100),          -- chỉ cho group
    created_at  TIMESTAMPTZ NOT NULL
);

-- Messages (partitioned by created_at)
CREATE TABLE messages (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    sender_id       BIGINT NOT NULL,
    content         TEXT,
    type            VARCHAR(20) NOT NULL, -- 'TEXT', 'IMAGE', 'FILE'
    created_at      TIMESTAMPTZ NOT NULL,
    deleted_at      TIMESTAMPTZ -- soft delete
) PARTITION BY RANGE (created_at); -- partition theo tháng

-- Read receipts
CREATE TABLE message_reads (
    message_id  UUID NOT NULL,
    user_id     BIGINT NOT NULL,
    read_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (message_id, user_id)
);
```

---

## 📊 Scenario 4: Notification System (Hệ Thống Thông Báo)

**Dùng cho:** Mọi hệ thống lớn cần gửi notification đa kênh

### Yêu Cầu

**Functional:**
- Gửi notification qua nhiều kênh: Email, SMS, Push, In-app
- Retry on failure (thử lại khi lỗi)
- Rate limiting per user (giới hạn tần suất mỗi user)
- Template management (quản lý mẫu)
- Delivery tracking (theo dõi giao tin)
- Unsubscribe / preference management

**Non-functional:**
- 100M notifications/ngày
- Email delivery < 1 phút; Push < 5 giây
- Guaranteed delivery (at-least-once)

### Kiến Trúc

```
Producer Services (Order Service, Auth Service, etc.)
  ↓ Publish to Kafka
"notifications.pending" topic (partitioned by userId)
  ↓
Notification Processor Service (Spring Boot + @KafkaListener)
  ├── Deduplication check (Redis — đã gửi chưa?)
  ├── User preference check (có muốn nhận không?)
  ├── Template rendering (điền dữ liệu vào template)
  └── Route to channel workers
        ↓
  ┌─────┬──────┬──────┬──────┐
Email  SMS  Push  In-app    (channel-specific services)
  ↓     ↓     ↓     ↓
SendGrid Twilio Firebase  DB
```

### Spring Boot Implementation

```java
@Service
public class NotificationProcessor {
    
    @KafkaListener(
        topics = "notifications.pending",
        groupId = "notification-processors",
        containerFactory = "notificationKafkaListenerContainerFactory"
    )
    public void processNotification(NotificationEvent event) {
        // 1. Deduplication (chống gửi trùng)
        String dedupeKey = "notif:sent:" + event.getIdempotencyKey();
        Boolean alreadySent = redisTemplate.opsForValue()
            .setIfAbsent(dedupeKey, "1", Duration.ofDays(1));
        
        if (Boolean.FALSE.equals(alreadySent)) {
            log.debug("Duplicate notification skipped: {}", event.getIdempotencyKey());
            return;
        }
        
        // 2. Kiểm tra preference của user
        NotificationPreference pref = prefService.getPreference(
            event.getUserId(), event.getType());
        
        if (!pref.isEnabled()) return;
        
        // 3. Render template
        String content = templateEngine.render(event.getTemplateId(), event.getParams());
        
        // 4. Gửi qua các kênh được bật
        pref.getEnabledChannels().forEach(channel ->
            channelRouter.send(channel, event.getUserId(), content));
    }
}

@Component
public class EmailChannelSender implements NotificationChannelSender {
    
    @Retry(name = "emailSend", maxAttempts = 3)
    public void send(Long userId, String content) {
        User user = userService.findById(userId);
        emailClient.send(user.getEmail(), content);
        
        // Track delivery
        deliveryTrackingRepo.save(new DeliveryRecord(
            userId, Channel.EMAIL, DeliveryStatus.SENT, Instant.now()));
    }
}
```

---

## 🎬 Scenario 5: Content Feed System (Hệ Thống Luồng Nội Dung)

**Tương tự:** News feed của Facebook, Twitter timeline, YouTube homepage

### Yêu Cầu

**Functional:**
- Hiển thị posts từ người dùng đang follow
- Pagination (phân trang) với cursor-based
- Real-time updates khi có post mới
- Like, comment, share
- Recommendation (gợi ý) content

**Non-functional:**
- 1B users, 500M DAU (Daily Active Users — Người Dùng Hoạt Động Hàng Ngày)
- Feed load < 500ms
- 1M new posts/ngày

### Fan-out Strategy

```java
@Service
public class FeedService {
    
    // Option A: Fan-out on Write (push model)
    @KafkaListener(topics = "posts.created")
    public void fanOutToFollowers(PostCreatedEvent event) {
        // Lấy danh sách followers (người theo dõi)
        List<Long> followerIds = followerService.getFollowers(event.getAuthorId());
        
        // Với celebrity (>10K followers) → skip, dùng pull model
        if (followerIds.size() > 10_000) {
            return; // sẽ handle bằng fan-out on read
        }
        
        // Push post vào feed của mỗi follower (Redis Sorted Set)
        followerIds.forEach(followerId -> {
            String feedKey = "feed:" + followerId;
            // score = timestamp → tự động sort theo thời gian
            redisTemplate.opsForZSet().add(feedKey, event.getPostId(), 
                event.getCreatedAt().toEpochMilli());
            
            // Giữ tối đa 500 posts trong feed
            redisTemplate.opsForZSet().removeRange(feedKey, 0, -501);
        });
    }
    
    // Get feed với cursor-based pagination
    public FeedPage getFeed(Long userId, Long cursor, int pageSize) {
        String feedKey = "feed:" + userId;
        
        // Lấy post IDs từ Redis (sắp xếp theo score giảm dần)
        Set<Long> postIds = redisTemplate.opsForZSet()
            .reverseRangeByScore(feedKey, 
                0, cursor != null ? cursor : Double.MAX_VALUE, 
                0, pageSize + 1); // +1 để check has_next
        
        // Merge với celebrity posts (fan-out on read)
        Set<Long> celebPostIds = getCelebrityPosts(userId, cursor);
        
        // Merge + sort + deduplicate
        return mergeFeed(postIds, celebPostIds, pageSize);
    }
}
```

---

## 📋 Checklist System Design

Trước khi kết thúc, kiểm tra đã cover chưa:

### Functional
- [ ] Đã xác định rõ core features
- [ ] Đã mô tả API design (request/response)
- [ ] Đã thiết kế database schema

### Non-functional
- [ ] Tính toán scale (QPS, storage, bandwidth)
- [ ] Xác định bottlenecks (điểm nghẽn cổ chai)
- [ ] Caching strategy
- [ ] Replication (sao chép) & Sharding (phân mảnh)

### Reliability (Độ Tin Cậy)
- [ ] Failure scenarios — what happens if X fails?
- [ ] Data backup & recovery strategy
- [ ] Circuit breaker / fallback mechanisms

### Trade-offs
- [ ] Giải thích tại sao chọn giải pháp này
- [ ] Nhắc đến alternatives đã xem xét
- [ ] Những gì sẽ làm khác nếu có thêm thời gian

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
