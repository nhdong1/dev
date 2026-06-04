# Redis vs Memcached — So Sánh Chi Tiết & Khi Nào Dùng Gì

> Cả Redis và Memcached đều là in-memory data stores (kho dữ liệu trong bộ nhớ RAM) với tốc độ cực cao. Hiểu rõ sự khác biệt giúp chọn đúng công cụ cho từng use case (trường hợp sử dụng).

---

## 📊 Bảng So Sánh Toàn Diện

| Tiêu Chí | Redis | Memcached |
|----------|-------|-----------|
| **Ra Đời** | 2009 (Salvatore Sanfilippo) | 2003 (Brad Fitzpatrick) |
| **Kiến Trúc** | Single-threaded với I/O multiplexing | Multi-threaded (Đa Luồng) |
| **Data Types** (Kiểu Dữ Liệu) | String, List, Set, Hash, Sorted Set, Stream, Bitmap, HyperLogLog | String (binary-safe) |
| **Persistence** (Bền Vững Dữ Liệu) | ✅ RDB Snapshots + AOF Log | ❌ Dữ liệu mất khi restart |
| **Replication** (Sao Chép) | ✅ Primary-Replica asynchronous | ❌ Không hỗ trợ |
| **Cluster** (Phân Cụm) | ✅ Redis Cluster (16384 hash slots) | ✅ Consistent Hashing (Băm Nhất Quán) |
| **High Availability** (Tính Sẵn Sàng Cao) | ✅ Automatic failover | ❌ Không có failover |
| **Pub/Sub** (Xuất Bản/Đăng Ký) | ✅ Hỗ trợ đầy đủ | ❌ Không |
| **Transactions** (Giao Dịch) | ✅ MULTI/EXEC, Lua scripts | ❌ Không |
| **Atomic Operations** (Thao Tác Nguyên Tử) | ✅ INCR, DECR, GETSET... | ✅ Hạn chế (CAS — Compare-and-Swap) |
| **Max Object Size** | 512 MB per key | 1 MB per key |
| **Memory Efficiency** | Thấp hơn do overhead metadata | Cao hơn cho simple strings |
| **CPU Usage** | Thấp hơn (single-threaded) | Cao hơn (multi-threaded, tận dụng nhiều CPU) |
| **Backup** (Sao Lưu AWS) | ✅ Automatic snapshots | ❌ Không có |
| **Geospatial** (Địa Lý) | ✅ GEO commands | ❌ Không |
| **Streams** | ✅ Redis Streams (như Kafka đơn giản) | ❌ Không |

---

## 🔴 Redis — Phân Tích Chi Tiết

### Kiến Trúc Single-Threaded (Đơn Luồng)

```
┌─────────────────────────────────────────────┐
│           Redis Process                      │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │  Event Loop (Vòng Lặp Sự Kiện)       │   │
│  │  - Xử lý 1 command tại một thời điểm │   │
│  │  - Không có race condition            │   │
│  │  - I/O multiplexing với epoll/kqueue  │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │  In-Memory Data Store (RAM)          │   │
│  │  - String: "user:1" → "John Doe"     │   │
│  │  - Hash: "session:abc" → {...}       │   │
│  │  - Sorted Set: "leaderboard" → score │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │  Persistence Layer (Lớp Bền Vững)    │   │
│  │  - RDB: Periodic snapshots           │   │
│  │  - AOF: Append-only log              │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Các Kiểu Dữ Liệu Redis

#### String — Chuỗi

```bash
# Cơ bản
SET product:1:name "iPhone 15"
GET product:1:name                     # → "iPhone 15"
SETEX session:token123 3600 "user_data" # SET với TTL (Time-To-Live) 1 giờ

# Numeric operations (Thao Tác Số)
SET counter:page_views 0
INCR counter:page_views               # → 1 (atomic increment — tăng nguyên tử)
INCRBY counter:page_views 5           # → 6
```

#### Hash — Bảng Băm

```bash
# Lưu object có nhiều fields
HSET user:1001 name "Nguyen Van A" email "a@example.com" age 30
HGET user:1001 name                   # → "Nguyen Van A"
HGETALL user:1001                     # → tất cả fields
HMSET user:1001 name "Tran B" age 31  # Set nhiều fields
```

#### List — Danh Sách (Ordered, Duplicates Allowed — Có Thứ Tự, Cho Phép Trùng)

```bash
# Queue (Hàng Đợi) — FIFO
RPUSH queue:emails "email1" "email2"  # Thêm vào đuôi
LPOP queue:emails                     # Lấy từ đầu → "email1"

