# DynamoDB Monitoring — Giám Sát DynamoDB Toàn Diện

> Hướng dẫn chi tiết về giám sát Amazon DynamoDB — CloudWatch Alarms (Cảnh Báo CloudWatch), Capacity Alerts (Cảnh Báo Công Suất), Contributor Insights (Thông Tin Người Đóng Góp) để phát hiện hot partitions (phân vùng nóng), và chiến lược đối phó throttling (giới hạn tốc độ).

---

## 📚 Mục Lục

1. [DynamoDB Metrics Tổng Quan](#dynamodb-metrics-tổng-quan)
2. [Capacity Monitoring](#capacity-monitoring)
3. [Throttling Detection & Response](#throttling-detection--response)
4. [Latency Monitoring](#latency-monitoring)
5. [Contributor Insights — Phát Hiện Hot Keys](#contributor-insights--phát-hiện-hot-keys)
6. [Global Secondary Index Monitoring](#global-secondary-index-monitoring)
7. [DynamoDB Streams Monitoring](#dynamodb-streams-monitoring)
8. [Alerting Checklist](#alerting-checklist)

---

## DynamoDB Metrics Tổng Quan

### CloudWatch Namespaces

DynamoDB metrics chia thành 2 namespaces:

```
AWS/DynamoDB — Metrics cho tables và indexes
  Dimensions: TableName, Operation, GlobalSecondaryIndexName

AWS/DynamoDB (Account-level metrics) — Chỉ có trong us-east-1
  Dành cho account-level capacity limits
```

### Bản Đồ Metrics Theo Nhóm

```
DynamoDB Metrics
├── Capacity Consumption (Tiêu Thụ Công Suất)
│   ├── ConsumedReadCapacityUnits
│   ├── ConsumedWriteCapacityUnits
│   ├── ProvisionedReadCapacityUnits
│   └── ProvisionedWriteCapacityUnits
│
├── Throttling (Giới Hạn Tốc Độ)
│   ├── ThrottledRequests
│   ├── ReadThrottleEvents
│   ├── WriteThrottleEvents
│   └── OnlineIndexThrottleEvents
│
├── Latency (Độ Trễ)
│   ├── SuccessfulRequestLatency (per operation type)
│   └── SystemErrors
│
├── Errors (Lỗi)
│   ├── SystemErrors (HTTP 500)
│   └── UserErrors (HTTP 400)
│
└── Replication (Sao Chép) — Global Tables
    ├── ReplicationLatency
    └── PendingReplicationCount
```

---

## Capacity Monitoring

### Provisioned Mode (Chế Độ Cung Cấp Sẵn)

Trong Provisioned Mode, bạn đặt trước RCU (Read Capacity Units — Đơn Vị Đọc) và WCU (Write Capacity Units — Đơn Vị Ghi). Vượt quá → throttling.

#### ConsumedReadCapacityUnits & ConsumedWriteCapacityUnits

```
Statistics cần dùng: Sum (trong 1-5 phút)
Sau đó chia cho seconds để ra average RCU/s hoặc WCU/s

Ví dụ:
  ConsumedRCU.Sum trong 5 phút = 18,000
  Average RCU/s = 18,000 / 300 = 60 RCU/s
  ProvisionedRCU = 100
  Utilization = 60% → OK, còn buffer

Ngưỡng cảnh báo:
  Warning:  Utilization > 80% trong 10 phút
  Critical: Utilization > 95% trong 5 phút
```

#### Tính Toán Utilization Bằng Metric Math

```
Trong CloudWatch, tạo Metric Math expression:
  m1 = ConsumedReadCapacityUnits (Sum, period 60s)
  m2 = ProvisionedReadCapacityUnits (Average, period 60s)

  Utilization% = (m1 / 60 / m2) * 100
                   ↑ chia 60 để chuyển Sum → per-second

Alarm trên expression này khi > 80%
```

### On-Demand Mode (Chế Độ Theo Yêu Cầu)

Trong On-Demand Mode, không có giới hạn cứng trên throughput. Tuy nhiên vẫn cần monitor:

```
Metrics cần theo dõi trong On-Demand Mode:
  ConsumedReadCapacityUnits  — Theo dõi chi phí
  ConsumedWriteCapacityUnits — Theo dõi chi phí

Throttling vẫn có thể xảy ra trong On-Demand Mode khi:
  - Traffic tăng quá nhanh (> 2x so với peak trước đó trong 30 phút)
  - Table mới tạo chưa có traffic history
  - Hot partition vẫn có giới hạn per-partition

→ Monitor throttling dù dùng On-Demand
```

### Capacity Planning Dashboard

```
CloudWatch dashboard cho capacity planning (lập kế hoạch công suất):

Widget 1: Consumed vs Provisioned (RCU & WCU)
  - 2 lines: ConsumedRCU và ProvisionedRCU
  - Annotate khi có spikes
  - Period: 5 phút, 7 ngày

Widget 2: Utilization % (từ Metric Math)
  - Line graph
  - Horizontal annotation tại 80% (warning threshold)
  - Period: 5 phút, 24 giờ

Widget 3: Cost estimate
  - ConsumedRCU + ConsumedWCU (Sum, 1 ngày)
  - Ước tính chi phí hàng ngày

Widget 4: Auto Scaling history
  - ProvisionedRCU qua thời gian
  - Xem Auto Scaling có đang hoạt động đúng không
```

### Auto Scaling Monitoring (Giám Sát Tự Động Co Giãn)

DynamoDB Application Auto Scaling (Tự Động Co Giãn Ứng Dụng) điều chỉnh provisioned capacity:

```bash
# Xem Auto Scaling policies hiện tại
aws application-autoscaling describe-scaling-policies \
  --service-namespace dynamodb \
  --resource-id table/orders

# Xem scaling activities (hoạt động co giãn) gần đây
aws application-autoscaling describe-scaling-activities \
  --service-namespace dynamodb \
  --resource-id table/orders

Output quan trọng:
  StatusCode: Successful / Failed
  Cause:      "monitor alarm ... triggered"
  StartTime:  Khi nào bắt đầu scale
  EndTime:    Khi nào hoàn thành scale
```

**CloudWatch Alarm cho Auto Scaling events:**
```bash
# Auto Scaling sẽ trigger alarm này khi cần scale up
# Alarm tên: DynamoDB:AlarmName:prod-orders-scaling-alarm-read
# Đây là alarm tự động tạo bởi Auto Scaling

# Bạn nên monitor:
#   1. Số lần scale-up trong ngày (nhiều lần = underprovisioned)
#   2. Throttling trước khi Auto Scaling kịp phản ứng
#   3. (Auto Scaling cần vài phút để có hiệu lực)
```

---

## Throttling Detection & Response

### Hiểu Throttling Trong DynamoDB

```
Throttling xảy ra khi:

1. Table-level throttling:
   ConsumedCapacity > ProvisionedCapacity
   → Toàn bộ table bị ảnh hưởng

2. Partition-level throttling (nguy hiểm hơn):
   DynamoDB phân phối capacity đều theo partition
   Mỗi partition có ~1000 WCU và ~3000 RCU tối đa
   → Hot partition vượt giới hạn này dù table vẫn còn capacity

3. GSI throttling (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu):
   GSI có capacity riêng biệt với base table
   → GSI throttling không ảnh hưởng base table (và ngược lại)
```

### Metrics Throttling

```
ThrottledRequests:
  Tổng số requests bị throttle
  Bao gồm cả read và write
  Alarm: ThrottledRequests.Sum > 0 trong 2 phút

ReadThrottleEvents:
  Số read requests bị throttle
  Dùng để phân biệt read vs write throttling

WriteThrottleEvents:
  Số write requests bị throttle

OnlineIndexThrottleEvents:
  Throttling khi DynamoDB đang rebuild GSI (index rebuild)
```

### CloudWatch Alarms Cho Throttling

```bash
# Alarm: Bất kỳ throttling nào
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-orders-any-throttle" \
  --metric-name ThrottledRequests \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=orders \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-warning

# Alarm: Throttling nghiêm trọng (> 100 events/phút)
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-orders-severe-throttle" \
  --metric-name ThrottledRequests \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=orders \
  --period 60 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-critical
```

### Response Playbook (Sổ Phản Ứng) Khi Có Throttling

```
Bước 1: Xác định nguồn throttling

  Kiểm tra CloudWatch:
    ReadThrottleEvents vs WriteThrottleEvents → loại nào nhiều hơn?
    Table metrics vs GSI metrics → nơi nào?

  Kiểm tra Contributor Insights:
    Top throttled read/write partition keys

Bước 2: Short-term fix (Sửa Nhanh)

  Option A — Tăng Provisioned Capacity ngay:
    aws dynamodb update-table \
      --table-name orders \
      --provisioned-throughput ReadCapacityUnits=500,WriteCapacityUnits=200

  Option B — Chuyển sang On-Demand Mode tạm thời:
    aws dynamodb update-table \
      --table-name orders \
      --billing-mode PAY_PER_REQUEST
    (Chú ý: tốn kém hơn nếu traffic cao)

  Option C — Thêm DAX (DynamoDB Accelerator):
    Giảm read throughput cần thiết 10x-100x
    Nhưng cần thời gian setup

Bước 3: Root cause investigation (Điều Tra Nguyên Nhân Gốc)

  Nếu hot partition: Xem Contributor Insights → redesign partition key
  Nếu traffic spike: Đánh giá lại capacity planning
  Nếu GSI throttle: Tăng GSI capacity riêng
```

---

## Latency Monitoring

### SuccessfulRequestLatency Per Operation

```
DynamoDB báo cáo latency riêng cho từng operation type:

Operations: GetItem, PutItem, Query, Scan, UpdateItem,
            DeleteItem, BatchGetItem, BatchWriteItem,
            TransactGetItems, TransactWriteItems

Cách tạo alarm cho Query latency P99:
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-orders-query-p99-latency" \
  --metric-name SuccessfulRequestLatency \
  --namespace AWS/DynamoDB \
  --dimensions \
    Name=TableName,Value=orders \
    Name=Operation,Value=Query \
  --period 300 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --extended-statistic p99 \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-warning
```

### Latency Baseline (Đường Cơ Sở Độ Trễ)

```
DynamoDB latency targets:
  GetItem (đọc đơn):       Average 1-3ms,   p99 < 10ms
  PutItem (ghi đơn):       Average 1-5ms,   p99 < 15ms
  Query:                   Average 3-10ms,  p99 < 30ms
  BatchGetItem:            Average 5-20ms,  p99 < 50ms
  TransactWriteItems:      Average 5-15ms,  p99 < 40ms
  Scan (toàn bảng):        Phụ thuộc size

Nếu vượt ngưỡng trên:
  → Hot partition (kiểm tra Contributor Insights)
  → Item size quá lớn (> 100KB per item)
  → Inefficient query (không dùng index đúng)
  → Network issues (dùng DAX để loại trừ)
```

### DAX Metrics (DynamoDB Accelerator — Bộ Tăng Tốc DynamoDB)

Nếu dùng DAX, monitor thêm:

```
Namespace: AWS/DAX

Metrics:
  CPUUtilization          — CPU của DAX node
  CacheHits               — Requests phục vụ từ cache
  CacheMisses             — Requests cần đến DynamoDB
  ItemCacheHits           — GetItem cache hits
  ItemCacheMisses         — GetItem cache misses
  QueryCacheHits          — Query cache hits
  QueryCacheMisses        — Query cache misses
  TotalRequestCount       — Tổng requests đến DAX

Tính Cache Hit Rate:
  CacheHitRate = CacheHits / (CacheHits + CacheMisses) * 100
  Mục tiêu: > 90%
```

---

## Contributor Insights — Phát Hiện Hot Keys

### Contributor Insights là gì

Contributor Insights (Thông Tin Người Đóng Góp) là tính năng CloudWatch giúp xác định **top keys gây ra hầu hết traffic** (hot keys — khóa nóng) trong DynamoDB.

```
Không có Contributor Insights:
  ThrottledRequests = 500/phút
  → Biết có throttling nhưng không biết key nào gây ra

Với Contributor Insights:
  Top 10 throttled partition keys:
  1. user_id=USER_99999:  250/phút (50%)
  2. user_id=USER_12345:   80/phút (16%)
  3. order_id=ORD_ABC:     40/phút  (8%)
  ...
  → Ngay lập tức biết cần xử lý gì
```

### Bật Contributor Insights

```bash
# Bật cho table
aws dynamodb enable-kinesis-streaming-destination \
  --table-name orders \
  --stream-arn arn:aws:kinesis:...  # Nếu dùng với Kinesis

# Bật Contributor Insights qua CloudWatch
aws cloudwatch enable-insight-rules \
  --rule-names \
    "DynamoDBContributorInsights-ReadThrottleEvents-orders-index" \
    "DynamoDBContributorInsights-WriteThrottleEvents-orders-index"

# Hoặc đơn giản hơn: bật qua DynamoDB Console
# DynamoDB Console → Tables → orders → Additional settings → Contributor Insights → Enable
```

### Đọc Kết Quả Contributor Insights

```
CloudWatch Console → Contributor Insights → chọn rule

Xem được:
  - Top 10 (hoặc 100) contributors
  - Partition key value + sort key value (nếu có)
  - Count of throttle events hoặc RCU/WCU consumed

Kết quả mẫu:
  Rank | Partition Key | Operations
    1  | USER#VIP001   | 4,520 WCU/phút ← HOT!
    2  | USER#VIP002   | 3,100 WCU/phút
    3  | PRODUCT#A01   |   890 WCU/phút
    4  | ORDER#ORD001  |   240 WCU/phút
    ...

→ USER#VIP001 và USER#VIP002 là hot keys
→ Cần xem xét partition key design
```

### Xử Lý Hot Keys

```
Sau khi Contributor Insights xác định hot key:

Giải pháp 1: Write Sharding (Phân Mảnh Ghi)
  Thay vì: PK = "PRODUCT#A01"
  Dùng:    PK = "PRODUCT#A01#SHARD_3"  (random 0-9)

  Lợi ích: Phân phối load sang 10 partitions thay vì 1
  Bất lợi: Phức tạp hơn khi đọc (cần query tất cả shards)

Giải pháp 2: Caching với DAX hoặc ElastiCache
  Nếu hot key là read-heavy (đọc nhiều):
    → Đặt trước ElastiCache/DAX
    → Chỉ access DynamoDB khi cache miss

Giải pháp 3: Redesign Data Model (Thiết Kế Lại Mô Hình Dữ Liệu)
  Nếu hot key do bad partition key design:
    → Dùng composite key với nhiều cardinality hơn
    → Ví dụ: Thêm timestamp hoặc category vào partition key

Giải pháp 4: Tăng Capacity Tạm Thời
  Với Provisioned Mode: Tăng WCU/RCU
  Với On-Demand: Tự động (nhưng hot partition vẫn có giới hạn)
```

---

## Global Secondary Index Monitoring

### Tại Sao GSI Cần Monitor Riêng

GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) có capacity **hoàn toàn độc lập** với base table:

```
Base table: PK=user_id, SK=order_id
  → ProvisionedRCU=100, ProvisionedWCU=50

GSI: GSI-PK=status, GSI-SK=created_at (để query theo trạng thái)
  → ProvisionedRCU=50, ProvisionedWCU=50 (riêng biệt!)

Khi write đến base table:
  → DynamoDB cũng ghi vào GSI
  → Nếu GSI WCU không đủ → GSI throttle → write bị chậm!

→ GSI write throttling ảnh hưởng đến write của base table
```

### GSI Metrics

```bash
# Consumed RCU/WCU của GSI riêng
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions \
    Name=TableName,Value=orders \
    Name=GlobalSecondaryIndexName,Value=status-created_at-index \
  --start-time 2026-05-15T00:00:00Z \
  --end-time 2026-05-15T01:00:00Z \
  --period 300 \
  --statistics Sum
```

### GSI Alarm

```bash
# Alarm cho GSI write throttling
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-orders-gsi-write-throttle" \
  --metric-name WriteThrottleEvents \
  --namespace AWS/DynamoDB \
  --dimensions \
    Name=TableName,Value=orders \
    Name=GlobalSecondaryIndexName,Value=status-created_at-index \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-warning
```

### GSI Capacity Best Practices

```
Rule of thumb (quy tắc ngón tay cái):
  GSI WCU = Base table WCU × (% of writes that touch this GSI attribute)

Ví dụ:
  Base table WCU = 100
  50% writes update "status" attribute (được index trong GSI)
  → GSI WCU cần ≈ 50 WCU (có thể cần thêm buffer)

Đặc biệt lưu ý với Sparse Index (Chỉ Mục Thưa):
  - Chỉ items có attribute đó mới được index trong GSI
  - Ít RCU/WCU hơn nhưng vẫn phải monitor
```

---

## DynamoDB Streams Monitoring

DynamoDB Streams (Luồng Dữ Liệu DynamoDB) cung cấp CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu):

### Streams Metrics

```
Namespace: AWS/DynamoDB

SuccessfulRequestLatency với Operation=GetRecords:
  → Thời gian để đọc records từ Stream

SystemErrors với Operation=GetRecords:
  → Lỗi khi đọc Stream

Nếu Stream + Lambda:
  → Monitor Lambda function metrics riêng:
     IteratorAge (Tuổi Iterator): Thời gian delay từ khi ghi đến khi Lambda xử lý
     Errors: Lambda execution errors
     Throttles: Lambda concurrency throttles
```

### Lambda Iterator Age — Chỉ Số Quan Trọng

```
IteratorAge = thời gian từ khi record được ghi vào Stream
              đến khi Lambda function xử lý nó

Ý nghĩa:
  IteratorAge thấp (< 1 phút): Lambda xử lý gần real-time
  IteratorAge cao (> 5 phút):  Lambda đang bị lag, backlog đang tăng

Nguyên nhân IteratorAge tăng:
  - Lambda function chậm (slow processing)
  - Lambda throttling (concurrency limit)
  - Lambda errors gây retry

Alarm:
  aws cloudwatch put-metric-alarm \
    --alarm-name "DynamoDB-stream-lambda-lag" \
    --metric-name IteratorAge \
    --namespace AWS/Lambda \
    --dimensions Name=FunctionName,Value=orders-stream-processor \
    --threshold 300000 \
    --comparison-operator GreaterThanThreshold \
    --statistic Maximum \
    (300000 ms = 5 phút)
```

---

## Alerting Checklist

```
□ ConsumedRCU Utilization alarm > 80% (Provisioned mode)
□ ConsumedWCU Utilization alarm > 80% (Provisioned mode)
□ ThrottledRequests alarm > 0 (bất kỳ throttling nào)
□ ThrottledRequests alarm > 100/phút (throttling nghiêm trọng)
□ SuccessfulRequestLatency P99 alarm (per operation)
□ SystemErrors alarm > 0 (HTTP 500)
□ GSI throttling alarms (mỗi GSI quan trọng)
□ Bật Contributor Insights cho tables có traffic cao
□ IteratorAge alarm nếu dùng DynamoDB Streams + Lambda
□ Global Tables ReplicationLatency alarm nếu multi-region

Nếu dùng DAX:
□ DAX CacheHitRate < 90%
□ DAX CPUUtilization > 80%
□ DAX CacheMisses đột ngột tăng
```

---

**Xem Trước:** [3-rds-enhanced-monitoring.md](3-rds-enhanced-monitoring.md) — Enhanced Monitoring
**Xem Tiếp:** [5-activity-streams.md](5-activity-streams.md) — Database Activity Streams

**Cập Nhật Lần Cuối:** 2026-05-15
