# Amazon EventBridge — Tổng Quan và Lộ Trình Học

> **EventBridge** là serverless event bus (xe buýt sự kiện không máy chủ) của AWS, cho phép xây dựng kiến trúc event-driven (hướng sự kiện) bằng cách kết nối các ứng dụng với nhau qua luồng sự kiện theo thời gian thực.

---

## 📚 Mục Lục Module

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| `README.md` | Tổng quan, kiến trúc, khi nào dùng | ⭐⭐⭐ |
| `1-event-bus.md` | Default, Custom, Partner Event Bus | ⭐⭐⭐ |
| `2-event-rules-patterns.md` | Rule & Pattern — Lọc và Định Tuyến Sự Kiện | ⭐⭐⭐ |
| `3-schema-registry.md` | Schema Registry — Quản Lý Lược Đồ Sự Kiện | ⭐⭐ |
| `4-archive-replay.md` | Archive & Replay — Lưu Trữ và Phát Lại | ⭐⭐ |
| `5-eventbridge-pipes.md` | EventBridge Pipes — Đường Ống Sự Kiện | ⭐⭐ |

---

## 🎯 EventBridge Là Gì?

**Amazon EventBridge** (trước đây là CloudWatch Events) là dịch vụ event bus serverless cho phép:

1. **Nhận sự kiện** từ nhiều nguồn: AWS services, ứng dụng tùy chỉnh, ứng dụng SaaS bên thứ ba
2. **Lọc sự kiện** theo rule (quy tắc) với event pattern (mẫu sự kiện) linh hoạt
3. **Định tuyến sự kiện** đến target (đích) phù hợp: Lambda, SQS, SNS, Step Functions, v.v.
4. **Quản lý schema** (lược đồ) sự kiện qua Schema Registry (kho lưu lược đồ)
5. **Lưu trữ và phát lại** sự kiện để debug và khôi phục

```
                    ┌─────────────────────────────────────────────┐
                    │            Amazon EventBridge               │
                    │                                             │
 ┌──────────┐       │  ┌──────────┐   ┌──────────┐   ┌────────┐  │   ┌──────────┐
 │ AWS      │──────▶│  │  Event   │──▶│   Rule   │──▶│Target  │──│──▶│ Lambda   │
 │ Services │       │  │  Bus     │   │ (filter) │   │        │  │   │ SQS/SNS  │
 └──────────┘       │  └──────────┘   └──────────┘   └────────┘  │   │ StepFn   │
 ┌──────────┐       │       ▲                                     │   └──────────┘
 │ Custom   │───────│───────┘                                     │
 │ Apps     │       │                                             │
 └──────────┘       │  ┌─────────────────────────────────────┐   │
 ┌──────────┐       │  │  Schema Registry | Archive | Pipes  │   │
 │ SaaS     │───────│──│  (Kho Lược Đồ   | Lưu Trữ | Ống)   │   │
 │ Partners │       │  └─────────────────────────────────────┘   │
 └──────────┘       └─────────────────────────────────────────────┘
```

---

## 🏗️ Các Khái Niệm Cốt Lõi

### Event (Sự Kiện)

Một **event** là JSON object mô tả thay đổi trạng thái xảy ra trong hệ thống:

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "source": "com.mycompany.orders",
  "account": "123456789012",
  "time": "2026-05-18T10:00:00Z",
  "region": "ap-southeast-1",
  "detail-type": "Order Placed",
  "detail": {
    "orderId": "ORD-001",
    "customerId": "CUST-123",
    "amount": 99.99,
    "status": "PENDING"
  }
}
```

**Các trường bắt buộc:**
- `source` — nguồn phát sự kiện (thường dùng reverse domain: `com.mycompany.service`)
- `detail-type` — loại sự kiện, mô tả ngắn gọn điều gì xảy ra
- `detail` — nội dung chi tiết của sự kiện (tối đa 256KB)

### Event Bus (Xe Buýt Sự Kiện)

**Event bus** là kênh nhận và phân phối sự kiện. Có 3 loại:

| Loại | Mô Tả | Khi Dùng |
|---|---|---|
| **Default event bus** | Bus mặc định nhận sự kiện từ AWS services | Sự kiện từ EC2, S3, RDS... |
| **Custom event bus** | Bus tùy chỉnh cho ứng dụng của bạn | Microservices nội bộ |
| **Partner event bus** | Bus nhận sự kiện từ SaaS partners | Shopify, Zendesk, Datadog... |

### Rule (Quy Tắc)

**Rule** xác định:
1. **Event Pattern** (Mẫu Sự Kiện) — sự kiện nào được chọn lọc
2. **Target** (Đích) — sự kiện được gửi đến đâu (tối đa 5 targets mỗi rule)

### Target (Đích)

Nơi sự kiện được gửi đến sau khi khớp với rule. Hỗ trợ hơn 20 loại target:

- AWS Lambda Function
- Amazon SQS Queue
- Amazon SNS Topic
- AWS Step Functions State Machine (Máy Trạng Thái)
- Amazon Kinesis Data Streams / Firehose
- Amazon ECS Task
- AWS API Gateway / HTTP endpoint
- AWS Batch Job
- Amazon CodeBuild / CodePipeline
- Chính EventBridge bus khác (cross-account)

---

## 📡 Nguồn Sự Kiện (Event Sources)

### 1. AWS Services (Dịch Vụ AWS)

EventBridge tự động nhận sự kiện từ hơn 100 dịch vụ AWS:

```
EC2 instance state change ──────▶ Default Bus ──▶ Rule ──▶ Lambda (ghi log)
S3 object created ───────────────▶ Default Bus ──▶ Rule ──▶ SQS (xử lý)
RDS snapshot created ────────────▶ Default Bus ──▶ Rule ──▶ SNS (notify)
CodePipeline stage change ───────▶ Default Bus ──▶ Rule ──▶ Slack webhook
```

### 2. Custom Applications (Ứng Dụng Tùy Chỉnh)

Ứng dụng của bạn gửi sự kiện qua API:

```python
import boto3

