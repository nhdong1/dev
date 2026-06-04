# 🔄 Concurrency & Throttling — Quản Lý Đồng Thời Lambda

> Concurrency (Đồng Thời) là số lượng Lambda function instances đang chạy song song tại một thời điểm. Hiểu và quản lý concurrency đúng cách là chìa khóa để Lambda hoạt động ổn định, không làm quá tải downstream services, và tránh mất requests do throttling.

## 📚 Mục Lục

1. [Concurrency Model — Mô Hình Đồng Thời](#concurrency-model--mô-hình-đồng-thời)
2. [Account Concurrency Limits](#account-concurrency-limits)
3. [Reserved Concurrency — Đặt Trước Đồng Thời](#reserved-concurrency--đặt-trước-đồng-thời)
4. [Provisioned Concurrency — Cung Cấp Trước](#provisioned-concurrency--cung-cấp-trước)
5. [Throttling — Giới Hạn Lưu Lượng](#throttling--giới-hạn-lưu-lượng)
6. [Concurrency cho Từng Event Source](#concurrency-cho-từng-event-source)
7. [Monitoring Concurrency](#monitoring-concurrency)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔢 Concurrency Model — Mô Hình Đồng Thời

### Công Thức Tính Concurrency

```
Concurrency = Requests mỗi giây × Thời gian xử lý trung bình (giây)

Ví dụ:
- API nhận 500 req/s
- Mỗi request xử lý 200ms (0.2s)
- Concurrency cần thiết = 500 × 0.2 = 100 instances
```

### Cách Lambda Scale Concurrency

```
Traffic pattern:                Lambda scaling:

Requests/s                      Concurrent instances
    │                               │
300 │          ****               300│          ****
200 │       ***    ***            200│       ***    ***
100 │    ***          ***         100│    ***          ***
 50 │  **                **       50│  **                **
  0 │──────────────────────        0│──────────────────────
    0    5    10   15   20           0    5    10   15   20
              Time (min)                       Time (min)

Lambda tự động tạo instances mới khi traffic tăng
và terminate instances khi traffic giảm
```

### Burst Scaling Limits (Giới Hạn Co Giãn Nhanh)

Lambda không scale vô hạn ngay lập tức — có **burst limit** (giới hạn bùng phát):

| Region                        | Burst Limit (instances mới/phút) |
| ----------------------------- | -------------------------------- |
| us-east-1, us-west-2, eu-west-1 | 3,000 instances đầu tiên         |
| Các region khác               | 500 instances đầu tiên           |
| Sau khi đạt burst limit        | +500 instances/phút               |

```
Ví dụ burst scaling ở us-east-1:
Giây 0:   0 → 3,000 instances (burst ngay lập tức)
Giây 60:  3,000 → 3,500 (+500/phút)
Giây 120: 3,500 → 4,000 (+500/phút)
...đến khi đạt account limit hoặc reserved limit
```

---

## 🏦 Account Concurrency Limits (Giới Hạn Đồng Thời Tài Khoản)

### Default Limits

```
Account-level limit: 1,000 concurrent executions (mặc định)
                          │
                          ├─ Function A: max 1,000 (nếu không có reserved)
                          ├─ Function B: max 1,000
                          └─ Tất cả functions cộng lại: max 1,000
```

### Request Limit Increase (Yêu Cầu Tăng Giới Hạn)

Có thể tăng account limit qua AWS Support:
- Thông thường: 1,000 → 10,000 (dễ approve)
- Lớn hơn: Cần business justification

```bash
# Xem current account concurrency limit
aws lambda get-account-settings
# Output:
# {
#   "AccountLimit": {
#     "TotalCodeSize": 80530636800,
#     "CodeSizeUnzipped": 262144000,
#     "CodeSizeZipped": 52428800,
#     "ConcurrentExecutions": 1000,
#     "UnreservedConcurrentExecutions": 900
#   }
# }
```

---

## 🔒 Reserved Concurrency — Đặt Trước Đồng Thời

### Reserved Concurrency Là Gì?

**Reserved Concurrency** (Đồng Thời Đặt Trước) đặt giới hạn **tối đa** cho một function cụ thể. Nó phục vụ 2 mục đích:

```
Mục đích 1: Đảm bảo function LUÔN có capacity
──────────────────────────────────────────────
Account có 1,000 concurrent
Function A: reserved = 200
  → Function A LUÔN có ít nhất 200 concurrent available
  → Các functions khác không thể "cướp" 200 này

Mục đích 2: GIỚI HẠN tối đa của function
─────────────────────────────────────────
Function B: reserved = 50
  → Function B KHÔNG BAO GIỜ vượt quá 50 concurrent
  → Bảo vệ downstream database khỏi quá tải
```

### Unreserved Concurrency Pool (Pool Đồng Thời Không Đặt Trước)

```
Account limit: 1,000
Reserved cho Function A: 200
Reserved cho Function B: 100
                         ────
Unreserved pool:         700  ← Chia sẻ cho tất cả functions khác

Function C (không reserved): Có thể dùng đến 700 (từ unreserved pool)
Function D (không reserved): Cũng dùng từ cùng pool 700
```

### Cấu Hình Reserved Concurrency

```bash
# Đặt reserved concurrency = 100 cho function
aws lambda put-function-concurrency \
  --function-name payment-processor \
  --reserved-concurrent-executions 100

# Set = 0 để disable function hoàn toàn (block tất cả invocations)
aws lambda put-function-concurrency \
  --function-name non-critical-function \
  --reserved-concurrent-executions 0

# Xóa reserved concurrency (về unreserved pool)
aws lambda delete-function-concurrency \
  --function-name my-function
```

### Ví Dụ Thực Tế: Bảo Vệ Database

```
Vấn đề:
Lambda function đọc/ghi RDS MySQL
RDS chỉ chịu được tối đa 100 connections

Giải pháp:
Reserved Concurrency = 80  (để dành 20 cho margin)
  → Mỗi Lambda instance giữ 1 connection
  → Tối đa 80 connections từ Lambda
  → RDS không bao giờ bị overwhelm
```

```bash
# Cấu hình reserved concurrency cho function kết nối RDS
aws lambda put-function-concurrency \
  --function-name api-database-function \
  --reserved-concurrent-executions 80
```

### Reserved Concurrency = 0 — Disable Function

```bash
# Hữu ích khi:
# - Maintenance mode
# - Debug production issue
# - Tạm thời tắt non-essential function để tiết kiệm cost

aws lambda put-function-concurrency \
  --function-name batch-report-generator \
  --reserved-concurrent-executions 0
# → Tất cả invocations sẽ nhận ThrottlingException
```

---

## ⚡ Provisioned Concurrency — Cung Cấp Trước

### Provisioned Concurrency Là Gì?

**Provisioned Concurrency** (Đồng Thời Được Cung Cấp Trước) pre-warms (làm nóng trước) một số lượng execution environments nhất định. Các environments này:

- Luôn ở trạng thái **initialized** (đã khởi tạo)
- Sẵn sàng xử lý ngay lập tức — **không có cold start**
- Bạn trả tiền ngay cả khi không có traffic

```
Không có Provisioned Concurrency:
Request → [Cold Start: 1-2s] → [Execute: 100ms] → Response
         ↑ Người dùng phải chờ cold start

Với Provisioned Concurrency = 10:
Request → [Execute: 100ms] → Response
         ↑ 10 environments luôn warm, không có cold start
```

### Reserved vs Provisioned — Sự Khác Biệt

| Tính Năng                   | Reserved Concurrency  | Provisioned Concurrency |
| --------------------------- | --------------------- | ----------------------- |
| Mục đích                    | Giới hạn/đảm bảo capacity | Loại bỏ cold start |
| Chi phí thêm                | ❌ Không              | ✅ Có (pay-per-use)     |
| Giảm cold start              | ❌ Không              | ✅ Hoàn toàn            |
| Giới hạn tối đa concurrent   | ✅ Có                 | ❌ Không giới hạn thêm  |
| Khi nào dùng                | Bảo vệ downstream    | Latency-sensitive APIs  |

### Cấu Hình Provisioned Concurrency

```bash
# Bước 1: Phải publish version trước (không thể dùng $LATEST)
aws lambda publish-version --function-name my-api-function
# Version ARN: arn:aws:lambda:...:function:my-api-function:5

# Bước 2: Tạo alias trỏ đến version
aws lambda create-alias \
  --function-name my-api-function \
  --name production \
  --function-version 5

# Bước 3: Cấu hình Provisioned Concurrency cho alias
aws lambda put-provisioned-concurrency-config \
  --function-name my-api-function \
  --qualifier production \
  --provisioned-concurrent-executions 20

# Kiểm tra trạng thái (cần vài phút để warm up)
aws lambda get-provisioned-concurrency-config \
  --function-name my-api-function \
  --qualifier production
# Output:
# {
#   "RequestedProvisionedConcurrentExecutions": 20,
#   "AvailableProvisionedConcurrentExecutions": 20,
#   "Status": "READY"
# }
```

### Auto Scaling Provisioned Concurrency

Để tự động tăng/giảm Provisioned Concurrency theo lịch hoặc load:

```bash
# Đăng ký scalable target (mục tiêu có thể scale)
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-api-function:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 5 \
  --max-capacity 100

# Scale theo lịch (Scheduled Scaling — Co Giãn Theo Lịch)
aws application-autoscaling put-scheduled-action \
  --service-namespace lambda \
  --resource-id function:my-api-function:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --scheduled-action-name business-hours \
  --schedule "cron(0 8 * * ? *)" \
  --scalable-target-action MinCapacity=20,MaxCapacity=100

# Scale theo target metric (Target Tracking — Theo Dõi Mục Tiêu)
aws application-autoscaling put-scaling-policy \
  --service-namespace lambda \
  --resource-id function:my-api-function:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --policy-name pc-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 0.7,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "LambdaProvisionedConcurrencyUtilization"
    }
  }'
# Giải thích: Scale để PC utilization ở mức 70%
# Khi > 70%: Tăng PC
# Khi < 70%: Giảm PC (tiết kiệm chi phí)
```

### Chi Phí Provisioned Concurrency

```
Provisioned Concurrency pricing:
- $0.000004646 per GB-second khi provisioned (dù không có requests)
- $0.000009646 per GB-second khi có requests (thay vì $0.0000166667)

Ví dụ:
Function: 1 GB RAM
Provisioned: 10 environments
Chạy 24/7 trong 30 ngày:
  = 10 × 1 GB × (30 × 24 × 3600)s × $0.000004646
  = 10 × 2,592,000 × $0.000004646
  = $120.38/tháng

So sánh với chi phí thực tế:
Nếu function nhận 100 req/s, 50ms mỗi request:
  Duration cost = 100 × 0.05 × 1 GB × 2,592,000s × $0.000009646 = ???
  → Dùng AWS Pricing Calculator để tính chính xác
```

---

## ⛔ Throttling — Giới Hạn Lưu Lượng

### Khi Nào Lambda Bị Throttle?

Lambda bị **throttle** (hạn chế) khi vượt quá concurrency limit:

```
Account limit: 1,000 concurrent
Current usage: 1,000 concurrent

New request → ThrottlingException (429 Too Many Requests)
```

### Throttle Behavior Theo Invocation Type

```
┌──────────────────────────────────────────────────────────────┐
│ Synchronous (API GW, ALB)                                    │
│   Throttle → 429 error ngay lập tức → Caller phải retry     │
├──────────────────────────────────────────────────────────────┤
│ Asynchronous (S3, SNS, EventBridge)                          │
│   Throttle → Lambda retry tự động (up to 6 giờ)             │
│   Nếu vẫn thất bại → DLQ hoặc Destination OnFailure         │
├──────────────────────────────────────────────────────────────┤
│ Poll-based (SQS, Kinesis, DynamoDB Streams)                  │
│   Throttle → Message ở lại queue/stream                      │
│   Lambda polling lại sau                                     │
└──────────────────────────────────────────────────────────────┘
```

### Xử Lý Throttle Ở Client Side

```python
import boto3
import time
from botocore.config import Config
from botocore.exceptions import ClientError

# Cấu hình retry với exponential backoff (tăng dần thời gian chờ)
config = Config(
    retries={
        'max_attempts': 10,
        'mode': 'adaptive'  # Adaptive retry với backoff
    }
)

lambda_client = boto3.client('lambda', config=config)

def invoke_with_retry(function_name, payload, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = lambda_client.invoke(
                FunctionName=function_name,
                Payload=payload
            )
            return response
        except ClientError as e:
            if e.response['Error']['Code'] == 'TooManyRequestsException':
                if attempt < max_retries - 1:
                    wait_time = (2 ** attempt) + (random.random() * 0.1)
                    print(f"Throttled. Retrying in {wait_time:.2f}s")
                    time.sleep(wait_time)
                else:
                    raise
            else:
                raise
```

### CloudWatch Alarms cho Throttles

```bash
# Alert khi có throttles
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-throttles-alert \
  --alarm-description "Lambda function đang bị throttle" \
  --metric-name Throttles \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=my-critical-function \
  --statistic Sum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts
```

---

## 🎯 Concurrency cho Từng Event Source

### API Gateway + Lambda

```
Traffic spike: 1000 req/s
Lambda concurrency: 500 instances (với 500ms response time)

Nếu không có reserved concurrency:
  → Cạnh tranh với tất cả functions trong account
  → Có thể bị throttle bởi functions khác

Giải pháp:
  Reserved Concurrency = 600 (đủ cho spike + buffer)
  Provisioned Concurrency = 50 (base traffic, không cold start)
```

```python
# API Gateway với usage throttling
# Tránh Lambda bị overwhelm bằng cách throttle ở API GW level
{
    "throttle": {
        "burstLimit": 200,   # Max requests đồng thời
        "rateLimit": 100     # Requests/giây sustained
    }
}
```

### SQS + Lambda

```
SQS Scaling:
- Lambda polls SQS với 5 concurrent pollers (mặc định)
- Mỗi poller lấy batch (tối đa 10 messages)
- Lambda scale số pollers theo queue depth

SQS Scale Out:
Queue: 0 msgs → 1 concurrent Lambda
Queue: 100 msgs → Lambda tăng dần
Queue: 1000+ msgs → Lambda scale đến reserved limit

Công thức:
Max concurrent = min(
    messages_in_queue / batch_size,
    reserved_concurrency,
    account_limit
)
```

```python
# Best practice: Cấu hình SQS Event Source Mapping đúng cách
aws lambda create-event-source-mapping \
  --function-name process-orders \
  --event-source-arn arn:aws:sqs:...:orders-queue \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 30 \
  --scaling-config '{"MaximumConcurrency": 50}'  # Giới hạn concurrent cho SQS cụ thể
```

### DynamoDB Streams + Lambda

```
Mỗi shard trong DynamoDB Stream → 1 concurrent Lambda invocation
(Lambda đọc tuần tự trong mỗi shard để đảm bảo ordering)

10 shards → Tối đa 10 concurrent Lambda instances
(Không thể tăng bằng cách tăng concurrency — phụ thuộc vào số shards)

Tối ưu: Tăng số shards của DynamoDB table nếu cần throughput cao hơn
```

### Kinesis Data Streams + Lambda

```
Mặc định: 1 concurrent Lambda per shard
Enhanced Fan-Out: Có thể có nhiều Lambda consumers per shard (khác nhau)

Parallelization Factor (Hệ Số Song Song Hóa):
  - Default: 1 (1 Lambda per shard)
  - Có thể set 1-10 (tối đa 10 Lambda instances per shard)
  - Hữu ích khi processing time > record arrival rate

aws lambda update-event-source-mapping \
  --uuid <mapping-id> \
  --parallelization-factor 5
```

---

## 📊 Monitoring Concurrency (Giám Sát Đồng Thời)

### CloudWatch Metrics Quan Trọng

| Metric                           | Ý Nghĩa                                            | Cảnh Báo Khi           |
| -------------------------------- | ------------------------------------------------- | ---------------------- |
| `ConcurrentExecutions`           | Số instances đang chạy ngay lúc này               | > 80% của reserved     |
| `Throttles`                      | Số invocations bị throttle                         | > 0 (nếu production)   |
| `ProvisionedConcurrencyUtilization` | % PC đang được sử dụng                         | > 80% (cần tăng PC)    |
| `ProvisionedConcurrencySpilloverInvocations` | Invocations vượt quá PC (cold start) | > threshold |
| `UnreservedConcurrentExecutions` | Concurrent từ unreserved pool                     | Monitor baseline       |

### Dashboard CloudWatch

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "title": "Lambda Concurrency Overview",
        "metrics": [
          ["AWS/Lambda", "ConcurrentExecutions", "FunctionName", "my-api", {"stat": "Maximum"}],
          ["AWS/Lambda", "Throttles", "FunctionName", "my-api", {"stat": "Sum", "color": "#d62728"}],
          ["AWS/Lambda", "ProvisionedConcurrencyUtilization", "FunctionName", "my-api", {"stat": "Average"}]
        ],
        "period": 60,
        "view": "timeSeries"
      }
    }
  ]
}
```

### Lambda Insights — Giám Sát Nâng Cao

```bash
# Enable Lambda Insights (thêm CloudWatch Lambda Insights layer)
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:580247275435:layer:LambdaInsightsExtension:38

# Lambda Insights cung cấp thêm metrics:
# - cpu_total_time
# - memory_utilization  
# - init_duration (cold start time)
# - rx_bytes, tx_bytes (network)
# - fd_use (file descriptors)
```

---

## 🏗️ Thiết Kế Concurrency Strategy (Chiến Lược Đồng Thời)

### Pattern 1: Tier-based Concurrency (Đồng Thời Theo Tầng)

```
Account limit: 1,000 concurrent

Critical services (mission-critical):
  payment-api:      reserved = 200  (luôn có sẵn)
  auth-service:     reserved = 100  (luôn có sẵn)

High-priority services:
  user-api:         reserved = 150
  order-api:        reserved = 150

Background jobs (batch processing):
  report-generator: reserved = 50   (giới hạn để không ảnh hưởng API)
  email-sender:     reserved = 50

Unreserved pool:    300             (cho functions nhỏ, test functions)
```

### Pattern 2: Peak Traffic Planning (Lập Kế Hoạch Lưu Lượng Cao Điểm)

```python
# Tính toán reserved concurrency cần thiết:

peak_rps = 500          # Requests/giây cao điểm
avg_duration_s = 0.2    # Thời gian xử lý trung bình (200ms)
safety_buffer = 1.3     # 30% buffer

required_concurrency = peak_rps * avg_duration_s * safety_buffer
# = 500 * 0.2 * 1.3 = 130 concurrent

# Set reserved = 150 (làm tròn lên, thêm buffer)
# Set provisioned = 30 (cho base load không cold start)
```

### Pattern 3: Database Connection Management (Quản Lý Kết Nối Database)

```
Vấn đề: 1,000 concurrent Lambda × 1 connection = 1,000 DB connections
Giải pháp:

Option A: Reserved Concurrency
  Lambda: reserved = 80
  RDS: max connections = 100 (có 20 dự phòng cho admin, monitoring)

Option B: RDS Proxy (Tốt hơn)
  Lambda → RDS Proxy (connection pooling) → RDS
  RDS Proxy pool: 100 connections
  Lambda concurrent: 1,000+ (RDS Proxy tự quản lý pool)
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Reserved Concurrency vs Provisioned Concurrency — khi nào dùng cái nào?**
> Reserved Concurrency: Dùng để (1) đảm bảo function luôn có capacity bất kể các functions khác, (2) bảo vệ downstream services (database, external APIs) bằng cách giới hạn tối đa concurrent. Không có thêm chi phí, không giảm cold start. Provisioned Concurrency: Dùng khi cần loại bỏ hoàn toàn cold start cho latency-sensitive APIs. Tốn thêm chi phí (trả tiền dù không có request).

**Q: Làm thế nào Lambda xử lý traffic spike?**
> Lambda tự động tạo execution environments mới khi có requests. Burst limit ban đầu là 3,000 instances (us-east-1) sau đó tăng +500/phút. Sau burst limit, vượt quá concurrency account limit sẽ bị throttle (429). Giải pháp: (1) Request limit increase, (2) Sử dụng SQS để buffer requests, (3) Cấu hình API Gateway throttling để bảo vệ Lambda.

**Q: Tại sao Reserved Concurrency = 0 lại useful?**
> Vô hiệu hóa hoàn toàn function mà không cần xóa — tất cả invocations trả về ThrottlingException. Hữu ích cho: maintenance mode, debug production issues, tạm thời tắt non-essential functions khi tài nguyên account cạn, blue/green deployment khi cần chuyển traffic.

**Q: SQS Lambda trigger tự scale thế nào?**
> Lambda service tự polling SQS. Khi queue có messages, Lambda tăng dần số concurrent pollers. Scale out nhanh (thêm 60 instances mỗi phút nếu queue depth tăng). Scale in chậm (giảm khi queue trống). Có thể đặt `MaximumConcurrency` trong Event Source Mapping để giới hạn concurrent cho SQS cụ thể mà không ảnh hưởng đến reserved concurrency của function.

**Q: Sự khác biệt giữa throttle với async vs sync invocation?**
> Sync (API Gateway): Throttle → ngay lập tức trả 429 về caller — caller phải tự xử lý retry. Async (S3, SNS): Throttle → Lambda service tự retry với backoff lên đến 6 giờ → sau đó gửi vào DLQ nếu vẫn thất bại. Poll-based (SQS): Throttle → message ở lại queue → Lambda polling lại sau → không bị mất.

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [5-performance-best-practices.md](./5-performance-best-practices.md) — Performance & Best Practices
