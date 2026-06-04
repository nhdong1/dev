# Redis Cluster Mode, Replication Groups & Sharding — Phân Cụm Redis

> Redis Cluster (Cụm Redis) cho phép phân tán dữ liệu trên nhiều node để vượt giới hạn RAM của một máy chủ đơn lẻ và đạt throughput (thông lượng) cao hơn. Hiểu rõ Replication Groups (Nhóm Sao Chép), Sharding (Phân Mảnh Dữ Liệu), và Failover (Chuyển Đổi Dự Phòng) là kỹ năng then chốt khi thiết kế ElastiCache trên AWS.

---

## 🏗️ Các Chế Độ Triển Khai ElastiCache Redis

### Mode 1: Single Node (Nút Đơn) — Không Có HA

```
┌──────────────────────────────────┐
│       Single Cache Node          │
│   Primary (Chính, Read/Write)    │
│   - Không có replica             │
│   - Node chết = cache mất        │
│   - Phù hợp: Dev/Test only       │
└──────────────────────────────────┘

Chi phí:    Thấp nhất
Availability: Thấp (node chết → downtime)
Dung lượng: Giới hạn bởi 1 node
```

### Mode 2: Replication Group — Cluster Mode Disabled (Nhóm Sao Chép — Tắt Chế Độ Cluster)

```
┌─────────────────────────────────────────────────────┐
│             Replication Group                        │
│          (Cluster Mode: DISABLED)                    │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │               Shard duy nhất                 │   │
│  │                                              │   │
│  │  ┌────────────────┐   Async Replication     │   │
│  │  │ Primary Node   │──────────────────┐      │   │
│  │  │ (Read + Write) │                  ▼      │   │
│  │  └────────────────┘  ┌─────────────────┐   │   │
│  │                      │  Replica Node 1 │   │   │
│  │                      │  (Read-only)    │   │   │
│  │                      └─────────────────┘   │   │
│  │                      ┌─────────────────┐   │   │
│  │                      │  Replica Node 2 │   │   │
│  │                      │  (Read-only)    │   │   │
│  │                      └─────────────────┘   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  Endpoints (Điểm Truy Cập):                         │
│  ├─ Primary Endpoint: my-redis.abc.cache.amazonaws.com│
│  └─ Reader Endpoint:  my-redis-ro.abc.cache.amazonaws.com│
└─────────────────────────────────────────────────────┘

Phù hợp:   Dữ liệu < RAM của 1 node, cần HA và read scaling
Giới hạn:  Toàn bộ dataset phải fit trong 1 node
```

### Mode 3: Cluster Mode Enabled (Bật Chế Độ Cluster) — Sharding

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Redis Cluster (Cluster Mode: ENABLED)              │
│                                                                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │
│  │   Shard 1    │    │   Shard 2    │    │   Shard 3    │           │
│  │ Slots 0-5460 │    │Slots 5461-   │    │Slots 10923-  │           │
│  │              │    │10922         │    │16383         │           │
│  │  [Primary]   │    │  [Primary]   │    │  [Primary]   │           │
│  │  [Replica 1] │    │  [Replica 1] │    │  [Replica 1] │           │
│  │  [Replica 2] │    │  [Replica 2] │    │  [Replica 2] │           │
│  └──────────────┘    └──────────────┘    └──────────────┘           │
│                                                                       │
│  Configuration Endpoint (Điểm Truy Cập Cấu Hình):                   │
│  my-redis-cluster.clustercfg.abc.cache.amazonaws.com:6379            │
│                                                                       │
│  Dung lượng: 3 × RAM_per_node (ví dụ: 3 × 13GB = 39GB)             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🔢 Hash Slots — Cơ Chế Phân Mảnh Dữ Liệu

### Cách Redis Cluster Phân Chia Key

Redis Cluster dùng **hash slots** (khe băm) — 16,384 slots tổng, chia đều cho các shard:

```
Tính toán slot cho một key:
  slot = CRC16(key) mod 16384

Ví dụ:
  CRC16("user:1001") mod 16384 = 7842  → Shard 2 (slots 5461-10922)
  CRC16("order:ABC") mod 16384 = 2105  → Shard 1 (slots 0-5460)
  CRC16("session:X") mod 16384 = 14200 → Shard 3 (slots 10923-16383)
```

