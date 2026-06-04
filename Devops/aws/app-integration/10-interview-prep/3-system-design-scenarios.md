# Bài Toán Thiết Kế Hệ Thống — AWS Application Integration

> 5 bài toán thực tế từ phỏng vấn senior engineer — đọc đề, tự vẽ sơ đồ, rồi so sánh với lời giải

---

## Hướng Dẫn Sử Dụng

Với mỗi bài toán:
1. **Đọc đề** — xác định các yêu cầu functional và non-functional
2. **Tự thiết kế** — vẽ sơ đồ kiến trúc, ghi chú lý do chọn từng dịch vụ
3. **So sánh với lời giải** — chú ý những điểm bạn bỏ sót hoặc quyết định khác
4. **Thực hành giải thích** — nói to như đang phỏng vấn thật

---

## Bài 1: Hệ Thống Xử Lý Đơn Hàng E-Commerce

### Đề Bài

Bạn được yêu cầu thiết kế hệ thống xử lý đơn hàng cho sàn thương mại điện tử với:

**Yêu cầu chức năng:**
- Xử lý 50,000 đơn hàng/giờ (peak: 150,000 đơn hàng/giờ vào dịp sale)
- Khi đặt hàng: trừ inventory → charge thẻ → tạo shipment
- Thông báo cho khách hàng qua email + SMS sau mỗi bước
- Nếu bước nào thất bại → rollback toàn bộ

**Yêu cầu phi chức năng:**
- Không mất đơn hàng dù có lỗi ở bất kỳ bước nào
- Đơn hàng của cùng khách hàng phải xử lý đúng thứ tự
- Hệ thống phải có thể debug được khi lỗi xảy ra
- SLA: xử lý 95% đơn hàng trong vòng 30 giây

---

### Lời Giải

#### Kiến Trúc Tổng Quan

```
[Client]
    ↓ POST /orders
[API Gateway + Lambda — Order API]
    ↓ (1) Ghi DB + Outbox trong cùng 1 transaction
[RDS PostgreSQL — orders + outbox table]
    ↓ (2) CDC với Debezium / DMS
[SQS FIFO Queue — order-intake]
  (MessageGroupId = customer_id — đảm bảo thứ tự per customer)
    ↓ (3) Lambda Trigger
[Step Functions — Order Workflow]
  ├── (4a) Reserve Inventory → Lambda → DynamoDB
  ├── (4b) Charge Payment → Lambda → Stripe API
  └── (4c) Create Shipment → Lambda → Shipping API
    ↓ (5) Khi tất cả bước thành công
[SNS — order-completed]
  ├── SQS → Lambda → Email Service (SES)
  └── SQS → Lambda → SMS Service (SNS SMS)
```

#### Quyết Định Thiết Kế

**SQS FIFO với MessageGroupId = customer_id:**
- Đảm bảo đơn hàng của cùng khách hàng được xử lý tuần tự
- Các customer khác nhau xử lý song song → throughput cao

**Outbox Pattern:**
- Ghi DB và Outbox trong cùng 1 transaction → không bao giờ mất đơn
- CDC (Change Data Capture — Thu Nạp Thay Đổi) đọc Outbox → đẩy vào SQS

**Step Functions Standard Workflow:**
- Saga pattern: mỗi bước có compensating transaction (giao dịch bù trừ)
- Execution history đầy đủ → debug dễ dàng
- Nếu Charge Payment fail → tự động chạy Release Inventory compensation

**Compensation Flow khi Payment fail:**
```
[Reserve Inventory ✅] → [Charge Payment ❌]
                                ↓
                    [Release Inventory] (compensation)
                                ↓
                    [Cancel Order] → [Notify Customer]
```

**SNS → SQS Fan-out cho notification:**
- Email và SMS xử lý độc lập, lỗi ở một bên không ảnh hưởng bên kia
- Có thể thêm notification channel mới mà không cần thay đổi order workflow

#### Xử Lý Peak Traffic (150,000 đơn/giờ = 41 đơn/giây)

- SQS FIFO limit: 300 TPS → 41 TPS thoải mái
- Lambda scale: Mỗi Lambda concurrently xử lý 1 message → auto scale
- Step Functions: 2,000 execution/giây → đủ dùng

#### Monitoring

```
Alert 1: SQS FIFO Depth > 1000 → Đang tồn đọng
Alert 2: Step Functions ExecutionsFailed > 0 → Có lỗi workflow
Alert 3: DLQ Depth > 0 → Có message không xử lý được
Alert 4: ApproximateAgeOfOldestMessage > 60s → Vi phạm SLA
```

---

## Bài 2: Hệ Thống Real-Time Analytics Clickstream