client = boto3.client('events')

response = client.put_events(
    Entries=[
        {
            'Source': 'com.mycompany.orders',
            'DetailType': 'Order Placed',
            'Detail': '{"orderId": "ORD-001", "amount": 99.99}',
            'EventBusName': 'my-order-bus'
        }
    ]
)
```

### 3. SaaS Partner Event Sources

Nhận sự kiện trực tiếp từ SaaS partners mà không cần polling (truy vấn định kỳ):

```
Shopify ──────▶ Partner Event Bus ──▶ Rule ──▶ Lambda (sync inventory)
Zendesk ──────▶ Partner Event Bus ──▶ Rule ──▶ SQS (process tickets)
Datadog ──────▶ Partner Event Bus ──▶ Rule ──▶ SNS (alert team)
```

---

## 🆚 EventBridge vs SNS vs SQS

Đây là câu hỏi phỏng vấn thường gặp — phải nắm rõ:

| Tiêu Chí | EventBridge | SNS | SQS |
|---|---|---|---|
| **Mô Hình** | Event routing (định tuyến sự kiện) | Pub/Sub (phát/đăng ký) | Queue (hàng đợi) |
| **Lọc Sự Kiện** | Mạnh — filter theo bất kỳ trường JSON | Cơ bản — message attributes | Không hỗ trợ |
| **Schema Management** | Có — Schema Registry | Không | Không |
| **Archive & Replay** | Có | Không | Không |
| **SaaS Integration** | Có — Partner Event Sources | Không | Không |
| **Cross-account** | Có | Có (hạn chế) | Có (hạn chế) |
| **Latency** | ~0.5 giây | Mili-giây | Mili-giây |
| **Throughput** | 10,000 events/giây/region | Cao | Cao |
| **Pricing** | $1/triệu events | $0.50/triệu | $0.40/triệu |
| **Best For** | Event-driven architecture phức tạp | Notifications, fan-out | Task queue, decoupling |

### Nguyên Tắc Chọn Lựa

```
Cần lọc sự kiện theo logic phức tạp? ──────────▶ EventBridge
Cần nhận sự kiện từ SaaS partners? ────────────▶ EventBridge
Cần Schema Registry? ───────────────────────────▶ EventBridge
Cần replay sự kiện cũ? ─────────────────────────▶ EventBridge

Cần thông báo đến nhiều subscriber ngay lập tức? ▶ SNS
Cần push delivery đến email/SMS/mobile? ──────────▶ SNS
Cần fan-out đơn giản? ───────────────────────────▶ SNS