### Hash Tags — Buộc Keys Cùng Shard

```bash
# Vấn đề: Multi-key operations (MGET, MSET, Transactions) yêu cầu
#          tất cả keys phải trên cùng 1 shard

# ❌ Sai — có thể ở khác shard
MGET "user:1001" "order:1001"  # Có thể fail trong cluster mode

# ✅ Đúng — Hash Tags {} buộc dùng cùng hash slot
MGET "{user:1001}.profile" "{user:1001}.orders"
# CRC16 chỉ tính phần trong {}, nên cả hai key → cùng slot → cùng shard
```

---

## 🔄 Replication — Cơ Chế Sao Chép

### Asynchronous Replication (Sao Chép Bất Đồng Bộ)

```
Primary Node (Nút Chính)
       │
       │ Ghi lệnh vào replication buffer
       │ (Vùng Đệm Sao Chép)
       │
       ├──────► Replica 1 — Nhận và áp dụng sau vài ms
       │
       └──────► Replica 2 — Nhận và áp dụng sau vài ms

Ý nghĩa: Primary không chờ replica xác nhận → thấp latency
Rủi ro:  Nếu primary crash trước khi replica nhận → mất vài lệnh cuối
         → Eventual Consistency (Nhất Quán Cuối Cùng), không phải strong consistency
```

### Replication Process — Quá Trình Sao Chép Lần Đầu

```
Bước 1: Replica kết nối với Primary và gửi PSYNC
Bước 2: Primary tạo RDB Snapshot (ảnh chụp) và gửi cho Replica
Bước 3: Trong khi truyền RDB, Primary ghi tiếp vào replication buffer
Bước 4: Replica load RDB xong → nhận tiếp replication buffer để sync
Bước 5: Sau đó: streaming replication (sao chép liên tục)
```

### Replica Lag (Độ Trễ Sao Chép)

```python
# Kiểm tra trạng thái replication
redis-cli INFO replication

# Output quan trọng:
# role: master
# connected_slaves: 2
# slave0: ip=10.0.1.10,port=6379,state=online,offset=12345,lag=0
# slave1: ip=10.0.1.11,port=6379,state=online,offset=12340,lag=1
#
# lag=0: Replica đồng bộ hoàn toàn
# lag=1: Replica trễ 1 second → cần theo dõi
```

---

## 🛡️ Automatic Failover — Chuyển Đổi Dự Phòng Tự Động

### Quá Trình Failover Khi Primary Chết

```
Bình thường:
  Primary (Active) ←── App writes ──── Application
       │
       └──── Async Replication ────► Replica (Standby)

Khi Primary chết:
  
  Bước 1: ElastiCache phát hiện primary không phản hồi
          (sau khoảng 10-30 giây tùy cấu hình)
  
  Bước 2: Replica có replication offset cao nhất được chọn
          làm Primary mới
  
  Bước 3: DNS của Primary Endpoint được cập nhật
          trỏ đến Replica cũ (nay là Primary mới)
  
  Bước 4: Application tự động reconnect đến Primary mới
          (nếu dùng ElastiCache endpoint, không phải IP)

Thời gian failover: ~1-3 phút
Dữ liệu có thể mất: Vài lệnh cuối chưa kịp replicate (asynchronous)
```

### Failover Đa AZ (Multi-AZ — Đa Vùng Sẵn Sàng)

```
AZ-a (Vùng Sẵn Sàng a)    AZ-b (Vùng Sẵn Sàng b)
┌──────────────────┐        ┌──────────────────┐
│                  │        │                  │
│   Primary Node   │───────►│  Replica Node    │
│   (Read/Write)   │  Async │  (Read-only)     │
│                  │  Repl  │                  │
└──────────────────┘        └──────────────────┘

Nếu AZ-a bị lỗi:
  → Replica ở AZ-b được promote thành Primary
  → App kết nối lại qua DNS endpoint (không đổi)
  → Downtime: ~1-3 phút

Khuyến nghị: Luôn bật Multi-AZ cho production
```

