# Top 25 Câu Hỏi Phỏng Vấn AWS Application Integration

> Bộ câu hỏi từ thực tế phỏng vấn tại các công ty sử dụng AWS — kèm gợi ý trả lời chi tiết

---

## 📌 Cách Sử Dụng Tài Liệu Này

1. Che phần **Gợi Ý Trả Lời** lại
2. Tự trả lời to bằng miệng trong 2–3 phút
3. Mở ra so sánh — chú ý những điểm bạn bỏ sót
4. Thực hành lại cho đến khi trả lời tự nhiên, không cần đọc

---

## 🔴 Phần 1: SQS — Simple Queue Service (Câu 1–7)

---

### Câu 1: SQS Standard Queue và FIFO Queue khác nhau thế nào? Khi nào dùng cái nào?

**Gợi Ý Trả Lời:**

**Standard Queue** (Hàng Đợi Tiêu Chuẩn):
- Throughput (thông lượng): Không giới hạn
- Thứ tự: Best-effort ordering — không đảm bảo
- Delivery: At-least-once — có thể nhận duplicate (trùng lặp)
- Dùng khi: throughput cao, consumer có thể xử lý idempotent (bất biến)

**FIFO Queue** (Hàng Đợi Vào Trước Ra Trước):
- Throughput: 300 TPS (transactions per second — giao dịch mỗi giây), tăng lên 3000 TPS với batching
- Thứ tự: Đảm bảo chính xác thứ tự trong một message group
- Delivery: Exactly-once — deduplication trong cửa sổ 5 phút
- Dùng khi: cần thứ tự (ví dụ: cập nhật số dư tài khoản), không được xử lý trùng

**Ví dụ thực tế:**
- Standard: Gửi email marketing cho 1 triệu user — không cần thứ tự
- FIFO: Xử lý giao dịch ngân hàng — debit rồi mới credit, đúng thứ tự

**Trade-off cần đề cập:** FIFO tốn hơn khoảng 50% và giới hạn throughput. Nếu không thực sự cần thứ tự, hãy dùng Standard và implement idempotency ở consumer.

---

### Câu 2: Dead Letter Queue (DLQ) là gì? Tại sao cần thiết trong production?

**Gợi Ý Trả Lời:**

**DLQ** (Hàng Đợi Thư Chết) là hàng đợi phụ nhận những message không thể xử lý thành công sau N lần thử.

**Cấu hình:**
```
maxReceiveCount: N lần thử (thường 3–5)
→ Sau N lần: message chuyển sang DLQ thay vì bị xóa
```

**Tại sao quan trọng trong production:**
1. **Không mất message** — message lỗi không bị xóa vĩnh viễn
2. **Cô lập lỗi** — message độc hại (poison message) không block toàn bộ queue
3. **Debug được** — DLQ là bằng chứng để điều tra nguyên nhân lỗi
4. **Alert được** — Monitor DLQ depth, cảnh báo khi có message lỗi

**Quy trình xử lý DLQ:**
```
1. Alert khi DLQ có message
2. Developer kiểm tra message body + lỗi
3. Sửa bug
4. Redrive (đẩy lại) message từ DLQ về queue gốc
```

**Poison message** (tin nhắn độc hại): Message khiến consumer crash mỗi lần xử lý. Không có DLQ, message này sẽ lặp mãi mãi và block các message sau.

---

### Câu 3: Visibility Timeout là gì? Cấu hình sai gây ra vấn đề gì?

**Gợi Ý Trả Lời:**

**Visibility Timeout** (Thời Gian Ẩn Tin Nhắn): Khoảng thời gian một message bị "ẩn" khỏi queue sau khi consumer nhận. Trong thời gian này, consumer khác không thấy message.

```
Consumer nhận message → Message ẩn trong X giây
→ Consumer xử lý xong → Delete message → Hoàn tất
→ Consumer crash / timeout → Message hiện lại → Consumer khác nhận
```

**Cấu hình sai gây vấn đề:**

1. **Visibility Timeout quá ngắn** (vấn đề phổ biến nhất):
   - Consumer chưa xử lý xong thì message đã hiện lại
   - Hai consumer xử lý cùng một message → duplicate processing (xử lý trùng)
   - **Giải pháp:** Đặt visibility timeout = 6× thời gian xử lý trung bình

2. **Visibility Timeout quá dài:**
   - Consumer crash → message mất nhiều giờ mới xử lý lại
   - **Giải pháp:** Consumer tự extend visibility timeout nếu đang xử lý lâu

