# Setup NoSQL Database trên Cloud — Chi Tiết & Lưu Ý

## 1. AWS DynamoDB

### Khái Niệm Cốt Lõi

```
Table Structure:
  Primary Key = Partition Key (PK) + Sort Key (SK) [optional]
  
  Ví dụ thiết kế bảng Users-Orders (Single Table Design):
  
  PK              | SK                | Attributes
  USER#u123       | PROFILE           | name, email, created_at
  USER#u123       | ORDER#o456        | total, status, created_at
  USER#u123       | ORDER#o789        | total, status, created_at
  ORDER#o456      | ITEM#i001         | product_id, qty, price
  ORDER#o456      | ITEM#i002         | product_id, qty, price
```

### Capacity Modes

```
On-Demand Mode:
  - Trả theo request: $1.25/triệu write, $0.25/triệu read (us-east-1)
  - Không cần estimate
  - Tốt cho: traffic không đều, new applications
  - ⚠️ Đắt hơn Provisioned nếu traffic cao và đều

Provisioned Mode:
  - Đặt trước WCU (Write Capacity Unit) và RCU (Read Capacity Unit)
  - 1 WCU = 1 write ≤ 1KB/s
  - 1 RCU = 1 strongly consistent read ≤ 4KB/s (hoặc 2 eventually consistent)
  - Bật Auto Scaling: min/max/target utilization (khuyến nghị 70%)
  - ⚠️ Throttling xảy ra khi vượt provisioned capacity → retry với backoff
  
Công thức tính RCU/WCU:
  WCU = (writes/s) × ceil(item_size_KB / 1)
  RCU = (reads/s)  × ceil(item_size_KB / 4)  [eventually consistent]
  RCU = (reads/s)  × ceil(item_size_KB / 4) × 2  [strongly consistent]
```

### Partition Key — Lỗi Phổ Biến Nhất

```
⚠️ HOT PARTITION là vấn đề nghiêm trọng nhất DynamoDB

Ví dụ BAD partition key:
  user_id = "admin"    → 80% traffic vào 1 partition
  date = "2026-04-30"  → tất cả insert hôm nay vào 1 partition
  status = "PENDING"   → workload xử lý order queue

Ví dụ GOOD partition key:
  user_id (UUID)       → phân tán đều
  order_id (UUID)      → phân tán đều
  device_id#timestamp  → sharding theo device

Kỹ thuật Write Sharding (nếu buộc dùng hot key):
  Thay vì pk = "COUNTER"
  Dùng pk = "COUNTER#" + random(1, 10)
  Khi đọc: query all 10 shards và aggregate
```

### Global Secondary Index (GSI) & Local Secondary Index (LSI)

```
GSI (Global Secondary Index):
  - Partition key khác với base table
  - Có RCU/WCU riêng (hoặc kế thừa từ On-Demand)
  - Có thể tạo sau khi table đã tồn tại
  - Eventual consistency only
  - Tối đa 20 GSI/table

LSI (Local Secondary Index):
  - Same partition key, sort key khác
  - Phải tạo lúc tạo table (KHÔNG thể thêm sau)
  - Dùng chung RCU/WCU với base table
  - Hỗ trợ strongly consistent reads
  - Tối đa 5 LSI/table
  - ⚠️ LSI partition size limit: 10GB (partition key value)

Khi nào dùng GSI:
  Cần query theo attribute khác partition key
  Ví dụ: bảng Orders có PK=order_id
  Cần query theo customer_id → tạo GSI với PK=customer_id
```

### DynamoDB Streams & Event-Driven

```
DynamoDB Streams:
  - Capture mọi thay đổi (INSERT, MODIFY, REMOVE)
  - Dùng kết hợp với Lambda trigger
  - Retention: 24 giờ
  
Usecase:
  - Invalidate cache khi data thay đổi
  - Sync sang Elasticsearch/OpenSearch
  - Audit log
  - Cross-region replication (hoặc Global Tables)

Global Tables:
  - Multi-region active-active
  - Tự động sync giữa các region
  - Eventual consistency giữa regions
  - ⚠️ Conflict resolution: "last writer wins" (theo timestamp)
```

---

## 2. Azure Cosmos DB

### Chọn API Phù Hợp