### Đề Bài

Thiết kế hệ thống thu thập và phân tích hành vi người dùng trên website:

**Yêu cầu chức năng:**
- Thu thập 100,000 events/giây từ frontend (page views, clicks, cart adds)
- Dashboard real-time với delay < 5 giây
- Lưu raw data vào data lake để phân tích batch sau
- Phát hiện fraud pattern (hành vi gian lận) real-time

**Yêu cầu phi chức năng:**
- Không mất event dù producer peak
- Có thể replay lại dữ liệu để re-process
- Scale theo traffic, giảm cost khi traffic thấp

---

### Lời Giải

#### Kiến Trúc

```
[Browser / Mobile App]
    ↓ Batch events (10 events mỗi request)
[API Gateway + Lambda — Event Ingestion]
    ↓
[Kinesis Data Streams — clickstream]
  (100 shard: 100,000 events/s ÷ 1,000 records/s per shard)
    │
    ├── [Consumer 1: Real-time Dashboard]
    │   Lambda (Enhanced Fan-Out — 2MB/s riêng)
    │       ↓
    │   [DynamoDB — aggregated_metrics]
    │       ↓
    │   [API Gateway WebSocket → Dashboard]
    │
    ├── [Consumer 2: Fraud Detection]
    │   Kinesis Data Analytics (Apache Flink)
    │   → Tumbling Window 30s — detect anomaly
    │       ↓ khi phát hiện fraud
    │   [EventBridge → SNS → Security Team Alert]
    │
    └── [Consumer 3: Data Lake]
        Kinesis Firehose (buffer 60s)
            ↓
        [S3 — raw/year=2026/month=05/]
            ↓ (daily)
        [AWS Glue ETL → Parquet format]
            ↓
        [Amazon Athena / Redshift — batch analytics]
```

#### Quyết Định Thiết Kế

**Kinesis Data Streams (không phải SQS):**
- Cần 3 consumer đọc cùng stream với mục đích khác nhau → SQS không làm được
- Cần replay để re-process khi có bug trong fraud detection
- Ordering trong shard theo session_id → coherent user journey

**Tính số shard:**
```
100,000 events/giây × 200 bytes/event = 20 MB/giây
→ 20 shard cho write (20MB / 1MB per shard)

3 consumer × 20 MB/giây read = 60 MB/giây read
Với Enhanced Fan-Out: 60MB / 2MB per consumer per shard = 30 shard
→ Cần max(20, 30) = 30 shard

Thêm 30% buffer = 40 shard (làm tròn lên 50 để dễ quản lý)
```

**Enhanced Fan-Out cho Real-time Dashboard:**
- 3 consumer đọc cùng stream → cần dedicated throughput cho real-time dashboard
- Fraud Detection có thể dùng standard consumer nếu 5–10 giây delay chấp nhận được

**Kinesis Firehose cho Data Lake:**
- Không cần viết consumer code — fully managed
- Tự động buffer và compress (GZIP) trước khi ghi S3 → giảm chi phí S3

#### Tối Ưu Chi Phí

```
Shard: 40 shard × $0.015/shard-hour = $0.60/giờ = $432/tháng
Enhanced Fan-Out: 2 consumer × 40 shard × $0.015/shard-consumer-hour = $864/tháng
Firehose: $0.029/GB → tùy volume

Tổng ước tính: ~$1,500/tháng cho 100K events/giây
```

---

## Bài 3: Microservices Communication — Multi-Service Saga

### Đề Bài

Bạn có 5 microservice độc lập: Order, Inventory, Payment, Shipping, Notification. Thiết kế kiến trúc để:

**Yêu cầu:**
- Khi đặt hàng: phải đảm bảo cả 4 service (Inventory, Payment, Shipping, Notification) đều được thực hiện
- Nếu bất kỳ service nào fail → rollback toàn bộ
- Mỗi service có database riêng (không share database)
- Hệ thống vẫn chạy khi một service tạm thời down

---

### Lời Giải

#### Lựa Chọn: Orchestration vs Choreography

**Choreography** (Vũ Đạo) — Event-driven, không trung tâm:
```
Order Service → [OrderCreated event] → Inventory Service
Inventory Service → [InventoryReserved event] → Payment Service
Payment Service → [PaymentCharged event] → Shipping Service
Shipping Service → [ShipmentCreated event] → Notification Service
```

Nhược điểm: Khó trace, khó debug, khó hiểu flow tổng thể.

**Orchestration** (Điều Phối) với Step Functions — Khuyến nghị cho bài này:

```
[Order API Lambda]
    ↓ Start Execution
[Step Functions — OrderSagaWorkflow]
  ├── ReserveInventory
  │     ↓ success
  ├── ChargePayment
  │     ↓ success
  ├── CreateShipment
  │     ↓ success
  └── SendNotification (không cần compensate — fire-and-forget)

Compensation khi ChargePayment fail:
  ChargePayment → Catch → ReleaseInventory → CancelOrder → NotifyFailure
```

#### Kiến Trúc Chi Tiết

```
[Order API]
    ↓
[Step Functions — Standard Workflow]
    ↓
    ├── Task: ReserveInventory
    │   Resource: arn:aws:states:::sqs:sendMessage.waitForTaskToken
    │   → SQS → Inventory Service Lambda
    │          → Inventory DB update
    │          → Gọi SendTaskSuccess(taskToken)
    │
    ├── Task: ChargePayment
    │   Resource: arn:aws:states:::sqs:sendMessage.waitForTaskToken
    │   → SQS → Payment Service Lambda
    │          → Stripe API
    │          → Gọi SendTaskSuccess(taskToken)
    │
    ├── Task: CreateShipment
    │   (tương tự)
    │
    └── Task: SendNotification
        Resource: arn:aws:states:::sns:publish
        → SNS → Email + SMS Lambda
```

#### Xử Lý Service Down

**Vấn đề:** Nếu Inventory Service Lambda down khi đang xử lý.

**Giải pháp với SQS + waitForTaskToken:**
```
Step Functions → SQS Queue → Inventory Lambda
                (message persist trong SQS khi Lambda down)
                              ↑
                Khi Lambda restart → lấy message từ SQS → xử lý → callback
```

Step Functions chờ callback không giới hạn (tối đa 1 năm). Message an toàn trong SQS. Khi service recover, tự động tiếp tục.

#### Idempotency Trong Mỗi Service

```python
# Inventory Service — Mỗi lần nhận message
def reserve_inventory(event):
    task_token = event["taskToken"]
    order_id = event["orderId"]

    # Check đã xử lý chưa
    if already_processed(order_id, "RESERVE"):
        stepfunctions.send_task_success(taskToken=task_token, output=...)
        return

    # Xử lý lần đầu
    with transaction():
        reserve_items(order_id)
        mark_processed(order_id, "RESERVE")

    stepfunctions.send_task_success(taskToken=task_token, output=...)
```

---

## Bài 4: Event-Driven Notification System

### Đề Bài

Thiết kế hệ thống thông báo cho ứng dụng social media với:

**Yêu cầu:**
- 10 triệu người dùng, mỗi action (like, comment, follow) sinh ra notification
- Push notification đến mobile, email, in-app
- User có thể tùy chỉnh notification preferences (cài đặt tùy chọn thông báo)
- Thông báo real-time qua WebSocket trong app
- Batch notification (digest email) hàng ngày

---

### Lời Giải

#### Kiến Trúc

```
[Social Actions — Like/Comment/Follow]
    ↓
[EventBridge Custom Bus — social-events]
    │
    ├── Rule: all events → Archive (lưu 30 ngày)
    │
    ├── Rule: source=like/comment → SQS — notification-queue
    │                               ↓
    │                   [Notification Router Lambda]
    │                   → Đọc User Preferences từ DynamoDB
    │                   → Fan-out theo preferences
    │                       ↓           ↓           ↓
    │                   Push Queue  Email Queue  In-App Queue
    │                       ↓           ↓           ↓
    │                   FCM/APNs      SES         WebSocket
    │
    └── Rule: schedule = daily 08:00 → Lambda
                                       → Aggregate notifications
                                       → Send Digest Email via SES
```

#### EventBridge vs SNS — Tại Sao Chọn EventBridge

- Cần filter phức tạp: chỉ gửi push notification khi user có `pushEnabled = true`
- Cần route theo nhiều điều kiện (platform: iOS vs Android)
- Cần Archive để debug và audit

#### Real-Time In-App với AppSync

```
[Notification Lambda]
    ↓ Publish event
[AWS AppSync — GraphQL Mutation]
    ↓ (subscription)
[Client WebSocket] — User nhận notification ngay lập tức
```

#### Tối Ưu Cho 10 Triệu User

**Vấn đề:** Không phải tất cả action đều sinh notification gửi đi.

```
Pre-filtering tại EventBridge:
{
  "detail": {
    "targetUserActive": [true],  // Chỉ user đang active
    "notificationEnabled": [true] // Chỉ user bật notification
  }
}
→ Giảm 70-80% message không cần thiết trước khi vào queue
```

**Batch Digest Email:**
```
DynamoDB — pending_notifications table:
  user_id | notification | created_at

Daily Lambda (triggered by EventBridge Scheduled Rule):
  → Scan user với digest preference
  → Aggregate notifications của từng user
  → Gửi email batch qua SES
  → Delete processed notifications
```