**Rule of thumb (quy tắc ngón tay cái):** `visibility_timeout = 6 × average_processing_time`

---

### Câu 4: Long Polling vs Short Polling trong SQS khác nhau thế nào?

**Gợi Ý Trả Lời:**

**Short Polling** (Lấy Tin Ngắn):
- SQS query một subset (tập con) server ngẫu nhiên
- Trả lời ngay dù queue rỗng
- Vấn đề: Empty responses (phản hồi rỗng) → tốn tiền và CPU

**Long Polling** (Lấy Tin Chờ Dài):
- SQS chờ đến khi có message hoặc hết WaitTimeSeconds (tối đa 20 giây)
- Query tất cả server → không bỏ sót message
- **Lợi ích:** Giảm 90%+ empty responses → tiết kiệm chi phí + CPU

**Nên dùng Long Polling hầu hết trường hợp:**
```
WaitTimeSeconds = 20  # Giá trị tối đa và khuyến nghị
```

**Khi nào dùng Short Polling:** Khi cần latency cực thấp và chắc chắn queue luôn có message.

---

### Câu 5: Tại sao SQS at-least-once delivery yêu cầu consumer phải idempotent?

**Gợi Ý Trả Lời:**

**At-least-once delivery** (Giao Vận Ít Nhất Một Lần): SQS đảm bảo message được giao ít nhất một lần, nhưng có thể giao nhiều hơn một lần.

**Tại sao xảy ra duplicate:**
1. Consumer xử lý xong nhưng bị crash trước khi gọi `DeleteMessage`
2. Message được giao lại sau khi visibility timeout hết
3. Rare case: SQS infrastructure replicate message

**Idempotency** (Tính Bất Biến): Xử lý cùng một message nhiều lần cho kết quả giống như xử lý một lần.

**Cách implement idempotency:**
```
# Cách 1: Idempotency Key trong database
INSERT INTO processed_messages (message_id, processed_at)
VALUES (message_id, NOW())
ON CONFLICT (message_id) DO NOTHING  -- Bỏ qua nếu đã xử lý

# Cách 2: Conditional update
UPDATE orders SET status = 'SHIPPED'
WHERE order_id = ? AND status = 'PROCESSING'
-- Nếu đã SHIPPED, update không có effect
```

**Lưu ý quan trọng:** FIFO Queue với `MessageDeduplicationId` cũng xử lý duplicate ở tầng SQS trong 5 phút, nhưng consumer vẫn nên idempotent để an toàn hoàn toàn.

---

### Câu 6: Batch Processing với SQS hoạt động thế nào? Lợi ích là gì?

**Gợi Ý Trả Lời:**

SQS hỗ trợ nhận và xóa tối đa **10 message** trong một API call.

**Lợi ích:**
- Giảm số API call → giảm chi phí (SQS tính tiền per request)
- Tăng throughput xử lý
- Giảm network round trips (chuyến đi khứ hồi mạng)

**Lambda integration với SQS:**
```json
{
  "BatchSize": 10,
  "FunctionResponseTypes": ["ReportBatchItemFailures"]
}
```

**Partial Batch Failure** (Thất Bại Một Phần Của Lô):
- Khi một số message trong batch thất bại, Lambda có thể báo cáo từng message thất bại
- SQS chỉ retry message thất bại, không retry cả batch
- Dùng `ReportBatchItemFailures` để tránh xử lý lại message đã thành công

---

### Câu 7: Thiết kế Fan-out Pattern với SNS + SQS như thế nào?

**Gợi Ý Trả Lời:**

**Fan-out Pattern** (Mẫu Khuếch Tán): Một message → nhiều hệ thống xử lý độc lập.

**Kiến trúc:**
```
Producer
    ↓
SNS Topic
    ├── SQS Queue A (Email Service)
    ├── SQS Queue B (Inventory Service)
    └── SQS Queue C (Analytics Service)
```

**Tại sao không dùng SNS trực tiếp → Lambda:**
- SNS + SQS: Message được buffer trong SQS → không mất nếu Lambda crash
- SNS → Lambda: Không có buffer → Lambda phải available ngay lập tức

**Lợi ích Fan-out Pattern:**
1. **Decoupling** (Tách Rời): Các service không phụ thuộc nhau
2. **Independent scaling** (Scale Độc Lập): Mỗi SQS → Lambda scale riêng
3. **Fault isolation** (Cô Lập Lỗi): Email service fail không ảnh hưởng Inventory
4. **Easy to add** (Dễ Bổ Sung): Thêm subscriber mới không cần thay đổi producer