```
Core (SQL) API:
  - Document model, query bằng SQL-like syntax
  - Tốt cho ứng dụng mới xây trên Azure
  
MongoDB API:
  - Compatible với MongoDB driver (4.x)
  - Migration từ MongoDB lên Azure dễ dàng
  - ⚠️ Không support 100% MongoDB features (check compatibility matrix)

Cassandra API:
  - Compatible với Cassandra CQL driver
  - Tốt cho time-series, wide-column patterns

Gremlin API:
  - Graph database
  - Tốt cho social network, recommendation

Table API:
  - Compatible với Azure Table Storage
  - Migration từ Azure Tables lên Cosmos DB
```

### Request Units (RU) — Quan Trọng Nhất

```
RU là đơn vị measure tất cả operations:
  Read  1KB item = 1 RU
  Write 1KB item = 5 RU
  Query (scan)   = 1-1000+ RU tùy query complexity

Ví dụ estimate RU:
  - App đọc 100 items/s, mỗi item 2KB:
    RU/s = 100 × ceil(2/1) = 200 RU/s read
  
  - App ghi 50 items/s, mỗi item 1KB:
    RU/s = 50 × 5 = 250 RU/s write
  
  - Total provisioned: 200 + 250 = 450 RU/s (+ buffer 30%) = ~600 RU/s

Provisioning options:
  Manual Provisioned:  đặt cố định RU/s (min 400 RU)
  Autoscale:           max RU, tự scale 10%-100%, phí = max provisioned × $
  Serverless:          trả theo RU dùng thực tế, tốt cho dev/test

⚠️ Khi vượt provisioned RU → 429 Too Many Requests → cần retry
   SDK Cosmos tự retry nhưng app cần handle gracefully
```

### Partition Key — Critical Decision

```
Cosmos DB shards data theo Partition Key (giống DynamoDB)

Tiêu chí chọn Partition Key TỐT:
  ✓ Cardinality cao (nhiều giá trị khác nhau)
  ✓ Phân tán đều RU consumption
  ✓ Xuất hiện trong phần lớn queries (tránh cross-partition query)
  ✓ Immutable (không thay đổi sau khi tạo item)

Ví dụ:
  E-commerce Orders: partitionKey = "/customerId"
    → mỗi customer là 1 logical partition
    → query "orders by customer" = single-partition (rẻ)
    → ⚠️ nếu 1 customer có >20GB orders = hot partition
  
  IoT telemetry: partitionKey = "/deviceId"
  User profiles:  partitionKey = "/userId"
  
Synthetic Partition Key (khi không có key tốt):
  partitionKey = userId + "-" + month  ("u123-2026-04")
  → phân tán theo tháng, tránh một user chiếm quá nhiều

⚠️ Không thể thay đổi partition key sau khi tạo container
   (phải migrate data sang container mới)
```

### Consistency Levels

```
Strong:
  - Đọc luôn thấy write mới nhất
  - Chỉ available trong 1 region hoặc khi có 1 write region
  - Chi phí: 2x RU cho reads
  - Latency: cao nhất

Bounded Staleness:
  - Đảm bảo đọc data cũ hơn tối đa N operations hoặc T giây
  - Tốt cho: global distribution với consistency gần-Strong
  - Chi phí: 2x RU cho reads ở single region

Session (DEFAULT - khuyến nghị):
  - Consistency trong cùng 1 session/client
  - Client luôn đọc được write của chính nó
  - Tốt cho: hầu hết ứng dụng
  - Chi phí: 1x RU

Consistent Prefix:
  - Đảm bảo thứ tự writes, không đọc out-of-order
  - Không đảm bảo xem write mới nhất

Eventual:
  - Rẻ nhất, nhanh nhất
  - Không đảm bảo thứ tự
  - Tốt cho: counter, aggregate không critical

Chọn consistency level:
  Financial data, inventory     → Strong hoặc Bounded Staleness
  User profile, order history   → Session
  Product catalog, feed         → Consistent Prefix hoặc Eventual
  Analytics, counters           → Eventual
```

---

## 3. ElastiCache for Redis (AWS)

### Cluster Modes

```
Cluster Mode Disabled (Replication Group):
  ┌────────────────────────────┐
  │  Primary (1 node, write)   │
  │  Replica 1 (read)          │
  │  Replica 2 (read)          │
  └────────────────────────────┘
  - Tất cả data trên 1 node
  - Tối đa 5 replicas
  - Multi-AZ với automatic failover
  - Phù hợp: dataset nhỏ (<100GB), simple caching

Cluster Mode Enabled:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
  │ P + 2R   │  │ P + 2R   │  │ P + 2R   │
  └──────────┘  └──────────┘  └──────────┘
  - Data sharded across shards
  - Tối đa 500 shards × 5 replicas
  - Scale horizontally
  - Phù hợp: dataset lớn, throughput cao

Serverless (mới nhất):
  - Không cần chọn node type
  - Auto-scale storage và compute
  - Trả theo ECPU + data storage
  - ⚠️ Đắt hơn nếu workload đều và cao
```