# Recent activity log (Nhật Ký Hoạt Động Gần Đây)
LPUSH activity:user:1 "login" "view_product" "add_to_cart"
LRANGE activity:user:1 0 9            # 10 hoạt động mới nhất
```

#### Set — Tập Hợp (Unordered, Unique — Không Thứ Tự, Không Trùng)

```bash
# Tags (Nhãn)
SADD product:1:tags "electronics" "smartphone" "apple"
SMEMBERS product:1:tags               # → tất cả tags

# Set operations (Thao Tác Tập Hợp)
SUNION product:1:tags product:2:tags  # Hợp (Union)
SINTER vip_users online_users         # Giao (Intersection) — user vừa VIP vừa online
```

#### Sorted Set — Tập Hợp Có Thứ Tự (Score-Based Ranking — Xếp Hạng Theo Điểm)

```bash
# Leaderboard (Bảng Xếp Hạng)
ZADD leaderboard:game1 1500 "player:alice"
ZADD leaderboard:game1 2300 "player:bob"
ZADD leaderboard:game1 1800 "player:carol"

# Top 3 players
ZREVRANGE leaderboard:game1 0 2 WITHSCORES
# → [("player:bob", 2300), ("player:carol", 1800), ("player:alice", 1500)]

# Hạng của alice (0-indexed, 0 = hạng 1)
ZREVRANK leaderboard:game1 "player:alice"  # → 2 (hạng 3)

# Players có điểm từ 1000 đến 2000
ZRANGEBYSCORE leaderboard:game1 1000 2000  # → [alice, carol]
```

#### Streams — Luồng Dữ Liệu (Redis 5.0+)

```bash
# Append messages (Thêm tin nhắn vào stream)
XADD orders:stream * order_id ORD-001 product_id P-123 qty 2
XADD orders:stream * order_id ORD-002 product_id P-456 qty 1

# Consumer Group (Nhóm Người Tiêu Dùng) — Xử lý song song
XGROUP CREATE orders:stream processors $ MKSTREAM
XREADGROUP GROUP processors worker1 COUNT 10 STREAMS orders:stream >
```

---

## 🔵 Memcached — Phân Tích Chi Tiết

### Kiến Trúc Multi-Threaded (Đa Luồng)

```
┌─────────────────────────────────────────────┐
│           Memcached Process                  │
│                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │Thread 1 │ │Thread 2 │ │Thread N │       │
│  │(CPU 1)  │ │(CPU 2)  │ │(CPU N)  │       │
│  └────┬────┘ └────┬────┘ └────┬────┘       │
│       └───────────┼───────────┘             │
│                   ▼                          │
│       ┌──────────────────────┐              │
│       │    Slab Allocator    │              │
│       │  (Bộ Cấp Phát Khối) │              │
│       │  - Memory organized  │              │
│       │    in fixed-size     │              │
│       │    slabs/chunks      │              │
│       └──────────────────────┘              │
└─────────────────────────────────────────────┘
```

### Slab Allocator (Bộ Cấp Phát Khối Cố Định)

Memcached dùng **slab allocation** để quản lý bộ nhớ hiệu quả hơn:

```
Slab Classes (Lớp Khối):
  Class 1:  80 bytes  → Lưu objects < 80 bytes
  Class 2: 104 bytes  → Lưu objects < 104 bytes
  Class 3: 136 bytes  → Lưu objects < 136 bytes
  ...
  Class N:   1 MB     → Lưu objects lớn nhất

Ưu điểm: Không bị memory fragmentation (Phân Mảnh Bộ Nhớ)
Nhược điểm: Lãng phí nếu object size không khớp với slab class
```

### API Memcached

```python
import pymemcache.client.base as memcache

client = memcache.Client(('localhost', 11211))

# Cơ bản — chỉ hỗ trợ string/bytes
client.set('key', 'value', expire=3600)
client.get('key')                    # → 'value'
client.delete('key')

# Multi-key operations (Thao Tác Nhiều Key)
client.set_many({'k1': 'v1', 'k2': 'v2'})
client.get_many(['k1', 'k2'])        # → {'k1': 'v1', 'k2': 'v2'}

