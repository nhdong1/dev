# Amazon SQS — Simple Queue Service (Dịch Vụ Hàng Đợi Đơn Giản)

> Nền tảng messaging của AWS: tách rời (decouple) các thành phần ứng dụng, xử lý tải không đều, đảm bảo không mất tin nhắn.

## 📚 Mục Lục Module Này

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| [README.md](./README.md) | Tổng quan, kiến trúc, khi nào dùng SQS | Đọc đầu tiên |
| [1-standard-vs-fifo.md](./1-standard-vs-fifo.md) | So sánh Standard Queue và FIFO Queue | ⭐⭐⭐ |
| [2-visibility-timeout.md](./2-visibility-timeout.md) | Visibility Timeout và Idempotency | ⭐⭐⭐ |
| [3-dead-letter-queue.md](./3-dead-letter-queue.md) | DLQ — Hàng Đợi Thư Chết | ⭐⭐⭐ |
| [4-long-polling.md](./4-long-polling.md) | Long Polling vs Short Polling | ⭐⭐ |
| [5-batch-processing.md](./5-batch-processing.md) | Xử lý theo lô với SQS | ⭐⭐ |

---

## 🎯 SQS Là Gì?

**Amazon SQS — Simple Queue Service** là dịch vụ hàng đợi tin nhắn được quản lý hoàn toàn (fully managed) bởi AWS. SQS cho phép:

- **Decouple** (tách rời) producer (nhà sản xuất) và consumer (người tiêu dùng) — hai bên không cần biết về nhau
- **Buffer** (đệm) tải đột biến — consumer xử lý ở tốc độ riêng của mình
- **Durability** (độ bền) — tin nhắn được lưu trữ dự phòng trên nhiều AZ (Availability Zone — Vùng Khả Dụng)
- **Scale** tự động — không cần quản lý infrastructure

### Mô Hình Cơ Bản

```
Producer          Queue (Hàng Đợi)         Consumer
─────────         ────────────────         ────────
[App A]  ──msg──▶  [ m1 | m2 | m3 ]  ──▶  [App B]
[App B]  ──msg──▶  [ m4 | m5 | m6 ]  ──▶  [App C]
```

SQS dùng mô hình **pull-based** (kéo): consumer chủ động lấy tin nhắn từ queue (khác với SNS dùng push-based — đẩy).

---

## 🏗️ Kiến Trúc SQS

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────┐
│                  SQS Queue                      │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Message (Tin nhắn)                      │   │
│  │  ├── MessageId: "abc-123"                │   │
│  │  ├── Body: "{ ... }"                     │   │
│  │  ├── MessageAttributes: { key: value }   │   │
│  │  ├── ReceiptHandle: "<opaque token>"     │   │
│  │  └── Metadata: timestamp, delay, ...    │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

### Vòng Đời Tin Nhắn (Message Lifecycle)

```
1. SEND (Gửi)
   Producer gọi SendMessage API
         │
         ▼
2. STORED (Lưu Trữ)
   Tin nhắn nằm trong queue, visible (hiển thị) với consumer
         │
         ▼
3. RECEIVED (Nhận)
   Consumer gọi ReceiveMessage API
   → Tin nhắn trở thành INVISIBLE (ẩn) trong Visibility Timeout
         │
         ▼
4a. DELETED (Xóa) — Xử lý thành công
    Consumer gọi DeleteMessage với ReceiptHandle
         │
    ─────┴─────
         │
4b. VISIBLE lại — Xử lý thất bại / timeout
    Sau khi Visibility Timeout hết → tin nhắn visible trở lại
    → Consumer khác có thể nhận và xử lý lại
```

---

## 📊 Hai Loại Queue

| Đặc Điểm | Standard Queue | FIFO Queue |
|---|---|---|
| **Thứ tự** | Best-effort (nỗ lực tốt nhất) | Đảm bảo FIFO chặt chẽ |
| **Throughput** (Thông Lượng) | Không giới hạn | 300 msg/s (3.000 với batching) |
| **Delivery** | At-least-once (ít nhất 1 lần) | Exactly-once (đúng 1 lần) |
| **Duplicate** (Trùng Lặp) | Có thể xảy ra | Deduplication (khử trùng) tự động |
| **Giá** | Thấp hơn | Cao hơn ~5% |
| **Tên queue** | Tùy ý | Phải kết thúc bằng `.fifo` |

Chi tiết xem: [1-standard-vs-fifo.md](./1-standard-vs-fifo.md)

---

