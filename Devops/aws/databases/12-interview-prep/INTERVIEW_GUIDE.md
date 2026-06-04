# 🏆 INTERVIEW GUIDE — Top 20 Câu Hỏi Phỏng Vấn AWS Database

> 20 câu hỏi xuất hiện thường xuyên nhất trong phỏng vấn về AWS Database Services — kèm câu trả lời mẫu chi tiết, điểm cần nhấn mạnh, và lỗi phổ biến cần tránh.

## Mục Lục

1. [RDS & Aurora (Câu 1-8)](#rds--aurora)
2. [DynamoDB (Câu 9-13)](#dynamodb)
3. [ElastiCache & Caching (Câu 14-16)](#elasticache--caching)
4. [HA, Backup & Security (Câu 17-19)](#ha-backup--security)
5. [System Design Tổng Hợp (Câu 20)](#system-design-tổng-hợp)
6. [Tips Phỏng Vấn](#tips-phỏng-vấn)

---

## RDS & Aurora

### Câu 1: Phân biệt RDS Multi-AZ và Read Replicas

**Mức độ:** ⭐ Cơ bản — xuất hiện trong gần 100% phỏng vấn

**Câu trả lời mẫu:**

Multi-AZ (Multi Availability Zone — Triển Khai Đa Vùng Sẵn Sàng) và Read Replicas (Bản Sao Đọc) là hai cơ chế hoàn toàn khác nhau về mục đích:

| Tiêu Chí | Multi-AZ | Read Replicas |
|----------|----------|---------------|
| **Mục đích chính** | High Availability — Tính Sẵn Sàng Cao | Scale reads — Mở rộng tải đọc |
| **Sync mode** | Synchronous (Đồng bộ) | Asynchronous (Bất đồng bộ) |
| **Có thể đọc trực tiếp không?** | Không — chỉ là standby | Có — có endpoint riêng |
| **Failover tự động?** | Có — trong 1-2 phút | Không — phải promote thủ công |
| **Hỗ trợ cross-region?** | Không (trong cùng region) | Có — cross-region replica |
| **Chi phí** | ~2x instance (luôn chạy) | Pay per instance |

**Khi nào dùng gì:**
- Multi-AZ: luôn bật trên production — đây là HA, không phải performance
- Read Replicas: khi có nhiều read workload cần scale out, hoặc cần analytics mà không muốn ảnh hưởng primary

**Lỗi thường gặp:** Nhầm tưởng Multi-AZ cải thiện performance vì có 2 instances — sai! Multi-AZ standby không nhận traffic.

---

### Câu 2: Aurora khác RDS ở điểm gì? Tại sao nhanh hơn?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

Aurora (Cơ Sở Dữ Liệu Đám Mây Hiệu Năng Cao) khác RDS ở kiến trúc storage:

**Kiến trúc RDS:**
```
Application → RDS Primary → EBS volume (gắn trực tiếp)
                         → EBS volume trên standby (sync replicate)
```

**Kiến trúc Aurora:**
```
Application → Aurora Writer → Shared Storage Layer (6 copies across 3 AZs)
           → Aurora Readers ↗ (cùng đọc từ shared storage)
```

**Tại sao Aurora nhanh hơn:**
1. **Log-based storage** — Aurora chỉ ghi redo logs, không ghi full pages → giảm I/O (Input/Output — Vào/Ra) 6 lần
2. **Shared storage** — Readers đọc từ cùng storage với Writer → không có replication lag (độ trễ sao chép)
3. **Auto-healing** — Tự sửa lỗi data blocks mà không cần restart
4. **Parallel query** — Có thể push query xuống storage layer để xử lý song song

**Con số quan trọng:**
- Aurora MySQL: nhanh hơn RDS MySQL **5 lần**
- Aurora PostgreSQL: nhanh hơn RDS PostgreSQL **3 lần**
- Failover time: **< 30 giây** (RDS là 1-2 phút)

---

### Câu 3: Giải thích Aurora Serverless v2 và khi nào nên dùng

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

Aurora Serverless v2 (Aurora Không Máy Chủ Phiên Bản 2) là tính năng auto-scaling (Tự Động Co Giãn) của Aurora, cho phép capacity tự động tăng/giảm theo workload theo đơn vị ACU (Aurora Capacity Unit — Đơn Vị Năng Lực Aurora).

**Cơ chế hoạt động:**
```
Min ACU ─────────────────────────── Max ACU
  0.5 ACU ────→ scale up ────→ 128 ACU
              (trong vài giây)
```

**Khi nên dùng Aurora Serverless v2:**
- Workload không đều (unpredictable) — sáng thấp, trưa cao
- Development/Test environment — trả tiền theo usage
- Ứng dụng mới chưa biết traffic profile
- Multi-tenant SaaS (Software as a Service — Phần Mềm Dạng Dịch Vụ) với nhiều tenants nhỏ

**Khi KHÔNG nên dùng:**
- Workload ổn định, dự đoán được → dùng provisioned + Reserved Instances (tiết kiệm hơn)
- Latency cực kỳ quan trọng → cold start có thể gây spike latency

**Lưu ý về chi phí:** Serverless v2 đắt hơn provisioned nếu workload chạy liên tục. Nếu database chạy 24/7 ở mức cao, provisioned instances với Reserved Instances rẻ hơn.

---

### Câu 4: RPO và RTO là gì? Làm thế nào để đạt RPO thấp với RDS?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

- **RPO** (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục): Lượng data tối đa có thể chấp nhận mất. RPO = 1 giờ nghĩa là có thể mất tối đa 1 giờ data.
- **RTO** (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục): Thời gian tối đa để hệ thống hoạt động trở lại. RTO = 4 giờ nghĩa là hệ thống phải online trong 4 giờ sau sự cố.

**Đạt RPO thấp với RDS:**

| RPO Target | Giải Pháp |
|------------|-----------|
| **RPO ≈ 5 phút** | Automated Backups (Sao Lưu Tự Động) + PITR (Point-in-Time Recovery) — Transaction logs lưu mỗi 5 phút |
| **RPO ≈ 0** | Multi-AZ — Synchronous replication (Sao Chép Đồng Bộ), không có data loss |
| **RPO ≈ seconds, cross-region** | Aurora Global Database — replication lag < 1 giây xuyên vùng |

**Đạt RTO thấp với RDS:**

| RTO Target | Giải Pháp |
|------------|-----------|
| **RTO < 2 phút** | Multi-AZ automatic failover |
| **RTO < 1 giờ** | PITR restore vào new instance |
| **RTO < 15 phút** | Read Replica promote (nếu đã sync) |

---

### Câu 5: RDS Proxy là gì và tại sao cần nó?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

RDS Proxy (Proxy RDS — Lớp Trung Gian Kết Nối) là managed connection pooling service (Dịch Vụ Gộp Kết Nối Được Quản Lý) nằm giữa ứng dụng và RDS/Aurora.

**Vấn đề RDS Proxy giải quyết:**

```
Không có RDS Proxy:
Lambda (1000 concurrent) ──→ RDS (max 1000 connections) → connection exhaustion!

Với RDS Proxy:
Lambda (1000 concurrent) ──→ RDS Proxy (reuse connections) ──→ RDS (50 connections)
```

**Lợi ích chính:**
1. **Connection pooling** — Tái sử dụng kết nối, giảm overhead tạo connection mới
2. **Failover nhanh hơn** — RDS Proxy giữ connections khi failover, app không cần reconnect → giảm failover time **66%**
3. **IAM Authentication** (Xác Thực IAM) — Hỗ trợ xác thực qua IAM thay vì username/password
4. **Secrets Manager integration** — Credentials rotation (Xoay Vòng Thông Tin Đăng Nhập) tự động không cần restart app

**Khi cần RDS Proxy:**
- Lambda functions với nhiều concurrent executions (Thực Thi Song Song)
- Ứng dụng có nhiều short-lived connections (kết nối ngắn hạn)
- Cần giảm failover impact cho production

**Chi phí:** ~$0.015/giờ per connection capacity unit — khá rẻ so với lợi ích.

---

### Câu 6: Khi nào chọn Aurora thay vì RDS?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

Chọn **Aurora** khi:
- Traffic cao, cần performance tốt nhất (Aurora ~3-5x nhanh hơn RDS)
- Cần failover nhanh (< 30 giây vs 1-2 phút)
- Cần nhiều Read Replicas (Aurora hỗ trợ 15 replicas vs RDS 5)
- Cần Global Database (Cơ Sở Dữ Liệu Toàn Cầu) xuyên region
- Muốn Pay Per I/O (Trả Theo I/O) thay vì pay for storage size
- Đang dùng MySQL hoặc PostgreSQL (Aurora tương thích hoàn toàn)

Chọn **RDS** khi:
- Cần database engine khác (Oracle, SQL Server, MariaDB)
- Budget thấp — RDS rẻ hơn Aurora ~20-30%
- Workload nhỏ, không cần tính năng nâng cao của Aurora
- Team đã quen với RDS specific features

**Rule of thumb:** Nếu đang dùng MySQL/PostgreSQL với traffic đáng kể → chuyển sang Aurora. Nếu cần Oracle/SQL Server → phải dùng RDS.

---

### Câu 7: Giải thích Aurora Global Database và use case

**Mức độ:** ⭐⭐⭐ Nâng cao

**Câu trả lời mẫu:**

Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu Aurora) cho phép một Aurora cluster phục vụ nhiều AWS regions với replication lag (Độ Trễ Sao Chép) cực thấp.

**Kiến trúc:**
```
Primary Region (us-east-1):
  Writer instance ──→ Replication (< 1 giây) ──→ Secondary Region (eu-west-1)
                                                   Read-only replicas
```

**Đặc điểm kỹ thuật:**
- Replication lag: **< 1 giây** (thường 100-200ms)
- Tối đa **5 secondary regions**
- RPO = **1 giây** cho cross-region
- RTO = **< 1 phút** khi failover cross-region (promote secondary)

**Use cases (Trường Hợp Sử Dụng):**
1. **Global read performance** — User Châu Á đọc từ region Nhật Bản, User Mỹ đọc từ US
2. **Disaster recovery** — Nếu us-east-1 sập, promote eu-west-1 làm primary trong < 1 phút
3. **Compliance** — Data phải ở trong specific region

**Lưu ý quan trọng:** Secondary regions chỉ là read-only. Writes luôn phải về primary region. Nếu cần multi-region writes → cần DynamoDB Global Tables hoặc custom application-level conflict resolution.

---

### Câu 8: Làm thế nào để xử lý database performance issue trên production?

**Mức độ:** ⭐⭐⭐ Nâng cao (câu hỏi tình huống)

**Câu trả lời mẫu:**

Quy trình chẩn đoán từng bước:

**Bước 1 — Identify Symptoms (Xác Định Triệu Chứng)**
```
- Latency cao? Connection timeout? High CPU?
- Bắt đầu khi nào? Có deploy mới không?
- Ảnh hưởng tất cả queries hay chỉ một số?
```

**Bước 2 — Check RDS Performance Insights (Thông Tin Hiệu Năng RDS)**
```
- AAS (Average Active Sessions — Số Phiên Hoạt Động Trung Bình) > vCPU count → database overloaded
- Top waits: "io/table/sql_temporary_table" → tmp tables lớn
- Top waits: "lock/table/lock" → locking issues
- Identify top SQL statements đang chiếm nhiều nhất
```

**Bước 3 — Analyze CloudWatch Metrics (Phân Tích Số Liệu CloudWatch)**
```
- CPUUtilization > 80%: CPU bound
- ReadIOPS / WriteIOPS cao bất thường: I/O bound
- DatabaseConnections gần max: connection exhaustion
- FreeableMemory thấp: memory pressure
```

**Bước 4 — Check Slow Query Log (Nhật Ký Truy Vấn Chậm)**
```
- Enable slow_query_log nếu chưa bật
- Dùng pt-query-digest hoặc mysqldumpslow để phân tích
- Tìm queries không có index (Full Table Scan)
```

**Bước 5 — Giải Pháp Tương Ứng**
```
Slow query + no index → ADD INDEX
High connections → Enable RDS Proxy (Connection Pooling)
High reads → Add Read Replica + cache với ElastiCache
High CPU → Upgrade instance class hoặc tối ưu query
Locking issues → Review transaction logic
```

---

## DynamoDB

### Câu 9: Partition Key tốt là gì? Tại sao quan trọng?

**Mức độ:** ⭐⭐ Trung bình — rất thường được hỏi

**Câu trả lời mẫu:**

Partition Key (Khóa Phân Vùng) quyết định data được phân phối như thế nào trên các storage partitions (Phân Vùng Lưu Trữ). DynamoDB sử dụng consistent hashing (Băm Nhất Quán) để map partition key → physical partition.

**Partition Key tốt cần:**
1. **High cardinality** (Bản Số Cao) — nhiều giá trị distinct (khác biệt), tránh hot partitions (Phân Vùng Nóng)
2. **Uniform distribution** (Phân Phối Đều) — requests không tập trung vào một vài keys
3. **Liên quan đến access pattern** — queries thường dùng key nào → đó là partition key

**Ví dụ tốt vs xấu:**

| Partition Key | Vấn Đề | Tốt/Xấu |
|---------------|--------|----------|
| `user_id` (UUID) | Cardinality cao, phân phối đều | ✅ Tốt |
| `order_id` (UUID) | Cardinality cao | ✅ Tốt |
| `status` ("pending"/"completed") | Chỉ 2 giá trị → hot partition | ❌ Xấu |
| `country` | Ít giá trị, không đều | ❌ Xấu |
| `created_date` (YYYY-MM-DD) | Chỉ 365 giá trị/năm | ❌ Xấu |

**Kỹ thuật xử lý hot partition:**
```
Write Sharding (Phân Mảnh Ghi):
  Thay vì key = "product_id"
  Dùng key = "product_id_" + random(1..10)
  → Phân tán writes ra 10 partitions
  → Khi read, scatter-gather (Truy Vấn Song Song) trên 10 partitions
```

---

### Câu 10: GSI vs LSI — khác nhau như thế nào và khi nào dùng?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

- **LSI** (Local Secondary Index — Chỉ Mục Phụ Cục Bộ): Index thêm sort key khác, cùng partition key, **phải tạo khi tạo table**, share storage với base table
- **GSI** (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu): Index với partition key VÀ sort key hoàn toàn mới, **thêm bất cứ lúc nào**, có provisioned capacity riêng

| Tiêu Chí | LSI | GSI |
|----------|-----|-----|
| **Partition key** | Giống base table | Khác được |
| **Tạo khi nào** | Chỉ lúc CREATE TABLE | Bất cứ lúc nào |
| **Consistency** | Strong hoặc eventual | Eventual only |
| **Storage** | Shared với base table | Riêng |
| **Capacity** | Shared với base table | Riêng, cần cấu hình |
| **Giới hạn** | 5 LSIs/table | 20 GSIs/table |

**Khi dùng LSI:**
- Cần query với cùng partition key nhưng sort khác
- Cần strong consistency trên index
- Ví dụ: Table `orders` partition by `customer_id`, sort by `order_date`. LSI cho phép sort by `total_amount` vẫn trong cùng customer

**Khi dùng GSI:**
- Cần query theo dimension hoàn toàn khác
- Ví dụ: Table `orders` partition by `order_id`. GSI partition by `customer_id` → query "tất cả orders của customer X"

---

### Câu 11: DynamoDB vs RDS — khi nào chọn cái nào?

**Mức độ:** ⭐⭐⭐ Thường xuyên — câu hỏi kiến trúc

**Câu trả lời mẫu:**

**Chọn DynamoDB khi:**
- Cần **single-digit millisecond latency** (Độ Trễ Mili-giây Đơn) ở bất kỳ scale nào
- Workload là key-value lookups (Tra Cứu Theo Khóa-Giá Trị) hoặc simple queries
- Scale có thể tăng đột biến, khó dự đoán (gaming, IoT)
- Không cần complex joins (Kết Hợp Phức Tạp) hay transactions phức tạp
- Muốn Serverless (Không Máy Chủ), pay per request

**Chọn RDS/Aurora khi:**
- Data có quan hệ phức tạp, cần JOIN nhiều bảng
- Cần ACID transactions (Giao Dịch ACID) phức tạp trên nhiều bảng
- Có reporting/analytics queries (Truy Vấn Phân Tích) ad-hoc phức tạp
- Team quen với SQL
- Cần stored procedures (Thủ Tục Lưu Trữ), triggers, views

**Framework quyết định:**
```
Câu hỏi 1: Access pattern có thể định nghĩa trước không?
  → Có: DynamoDB phù hợp (cần biết access pattern trước)
  → Không: RDS/Aurora linh hoạt hơn với ad-hoc queries

Câu hỏi 2: Có cần complex joins không?
  → Có: RDS/Aurora
  → Không: DynamoDB

Câu hỏi 3: Scale dự kiến?
  → Millions+ req/sec: DynamoDB (unlimited scale)
  → Moderate: cả hai đều OK
```

---

### Câu 12: Giải thích DynamoDB hot partition và cách xử lý

**Mức độ:** ⭐⭐⭐ Nâng cao

**Câu trả lời mẫu:**

**Hot partition** xảy ra khi một partition nhận quá nhiều traffic, vượt quá giới hạn của partition đó (3000 RCU — Read Capacity Units — Đơn Vị Đọc, 1000 WCU — Write Capacity Units — Đơn Vị Ghi per partition).

**Nguyên nhân phổ biến:**
- Partition key có cardinality thấp (ví dụ: `country`)
- Viral content/product được access nhiều (celebrity problem)
- Time-series data với `timestamp` là partition key

**Giải pháp:**

1. **Write Sharding (Phân Mảnh Ghi):**
```
// Thêm suffix ngẫu nhiên vào partition key
key = "product_123_" + random(1, 10)  // Phân tán ra 10 partitions
// Khi query: scatter-gather trên tất cả 10 shards
```

2. **DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB):**
```
Caching layer trước DynamoDB
Read hits → trả về từ cache, không vào DynamoDB
Giảm pressure lên hot partition
```

3. **Thiết kế lại partition key:**
```
Thay vì: partition_key = "category" (low cardinality)
Dùng:    partition_key = "category#user_id"  (higher cardinality)
```

4. **Adaptive Capacity (Năng Lực Thích Ứng):**
```
Tính năng tự động của DynamoDB — tự phân bổ lại capacity
Không cần can thiệp, nhưng chậm hơn các giải pháp trên
```

---

### Câu 13: DynamoDB Streams là gì và dùng để làm gì?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

DynamoDB Streams (Luồng Dữ Liệu DynamoDB) là ordered stream (Luồng Có Thứ Tự) ghi lại mọi thay đổi (INSERT, UPDATE, DELETE) trên DynamoDB table theo thứ tự thời gian.

**Đặc điểm:**
- Retention: **24 giờ**
- Thứ tự: đảm bảo trong cùng partition key
- Near real-time: độ trễ < 1 giây

**Use cases (Trường Hợp Sử Dụng):**

```
1. Event-driven processing (Xử Lý Hướng Sự Kiện):
   DynamoDB change → Streams → Lambda → send notification

2. Cross-region replication thủ công:
   DynamoDB → Streams → Lambda → DynamoDB region khác
   (Ngày nay dùng Global Tables thay thế)

3. Audit log (Nhật Ký Kiểm Toán):
   Ghi mọi thay đổi ra S3/OpenSearch để audit

4. Cache invalidation (Làm Vô Hiệu Cache):
   DynamoDB update → Streams → Lambda → invalidate ElastiCache entry

5. Search indexing (Đánh Mục Tìm Kiếm):
   DynamoDB change → Streams → Lambda → update OpenSearch index
```

**Tích hợp với Lambda:**
```
Lambda trigger: event source = DynamoDB Stream
Lambda nhận batches (Lô) gồm nhiều records
Xử lý theo thứ tự, tự động retry khi lỗi
```

---

## ElastiCache & Caching

### Câu 14: Redis vs Memcached — khi nào dùng cái nào?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

| Tiêu Chí | Redis | Memcached |
|----------|-------|-----------|
| **Data structures** | Strings, Lists, Sets, Sorted Sets, Hashes, Bitmaps | Chỉ Strings |
| **Persistence** | RDB (Redis Database — Tập Tin Cơ Sở Dữ Liệu Redis) + AOF (Append-Only File — Tệp Chỉ Thêm) | Không có |
| **Replication** | Master-Replica (Primary-Secondary) | Không có |
| **Clustering** | Redis Cluster (Cụm Redis) — sharding tự động | Multi-threaded scaling |
| **Pub/Sub** | Có — publish/subscribe messaging | Không |
| **Geospatial** | Có | Không |
| **Transactions** | MULTI/EXEC commands | Không |

**Chọn Redis khi:**
- Cần persistence (Tính Bền Vững) — data không muốn mất khi restart
- Session management (Quản Lý Phiên) — cần expire, data structures
- Pub/Sub messaging (Nhắn Tin Phát/Nhận)
- Leaderboard (Bảng Xếp Hạng) — Sorted Sets
- Rate limiting (Giới Hạn Tốc Độ) — atomic increment operations

**Chọn Memcached khi:**
- Chỉ cần simple string caching (Lưu Cache Chuỗi Đơn Giản)
- Cần multi-threaded performance (Hiệu Năng Đa Luồng)
- Không cần persistence hay replication
- Scale bằng cách thêm nodes đơn giản

**Kết luận thực tế:** Trong hầu hết trường hợp hiện đại, Redis là lựa chọn mặc định vì tính năng phong phú hơn với chi phí tương đương.

---

### Câu 15: Giải thích Lazy Loading vs Write-Through caching

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

**Lazy Loading (Tải Lười):**
```
Cache miss → Load từ DB → Lưu vào cache → Trả về client
Cache hit  → Trả về từ cache (không cần DB)

Ưu điểm:
- Cache chỉ chứa data thực sự được đọc
- Cache miss đầu tiên chậm nhưng sau đó nhanh

Nhược điểm:
- Cache miss = 3 steps (đọc cache, đọc DB, ghi cache) → slow first read
- Stale data (Dữ Liệu Cũ): nếu DB update nhưng cache chưa expire
```

**Write-Through (Ghi Xuyên):**
```
Write to app → Ghi vào DB → Ghi vào cache → Confirm

Ưu điểm:
- Cache luôn up-to-date (Cập Nhật)
- Không bao giờ stale data

Nhược điểm:
- Write penalty (Chi Phí Ghi Tăng): mọi write đều phải update cache
- Cache lưu nhiều data không được đọc (write nhiều hơn read)
```

**Kết hợp tốt nhất:**
```
Lazy Loading + TTL (Time-to-Live — Thời Gian Sống):
- Lazy load khi read
- TTL để expire stale data (ví dụ: 1 giờ)
- Write-Through cho các field critical (quan trọng)
```

---

### Câu 16: Thiết kế session management với ElastiCache Redis

**Mức độ:** ⭐⭐⭐ Nâng cao — câu hỏi thiết kế

**Câu trả lời mẫu:**

**Kiến trúc:**
```
User Request → Load Balancer → App Server 1 ─→ ElastiCache Redis
                            → App Server 2 ─↗  (shared session store)
                            → App Server N ─↗
```

**Thiết kế Redis Key:**
```
Key format: "session:{session_id}"
Value: JSON hoặc Hash với user data
TTL: session timeout (ví dụ: 30 phút idle, 24 giờ absolute)

Ví dụ:
SET "session:abc123" '{"user_id": 456, "role": "admin", "cart": [...]}' EX 1800
```

**Xử lý expiry:**
```
Sliding expiry (Hết Hạn Trượt): reset TTL mỗi request
  → Dùng EXPIRE hoặc SET với EX option

Absolute expiry (Hết Hạn Tuyệt Đối): expire sau X giờ dù có activity
  → Lưu thêm "created_at" trong session data
```

**HA considerations:**
```
Redis Cluster với Multi-AZ:
- Replication Group (Nhóm Sao Chép) với primary + replica
- Automatic failover khi primary down
- Session data không mất khi failover
```

**Security:**
```
- Mã hóa TLS in transit (Trong Quá Trình Truyền Tải)
- AUTH token cho Redis authentication (Xác Thực Redis)
- Không lưu sensitive data (Dữ Liệu Nhạy Cảm) như password trong session
- Dùng signed session tokens (Token Phiên Có Chữ Ký) để chống tampering (Giả Mạo)
```

---

## HA, Backup & Security

### Câu 17: Làm thế nào để thiết kế DR strategy cho production database?

**Mức độ:** ⭐⭐⭐ Nâng cao

**Câu trả lời mẫu:**

Bước đầu tiên là xác định RPO và RTO từ business requirements:
- RPO = Chấp nhận mất bao nhiêu data?
- RTO = Hệ thống cần online lại trong bao lâu?

**Các chiến lược DR từ đơn giản đến phức tạp:**

| Chiến Lược | RPO | RTO | Chi Phí |
|------------|-----|-----|---------|
| **Backup & Restore** | Giờ | Giờ | Thấp nhất |
| **Pilot Light** (Ngọn Lửa Dự Phòng) | Phút | 10-60 phút | Thấp |
| **Warm Standby** (Dự Phòng Ấm) | Giây-Phút | Phút | Trung bình |
| **Multi-Site Active-Active** | Gần 0 | Gần 0 | Cao nhất |

**Pilot Light pattern với RDS:**
```
Primary region (us-east-1):
  RDS Primary (full size) ──→ Cross-region snapshot copy (Sao Chép Ảnh Chụp Xuyên Vùng)

DR region (us-west-2):
  RDS Read Replica (minimal size) — sync từ primary

Khi disaster xảy ra:
  1. Promote Read Replica thành standalone DB (5-10 phút)
  2. Update DNS record trỏ sang DR region
  3. Scale up DR instance nếu cần
```

**Warm Standby với Aurora:**
```
Aurora Global Database:
  Primary: us-east-1 (full traffic)
  Secondary: eu-west-1 (minimal, read-only)

Failover:
  1. Promote secondary → < 1 phút
  2. Update application endpoints
  RPO: < 1 giây, RTO: < 1 phút
```

---

### Câu 18: Giải thích cách KMS encryption hoạt động với RDS

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) tích hợp với RDS để cung cấp encryption at rest (Mã Hóa Khi Lưu Trữ).

**Cơ chế hoạt động:**
```
Data plaintext (Dữ Liệu Thô)
    ↓
KMS Data Key (Khóa Dữ Liệu KMS) [Tạo mới mỗi lần]
    ↓
Encrypted data → EBS volume
    
KMS Customer Master Key (CMK — Khóa Chính Khách Hàng):
    - Lưu trong KMS, không bao giờ ra ngoài
    - Dùng để encrypt/decrypt Data Keys
    - Rotate (Xoay Vòng) hàng năm tự động
```

**Những thứ được encrypt:**
- Storage volumes (EBS)
- Automated backups
- Read Replicas
- Snapshots

**Lưu ý quan trọng:**
- Encryption chỉ bật được khi **tạo RDS instance** — không thể enable sau
- Muốn encrypt RDS đang chạy: Snapshot → Encrypt snapshot → Restore từ encrypted snapshot
- Cross-region snapshot copy có thể dùng KMS key khác

**Encryption in transit (Mã Hóa Khi Truyền Tải):**
```
- TLS/SSL cho connections từ app → RDS
- Certificate rotation với RDS Certificate Authority
- Enforce TLS qua parameter: rds.force_ssl = 1 (MySQL/PostgreSQL)
```

---

### Câu 19: Secrets Manager vs Parameter Store — khi nào dùng gì?

**Mức độ:** ⭐⭐ Trung bình

**Câu trả lời mẫu:**

| Tiêu Chí | Secrets Manager | Parameter Store |
|----------|----------------|-----------------|
| **Tính năng chính** | Secret rotation tự động | Lưu config parameters |
| **Rotation tự động** | Có — built-in cho RDS, Redshift, DocumentDB | Không có built-in |
| **Cross-account access** | Có | Không native |
| **Chi phí** | $0.40/secret/tháng + API calls | Standard tier miễn phí, Advanced tier có phí |
| **Giới hạn kích thước** | 65,536 bytes | 4,096 bytes (Standard), 8,192 bytes (Advanced) |
| **Versioning** | Có | Có |
| **Integration với RDS** | Tự động rotate | Thủ công |

**Secrets Manager khi:**
- Cần rotate credentials (Xoay Vòng Thông Tin Xác Thực) tự động (database passwords)
- Đây là secrets thực sự — API keys, passwords, tokens
- Cần audit trail (Lịch Sử Kiểm Toán) chi tiết ai đã access secret nào

**Parameter Store khi:**
- Lưu configuration values (Giá Trị Cấu Hình) không phải secrets — app configs, feature flags
- Budget thấp — Standard tier miễn phí
- Giá trị không cần rotate

**Best practice:** Database credentials → Secrets Manager. Application config → Parameter Store.

---

## System Design Tổng Hợp

### Câu 20: Thiết kế hệ thống e-commerce — chọn database cho từng component

**Mức độ:** ⭐⭐⭐ Nâng cao — câu hỏi tổng hợp

**Câu trả lời mẫu:**

**Requirements (Yêu Cầu):**
- 10 triệu users, 100,000 đơn hàng/ngày
- Black Friday peak: 10x traffic
- 99.9% uptime (Thời Gian Hoạt Động)

**Kiến trúc Database:**

```
┌─────────────────────────────────────────────┐
│ User Service (Dịch Vụ Người Dùng)            │
│  DB: Aurora PostgreSQL (ACID transactions)   │
│  Cache: ElastiCache Redis (session, profile) │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Product Catalog (Danh Mục Sản Phẩm)          │
│  DB: Aurora MySQL (product data, categories) │
│  Cache: ElastiCache Redis (product pages)    │
│  Search: OpenSearch (full-text search)       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Order Service (Dịch Vụ Đặt Hàng)             │
│  DB: Aurora PostgreSQL (ACID critical)       │
│  Pattern: Multi-AZ + Read Replicas           │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Cart Service (Dịch Vụ Giỏ Hàng)              │
│  DB: DynamoDB (key=user_id, fast reads)      │
│  Cache: DAX (thêm cache layer)               │
│  Reason: cart là key-value, scale cực cao    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Inventory Service (Dịch Vụ Tồn Kho)          │
│  DB: Aurora với DynamoDB Streams             │
│  Cache: ElastiCache Redis (stock levels)     │
│  Reason: cần ACID cho stock updates          │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Analytics (Phân Tích)                        │
│  DB: Redshift (OLAP queries)                 │
│  Data flow: Aurora → DMS → Redshift          │
└─────────────────────────────────────────────┘
```

**Trade-offs (Đánh Đổi) đã cân nhắc:**

```
Cart: DynamoDB vs Redis
→ Chọn DynamoDB vì: persistent (bền vững), global, không mất nếu Redis restart

Orders: DynamoDB vs Aurora
→ Chọn Aurora vì: ACID transactions quan trọng, joins với inventory/user

Product Search: Aurora vs OpenSearch
→ Cả hai: Aurora cho CRUD, OpenSearch cho full-text search
```

---

## Tips Phỏng Vấn

### Trước Phỏng Vấn

1. **Ôn lại số liệu cụ thể** — Interviewer ấn tượng khi bạn biết "Aurora failover < 30 giây" thay vì chỉ "nhanh hơn"
2. **Chuẩn bị câu hỏi ngược** — "Team đang dùng Aurora hay RDS? Gặp vấn đề gì với database hiện tại?"
3. **Biết giới hạn của mình** — Nếu không biết, nói thẳng rồi giải thích cách bạn sẽ tìm hiểu

### Trong Phỏng Vấn

1. **Clarify trước khi answer** — Với system design, hỏi requirements trước khi thiết kế
2. **Think aloud (Suy Nghĩ To)** — Nói ra quá trình suy nghĩ, interviewer muốn xem bạn giải quyết vấn đề như thế nào
3. **Acknowledge trade-offs** — Không có perfect solution, luôn có đánh đổi
4. **Kết nối với kinh nghiệm thực tế** — "Tôi đã xử lý trường hợp tương tự khi..."

### Câu Hỏi Thường Bị Bỏ Sót

- **Monitoring** — Bạn biết chọn database, nhưng biết monitor nó không?
- **Cost** — Giải pháp có scalable (Có Thể Mở Rộng) về chi phí không?
- **Security** — Data được bảo vệ như thế nào?
- **Operational complexity** — Ai sẽ maintain (Duy Trì) database này?

### Những Lỗi Phổ Biến Cần Tránh

1. Nói "DynamoDB luôn tốt hơn RDS" — Sai! Phụ thuộc use case
2. Quên mention caching khi nói về performance
3. Không đề cập đến backup và DR khi thiết kế hệ thống
4. Thiết kế xong không nói về cost và operational overhead
5. Quên authentication và authorization trong kiến trúc

---

**Xem Thêm:**
- [system-design-scenarios.md](system-design-scenarios.md) — 5 kịch bản thiết kế chi tiết
- [trade-off-discussions.md](trade-off-discussions.md) — Phân tích sâu các quyết định kiến trúc
- [star-stories.md](star-stories.md) — Mẫu câu chuyện STAR cho phỏng vấn behavioral
