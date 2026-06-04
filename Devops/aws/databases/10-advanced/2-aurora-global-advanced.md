# 2. Aurora Global Database — Multi-Region Active-Active Nâng Cao

> Aurora Global Database (Cơ Sở Dữ Liệu Toàn Cầu Aurora) mở rộng Aurora cluster ra nhiều AWS Region, đạt **RPO gần 0** (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) và **RTO dưới 1 phút** (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục), với replication lag (độ trễ sao chép) thường dưới 1 giây.

## 📚 Mục Lục

1. [Kiến Trúc Global Database](#kiến-trúc-global-database)
2. [Primary vs Secondary Regions](#primary-vs-secondary-regions)
3. [Replication Lag & Consistency](#replication-lag--consistency)
4. [Planned Failover vs Unplanned Failover](#planned-failover-vs-unplanned-failover)
5. [Managed Planned Failover](#managed-planned-failover)
6. [Write Forwarding](#write-forwarding)
7. [Active-Active Pattern](#active-active-pattern)
8. [Monitoring Global Database](#monitoring-global-database)
9. [Chi Phí & Tối Ưu](#chi-phí--tối-ưu)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Global Database

### Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AURORA GLOBAL DATABASE                          │
│                                                                     │
│  ┌──────────────────────────┐    ┌──────────────────────────┐      │
│  │   PRIMARY REGION         │    │   SECONDARY REGION        │      │
│  │   (Vùng Chính)           │    │   (Vùng Phụ)             │      │
│  │                          │    │                           │      │
│  │  ┌───────────────────┐   │    │  ┌──────────────────────┐│      │
│  │  │   Writer Instance │   │    │  │  Reader Instances     ││      │
│  │  │   (Nút Ghi)       │   │    │  │  (Các Nút Đọc)       ││      │
│  │  └───────────────────┘   │    │  └──────────────────────┘│      │
│  │  ┌───────────────────┐   │    │                           │      │
│  │  │   Reader Instances│   │    │  Storage Volume (replica) │      │
│  │  └───────────────────┘   │    └──────────────────────────┘      │
│  │                          │              ↑                         │
│  │  Storage Volume          │    Replication ~1 giây                │
│  └──────────────────────────┘              │                         │
│               │                           │                          │
│               └───────────────────────────┘                          │
│                   Storage-level replication                          │
│                   (Không đi qua database layer)                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Điểm Độc Đáo Của Aurora Global: Storage-Level Replication

Aurora Global Database không sao chép **ở database layer** (tầng cơ sở dữ liệu) như MySQL binlog replication, mà sao chép **ở storage layer** (tầng lưu trữ):

```
Traditional Replication:
Primary DB → Write log → Send binlog → Apply on secondary DB
(Nhiều bước, CPU intensive, higher lag)

Aurora Global Replication:
Primary Storage → Replicate storage blocks → Secondary Storage
(Storage-level, low CPU, sub-second lag thường đạt được)
```

### Các Giới Hạn

| Thông Số | Giới Hạn |
|----------|----------|
| Secondary regions tối đa | 5 |
| Reader instances mỗi secondary | Tối đa 16 |
| Replication lag mục tiêu | < 1 giây (thường 100-200ms) |
| RPO khi disaster | < 1 giây |
| RTO khi promote secondary | < 1 phút |

---

## Primary vs Secondary Regions

### Primary Region (Vùng Chính)

- Chứa **writer instance** (nút ghi) — **duy nhất một** trong toàn Global Database
- Chứa cả reader instances phục vụ read workload local
- Nhận tất cả write requests (yêu cầu ghi) từ ứng dụng

### Secondary Region (Vùng Phụ)

- Chỉ có **reader instances** — không ghi trực tiếp (trừ khi bật Write Forwarding)
- Phục vụ read workload với **low latency** (độ trễ thấp) cho users ở gần region đó
- Có thể được **promoted** (thăng cấp) thành primary khi primary region gặp sự cố

### Endpoint Types (Loại Điểm Truy Cập)

```
Primary Region:
  cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com         ← Writer endpoint
  cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com      ← Reader endpoint

Secondary Region:
  cluster.cluster-ro-xxxxx.ap-southeast-1.rds.amazonaws.com ← Reader endpoint (chỉ đọc)
```

---

## Replication Lag & Consistency

### Đo Lường Replication Lag

```sql
-- Trên secondary region, kiểm tra lag hiện tại
SELECT 
    server_id,
    session_id,
    last_update_timestamp,
    DATEDIFF(millisecond, last_update_timestamp, GETDATE()) AS lag_ms
FROM aurora_global_db_instance_status();
```

### CloudWatch Metric: AuroraGlobalDBReplicationLag

```
Metric: AuroraGlobalDBReplicationLag
Unit:   Milliseconds
Normal: < 1000ms (1 giây)
Alert:  > 5000ms (5 giây) — cần điều tra
```

### Consistency Model (Mô Hình Nhất Quán)

Aurora Global Database cung cấp **eventual consistency** (nhất quán cuối cùng) trên secondary regions:

```
Client viết ở Primary (us-east-1):
  INSERT INTO orders VALUES (123, 'Alice', 1500);
  → Commit ở primary storage

100-200ms sau, Secondary (ap-southeast-1):
  → Nhận storage replica updates
  → Đọc được data mới

Trong khoảng thời gian ~100-200ms:
  → Secondary chưa có dữ liệu mới (stale read — đọc dữ liệu cũ)
```

**Thiết kế ứng dụng:** Nếu cần read-your-own-writes (đọc lại dữ liệu vừa ghi), phải route về primary region writer.

---

## Planned Failover vs Unplanned Failover

### Planned Failover (Chuyển Đổi Dự Phòng Có Kế Hoạch)

Khi cần maintenance hoặc chủ động chuyển vùng chính:

```
1. Ứng dụng ngừng ghi về primary (hoặc chấp nhận brief downtime)
2. Chờ replication lag về 0
3. Promote secondary thành primary
4. Cập nhật DNS / connection strings
5. Ứng dụng kết nối vào primary mới
```

**Thời gian:** Vài phút (do bước chờ replication lag)

### Unplanned Failover (Chuyển Đổi Dự Phòng Không Có Kế Hoạch)

Khi primary region không khả dụng do sự cố:

```
1. Sự cố xảy ra ở primary region
2. Phát hiện qua CloudWatch + health checks (kiểm tra sức khỏe)
3. Kỹ sư hoặc automation quyết định promote secondary
4. Promote hoàn tất < 1 phút
5. Ứng dụng kết nối vào secondary (giờ là primary mới)
```

**Data loss:** RPO < 1 giây (lượng giao dịch trong 1 giây cuối có thể mất)

### So Sánh

| | Planned Failover | Unplanned Failover |
|--|------------------|--------------------|
| **Trigger** | Chủ động (bảo trì, di chuyển) | Sự cố (region down) |
| **RPO** | ~0 (chờ lag = 0) | < 1 giây |
| **RTO** | Vài phút | < 1 phút |
| **Data loss** | Không | Có thể < 1 giây |

---

## Managed Planned Failover

AWS cung cấp **Managed Planned Failover** (Chuyển Đổi Dự Phòng Có Kế Hoạch Được Quản Lý) — tự động hóa quy trình failover an toàn:

```bash
# CLI để thực hiện managed planned failover
aws rds failover-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --target-db-cluster-identifier arn:aws:rds:ap-southeast-1:123456789:cluster:my-cluster
```

**Quy trình tự động:**
1. Chặn write mới vào primary
2. Chờ secondary bắt kịp (lag = 0)
3. Promote secondary thành primary
4. Downgrade primary thành secondary
5. Cho phép write vào primary mới

---

## Write Forwarding

**Write Forwarding** (Chuyển Tiếp Ghi) cho phép ứng dụng ghi vào secondary region — Aurora tự động forward (chuyển tiếp) write đến primary region.

### Kiến Trúc Write Forwarding

```
User ở AP-Southeast-1 ghi:
  App → Secondary Region Reader → [Write Forwarding] → Primary Region Writer
                                                              ↓
                                                    Storage replication
                                                              ↓
                              Secondary Region ← Nhận replica update
```

### Cấu Hình

```sql
-- Bật write forwarding khi tạo secondary cluster
-- Hoặc modify sau:
aws rds modify-db-cluster \
  --db-cluster-identifier my-secondary-cluster \
  --enable-global-write-forwarding
```

### Lưu Ý Quan Trọng

- **Latency cao hơn:** Write phải round-trip qua primary region (thêm ~100-300ms)
- **Không hỗ trợ transactions:** Không thể dùng multi-statement transactions qua write forwarding
- **Use case phù hợp:** Occasional writes từ secondary region không yêu cầu low latency write

---

## Active-Active Pattern

Aurora Global Database về bản chất là **Active-Passive** (1 primary ghi, nhiều secondary đọc). Để đạt **Active-Active** thực sự, cần pattern ở tầng ứng dụng:

### Pattern 1: Region-based Routing (Định Tuyến Theo Vùng)

```
Region US-East-1 (Primary):
  - Xử lý write cho users US
  - Read từ local reader

Region AP-Southeast-1 (Secondary):
  - Xử lý write cho users Asia → Write Forwarding → US-East-1
  - Read từ local reader (low latency)

→ Users Asia đọc nhanh (local), ghi chậm hơn (forwarded)
→ Không phải true active-active, nhưng đủ cho nhiều use case
```

### Pattern 2: Entity Partitioning (Phân Vùng Theo Thực Thể)

```
Database A (Primary US):  Xử lý users US, data US
Database B (Primary EU):  Xử lý users EU, data EU

Mỗi DB có Global Database secondary ở vùng kia (cho DR)
Users không cross-region read/write vào cùng entity

→ Phức tạp về routing logic nhưng true active-active
→ Cần giải quyết: user US muốn đọc data EU entity?
```

### Khi Nào Dùng DynamoDB Global Tables Thay Thế?

Nếu cần **true active-active** (tất cả regions đều ghi và đọc cùng data), DynamoDB Global Tables là lựa chọn tự nhiên hơn Aurora Global Database. Xem `3-dynamodb-global-tables.md`.

---

## Monitoring Global Database

### CloudWatch Metrics Quan Trọng

```
AuroraGlobalDBReplicationLag       — Lag sao chép (ms), cần < 1000ms
AuroraGlobalDBDataTransferBytes    — Lượng data transfer qua regions
DatabaseConnections                — Số kết nối đến secondary
CPUUtilization                     — Theo dõi mỗi instance
```

### Alarm Khuyên Dùng

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "GlobalDB-ReplicationLag-High" \
  --metric-name AuroraGlobalDBReplicationLag \
  --namespace AWS/RDS \
  --period 60 \
  --threshold 5000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:db-alerts
```

### Kiểm Tra Sức Khỏe Tổng Thể

```sql
-- Chạy trên primary region
SELECT 
    aws_region,
    db_cluster_id,
    status,
    replication_lag_in_msec
FROM aurora_global_db_status();
```

---

## Chi Phí & Tối Ưu

### Các Thành Phần Chi Phí

| Thành Phần | Chi Tiết |
|------------|----------|
| **Instance costs** | Primary + tất cả secondary instances |
| **Storage** | Chỉ tính 1 lần ở primary (secondary storage không tính thêm) |
| **Data transfer** | Cross-region replication data transfer (GB) |
| **I/O** | Theo giao dịch ghi/đọc |

### Tối Ưu Chi Phí

```
1. Dùng Aurora I/O-Optimized nếu I/O intensive (bao gồm I/O trong storage cost)
2. Secondary instances: Có thể dùng instance type nhỏ hơn nếu chỉ phục vụ read
3. Tắt secondary cluster khi không cần (dev/test environments)
4. Monitor replication traffic — tối ưu batch size writes
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Aurora Global Database replication hoạt động như thế nào?

> **Trả lời:** Aurora Global Database sao chép ở **storage layer** (tầng lưu trữ), không phải database layer. Khi primary viết, Aurora storage tự động replicates storage blocks sang secondary region. Không cần binlog, không tốn CPU của DB instances. Lag thường dưới 1 giây, RPO < 1 giây, RTO < 1 phút khi promote secondary.

### Q2: Khi nào nên dùng Aurora Global Database thay vì DynamoDB Global Tables?

> **Trả lời:** Dùng Aurora Global Database khi:
> - Workload cần **SQL** (quan hệ, JOIN, stored procedures)
> - Có thể sống với **single writer** (1 vùng ghi)
> - Cần **ACID transactions** (Tính Nguyên Tử, Nhất Quán, Cô Lập, Bền Vững) đầy đủ
>
> Dùng DynamoDB Global Tables khi:
> - Cần **multi-master write** (ghi từ nhiều vùng cùng lúc)
> - Workload là NoSQL key-value
> - Chấp nhận **eventual consistency** và last-write-wins conflict resolution

### Q3: Write Forwarding có phù hợp cho production không?

> **Trả lời:** Phụ thuộc use case. Write Forwarding phù hợp khi:
> - Ứng dụng ở secondary region cần **occasional writes** (ghi không thường xuyên)
> - Không yêu cầu low latency write (thêm 100-300ms là chấp nhận được)
> - Không cần multi-statement transactions
>
> Không phù hợp khi: write intensive workload, cần latency < 10ms, cần transactions phức tạp.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