## ⚙️ Các Tham Số Cấu Hình Quan Trọng

### Visibility Timeout (Thời Gian Ẩn Tin Nhắn)

- **Mặc định:** 30 giây
- **Tối thiểu / Tối đa:** 0 giây / 12 giờ
- **Ý nghĩa:** Khi consumer nhận tin nhắn, tin nhắn ẩn khỏi queue trong khoảng thời gian này để tránh consumer khác nhận trùng

### Message Retention Period (Chu Kỳ Lưu Giữ Tin Nhắn)

- **Mặc định:** 4 ngày
- **Tối thiểu / Tối đa:** 1 phút / 14 ngày
- **Ý nghĩa:** Tin nhắn chưa bị xóa sẽ tự động bị xóa sau thời gian này

### Message Size (Kích Thước Tin Nhắn)

- **Tối đa:** 256 KB
- **Mẹo:** Dùng S3 + SQS Extended Client Library cho payload lớn hơn

### Delivery Delay (Độ Trễ Giao Vận)

- **Mặc định:** 0 giây
- **Tối đa:** 15 phút
- **Ý nghĩa:** Tin nhắn sẽ không xuất hiện trong queue cho đến khi delay hết — dùng cho scheduled tasks (tác vụ lên lịch)

### Receive Message Wait Time (Thời Gian Chờ Nhận Tin Nhắn)

- **Mặc định:** 0 giây (short polling — thăm dò ngắn)
- **Tối đa:** 20 giây (long polling — thăm dò dài)
- **Khuyến nghị:** Đặt ≥ 1 giây để giảm empty response và chi phí

---

## 🔑 Các API Quan Trọng

```bash
# Tạo queue
aws sqs create-queue \
  --queue-name my-queue \
  --attributes VisibilityTimeout=30

# Gửi tin nhắn
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --message-body '{"orderId": "ORD-001", "amount": 100}'

# Nhận tin nhắn (Long Polling 10 giây)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --wait-time-seconds 10 \
  --max-number-of-messages 10

# Xóa tin nhắn sau khi xử lý
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --receipt-handle "<receipt-handle-from-receive>"

# Lấy số tin nhắn đang chờ
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456/my-queue \
  --attribute-names ApproximateNumberOfMessages
```

---

## 🕐 Khi Nào Dùng SQS?

### Dùng SQS khi:

✅ Cần **tách rời** (decouple) producer và consumer  
✅ Consumer xử lý chậm hơn producer — cần **buffer** (đệm)  
✅ Cần đảm bảo **không mất tin nhắn** kể cả khi consumer bị lỗi  
✅ Xử lý **background jobs** (tác vụ nền): gửi email, resize ảnh, xuất PDF  
✅ **Load leveling** (san phẳng tải) — tránh spike (đột biến) đến backend  
✅ **Task distribution** (phân phối tác vụ) giữa nhiều worker

### KHÔNG dùng SQS khi:

❌ Cần **broadcast** (phát sóng) đến nhiều subscriber → dùng **SNS**  
❌ Cần xử lý **real-time streaming** (luồng thời gian thực) → dùng **Kinesis**  
❌ Cần **complex routing** (định tuyến phức tạp) theo event type → dùng **EventBridge**  
❌ Cần **request-response** đồng bộ → dùng API Gateway + Lambda  
❌ Cần protocol AMQP/MQTT (for legacy systems — hệ thống cũ) → dùng **Amazon MQ**

---

## 🏭 Use Cases Thực Tế

### 1. Order Processing (Xử Lý Đơn Hàng)

```
[Web API]
    │ POST /orders
    ▼
[SQS: order-queue]
    │
    ▼
[Order Service]    [Payment Service]    [Inventory Service]
 - Validate order   - Charge card        - Reserve stock
 - Create record    - Send receipt       - Update count
```

**Lợi ích:** Nếu Payment Service bị lỗi, đơn hàng vẫn được lưu trong queue, không mất.

### 2. Email/Notification Queue (Hàng Đợi Email/Thông Báo)

```
[Trigger Event]  ──▶  [SQS: email-queue]  ──▶  [Email Worker]
  - User signup                                    - Render template
  - Password reset                                 - Send via SES
  - Order shipped                                  - Log result
```

**Lợi ích:** Email worker xử lý ở tốc độ riêng, không làm chậm API.

### 3. Image Processing (Xử Lý Ảnh)

```
[S3 Upload Event]
    │
    ▼
[SQS: image-queue]
    │
    ├─▶ [Worker 1] Resize 1080p
    ├─▶ [Worker 2] Resize 720p
    └─▶ [Worker 3] Generate thumbnail
```