---

## 📐 Sharding Strategies — Chiến Lược Phân Mảnh

### Online Resharding (Phân Mảnh Lại Trực Tuyến)

ElastiCache hỗ trợ thêm/bớt shard **không downtime** (không gián đoạn):

```
Trước: 3 shards, mỗi shard ~5.46K slots
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │
│ 0-5460   │  │5461-10922│  │10923-16383│
└──────────┘  └──────────┘  └──────────┘

Thêm 1 shard (Online Resharding — Phân Lại Trực Tuyến):
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard 4  │
│ 0-4095   │  │4096-8191 │  │8192-12287│  │12288-16383│
└──────────┘  └──────────┘  └──────────┘  └──────────┘

Quá trình:
1. AWS tạo Shard 4 mới (rỗng)
2. Migrate (Di Chuyển) hash slots từ các shard cũ sang Shard 4
3. Trong khi migrate, cả hai shard đều có thể phục vụ
4. Sau migrate xong, Shard 4 nhận requests chính thức
```

### Khi Nào Cần Thêm Shard?

```
Dấu hiệu cần thêm shard:
  ├─ Memory usage > 75% trên bất kỳ shard nào
  ├─ CPU usage > 90% liên tục
  ├─ Network bandwidth bão hòa
  └─ Latency tăng do memory pressure và eviction
```

---

## 🔧 Cấu Hình AWS — Terraform

### Cluster Mode Disabled (Không Bật Cluster) — Replication Group

```hcl
resource "aws_elasticache_replication_group" "redis_ha" {
  replication_group_id = "redis-ha-production"
  description          = "Redis HA, cluster mode OFF, 1 primary + 2 replicas"

  # Engine
  engine         = "redis"
  engine_version = "7.1"
  node_type      = "cache.r7g.large"   # 13.07 GB RAM
  port           = 6379

  # Replication — 1 primary + 2 replicas
  num_cache_clusters = 3

  # Multi-AZ
  multi_az_enabled           = true
  automatic_failover_enabled = true

  # Network
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  # Security
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token

  # Backup
  snapshot_retention_limit = 7
  snapshot_window          = "02:00-03:00"
  maintenance_window       = "sun:03:00-sun:04:00"

  # Parameter Group (Nhóm Tham Số)
  parameter_group_name = aws_elasticache_parameter_group.redis7.name
}
```

### Cluster Mode Enabled — Sharded Cluster

```hcl
resource "aws_elasticache_replication_group" "redis_cluster" {
  replication_group_id = "redis-cluster-production"
  description          = "Redis Cluster Mode, 3 shards × (1P + 2R)"

  engine         = "redis"
  engine_version = "7.1"
  node_type      = "cache.r7g.large"
  port           = 6379

  # Cluster Mode ON — sharding
  cluster_mode {
    num_node_groups         = 3   # Số shards (nhóm node)
    replicas_per_node_group = 2   # Số replicas mỗi shard
  }

  # Multi-AZ bắt buộc khi cluster mode
  multi_az_enabled           = true
  automatic_failover_enabled = true

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token

  snapshot_retention_limit = 7
  snapshot_window          = "02:00-03:00"
}
```

### Parameter Group (Nhóm Tham Số) Tùy Chỉnh

```hcl
resource "aws_elasticache_parameter_group" "redis7" {
  family = "redis7"
  name   = "redis7-production"

  # Bật cluster mode
  parameter {
    name  = "cluster-enabled"
    value = "yes"
  }

  # Maxmemory Policy (Chính Sách Khi Đầy Bộ Nhớ)
  # allkeys-lru: Đẩy ra key ít dùng nhất (LRU — Least Recently Used)
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  # Slow log (Nhật Ký Thao Tác Chậm) — log commands > 10ms
  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"  # microseconds
  }

  # Số slow log entries lưu giữ
  parameter {
    name  = "slowlog-max-len"
    value = "128"
  }
}
```

---

## 📊 Maxmemory Policies — Chính Sách Khi Hết RAM

