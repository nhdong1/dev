# 2 — CloudWatch Logs — Quản Lý Nhật Ký Tập Trung

> CloudWatch Logs là dịch vụ lưu trữ, tìm kiếm và phân tích logs tập trung — từ EC2, Lambda, ECS đến bất kỳ ứng dụng nào có thể gửi log lên AWS

## 📚 Mục Lục

1. [Kiến Trúc CloudWatch Logs](#kiến-trúc-cloudwatch-logs)
2. [Log Groups và Log Streams](#log-groups-và-log-streams)
3. [Thu Thập Logs Từ Các Dịch Vụ](#thu-thập-logs-từ-các-dịch-vụ)
4. [CloudWatch Logs Insights](#cloudwatch-logs-insights)
5. [Metric Filters — Trích Xuất Metrics Từ Logs](#metric-filters)
6. [Subscription Filters — Streaming Logs](#subscription-filters)
7. [Retention Policies — Chính Sách Lưu Giữ](#retention-policies)
8. [Structured Logging — Ghi Log Có Cấu Trúc](#structured-logging)
9. [Chi Phí Và Tối Ưu](#chi-phí-và-tối-ưu)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc CloudWatch Logs

```
┌────────────────────────────────────────────────────────────────────┐
│                    CLOUDWATCH LOGS ARCHITECTURE                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  LOG SOURCES (Nguồn Log)                                          │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────────┐  │
│  │  EC2   │ │Lambda  │ │  ECS   │ │  EKS   │ │  Custom App    │  │
│  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───────┬────────┘  │
│      │          │          │          │               │            │
│      ▼          ▼          ▼          ▼               ▼            │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              CloudWatch Logs Service                        │   │
│  │                                                            │   │
│  │  Log Group: /aws/lambda/my-function                        │   │
│  │    └── Log Stream: 2026/05/15/[$LATEST]abc123             │   │
│  │    └── Log Stream: 2026/05/15/[$LATEST]def456             │   │
│  │                                                            │   │
│  │  Log Group: /aws/ecs/my-cluster/my-service                 │   │
│  │    └── Log Stream: my-container/i-0abc123                 │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│         ┌────────────────────┼─────────────────────┐              │
│         ▼                    ▼                     ▼              │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
│  │ Logs Insights│  │ Metric Filters   │  │  Subscriptions   │    │
│  │  (Queries)   │  │ (→ CloudWatch    │  │  (→ Kinesis,     │    │
│  │              │  │    Metrics)      │  │   Lambda, S3)    │    │
│  └──────────────┘  └──────────────────┘  └──────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Log Groups và Log Streams

### Log Group — Nhóm Log

Log Group là container chứa nhiều Log Streams cùng loại. Thường ánh xạ 1-1 với một ứng dụng hoặc dịch vụ:

```
Quy ước đặt tên (naming convention) thường dùng:
  /aws/lambda/{function-name}           ← Lambda tự động tạo
  /aws/ecs/{cluster}/{service}          ← ECS
  /aws/eks/{cluster-name}/cluster       ← EKS control plane
  /ec2/{environment}/{app-name}         ← EC2 custom
  /app/{environment}/{service-name}     ← Custom application
```

### Log Stream — Luồng Log

Log Stream là chuỗi log events từ **một nguồn duy nhất**. Trong cùng Log Group:

```
Log Group: /aws/lambda/order-service
  ├── Log Stream: 2026/05/15/[$LATEST]abc1234567890abcdef
  │     (Logs từ một Lambda execution environment cụ thể)
  ├── Log Stream: 2026/05/15/[$LATEST]def1234567890abcdef
  │     (Logs từ một execution environment khác)
  └── Log Stream: 2026/05/14/[$LATEST]ghi1234567890abcdef
        (Logs ngày hôm trước)

Log Group: /aws/ecs/production/payment-service
  ├── Log Stream: payment-container/ecs-task-id-abc123
  └── Log Stream: payment-container/ecs-task-id-def456
```

### Tạo Log Group Và Cấu Hình

```bash
# Tạo Log Group
aws logs create-log-group \
  --log-group-name /app/production/order-service

# Set retention (lưu giữ) 30 ngày
aws logs put-retention-policy \
  --log-group-name /app/production/order-service \
  --retention-in-days 30

# Tạo Log Stream trong Group
aws logs create-log-stream \
  --log-group-name /app/production/order-service \
  --log-stream-name "server-1"
```

---

## Thu Thập Logs Từ Các Dịch Vụ

### Lambda — Tự Động

Lambda **tự động** gửi logs vào CloudWatch Logs — không cần cấu hình thêm:

```python
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info("Processing order", extra={
        "order_id": event["order_id"],
        "user_id": event["user_id"]
    })
    # Mọi print() và logger.* đều xuất hiện trong CloudWatch Logs
    print("This also goes to CloudWatch Logs")
```

Log Group được tạo tự động: `/aws/lambda/{function-name}`

### ECS — Cấu Hình AWSlogs Driver

Trong Task Definition, cấu hình `logConfiguration`:

```json
{
  "containerDefinitions": [
    {
      "name": "order-service",
      "image": "my-account.dkr.ecr.ap-southeast-1.amazonaws.com/order-service:latest",
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/production/order-service",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
```

### EC2 — CloudWatch Agent

Cấu hình trong `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/application/app.log",
            "log_group_name": "/ec2/production/order-service",
            "log_stream_name": "{hostname}",
            "timestamp_format": "%Y-%m-%dT%H:%M:%S%z",
            "multi_line_start_pattern": "^\\d{4}-\\d{2}-\\d{2}"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/ec2/production/nginx-access",
            "log_stream_name": "{hostname}"
          }
        ]
      }
    }
  }
}
```

### EKS — Fluent Bit (Sidecar/DaemonSet)

```yaml
# DaemonSet Fluent Bit gửi logs từ mọi node về CloudWatch
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    spec:
      containers:
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:latest
        env:
        - name: AWS_REGION
          value: ap-southeast-1
        - name: CLUSTER_NAME
          value: my-eks-cluster
        - name: LOG_GROUP_NAME
          value: /aws/eks/my-eks-cluster/application
```

---

## CloudWatch Logs Insights

CloudWatch Logs Insights là công cụ query phân tích logs với ngôn ngữ riêng, tối ưu cho log data.

### Cú Pháp Cơ Bản

```sql
-- Cấu trúc query cơ bản:
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Giải thích:
-- fields   → chọn fields muốn hiển thị
-- filter   → lọc theo điều kiện
-- sort     → sắp xếp
-- limit    → giới hạn số kết quả
```

### Query Thực Tế Hay Dùng

```sql
-- 1. Tìm tất cả lỗi trong 1 giờ qua
fields @timestamp, @message
| filter @message like /ERROR/ or @message like /Exception/
| sort @timestamp desc
| limit 50

-- 2. Thống kê số lỗi theo loại (parsing JSON logs)
fields @timestamp, level, errorType, @message
| filter level = "ERROR"
| stats count(*) as errorCount by errorType
| sort errorCount desc

-- 3. Tính P50/P95/P99 latency của Lambda
filter @type = "REPORT"
| parse @message "Duration: * ms" as duration
| stats
    avg(duration) as p50_approx,
    pct(duration, 95) as p95,
    pct(duration, 99) as p99,
    max(duration) as maxDuration
  by bin(5m)

-- 4. Tìm Lambda cold starts
filter @message like /Init Duration/
| parse @message "Init Duration: * ms" as initDuration
| stats count() as coldStarts, avg(initDuration) as avgInitTime by bin(1h)

-- 5. Tìm requests chậm nhất
fields @timestamp, requestId, @duration
| filter @type = "REPORT"
| sort @duration desc
| limit 20

-- 6. Thống kê HTTP status codes từ ALB logs
fields @timestamp, status, request
| stats count(*) as requestCount by status
| sort requestCount desc
```

### Saved Queries — Lưu Query Hay Dùng

```bash
# Lưu query để dùng lại
aws logs put-query-definition \
  --name "Lambda Error Analysis" \
  --query-string "filter @type='REPORT' | stats avg(@duration), max(@duration) by bin(5m)"
```

---

## Metric Filters

Metric Filters (Bộ Lọc Metric) trích xuất patterns từ log entries và chuyển thành CloudWatch Metrics — cho phép tạo alarms dựa trên log content.

### Ví Dụ: Đếm Lỗi Thanh Toán

```bash
# Tạo Metric Filter đếm "PaymentFailed" trong logs
aws logs put-metric-filter \
  --log-group-name /app/production/order-service \
  --filter-name PaymentFailures \
  --filter-pattern '[timestamp, requestId, level="ERROR", message="PaymentFailed*"]' \
  --metric-transformations \
    metricName=PaymentFailureCount,\
    metricNamespace=MyApp/Business,\
    metricValue=1,\
    unit=Count,\
    defaultValue=0
```

### Filter Patterns — Mẫu Lọc

```
# Tìm từ khóa đơn giản:
"ERROR"                         → Dòng chứa "ERROR"
"PaymentFailed"                 → Dòng chứa "PaymentFailed"
?"ERROR" ?"WARN"                → Dòng chứa ERROR hoặc WARN
-"DEBUG"                        → Dòng KHÔNG chứa "DEBUG"

# JSON structured logs:
{ $.level = "ERROR" }           → JSON có level = "ERROR"
{ $.httpStatusCode >= 500 }     → HTTP 5xx errors
{ $.latency > 1000 }            → Requests chậm > 1 giây

# Kết hợp:
{ $.level = "ERROR" && $.service = "payment" }
```

### Ví Dụ Metric Filters Phổ Biến

```bash
# 1. Đếm HTTP 5xx errors
filter-pattern: '{ $.status >= 500 }'
metric: HTTP5xxErrors, Count=1

# 2. Đếm authentication failures
filter-pattern: '"Authentication failed"'
metric: AuthFailures, Count=1

# 3. Trích xuất latency từ log (nếu log format cố định)
filter-pattern: '[date, time, level, service, message="latency=*"]'
metric: AppLatency, Count=$message (extract số từ latency=...)

# 4. Đếm cold starts Lambda
filter-pattern: '"Init Duration"'
metric: LambdaColdStarts, Count=1
```

---

## Subscription Filters

Subscription Filters (Bộ Lọc Đăng Ký) stream log events thời gian thực đến các đích khác:

```
CloudWatch Logs → Kinesis Data Streams    (stream analytics)
CloudWatch Logs → Kinesis Data Firehose   (deliver đến S3/OpenSearch)
CloudWatch Logs → Lambda                  (real-time processing)
CloudWatch Logs → OpenSearch Service      (full-text search)
```

### Ví Dụ: Stream Logs Đến Lambda Để Xử Lý

```bash
# Tạo Subscription Filter gửi ERROR logs đến Lambda
aws logs put-subscription-filter \
  --log-group-name /app/production/order-service \
  --filter-name ErrorToSlack \
  --filter-pattern '"ERROR"' \
  --destination-arn arn:aws:lambda:ap-southeast-1:123456789012:function:send-slack-alert
```

### Cross-Account Log Aggregation (Tổng Hợp Log Đa Tài Khoản)

```
Account A (Production)  ──→ Kinesis Data Stream
Account B (Staging)     ──→ (cùng stream)
Account C (Dev)         ──→ (cùng stream)
                              │
                              ▼
                     Central Logging Account
                     (CloudWatch Logs / S3 / OpenSearch)
```

---

## Retention Policies — Chính Sách Lưu Giữ

Mặc định, CloudWatch Logs giữ logs **mãi mãi** (bạn trả tiền mãi). Luôn set retention:

### Các Khoảng Retention Hợp Lệ (Ngày)

```
1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365,
400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653
```

### Chiến Lược Retention Theo Môi Trường

```bash
# Production critical logs — giữ 1 năm (compliance)
aws logs put-retention-policy \
  --log-group-name /app/production/order-service \
  --retention-in-days 365

# Staging/Dev — giữ 14 ngày thôi
aws logs put-retention-policy \
  --log-group-name /app/staging/order-service \
  --retention-in-days 14

# Lambda execution logs — 30 ngày thường đủ
aws logs put-retention-policy \
  --log-group-name /aws/lambda/my-function \
  --retention-in-days 30
```

### Bulk Set Retention (Terraform)

```hcl
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/app/${var.environment}/${var.service_name}"
  retention_in_days = var.environment == "prod" ? 365 : 14

  tags = {
    Environment = var.environment
    Service     = var.service_name
  }
}
```

---

## Structured Logging — Ghi Log Có Cấu Trúc

Structured logging (ghi log có cấu trúc — JSON format) giúp Logs Insights query hiệu quả hơn nhiều.

### So Sánh Unstructured vs Structured

```
❌ UNSTRUCTURED (khó query):
2026-05-15T14:30:01Z ERROR PaymentService Order 12345 failed: timeout after 5000ms

✅ STRUCTURED (dễ query, filter, aggregate):
{
  "timestamp": "2026-05-15T14:30:01Z",
  "level": "ERROR",
  "service": "PaymentService",
  "orderId": "12345",
  "error": "timeout",
  "durationMs": 5000,
  "userId": "user-789",
  "correlationId": "req-abc-def-123"
}
```

### Python Structured Logging

```python
import json
import logging
from datetime import datetime

class StructuredLogger:
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)

    def _log(self, level: str, message: str, **kwargs):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": level,
            "service": self.service_name,
            "message": message,
            **kwargs
        }
        print(json.dumps(log_entry))  # Lambda gửi stdout → CloudWatch

    def info(self, message: str, **kwargs):
        self._log("INFO", message, **kwargs)

    def error(self, message: str, **kwargs):
        self._log("ERROR", message, **kwargs)

# Sử dụng trong Lambda
logger = StructuredLogger("order-service")

def lambda_handler(event, context):
    logger.info(
        "Processing order",
        orderId=event["order_id"],
        userId=event["user_id"],
        correlationId=context.aws_request_id
    )
```

### Correlation IDs — ID Tương Quan

Luôn truyền `correlationId` xuyên suốt request để link logs từ nhiều services:

```python
def lambda_handler(event, context):
    correlation_id = (
        event.get("headers", {}).get("X-Correlation-ID")
        or context.aws_request_id
    )

    logger.info(
        "Request received",
        correlationId=correlation_id,
        requestId=context.aws_request_id
    )

    # Truyền correlationId khi gọi service khác
    headers = {"X-Correlation-ID": correlation_id}
    response = requests.post("http://payment-service/pay", headers=headers)
```

---

## Chi Phí Và Tối Ưu

### Bảng Chi Phí CloudWatch Logs (ap-southeast-1)

| Hoạt Động                     | Chi Phí                    |
| ----------------------------- | -------------------------- |
| **Ingestion** (ghi vào)       | $0.76/GB                   |
| **Storage** (lưu trữ)         | $0.033/GB/tháng            |
| **Logs Insights queries**     | $0.0076/GB data scanned    |
| **Delivered to S3** (archive) | $0.023/GB                  |

### Chiến Lược Tiết Kiệm

```
1. SET RETENTION POLICIES:
   Dev/Staging → 7-14 ngày (không cần giữ lâu)
   Production  → 30-90 ngày (trừ khi có compliance requirement)

2. FILTER TRƯỚC KHI GỬI:
   Đừng log DEBUG trong production
   Dùng log level thích hợp: ERROR > WARN > INFO > DEBUG

3. ARCHIVE CŨ VÀO S3:
   Dùng S3 Export + Glacier cho logs cần giữ lâu nhưng ít query
   CloudWatch Logs → S3 → Glacier: giảm cost 90%+

4. SAMPLE HIGH-VOLUME LOGS:
   Với debug logs có volume lớn, chỉ log 10% (sampling)
   Luôn log ERROR (100%), INFO (100%), DEBUG (10%)

5. OPTIMIZE LOG SIZE:
   Đừng log full request/response body trừ khi cần thiết
   Dùng log levels đúng cách
```

### Export Logs Cũ Sang S3

```bash
# Export logs sang S3 để lưu trữ rẻ hơn
aws logs create-export-task \
  --log-group-name /app/production/order-service \
  --from 1700000000000 \
  --to 1700086400000 \
  --destination my-logs-archive-bucket \
  --destination-prefix "cloudwatch-logs/order-service/2026-05/"
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Log Group và Log Stream?**

> Log Group là container logic nhóm các logs liên quan (thường 1 ứng dụng = 1 Log Group). Log Stream là chuỗi logs từ một nguồn cụ thể trong thời gian (1 Lambda execution environment = 1 Log Stream, 1 ECS task instance = 1 Log Stream). Log Group là nơi apply retention và Metric Filters; Log Streams là nơi logs thực sự được ghi.

**Q: Metric Filter hoạt động như thế nào? Khi nào dùng?**

> Metric Filter scan log events mới theo pattern (text hoặc JSON field). Mỗi match → publish 1 data point lên CloudWatch Metric. Dùng khi muốn alert dựa trên log content: vd đếm số lần "PaymentFailed" xuất hiện trong logs → alarm nếu > 10 lần/phút. Hiệu quả hơn so với query liên tục bằng Logs Insights.

**Q: Làm sao debug Lambda performance bằng CloudWatch Logs Insights?**

> Lambda tự động log REPORT line chứa Duration, Billed Duration, Memory. Query: `filter @type = "REPORT" | stats pct(@duration, 99) as p99 by bin(5m)` → thấy latency trend. Tìm cold starts: `filter @message like "Init Duration"`. Link với X-Ray trace ID để xem detail.

**Q: Tại sao phải dùng structured logging (JSON)?**

> Text logs khó query: phải dùng regex dễ sai, khó extract giá trị số để tính aggregate. JSON logs: có thể query bằng `{ $.orderId = "12345" }`, dễ dàng `stats avg($.latencyMs)`, filter theo bất kỳ field nào. CloudWatch Logs Insights hiểu JSON natively — đây là best practice hiện đại.

**Q: Làm sao aggregrate logs từ nhiều AWS accounts?**

> Dùng Subscription Filter trong từng account gửi logs đến Kinesis Data Stream trong Central Logging Account. Hoặc dùng CloudWatch cross-account observability (tính năng mới hơn) cho phép query logs cross-account trực tiếp. Cần Resource Policy trên Kinesis cho phép accounts khác put records.

---

**← Trước:** [1-cloudwatch-metrics.md](./1-cloudwatch-metrics.md) | **Tiếp theo →** [3-cloudwatch-alarms.md](./3-cloudwatch-alarms.md)

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