### Cấu hình Quan Trọng

```
maxmemory-policy (eviction):
  allkeys-lru    → evict key ít dùng nhất (khuyến nghị cho cache)
  volatile-lru   → evict key có TTL ít dùng nhất
  allkeys-lfu    → evict key ít tần suất nhất (Redis 4+)
  noeviction     → trả error khi đầy (dùng khi Redis là primary store)
  
  ⚠️ Default là noeviction → cache đầy sẽ trả lỗi!
     Với pure cache: dùng allkeys-lru

maxmemory = 75% available RAM
  Để lại 25% cho overhead, AOF rewrite, forking

save (persistence):
  Tắt hoàn toàn nếu Redis chỉ dùng làm cache (tăng performance)
  Bật AOF nếu Redis lưu session hoặc data quan trọng:
    appendonly yes
    appendfsync everysec  # balance giữa performance và durability

lazyfree-lazy-eviction yes  # evict async, không block
lazyfree-lazy-expire yes    # expire async
```

### Kết Nối từ Application

```python
# Redis Cluster Mode Enabled
from redis.cluster import RedisCluster

rc = RedisCluster(
    startup_nodes=[{"host": "xxx.cache.amazonaws.com", "port": 6379}],
    decode_responses=True,
    skip_full_coverage_check=True,
    ssl=True,                    # Bắt buộc production
    ssl_certfile=None,
    password=get_secret("redis/password"),
    socket_timeout=0.5,          # Fail fast
    socket_connect_timeout=0.5,
    retry_on_timeout=True,
    max_connections=50           # Per node
)

# ⚠️ Với Cluster Mode: keys phải trong cùng slot nếu dùng multi-key ops
# Dùng hash tags: {user}:session và {user}:profile → cùng slot
```

---

## 4. Azure Cache for Redis

### Tiers

```
Basic:  1 node, no SLA, dev/test only
Standard: 2 nodes (primary + replica), SLA 99.9%
Premium:  Cluster, Persistence, Private VNet, Geo-replication
Enterprise: Redis Stack (Search, JSON, Bloom), 99.99% SLA
Enterprise Flash: NVMe + DRAM, dataset lớn hơn RAM

Production rule:
  Standard C1 (1GB)  → nhỏ
  Standard C4 (6GB)  → vừa
  Premium P1 (6GB)   → cần cluster hoặc VNet
```

### Geo-Replication (Premium+)

```
Active geo-replication (Enterprise):
  - Multi-region active-active
  - Eventual consistency giữa regions
  - Mỗi region serve local reads/writes

Passive geo-replication (Premium):
  - Primary region nhận writes
  - Linked cache ở region khác read-only
  - Failover: manual (unlink và promote)
```

---

## 5. Lưu Ý Chung Khi Setup NoSQL Cloud

```
✅ DynamoDB:
   - Bật Point-in-Time Recovery (PITR) ngay khi tạo table
   - Bật encryption at rest (mặc định từ 2017)
   - Bật DynamoDB Streams nếu cần event-driven
   - Dùng Condition Expressions để tránh lost updates
   - Không dùng Scan nếu table lớn (quét toàn bộ = tốn RU)
   - Bật TTL cho data có thời hạn (session, cache, log)

✅ Cosmos DB:
   - Bật "Continuous backup" (PITR 30 ngày) thay vì Periodic
   - Chọn consistency level phù hợp (Session là đủ cho hầu hết)
   - Đặt TTL trên container cho data tạm thời
   - Monitor "Normalized RU Consumption" — nếu > 100% = throttling
   - Dùng SDK retry policy: CosmosClientOptions.MaxRetryAttempts

✅ ElastiCache / Azure Cache for Redis:
   - Không để Redis tiếp xúc Internet (luôn trong private subnet)
   - Bật AUTH (password) + TLS
   - Monitor: CurrConnections, Evictions, CacheMisses
   - Evictions tăng đột biến → tăng instance size hoặc giảm TTL
   - Không lưu large blobs (>100KB) → gây serialization latency
   - Key naming convention: "entity:id:field" (vd: "user:123:session")
```
