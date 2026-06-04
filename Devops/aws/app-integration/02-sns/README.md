# Amazon SNS — Simple Notification Service (Dịch Vụ Thông Báo Đơn Giản)

> Dịch vụ Pub/Sub (Publisher/Subscriber — Nhà Phát/Người Đăng Ký) được quản lý hoàn toàn của AWS: một tin nhắn gửi đi, nhiều nơi nhận đồng thời.

## 📚 Mục Lục Module Này

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| [README.md](./README.md) | Tổng quan, kiến trúc, so sánh với SQS | Đọc đầu tiên |
| [1-topic-subscription.md](./1-topic-subscription.md) | Topic, Subscription, Protocol | ⭐⭐⭐ |
| [2-message-filtering.md](./2-message-filtering.md) | Message Filtering — Lọc Tin Nhắn | ⭐⭐⭐ |
| [3-fanout-pattern.md](./3-fanout-pattern.md) | Fan-out Pattern với SNS + SQS | ⭐⭐⭐ |
| [4-fifo-topic.md](./4-fifo-topic.md) | FIFO Topic — Thứ Tự & Exactly-Once | ⭐⭐ |

---

## 🎯 SNS Là Gì?

**Amazon SNS — Simple Notification Service** là dịch vụ Pub/Sub (Publisher/Subscriber — Nhà Phát/Người Đăng Ký) và A2A (Application-to-Application — Ứng Dụng Tới Ứng Dụng) / A2P (Application-to-Person — Ứng Dụng Tới Người) được quản lý hoàn toàn bởi AWS. SNS cho phép:

- **Fan-out** (khuếch tán) — một tin nhắn đến hàng triệu subscriber đồng thời
- **Push-based delivery** (giao vận kiểu đẩy) — SNS chủ động đẩy tin đến endpoint
- **Decoupling** (tách rời) — publisher không cần biết ai đang subscribe
- **Multi-protocol** (đa giao thức) — giao vận qua SQS, Lambda, HTTP/S, Email, SMS, Mobile Push

### Mô Hình Pub/Sub Cơ Bản

```
Publisher (Nhà Phát)        SNS Topic (Chủ Đề)       Subscribers (Người Đăng Ký)
─────────────────           ──────────────────        ───────────────────────────
                                                       ┌──▶ [SQS Queue A]
[Order Service]  ──msg──▶  [order-events topic]  ──┬──┼──▶ [SQS Queue B]
                                                   │  ├──▶ [Lambda Function]
[Payment Service] ──msg──▶ [payment-topic]        │  ├──▶ [Email: admin@...]
                                                   └──└──▶ [HTTP Endpoint]
```

SNS dùng mô hình **push-based** (đẩy): SNS chủ động gửi tin đến tất cả subscriber (khác với SQS — consumer phải chủ động kéo tin về).

---

## 🏗️ Kiến Trúc SNS

### Các Thành Phần Chính

```
┌──────────────────────────────────────────────────────┐
│                    SNS Topic                          │
│                                                        │
│  Publisher gọi: Publish(TopicArn, Message)             │
│                       │                               │
│                       ▼                               │
│         ┌─────────────────────────┐                   │
│         │   Message (Tin Nhắn)    │                   │
│         │  ├── MessageId          │                   │
│         │  ├── Subject            │                   │
│         │  ├── Message (body)     │                   │
│         │  ├── MessageAttributes  │                   │
│         │  └── Timestamp          │                   │
│         └─────────────────────────┘                   │
│                       │                               │
│            Fan-out tới tất cả Subscriptions           │
│          (Đăng Ký) đang Active (Đang Hoạt Động)       │
└──────────────────────────────────────────────────────┘
```

### Vòng Đời Tin Nhắn (Message Lifecycle)

```
1. PUBLISH (Phát)
   Publisher gọi Publish API với TopicArn
         │
         ▼
2. STORED (Lưu Tạm)
   SNS lưu tin nhắn tạm thời trên nhiều AZ (Availability Zone — Vùng Khả Dụng)
         │
         ▼
3. FAN-OUT (Khuếch Tán)
   SNS đẩy đồng thời đến TẤT CẢ subscription active
         │
     ┌───┴──────────────────┐
     ▼                      ▼
4a. DELIVERED (Đã Giao)  4b. FAILED (Thất Bại)
    Subscriber nhận thành công  Endpoint không phản hồi
                                 │
                                 ▼
                          RETRY (Thử Lại) tự động
                          → sau max retries → DLQ
```

---

## 📊 Các Loại SNS Topic

| Đặc Điểm | Standard Topic | FIFO Topic |
|---|---|---|
| **Thứ tự** | Best-effort (nỗ lực tốt nhất) | Đảm bảo FIFO chặt chẽ |
| **Throughput** (Thông Lượng) | 30.000.000 msg/s | 300 msg/s (3.000 với batching) |
| **Delivery** | At-least-once (ít nhất 1 lần) | Exactly-once (đúng 1 lần) |
| **Subscriber** | SQS, Lambda, HTTP/S, Email, SMS, Mobile | Chỉ SQS FIFO Queue |
| **Tên Topic** | Tùy ý | Phải kết thúc bằng `.fifo` |
| **Message Filtering** | ✅ | ✅ |