Khi Redis đầy bộ nhớ, cần chính sách để quyết định key nào bị đẩy ra (eviction):

| Policy | Mô Tả | Dùng Khi |
|--------|-------|---------|
| `noeviction` | Từ chối ghi mới khi đầy, trả lỗi | Database, không muốn mất data |
| `allkeys-lru` | Đẩy ra key ít dùng nhất (toàn bộ keys) | Cache thuần túy |
| `volatile-lru` | Đẩy ra key ít dùng nhất **có TTL** | Mix cache + persistent keys |
| `allkeys-lfu` | Đẩy ra key ít **tần suất** nhất (LFU — Least Frequently Used) | Cache với access pattern không đều |
| `volatile-ttl` | Đẩy ra key có TTL ngắn nhất | Session cache |
| `allkeys-random` | Đẩy ra ngẫu nhiên | Không khuyến nghị |

```
Khuyến nghị:
  Cache thuần túy (tất cả keys đều có thể mất): allkeys-lru
  Có một số keys quan trọng không được mất:     volatile-lru
  Sessions với TTL:                             volatile-ttl
  Primary data store (không muốn mất gì):       noeviction
```

---

## 🔍 Giám Sát Cluster — CloudWatch Metrics

### Metrics Quan Trọng (Chỉ Số Quan Trọng)

```
Nhóm 1: Memory (Bộ Nhớ)
  DatabaseMemoryUsagePercentage  → Cảnh báo > 75%
  CurrItems                      → Số items hiện có trong cache
  Evictions                      → Số items bị đẩy ra (nên = 0 hoặc thấp)

Nhóm 2: Performance (Hiệu Năng)
  CacheHits                      → Số lần truy cập cache thành công
  CacheMisses                    → Số lần truy cập cache thất bại
  CacheHitRate = Hits/(Hits+Misses) → Mục tiêu > 90%
  GetLatency                     → P99 latency < 1ms
  SetLatency                     → P99 latency < 1ms

Nhóm 3: Connections (Kết Nối)
  CurrConnections                → Số kết nối hiện tại
  NewConnections                 → Kết nối mới/giây

Nhóm 4: Replication (Sao Chép)
  ReplicationLag                 → Độ trễ replica (giây) → Cảnh báo > 5s
  ReplicationBytes               → Băng thông replication
```

### CloudWatch Alarm Ví Dụ

```hcl
# Cảnh báo khi evictions cao (đẩy ra nhiều items)
resource "aws_cloudwatch_metric_alarm" "redis_evictions" {
  alarm_name          = "redis-high-evictions"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Evictions"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Sum"
  threshold           = 100

  dimensions = {
    CacheClusterId = aws_elasticache_replication_group.redis_cluster.id
  }

  alarm_description = "Evictions cao → cần tăng RAM hoặc giảm TTL"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}

# Cảnh báo khi memory sắp đầy
resource "aws_cloudwatch_metric_alarm" "redis_memory" {
  alarm_name          = "redis-high-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Maximum"
  threshold           = 80

  dimensions = {
    CacheClusterId = aws_elasticache_replication_group.redis_cluster.id
  }

  alarm_description = "Redis memory > 80%, cân nhắc scale up/out"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}
```

---

## 🎯 Kết Luận — Chọn Mode Nào?

```
                    Dữ liệu < RAM 1 node?
                           │
               ┌───────────┴────────────┐
               │ Có                     │ Không
               ▼                        ▼
    Cần HA / Read Scaling?    CLUSTER MODE ENABLED
               │               (Sharding cần thiết)
     ┌─────────┴──────────┐
     │ Có                 │ Không
     ▼                    ▼
REPLICATION GROUP    SINGLE NODE
(Cluster Mode OFF)   (Chỉ Dev/Test)
1 Primary + N Replicas
```

**Khuyến nghị production:**
- Luôn bật **Multi-AZ** và **automatic failover**
- Bắt đầu với **Replication Group** (cluster mode OFF) cho đơn giản
- Chuyển sang **Cluster Mode** khi dataset vượt ~10GB hoặc cần scale write throughput

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
