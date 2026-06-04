# ⚖️ Trade-off Discussions — Thảo Luận Đánh Đổi Kiến Trúc Database

> Phân tích sâu các quyết định kiến trúc quan trọng nhất trong AWS Database — giúp bạn trả lời tự tin khi interviewer hỏi "Tại sao bạn chọn X thay vì Y?"

## Mục Lục

1. [SQL vs NoSQL — Khi Nào Dùng Gì?](#sql-vs-nosql)
2. [RDS vs Aurora — Phân Tích Chi Tiết](#rds-vs-aurora)
3. [DynamoDB vs Aurora — Database Choice Framework](#dynamodb-vs-aurora)
4. [Caching: Redis vs Memcached vs DAX](#caching-redis-vs-memcached-vs-dax)
5. [Consistency vs Availability — CAP Theorem](#consistency-vs-availability)
6. [Vertical Scaling vs Horizontal Scaling](#vertical-vs-horizontal-scaling)
7. [On-Demand vs Provisioned Capacity](#on-demand-vs-provisioned-capacity)
8. [Single-Region vs Multi-Region](#single-region-vs-multi-region)

---

## SQL vs NoSQL

### Khi Nào Chọn SQL (Relational Database — Cơ Sở Dữ Liệu Quan Hệ)

**Chọn SQL khi:**

```
✅ Data có quan hệ phức tạp và thay đổi thường xuyên
   → Sản phẩm có nhiều categories, tags, suppliers

✅ Cần ACID transactions (Giao Dịch ACID) trên nhiều bảng
   → Chuyển tiền: debit account A + credit account B cùng lúc

✅ Access patterns chưa biết trước (ad-hoc queries)
   → Business intelligence, reporting với queries động

✅ Team quen với SQL và relational modeling
   → Giảm learning curve (Đường Cong Học Tập)

✅ Cần complex aggregations (Tổng Hợp Phức Tạp)
   → SUM, GROUP BY, window functions phức tạp
```

**Ví dụ use cases:**
- ERP (Enterprise Resource Planning — Hoạch Định Nguồn Lực Doanh Nghiệp)
- Banking systems (Hệ Thống Ngân Hàng)
- Order management (Quản Lý Đơn Hàng) với complex workflows
- HR systems (Hệ Thống Nhân Sự)

### Khi Nào Chọn NoSQL

**Chọn NoSQL khi:**

```
✅ Access patterns rõ ràng, ít thay đổi
   → "Lấy tất cả orders của user X" — luôn là query này

✅ Cần scale cực cao với low latency
   → Millions requests/giây, < 10ms latency

✅ Schema linh hoạt, thay đổi thường xuyên
   → Event tracking với attributes khác nhau mỗi event type

✅ Hierarchical hoặc document data
   → Product với nested specifications, user profiles

✅ Serverless / event-driven architecture
   → Lambda + DynamoDB pattern
```

**Ví dụ use cases:**
- Gaming (session state, leaderboards)
- E-commerce cart (Giỏ Hàng Thương Mại Điện Tử)
- IoT data collection (Thu Thập Dữ Liệu IoT)
- Real-time bidding (Đấu Giá Thời Gian Thực)
- Content management (Quản Lý Nội Dung)

### Bảng So Sánh Tổng Hợp

| Tiêu Chí | SQL (RDS/Aurora) | NoSQL (DynamoDB) |
|----------|-----------------|-----------------|
| **Data model** | Normalized tables + JOINs | Denormalized, key-value/document |
| **Schema** | Strict schema | Flexible schema |
| **Transactions** | Full ACID | Limited ACID (TransactWrite) |
| **Query flexibility** | Cao — ad-hoc SQL | Thấp — phải biết access pattern trước |
| **Scale** | Vertical chủ yếu | Horizontal unlimited |
| **Latency** | Ms range | Sub-ms range |
| **Learning curve** | Thấp (SQL chuẩn) | Cao (cần học access pattern design) |
| **Cost at scale** | Tăng nhanh | Tăng tuyến tính |

### Câu Trả Lời Hay Cho Phỏng Vấn

> "Việc chọn SQL hay NoSQL phụ thuộc vào ba yếu tố chính: (1) độ phức tạp của data relationships, (2) khả năng dự đoán access patterns, và (3) scale requirements. Với hệ thống financial transactions, tôi luôn chọn Aurora vì ACID là bắt buộc. Với user session data hoặc shopping cart, DynamoDB là lựa chọn tốt hơn vì access pattern đơn giản và cần scale cao."

---

## RDS vs Aurora

### Cả Hai Đều Là SQL — Sự Khác Biệt Là Gì?

Aurora không phải là một database engine mới — nó là MySQL và PostgreSQL được viết lại kiến trúc storage layer để tối ưu cho cloud.

### Kiến Trúc Storage: Điểm Khác Biệt Cốt Lõi

```
RDS Storage Architecture (Kiến Trúc Lưu Trữ RDS):
┌─────────────────────────────────────┐
│ Primary Instance                     │
│  └── EBS Volume (local attachment)  │
│       (replicate synchronously)      │
│           ↓                          │
│ Standby Instance                     │
│  └── EBS Volume (copy)              │
└─────────────────────────────────────┘
→ Write = write to EBS + replicate EBS to standby
→ Replication lag: có thể xảy ra

Aurora Storage Architecture (Kiến Trúc Lưu Trữ Aurora):
┌──────────────────────────────────────────────────┐
│ Writer + Reader instances                         │
│  └── Connected to shared distributed storage     │
│                                                   │
│  Shared Storage Layer:                            │
│  ├── AZ1: copy 1, copy 2                        │
│  ├── AZ2: copy 3, copy 4                        │
│  └── AZ3: copy 5, copy 6                        │
└──────────────────────────────────────────────────┘
→ Write = only redo logs, không cần replicate full data
→ Readers share same storage, no replication lag
```

### So Sánh Chi Tiết

| Tiêu Chí | RDS | Aurora |
|----------|-----|--------|
| **Performance** | Baseline | 3-5x nhanh hơn |
| **Storage auto-scaling** | Có (storage auto-scale) | Có (shared storage tự grow) |
| **Read Replicas tối đa** | 5 | 15 |
| **Replication lag** | Có (async) | Gần 0 (shared storage) |
| **Failover time** | 1-2 phút | < 30 giây |
| **Cross-region replication** | Read Replica cross-region | Global Database (< 1 giây lag) |
| **Supported engines** | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL, PostgreSQL only |
| **Cost** | Base cost | ~20-30% đắt hơn RDS |
| **Storage model** | Pay per GB provisioned | Pay per GB stored + I/O |
| **Backtrack** (Đi Ngược Thời Gian) | Không | Có — rewind table trong phút |
| **Serverless option** | Không | Aurora Serverless v2 |

### Khi Nào Chọn RDS Thay Vì Aurora

```
1. Cần Oracle hoặc SQL Server → Chỉ có trên RDS
2. Budget thấp, traffic nhỏ → RDS rẻ hơn ~20-30%
3. MariaDB → Chỉ có trên RDS
4. Workload đặc thù Oracle/SQL Server stored procedures
5. Team đã có RDS expertise và không muốn migrate
```

### Khi Nào Chọn Aurora Thay Vì RDS

```
1. MySQL hoặc PostgreSQL với traffic đáng kể
2. Cần nhiều Read Replicas (> 5)
3. Cần failover nhanh (< 30 giây)
4. Cần Global Database cho multi-region
5. Cần Aurora Serverless v2 cho unpredictable workload
6. Cần Backtrack cho quick recovery
```

### ROI Analysis (Phân Tích ROI — Return on Investment — Lợi Tức Đầu Tư)

```
Aurora đắt hơn ~20-30% về instance cost
NHƯNG:
- Performance 3-5x tốt hơn → có thể dùng instance nhỏ hơn
- Failover nhanh hơn → ít downtime hơn ($$$)
- Ít Read Replicas cần → mỗi Aurora replica handle nhiều hơn

Kết quả thực tế: Aurora thường rẻ hơn tổng thể khi scale lên
```

---

## DynamoDB vs Aurora

### Framework Quyết Định

```
Câu hỏi 1: Data có quan hệ phức tạp không?
  → Có (nhiều JOINs): Aurora
  → Không (key-value, document): DynamoDB

Câu hỏi 2: Có thể define access patterns trước không?
  → Có: DynamoDB (cần define trước để design table)
  → Không (ad-hoc queries): Aurora

Câu hỏi 3: Scale requirements?
  → > 10,000 req/giây ổn định: DynamoDB
  → < 10,000 req/giây hoặc variable: cả hai

Câu hỏi 4: Latency requirements?
  → < 10ms bắt buộc: DynamoDB
  → 10-100ms acceptable: Aurora

Câu hỏi 5: ACID transactions phức tạp?
  → Multi-table ACID: Aurora
  → Single-table hoặc simple multi-item: DynamoDB (TransactWrite)
```

### Chi Tiết So Sánh

| Tiêu Chí | DynamoDB | Aurora |
|----------|----------|--------|
| **Query model** | Primary key lookups + GSI | Full SQL |
| **Joins** | Không native (application-level) | Có — native SQL JOINs |
| **Transactions** | TransactWrite (tối đa 25 items) | Full ACID, unlimited |
| **Latency** | Single-digit ms | 1-10ms (read replicas) |
| **Scale** | Virtually unlimited | Vertical + read scale-out |
| **Schema changes** | No downtime, add attributes any time | ALTER TABLE (cần careful) |
| **Pricing model** | Per request hoặc provisioned RCU/WCU | Per instance hour |
| **Operational overhead** | Thấp — fully managed, no tuning | Cao hơn — cần index tuning, query optimization |
| **Learning curve** | Cao — access pattern design | Thấp — SQL chuẩn |

### Câu Trả Lời Hay Cho Phỏng Vấn

> "Tôi thường nghĩ về nó như sau: nếu tôi biết chính xác làm thế nào application sẽ query data và cần millisecond latency ở bất kỳ scale nào, DynamoDB là lựa chọn mạnh. Nhưng nếu data model phức tạp, cần flexible queries, hoặc cần full ACID trên nhiều entities, Aurora sẽ phù hợp hơn. Trong thực tế, nhiều hệ thống lớn dùng cả hai: Aurora cho transactional core data, DynamoDB cho high-throughput, low-latency access."

---

## Caching: Redis vs Memcached vs DAX

### Bảng So Sánh Ba Lựa Chọn

| Tiêu Chí | Redis | Memcached | DAX (DynamoDB Accelerator) |
|----------|-------|-----------|---------------------------|
| **Target database** | Bất kỳ | Bất kỳ | DynamoDB only |
| **Data structures** | Phong phú (Sets, Lists, etc.) | String only | DynamoDB items |
| **Persistence** | RDB + AOF | Không | Không |
| **Replication** | Master-Replica | Không | Có |
| **Cluster mode** | Redis Cluster | Multi-node | Built-in cluster |
| **Write-through** | Tự implement | Tự implement | Built-in |
| **Transparent caching** | Không | Không | Có — API giống DynamoDB |
| **Latency** | Sub-ms | Sub-ms | Microseconds |
| **Use case chính** | Multi-purpose | Simple caching | DynamoDB acceleration |

### Khi Nào Dùng DAX

```
DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB) là caching service
đặc biệt cho DynamoDB:

✅ Application code không cần thay đổi (transparent — trong suốt)
   → Chỉ thay endpoint từ DynamoDB → DAX

✅ Read-heavy DynamoDB workload
   → Giảm đọc từ DynamoDB 90%+

✅ Latency cần < 1ms
   → DAX response thường < 100 microseconds

❌ Không dùng DAX khi:
   → Cần strongly consistent reads (DAX chỉ eventual consistent)
   → Writes nhiều hơn reads (DAX mainly read cache)
   → Budget thấp (DAX khá đắt)
```

### Redis vs Memcached Decision

```
Chọn Redis khi:
├── Cần data structures (Sorted Sets cho leaderboard, Sets cho tags)
├── Cần persistence (session data không muốn mất)
├── Cần pub/sub (Chat, notifications)
├── Cần atomic operations (rate limiting, inventory)
└── Cần replication cho HA

Chọn Memcached khi:
├── Chỉ cần simple key-value caching
├── Cần multi-threaded performance
├── Scale bằng cách thêm nodes đơn giản
└── Không cần persistence hay replication
```

---

## Consistency vs Availability

### CAP Theorem (Định Lý CAP)

CAP Theorem phát biểu: Một hệ thống phân tán không thể đồng thời đảm bảo cả ba:
- **C**onsistency (Tính Nhất Quán): Mọi node thấy cùng data
- **A**vailability (Tính Sẵn Sàng): Mọi request nhận được response
- **P**artition Tolerance (Khả Năng Chịu Phân Vùng): Hệ thống hoạt động khi có network partition (Phân Vùng Mạng)

```
Trong cloud (AWS), network partition là điều không thể tránh
→ Phải chọn P (luôn có)
→ Trade-off là: C vs A

CP systems: Consistent khi partition, nhưng có thể từ chối requests
AP systems: Available khi partition, nhưng data có thể stale
```

### AWS Database Choices

| Service | CP hay AP | Giải Thích |
|---------|-----------|-----------|
| **RDS Multi-AZ** | CP leaning | Synchronous replication, failover có thể brief unavailability |
| **DynamoDB (strong consistency)** | CP | Reads từ primary, có thể fail khi partition |
| **DynamoDB (eventual consistency)** | AP | Reads từ any replica, luôn available |
| **ElastiCache Redis** | CP (single), AP (cluster) | Cluster có thể serve stale data |
| **Aurora** | CP | Quorum-based writes trên 6 copies |

### Khi Nào Cần Strong Consistency (Nhất Quán Mạnh)

```
Strong consistency bắt buộc:
- Financial transactions (Giao Dịch Tài Chính)
- Inventory management (Quản Lý Tồn Kho) — không oversell
- Authentication tokens (Token Xác Thực) — không dùng revoked token

Eventual consistency (Nhất Quán Cuối Cùng) chấp nhận được:
- Social media feed (Nguồn Cấp Mạng Xã Hội) — 1 giây delay OK
- Product views/like counts — không critical
- Search index updates — delay OK
- Leaderboard updates — vài giây delay OK
```

### DynamoDB: Strong vs Eventual Consistency

```python
# DynamoDB Read — chọn consistency model

# Eventual Consistency (Mặc định, rẻ hơn 50%)
response = table.get_item(
    Key={'pk': 'user#123'},
    ConsistentRead=False  # default
)

# Strong Consistency (Đắt hơn, chỉ từ primary)
response = table.get_item(
    Key={'pk': 'user#123'},
    ConsistentRead=True  # 2x RCU cost
)
```

**Recommendation:** Dùng eventual consistency cho 90% trường hợp. Strong consistency chỉ khi thực sự cần (sau write phải đọc ngay).

---

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up — Mở Rộng Dọc)

```
Tăng size instance:
db.t3.micro → db.r6g.4xlarge

Ưu điểm:
✅ Đơn giản — chỉ cần thay đổi instance type
✅ Không cần thay đổi application code
✅ Phù hợp cho SQL databases

Nhược điểm:
❌ Có giới hạn — không thể scale vô hạn
❌ Downtime khi change instance type (RDS ~10-15 phút)
❌ Expensive — instance lớn tốn kém
❌ Single point of failure vẫn còn
```

### Horizontal Scaling (Scale Out — Mở Rộng Ngang)

```
Thêm nhiều instances:
1 db.r6g.2xlarge → 3 db.r6g.2xlarge (1 writer + 2 readers)

Ưu điểm:
✅ Scale vô hạn về lý thuyết
✅ Không có giới hạn cứng
✅ Tốt hơn cho read-heavy workloads

Nhược điểm:
❌ Phức tạp hơn — cần load balancer, replication
❌ Application cần biết read vs write endpoints
❌ Replication lag (Độ Trễ Sao Chép) có thể xảy ra
```

### AWS Database Scaling Strategies

| Service | Vertical Scaling | Horizontal Scaling |
|---------|-----------------|-------------------|
| **RDS** | Change instance type (reboot needed) | Add Read Replicas (up to 5) |
| **Aurora** | Change instance type (seamless) | Add Aurora Readers (up to 15, no lag) |
| **DynamoDB** | Không áp dụng | Tự động — thêm partitions |
| **ElastiCache Redis** | Resize node | Add shards (Cluster mode) hoặc replicas |

---

## On-Demand vs Provisioned Capacity

### DynamoDB Capacity Models

**On-Demand (Theo Yêu Cầu):**
```
Thanh toán: per request (mỗi yêu cầu)
Price: ~$1.25/million WRU (Write Request Unit — Đơn Vị Yêu Cầu Ghi)
       ~$0.25/million RRU (Read Request Unit — Đơn Vị Yêu Cầu Đọc)

Tốt cho:
✅ Traffic không dự đoán được
✅ Application mới chưa biết traffic pattern
✅ Workload có spikes (Đột Biến) lớn, không thường xuyên
✅ Dev/Test environments

Tránh khi:
❌ Traffic ổn định và cao — Provisioned sẽ rẻ hơn nhiều
❌ Cost predictability (Dự Đoán Chi Phí) quan trọng
```

**Provisioned (Được Cấp Phát Trước):**
```
Thanh toán: per hour cho số RCU/WCU đã cấu hình
Price: ~$0.00065/WCU/giờ = ~$0.47/WCU/tháng
       ~$0.00013/RCU/giờ = ~$0.09/RCU/tháng

Tốt cho:
✅ Traffic ổn định và dự đoán được
✅ Muốn cost predictability
✅ Traffic cao sustained (Liên Tục)

NHƯNG:
Cần kết hợp với Auto Scaling (Tự Động Co Giãn)!
```

**Quy Tắc Ngón Tay Cái:**

```
Traffic spikes > 3x average → On-Demand
Traffic ổn định, < 2x variation → Provisioned + Auto Scaling
High sustained traffic → Provisioned (50-80% tiết kiệm hơn On-Demand)
```

### RDS: Reserved Instances vs On-Demand

```
On-Demand: Trả theo giờ, không commit
Reserved Instances (Máy Chủ Đặt Trước): Commit 1 hoặc 3 năm → tiết kiệm 30-60%

1-year Reserved, Partial Upfront (Trả Trước Một Phần):
  → Tiết kiệm ~35% so với On-Demand

3-year Reserved, All Upfront (Trả Trước Toàn Bộ):
  → Tiết kiệm ~60% so với On-Demand

Dùng Reserved Instances khi:
✅ Production workload chạy 24/7
✅ Instance size ổn định (không cần resize thường xuyên)
✅ 1+ năm commit là OK với business
```

---

## Single-Region vs Multi-Region

### Tại Sao Multi-Region?

```
Lý do kỹ thuật:
1. Disaster Recovery — Nếu một region sập hoàn toàn
2. Low latency — User ở Châu Á đọc từ Singapore, User Mỹ từ us-east-1
3. Compliance — Data phải ở trong specific region (GDPR — General Data Protection Regulation)

Lý do kinh doanh:
1. Regulatory requirements — data sovereignty (Chủ Quyền Dữ Liệu)
2. Business continuity (Liên Tục Kinh Doanh) requirements
3. Global user base
```

### Multi-Region Options Trên AWS

| Option | RPO | RTO | Cost | Complexity |
|--------|-----|-----|------|------------|
| **Cross-region snapshot backup** | Giờ | Giờ | Thấp | Thấp |
| **RDS Cross-region Read Replica** | Giây | 10-30 phút | Trung bình | Trung bình |
| **Aurora Global Database** | < 1 giây | < 1 phút | Cao | Trung bình |
| **DynamoDB Global Tables** | < 1 giây | < 1 phút (auto) | Cao | Thấp |

### Trade-offs Của Multi-Region

```
Lợi ích:
+ Disaster Recovery (RPO/RTO tốt hơn)
+ Reduced latency cho global users
+ Compliance với data residency requirements

Chi phí:
- Data transfer costs (Chi Phí Truyền Dữ Liệu) xuyên region
- Replication costs
- Operational complexity tăng
- Potential conflict resolution issues (DynamoDB Global Tables)

Ví dụ cost:
Aurora Global Database secondary region:
  ~$0.20/million replicated I/O operations
  Cộng với storage costs ở mỗi region
  Cộng với instance costs ở secondary region

Kết luận: Multi-region có giá — cần business justification rõ ràng
```

### Active-Active vs Active-Passive

```
Active-Passive (Chủ-Bị):
  Primary: nhận ALL writes + reads
  Secondary: chỉ standby, sẵn sàng failover
  
  Lợi ích: Đơn giản, không conflict
  Nhược điểm: Secondary không serve traffic thường ngày

Active-Active (Chủ-Chủ):
  Both regions: nhận writes và reads
  Cần conflict resolution (Giải Quyết Xung Đột)
  
  DynamoDB Global Tables: last-writer-wins (Người Ghi Cuối Thắng)
  Aurora: không native active-active writes
  
  Lợi ích: Both regions serve traffic → better latency globally
  Nhược điểm: Conflict resolution phức tạp, eventual consistency
```

---

## Tổng Kết: Decision Framework

```
Khi interviewer hỏi "Tại sao bạn chọn X?", cấu trúc câu trả lời:

1. REQUIREMENT (Yêu Cầu): "Với requirement là [scale/latency/consistency/cost]..."

2. OPTION COMPARISON (So Sánh Lựa Chọn): "Tôi đã xem xét [option A] và [option B]..."

3. TRADE-OFFS (Đánh Đổi): "[Option A] cho [benefit] nhưng [cost]; [option B] ngược lại..."

4. DECISION (Quyết Định): "Vì [requirement chính quan trọng nhất], tôi chọn [option]..."

5. MITIGATION (Giảm Thiểu): "Để giảm thiểu nhược điểm của [option], tôi sẽ [mitigation]..."
```