---

## ⚙️ Các Tham Số Cấu Hình Quan Trọng

### Message Size (Kích Thước Tin Nhắn)

- **Tối đa:** 256 KB
- **Mẹo:** Dùng SNS Extended Client Library kết hợp S3 cho payload lớn hơn

### Message Retention (Lưu Giữ Tin Nhắn)

- SNS **không lưu trữ tin nhắn** — nếu subscriber không nhận được ngay, SNS sẽ retry trong thời gian nhất định rồi bỏ qua (hoặc đẩy vào DLQ — Dead Letter Queue — Hàng Đợi Thư Chết nếu đã cấu hình)
- **Không có retention period** như SQS — đây là điểm khác biệt quan trọng

### Delivery Retry Policy (Chính Sách Thử Lại Giao Vận)

SNS tự động retry khi giao vận thất bại:

| Giai Đoạn | Thời Gian Chờ | Số Lần |
|---|---|---|
| **Immediate** (Ngay Lập Tức) | 0 giây | 3 lần |
| **Pre-backoff** (Trước Giãn Cách) | 20 giây | 2 lần |
| **Backoff** (Giãn Cách Lũy Thừa) | 20–1880 giây | 10 lần |
| **Post-backoff** (Sau Giãn Cách) | 1880 giây | Cho đến hết TTL |

**TTL (Time-To-Live — Thời Gian Sống):** tổng thời gian retry tối đa, mặc định 1 giờ đến 23 ngày tùy endpoint type.

---

## 🔑 Các API Quan Trọng

```bash
# Tạo SNS topic
aws sns create-topic --name order-events

# Tạo FIFO topic
aws sns create-topic \
  --name order-events.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=true

# Subscribe SQS queue vào topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456:order-queue

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --protocol email \
  --notification-endpoint admin@example.com

# Publish tin nhắn
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --message '{"orderId":"ORD-001","status":"created","amount":100}' \
  --subject "New Order"

# Publish với message attributes (cho filtering)
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events \
  --message '{"orderId":"ORD-001"}' \
  --message-attributes '{"orderType":{"DataType":"String","StringValue":"express"}}'

# Liệt kê subscriptions của topic
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:123456:order-events
```

---

## 🕐 Khi Nào Dùng SNS?

### Dùng SNS khi:

✅ Cần **broadcast** (phát sóng) một event đến nhiều subscriber đồng thời  
✅ Cần **push notification** (thông báo đẩy) đến user qua Email, SMS, Mobile  
✅ Cần **fan-out** (khuếch tán) — một sự kiện kích hoạt nhiều luồng xử lý  
✅ Cần **alert** (cảnh báo) người dùng hoặc hệ thống khác theo thời gian thực  
✅ Khi các subscriber cần nhận **cùng một tin nhắn** tại cùng một thời điểm

### KHÔNG dùng SNS khi:

❌ Cần **queue** (hàng đợi) và consumer xử lý theo tốc độ riêng → dùng **SQS**  
❌ Cần **store** (lưu trữ) tin nhắn chờ consumer sẵn sàng → dùng **SQS**  
❌ Cần **complex event routing** (định tuyến sự kiện phức tạp) dựa trên schema → dùng **EventBridge**  
❌ Cần **real-time streaming** (luồng thời gian thực) với replay → dùng **Kinesis**  
❌ Cần protocol AMQP/MQTT (cho legacy systems — hệ thống cũ) → dùng **Amazon MQ**

---

## 🔄 SNS vs SQS — Phân Biệt Rõ Ràng

| Tiêu Chí | SNS | SQS |
|---|---|---|
| **Mô hình** | Pub/Sub (Phát/Đăng Ký) | Point-to-point (Điểm Tới Điểm) |
| **Delivery** | Push (Đẩy) | Pull (Kéo) |
| **Lưu trữ** | Không lưu tin nhắn | Lưu tối đa 14 ngày |
| **Receivers** | Nhiều subscriber cùng lúc | Một consumer mỗi lần |
| **Persistence** (Lưu Bền) | Không | Có |
| **Retry** | Tự động retry theo policy | Consumer tự retry |
| **Use case** | Notification, fan-out, alert | Task queue, decoupling, buffer |

### Kết Hợp SNS + SQS — Best Practice

```
[Publisher]
    │ Publish
    ▼
[SNS Topic]
    │
    ├──▶ [SQS Queue A] ──▶ [Consumer A — xử lý chậm, không mất dữ liệu]
    ├──▶ [SQS Queue B] ──▶ [Consumer B — xử lý độc lập]
    └──▶ [Lambda]      ──▶ [Xử lý ngay, không cần queue]
```

Kết hợp SNS + SQS = **tốt nhất của cả hai**: fan-out của SNS + durability (độ bền) và buffering của SQS.

---