**Bước triển khai:**
```
1. Tạo SNS Topic
2. Tạo SQS Queue cho mỗi consumer
3. Subscribe SQS Queue vào SNS Topic
4. Cấu hình SQS Queue Policy để SNS có quyền gửi
5. Lambda đọc từ từng SQS Queue
```

---

## 🟡 Phần 2: EventBridge & Step Functions (Câu 8–14)

---

### Câu 8: EventBridge khác SNS ở điểm nào quan trọng nhất?

**Gợi Ý Trả Lời:**

Sự khác biệt quan trọng nhất: **EventBridge có Schema Registry và content-based filtering**.

| Khía Cạnh | SNS | EventBridge |
|---|---|---|
| Filtering | Theo message attributes | Theo bất kỳ field nào trong event body |
| Schema | Không có | Schema Registry tự động discover |
| SaaS partners | Không | 90+ partner integrations (Salesforce, Zendesk...) |
| Archive & Replay | Không | Có |
| Routing phức tạp | Hạn chế | Nhiều rule, nhiều target per rule |

**Khi nào dùng EventBridge thay SNS:**
1. Cần filter theo nội dung event (ví dụ: chỉ route event khi `order.status = "CANCELLED"`)
2. Tích hợp với SaaS application
3. Cần replay event lịch sử
4. Cần Schema Registry để document event contracts

**Khi nào SNS vẫn tốt hơn:**
- Fan-out đơn giản không cần filtering phức tạp
- Cần latency thấp hơn (SNS nhanh hơn EventBridge)
- Chi phí thấp hơn

---

### Câu 9: Giải thích cách EventBridge Rules và Event Patterns hoạt động?

**Gợi Ý Trả Lời:**

**Event Rule** (Quy Tắc Sự Kiện): Định nghĩa ĐIỀU KIỆN để route event đến target.

**Event Pattern** (Mẫu Sự Kiện): JSON filter mô tả event nào khớp với rule.

```json
// Rule: Chỉ route khi order bị cancel
{
  "source": ["com.myapp.orders"],
  "detail-type": ["Order State Change"],
  "detail": {
    "status": ["CANCELLED"],
    "amount": [{ "numeric": [">", 1000] }]
  }
}
```

**Các loại pattern:**
- **Exact match** (Khớp Chính Xác): `"status": ["CANCELLED"]`
- **Prefix match** (Khớp Tiền Tố): `"source": [{ "prefix": "com.myapp" }]`
- **Numeric range** (Phạm Vi Số): `"amount": [{ "numeric": [">", 100, "<=", 1000] }]`
- **Exists** (Tồn Tại): `"errorCode": [{ "exists": true }]`
- **Anything-but** (Trừ Ra): `"status": [{ "anything-but": "ACTIVE" }]`

**Scheduled Rules** (Quy Tắc Lịch Trình): Dùng cron hoặc rate expression để trigger định kỳ:
```
rate(5 minutes)     # Mỗi 5 phút
cron(0 12 * * ? *)  # Mỗi ngày lúc 12:00 UTC
```

---

### Câu 10: Archive và Replay trong EventBridge giải quyết vấn đề gì?

**Gợi Ý Trả Lời:**

**Vấn đề:** Trong event-driven architecture, khi deploy service mới hoặc khi có bug, bạn cần xử lý lại các event đã xảy ra trong quá khứ.

**Archive** (Lưu Trữ Sự Kiện): EventBridge lưu lại tất cả event vào một archive theo rule.

```
EventBridge Bus → Archive Rule → Lưu event vào S3-backed storage
Retention: 1 ngày – vĩnh viễn (tùy cấu hình)
```

**Replay** (Phát Lại): Gửi lại event từ archive vào event bus, với time range tùy chọn.

**Use cases:**
1. **Bug fix replay**: Phát hiện bug trong consumer → sửa → replay event bị lỗi
2. **New service onboarding**: Service mới cần xử lý tất cả event từ 3 tháng trước
3. **Testing**: Test consumer với event thật từ production
4. **Audit**: Kiểm tra lại chuỗi sự kiện đã xảy ra

**Lưu ý chi phí:** Archive tính tiền theo GB lưu trữ. Replay tính tiền theo event được phát lại.

---

### Câu 11: Standard Workflow vs Express Workflow trong Step Functions — chọn khi nào?

**Gợi Ý Trả Lời:**

