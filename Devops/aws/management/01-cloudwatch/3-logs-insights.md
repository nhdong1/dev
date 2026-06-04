# CloudWatch Logs & Logs Insights

> **CloudWatch Logs** thu thập, lưu trữ và tìm kiếm log từ mọi nguồn trên AWS và on-premises. **Logs Insights** (Phân Tích Nhật Ký) cung cấp ngôn ngữ truy vấn mạnh mẽ để phân tích log hàng tỷ dòng trong vài giây.

---

## 📚 Mục Lục

1. [Kiến Trúc CloudWatch Logs](#kiến-trúc-cloudwatch-logs)
2. [Log Groups & Log Streams](#log-groups--log-streams)
3. [Retention & Encryption](#retention--encryption)
4. [Nguồn Log Tự Động](#nguồn-log-tự-động)
5. [Metric Filters — Trích Xuất Metric Từ Log](#metric-filters--trích-xuất-metric-từ-log)
6. [Subscription Filters — Streaming Log Real-time](#subscription-filters--streaming-log-real-time)
7. [CloudWatch Logs Insights](#cloudwatch-logs-insights)
8. [Live Tail — Xem Log Thời Gian Thực](#live-tail--xem-log-thời-gian-thực)
9. [Cross-Account Log Sharing](#cross-account-log-sharing)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc CloudWatch Logs

```
┌──────────────────────────────────────────────────────────────┐
│                      DATA SOURCES                            │
│  Lambda │ EC2+Agent │ ECS │ API Gateway │ CloudTrail │ VPC  │
└───────────────────────────┬──────────────────────────────────┘
                            │  PutLogEvents API
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                    LOG GROUP (Nhóm Log)                      │
│  /aws/lambda/my-function                                     │
│  /aws/ecs/my-cluster/my-service                              │
│  /aws/ec2/my-application                                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │               LOG STREAM (Luồng Log)                    │ │
│  │  2026/05/17/[$LATEST]abc123   ← Lambda instance         │ │
│  │  2026/05/17/[$LATEST]def456   ← Lambda instance khác    │ │
│  │  i-0abc123/application.log    ← EC2 instance log        │ │
│  └─────────────────────────────────────────────────────────┘ │
└────────┬──────────────────────┬───────────────────────────────┘
         │                      │
    ┌────▼─────┐          ┌─────▼──────────┐
    │  Metric  │          │  Subscription  │
    │  Filter  │          │  Filter        │
    └────┬─────┘          └─────┬──────────┘
         │                      │
    ┌────▼─────┐    ┌───────────▼────────────┐
    │  Custom  │    │  Kinesis │ Lambda │ S3 │
    │  Metric  │    │  (Streaming realtime)  │
    └──────────┘    └────────────────────────┘
         │
    ┌────▼─────────────┐
    │   Logs Insights   │
    │   (Query Engine)  │
    └───────────────────┘
```

---

## Log Groups & Log Streams

### Log Group (Nhóm Nhật Ký)

**Log Group** là container tổ chức logs theo ứng dụng, service, hoặc môi trường. Mọi cài đặt retention, encryption, metric filter áp dụng ở cấp độ Log Group.

**Naming Conventions (Quy Ước Đặt Tên):**

```
# AWS Services — tự động tạo:
/aws/lambda/{function-name}
/aws/ecs/containerinsights/{cluster}/{service}
/aws/apigateway/{api-id}/
/aws/rds/instance/{db-id}/error
/aws/cloudtrail (nếu bật gửi vào CW Logs)

# Application logs — tự đặt tên:
/app/{environment}/{service}
  /app/prod/order-service
  /app/staging/payment-service

# System logs từ CloudWatch Agent:
/ec2/{instance-id}/var/log/application.log
/ec2/os/messages
```

### Log Stream (Luồng Nhật Ký)

**Log Stream** là chuỗi log events từ **một nguồn cụ thể** trong một Log Group:
- Mỗi Lambda execution environment → một Log Stream riêng
- Mỗi EC2 instance → một Log Stream (hoặc nhiều, theo cấu hình Agent)
- ECS container → một Log Stream theo task

```
Log Group: /aws/lambda/order-processor
├── Log Stream: 2026/05/17/[$LATEST]abc123ef   ← Lambda container 1
├── Log Stream: 2026/05/17/[$LATEST]def456ab   ← Lambda container 2
└── Log Stream: 2026/05/17/[$LATEST]ghi789cd   ← Lambda container 3
```

> **Thực tế:** Khi Lambda scale up (mở rộng), mỗi concurrent execution (thực thi đồng thời) có Log Stream riêng. Để tìm log của một request, dùng Logs Insights thay vì browse từng stream.

---

## Retention & Encryption

### Log Retention (Thời Gian Lưu Nhật Ký)

Mặc định: **Logs lưu vĩnh viễn** (và tốn tiền). Luôn đặt retention policy!

| Thời Gian Lưu    | Use Case                                        |
| ---------------- | ----------------------------------------------- |
| 1 ngày           | Debug log môi trường dev                        |
| 7 ngày           | Application log môi trường staging              |
| 30 ngày          | Production log thông thường                     |
| 90 ngày          | Security/audit log (PCI-DSS minimum)            |
| 365 ngày         | Compliance-sensitive logs (HIPAA, SOC2)         |
| 3 năm            | Financial industry regulatory requirement       |
| Never expire     | Tránh dùng — tốn kém, backup ra S3 thay thế    |

```bash
# Đặt retention policy qua CLI
aws logs put-retention-policy \
  --log-group-name /app/prod/order-service \
  --retention-in-days 90
```

**Lưu ý:** Nếu cần lưu lâu hơn 10 năm → Export sang S3 + S3 Glacier. CloudWatch tối đa hỗ trợ lưu đến "never expire" nhưng S3 rẻ hơn nhiều cho cold storage (lưu trữ lạnh).

### Encryption (Mã Hóa)

- **Mặc định:** CloudWatch Logs mã hóa log at rest (lúc lưu trữ) bằng AWS-managed key — miễn phí, không cần cấu hình
- **Customer-managed KMS key** (Khóa KMS Do Khách Quản Lý): Tùy chọn nếu cần kiểm soát hoàn toàn key rotation và audit

```bash
# Gắn KMS key vào Log Group
aws logs associate-kms-key \
  --log-group-name /app/prod/payment-service \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc-123
```

---

## Nguồn Log Tự Động

### Lambda

Lambda **tự động** gửi log vào `/aws/lambda/{function-name}` — không cần cấu hình gì. Tuy nhiên:
- Phải đảm bảo Lambda execution role có permission `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
- `print()` / `console.log()` → tự động thành log events
- Exception stacktrace → tự động captured

### ECS / EKS

Cần cấu hình **log driver** trong task definition:

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-app",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "ecs"
    }
  }
}
```

### API Gateway

Cần bật Access Logging và Execution Logging trong Stage settings:
- **Access Logs**: Request/response log (latency, status, caller IP)
- **Execution Logs**: Debug log từng bước xử lý request

### VPC Flow Logs (Nhật Ký Luồng VPC)

Ghi lại traffic (lưu lượng mạng) vào/ra VPC, subnet, ENI — hữu ích để debug kết nối và phân tích bảo mật:

```
Ví dụ VPC Flow Log record:
2 123456789012 eni-0abc123 10.0.1.5 10.0.2.10 443 54321 6 25 52428 ACCEPT OK
  │            │           │         │          │   │     │  │   │      │
  version    account-id  eni-id  src-ip   dst-ip  dst-port src-port  bytes  action
```

### CloudTrail → CloudWatch Logs

Có thể cấu hình CloudTrail gửi API event logs vào CloudWatch Logs Group để:
- Tạo metric filter trên CloudTrail events
- Alert khi có hành động nhạy cảm (xóa S3, thay đổi IAM)

---

## Metric Filters — Trích Xuất Metric Từ Log

**Metric Filter** (Bộ Lọc Metric) theo dõi Log Group, tìm pattern trong log, và tạo CloudWatch Metric từ kết quả khớp.

### Ví Dụ Thực Tế

**Đếm lỗi từ application log:**
```
Filter Pattern: [ERROR]
Metric: ErrorCount (tăng 1 mỗi khi tìm thấy [ERROR])
Alarm: ErrorCount > 10/phút → SNS notification
```

**Đếm HTTP 500 từ Access Log:**
```
Log format: [time] [status] [method] [path] [latency]
Filter Pattern: [time, status=5*, method, path, latency]
Metric: HTTP500Count
```

**Trích xuất giá trị số từ log:**
```
Log line: "OrderProcessingTime: 245ms OrderId: ORD-123"

Filter Pattern: [text="OrderProcessingTime:", value, unit="ms", ...]
Metric Value: $value (245)
Metric: OrderProcessingTimeMs
```

### Cú Pháp Filter Pattern

```
# Tìm chuỗi đơn giản:
ERROR
"OutOfMemoryError"

# Kết hợp AND:
ERROR Exception

# OR (dùng ?):
?ERROR ?WARN

# NOT:
ERROR -"expected error"

# JSON pattern:
{ $.level = "ERROR" }
{ $.httpStatus = 5* }
{ $.responseTime > 1000 }

# Space-delimited log pattern:
[ip, identity, user, timestamp, request, status_code=4*, size]
```

---

## Subscription Filters — Streaming Log Real-time

**Subscription Filter** (Bộ Lọc Đăng Ký) stream (truyền) log events theo thời gian thực đến destination (đích) khác để xử lý thêm.

### Destinations (Đích)

```
CloudWatch Logs
       │
       │ (Subscription Filter)
       ├──► Lambda Function        ← Real-time processing, custom alerting
       ├──► Kinesis Data Streams   ← High-throughput streaming
       ├──► Kinesis Firehose       ← Delivery đến S3, OpenSearch, Redshift
       └──► (Cross-account)       ← Centralized log account
```

### Use Cases Thực Tế

**Centralized Logging Architecture (Kiến Trúc Log Tập Trung):**
```
Account A (App) → CW Logs → Subscription Filter → Kinesis Firehose
Account B (App) → CW Logs → Subscription Filter → Kinesis Firehose
Account C (App) → CW Logs → Subscription Filter → Kinesis Firehose
                                                          │
                                                          ▼
                                                   Log Archive Account
                                                        S3 Bucket
                                                   (Long-term storage)
```

**Real-time Security Monitoring:**
```
/aws/cloudtrail → Subscription Filter (pattern: DeleteBucket, DetachPolicy)
                        → Lambda → Security team Slack alert
```

**Log to OpenSearch (Elasticsearch):**
```
Application Logs → Subscription Filter → Kinesis Firehose → Amazon OpenSearch
                                                                    │
                                                               Kibana Dashboard
                                                               Full-text search
```

---

## CloudWatch Logs Insights

**CloudWatch Logs Insights** là query engine (công cụ truy vấn) cho phép phân tích log với ngôn ngữ truy vấn riêng, xử lý hàng tỷ log events trong vài giây.

### Cú Pháp Cơ Bản

```
# Cấu trúc truy vấn:
fields @timestamp, @message       # Chọn fields
| filter @message like /ERROR/    # Lọc
| stats count() by bin(5m)        # Tổng hợp
| sort @timestamp desc             # Sắp xếp
| limit 20                         # Giới hạn kết quả
```

### Queries Thực Tế Quan Trọng

**1. Tìm lỗi gần nhất:**
```
fields @timestamp, @message
| filter @message like /Exception|ERROR|FATAL/
| sort @timestamp desc
| limit 50
```

**2. Đếm errors theo loại:**
```
fields @message
| filter @message like /ERROR/
| parse @message "ERROR * - *" as errorType, errorMsg
| stats count(*) as errorCount by errorType
| sort errorCount desc
```

**3. Phân tích latency Lambda:**
```
fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
| filter @type = "REPORT"
| stats
    avg(@duration) as avgDuration,
    max(@duration) as maxDuration,
    percentile(@duration, 99) as p99Duration,
    avg(@maxMemoryUsed) as avgMemUsed
  by bin(1h)
| sort @timestamp desc
```

**4. Tìm Lambda cold starts:**
```
filter @type = "REPORT"
| filter @initDuration > 0
| stats count() as coldStartCount, avg(@initDuration) as avgInitMs by bin(1h)
```

**5. Top 10 API endpoints chậm nhất:**
```
# Với API Gateway Access Log format
fields @timestamp, httpMethod, path, responseLatency, status
| filter status >= 400 OR responseLatency > 1000
| stats
    count(*) as requestCount,
    avg(responseLatency) as avgLatency,
    percentile(responseLatency, 99) as p99Latency
  by httpMethod, path
| sort p99Latency desc
| limit 10
```

**6. VPC Flow Logs — Tìm traffic bị REJECT:**
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) as rejectCount by srcAddr, dstAddr, dstPort
| sort rejectCount desc
| limit 20
```

**7. CloudTrail — Ai đã xóa tài nguyên:**
```
fields @timestamp, userIdentity.arn, eventName, requestParameters
| filter eventName like /Delete|Terminate|Remove/
| filter userIdentity.type != "AWSService"
| sort @timestamp desc
| limit 50
```

**8. Phân tích tỷ lệ lỗi theo thời gian:**
```
fields @timestamp, status
| stats
    sum(status >= 500) as errorCount,
    count(*) as totalCount,
    sum(status >= 500) / count(*) * 100 as errorRate
  by bin(5m)
| sort @timestamp desc
```

### Từ Khóa Logs Insights Quan Trọng

| Lệnh       | Chức Năng                                     | Ví Dụ                                    |
| ---------- | --------------------------------------------- | ---------------------------------------- |
| `fields`   | Chọn fields hiển thị                          | `fields @timestamp, @message`            |
| `filter`   | Lọc log events                                | `filter @message like /ERROR/`           |
| `parse`    | Trích xuất giá trị từ text                    | `parse @message "* took * ms" as op, ms`|
| `stats`    | Tổng hợp và tính toán                         | `stats count() by bin(5m)`               |
| `sort`     | Sắp xếp kết quả                               | `sort @timestamp desc`                   |
| `limit`    | Giới hạn số dòng kết quả                      | `limit 100`                              |
| `dedup`    | Loại bỏ trùng lặp                             | `dedup requestId`                        |
| `display`  | Định dạng output                              | `display @timestamp, errorType`          |
| `unmask`   | Hiển thị log đã mask (nếu có permission)      | `unmask`                                 |

### Hàm Tổng Hợp (Aggregation Functions)

```
count(*)                    → Đếm log events
count_distinct(field)       → Đếm giá trị duy nhất
sum(field)                  → Tổng
avg(field)                  → Trung bình
min(field) / max(field)     → Min/Max
percentile(field, 99)       → Phân vị P99
stddev(field)               → Độ lệch chuẩn
```

### Built-in Fields (Trường Tự Động)

| Field                | Nội Dung                                      |
| -------------------- | --------------------------------------------- |
| `@timestamp`         | Thời điểm log event                           |
| `@message`           | Toàn bộ nội dung log event                   |
| `@logStream`         | Tên Log Stream                                |
| `@log`               | Tên Log Group                                 |
| `@ingestionTime`     | Thời điểm CloudWatch nhận log (khác @timestamp)|
| `@type`              | Loại event Lambda (START, END, REPORT)        |
| `@duration`          | Thời gian chạy Lambda (ms)                   |
| `@billedDuration`    | Thời gian tính phí Lambda (ms)               |
| `@initDuration`      | Cold start duration (ms)                     |
| `@maxMemoryUsed`     | Peak memory Lambda (MB)                      |

---

## Live Tail — Xem Log Thời Gian Thực

**CloudWatch Logs Live Tail** (ra mắt 2023) cho phép stream log theo thời gian thực trong console — giống `tail -f` nhưng trên cloud.

```
# Tính năng:
- Chọn Log Groups và filter pattern
- Highlight pattern khớp (màu sắc)
- Không cần refresh tay
- Tối đa 5 Log Groups đồng thời
- Không lưu vào CloudWatch → không tốn tiền scan
```

**Giá:** $0.01/phút sử dụng Live Tail (billing per session, không phải per log volume).

---

## Cross-Account Log Sharing

### Kiến Trúc Centralized Logging (Logging Tập Trung)

Trong môi trường multi-account, tập hợp log về một Log Archive Account để:
- Phân tích tập trung với Logs Insights
- Tuân thủ compliance (không ai trong app account xóa được log)
- Giảm chi phí (query một nơi thay vì nhiều account)

```
┌─────────────────────────────────────────────────────────┐
│              MULTI-ACCOUNT LOG ARCHITECTURE             │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Account A   │  │  Account B   │  │  Account C   │  │
│  │  App Team 1  │  │  App Team 2  │  │  App Team 3  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │ Subscription    │ Subscription    │           │
│         │ Filter          │ Filter          │           │
│         └────────────────┬┴─────────────────┘           │
│                          │                              │
│                          ▼                              │
│              ┌───────────────────────┐                  │
│              │   Log Archive Account │                  │
│              │   Kinesis Firehose    │                  │
│              │         ↓             │                  │
│              │      S3 Bucket        │                  │
│              │   (long-term storage) │                  │
│              │         ↓             │                  │
│              │    CloudWatch Logs    │                  │
│              │   (cross-account      │                  │
│              │    Logs Insights)     │                  │
│              └───────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### Cấu Hình Cross-Account Subscription

```bash
# 1. Tạo destination trong Log Archive Account
aws logs put-destination \
  --destination-name "CentralizedLogs" \
  --target-arn arn:aws:kinesis:us-east-1:ARCHIVE-ACCOUNT:stream/LogStream \
  --role-arn arn:aws:iam::ARCHIVE-ACCOUNT:role/CloudWatchLogsRole

# 2. Đặt destination policy (cho phép App Accounts gửi log)
aws logs put-destination-policy \
  --destination-name "CentralizedLogs" \
  --access-policy '{"Version":"2012-10-17","Statement":[{
    "Effect":"Allow",
    "Principal":{"AWS":["arn:aws:iam::APP-ACCOUNT-A:root"]},
    "Action":"logs:PutSubscriptionFilter",
    "Resource":"arn:aws:logs:us-east-1:ARCHIVE-ACCOUNT:destination:CentralizedLogs"
  }]}'

# 3. Trong App Account — tạo Subscription Filter
aws logs put-subscription-filter \
  --log-group-name /app/prod/order-service \
  --filter-name "ForwardToArchive" \
  --filter-pattern "" \
  --destination-arn arn:aws:logs:us-east-1:ARCHIVE-ACCOUNT:destination:CentralizedLogs
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa Metric Filter và Logs Insights?

| Tiêu Chí       | Metric Filter                     | Logs Insights                         |
| -------------- | --------------------------------- | ------------------------------------- |
| **Mục đích**   | Tạo metric từ log (real-time)     | Query log để phân tích                |
| **Thời gian**  | Real-time, liên tục               | Ad-hoc query lịch sử                  |
| **Output**     | CloudWatch Metric → Alarm         | Bảng kết quả, visualize              |
| **Dùng khi**   | "Alert khi có ERROR trong log"    | "Tại sao có lỗi lúc 14:30?"          |
| **Chi phí**    | Metric fee sau khi tạo            | $0.005/GB data scanned                |

### Q2: Log Group vs Log Stream khác nhau thế nào?

**Log Group** = nhóm log theo ứng dụng/service (có retention policy, encryption, metric filter áp dụng chung).

**Log Stream** = chuỗi log từ một nguồn cụ thể trong group (ví dụ: một Lambda container, một EC2 instance).

Analogy (Ví Von): Log Group = Database, Log Stream = Table trong database đó.

### Q3: Tại sao Logs Insights tính phí theo data scanned chứ không phải kết quả?

Logs Insights phải **quét toàn bộ data** trong time range bạn chỉ định để tìm log khớp — ngay cả khi chỉ có 10 kết quả. Chi phí dựa trên lượng data quét, không phải kết quả trả về. Vì vậy:
- Thu hẹp time range → giảm chi phí đáng kể
- Chỉ query Log Groups cần thiết, không query tất cả
- Dùng `filter` sớm trong query để CloudWatch optimize

### Q4: Làm thế nào tìm log của một request cụ thể khi có nhiều Lambda instances?

Dùng **request ID** hoặc **correlation ID** (mã tương quan) được ghi vào log:

```python
# Lambda handler — ghi request ID vào mọi log
import logging
logger = logging.getLogger()

def handler(event, context):
    request_id = context.aws_request_id
    logger.info(f"[{request_id}] Processing order {event['orderId']}")
    # ...
    logger.info(f"[{request_id}] Order processed successfully")
```

```
# Logs Insights — tìm tất cả log của request đó:
fields @timestamp, @message
| filter @message like /req-abc-123/
| sort @timestamp asc
```

### Q5: CloudWatch Logs vs S3 — khi nào dùng cái nào để lưu log?

| Tiêu Chí              | CloudWatch Logs                    | S3                                  |
| --------------------- | ---------------------------------- | ----------------------------------- |
| **Truy vấn nhanh**    | ✅ Logs Insights trong giây        | ❌ Cần Athena (phút)               |
| **Realtime**          | ✅ Metric Filter, Live Tail        | ❌ Không                           |
| **Chi phí lưu trữ**   | Đắt hơn ($0.03/GB/tháng)          | Rẻ hơn ($0.023/GB S3 Standard)     |
| **Long-term (>1 năm)**| Tốn kém                           | ✅ Glacier ($0.004/GB)             |
| **Compliance archival**| Không lý tưởng                   | ✅ S3 Object Lock (immutable)       |
| **Best practice**     | 30–90 ngày hot logs               | Archive sau 90 ngày, giữ lâu dài   |

**Pattern chuẩn:** Log → CloudWatch Logs (30–90 ngày) → Subscription Filter → Kinesis Firehose → S3 (long-term archive)

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [2-alarms-composite.md](./2-alarms-composite.md) | [4-dashboards-widgets.md](./4-dashboards-widgets.md)
