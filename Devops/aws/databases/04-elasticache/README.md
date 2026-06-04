# Amazon ElastiCache — Bộ Nhớ Đệm Phân Tán Trên AWS

> ElastiCache (Elastic Cache — Bộ Nhớ Đệm Co Giãn) là dịch vụ managed in-memory caching (bộ nhớ đệm trong RAM được quản lý hoàn toàn) của AWS. Hỗ trợ hai engine: **Redis** và **Memcached**, giúp giảm tải đọc từ database, đạt độ trễ sub-millisecond (dưới một mili-giây), và tăng throughput (thông lượng) của ứng dụng.

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Trạng Thái |
|------|----------|-----------|
| [README.md](./README.md) | Tổng quan ElastiCache, Redis vs Memcached, kiến trúc | ✅ |
| [1-redis-vs-memcached.md](./1-redis-vs-memcached.md) | So sánh chi tiết Redis và Memcached, khi nào dùng gì | ✅ |
| [2-redis-cluster.md](./2-redis-cluster.md) | Cluster Mode, Replication Groups, Sharding | ✅ |
| [3-caching-strategies.md](./3-caching-strategies.md) | Lazy Loading, Write-Through, Write-Around, TTL | ✅ |
| [4-persistence.md](./4-persistence.md) | RDB Snapshots, AOF Logging, Backup & Restore | ✅ |
| [5-security.md](./5-security.md) | VPC, Encryption, Auth Tokens, IAM | ✅ |

---

## 🎯 ElastiCache Là Gì?

**Amazon ElastiCache** là dịch vụ managed in-memory data store (kho dữ liệu trong bộ nhớ được quản lý) tương thích với Redis và Memcached. AWS tự động xử lý:
- Provisioning (Cung cấp tài nguyên) & patching (vá lỗi)
- Monitoring (Giám sát) với CloudWatch
- Automatic failover (Chuyển đổi dự phòng tự động)
- Backup & restore (Sao lưu & khôi phục) — chỉ Redis

```
                  ┌─────────────────────────────────┐
                  │         Application Layer        │
                  │      (Tầng Ứng Dụng)             │
                  └──────────────┬──────────────────┘
                                 │
               ┌─────────────────▼─────────────────┐
               │         ElastiCache Cluster        │
               │     (Sub-millisecond Latency)      │
               │                                    │
               │  ┌────────────┐ ┌────────────┐    │
               │  │   Redis    │ │ Memcached  │    │
               │  │  (Primary) │ │  (Node 1)  │    │
               │  └─────┬──────┘ └────────────┘    │
               │        │ Replication               │
               │  ┌─────▼──────┐                   │
               │  │  Replica   │                   │
               │  │ (Read-only)│                   │
               │  └────────────┘                   │
               └──────────────┬────────────────────┘
                              │ Cache Miss
               ┌──────────────▼────────────────────┐
               │           Database Layer           │
               │      (RDS / Aurora / DynamoDB)     │
               └───────────────────────────────────┘
```

---

## ⚡ Tại Sao Cần ElastiCache?

### Vấn Đề Khi Không Có Cache

```
Scenario (Tình Huống):
- 10,000 user đọc cùng 1 trang product (chi tiết sản phẩm)
- Mỗi request → 1 database query
- Database chịu 10,000 queries/giây → Slow, expensive (chậm, tốn kém)
```

### Giải Pháp Với ElastiCache

```
Cache Hit (Truy Cập Cache Thành Công):
- Lần 1: Cache miss → Query DB → Lưu kết quả vào cache
- Lần 2-10,000: Cache hit → Trả về từ RAM → ~0.1ms
- Database chỉ nhận 1 query thay vì 10,000
```

### Lợi Ích Định Lượng

| Chỉ Số | Không Có Cache | Có ElastiCache |
|--------|---------------|----------------|
| Latency đọc | 5-50ms (DB) | < 1ms (RAM) |
| Database load | 100% reads | Giảm 80-95% |
| Throughput | Giới hạn bởi DB | Tăng 10-100x |
| Cost | Tăng instance size | Giảm tổng chi phí |

---

## 🔑 Hai Engine: Redis vs Memcached

### Tổng Quan Nhanh

| Tiêu Chí | Redis | Memcached |
|----------|-------|-----------|
| **Data Structures** (Cấu Trúc Dữ Liệu) | Strings, Lists, Sets, Hashes, Sorted Sets, Streams | Strings only |
| **Persistence** (Bền Vững Dữ Liệu) | ✅ RDB + AOF | ❌ Không |
| **Replication** (Sao Chép) | ✅ Primary/Replica | ❌ Không |
| **Clustering** (Phân Cụm) | ✅ Cluster Mode | ✅ Multi-node |
| **Pub/Sub** | ✅ Có | ❌ Không |
| **Lua Scripting** | ✅ Có | ❌ Không |
| **Transactions** (Giao Dịch) | ✅ MULTI/EXEC | ❌ Không |
| **Backup** (Sao Lưu) | ✅ Tự động | ❌ Không |