# CAS — Compare-and-Swap (So Sánh Và Hoán Đổi) — tránh race condition
value, cas_token = client.gets('counter')
client.cas('counter', int(value) + 1, cas_token)
```

---

## ⚖️ Khi Nào Chọn Redis?

### ✅ Dùng Redis Cho

#### 1. Session Management (Quản Lý Phiên) Quan Trọng

```
Lý do: Redis có persistence — session không mất khi node restart
       Redis có TTL chính xác per-key
       Nếu dùng Memcached, user bị đăng xuất đột ngột khi node lỗi
```

#### 2. Leaderboard & Ranking (Bảng Xếp Hạng)

```
Lý do: Sorted Sets cho phép:
  - Cập nhật điểm số O(log N)
  - Query top-K O(log N + K)
  - Range queries theo điểm số
  Không thể làm được với Memcached chỉ có strings
```

#### 3. Rate Limiting (Giới Hạn Tốc Độ)

```
Lý do: INCR + EXPIRE là atomic → không cần distributed lock
       Sliding window algorithms với Sorted Sets
```

#### 4. Pub/Sub & Message Queue (Hàng Đợi Tin Nhắn)

```
Lý do: Redis Pub/Sub cho real-time messaging
       Redis Lists/Streams cho lightweight job queue
       Thay thế đơn giản cho SQS trong một số use cases
```

#### 5. Distributed Locks (Khóa Phân Tán) — Redlock Algorithm

```python
# RedLock: Khóa tài nguyên trên nhiều Redis nodes
# Đảm bảo chỉ 1 process thực hiện thao tác tại một thời điểm

lock = redis.set(
    f"lock:{resource_id}",
    unique_id,
    nx=True,       # Only Set if Not eXists (Chỉ Đặt Nếu Chưa Tồn Tại)
    ex=30          # TTL = 30 giây
)
if lock:
    try:
        # Critical section (Vùng Quan Trọng)
        process_order(order_id)
    finally:
        # Chỉ xóa lock nếu chúng ta sở hữu nó
        if redis.get(f"lock:{resource_id}") == unique_id:
            redis.delete(f"lock:{resource_id}")
```

#### 6. Cần High Availability (Tính Sẵn Sàng Cao)

```
Lý do: Redis có automatic failover với Sentinel/Cluster
       Memcached không có failover — node chết = cache miss toàn bộ
```

---

## ⚖️ Khi Nào Chọn Memcached?

### ✅ Dùng Memcached Cho

#### 1. Pure Caching (Bộ Nhớ Đệm Thuần Túy) — Dữ Liệu Có Thể Tái Tạo

```
Lý do: Không cần persistence — nếu cache mất, chỉ cần query lại từ DB
       Đơn giản, overhead thấp hơn Redis
       Phù hợp: HTML page fragments, API responses, computed results
```

#### 2. Multi-Threaded Workloads Cần Nhiều CPU

```
Lý do: Memcached multi-threaded → tận dụng tốt hơn multi-core CPU
       Redis single-threaded → một lõi bottleneck khi có nhiều connections

Kịch bản: Server có 32 cores, cache reads cực kỳ cao
  Memcached: Dùng 32 cores → throughput cao hơn
  Redis: 1 core → tắc nghẽn nếu operations nhiều
```

#### 3. Horizontal Scaling Đơn Giản (Mở Rộng Theo Chiều Ngang)

```
Lý do: Memcached cluster là "dumb" — client-side sharding
       Thêm node → cấu hình lại client → không cần resharding phức tạp như Redis Cluster
       Phù hợp khi đội kỹ thuật muốn giải pháp đơn giản nhất
```

#### 4. Large Static Objects (Đối Tượng Tĩnh Lớn)

```
Lý do: Cache HTML của trang web tĩnh, image thumbnails metadata
       Không cần data structures phức tạp
       Chỉ cần: set key → get key → delete key