| | Standard Workflow | Express Workflow |
|---|---|---|
| **Duration** | Tối đa 1 năm | Tối đa 5 phút |
| **Execution model** | At-most-once (tối đa một lần) | At-least-once (ít nhất một lần) |
| **Execution history** | Có — lưu toàn bộ | Chỉ trong CloudWatch Logs |
| **Cost** | Per state transition | Per execution duration + requests |
| **Throughput** | 2,000 executions/giây | 100,000 executions/giây |
| **Use case** | Long-running, audit trail | High-volume, short-lived |

**Chọn Standard khi:**
- Workflow chạy lâu (đặt hàng → giao hàng → thanh toán → nhiều ngày)
- Cần audit trail đầy đủ (tài chính, y tế, pháp lý)
- Không được thực thi lại (exactly-once semantics quan trọng)
- Ví dụ: Order fulfillment, loan approval workflow

**Chọn Express khi:**
- Volume cao, thời gian ngắn (IoT event processing)
- Chi phí là ưu tiên (Express rẻ hơn nhiều với high volume)
- Ví dụ: ETL pipeline, IoT data processing, streaming analytics

---

### Câu 12: Callback Pattern với waitForTaskToken trong Step Functions dùng khi nào?

**Gợi Ý Trả Lời:**

**Vấn đề:** Step Functions cần chờ một hành động bên ngoài hoàn tất (ví dụ: con người phê duyệt, job chạy lâu trong hệ thống khác).

**Callback Pattern** (Mẫu Gọi Lại) với `waitForTaskToken`:
```json
// State definition
{
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs...",
    "MessageBody": {
      "TaskToken.$": "$$.Task.Token",
      "OrderId.$": "$.orderId"
    }
  }
}
```

```
Step Functions → Gửi TaskToken vào SQS
                 ↓
           Worker / Human xử lý
                 ↓
           Gọi SendTaskSuccess/SendTaskFailure với TaskToken
                 ↓
           Step Functions tiếp tục
```

**Use cases:**
- Human approval workflow (yêu cầu phê duyệt của con người)
- Chờ external system (hệ thống bên ngoài) hoàn tất xử lý
- Chờ manual review trong compliance workflow (quy trình tuân thủ)

**Lưu ý:** TaskToken có TTL (time-to-live — thời gian sống) tối đa bằng execution timeout. Cần heartbeat nếu task chạy lâu.

---

### Câu 13: Giải thích Saga Pattern và cách implement với Step Functions?

**Gợi Ý Trả Lời:**

**Saga Pattern** (Mẫu Saga): Quản lý distributed transaction (giao dịch phân tán) bằng cách chia thành nhiều bước, mỗi bước có compensating transaction (giao dịch bù trừ) nếu thất bại.

**Ví dụ Order Flow:**
```
Đặt hàng → Trừ inventory → Charge thẻ → Giao hàng
```

**Nếu "Giao hàng" thất bại → Compensation:**
```
Rollback: Hoàn tiền thẻ → Cộng lại inventory → Hủy đơn hàng
```

**Implement với Step Functions:**
```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Next": "ChargePayment",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "CancelOrder" }]
    },
    "ChargePayment": {
      "Type": "Task",
      "Next": "CreateShipment",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory" }]
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Comment": "Compensating transaction",
      "Next": "CancelOrder"
    }
  }
}
```

**Choreography vs Orchestration:**
- **Orchestration** (Điều Phối) với Step Functions: Trung tâm điều phối, dễ debug, visible
- **Choreography** (Vũ Đạo) với EventBridge: Không trung tâm, loosely coupled, khó trace

**Khi nào dùng Orchestration:** Workflow phức tạp, nhiều bước, cần audit trail — **Step Functions là lựa chọn tốt nhất**.

---

### Câu 14: EventBridge Pipes giải quyết vấn đề gì?

**Gợi Ý Trả Lời:**

**EventBridge Pipes** (Đường Ống Sự Kiện): Kết nối source và target một-một, với optional filtering và enrichment, không cần viết code glue (kết nối).

**Không có Pipes:**
```
SQS Queue → Lambda (chỉ để forward event) → EventBridge
```

**Với Pipes:**
```
SQS Queue → [Filter] → [Enrichment] → EventBridge Target
          (tất cả managed, không cần Lambda)
```