## 🏭 Use Cases Thực Tế

### 1. Order Processing Fan-out (Khuếch Tán Xử Lý Đơn Hàng)

```
[Order API] POST /orders
    │ Publish "order.created"
    ▼
[SNS: order-events]
    │
    ├──▶ [SQS: payment-queue]   ──▶ [Payment Service — Dịch Vụ Thanh Toán]
    ├──▶ [SQS: inventory-queue] ──▶ [Inventory Service — Dịch Vụ Kho Hàng]
    ├──▶ [SQS: shipping-queue]  ──▶ [Shipping Service — Dịch Vụ Vận Chuyển]
    └──▶ [Lambda: notify-user]  ──▶ [Email/SMS xác nhận đơn hàng cho khách]
```

### 2. System Monitoring Alert (Cảnh Báo Giám Sát Hệ Thống)

```
[CloudWatch Alarm]
    │ Trigger
    ▼
[SNS: ops-alerts]
    │
    ├──▶ [Email: on-call-engineer@...]
    ├──▶ [SMS: +84-xxx-xxx-xxx]
    └──▶ [Lambda: PagerDuty / Slack webhook]
```

### 3. Microservices Event Broadcasting (Phát Sóng Sự Kiện Giữa Microservices)

```
[User Service]
    │ Publish "user.registered"
    ▼
[SNS: user-events]
    │
    ├──▶ [SQS] ──▶ [Email Service — gửi email chào mừng]
    ├──▶ [SQS] ──▶ [Analytics Service — ghi nhận sự kiện đăng ký]
    └──▶ [SQS] ──▶ [Recommendation Service — khởi tạo profile gợi ý]
```

---

## 💰 Chi Phí (Pricing)

| Loại | Chi Phí |
|---|---|
| **API Request** (Yêu Cầu API) | $0.50 / 1 triệu request |
| **Notification to SQS/Lambda** | $0.50 / 1 triệu notification |
| **Email/Email-JSON** | $2.00 / 100.000 notification |
| **SMS (US)** | $0.00645 / tin nhắn |
| **HTTP/HTTPS** | $0.60 / 1 triệu notification |
| **Mobile Push** | $1.00 / 1 triệu notification |
| **Free Tier** | 1 triệu SNS request/tháng miễn phí |

---

## 🔒 Bảo Mật

### IAM Policy (Chính Sách Quyền Truy Cập)

```json
{
  "Effect": "Allow",
  "Action": [
    "sns:Publish",
    "sns:Subscribe",
    "sns:ListTopics",
    "sns:GetTopicAttributes"
  ],
  "Resource": "arn:aws:sns:us-east-1:123456789:order-events"
}
```

### Topic Policy (Chính Sách Topic)

Cho phép cross-account publishing (phát sóng liên tài khoản):

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::OTHER_ACCOUNT_ID:root"
  },
  "Action": "sns:Publish",
  "Resource": "arn:aws:sns:us-east-1:123456789:order-events"
}
```

### Encryption (Mã Hóa)

- **SSE** (Server-Side Encryption — Mã Hóa Phía Máy Chủ) với AWS KMS
- **In-transit** — HTTPS mặc định
- **At-rest** — mã hóa tin nhắn lưu tạm trong SNS

---

## 📈 Monitoring (Giám Sát)

### Metrics CloudWatch Quan Trọng

| Metric (Chỉ Số) | Ý Nghĩa | Ngưỡng Cảnh Báo |
|---|---|---|
| `NumberOfMessagesSent` | Số tin nhắn publish vào topic | Đột biến bất thường |
| `NumberOfNotificationsDelivered` | Số notification giao thành công | Nên ≈ Sent |
| `NumberOfNotificationsFailed` | Số notification thất bại | > 0 → cần điều tra |
| `PublishSize` | Kích thước tin nhắn | Gần 256KB → tối ưu payload |
| `SMSSuccessRate` | Tỷ lệ giao SMS thành công | < 95% → cần xem xét |

---

## 🎯 Tóm Tắt Nhanh Cho Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
|---|---|
| SNS là gì? | Managed Pub/Sub service, push-based, fan-out đến nhiều subscriber |
| SNS vs SQS? | SNS: push, fan-out, không lưu. SQS: pull, one-consumer, lưu tới 14 ngày |
| Subscriber là gì? | SQS, Lambda, HTTP/S, Email, SMS, Mobile Push |
| Message Filtering? | Subscriber chỉ nhận tin nhắn khớp filter policy dựa trên message attributes |
| Fan-out pattern? | SNS publish một lần → nhiều SQS queue nhận đồng thời |
| FIFO Topic? | Đảm bảo thứ tự và exactly-once, chỉ subscribe được SQS FIFO |
| SNS có lưu tin nhắn không? | Không — nếu delivery thất bại sau retry → mất (trừ khi có DLQ) |
| Tối đa message size? | 256 KB |

---

**Tiếp theo:** [1-topic-subscription.md](./1-topic-subscription.md) — Topic, Subscription, Protocol chi tiết