Cần xử lý tin nhắn tuần tự, có retry? ──────────▶ SQS
Cần decouple producer và consumer? ──────────────▶ SQS
Cần xử lý batch (lô) lớn? ──────────────────────▶ SQS
```

---

## ⚡ Scheduled Rules — Quy Tắc Lịch Trình

EventBridge có thể kích hoạt target theo lịch (thay thế cron job):

### Rate Expression (Biểu Thức Tốc Độ)

```
rate(5 minutes)   — mỗi 5 phút
rate(1 hour)      — mỗi giờ
rate(1 day)       — mỗi ngày
```

### Cron Expression (Biểu Thức Cron)

```
cron(0 12 * * ? *)          — mỗi ngày lúc 12:00 UTC
cron(15 10 ? * MON-FRI *)   — 10:15 UTC các ngày thứ 2–6
cron(0 18 L * ? *)          — 18:00 UTC ngày cuối tháng
```

**Ví dụ thực tế:**

```python
# Tạo scheduled rule chạy Lambda mỗi ngày lúc 2 giờ sáng
aws events put-rule \
  --name "daily-cleanup-job" \
  --schedule-expression "cron(0 19 * * ? *)" \  # 19:00 UTC = 2:00 AM ICT
  --state ENABLED
```

---

## 🌐 Cross-Account và Cross-Region

### Cross-Account Event Routing (Định Tuyến Liên Tài Khoản)

```
Account A (Producer)              Account B (Consumer)
┌─────────────────┐               ┌─────────────────────┐
│ Custom Bus      │──────────────▶│ Default/Custom Bus  │
│ (source)        │  EventBridge  │ (target)            │
└─────────────────┘  Bus Policy  └─────────────────────┘
```

**Resource policy** trên Account B cần cho phép Account A:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_A_ID:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:ap-southeast-1:ACCOUNT_B_ID:event-bus/default"
    }
  ]
}
```

### Cross-Region (Liên Vùng)

EventBridge hỗ trợ gửi sự kiện từ vùng này sang vùng khác — hữu ích cho disaster recovery (khôi phục thảm họa) hoặc kiến trúc multi-region.

---

## 💰 Pricing (Giá Cả)

| Loại | Giá |
|---|---|
| **Custom events** | $1.00 / triệu events |
| **AWS service events** | Miễn phí trên default bus |
| **Cross-account events** | $1.00 / triệu events |
| **Schema discovery** | $0.10 / triệu events |
| **Archive** | $0.023 / GB-tháng |
| **Replay** | $1.00 / triệu events được phát lại |

**Free Tier (Bậc Miễn Phí):** Không có free tier cho EventBridge custom events.

---

## 🚀 Khi Nào Dùng EventBridge?

### ✅ Nên Dùng EventBridge Khi

- Cần **event routing phức tạp** — filter theo nhiều điều kiện JSON
- Cần tích hợp với **SaaS applications** (Shopify, Stripe, Zendesk...)
- Cần **Schema Registry** để quản lý contract (hợp đồng) giữa services
- Cần **Archive & Replay** để debug hoặc khôi phục sau sự cố
- Cần **cross-account event routing** trong tổ chức AWS Organizations
- Cần **scheduled tasks** thay thế cronjob
- Cần **EventBridge Pipes** để kết nối nguồn và đích đơn giản

### ❌ Không Nên Dùng EventBridge Khi

- Cần **latency rất thấp** (dưới 10ms) — dùng SNS
- Cần **task queue** với retry và DLQ — dùng SQS
- Cần **throughput cực cao** (triệu msg/giây) — dùng Kinesis
- Hệ thống **legacy** cần giao thức AMQP/MQTT — dùng Amazon MQ

---

## 🔗 Điều Hướng Module

| Bước | File | Nội Dung |
|---|---|---|
| 1 | [1-event-bus.md](1-event-bus.md) | Hiểu các loại Event Bus |
| 2 | [2-event-rules-patterns.md](2-event-rules-patterns.md) | Tạo Rule và Pattern lọc sự kiện |
| 3 | [3-schema-registry.md](3-schema-registry.md) | Quản lý Schema sự kiện |
| 4 | [4-archive-replay.md](4-archive-replay.md) | Lưu trữ và phát lại sự kiện |
| 5 | [5-eventbridge-pipes.md](5-eventbridge-pipes.md) | EventBridge Pipes nâng cao |

---

## 📝 Checklist Kiến Thức Nền Tảng

Sau khi đọc module này, bạn có thể:

- [ ] Giải thích EventBridge là gì và khác gì SNS/SQS
- [ ] Mô tả 3 loại Event Bus và khi nào dùng mỗi loại
- [ ] Viết được event JSON đúng chuẩn EventBridge
- [ ] Tạo Rule với Event Pattern lọc sự kiện
- [ ] Giải thích Schema Registry giải quyết vấn đề gì
- [ ] Mô tả cơ chế Archive & Replay
- [ ] Thiết kế cross-account event routing đơn giản
- [ ] So sánh EventBridge với SNS và SQS trong phỏng vấn

---

**Liên Kết:** [INDEX.md](../INDEX.md) | [02-sns/](../02-sns/) | [04-step-functions/](../04-step-functions/)