**Các thành phần:**
- **Source** (Nguồn): SQS, Kinesis, DynamoDB Streams, Kafka
- **Filter** (Lọc): Lọc event trước khi xử lý (giảm chi phí)
- **Enrichment** (Làm Giàu): Lambda, Step Functions, API GW để transform data
- **Target** (Đích): EventBridge Bus, SQS, SNS, Lambda, Step Functions

**Use case:** Đọc từ DynamoDB Streams → filter chỉ INSERT events → enrich với data từ API → gửi vào EventBridge.

---

## 🟠 Phần 3: Kinesis (Câu 15–18)

---

### Câu 15: Kinesis Shard là gì? Cách tính số shard cần thiết?

**Gợi Ý Trả Lời:**

**Shard** (Mảnh): Đơn vị năng lực throughput (thông lượng) trong Kinesis Data Streams. Mỗi shard cung cấp:
- **Write:** 1 MB/s hoặc 1,000 records/giây
- **Read (standard):** 2 MB/s chia sẻ cho tất cả consumer
- **Read (Enhanced Fan-Out):** 2 MB/s riêng cho mỗi consumer

**Công thức tính số shard:**
```
Số shard = max(
  incoming_bytes_per_second / 1MB,
  incoming_records_per_second / 1000,
  outgoing_bytes_per_second / 2MB (chia cho số consumer)
)

Luôn làm tròn lên và thêm 20–25% dự phòng
```

**Ví dụ:**
```
Dữ liệu vào: 5 MB/s
→ Cần ít nhất 5 shard (5MB / 1MB per shard)

Bản ghi vào: 3,000 records/giây, mỗi bản ghi 100 bytes = 0.3 MB/s
→ Cần 3 shard (3000 / 1000 per shard)
→ Bottleneck là records/giây → dùng 3 shard

Dữ liệu ra: 8 MB/s với 2 consumer
→ Mỗi consumer đọc 8MB/s → cần 4 shard (8MB / 2MB per shard per consumer)
→ Nếu dùng Enhanced Fan-Out: mỗi shard cung cấp 2MB/s riêng cho từng consumer
```

**Hot shard** (Mảnh Nóng): Khi nhiều bản ghi cùng partition key đổ vào một shard, tạo bottleneck. Giải pháp: Chọn partition key có cardinality cao (nhiều giá trị khác nhau).

---

### Câu 16: Enhanced Fan-Out khác gì so với standard consumer trong Kinesis?

**Gợi Ý Trả Lời:**

**Standard Consumer** (Consumer Tiêu Chuẩn):
- 2 MB/s per shard, **chia sẻ** cho tất cả consumer
- 5 consumer → mỗi consumer chỉ còn 400 KB/s per shard
- Pull-based với `GetRecords` API
- Latency: 200ms+

**Enhanced Fan-Out** (Khuếch Tán Nâng Cao):
- 2 MB/s per shard **riêng** cho mỗi consumer — không chia sẻ
- Push-based với `SubscribeToShard` API
- Latency: 70ms (thấp hơn nhiều)
- Tốn thêm chi phí per consumer-shard-hour

**Khi nào dùng Enhanced Fan-Out:**
- Có nhiều hơn 2–3 consumer đọc cùng stream
- Cần latency thấp (< 200ms)
- Consumer không thể chấp nhận bandwidth bị chia sẻ

---

### Câu 17: Iterator Age metric trong Kinesis là gì? Tại sao quan trọng?

**Gợi Ý Trả Lời:**

**Iterator Age** (Tuổi Trình Lặp): Khoảng cách thời gian giữa khi record được ghi vào Kinesis và khi consumer đọc được record đó.

```
Iterator Age = Thời Điểm Hiện Tại - Thời Điểm Record Được Ghi Vào
```

**Ý nghĩa:**
- Iterator Age thấp (~0ms) → Consumer đang xử lý gần real-time
- Iterator Age cao (hàng phút/giờ) → Consumer đang bị lag — không theo kịp

**Tại sao quan trọng:**
- **Latency indicator** (Chỉ Số Độ Trễ): Đo độ trễ end-to-end thực tế
- **Scaling trigger** (Kích Hoạt Scale): Iterator Age tăng → cần thêm shard hoặc consumer
- **Alert metric** (Số Liệu Cảnh Báo): Alert khi Iterator Age > ngưỡng (ví dụ: > 30 giây cho realtime analytics)

**CloudWatch Alarm:**
```
Metric: GetRecords.IteratorAgeMilliseconds
Condition: Average > 30000 (30 giây)
→ Alert: Consumer không xử lý kịp, cần scale
```

---

### Câu 18: Khi nào dùng Kinesis Firehose thay vì Kinesis Data Streams?