```

---

## 🎯 Decision Matrix (Ma Trận Quyết Định)

```
Bạn cần:                          → Chọn
─────────────────────────────────────────────────────────
Persistence (dữ liệu tồn tại)    → Redis
High Availability với failover    → Redis
Leaderboard / ranking             → Redis (Sorted Set)
Pub/Sub messaging                 → Redis
Session management quan trọng     → Redis
Distributed locks                 → Redis
Queues / job processing           → Redis (List/Stream)
Geospatial queries                → Redis (GEO)
Complex data structures           → Redis
─────────────────────────────────────────────────────────
Simple key-value, data throwaway  → Memcached
Maximum throughput, multi-core    → Memcached (đặc thù)
Tối giản, dễ vận hành            → Memcached
Cache HTML, API responses         → Cả hai đều ổn
─────────────────────────────────────────────────────────
Default choice (Lựa chọn mặc định) → Redis
```

---

## 📈 Hiệu Năng Thực Tế (Real-World Performance)

### Benchmark So Sánh

| Operation | Redis | Memcached |
|-----------|-------|-----------|
| GET (đọc đơn) | ~100,000 ops/s | ~150,000 ops/s |
| SET (ghi đơn) | ~80,000 ops/s | ~120,000 ops/s |
| Multi-GET | Tốt | Tốt hơn (multi-threaded) |
| Sorted Set ops | ✅ Hỗ trợ | ❌ Không |

> **Lưu ý thực tế:** Sự khác biệt về hiệu năng thường **không đáng kể** với hầu hết ứng dụng. Chọn dựa trên **feature requirements** (yêu cầu tính năng) hơn là raw throughput (thông lượng thô).

### ElastiCache Node Performance (Hiệu Năng Node ElastiCache)

| Node Type | Memory | Network | Phù Hợp |
|-----------|--------|---------|---------|
| cache.t4g.micro | 0.5 GB | Low | Dev/Test |
| cache.t4g.medium | 3.09 GB | Low-Medium | Small apps |
| cache.r7g.large | 13.07 GB | Up to 12.5 Gbps | Production |
| cache.r7g.4xlarge | 52.82 GB | Up to 25 Gbps | High traffic |
| cache.r7g.16xlarge | 209.55 GB | 25 Gbps | Large scale |

---

## 💡 Anti-Patterns (Chống Mẫu Thiết Kế Sai)

### ❌ Sai: Dùng Redis Làm Primary Database (Database Chính)

```
Vấn đề: Redis là in-memory → dung lượng giới hạn và đắt hơn RDS
        Persistence của Redis không đảm bảo bằng ACID database
Đúng: Redis là caching layer (tầng đệm) — source of truth (nguồn sự thật) 
      vẫn là database phía sau
```

### ❌ Sai: Cache Mọi Thứ Không Suy Nghĩ

```
Vấn đề: Cache data thay đổi liên tục → stale data (dữ liệu cũ)
        Cache key quá chung chung → cache pollution (ô nhiễm cache)
Đúng:   Cache dữ liệu:
        ✅ Đọc nhiều, thay đổi ít (product catalog, config)
        ✅ Expensive to compute (top 100 products, aggregations)
        ❌ Thay đổi mỗi request (realtime stock price per user)
```

### ❌ Sai: Không Đặt TTL

```
Vấn đề: Cache đầy dần, eviction (đẩy ra) ngẫu nhiên
        Stale data tồn tại vô thời hạn
Đúng:   Luôn đặt TTL phù hợp với business logic:
        - Session: 24h
        - Product info: 1h
        - Config: 5 phút
        - Rate limit counters: bằng window size
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Q1: "Redis vs Memcached — khi nào chọn cái nào?"

**Trả lời mẫu:**
> "Tôi hầu như luôn chọn Redis làm default vì bộ tính năng phong phú hơn: persistence cho session quan trọng, Sorted Sets cho leaderboard, Pub/Sub cho real-time features, và automatic failover cho HA. Memcached có lợi thế khi cần multi-threaded throughput tối đa với workload pure key-value đơn giản và không cần HA, nhưng trong thực tế tôi hiếm khi gặp use case đòi hỏi điều đó hơn Redis."

### Q2: "Giải thích Redis data structures và use cases"

**Trả lời mẫu:**
> "Redis có 5+ data types chính: String cho simple caching, Hash cho object storage hiệu quả, List cho queues (hàng đợi), Set cho tag systems và deduplication (loại bỏ trùng lặp), và Sorted Set — quan trọng nhất — cho leaderboards và rate limiting với sorted access trong O(log N). Ngoài ra còn có Streams cho event sourcing nhẹ và Bitmaps cho analytics hiệu quả bộ nhớ."

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
