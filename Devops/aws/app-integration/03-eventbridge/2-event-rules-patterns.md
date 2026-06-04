# Event Rules & Patterns — Lọc và Định Tuyến Sự Kiện

> **Rule** (Quy Tắc) xác định sự kiện nào cần xử lý và gửi đến đâu. **Event Pattern** (Mẫu Sự Kiện) là bộ lọc JSON quyết định sự kiện nào khớp với rule. Đây là trái tim của EventBridge.

---

## 📚 Mục Lục

1. [Event Rule Là Gì?](#1-event-rule-là-gì)
2. [Event Pattern — Cú Pháp Lọc](#2-event-pattern--cú-pháp-lọc)
3. [Input Transformation — Biến Đổi Dữ Liệu](#3-input-transformation--biến-đổi-dữ-liệu)
4. [Target Configuration](#4-target-configuration--cấu-hình-đích)
5. [Dead Letter Queue Cho EventBridge](#5-dead-letter-queue-cho-eventbridge)
6. [Retry Policy — Chính Sách Thử Lại](#6-retry-policy--chính-sách-thử-lại)
7. [Scheduled Rules — Quy Tắc Lịch Trình](#7-scheduled-rules--quy-tắc-lịch-trình)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Event Rule Là Gì?

**Rule** là cấu hình xác định:

1. **Trên bus nào** — `EventBusName`
2. **Sự kiện nào được chọn** — `EventPattern` hoặc `ScheduleExpression`
3. **Gửi đến target nào** — tối đa **5 targets** mỗi rule
4. **Biến đổi dữ liệu như thế nào** — `InputTransformer` (tùy chọn)

```
Event arrives ──▶ EventBridge Bus ──▶ Rule matches? ──▶ Yes ──▶ Send to Target(s)
                                          │
                                          No ──▶ Event discarded (sự kiện bị loại bỏ)
```

### Giới Hạn Quan Trọng

| Thông Số | Giới Hạn |
|---|---|
| Rules per event bus | 300 (có thể tăng theo yêu cầu) |
| Targets per rule | 5 |
| Event size tối đa | 256 KB |
| Entries per PutEvents call | 10 |

---

## 2. Event Pattern — Cú Pháp Lọc

**Event Pattern** là JSON object dùng để so khớp (match) với sự kiện. Chỉ sự kiện **khớp hoàn toàn** mới được gửi đến target.

### Nguyên Tắc So Khớp Cơ Bản

```
Rule: Chỉ định trường nào cần khớp và giá trị nào
Sự kiện: Phải có tất cả các trường được chỉ định trong rule
Các trường không có trong rule: Được bỏ qua (ignored)
```

### 2.1 Exact Match — Khớp Chính Xác

```json
// Event Pattern
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["Order Placed"]
}

// Khớp với sự kiện:
{
  "source": "com.mycompany.orders",
  "detail-type": "Order Placed",
  "detail": { "orderId": "ORD-001" }  // trường này không cần trong pattern
}

// KHÔNG khớp với:
{
  "source": "com.mycompany.payments",  // source khác
  "detail-type": "Order Placed"
}
```

### 2.2 Prefix Match — Khớp Tiền Tố

```json
{
  "source": [{ "prefix": "com.mycompany" }]
}
// Khớp với: com.mycompany.orders, com.mycompany.payments, com.mycompany.inventory
```

### 2.3 Anything-But — Ngoại Trừ

```json
{
  "detail": {
    "status": [{ "anything-but": ["CANCELLED", "REFUNDED"] }]
  }
}
// Khớp với bất kỳ status nào TRỪ CANCELLED và REFUNDED
```

### 2.4 Numeric Range — Phạm Vi Số

```json
{
  "detail": {
    "amount": [{ "numeric": [">", 100, "<=", 10000] }]
  }
}
// Khớp khi amount > 100 VÀ amount <= 10000
```

**Các toán tử số:** `=`, `<`, `<=`, `>`, `>=`

### 2.5 Exists — Kiểm Tra Tồn Tại

```json
{
  "detail": {
    "discountCode": [{ "exists": true }]
  }
}
// Chỉ khớp khi sự kiện CÓ trường discountCode

{
  "detail": {
    "errorMessage": [{ "exists": false }]
  }
}
// Chỉ khớp khi sự kiện KHÔNG CÓ trường errorMessage
```

### 2.6 Wildcard — Ký Tự Đại Diện

```json
{
  "detail": {
    "email": [{ "wildcard": "*@mycompany.com" }]
  }
}
// Khớp với: admin@mycompany.com, support@mycompany.com
// KHÔNG khớp: user@gmail.com
```

### 2.7 Suffix Match — Khớp Hậu Tố

```json
{
  "detail": {
    "fileName": [{ "suffix": ".jpg" }]
  }
}
// Khớp với: photo.jpg, avatar.jpg
// KHÔNG khớp: document.pdf
```

### 2.8 IP Address — Địa Chỉ IP

```json
{
  "detail": {
    "sourceIp": [{ "cidr": "10.0.0.0/8" }]
  }
}
// Khớp với bất kỳ IP nào trong subnet 10.x.x.x
```

### 2.9 Kết Hợp Nhiều Điều Kiện

```json
// Logic AND (VÀ) — ngầm định khi nhiều trường trong cùng object
{
  "source": ["com.mycompany.orders"],
  "detail": {
    "status": ["PENDING"],
    "amount": [{ "numeric": [">", 1000000] }]  // >1 triệu VND
  }
}
// Khớp khi: source = orders VÀ status = PENDING VÀ amount > 1000000

// Logic OR (HOẶC) — nhiều giá trị trong cùng array
{
  "detail": {
    "status": ["FAILED", "ERROR", "TIMEOUT"]
  }
}
// Khớp khi status là FAILED HOẶC ERROR HOẶC TIMEOUT
```

### 2.10 Nested Object — Object Lồng Nhau

```json
// Sự kiện
{
  "detail": {
    "user": {
      "type": "premium",
      "region": "VN"
    },
    "order": {
      "items": [
        { "category": "electronics" }
      ]
    }
  }
}

// Pattern lọc sự kiện của premium user ở Việt Nam
{
  "detail": {
    "user": {
      "type": ["premium"],
      "region": ["VN"]
    }
  }
}
```

### 2.11 Ví Dụ Thực Tế — E-Commerce

```json
// Rule 1: Đơn hàng lớn (>5 triệu VND) — cần xử lý thủ công
{
  "source": ["com.mycompany.orders"],
  "detail-type": ["Order Placed"],
  "detail": {
    "amount": [{ "numeric": [">", 5000000] }],
    "status": [{ "anything-but": "CANCELLED" }]
  }
}

// Rule 2: Lỗi thanh toán — cần retry
{
  "source": ["com.mycompany.payments"],
  "detail-type": ["Payment Failed"],
  "detail": {
    "errorCode": [{ "prefix": "GATEWAY_" }],
    "retryCount": [{ "numeric": ["<", 3] }]
  }
}

// Rule 3: Sự kiện từ VIP customers
{
  "source": ["com.mycompany.orders"],
  "detail": {
    "customer": {
      "tier": ["gold", "platinum", "diamond"]
    }
  }
}
```

---

## 3. Input Transformation — Biến Đổi Dữ Liệu

**Input Transformer** (Bộ Biến Đổi Đầu Vào) cho phép biến đổi sự kiện trước khi gửi đến target — không cần Lambda chỉ để chuyển đổi format.

### 3.1 Cú Pháp Cơ Bản

```json
{
  "InputPathsMap": {
    "orderId": "$.detail.orderId",
    "amount": "$.detail.amount",
    "customerEmail": "$.detail.customer.email",
    "eventTime": "$.time"
  },
  "InputTemplate": "{\"message\": \"Đơn hàng <orderId> trị giá <amount> VND đã được đặt\", \"notify\": \"<customerEmail>\", \"timestamp\": \"<eventTime>\"}"
}
```

### 3.2 Ví Dụ — Gửi Notification Đến Slack

```json
// Sự kiện gốc
{
  "source": "com.mycompany.orders",
  "detail-type": "Order Failed",
  "detail": {
    "orderId": "ORD-001",
    "errorMessage": "Payment gateway timeout",
    "customerId": "CUST-123"
  }
}

// InputPathsMap — trích xuất giá trị
{
  "InputPathsMap": {
    "order": "$.detail.orderId",
    "error": "$.detail.errorMessage",
    "time": "$.time"
  }
}

// InputTemplate — tạo payload mới cho Slack
{
  "InputTemplate": "{\"text\": \":x: Order <order> FAILED at <time>\", \"attachments\": [{\"text\": \"Error: <error>\", \"color\": \"danger\"}]}"
}
```

### 3.3 Ghi Đè Toàn Bộ — Input Override

```json
// Thay toàn bộ payload bằng giá trị cố định
{
  "Input": "{\"action\": \"process\", \"source\": \"eventbridge\"}"
}
```

### 3.4 Chuyển Đổi Sang SQS Message

```json
// InputPathsMap
{
  "orderId": "$.detail.orderId",
  "amount": "$.detail.amount",
  "region": "$.region"
}

// InputTemplate cho SQS
{
  "InputTemplate": "{\"type\": \"ORDER_PROCESSING\", \"orderId\": \"<orderId>\", \"amount\": <amount>, \"processedRegion\": \"<region>\"}"
}
```

---

## 4. Target Configuration — Cấu Hình Đích

### 4.1 Lambda Target

```json
{
  "Id": "LambdaOrderProcessor",
  "Arn": "arn:aws:lambda:ap-southeast-1:123456789:function:order-processor",
  "InputTransformer": {
    "InputPathsMap": {
      "orderId": "$.detail.orderId"
    },
    "InputTemplate": "{\"orderId\": \"<orderId>\"}"
  }
}
```

### 4.2 SQS Target

```json
{
  "Id": "SQSOrderQueue",
  "Arn": "arn:aws:sqs:ap-southeast-1:123456789:orders-queue",
  "SqsParameters": {
    "MessageGroupId": "$.detail.customerId"  // Dùng cho FIFO queue
  }
}
```

### 4.3 Step Functions Target

```json
{
  "Id": "StepFunctionsOrderWorkflow",
  "Arn": "arn:aws:states:ap-southeast-1:123456789:stateMachine:OrderProcessingWorkflow",
  "RoleArn": "arn:aws:iam::123456789:role/EventBridgeStepFunctionsRole",
  "Input": "$.detail"  // Chỉ truyền phần detail của sự kiện
}
```

### 4.4 API Gateway Target

```json
{
  "Id": "APIGatewayWebhook",
  "Arn": "arn:aws:execute-api:ap-southeast-1:123456789:api-id/stage/POST/webhook",
  "HttpParameters": {
    "PathParameterValues": ["$.detail.orderId"],
    "HeaderParameters": {
      "X-Event-Type": "$.detail-type"
    },
    "QueryStringParameters": {
      "source": "$.source"
    }
  }
}
```

### 4.5 EventBridge Bus Target — Cross-Account

```json
{
  "Id": "CrossAccountEventBus",
  "Arn": "arn:aws:events:ap-southeast-1:CONSUMER_ACCOUNT:event-bus/consumer-bus",
  "RoleArn": "arn:aws:iam::PRODUCER_ACCOUNT:role/EventBridgeCrossAccountRole"
}
```

---

## 5. Dead Letter Queue Cho EventBridge

Khi target không xử lý được sự kiện sau retry, EventBridge gửi sự kiện vào **DLQ** (Hàng Đợi Thư Chết) để phân tích sau:

```json
{
  "Id": "LambdaTarget",
  "Arn": "arn:aws:lambda:...:function:order-processor",
  "DeadLetterConfig": {
    "Arn": "arn:aws:sqs:ap-southeast-1:123456789:eventbridge-dlq"
  }
}
```

**Cấu trúc message trong DLQ:**

```json
{
  "ErrorCode": "InvocationError",
  "ErrorMessage": "Lambda function returned an error",
  "EventBusName": "orders-bus",
  "Event": "{\"source\":\"com.mycompany.orders\",...}",  // Sự kiện gốc
  "RuleArn": "arn:aws:events:...:rule/orders-bus/process-order"
}
```

---

## 6. Retry Policy — Chính Sách Thử Lại

```json
{
  "Id": "LambdaTarget",
  "Arn": "arn:aws:lambda:...",
  "RetryPolicy": {
    "MaximumRetryAttempts": 3,           // Số lần thử lại tối đa (0-185)
    "MaximumEventAgeInSeconds": 3600     // Bỏ sự kiện sau 1 giờ nếu vẫn lỗi
  },
  "DeadLetterConfig": {
    "Arn": "arn:aws:sqs:...:eventbridge-dlq"
  }
}
```

**Cơ chế retry:**

```
Lần 1: Thất bại
  ↓ Đợi (exponential backoff — chờ tăng dần)
Lần 2: Thất bại
  ↓ Đợi
Lần 3: Thất bại
  ↓ MaximumRetryAttempts đạt ngưỡng
DLQ: Sự kiện được gửi vào Dead Letter Queue
```

---

## 7. Scheduled Rules — Quy Tắc Lịch Trình

### 7.1 Rate Expression (Biểu Thức Tốc Độ)

```bash
aws events put-rule \
  --name "every-5-minutes" \
  --schedule-expression "rate(5 minutes)" \
  --state ENABLED

# Các ví dụ:
# rate(1 minute)   — mỗi phút
# rate(1 hour)     — mỗi giờ
# rate(1 day)      — mỗi ngày
# rate(7 days)     — mỗi tuần
```

### 7.2 Cron Expression (Biểu Thức Cron)

```
Cú pháp: cron(phút giờ ngày-tháng tháng ngày-tuần năm)

cron(0 12 * * ? *)          — Mỗi ngày 12:00 UTC
cron(15 10 ? * MON-FRI *)   — 10:15 UTC, thứ 2 đến thứ 6
cron(0 18 L * ? *)          — 18:00 UTC ngày cuối tháng
cron(0/5 8-17 ? * MON-FRI *)— Mỗi 5 phút, 8h-17h các ngày thường
```

**Lưu ý:** EventBridge cron dùng **UTC**. Để chạy lúc 7:00 sáng ICT (UTC+7), cần dùng `cron(0 0 * * ? *)`.

### 7.3 Ví Dụ Thực Tế

```bash
# Report hàng ngày lúc 8 giờ sáng ICT (1:00 UTC)
aws events put-rule \
  --name "daily-report-8am-ict" \
  --schedule-expression "cron(0 1 * * ? *)" \
  --state ENABLED

# Cleanup job mỗi Chủ Nhật 2 giờ sáng ICT (19:00 UTC Thứ 7)
aws events put-rule \
  --name "weekly-cleanup" \
  --schedule-expression "cron(0 19 ? * SAT *)" \
  --state ENABLED

# Health check mỗi 30 giây — dùng rate
aws events put-rule \
  --name "health-check" \
  --schedule-expression "rate(1 minute)" \
  --state ENABLED
```

### 7.4 EventBridge Scheduler (Bộ Lên Lịch EventBridge)

**EventBridge Scheduler** (ra mắt 2022) là dịch vụ tách biệt, mạnh hơn scheduled rules:

```python
import boto3

scheduler = boto3.client('scheduler')

# Tạo one-time schedule (lịch chạy một lần)
scheduler.create_schedule(
    Name='send-reminder-email',
    ScheduleExpression='at(2026-05-20T07:00:00)',  # Chạy đúng thời điểm
    ScheduleExpressionTimezone='Asia/Ho_Chi_Minh',
    FlexibleTimeWindow={'Mode': 'OFF'},
    Target={
        'Arn': 'arn:aws:lambda:...:function:send-reminder',
        'RoleArn': 'arn:aws:iam::...:role/scheduler-role',
        'Input': json.dumps({'type': 'REMINDER', 'userId': 'user-123'})
    }
)

# Recurring schedule với timezone đúng
scheduler.create_schedule(
    Name='daily-cleanup',
    ScheduleExpression='cron(0 2 * * ? *)',  # 2 giờ sáng theo timezone chỉ định
    ScheduleExpressionTimezone='Asia/Ho_Chi_Minh',  # ICT timezone
    FlexibleTimeWindow={
        'Mode': 'FLEXIBLE',
        'MaximumWindowInMinutes': 15  # Chạy trong cửa sổ 15 phút
    },
    Target={
        'Arn': 'arn:aws:sqs:...:cleanup-queue',
        'RoleArn': 'arn:aws:iam::...:role/scheduler-role'
    }
)
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Tối đa bao nhiêu targets trong một rule? Cách vượt giới hạn?**

> Mỗi rule có tối đa 5 targets. Nếu cần nhiều hơn: (1) dùng SNS topic làm target rồi fan-out đến nhiều subscriber, (2) dùng SQS + Lambda consumer để xử lý và gọi nhiều service, (3) tạo nhiều rules cùng event pattern (mỗi rule có 5 targets).

**Q: Sự kiện không khớp với bất kỳ rule nào thì sao?**

> Sự kiện bị discard (loại bỏ) ngay lập tức và miễn phí — EventBridge không lưu sự kiện không khớp. Đây là lý do cần thiết kế event pattern cẩn thận và dùng Archive để có thể replay sau.

**Q: Input Transformer dùng khi nào? Có cần Lambda không?**

> Input Transformer giải quyết bài toán chuyển đổi format đơn giản (rename fields, format string) mà không cần Lambda. Ví dụ: EventBridge nhận sự kiện S3, cần gửi message Slack với format khác — Input Transformer xử lý được mà không tốn Lambda invocation. Nếu logic phức tạp (conditional logic, external API calls) thì mới cần Lambda.

**Q: Scheduled rule vs EventBridge Scheduler — dùng cái nào?**

> EventBridge Scheduler mới và mạnh hơn: hỗ trợ timezone đúng chuẩn (scheduled rule chỉ dùng UTC), one-time schedules, flexible time window (giảm cold start Lambda), và scale đến hàng triệu schedules. Dùng EventBridge Scheduler cho production; scheduled rule cho use case đơn giản.

---

**Liên Kết:** [1-event-bus.md](1-event-bus.md) | [3-schema-registry.md](3-schema-registry.md)