**Gợi Ý Trả Lời:**

**Dùng Kinesis Firehose khi:**
- Đích đến là S3, Redshift, OpenSearch, Splunk — Firehose tích hợp sẵn
- Không cần consumer tự viết — Firehose hoàn toàn managed
- Chấp nhận near-real-time (60–900 giây buffer) thay vì milliseconds
- Cần data transformation đơn giản (Firehose gọi Lambda để transform)

**Dùng Kinesis Data Streams khi:**
- Cần xử lý real-time thực sự (milliseconds)
- Cần nhiều consumer đọc cùng stream với logic khác nhau
- Cần replay capability (phát lại dữ liệu)
- Cần custom routing phức tạp

**Ví dụ quyết định:**
```
Log server → phân tích bất thường real-time + lưu vào S3:

Solution A (Firehose): Log → Firehose → S3 (đơn giản, nhưng delay 60s)
Solution B (KDS + Firehose):
  Log → Kinesis Data Streams → Lambda (real-time anomaly detection)
                              → Kinesis Firehose → S3 (lưu trữ)
```

---

## 🔵 Phần 4: Thiết Kế Hệ Thống & Pattern (Câu 19–25)

---

### Câu 19: Thiết kế hệ thống Order Processing sử dụng AWS messaging services?

**Gợi Ý Trả Lời:**

**Yêu cầu:** Hệ thống xử lý đơn hàng, đảm bảo không mất đơn, thông báo cho nhiều hệ thống.

**Kiến Trúc:**
```
[API Gateway]
      ↓
[Order Service Lambda]
      ↓ (ghi DB trước — Outbox Pattern)
[SQS FIFO — order-created]  ← đảm bảo thứ tự per customer
      ↓
[Order Processor Lambda]
      ↓ (nếu cần nhiều bước)
[Step Functions — Order Workflow]
  ├── Reserve Inventory (Lambda)
  ├── Charge Payment (Lambda)
  └── Create Shipment (Lambda)
      ↓ (khi hoàn tất)
[SNS — order-completed]
  ├── SQS → Email Service
  ├── SQS → Analytics Service
  └── SQS → Notification Service
```

**Giải thích quyết định:**
- **SQS FIFO**: Đảm bảo đơn hàng của cùng customer được xử lý đúng thứ tự
- **Step Functions**: Saga pattern, dễ debug, compensation khi payment fail
- **SNS → SQS Fan-out**: Thông báo nhiều hệ thống, mỗi hệ thống xử lý độc lập
- **Outbox Pattern**: Đảm bảo ghi DB và gửi message nhất quán (không mất message khi crash)

---

### Câu 20: Idempotency trong distributed system — implement thế nào?

**Gợi Ý Trả Lời:**

**Idempotency** (Tính Bất Biến): Thực hiện cùng một operation nhiều lần → kết quả giống như thực hiện một lần.

**Tầm quan trọng:** Mọi consumer của SQS, SNS, Kinesis đều có thể nhận duplicate message → phải idempotent.

**Cách implement:**

**1. Idempotency Key với Database:**
```sql
-- Bảng tracking message đã xử lý
CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  result JSONB,
  created_at TIMESTAMP,
  expires_at TIMESTAMP  -- TTL để không tốn storage mãi
);

-- Xử lý message
BEGIN TRANSACTION;
  INSERT INTO idempotency_keys (key, result, expires_at)
  VALUES (message_id, NULL, NOW() + INTERVAL '24 hours')
  ON CONFLICT (key) DO NOTHING;
  
  IF FOUND THEN
    -- Xử lý lần đầu tiên
    -- ... business logic ...
    UPDATE idempotency_keys SET result = response WHERE key = message_id;
  END IF;
COMMIT;
```

**2. Conditional Update:**
```sql
-- Chỉ update nếu state phù hợp
UPDATE orders SET status = 'PAID', updated_at = NOW()
WHERE order_id = ? AND status = 'PENDING';
-- Nếu đã PAID, câu lệnh không có tác dụng → safe
```

**3. Natural Idempotency:**
```python
# Upsert thay vì Insert
# S3 PutObject — ghi lại cùng file không tạo duplicate
# DynamoDB PutItem — thay thế item hiện tại
```

**Lưu ý:** Chọn Idempotency Key cẩn thận — nên là business ID (order_id, payment_id), không phải SQS message ID (vì message ID thay đổi khi redrive từ DLQ).

---