### Chọn Redis Khi Nào?
- Cần **persistence** (dữ liệu tồn tại khi restart)
- Cần **advanced data types** (Sorted Sets cho leaderboard, Lists cho queues)
- Cần **replication & HA** (High Availability — Tính Sẵn Sàng Cao)
- Cần **pub/sub** (publish/subscribe — xuất bản/đăng ký)
- Session caching quan trọng, không muốn mất

### Chọn Memcached Khi Nào?
- Chỉ cần **simple key-value caching** (bộ nhớ đệm cặp khóa-giá trị đơn giản)
- **Multi-threaded** (Đa Luồng) — muốn tận dụng nhiều CPU core
- Không cần HA/persistence
- Scale out bằng cách thêm node đơn giản

> **Khuyến nghị:** Trong hầu hết use cases, **Redis** là lựa chọn mặc định vì bộ tính năng phong phú hơn.

---

## 🏗️ Kiến Trúc ElastiCache Redis

### Mode 1: Standalone Node (Nút Độc Lập)

```
┌─────────────────────────┐
│   ElastiCache Node      │
│   (Single instance)     │
│   ┌─────────────────┐   │
│   │   Redis Data    │   │
│   │   (RAM only)    │   │
│   └─────────────────┘   │
└─────────────────────────┘
Dùng cho: Dev/Test, non-critical caching
Nhược điểm: Single Point of Failure (Điểm Lỗi Duy Nhất)
```

### Mode 2: Replication Group (Nhóm Sao Chép) — Không Có Cluster Mode

```
┌──────────────────────────────────────────┐
│          Replication Group               │
│                                          │
│  ┌───────────────┐  ┌───────────────┐   │
│  │  Primary Node │  │  Replica Node │   │
│  │   (Read/Write)│  │  (Read-only)  │   │
│  └───────┬───────┘  └───────▲───────┘   │
│          │   Asynchronous   │           │
│          └──── Replication ─┘           │
│                                         │
│  Endpoints (Điểm Truy Cập):             │
│  - Primary Endpoint → Write operations  │
│  - Reader Endpoint  → Read operations   │
└──────────────────────────────────────────┘
```

### Mode 3: Cluster Mode Enabled (Chế Độ Cluster Được Bật)

```
┌─────────────────────────────────────────────────────┐
│                  Redis Cluster                       │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   Shard 1    │  │   Shard 2    │  │  Shard 3   │ │
│  │  Slots 0-    │  │  Slots 5461- │  │Slots 10923-│ │
│  │  5460        │  │  10922       │  │16383       │ │
│  │  P + 2R      │  │  P + 2R      │  │  P + 2R    │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                      │
│  P = Primary (Nút Chính)                            │
│  R = Replica (Nút Sao Chép — Read-only)             │
│  Slots = Hash Slots (Khe Băm — 0 to 16383)          │
└─────────────────────────────────────────────────────┘
```

---

## 📊 Use Cases Phổ Biến (Trường Hợp Sử Dụng Phổ Biến)

### 1. Database Query Caching (Cache Kết Quả Truy Vấn Database)

```python
def get_product(product_id):
    cache_key = f"product:{product_id}"
    
    # Thử đọc từ cache trước
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit
    
    # Cache miss → truy vấn database
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    
    # Lưu vào cache với TTL (Time-To-Live — Thời Gian Tồn Tại) 1 giờ
    redis.setex(cache_key, 3600, json.dumps(product))
    return product
```

### 2. Session Management (Quản Lý Phiên Người Dùng)

```python
# Lưu session user sau khi đăng nhập
session_data = {
    "user_id": 12345,
    "email": "user@example.com",
    "roles": ["admin", "editor"],
    "login_time": "2026-05-15T10:00:00"
}
redis.setex(f"session:{session_token}", 86400, json.dumps(session_data))

# Đọc lại session
session = redis.get(f"session:{session_token}")
```

### 3. Rate Limiting (Giới Hạn Tốc Độ)

```python
def check_rate_limit(user_id, limit=100, window=60):
    key = f"rate:{user_id}:{int(time.time() // window)}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count <= limit
```

### 4. Leaderboard (Bảng Xếp Hạng) với Sorted Sets

```python
# Cập nhật điểm số
redis.zadd("leaderboard:global", {user_id: score})

# Top 10 người chơi (điểm cao nhất)
top_players = redis.zrevrange("leaderboard:global", 0, 9, withscores=True)

# Hạng của một người chơi
rank = redis.zrevrank("leaderboard:global", user_id)
```

### 5. Pub/Sub (Xuất Bản/Đăng Ký) cho Real-time Messaging

```python
# Publisher (Bên Xuất Bản)
redis.publish("notifications:user:12345", json.dumps({
    "type": "order_shipped",
    "order_id": "ORD-789"
}))

# Subscriber (Bên Đăng Ký)
pubsub = redis.pubsub()
pubsub.subscribe("notifications:user:12345")
for message in pubsub.listen():
    process_notification(message["data"])
```

