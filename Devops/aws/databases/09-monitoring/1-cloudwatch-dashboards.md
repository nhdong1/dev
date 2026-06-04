# CloudWatch Dashboards — Bảng Điều Khiển & Số Liệu Chính

> Hướng dẫn thiết lập CloudWatch Dashboards (Bảng Điều Khiển CloudWatch) và theo dõi key database KPIs (Key Performance Indicators — Chỉ Số Hiệu Suất Chính) cho RDS, Aurora, DynamoDB và ElastiCache.

---

## 📚 Mục Lục

1. [CloudWatch Metrics Fundamentals](#cloudwatch-metrics-fundamentals)
2. [RDS & Aurora Key Metrics](#rds--aurora-key-metrics)
3. [DynamoDB Key Metrics](#dynamodb-key-metrics)
4. [ElastiCache Key Metrics](#elasticache-key-metrics)
5. [Tạo Dashboard Toàn Diện](#tạo-dashboard-toàn-diện)
6. [CloudWatch Logs Insights](#cloudwatch-logs-insights)
7. [Best Practices](#best-practices)

---

## CloudWatch Metrics Fundamentals

### Cấu Trúc Metric trong CloudWatch

Mỗi CloudWatch Metric (Số Liệu CloudWatch) có cấu trúc:

```
Namespace / MetricName { Dimensions }

Ví dụ:
  AWS/RDS / CPUUtilization { DBInstanceIdentifier: prod-mysql-01 }
  AWS/DynamoDB / ConsumedReadCapacityUnits { TableName: orders }
  AWS/ElastiCache / CurrConnections { CacheClusterId: prod-redis }
```

### Namespaces (Không Gian Tên) Quan Trọng

| Namespace           | Dịch Vụ                           |
| ------------------- | --------------------------------- |
| `AWS/RDS`           | RDS & Aurora instances            |
| `AWS/DynamoDB`      | DynamoDB tables & indexes         |
| `AWS/ElastiCache`   | ElastiCache Redis & Memcached     |
| `AWS/DocDB`         | DocumentDB                        |
| `AWS/Redshift`      | Redshift clusters                 |

### Statistics (Thống Kê) Có Sẵn

```
Average  — Trung bình trong khoảng thời gian
Sum      — Tổng (dùng cho counts: số request, số lỗi)
Minimum  — Giá trị nhỏ nhất
Maximum  — Giá trị lớn nhất (dùng cho latency worst case)
p50, p90, p95, p99 — Percentiles (Phân Vị) — quan trọng cho latency

→ Latency: Dùng p99 (99th percentile) để đo worst-case
→ Count: Dùng Sum
→ Utilization %: Dùng Average
```

---

## RDS & Aurora Key Metrics

### Nhóm 1: Compute — Tính Toán

#### CPUUtilization — Mức Sử Dụng CPU

```
Namespace: AWS/RDS
Metric:    CPUUtilization
Unit:      Percent (%)

Ngưỡng cảnh báo thông thường:
  Warning:  > 70% trong 10 phút
  Critical: > 90% trong 5 phút

Nguyên nhân thường gặp khi CPU cao:
  - Query không có index đúng (full table scan — quét toàn bảng)
  - N+1 query problem (vấn đề truy vấn N+1)
  - Missing index trên WHERE clause
  - Bulk operations không được optimize

Cách debug:
  1. Bật Performance Insights → xem top SQL by CPU
  2. Kiểm tra slow query log
  3. Dùng EXPLAIN để phân tích query plan
```

#### FreeableMemory — Bộ Nhớ Khả Dụng

```
Namespace: AWS/RDS
Metric:    FreeableMemory
Unit:      Bytes

Ngưỡng cảnh báo:
  Warning:  < 25% total RAM
  Critical: < 10% total RAM
  Emergency: < 5% total RAM (nguy cơ OOM killer)

Lưu ý:
  - MySQL/MariaDB: InnoDB buffer pool chiếm nhiều RAM nhất
  - PostgreSQL: shared_buffers + work_mem × connections
  - Giảm FreeableMemory không phải lúc nào cũng xấu
    nếu là buffer cache (sẽ giải phóng khi cần)

Khi FreeableMemory thấp liên tục:
  → Upsize instance class
  → Hoặc giảm innodb_buffer_pool_size / shared_buffers
```

#### SwapUsage — Sử Dụng Bộ Nhớ Ảo

```
Namespace: AWS/RDS
Metric:    SwapUsage
Unit:      Bytes

Ngưỡng:
  Warning:  > 256 MB
  Critical: > 1 GB

Swap usage cao = RAM không đủ = performance degradation nghiêm trọng
→ Disk I/O chậm hơn RAM 100x-1000x lần
→ Cần upsize instance ngay lập tức nếu swap thường xuyên cao
```

---

### Nhóm 2: Storage I/O — Đọc/Ghi Lưu Trữ

#### ReadIOPS & WriteIOPS — Số Thao Tác Đọc/Ghi

```
Metrics:
  ReadIOPS   — Số lần đọc từ disk/giây
  WriteIOPS  — Số lần ghi vào disk/giây

Liên quan đến provisioned storage:
  gp2: Baseline 3 IOPS/GB, burst lên 3,000 IOPS
  gp3: Cấu hình riêng, baseline 3,000 IOPS
  io1/io2: Tối đa 64,000 IOPS (tùy instance)

Cảnh báo khi IOPS > 80% provisioned capacity
→ Có thể cần upgrade storage type hoặc tăng provisioned IOPS
```

#### ReadLatency & WriteLatency — Độ Trễ Đọc/Ghi

```
Metrics:
  ReadLatency   — Thời gian trung bình mỗi disk read (giây)
  WriteLatency  — Thời gian trung bình mỗi disk write (giây)

Ngưỡng thông thường (gp3/io2):
  Bình thường:  < 1ms (0.001 giây)
  Warning:      > 5ms
  Critical:     > 20ms

Latency cao nguyên nhân:
  - IOPS đã đạt giới hạn (I/O saturation — bão hòa I/O)
  - Storage type không phù hợp workload
  - Instance size quá nhỏ
```

#### FreeStorageSpace — Dung Lượng Còn Trống

```
Metric: FreeStorageSpace
Unit:   Bytes

Ngưỡng cảnh báo:
  Warning:  < 20% total storage
  Critical: < 10 GB (tuyệt đối)
  Emergency: < 5 GB

Khi hết disk:
  - MySQL: Không thể ghi transaction logs → rollback toàn bộ
  - PostgreSQL: Crash khi hết space → manual recovery cần thiết
  - Aurora: Storage tự động mở rộng → ít lo hơn nhưng vẫn cần monitor

Bật Storage Auto Scaling (Tự Động Mở Rộng Lưu Trữ) cho RDS:
  - Tự động tăng khi FreeStorageSpace < 10% hoặc < 5 GB
  - Maximum storage limit cần đặt hợp lý
```

---

### Nhóm 3: Database Connections — Kết Nối Cơ Sở Dữ Liệu

#### DatabaseConnections — Số Kết Nối Đang Hoạt Động

```
Metric: DatabaseConnections
Unit:   Count

Max connections theo instance class:
  db.t3.micro:   66 connections  (MySQL)
  db.t3.small:   150 connections
  db.t3.medium:  312 connections
  db.r6g.large:  1,320 connections
  db.r6g.4xlarge: 10,542 connections

Ngưỡng cảnh báo:
  Warning:  > 70% max_connections
  Critical: > 90% max_connections

Khi connections đầy:
  - New connections bị từ chối với lỗi "Too many connections"
  - Giải pháp: RDS Proxy (Proxy RDS) để pooling kết nối
```

#### ReplicaLag — Độ Trễ Bản Sao

```
Metric: ReplicaLag (chỉ có trên Read Replicas)
Unit:   Seconds (giây)

Ngưỡng:
  Warning:  > 30 giây
  Critical: > 5 phút
  Emergency: > 30 phút

ReplicaLag cao nguyên nhân:
  - Read Replica bị overloaded (quá tải)
  - Network latency giữa primary và replica
  - Long-running transactions trên primary

Hệ quả của ReplicaLag cao:
  - Đọc từ replica trả về data cũ (stale data — dữ liệu cũ)
  - Ảnh hưởng đến consistency trong ứng dụng
```

---

### Nhóm 4: Aurora-Specific Metrics — Số Liệu Đặc Thù Aurora

```
AuroraReplicaLag        — Độ trễ Aurora Replica (thường < 100ms)
AuroraReplicaLagMaximum — Lag của replica chậm nhất
CommitLatency           — Thời gian để commit transaction
DDLLatency              — Thời gian cho Data Definition Language operations
SelectLatency           — Thời gian thực hiện SELECT queries
UpdateLatency           — Thời gian thực hiện UPDATE queries
DeleteLatency           — Thời gian thực hiện DELETE queries
InsertLatency           — Thời gian thực hiện INSERT queries

VolumeBytesUsed         — Bytes dùng bởi Aurora cluster
VolumeReadIOPs          — Read IOPS từ Aurora storage
VolumeWriteIOPs         — Write IOPS vào Aurora storage

EngineUptime            — Số giây instance đã chạy
LoginFailures           — Số lần đăng nhập thất bại (bảo mật)
```

---

## DynamoDB Key Metrics

### Nhóm 1: Capacity Consumption — Tiêu Thụ Công Suất

#### ConsumedReadCapacityUnits & ConsumedWriteCapacityUnits

```
Metrics:
  ConsumedReadCapacityUnits  (ConsumedRCU)
  ConsumedWriteCapacityUnits (ConsumedWCU)

Statistics nên dùng: Sum (theo 1-5 phút)

Công thức kiểm tra utilization:
  Utilization% = (ConsumedRCU / ProvisionedRCU) × 100

Ngưỡng cảnh báo (Provisioned mode — Chế Độ Cung Cấp Sẵn):
  Warning:  Utilization > 80%
  Critical: Utilization > 95%

Với On-Demand mode (Chế Độ Theo Yêu Cầu):
  - Không giới hạn cứng
  - Nhưng monitor chi phí qua BillingMode metric
```

#### ProvisionedReadCapacityUnits & ProvisionedWriteCapacityUnits

```
Dùng để tính toán utilization ratio (tỷ lệ sử dụng)
Thay đổi khi:
  - Auto Scaling điều chỉnh capacity
  - Bạn thay đổi thủ công

Xem xu hướng:
  - Nếu ConsumedRCU luôn < 20% ProvisionedRCU → Over-provisioned (cung cấp thừa)
  - Nếu ConsumedRCU thường > 80% → Nguy cơ throttling
```

---

### Nhóm 2: Throttling — Giới Hạn Tốc Độ

#### ReadThrottleEvents & WriteThrottleEvents

```
Metrics:
  ReadThrottleEvents
  WriteThrottleEvents
  ThrottledRequests (sum của cả hai)

Unit: Count

Ngưỡng:
  Warning:  > 0 trong 5 phút
  Critical: > 100 trong 1 phút

Throttling xảy ra khi:
  - Vượt quá ProvisionedCapacity
  - Hot partition (phân vùng nóng) gây local throttling
  - Burst consumption quá mức

SDK retry behavior (Hành Vi Thử Lại của SDK):
  - AWS SDK tự retry với exponential backoff
  - Nhưng high throttling → latency cao → trải nghiệm người dùng kém
  - ThrottledRequests > 0 liên tục = cần hành động
```

#### SystemErrors — Lỗi Hệ Thống

```
Metric: SystemErrors
Unit:   Count

HTTP 500 errors từ phía DynamoDB service
Rất hiếm nhưng cần monitor
→ Nếu cao: Liên hệ AWS Support
```

---

### Nhóm 3: Latency — Độ Trễ

```
SuccessfulRequestLatency — Thời gian DynamoDB xử lý request

Thống kê quan trọng:
  Average — Baseline latency thông thường
  p99     — Worst-case latency (99% requests nhanh hơn giá trị này)

Ngưỡng thông thường:
  GetItem (đọc đơn):    Average < 5ms,  p99 < 10ms
  PutItem (ghi đơn):    Average < 5ms,  p99 < 10ms
  Query:                Average < 10ms, p99 < 50ms
  Scan (quét toàn bảng): Cao hơn, tùy table size

Latency tăng nguyên nhân:
  - Hot partition → local contention
  - Large items (item quá lớn, > 400KB)
  - Query không efficient (không dùng index đúng)
```

---

## ElastiCache Key Metrics

### Redis Core Metrics — Số Liệu Cốt Lõi Redis

#### CurrConnections — Số Kết Nối Hiện Tại

```
Metric: CurrConnections
Unit:   Count

Ngưỡng:
  Warning:  > 80% max connections
  Critical: > 95%

Redis default max connections: 65,000
Nhưng thực tế giới hạn bởi memory và file descriptors

Khi connections đột ngột spike (tăng vọt):
  - Connection leak trong ứng dụng
  - Deploy mới có bug tạo connections không đóng
```

#### CacheHitRate — Tỷ Lệ Cache Hit

```
Tính toán:
  CacheHitRate = CacheHits / (CacheHits + CacheMisses) × 100

Không có sẵn trong CloudWatch, phải tính:
  CacheHits   = Metric: CacheHits
  CacheMisses = Metric: CacheMisses

Ngưỡng:
  Tốt:    > 90%
  Warning: 70-90%
  Bad:    < 70%

CacheHitRate thấp nguyên nhân:
  - TTL (Time To Live — Thời Gian Tồn Tại) quá ngắn
  - Evictions (trục xuất) do hết memory
  - Access pattern thay đổi (không predictable)
  - Cache cold start sau restart
```

#### Evictions — Trục Xuất Cache

```
Metric: Evictions
Unit:   Count

Eviction = Redis buộc phải xóa key cũ để có chỗ cho key mới
          khi memory đầy và maxmemory-policy được kích hoạt

Ngưỡng:
  Tốt:    = 0 liên tục
  Warning: > 0 đều đặn
  Critical: Tăng nhanh (cache thrashing — vật lộn cache)

Khi Evictions cao:
  → Cache size không đủ cho working set (tập dữ liệu làm việc)
  → Cần upsize node type
  → Hoặc xem xét lại TTL strategy
```

#### BytesUsedForCache — Bộ Nhớ Đã Dùng

```
Metric: BytesUsedForCache
Unit:   Bytes

Theo dõi Memory utilization:
  Utilization% = BytesUsedForCache / maxmemory × 100

Ngưỡng:
  Warning:  > 75%
  Critical: > 90%
  (Để buffer cho key spike và replication)
```

#### ReplicationLag — Độ Trễ Sao Chép (Redis Cluster)

```
Metric: ReplicationLag
Unit:   Seconds

Cho Redis Replication Groups (Nhóm Sao Chép Redis)

Ngưỡng:
  Normal:   < 1 giây
  Warning:  > 10 giây
  Critical: > 60 giây

Replication lag cao → Đọc từ replica có thể trả về stale data
```

---

## Tạo Dashboard Toàn Diện

### Dashboard 1: Database Operations (Hoạt Động Database)

```
Thiết kế dashboard cho on-call engineer (kỹ sư trực):

Row 1: Health Overview (Tổng Quan Sức Khỏe)
  ├── RDS CPUUtilization (line graph, 24h)
  ├── RDS DatabaseConnections (line graph, 24h)
  ├── RDS FreeStorageSpace (number widget)
  └── DynamoDB ThrottledRequests (number, last hour)

Row 2: Performance (Hiệu Năng)
  ├── RDS ReadLatency + WriteLatency (line graph, 6h)
  ├── DynamoDB SuccessfulRequestLatency P99 (line graph)
  ├── ElastiCache CacheHitRate (gauge widget)
  └── ElastiCache Evictions (bar chart)

Row 3: Capacity (Công Suất)
  ├── RDS ReadIOPS + WriteIOPS vs provisioned
  ├── DynamoDB ConsumedRCU vs ProvisionedRCU
  ├── DynamoDB ConsumedWCU vs ProvisionedWCU
  └── ElastiCache BytesUsedForCache / maxmemory

Row 4: Replication (Sao Chép)
  ├── RDS ReplicaLag (if applicable)
  ├── Aurora AuroraReplicaLag
  └── ElastiCache ReplicationLag
```

### Tạo Dashboard bằng CloudFormation (Hạ Tầng Dưới Dạng Mã)

```yaml
# cloudformation/monitoring-dashboard.yaml
Resources:
  DatabaseDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: prod-database-overview
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "properties": {
                "title": "RDS CPU Utilization",
                "metrics": [
                  ["AWS/RDS", "CPUUtilization",
                   "DBInstanceIdentifier", "${RDSInstanceId}"]
                ],
                "period": 300,
                "stat": "Average",
                "view": "timeSeries",
                "yAxis": { "left": { "min": 0, "max": 100 } }
              }
            },
            {
              "type": "metric",
              "properties": {
                "title": "DynamoDB Throttled Requests",
                "metrics": [
                  ["AWS/DynamoDB", "ThrottledRequests",
                   "TableName", "${DynamoDBTableName}",
                   { "stat": "Sum", "period": 60 }]
                ]
              }
            }
          ]
        }
```

### Tạo Dashboard bằng AWS CLI (Giao Diện Dòng Lệnh AWS)

```bash
# Tạo dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "prod-database" \
  --dashboard-body file://dashboard.json

# Xem danh sách dashboards
aws cloudwatch list-dashboards

# Lấy dashboard data
aws cloudwatch get-dashboard \
  --dashboard-name "prod-database"
```

---

## CloudWatch Logs Insights

CloudWatch Logs Insights là công cụ truy vấn logs theo cú pháp riêng, mạnh mẽ để phân tích database logs.

### Truy Vấn RDS Slow Query Logs

```
# Tìm queries chậm nhất (> 5 giây)
fields @timestamp, @message
| parse @message "Query_time: * Lock_time:" as queryTime
| filter queryTime > 5
| sort queryTime desc
| limit 20
```

### Truy Vấn RDS Error Logs

```
# Tìm các lỗi connection trong 24 giờ
fields @timestamp, @message
| filter @logStream = "error"
| filter @message like /ERROR/
| stats count() as errorCount by bin(1h)
| sort @timestamp desc
```

### Truy Vấn CloudTrail Logs Cho DynamoDB

```
# Phát hiện các operation bất thường trên DynamoDB
fields eventTime, userIdentity.arn, eventName, requestParameters.tableName
| filter eventSource = "dynamodb.amazonaws.com"
| filter eventName in ["DeleteTable", "PutItem", "BatchWriteItem"]
| sort eventTime desc
| limit 50
```

---

## Best Practices

### 1. Hierarchical Dashboards (Dashboards Phân Cấp)

```
Level 1: Executive Dashboard — SLA/SLO tổng hợp (1 page)
Level 2: Service Dashboard — metrics theo dịch vụ (RDS, DynamoDB...)
Level 3: Instance Dashboard — chi tiết từng instance/table
Level 4: Debugging Dashboard — granular metrics khi có incident
```

### 2. Metric Math (Toán Học Số Liệu)

CloudWatch hỗ trợ tính toán trực tiếp trên metrics:

```
# Tính CacheHitRate từ CacheHits và CacheMisses
CacheHitRate = (m1 / (m1 + m2)) * 100
  where m1 = CacheHits, m2 = CacheMisses

# Tính DynamoDB Write Utilization %
WriteUtil = (m1 / m2) * 100
  where m1 = ConsumedWriteCapacityUnits (Sum)
  where m2 = ProvisionedWriteCapacityUnits (Average)
```

### 3. Cross-Account Dashboards (Dashboard Xuyên Tài Khoản)

```
Nếu có nhiều AWS accounts (dev, staging, prod):
  - Dùng CloudWatch cross-account observability
  - Consolidate metrics vào monitoring account trung tâm
  - Giảm context switching khi on-call
```

### 4. Retention Policy (Chính Sách Lưu Giữ)

```
CloudWatch Metrics tự động:
  High-resolution (1 giây):  lưu 3 giờ
  Standard (60 giây):        lưu 15 ngày
  Aggregated (5 phút):       lưu 63 ngày
  Aggregated (1 giờ):        lưu 455 ngày (~15 tháng)
  Aggregated (1 ngày):       lưu 455 ngày

→ Dữ liệu tự động downsampled (giảm độ phân giải) theo thời gian
→ Cho capacity planning dài hạn: export sang S3 hoặc dùng CloudWatch Metrics Insights
```

### 5. Tagging Strategy (Chiến Lược Gắn Nhãn)

```
Gắn tags (nhãn) nhất quán cho tất cả database resources:
  Environment: prod / staging / dev
  Team:        backend / data / platform
  Service:     user-service / order-service
  CostCenter:  engineering-123

Lợi ích:
  - Filter dashboards theo tag
  - Phân tích chi phí theo team/service
  - Automation rules dựa trên tags
```

---

## Checklist Thiết Lập CloudWatch

```
□ Bật CloudWatch agent trên tất cả RDS instances
□ Bật Enhanced Monitoring (granularity 60s cho prod)
□ Bật Performance Insights (retention 7 ngày)
□ Tạo dashboard tiêu chuẩn cho mỗi database cluster
□ Thiết lập metric math để tính utilization %
□ Cấu hình log retention cho RDS logs (30 ngày tối thiểu)
□ Bật Contributor Insights cho DynamoDB tables lớn
□ Thiết lập cross-account dashboard nếu multi-account
□ Tài liệu hóa threshold rationale cho mỗi alarm
□ Review và update dashboards mỗi quý
```

---

**Xem Tiếp:** [2-alerting-strategy.md](2-alerting-strategy.md) — Ngưỡng Cảnh Báo & SLO

**Cập Nhật Lần Cuối:** 2026-05-15