**Lợi ích:** Nhiều worker xử lý song song, mỗi worker lấy một tin nhắn riêng.

---

## 💰 Chi Phí (Pricing)

| Loại | Chi Phí |
|---|---|
| **Requests** (Yêu Cầu) | $0.40 / 1 triệu request |
| **Free Tier** | 1 triệu request miễn phí / tháng |
| **Data Transfer** | Miễn phí nội vùng AWS |

**Chú ý:**
- Mỗi `ReceiveMessage` tính là 1 request kể cả khi queue rỗng
- Dùng **Long Polling** để giảm số request rỗng
- `SendMessageBatch` và `DeleteMessageBatch` giúp gửi/xóa tới 10 tin nhắn trong 1 request

---

## 🔒 Bảo Mật

### IAM Policy (Chính Sách Quyền Truy Cập)

```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage",
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage",
    "sqs:GetQueueAttributes"
  ],
  "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue"
}
```

### Encryption (Mã Hóa)

- **SSE-SQS** — SQS quản lý key (khóa mã hóa) tự động
- **SSE-KMS** — dùng AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) cho kiểm soát tốt hơn
- **In-transit encryption** — HTTPS mặc định

### Queue Policy (Chính Sách Hàng Đợi)

Cho phép cross-account access (truy cập liên tài khoản) hoặc từ dịch vụ AWS khác:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "sns.amazonaws.com"
  },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue",
  "Condition": {
    "ArnEquals": {
      "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:my-topic"
    }
  }
}
```

---

## 📈 Monitoring (Giám Sát)

### Metrics CloudWatch Quan Trọng

| Metric (Chỉ Số) | Ý Nghĩa | Ngưỡng Cảnh Báo |
|---|---|---|
| `ApproximateNumberOfMessagesVisible` | Số tin nhắn đang chờ xử lý | > 1000 (tùy hệ thống) |
| `ApproximateAgeOfOldestMessage` | Tuổi của tin nhắn cũ nhất (giây) | > visibility timeout |
| `NumberOfMessagesSent` | Số tin nhắn được gửi / phút | Đột biến bất thường |
| `NumberOfMessagesDeleted` | Số tin nhắn bị xóa / phút | Nên tương đương Sent |
| `NumberOfMessagesReceived` | Số tin nhắn được nhận / phút | Theo dõi consumer |

```bash
# Kiểm tra queue depth (độ sâu hàng đợi)
aws cloudwatch get-metric-statistics \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=my-queue \
  --start-time 2026-01-01T00:00:00Z \
  --end-time 2026-01-01T01:00:00Z \
  --period 60 \
  --statistics Average
```

---

## 🔗 Tích Hợp Với Dịch Vụ AWS Khác

```
SNS ──fanout──▶  SQS ──trigger──▶  Lambda
                  │                  │
                  │                  ▼
                  │               DynamoDB / RDS
                  │
                  └──trigger──▶  ECS Task (Fargate)
                  │
                  └──trigger──▶  EC2 Auto Scaling Group
```

- **SNS → SQS** — Fan-out pattern (Mẫu Khuếch Tán): một event đến nhiều queue
- **SQS → Lambda** — Event Source Mapping (Ánh Xạ Nguồn Sự Kiện): Lambda tự động polling queue
- **SQS → ECS** — Scale ECS tasks dựa trên queue depth

---

## 🎯 Tóm Tắt Nhanh Cho Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
|---|---|
| SQS là gì? | Managed message queue, decouple producer-consumer, pull-based |
| Standard vs FIFO? | Standard: unlimited throughput, at-least-once. FIFO: ordered, exactly-once, 300/3000 msg/s |
| Visibility Timeout? | Khoảng thời gian tin nhắn ẩn sau khi consumer nhận, tránh xử lý trùng |
| DLQ là gì? | Queue nhận tin nhắn thất bại sau N lần xử lý — Dead Letter Queue |
| Long Polling? | ReceiveMessage chờ tối đa 20s nếu queue rỗng, giảm empty call và chi phí |
| Khi nào dùng FIFO? | Cần thứ tự chặt chẽ: financial transactions, inventory updates |
| Tối đa message size? | 256 KB; dùng S3 Extended Client cho payload lớn hơn |
| Retention tối đa? | 14 ngày |

---

**Tiếp theo:** [1-standard-vs-fifo.md](./1-standard-vs-fifo.md) — So sánh chi tiết hai loại queue