---

## 🌐 ElastiCache Trong Kiến Trúc AWS

### Kiến Trúc Điển Hình (Typical Architecture)

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                    VPC (Virtual Private Cloud)       │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │           Public Subnet (Mạng Con Công Khai) │    │
│  │  ┌─────────────────┐                        │    │
│  │  │  Load Balancer  │                        │    │
│  │  │  (ALB/NLB)      │                        │    │
│  │  └────────┬────────┘                        │    │
│  └───────────┼─────────────────────────────────┘    │
│              │                                       │
│  ┌───────────▼─────────────────────────────────┐    │
│  │         Private Subnet (Mạng Con Riêng Tư)   │    │
│  │                                              │    │
│  │  ┌────────────────┐  ┌──────────────────┐   │    │
│  │  │   App Servers  │  │   ElastiCache    │   │    │
│  │  │  (EC2/ECS/EKS) │◄─┤   Redis Cluster  │   │    │
│  │  └───────┬────────┘  └──────────────────┘   │    │
│  │          │                                  │    │
│  │  ┌───────▼────────┐                         │    │
│  │  │   RDS / Aurora │                         │    │
│  │  │   (Database)   │                         │    │
│  │  └────────────────┘                         │    │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

> **Quy tắc bảo mật:** ElastiCache luôn nằm trong **private subnet** (mạng con riêng tư), không bao giờ expose ra internet.

---

## 🔧 Tạo ElastiCache Cluster — Ví Dụ Terraform

```hcl
# Subnet Group cho ElastiCache (Nhóm Mạng Con)
resource "aws_elasticache_subnet_group" "main" {
  name       = "redis-subnet-group"
  subnet_ids = var.private_subnet_ids
}

# Replication Group (Nhóm Sao Chép) — Redis với HA
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "my-redis-cluster"
  description          = "Redis cluster cho production"

  node_type            = "cache.r7g.large"  # Instance type
  num_cache_clusters   = 3                  # 1 primary + 2 replicas
  
  engine               = "redis"
  engine_version       = "7.1"
  port                 = 6379

  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]

  # Bảo mật
  at_rest_encryption_enabled  = true   # Mã hóa dữ liệu khi lưu trữ
  transit_encryption_enabled  = true   # Mã hóa khi truyền tải (TLS)
  auth_token                  = var.redis_auth_token  # Mật khẩu xác thực

  # Backup (Sao Lưu)
  snapshot_retention_limit = 7         # Giữ 7 ngày backup
  snapshot_window          = "03:00-04:00"

  # Maintenance Window (Cửa Sổ Bảo Trì)
  maintenance_window = "sun:04:00-sun:05:00"

  automatic_failover_enabled = true    # Bật tự động chuyển đổi dự phòng

  tags = { Environment = "production" }
}
```

---

## 📋 Node Types (Loại Node) & Sizing (Định Cỡ)

### Các Họ Instance

| Họ | Đặc Điểm | Phù Hợp |
|----|----------|---------|
| **cache.t4g** | Burstable (Có Khả Năng Tăng Đột Biến), ARM | Dev/Test, workload nhỏ |
| **cache.r7g** | Memory-optimized (Tối Ưu Bộ Nhớ), ARM | Production Redis |
| **cache.m7g** | Balanced (Cân Bằng) CPU/Memory | General purpose |

### Ước Tính Kích Thước

```
Công thức cơ bản:
  Memory cần = (Kích thước 1 object × Số lượng objects) / Hit Rate × Overhead factor

Ví dụ:
  - 1 million sessions × 2KB/session = 2GB raw data
  - Overhead Redis ~1.5x = 3GB
  - Chọn cache.r7g.large (13.07 GB) để có buffer
```

---

## 🔗 Điều Hướng Nhanh

| Chủ Đề | File |
|--------|------|
| Redis vs Memcached — So sánh chi tiết | [1-redis-vs-memcached.md](./1-redis-vs-memcached.md) |
| Cluster Mode & Replication | [2-redis-cluster.md](./2-redis-cluster.md) |
| Chiến lược cache (Lazy Loading, Write-Through) | [3-caching-strategies.md](./3-caching-strategies.md) |
| Persistence — RDB & AOF | [4-persistence.md](./4-persistence.md) |
| Bảo mật — VPC, Encryption, Auth | [5-security.md](./5-security.md) |

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Redis vs Memcached — khi nào chọn loại nào?**
2. **Giải thích Lazy Loading vs Write-Through caching — trade-offs?**
3. **Cluster Mode enabled vs disabled — khác nhau gì?**
4. **Làm thế nào tránh cache stampede (Cơn Lũ Cache)?**
5. **TTL (Time-To-Live) là gì? Cách đặt TTL phù hợp?**
6. **Cache invalidation (Vô Hiệu Hóa Cache) — các chiến lược?**

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