### Câu 21: Circuit Breaker Pattern — khi nào cần thiết trong messaging?

**Gợi Ý Trả Lời:**

**Circuit Breaker** (Cầu Dao Ngắt Lỗi): Pattern bảo vệ hệ thống khỏi cascade failure (lỗi dây chuyền) khi downstream service (dịch vụ phía sau) không available.

**Ba trạng thái:**
```
CLOSED (Đóng) → Bình thường, request đi qua
    ↓ (nhiều lỗi liên tiếp)
OPEN (Mở) → Chặn request, trả lỗi ngay lập tức
    ↓ (sau timeout)
HALF-OPEN (Nửa Mở) → Cho thử một request
    ↓ Thành công        ↓ Thất bại
CLOSED               OPEN lại
```

**Trong context messaging:**
- Lambda consumer gọi downstream service (ví dụ: payment API)
- Nếu payment API liên tục timeout → Lambda cũng timeout → visibility timeout hết → message back vào queue → xử lý lại → DLQ
- Với Circuit Breaker: Phát hiện sớm, fail fast (thất bại nhanh), message về DLQ chờ service recover

**Implement trên AWS:**
- `AWS SDK retry/backoff` tích hợp sẵn trong SDK
- AWS AppConfig cho dynamic circuit breaker config
- Hoặc dùng library: `resilience4j` (Java), `pybreaker` (Python)

---

### Câu 22: Outbox Pattern giải quyết vấn đề gì trong messaging?

**Gợi Ý Trả Lời:**

**Vấn đề:** Làm thế nào đảm bảo cả ghi database VÀ gửi message đều thành công, hoặc cả hai đều thất bại?

**Vấn đề không có Outbox:**
```
Ghi DB thành công ✅
Gửi SQS message ❌ (network fail, SQS down)
→ DB và message không nhất quán
```

**Outbox Pattern** (Mẫu Hộp Thư Đi):
```
1. Ghi DB + ghi vào bảng outbox trong CÙNG một transaction
2. Outbox Poller (hoặc CDC — Change Data Capture) đọc bảng outbox
3. Gửi message lên SQS/SNS
4. Đánh dấu outbox record là đã gửi

→ Nếu gửi thất bại: Retry — message chưa bị đánh dấu, sẽ được gửi lại
→ Nếu crash sau khi gửi nhưng trước khi đánh dấu: Gửi lại → duplicate → consumer phải idempotent
```

**Implement:**
```sql
-- Ghi cùng một transaction
BEGIN;
  INSERT INTO orders (id, status) VALUES (?, 'CREATED');
  INSERT INTO outbox (id, event_type, payload, status)
    VALUES (uuid(), 'ORDER_CREATED', '{"orderId": ?}', 'PENDING');
COMMIT;
```

**Công cụ cho CDC:** AWS DMS (Database Migration Service), Debezium, PostgreSQL logical replication → EventBridge Pipes → SQS/SNS.

---

### Câu 23: Làm thế nào thiết kế kiến trúc messaging có khả năng xử lý 1 triệu message/ngày?

**Gợi Ý Trả Lời:**

**1 triệu message/ngày ≈ 11.5 message/giây** (peak có thể cao hơn 3–5× = 35–57 msg/s)

**SQS Standard Queue — hoàn toàn xử lý được:**
```
Producer → SQS Standard Queue → Lambda (auto-scale)
         (unlimited throughput)  (scale đến 1000 concurrent)
```

**Nếu cần fan-out (nhiều consumer):**
```
SNS Topic → SQS Queue A (Processing)    → Lambda
          → SQS Queue B (Notification)  → Lambda
          → SQS Queue C (Analytics)     → Lambda Async
```

**Tối ưu chi phí tại 1 triệu message/ngày:**
- SQS: 1 triệu request = ~$0.40/ngày (rất rẻ)
- Dùng batch size 10 để giảm Lambda invocations
- Dùng Long Polling (waitTimeSeconds=20) để giảm empty receives
- Nếu dùng Kinesis: Cần ít nhất 1 shard = $0.015/giờ = $10.8/tháng → đắt hơn cho volume này

**Bottleneck cần tránh:**
- Không dùng FIFO nếu không cần thứ tự — giới hạn 300 TPS
- Database là thường là bottleneck thực sự — dùng connection pooling, write batching
- Lambda concurrency limit — monitor và request tăng nếu cần

---

### Câu 24: Giải thích Event Sourcing và CQRS — khi nào áp dụng?

**Gợi Ý Trả Lời:**

