# DynamoDB Cost Optimization — Tối Ưu Chi Phí DynamoDB

> Hướng dẫn toàn diện tối ưu chi phí Amazon DynamoDB — từ việc chọn đúng capacity mode (chế độ năng lực), thiết kế bảng thông minh, đến việc dùng TTL (Time To Live — Thời Gian Sống), DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB), và tránh những bẫy chi phí phổ biến.

## 📚 Mục Lục

1. [Cấu Trúc Giá DynamoDB](#cấu-trúc-giá-dynamodb)
2. [On-Demand vs Provisioned — Khi Nào Dùng Gì](#on-demand-vs-provisioned)
3. [Auto Scaling — Tự Động Co Giãn](#auto-scaling)
4. [Bẫy Chi Phí Thường Gặp](#bẫy-chi-phí-thường-gặp)
5. [TTL — Tiết Kiệm Storage Tự Động](#ttl--tiết-kiệm-storage-tự-động)
6. [DAX — Chi Phí vs Lợi Ích](#dax--chi-phí-vs-lợi-ích)
7. [Global Tables — Chi Phí Multi-Region](#global-tables--chi-phí-multi-region)
8. [Chiến Lược Thiết Kế Tiết Kiệm Chi Phí](#chiến-lược-thiết-kế-tiết-kiệm-chi-phí)

---

## Cấu Trúc Giá DynamoDB

```
Tổng Chi Phí DynamoDB = Read/Write operations
                      + Storage
                      + Streams (nếu bật)
                      + Backup/PITR
                      + DAX (nếu dùng)
                      + Global Tables replication
                      + Data Transfer
```

### Chi Tiết Từng Thành Phần (us-east-1)

#### Read/Write Capacity

| Mode                 | Write                            | Read                             |
| -------------------- | -------------------------------- | -------------------------------- |
| On-Demand            | $1.25/million WRU                | $0.25/million RRU                |
| Provisioned          | $0.00065/WCU/hour                | $0.00013/RCU/hour                |

```
Giải thích đơn vị:
WCU — Write Capacity Unit (Đơn Vị Năng Lực Ghi):
  1 WCU = 1 write/giây với item ≤ 1 KB
  Item 2.5 KB = 3 WCU (làm tròn lên)

RCU — Read Capacity Unit (Đơn Vị Năng Lực Đọc):
  1 RCU = 1 strongly consistent read/giây với item ≤ 4 KB
  1 RCU = 2 eventually consistent reads/giây với item ≤ 4 KB

WRU — Write Request Unit (On-Demand):
  Tương đương 1 WCU nhưng tính theo requests thay vì provisioned

RRU — Read Request Unit (On-Demand):
  Tương đương 1 RCU nhưng tính theo requests
```

#### Storage

```
$0.25/GB-month
First 25 GB MIỄN PHÍ mỗi tháng (Free Tier — Tầng Miễn Phí)

Tính storage:
- Mỗi item có overhead = 100 bytes (primary key + metadata)
- Actual item size (item size thực tế) + indexes
- GSI lưu trữ riêng: mỗi GSI thêm storage cost
```

#### DynamoDB Streams

```
$0.02/100,000 read requests (từ Streams)
Không tính phí cho write operations vào Streams
Dữ liệu trong Streams giữ 24 giờ (không tính phí storage)
```

#### Backup & PITR

```
On-Demand Backup (Sao Lưu Theo Yêu Cầu):
  $0.10/GB-month

PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm):
  $0.20/GB-month
  Giữ 35 ngày liên tục
```

---

## On-Demand vs Provisioned

### On-Demand Capacity Mode (Chế Độ Theo Yêu Cầu)

```
Đặc điểm:
- Tự động scale không giới hạn (không bao giờ bị throttle)
- Trả theo từng request — không waste capacity
- Không cần dự đoán traffic

Chi phí:
- $1.25/million writes
- $0.25/million reads

Phù hợp khi:
✓ Workload mới — không có traffic baseline
✓ Traffic rất biến động (spike ngắn, khó dự đoán)
✓ Workload không đều (busy 1-2h/ngày, còn lại idle)
✓ Event-driven, batch jobs
✓ Dev/test environments
✓ Traffic thấp (< 28M writes/tháng so với Provisioned)
```

### Provisioned Capacity Mode (Chế Độ Được Cấp Phát)

```
Đặc điểm:
- Cấp phát cụ thể WCU và RCU
- Phí theo capacity được cấp, không theo usage
- Nếu vượt capacity → ThrottlingException (Ngoại Lệ Giới Hạn)

Chi phí (On-Demand tương đương):
1 WCU × 24h × 30d = 720 WCU-hours × $0.00065 = $0.468/tháng
→ 1 WCU = $0.468/tháng = 0.374M writes/tháng

Breakeven: 1 WCU Provisioned vs On-Demand
- Provisioned 1 WCU: $0.468/tháng
- On-Demand tương đương: 0.374M × $1.25/M = $0.468
→ Breakeven ở ~374,000 writes/tháng per WCU

Phù hợp khi:
✓ Traffic dự đoán được và khá đều đặn
✓ Workload production 24/7 với baseline rõ ràng
✓ Budget cần predictable (dễ dự toán)
✓ Bổ sung với Auto Scaling để handle peaks
```

### Tính Breakeven Chi Tiết

```
Câu hỏi: Bao nhiêu WCU cần để Provisioned rẻ hơn On-Demand?

Scenario: 100M writes/tháng

On-Demand:
100M × $1.25/1M = $125/tháng

Provisioned (ước tính cần WCU):
- 100M writes/tháng ÷ (30 ngày × 24h × 3600s) = ~38.6 writes/giây
- Cần ~39 WCU (giả sử traffic đều)
- 39 WCU × $0.468/WCU/tháng = $18.25/tháng

Tiết kiệm: $125 - $18.25 = $106.75/tháng (~85%)

Nhưng thực tế traffic không đều:
- Nếu peak traffic = 10x average: cần 390 WCU
- 390 WCU × $0.468 = $182.52/tháng → ĐẮT HƠN On-Demand!

→ Key insight: Provisioned rẻ hơn ON LY KHI:
  peak/average ratio < 2-3x
  Ngược lại, On-Demand hoặc Provisioned + Auto Scaling tốt hơn
```

### Chuyển Đổi Giữa Các Modes

```bash
# Chuyển từ On-Demand sang Provisioned
aws dynamodb update-table \
  --table-name my-table \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50

# Chuyển từ Provisioned sang On-Demand
aws dynamodb update-table \
  --table-name my-table \
  --billing-mode PAY_PER_REQUEST

# Giới hạn: Chỉ có thể chuyển 2 lần trong 24 giờ
# (1 lần từ mỗi direction)
```

---

## Auto Scaling

### DynamoDB Auto Scaling (Tự Động Co Giãn DynamoDB)

```
Cách hoạt động:
- Dựa trên CloudWatch Alarms để scale WCU và RCU
- Target utilization: Cấu hình % utilization mục tiêu (ví dụ: 70%)
- Scale out khi utilization > target
- Scale in khi utilization < target

Cấu hình:
- Minimum capacity:    Không bao giờ scale xuống dưới mức này
- Maximum capacity:    Không bao giờ scale lên trên mức này
- Target utilization:  70% là khuyến nghị (giữ buffer để scale kịp)
```

```bash
# Bật Auto Scaling cho DynamoDB table
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/my-table" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 1000

aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/my-table" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --policy-name "my-table-write-scaling-policy" \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration \
    '{"TargetValue": 70.0, "PredefinedMetricSpecification": {"PredefinedMetricType": "DynamoDBWriteCapacityUtilization"}}'
```

### Auto Scaling Limitations — Hạn Chế

```
Scale-out response time (Thời Gian Phản Hồi Scale Out):
- CloudWatch alarm evaluation: 2 phút
- Scale action trigger: 1-2 phút
- Total lag: 3-4 phút từ khi traffic tăng

Vấn đề với sudden spikes (Đột Biến Bất Ngờ):
- Traffic tăng 10x trong 30 giây → Throttling xảy ra
  trước khi Auto Scaling kịp phản ứng
- Solution: Đặt minimum capacity đủ cao
  hoặc dùng On-Demand cho unpredictable spikes

Cost với Auto Scaling:
- Phí tính theo MAXIMUM capacity trong thời gian billing
- Capacity không giảm ngay khi traffic giảm (cooldown period 5 phút)
→ Có thể trả cho capacity cao hơn cần trong 5-10 phút sau spike
```

### On-Demand vs Provisioned + Auto Scaling — Decision Framework

```
Dùng On-Demand khi:
- Traffic unpredictable, spiky
- Peak/average > 5x
- Không muốn quản lý capacity planning

Dùng Provisioned + Auto Scaling khi:
- Traffic có pattern rõ ràng nhưng vẫn có variation
- Peak/average 2-5x
- Muốn tiết kiệm chi phí so với On-Demand
- Có thể chịu được vài giây throttling khi spike bất ngờ

Dùng Provisioned + Fixed (Cố Định) khi:
- Traffic cực kỳ đều đặn (peak/average < 1.5x)
- Muốn chi phí hoàn toàn dự đoán được
- Batch jobs với throughput ổn định
```

---

## Bẫy Chi Phí Thường Gặp

### 1. GSI Over-provisioning — Cấp Phát Quá Mức Cho GSI

```
GSI — Global Secondary Index (Chỉ Mục Phụ Toàn Cầu):
- Mỗi GSI có capacity RIÊNG, tính phí RIÊNG
- Mặc định: GSI thừa hưởng capacity của base table
  → Nếu table có 1000 WCU → GSI cũng có 1000 WCU
  → Tổng chi phí = 2000 WCU!

Tối ưu:
- Không dùng GSI nếu không cần thiết
- Set GSI capacity riêng phù hợp với usage thực tế
- Monitor GSI utilization riêng trong CloudWatch

Ví dụ tính toán:
Table: 100 WCU + 1 GSI với 100 WCU → 200 WCU tổng
100 WCU × $0.00065 × 720h = $46.80/tháng
+ 100 GSI WCU × $0.00065 × 720h = $46.80/tháng
Total: $93.60/tháng (thay vì $46.80 nếu không có GSI!)
```

### 2. Large Item Size — Item Quá Lớn

```
DynamoDB tính theo KB:
- 1 WCU = 1 KB item
- Item 100 KB = 100 WCU/write → 100x đắt hơn!
- Item tối đa: 400 KB

Anti-pattern:
- Lưu JSON lớn, binary data, hoặc arrays dài vào DynamoDB
- Mỗi write item 50 KB = 50 WCU

Best practice:
- Compress (nén) data trước khi lưu (giảm 70-90% size)
- Lưu large objects vào S3, chỉ lưu S3 key vào DynamoDB
- Giữ items nhỏ (< 1 KB ideal, < 10 KB max practical)

Ví dụ tiết kiệm:
Item 50 KB → sau compress → 10 KB
Write cost giảm 5x: 50 WCU/write → 10 WCU/write
```

### 3. Scan Operations — Quét Toàn Bộ Bảng

```
Scan (Quét) vs Query (Truy Vấn):
- Query: Đọc chỉ items match với partition key → hiệu quả
- Scan:  Đọc TẤT CẢ items trong table → lãng phí

Chi phí Scan:
- Scan 1 GB data = 1,000,000 KB / 4 KB per RCU = 250,000 RCU
- 250,000 × 2 (eventually consistent) = 125,000 RRU
- 125,000 / 1,000,000 × $0.25 = $0.03 cho 1 lần scan
- Nếu scan 100 lần/ngày × 30 ngày = $90/tháng chỉ từ scan!

Giải pháp:
- Thiết kế access patterns → dùng Query thay vì Scan
- Thêm GSI để query theo attributes khác
- Dùng parallel scan chỉ cho one-time data migration
- Export to S3 + Athena cho analytics (rẻ hơn nhiều)
```

### 4. Unnecessary PITR và Backup — Backup Không Cần Thiết

```
PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm):
- $0.20/GB-month
- Với 100 GB table: $20/tháng = $240/năm

Câu hỏi: Có thực sự cần PITR không?
- Dev tables: Không cần
- Staging: Có thể không cần
- Production critical: Cần thiết

On-Demand Backup:
- $0.10/GB-month
- Backup 100 GB = $10/tháng

Kiểm tra và xóa backup không cần thiết:
aws dynamodb list-backups --table-name my-table
aws dynamodb delete-backup --backup-arn arn:aws:dynamodb:...
```

### 5. DynamoDB Streams Không Dùng — Streams Lãng Phí

```
DynamoDB Streams (Luồng Dữ Liệu DynamoDB):
- Không tính phí write vào Streams
- Tính phí $0.02/100,000 reads từ Streams
- Nếu Lambda đọc Streams liên tục: có thể tốn kém

Tối ưu:
- Chỉ bật Streams khi có consumer (Lambda, Kinesis Data Streams)
- Tắt Streams khi không cần tính năng event-driven
- Xem xét EventBridge Pipes thay vì custom Lambda polling
```

---

## TTL — Tiết Kiệm Storage Tự Động

### TTL (Time To Live — Thời Gian Sống) Là Gì

```
TTL là tính năng tự động xóa items đã hết hạn:
- Đặt timestamp (Unix epoch) trong một attribute của item
- DynamoDB tự động xóa items khi timestamp < current time
- Xóa trong vòng 48 giờ sau khi hết hạn (không đảm bảo chính xác)
- MIỄN PHÍ — không tốn WCU khi TTL xóa items
```

### Use Cases Cho TTL

```
1. Session data (Dữ liệu phiên):
   sessions table với TTL = login_time + 24h
   → Tự động xóa sessions cũ

2. Cache entries (Mục nhập cache):
   cache table với TTL = created_at + 1h
   → Cache tự động expired

3. Temporary data (Dữ liệu tạm thời):
   one-time-tokens, OTP với TTL = 5 phút
   → Tự động xóa sau khi dùng

4. Audit logs (Nhật ký kiểm toán) với retention policy:
   logs table với TTL = log_time + 90 ngày
   → Tự động tuân thủ data retention policy

5. Rate limiting counters (Bộ đếm giới hạn tốc độ):
   counters với TTL = window_end_time
   → Tự động reset sau mỗi time window
```

### Cài Đặt TTL

```bash
# Bật TTL cho table
aws dynamodb update-time-to-live \
  --table-name my-table \
  --time-to-live-specification \
    "Enabled=true, AttributeName=expiry_time"

# Item phải có attribute "expiry_time" là Unix timestamp
# Ví dụ item:
{
  "user_id": "user123",
  "session_token": "abc123",
  "expiry_time": 1747699200  # Unix timestamp = 2025-05-20 00:00:00 UTC
}
```

### Tiết Kiệm Từ TTL

```
Ví dụ thực tế:
- Table sessions: 10 GB, 100,000 writes/ngày
- Average session lifetime: 24 giờ
- Không có TTL: Table tăng liên tục
  Sau 1 năm: ~3.65 TB storage = $912.50/tháng

- Với TTL: Chỉ giữ 24h × 100,000 = 100,000 active sessions
  Average 1 KB/session = 100 MB storage
  Storage cost: $0.025/tháng (không đáng kể!)

- TTL còn tiết kiệm scan time, read capacity cho queries
```

---

## DAX — Chi Phí vs Lợi Ích

### Chi Phí DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB)

```
DAX cluster pricing:
- Tính theo node type và giờ chạy
- Ví dụ (us-east-1):
  dax.r4.large:   $0.269/node-hour = ~$194/tháng/node
  dax.r4.xlarge:  $0.537/node-hour = ~$387/tháng/node

Minimum cluster: 3 nodes (khuyến nghị cho HA)
3 × dax.r4.large = $194 × 3 = $582/tháng

DynamoDB Accelerator tiết kiệm chi phí khi:
  DAX cache hit rate × DynamoDB read cost savings > DAX cluster cost
```

### Tính ROI của DAX

```
Ví dụ:
- DynamoDB read cost hiện tại: $1,000/tháng
- DAX cluster cost: $582/tháng
- Cache hit rate dự kiến: 90%

DynamoDB reads sau khi có DAX:
$1,000 × (1 - 0.90) = $100/tháng (chỉ cache misses)

Tổng cost với DAX: $582 + $100 = $682/tháng
Tiết kiệm: $1,000 - $682 = $318/tháng (~32%)

Nhưng khi DynamoDB read cost thấp hơn:
- DynamoDB reads: $200/tháng
- DAX: $582/tháng
- DAX KHÔNG cost-effective trong trường hợp này

→ Rule of thumb: DAX cost-effective khi read cost > $700/tháng
  và cache hit rate > 80%
```

### Khi Nào Không Dùng DAX

```
Không cần DAX khi:
- Read cost thấp (< $500/tháng)
- Workload write-heavy (DAX chỉ cache reads, không giúp writes)
- Strongly consistent reads (DAX không cache strongly consistent)
- Rarely repeated reads (cache hit rate thấp)
- Dev/test environments

Thay thế rẻ hơn:
- Application-level caching với ElastiCache Redis
  → Linh hoạt hơn, có thể cache custom data
  → Shared cache giữa nhiều services
  → Thường rẻ hơn DAX cho read-heavy workloads
```

---

## Global Tables — Chi Phí Multi-Region

### Cấu Trúc Chi Phí Global Tables

```
DynamoDB Global Tables v2:
- Mỗi region tính phí riêng
- Thêm replication writes giữa các regions

Replicated Write Request Unit (rWRU):
- On-Demand: $1.875/million rWRU
  (cao hơn WRU thông thường do replication overhead)
- Provisioned: replicated WCU đắt hơn WCU thông thường

Ví dụ 2 regions (us-east-1 + us-west-2):
On-Demand:
- Region 1 reads:  $0.25/M RRU
- Region 1 writes: $1.875/M rWRU (replicated)
- Region 2 reads:  $0.25/M RRU
- Region 2 storage: duplicate cost

→ Global Tables tăng chi phí writes ~50%
→ Tăng chi phí storage 2x (mỗi region lưu đầy đủ data)
```

### Khi Nào Global Tables Justify Chi Phí

```
Không dùng Global Tables khi:
- Chỉ cần disaster recovery (DR) → dùng PITR + on-demand backup rẻ hơn
- Cần read replicas → xem xét DAX hoặc ElastiCache thay thế
- Budget nhạy cảm

Dùng Global Tables khi:
✓ Cần multi-region active-active (Đa Vùng Chủ-Chủ)
✓ Users ở nhiều châu lục, cần low-latency reads tất cả regions
✓ Compliance yêu cầu data residency ở multiple regions
✓ RTO < 1 phút cho regional failover
```

---

## Chiến Lược Thiết Kế Tiết Kiệm Chi Phí

### 1. Chọn Partition Key Phù Hợp

```
Bad partition key → Hot partition (Phân Vùng Nóng) → Throttling → Waste

Ví dụ bad partition key:
- "date" → tất cả writes today → 1 partition quá tải
- "status" → chỉ vài giá trị → phân bổ không đều

Good partition key:
- High cardinality (Độ Phân Tán Cao): user_id, order_id, uuid
- Even distribution (Phân Bổ Đều): không có "hot keys"

Nếu bắt buộc dùng low-cardinality key:
- Write sharding: thêm suffix ngẫu nhiên vào partition key
  "status_hot" → "status_hot#1", "status_hot#2", ..., "status_hot#10"
  Phân tán writes qua 10 partitions
```

### 2. Single-Table Design Tiết Kiệm Hơn Multi-Table

```
Multi-table design (Thiết Kế Nhiều Bảng):
- Mỗi table có minimum provisioned capacity
- Không share capacity giữa tables
- Quản lý phức tạp hơn

Single-table design (Thiết Kế Đơn Bảng):
- 1 table, nhiều entity types
- Share capacity → tiết kiệm tổng capacity cần thiết
- Ít GSI hơn (vì access patterns được thiết kế trước)

Ví dụ:
Multi-table: orders(50WCU) + users(30WCU) + products(20WCU) = 100 WCU
Single-table: 1 table với 60 WCU (peaks không xảy ra cùng lúc)
Tiết kiệm: 40 WCU × $0.468/tháng = $18.72/tháng
```

### 3. Query Optimization — Tối Ưu Truy Vấn

```
Sử dụng Projection Expressions (Biểu Thức Chiếu):
- Chỉ lấy attributes cần thiết
- Giảm data transferred, giảm cost

Bad:
# Lấy toàn bộ item 10 KB chỉ để dùng 1 field
response = table.get_item(Key={"id": "123"})
name = response["Item"]["name"]

Good (tiết kiệm ~90% nếu item lớn):
response = table.get_item(
    Key={"id": "123"},
    ProjectionExpression="name"
)
name = response["Item"]["name"]

Sử dụng Filter Expressions đúng cách:
- Filter Expressions KHÔNG giảm RCU đọc (vẫn đọc tất cả matching items)
- Chỉ giảm data trả về
- Dùng GSI để query thay vì filter để thực sự tiết kiệm RCU
```

### 4. Export to S3 + Athena Cho Analytics

```
Thay vì Scan DynamoDB cho analytics:
- Export DynamoDB table to S3 (Full export hoặc Incremental export)
- Query với Athena: $5/TB data scanned
- Rẻ hơn nhiều so với Scan

Chi phí so sánh (100 GB table, 10 analytics queries/ngày):
DynamoDB Scan:
- 100 GB scan = 25M RCU = $6.25/scan
- 10 scans/ngày × 30 ngày = $1,875/tháng

S3 + Athena:
- Export cost: ~$0.10 (chỉ 1 lần hoặc incremental)
- Athena query: 100 GB × $0.005/GB = $0.50/query
- 10 queries/ngày × 30 ngày = $150/tháng
Tiết kiệm: $1,725/tháng ($20,700/năm!)
```

```bash
# Export DynamoDB table sang S3
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789:table/my-table \
  --s3-bucket my-analytics-bucket \
  --s3-prefix dynamodb-exports/my-table \
  --export-format DYNAMODB_JSON

# Sau đó dùng Athena để query JSON files trong S3
```

### 5. Reserved Capacity Cho DynamoDB Provisioned

```
DynamoDB Reserved Capacity (Năng Lực Đặt Trước):
- Khác với RDS Reserved Instances
- Áp dụng cho Provisioned WCU và RCU
- Tối thiểu mua: 100 WCU hoặc 100 RCU

Pricing (us-east-1):
- 100 WCU × 1yr: $1,012.80/năm (thay vì $1,404/năm) → 28% tiết kiệm
- 100 WCU × 3yr: $1,876.40/3 năm ($625/năm) → 55% tiết kiệm

Phù hợp khi:
- Provisioned table với capacity ổn định
- Production workload ít thay đổi
- Không dùng cho On-Demand mode
```

---

## Checklist Tối Ưu Chi Phí DynamoDB

```
Thiết kế ban đầu:
□ Capacity mode đúng (On-Demand vs Provisioned)?
□ Partition key có high cardinality và even distribution?
□ Single-table design thay vì multi-table?
□ Tối thiểu hóa số GSI — chỉ tạo khi thực sự cần?
□ Item size nhỏ — compress large data, dùng S3 cho binary/blob?
□ TTL cho mọi temporary data?

Hàng tuần:
□ Monitor capacity utilization (utilization < 20% → scale down hoặc đổi On-Demand)
□ Check CloudWatch throttling events → tăng capacity hoặc fix hot partitions
□ Review new scan operations → convert sang Query + GSI

Hàng tháng:
□ Xem Cost Explorer → DynamoDB cost breakdown
□ Check PITR và backup không cần thiết → tắt nếu không cần
□ Review Streams consumers → tắt nếu không có consumer
□ Evaluate DAX ROI nếu đang dùng

Hàng quý:
□ Xem xét chuyển đổi capacity mode nếu traffic pattern thay đổi
□ Mua Reserved Capacity cho stable provisioned tables
□ Audit GSI usage → xóa GSI không dùng
□ Đánh giá export-to-S3 cho analytics use cases
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**File:** 11-cost-optimization/3-dynamodb-cost.md
