# 🏗️ System Design Scenarios — Kịch Bản Thiết Kế Hệ Thống Database

> 5 kịch bản thiết kế hệ thống hoàn chỉnh với database considerations — từ phân tích requirements đến kiến trúc chi tiết, trade-offs, và cách giải thích trong phỏng vấn.

## Mục Lục

1. [Scenario 1: URL Shortener quy mô lớn](#scenario-1-url-shortener-quy-mô-lớn)
2. [Scenario 2: Real-time Leaderboard cho Game](#scenario-2-real-time-leaderboard-cho-game)
3. [Scenario 3: Social Media Feed](#scenario-3-social-media-feed)
4. [Scenario 4: E-Commerce với Flash Sale](#scenario-4-e-commerce-với-flash-sale)
5. [Scenario 5: IoT Data Ingestion Platform](#scenario-5-iot-data-ingestion-platform)
6. [Framework Chung Cho System Design](#framework-chung-cho-system-design)

---

## Scenario 1: URL Shortener quy mô lớn

### Yêu Cầu (Requirements)

**Functional requirements (Yêu Cầu Chức Năng):**
- Tạo URL ngắn từ URL dài
- Redirect (Chuyển Hướng) từ URL ngắn → URL dài
- Custom alias (Bí Danh Tùy Chỉnh) tùy chọn
- Thống kê lượt click

**Non-functional requirements (Yêu Cầu Phi Chức Năng):**
- 100 triệu URLs được tạo mỗi ngày
- 10 tỷ redirects mỗi ngày (~115,000 redirects/giây)
- Latency (Độ Trễ) cho redirect < 50ms
- 99.9% availability (Tính Sẵn Sàng)

### Ước Lượng Quy Mô (Scale Estimation)

```
Write operations (Thao Tác Ghi):
  100 triệu URLs/ngày = ~1,200 URLs/giây

Read operations (Thao Tác Đọc):
  10 tỷ redirects/ngày = ~115,000 redirects/giây

Read:Write ratio = ~95:1 — Read heavy (Nặng Về Đọc)

Storage:
  100 triệu URLs/ngày × 365 ngày × 5 năm = 182 tỷ records
  Mỗi record ~200 bytes → ~36 TB total
```

### Kiến Trúc Database

```
┌──────────────────────────────────────────────────────┐
│                   URL Metadata Store                  │
│                                                       │
│  DynamoDB Table: "urls"                               │
│  ├── PK: short_code (e.g., "abc123")                 │
│  ├── long_url: "https://very-long-url.com/..."       │
│  ├── created_at: timestamp                            │
│  ├── expires_at: timestamp (TTL — Time-to-Live)      │
│  ├── owner_id: user reference                         │
│  └── custom_alias: boolean                            │
│                                                       │
│  GSI 1: owner_id + created_at → list user's URLs     │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│               ElastiCache Redis (Cache Layer)         │
│                                                       │
│  Key: "url:{short_code}"                             │
│  Value: long_url                                      │
│  TTL: 24 giờ (hot URLs), 1 giờ (cold URLs)          │
│                                                       │
│  Cache hit rate target: > 95%                         │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                   Analytics Store                     │
│                                                       │
│  DynamoDB Table: "click_stats"                        │
│  ├── PK: short_code + date                           │
│  ├── click_count: atomic counter                      │
│  └── TTL: 90 ngày                                    │
│                                                       │
│  Hoặc: Kinesis Data Streams → S3 → Athena (rẻ hơn)  │
└──────────────────────────────────────────────────────┘
```

### Redirect Flow (Luồng Chuyển Hướng)

```
1. User request: GET /abc123
2. App check Redis cache
   - Cache HIT: return 301 redirect (< 5ms)
   - Cache MISS:
     a. Query DynamoDB (10-20ms)
     b. Store in Redis cache
     c. Return 301 redirect
3. Async: ghi click event vào DynamoDB/Kinesis
```

### Trade-offs (Đánh Đổi)

| Quyết Định | Lý Do |
|------------|-------|
| **DynamoDB thay vì RDS** | Key-value lookup đơn giản, scale cực cao |
| **DynamoDB TTL** | Tự động xóa expired URLs, tiết kiệm chi phí |
| **301 vs 302 redirect** | 301 (Permanent — Vĩnh Viễn) → browser cache → giảm tải server; 302 (Temporary — Tạm Thời) → analytics chính xác hơn |
| **Redis cache trước DynamoDB** | 95%+ hit rate → DynamoDB chỉ nhận 5% traffic |

---

## Scenario 2: Real-time Leaderboard cho Game

### Yêu Cầu

**Functional requirements:**
- Update điểm người chơi real-time
- Xem top 100 người chơi toàn cầu
- Xem ranking (Thứ Hạng) của một người chơi cụ thể
- Leaderboard theo tuần/tháng/all-time
- 10 triệu active players, 1000 score updates/giây

### Kiến Trúc Database

```
┌──────────────────────────────────────────────────────┐
│              ElastiCache Redis — Core Leaderboard     │
│                                                       │
│  Sorted Set (Tập Hợp Sắp Xếp):                      │
│  Key: "leaderboard:global:2026-W20" (weekly)         │
│  Score: game score (số điểm)                          │
│  Member: user_id                                      │
│                                                       │
│  Redis commands (Lệnh Redis):                         │
│  ZADD leaderboard:global score user_id               │
│  ZREVRANK leaderboard:global user_id → ranking       │
│  ZREVRANGE leaderboard:global 0 99 → top 100         │
│                                                       │
│  TTL: 8 ngày (weekly leaderboard tự expire)          │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              DynamoDB — Persistent Storage            │
│                                                       │
│  Table: "player_scores"                               │
│  PK: player_id                                        │
│  Attributes: total_score, weekly_score, rank_history  │
│                                                       │
│  Table: "score_history"                               │
│  PK: player_id, SK: timestamp                         │
│  TTL: 90 ngày                                        │
└──────────────────────────────────────────────────────┘
```

### Score Update Flow (Luồng Cập Nhật Điểm)

```
1. Game server gửi score update
2. App: ZADD redis_leaderboard new_score player_id (atomic O(log N))
3. Async write to DynamoDB (eventual consistency OK here)
4. Fan-out nếu cần: DynamoDB Streams → Lambda → notification

Ranking query:
1. ZREVRANK leaderboard:global player_id → O(log N) rất nhanh
2. Top 100: ZREVRANGE leaderboard:global 0 99 WITHSCORES
```

### Trade-offs

| Quyết Định | Lý Do |
|------------|-------|
| **Redis Sorted Sets thay vì SQL** | O(log N) cho rank query, không cần full table scan |
| **Redis làm primary, DynamoDB làm secondary** | Redis cho real-time speed, DynamoDB cho persistence |
| **Multiple timeframe keys** | "leaderboard:weekly:W20", "leaderboard:monthly:M05" — independent TTLs |
| **Async DynamoDB write** | Tốc độ quan trọng hơn consistency nghiêm ngặt cho leaderboard |

### Vấn Đề Và Giải Pháp

```
Vấn đề: Redis restart → mất leaderboard
Giải pháp: Enable AOF persistence + snapshot;
           hoặc reconstruct từ DynamoDB khi startup (cold start)

Vấn đề: Tie-breaking (Phá Vỡ Hòa) khi cùng điểm
Giải pháp: Score = original_score * 10^10 + (MAX_TIMESTAMP - submission_time)
           → Ai đạt điểm trước xếp cao hơn
```

---

## Scenario 3: Social Media Feed

### Yêu Cầu

**Functional requirements:**
- User đăng bài (post) với text/image
- Follow (Theo Dõi) / Unfollow người khác
- Home feed (Bảng Tin Trang Chủ) hiển thị posts của người được follow
- Like, comment trên posts
- 50 triệu daily active users
- 1 triệu posts mới mỗi ngày
- Feed phải load trong < 200ms

### Kiến Trúc Database

```
┌──────────────────────────────────────────────────────┐
│                User & Relationship Store              │
│                                                       │
│  Aurora PostgreSQL:                                   │
│  users(id, username, email, created_at)              │
│  follows(follower_id, followee_id, created_at)       │
│                                                       │
│  Lý do SQL: Graph-like queries (Truy Vấn Kiểu Đồ Thị)│
│  "Ai follow X?" — JOIN đơn giản                      │
│  ACID cho follow/unfollow                             │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                    Post Store                         │
│                                                       │
│  DynamoDB Table: "posts"                              │
│  PK: post_id (UUID)                                   │
│  SK: created_at                                       │
│  Attributes: user_id, content, media_url, likes_count │
│                                                       │
│  GSI: user_id + created_at → user's posts            │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│               Feed Cache (ElastiCache Redis)          │
│                                                       │
│  Key: "feed:{user_id}"                               │
│  Value: List of post_ids (ordered by time)            │
│  TTL: 24 giờ                                         │
│  Size: Chỉ lưu 200 post IDs mới nhất                │
└──────────────────────────────────────────────────────┘
```

### Feed Generation Approaches (Phương Pháp Tạo Feed)

**Pull Model (Mô Hình Kéo) — Tốt cho accounts nhỏ:**
```
Khi user mở app:
1. Lấy danh sách followings từ Aurora
2. Query DynamoDB posts của mỗi following
3. Merge và sort by time
4. Cache kết quả vào Redis

Ưu điểm: Storage đơn giản
Nhược điểm: Latency cao khi có nhiều followings
```

**Push Model (Mô Hình Đẩy) — Tốt cho engagement:**
```
Khi user đăng bài mới:
1. Lấy followers list từ Aurora
2. Push post_id vào Redis feed của từng follower
   LPUSH feed:{follower_id} post_id
   LTRIM feed:{follower_id} 0 199  # Chỉ giữ 200 mới nhất

Khi user xem feed:
1. LRANGE feed:{user_id} 0 49  # Lấy 50 post_ids
2. Batch get posts từ DynamoDB
3. Return

Ưu điểm: Read cực nhanh
Nhược điểm: Celebrity với triệu followers → fanout (Phân Tán) tốn kém
```

**Hybrid Model (Mô Hình Lai) — Best practice:**
```
Regular users (< 10,000 followers): Push model
Celebrity accounts (> 10,000 followers): Pull model khi load
→ Kết hợp cả hai để handle celebrity problem (Vấn Đề Người Nổi Tiếng)
```

### Trade-offs

| Quyết Định | Lý Do |
|------------|-------|
| **Aurora cho relationships** | SQL phù hợp cho graph-like queries |
| **DynamoDB cho posts** | Scale cực cao, access by post_id |
| **Redis feed cache** | Sub-millisecond read cho home feed |
| **Hybrid feed model** | Giải quyết celebrity problem |

---

## Scenario 4: E-Commerce với Flash Sale

### Yêu Cầu

**Functional requirements:**
- Product catalog với inventory (Tồn Kho)
- Flash sale: giá giảm trong 1 giờ, 10,000 sản phẩm
- Xử lý 100,000 concurrent users khi flash sale
- Inventory accuracy (Chính Xác Tồn Kho) — không oversell (Bán Quá Số Lượng)
- Order processing với ACID guarantees

### Thách Thức Chính

```
Flash sale problem:
- 100,000 users cùng lúc mua 10,000 items
- Phải prevent overselling (ngăn bán quá)
- Phải handle thundering herd (Bầy Sấm Sét) — request spike đột ngột
```

### Kiến Trúc Database

```
┌──────────────────────────────────────────────────────┐
│           Inventory Management (Quản Lý Tồn Kho)     │
│                                                       │
│  Redis: "inventory:{product_id}" = 10000 (count)    │
│                                                       │
│  Giảm tồn kho atomic (không race condition):          │
│  DECRBY inventory:{product_id} 1                     │
│  → Nếu result < 0: INCRBY lại (rollback), sold out  │
│  → Nếu result >= 0: proceed with order               │
│                                                       │
│  Lý do Redis: atomic DECRBY, sub-millisecond latency │
│  → Handle 100,000 concurrent decrements mà không lock│
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                    Order Processing                   │
│                                                       │
│  Aurora PostgreSQL:                                   │
│  orders(id, user_id, product_id, quantity, status)  │
│  order_items(order_id, product_id, price, qty)       │
│                                                       │
│  Multi-AZ + Read Replicas                             │
│  RDS Proxy cho connection pooling                     │
│                                                       │
│  Lý do Aurora: ACID transactions, financial data      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              Product Catalog Cache                    │
│                                                       │
│  ElastiCache Redis:                                   │
│  Key: "product:{id}"                                  │
│  Value: product details, flash sale price             │
│  TTL: 5 phút (đủ fresh, không quá stale)             │
│                                                       │
│  CDN (Content Delivery Network) cho product images   │
└──────────────────────────────────────────────────────┘
```

### Flash Sale Flow (Luồng Flash Sale)

```
Pre-flash sale (Trước Flash Sale):
1. Load 10,000 inventory vào Redis: SET inventory:prod123 10000
2. Warm up product cache
3. Pre-scale Aurora với Read Replicas

Khi user checkout:
1. Check inventory Redis: DECRBY inventory:prod123 1
   - Result < 0: sold out, rollback INCRBY
   - Result >= 0: proceed
2. Create order in Aurora (ACID transaction):
   BEGIN
   INSERT INTO orders ...
   INSERT INTO order_items ...
   COMMIT
3. Publish to SQS (Simple Queue Service — Dịch Vụ Hàng Đợi Đơn Giản)
   → Payment service (async)

Post-order:
4. Async sync Redis inventory → Aurora inventory table
5. DynamoDB Streams cho audit/analytics
```

### Trade-offs

| Quyết Định | Lý Do |
|------------|-------|
| **Redis cho inventory check** | Atomic operations, sub-ms latency |
| **Aurora cho order persistence** | ACID critical cho financial data |
| **Async payment processing** | Tách payment flow → không block checkout |
| **Queue (SQS) cho downstream** | Absorb (Hấp Thụ) traffic spike |

---

## Scenario 5: IoT Data Ingestion Platform

### Yêu Cầu

**Functional requirements:**
- 1 triệu IoT devices gửi metrics mỗi 30 giây
- ~33,000 writes/giây
- Query: device metrics trong time range (Khoảng Thời Gian)
- Alerting (Cảnh Báo) khi metric vượt ngưỡng
- Lưu trữ 1 năm hot data, cold storage sau đó

### Tại Sao Time-Series Database?

```
Thách thức của IoT data:
- Write-heavy workload (Nặng Về Ghi): 33,000 writes/giây liên tục
- Time-ordered queries: "temperature từ 8am-9am"
- Retention policies: data cũ ít giá trị hơn
- Compression (Nén): time-series data nén được rất hiệu quả
```

### Kiến Trúc Database

```
┌──────────────────────────────────────────────────────┐
│              Data Ingestion (Thu Thập Dữ Liệu)        │
│                                                       │
│  IoT Devices → AWS IoT Core → Kinesis Data Streams   │
│                                           ↓           │
│                                   Lambda (batch)      │
│                                           ↓           │
│                              Amazon Timestream        │
│                                           ↓           │
│                                   S3 (magnetic store) │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              Amazon Timestream (Main Store)           │
│                                                       │
│  Bảng: "device_metrics"                              │
│  Dimensions (Chiều Dữ Liệu):                        │
│  - device_id, location, device_type                  │
│  Measures (Số Đo):                                   │
│  - temperature, humidity, battery_level, cpu_usage   │
│  Time: auto-timestamp                                 │
│                                                       │
│  Memory Store (Bộ Nhớ Nhanh): hot data 24 giờ       │
│  Magnetic Store (Lưu Trữ Từ Tính): 1 năm             │
│  S3 export (Xuất): sau 1 năm → Glacier              │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              DynamoDB — Device Registry               │
│                                                       │
│  Table: "devices"                                     │
│  PK: device_id                                        │
│  Attributes: firmware_version, location, owner, status│
│                                                       │
│  Lý do: metadata lookup không liên quan đến time-series│
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              Alerting Layer (Lớp Cảnh Báo)           │
│                                                       │
│  Timestream Scheduled Query → Lambda → SNS alert     │
│  Hoặc: Kinesis → Lambda (real-time alert, < 1 giây)  │
└──────────────────────────────────────────────────────┘
```

### Query Examples (Ví Dụ Truy Vấn)

```sql
-- Average temperature theo giờ cho device cụ thể
SELECT device_id,
       bin(time, 1h) AS hour,
       AVG(measure_value::double) AS avg_temp
FROM "device_metrics"."temperature"
WHERE device_id = 'device-001'
  AND time BETWEEN ago(24h) AND now()
GROUP BY device_id, bin(time, 1h)
ORDER BY hour DESC

-- Devices có battery thấp
SELECT device_id, measure_value::double AS battery
FROM "device_metrics"."battery_level"
WHERE time > ago(1h)
  AND measure_value::double < 20
```

### Trade-offs

| Quyết Định | Lý Do |
|------------|-------|
| **Timestream thay vì DynamoDB cho metrics** | Built-in time-series optimization, tiered storage tự động |
| **Kinesis thay vì direct ingest** | Buffer (Vùng Đệm) writes, handle spikes |
| **DynamoDB riêng cho device registry** | Metadata không phải time-series, access pattern khác |
| **Memory store + Magnetic store** | Tiết kiệm chi phí: hot data in-memory, cold data cheaper |

---

## Framework Chung Cho System Design

### Template Trả Lời (4-5 phút)

```
1. CLARIFY requirements (1-2 phút)
   "Trước khi bắt đầu, tôi muốn làm rõ một số điểm:
   - Quy mô: bao nhiêu users? QPS (Queries Per Second) dự kiến?
   - Consistency requirements: có cần strong consistency không?
   - Latency requirements: SLA (Service Level Agreement) là gì?
   - Read:Write ratio?"

2. ESTIMATE scale (30 giây)
   "Với X users và Y requests/giây, tôi ước tính..."

3. CHOOSE database(s) + JUSTIFY (1-2 phút)
   "Tôi chọn Aurora vì... DynamoDB vì... ElastiCache vì..."

4. DESIGN data model (1-2 phút)
   "Schema/table design cho từng database..."

5. DISCUSS trade-offs (1 phút)
   "Điểm mạnh: ... | Điểm yếu và cách giảm thiểu: ..."
```

### Database Selection Cheat Sheet

```
Traffic pattern \ Data model | Relational | Key-Value/Document | Time-Series
─────────────────────────────────────────────────────────────────────────────
Low traffic, complex queries  | RDS        | DynamoDB           | Timestream
High traffic, simple lookups  | Aurora     | DynamoDB + DAX     | Timestream
Real-time, sub-ms             | ElastiCache Redis               | Redis TSDB
Analytics (Phân Tích)         | Redshift   | Athena on DynamoDB | Timestream
```

### Checklist System Design Database

- [ ] Đã clarify consistency vs availability requirements
- [ ] Đã estimate scale (QPS, storage size, growth)
- [ ] Đã giải thích tại sao chọn SQL vs NoSQL
- [ ] Đã thiết kế HA và DR strategy
- [ ] Đã đề cập caching layer
- [ ] Đã tính đến cost (chi phí)
- [ ] Đã nói về monitoring và alerting
- [ ] Đã thảo luận trade-offs của thiết kế