**Event Sourcing** (Nguồn Sự Kiện): Thay vì lưu trạng thái hiện tại, lưu toàn bộ chuỗi sự kiện đã xảy ra. Trạng thái hiện tại = replay toàn bộ sự kiện.

```
Thông thường:   orders table: {id: 1, status: "SHIPPED", amount: 100}

Event Sourcing: events table:
  {OrderCreated, amount: 100}
  {InventoryReserved, items: [...]}
  {PaymentCharged, amount: 100}
  {OrderShipped, tracking: "XYZ"}
```

**CQRS** — Command Query Responsibility Segregation (Phân Tách Trách Nhiệm Lệnh và Truy Vấn):
- **Command** (Lệnh): Ghi dữ liệu — gửi event vào Event Store
- **Query** (Truy Vấn): Đọc dữ liệu — từ Read Model được optimize riêng

```
[Write Path]: API → Command Handler → Event Store (append-only)
                                           ↓ (async)
[Read Path]:                    Event Handler → Read Model (DynamoDB, OpenSearch)
             API → Query Handler ←──────────────────────┘
```

**Khi nào áp dụng:**
- Cần audit trail đầy đủ (tài chính, y tế, pháp lý)
- Cần replay lại lịch sử để rebuild state
- Read/write có scale và consistency requirement khác nhau
- Complex domain với nhiều aggregate

**Khi KHÔNG nên áp dụng:**
- CRUD đơn giản — overengineering nghiêm trọng
- Team nhỏ, ít kinh nghiệm với pattern này
- Không có yêu cầu audit trail

---

### Câu 25: Làm thế nào monitor và debug một distributed messaging system?

**Gợi Ý Trả Lời:**

**Tầng 1: Metrics (Số Liệu)**

| Dịch Vụ | Metric Quan Trọng Nhất | Cảnh Báo Khi |
|---|---|---|
| SQS | ApproximateNumberOfMessagesVisible | > ngưỡng theo business SLA |
| SQS | ApproximateAgeOfOldestMessage | > max processing time |
| SQS DLQ | ApproximateNumberOfMessagesVisible | > 0 (alert ngay) |
| Kinesis | GetRecords.IteratorAgeMilliseconds | > 30 giây |
| Step Functions | ExecutionsFailed | > 0 |
| Lambda | Errors + Throttles | > 1% |

**Tầng 2: Distributed Tracing với AWS X-Ray**

```python
# Truyền trace context qua message
import aws_xray_sdk

@xray_recorder.capture('process_order')
def process_order(message):
    # X-Ray tự động trace Lambda invocation
    # Tạo subsegment cho mỗi downstream call
    with xray_recorder.in_subsegment('database_write') as subsegment:
        # write to DB
```

**Tầng 3: Correlation ID (ID Tương Quan)**

```python
# Producer thêm correlation_id vào message
message = {
    "correlation_id": str(uuid4()),  # Truyền qua toàn bộ chain
    "order_id": order_id,
    "payload": data
}

# Consumer log với correlation_id
logger.info("Processing message", extra={
    "correlation_id": message["correlation_id"],
    "service": "order-processor"
})
```

**Tầng 4: CloudWatch Logs Insights**

```
# Query tìm toàn bộ log của một correlation_id
fields @timestamp, @message
| filter correlationId = "abc-123"
| sort @timestamp asc
```

**Tầng 5: DLQ như "canary in the coal mine" (chim hoàng yến trong mỏ than)**

```
DLQ > 0 messages → Có gì đó sai → Alert → Investigate → Fix → Redrive
```

---

## 📊 Tóm Tắt Câu Trả Lời Cần Nhớ

| Câu Hỏi | Từ Khóa Cần Đề Cập |
|---|---|
| SQS Standard vs FIFO | Throughput, ordering, exactly-once, cost |
| DLQ | Poison message, maxReceiveCount, redrive, alert |
| Visibility Timeout | Pull, hide, delete, duplicate if too short |
| Long Polling | Empty responses, waitTimeSeconds=20, cost |
| Fan-out Pattern | SNS → SQS, decoupling, independent scaling |
| EventBridge vs SNS | Content filtering, Schema Registry, Archive & Replay |
| Saga Pattern | Compensating transaction, orchestration vs choreography |
| Kinesis Shard | 1MB/s write, 2MB/s read, partition key, hot shard |
| Iterator Age | Lag indicator, scale trigger, near real-time |
| Idempotency | Idempotency key, conditional update, at-least-once |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