---

## Bài 5: Multi-Region Disaster Recovery cho Messaging System

### Đề Bài

Hệ thống tài chính yêu cầu:

**Yêu cầu:**
- RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi): < 5 phút
- RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi): 0 — không mất message
- Hệ thống phải chạy ở 2 region: ap-southeast-1 (primary) và ap-east-1 (secondary)
- Failover (chuyển đổi dự phòng) tự động khi region primary down

---

### Lời Giải

#### Chiến Lược Active-Active vs Active-Passive

**Active-Passive** (Đơn Giản Hơn) — Khuyến nghị cho RTO < 5 phút:
```
Primary Region (ap-southeast-1): Nhận tất cả traffic
Secondary Region (ap-east-1): Standby, sync data, sẵn sàng tiếp nhận
```

**Active-Active** (Phức Tạp Hơn): Cả hai region nhận traffic — cần giải quyết conflict.

#### Kiến Trúc Active-Passive

```
[Route 53 Health Check]
    ↓ Primary healthy
[Primary: ap-southeast-1]
    ├── SQS Queues
    ├── Lambda Consumers
    └── RDS (Multi-AZ)
         ↓ Replication (liên tục)
[Secondary: ap-east-1]
    ├── SQS Queues (standby, empty)
    ├── Lambda Consumers (disabled)
    └── RDS Read Replica → promoted to primary khi failover
```

**Vấn đề: SQS không có native cross-region replication.**

**Giải pháp — Message Mirroring Pattern:**
```
[Producer]
    ↓
[Primary SQS Queue]
    ↓ (Lambda reads + writes)
[Lambda — Mirror Worker]
    ├── Ghi vào Primary SQS Queue Consumer
    └── Ghi vào Secondary SQS Queue (ap-east-1) — với delay nếu Primary healthy

→ Trong điều kiện bình thường: Secondary Queue nhận copy của tất cả messages
→ Khi Primary fail: Secondary Queue đã có messages sẵn
```

**Cách khác: EventBridge Global Endpoints (Endpoint Toàn Cầu EventBridge):**
```
[Producer]
    ↓
[EventBridge Global Endpoint]
    ├── Primary Bus (ap-southeast-1) → Targets
    └── Secondary Bus (ap-east-1) → Targets (standby)

→ Global Endpoint tự động failover khi primary health check fail
→ Route 53 health check + latency-based routing
```

#### Failover Process (< 5 phút)

```
Bước 1 (0–1 phút): Route 53 health check fail → DNS failover
Bước 2 (1–2 phút): Traffic chuyển sang secondary region
Bước 3 (2–3 phút): Secondary Lambda consumers được enable (EventBridge rule hoặc AWS Systems Manager)
Bước 4 (3–5 phút): RDS replica được promoted → secondary hoàn toàn operational
```

#### Zero Message Loss — Đảm Bảo RPO = 0

1. **Outbox Pattern**: Ghi event vào DB trước, DB đã có cross-region replication
2. **SQS message mirroring**: Mọi message đều được copy sang secondary ngay khi nhận
3. **DLQ cho cả hai region**: Message lỗi không bị mất dù có failover

```
Message lifecycle:
  Producer ghi → Primary DB + Outbox (replicated sang Secondary DB)
  CDC đọc Outbox → Primary SQS + Secondary SQS (mirror)
  
  Nếu Primary fail sau khi message đã vào Primary SQS nhưng chưa mirror:
  → Consumer ở Secondary đọc từ Secondary DB Outbox → gửi lại
  → Idempotency key đảm bảo không xử lý duplicate
```

---

## 📝 Điểm Đánh Giá Phỏng Vấn

Người phỏng vấn thường cho điểm dựa trên:

| Tiêu Chí | Điểm Cao | Điểm Thấp |
|---|---|---|
| **Yêu cầu hóa bài toán** | Hỏi clarifying questions, xác định constraints | Nhảy vào thiết kế ngay |
| **Chọn dịch vụ** | Giải thích tại sao chọn, tại sao không chọn cái khác | Chỉ nêu tên dịch vụ |
| **Trade-off** | Chủ động nêu nhược điểm của solution | Chỉ nêu điểm mạnh |
| **Edge cases** | Xử lý failure, retry, duplicate | Chỉ thiết kế happy path |
| **Scalability** | Tính toán số shard, queue depth, Lambda concurrency | Không đề cập scale |
| **Cost** | Ước tính chi phí, so sánh options | Không quan tâm cost |
| **Monitoring** | Nêu metrics quan trọng, alert strategy | Không đề cập observability |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
