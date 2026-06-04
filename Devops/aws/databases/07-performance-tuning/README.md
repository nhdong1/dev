# 07 — Performance Tuning — Tối Ưu Hiệu Năng Cơ Sở Dữ Liệu AWS

> Hướng dẫn toàn diện về tối ưu hiệu năng cho RDS (Relational Database Service — Dịch Vụ Cơ Sở Dữ Liệu Quan Hệ), Aurora, DynamoDB và ElastiCache trên AWS — từ phân tích truy vấn đến thiết kế chỉ mục, gộp kết nối và xử lý hot partition.

## 📚 Mục Lục

1. [Tổng Quan — Methodology](#tổng-quan--methodology)
2. [Các Công Cụ Chính](#các-công-cụ-chính)
3. [Lộ Trình Học](#lộ-trình-học)
4. [Các File Trong Topic Này](#các-file-trong-topic-này)
5. [Ma Trận Vấn Đề & Giải Pháp](#ma-trận-vấn-đề--giải-pháp)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Tổng Quan — Methodology

### Tư Duy Tối Ưu Hiệu Năng

Performance tuning (tối ưu hiệu năng) không phải là việc áp dụng mọi kỹ thuật tối ưu cùng lúc. Đây là **quy trình có hệ thống** gồm bốn bước:

```
┌─────────────────────────────────────────────────────────────────┐
│              VÒNG LẶP TỐI ƯU HIỆU NĂNG                         │
│                                                                   │
│   1. MEASURE     →   2. IDENTIFY    →   3. OPTIMIZE             │
│   (Đo lường)         (Xác định)         (Tối ưu)                │
│                                              │                   │
│   4. VALIDATE    ←────────────────────────────                  │
│   (Xác nhận)                                                     │
└─────────────────────────────────────────────────────────────────┘
```

**Nguyên tắc vàng:**
- Đo trước — optimize sau (không đoán mò)
- Tối ưu nơi thắt cổ chai (bottleneck) thực sự, không phải nơi dễ tối ưu
- Mỗi thay đổi cần được đo lại để xác nhận hiệu quả

### Ba Tầng Hiệu Năng

```
┌──────────────────────────────────────────────────────────────────┐
│  TẦNG ỨNG DỤNG (Application Layer)                               │
│  ├── Connection pooling — Gộp kết nối                            │
│  ├── Query optimization — Tối ưu truy vấn                        │
│  └── Caching strategies — Chiến lược cache                       │
├──────────────────────────────────────────────────────────────────┤
│  TẦNG CƠ SỞ DỮ LIỆU (Database Layer)                            │
│  ├── Index design — Thiết kế chỉ mục                             │
│  ├── Schema optimization — Tối ưu schema                         │
│  └── Query plans — Kế hoạch truy vấn                             │
├──────────────────────────────────────────────────────────────────┤
│  TẦNG HẠ TẦNG (Infrastructure Layer)                             │
│  ├── Instance sizing — Định cỡ máy chủ                           │
│  ├── Storage IOPS — Tốc độ vào/ra lưu trữ                       │
│  └── Network throughput — Thông lượng mạng                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Các Công Cụ Chính

| Công Cụ | Mục Đích | Phạm Vi |
|---------|---------|---------|
| **RDS Performance Insights** | Phân tích SQL chậm, wait events | RDS, Aurora |
| **CloudWatch** | Metrics hệ thống, cảnh báo | Tất cả dịch vụ |
| **Enhanced Monitoring** | OS-level metrics — số liệu cấp hệ điều hành | RDS, Aurora |
| **Slow Query Log** | Nhật ký truy vấn chậm | MySQL, PostgreSQL |
| **EXPLAIN / EXPLAIN ANALYZE** | Kế hoạch thực thi truy vấn | SQL databases |
| **RDS Proxy** | Gộp kết nối, giảm overhead | RDS, Aurora |
| **DAX** (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB) | Cache in-memory cho DynamoDB | DynamoDB |
| **AWS Trusted Advisor** | Khuyến nghị tối ưu tổng thể | Tất cả dịch vụ |
| **Compute Optimizer** | Khuyến nghị instance type | RDS |

---

## 🗺️ Lộ Trình Học

### Giai Đoạn 1 — Quan Sát & Đo Lường (Tuần 1)

```
Bước 1: CloudWatch Metrics cơ bản
  → CPU, Memory, IOPS, Connections
  → Thiết lập dashboard và alarm

Bước 2: RDS Performance Insights
  → Kích hoạt và đọc dashboard
  → Tìm top SQL và wait events

Bước 3: Enhanced Monitoring
  → OS-level metrics
  → Process-level stats
```

### Giai Đoạn 2 — Phân Tích & Chẩn Đoán (Tuần 2)

```
Bước 4: Slow Query Analysis
  → Bật slow query log
  → Phân tích EXPLAIN plans
  → Xác định missing indexes

Bước 5: Connection Pooling
  → Phân tích connection patterns
  → Cấu hình RDS Proxy
  → Đo cải thiện

Bước 6: DynamoDB Performance
  → Phân tích hot partitions
  → Xem xét throttling patterns
  → Đánh giá DAX necessity
```

### Giai Đoạn 3 — Tối Ưu & Xác Nhận (Tuần 3)

```
Bước 7: Index Optimization
  → Audit chỉ mục hiện có
  → Tạo covering indexes
  → Xóa unused indexes

Bước 8: DynamoDB GSI Optimization
  → Review GSI design
  → Áp dụng write sharding
  → Tối ưu partition key distribution
```

---

## 📁 Các File Trong Topic Này

| File | Nội Dung | Độ Khó |
|------|---------|--------|
| **1-performance-insights.md** | RDS Performance Insights — dashboard, top SQL, wait events | ⭐⭐ |
| **2-cloudwatch-metrics.md** | CloudWatch metrics, Enhanced Monitoring, cảnh báo | ⭐⭐ |
| **3-slow-query-analysis.md** | Slow query log, EXPLAIN plans, chẩn đoán truy vấn chậm | ⭐⭐⭐ |
| **4-connection-pooling.md** | RDS Proxy, pgBouncer, giới hạn kết nối | ⭐⭐ |
| **5-dynamodb-performance.md** | Hot partitions, throttling, DAX, write sharding | ⭐⭐⭐ |
| **6-index-optimization.md** | Index design, GSI optimization, covering indexes | ⭐⭐⭐ |

---

## 🔥 Ma Trận Vấn Đề & Giải Pháp

### Vấn Đề RDS / Aurora

| Triệu Chứng | Nguyên Nhân Thường Gặp | Giải Pháp |
|-------------|----------------------|---------|
| CPU cao liên tục | Truy vấn không có index | Thêm index, tối ưu query |
| Latency tăng đột biến | Lock contention — xung đột khóa | Phân tích wait events trong Performance Insights |
| Connections bị từ chối | Vượt `max_connections` | Bật RDS Proxy |
| IOPS cao bất thường | Full table scan | Thêm index, bật query cache |
| Replication lag tăng | Write-heavy workload | Read replica sizing, filter events |
| Storage đầy nhanh | Binlog không được xóa | Giảm retention, bật auto-scaling |

### Vấn Đề DynamoDB

| Triệu Chứng | Nguyên Nhân Thường Gặp | Giải Pháp |
|-------------|----------------------|---------|
| ProvisionedThroughputExceededException | Hot partition — phân vùng nóng | Thay đổi partition key design |
| Throttling cao | Capacity không đủ | Tăng RCU/WCU hoặc dùng on-demand |
| Query chậm | Scan thay vì Query | Tạo GSI phù hợp |
| High latency đột biến | Cold start (DAX chưa warm) | Bật DAX, warm-up cache |
| ConsumedWriteCapacity cao bất thường | Batch writes không hiệu quả | Dùng BatchWriteItem, tối ưu item size |

### Vấn Đề ElastiCache

| Triệu Chứng | Nguyên Nhân | Giải Pháp |
|-------------|------------|---------|
| Cache hit rate thấp | TTL quá ngắn, eviction cao | Tăng TTL, tăng node size |
| Eviction cao | Memory không đủ | Scale up node hoặc thêm shard |
| Latency cao | Network hoặc keyspace thiếu tối ưu | Dùng pipeline, connection pooling |

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: Khi RDS bị chậm, bạn làm gì đầu tiên?**

```
A: Quy trình chẩn đoán từng bước:

1. Kiểm tra CloudWatch metrics:
   - CPU Utilization > 80%? → Query heavy hoặc thiếu index
   - FreeableMemory thấp? → Cần scale up
   - DatabaseConnections gần max? → Cần connection pooling

2. Mở RDS Performance Insights:
   - Xem DB load theo wait events
   - Tìm top SQL by load
   - Phân tích execution plan của query chậm nhất

3. Kiểm tra slow query log:
   - Truy vấn nào có execution time cao nhất?
   - Có full table scan không?

4. Chạy EXPLAIN ANALYZE trên top SQL:
   - Tìm Seq Scan → cần index
   - Tìm nested loop với nhiều rows → cần tối ưu join

5. Đề xuất giải pháp:
   - Thêm index phù hợp
   - Rewrite query
   - Bật RDS Proxy nếu vấn đề là connections
   - Scale instance nếu cần
```

**Q: DynamoDB bị throttling, bạn xử lý thế nào?**

```
A: Chẩn đoán và xử lý theo thứ tự:

1. Xác định loại throttling:
   - ReadThrottleEvents hay WriteThrottleEvents?
   - Ảnh hưởng toàn bảng hay một số partition cụ thể?

2. Phân tích access pattern:
   - Có hot partition không? (nhiều requests tới cùng partition key)
   - Dùng CloudWatch Contributor Insights để xác định partition key nào bị hot

3. Giải pháp ngắn hạn:
   - Nếu on-demand: chờ DynamoDB tự scale (vài phút)
   - Nếu provisioned: tăng RCU/WCU thủ công

4. Giải pháp dài hạn:
   - Redesign partition key để phân phối đều hơn
   - Thêm random suffix hoặc dùng write sharding
   - Bật DAX nếu read-heavy
   - Chuyển sang on-demand nếu traffic không đều
```

**Q: Giải thích connection pooling và RDS Proxy giải quyết vấn đề gì?**

```
A: Vấn đề:
- Mỗi connection tới RDS tốn ~10MB RAM
- Lambda functions tạo connection mới mỗi invocation
- Burst traffic → too many connections → OOM crash

RDS Proxy giải quyết:
- Duy trì pool kết nối ổn định tới database
- Nhiều Lambda/app share cùng pool
- Giảm 99% số connections thực tế tới DB
- Failover nhanh hơn (< 30 giây vs vài phút)
- Tích hợp IAM authentication và Secrets Manager
```

---

## 📊 Mục Tiêu Học Tập

Sau khi hoàn thành topic này, bạn có thể:

- [ ] Kích hoạt và đọc dashboard RDS Performance Insights
- [ ] Thiết lập CloudWatch alarms cho các metric quan trọng
- [ ] Phân tích slow query log và tạo EXPLAIN plan
- [ ] Cấu hình RDS Proxy để giảm connection overhead
- [ ] Chẩn đoán và giải quyết DynamoDB hot partition
- [ ] Thiết kế covering indexes cho SQL databases
- [ ] Tối ưu GSI design cho DynamoDB

---

## 🔗 Điều Hướng Nhanh

| Tiếp Theo | File |
|----------|------|
| Performance Insights chi tiết | [1-performance-insights.md](1-performance-insights.md) |
| CloudWatch Metrics | [2-cloudwatch-metrics.md](2-cloudwatch-metrics.md) |
| Slow Query Analysis | [3-slow-query-analysis.md](3-slow-query-analysis.md) |
| Connection Pooling | [4-connection-pooling.md](4-connection-pooling.md) |
| DynamoDB Performance | [5-dynamodb-performance.md](5-dynamodb-performance.md) |
| Index Optimization | [6-index-optimization.md](6-index-optimization.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Chủ Đề Liên Quan:** [06-security](../06-security/) | [09-monitoring](../09-monitoring/) | [11-cost-optimization](../11-cost-optimization/)
