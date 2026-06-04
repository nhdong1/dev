# Bảng So Sánh Dịch Vụ AWS Integration Đầy Đủ

> Tài liệu tham khảo nhanh — học thuộc bảng này trước phỏng vấn

---

## 🔑 Bảng So Sánh Cốt Lõi

### SQS vs SNS vs EventBridge vs Kinesis

| Tiêu Chí | SQS | SNS | EventBridge | Kinesis |
|---|---|---|---|---|
| **Mô hình** | Queue (Hàng Đợi) | Pub/Sub (Phát/Nhận) | Event Bus (Xe Buýt Sự Kiện) | Streaming (Luồng Dữ Liệu) |
| **Giao vận** | Pull-based (Kéo) | Push-based (Đẩy) | Push-based (Đẩy) | Pull-based (Kéo) |
| **Số consumer** | Một consumer group | Nhiều subscriber | Nhiều target | Nhiều consumer độc lập |
| **Thứ tự** | FIFO có thứ tự, Standard không | Không đảm bảo | Không đảm bảo | Có thứ tự trong shard |
| **Lưu trữ** | Tối đa 14 ngày | Không lưu (fire-and-forget) | Không lưu (có Archive tùy chọn) | 1–365 ngày |
| **Phát lại** | Không | Không | Có (Archive & Replay) | Có |
| **Throughput** | Unlimited (Standard) / 300 TPS (FIFO) | Unlimited | Không có giới hạn rõ | 1MB/s hoặc 1000 record/s per shard |
| **Giá** | Per request | Per notification | Per event | Per shard-hour + PUT payload |
| **Điểm mạnh** | Decoupling, durability | Fan-out nhanh | Event routing phức tạp | Real-time, replay |
| **Use case** | Task queue, background job | Alerts, fan-out | Microservices event-driven | Log ingestion, analytics |

---

## 📊 So Sánh Chi Tiết Từng Cặp

### SQS vs SNS

| | SQS | SNS |
|---|---|---|
| **Hướng dữ liệu** | Point-to-point (Điểm đến Điểm) | Broadcast (Phát Rộng) |
| **Consumer** | Pull — consumer tự lấy khi sẵn sàng | Push — tự đẩy đến subscriber ngay |
| **Message persistence** | Lưu đến khi consumer xử lý xong | Không lưu — nếu subscriber fail thì mất |
| **Retry** | Message ở lại queue, thử lại sau | Không retry theo mặc định |
| **Dead Letter Queue** | Có hỗ trợ đầy đủ | Có (subscription-level) |
| **Filtering** | Không filter (nhận hết) | Có message filtering theo attributes |
| **Ordering** | Standard: Không. FIFO: Có | Không đảm bảo |
| **Max message size** | 256 KB | 256 KB |
| **Visibility timeout** | Có (15 phút đến 12 giờ) | Không áp dụng |
| **Long polling** | Có (giảm chi phí polling) | Không áp dụng |

**Quy tắc chọn:**
- Dùng **SQS** khi: cần task queue, worker pool, load leveling (san phẳng tải), chịu được delay
- Dùng **SNS** khi: cần thông báo ngay, nhiều hệ thống cần nhận cùng lúc, push notifications

---

### SNS vs EventBridge

| | SNS | EventBridge |
|---|---|---|
| **Phức tạp routing** | Đơn giản (filter by attributes) | Phức tạp (filter by pattern, content-based) |
| **Nguồn sự kiện** | Chỉ từ AWS services và custom | AWS services, SaaS partners, custom |
| **Schema Registry** | Không | Có — quản lý cấu trúc event |
| **Archive & Replay** | Không | Có — phát lại sự kiện lịch sử |
| **Cross-account** | Hỗ trợ (topic policy) | Hỗ trợ (event bus policy) |
| **Latency** | Thấp hơn | Cao hơn một chút (~500ms) |
| **Transform** | Không | Có (input transformer) |
| **Target** | SQS, Lambda, HTTP, SMS, Email | SQS, SNS, Lambda, Step Functions, API GW, Kinesis... |
| **Cost** | Rẻ hơn (per notification) | Đắt hơn (per event) |
| **Debugging** | Khó hơn | Tốt hơn (CloudWatch Logs, Schema Registry) |

**Quy tắc chọn:**
- Dùng **SNS** khi: cần fan-out đơn giản, latency quan trọng, chi phí thấp
- Dùng **EventBridge** khi: cần event routing phức tạp, tích hợp SaaS, cần Schema Registry, Archive & Replay

---

### Kinesis Data Streams vs SQS

| | Kinesis Data Streams (KDS) | SQS |
|---|---|---|
| **Mô hình** | Streaming — nhiều consumer đọc song song | Queue — một consumer group |
| **Thứ tự** | Có trong shard (partition key) | Chỉ với FIFO Queue |
| **Phát lại** | Có — đọc lại từ bất kỳ offset nào | Không — message bị xóa sau khi xử lý |
| **Lưu trữ** | 24 giờ – 365 ngày | 1 giây – 14 ngày |
| **Throughput** | 1MB/s write, 2MB/s read per shard | Unlimited (Standard) |
| **Consumer** | Nhiều consumer đọc cùng stream độc lập | Một consumer group, message chỉ tới một consumer |
| **Delay xử lý** | Milliseconds | Milliseconds – Seconds |
| **Quản lý** | Phải quản lý shard | Serverless hoàn toàn |
| **Cost model** | Per shard-hour (tốn hơn) | Per request (rẻ hơn khi ít message) |
| **Use case** | Analytics, log ingestion, IoT | Background jobs, task queue |

**Quy tắc chọn:**
- Dùng **Kinesis** khi: nhiều consumer cần đọc cùng data, cần replay, real-time analytics, ordering trong shard
- Dùng **SQS** khi: task queue đơn giản, serverless hoàn toàn, không cần replay

---

### Kinesis Data Streams vs Kinesis Firehose

| | Kinesis Data Streams (KDS) | Kinesis Data Firehose |
|---|---|---|
| **Kiểm soát** | Tự viết consumer | Fully managed — không cần consumer |
| **Latency** | Milliseconds | Near real-time (60–900 giây) |
| **Destination** | Bất kỳ (consumer tự xử lý) | S3, Redshift, OpenSearch, Splunk |
| **Transform** | Tự code trong consumer | Có (Lambda transformation) |
| **Replay** | Có | Không |
| **Quản lý shard** | Phải tự quản lý | Không cần |
| **Use case** | Custom analytics, real-time processing | Batch delivery to data lake / data warehouse |

---

### Step Functions vs Lambda Chaining

| | Step Functions | Lambda Chaining (Xích Lambda) |
|---|---|---|
| **Visibility** | Visual workflow trên console | Không có — logic ẩn trong code |
| **Error handling** | Catch + Retry tích hợp sẵn | Tự code try/catch |
| **State management** | Tự động | Tự truyền state qua event |
| **Long-running** | Tối đa 1 năm (Standard) | Tối đa 15 phút |
| **Parallel** | Parallel State tích hợp | Gọi nhiều Lambda từ orchestrator |
| **Debugging** | Execution history đầy đủ | CloudWatch Logs rời rạc |
| **Cost** | Tính tiền per state transition | Tính tiền per invocation + duration |
| **Phức tạp** | Phù hợp workflow nhiều bước | Phù hợp workflow đơn giản |

**Quy tắc chọn:**
- Dùng **Step Functions** khi: workflow nhiều bước, cần human approval, cần retry phức tạp, long-running
- Dùng **Lambda Chaining** khi: pipeline đơn giản 2–3 bước, chi phí quan trọng

---

### SQS vs Amazon MQ

| | SQS | Amazon MQ |
|---|---|---|
| **Protocol** | AWS proprietary (API) | AMQP, MQTT, STOMP, OpenWire (chuẩn mở) |
| **Migration** | Cần rewrite code | Lift-and-shift từ on-premises |
| **Scale** | Unlimited, fully serverless | Giới hạn bởi broker instance size |
| **Management** | Không cần quản lý | Cần quản lý broker, HA setup |
| **Advanced features** | Hạn chế | Topics, queues, routing phức tạp hơn |
| **Cost** | Rẻ hơn, pay-per-use | Tốn hơn (broker instance + storage) |
| **Use case** | Greenfield (dự án mới) | Legacy migration từ ActiveMQ/RabbitMQ |

**Quy tắc chọn:**
- Dùng **SQS** cho mọi dự án mới — đơn giản, rẻ, serverless
- Dùng **Amazon MQ** chỉ khi đang migrate hệ thống on-premises dùng ActiveMQ hoặc RabbitMQ

---

## 🎯 Quyết Định Theo Use Case

### Khi Nào Dùng Gì — Bảng Quyết Định Nhanh

| Tình Huống | Dịch Vụ Khuyến Nghị | Lý Do |
|---|---|---|
| Xử lý đơn hàng bất đồng bộ | SQS | Task queue, consumer xử lý từng đơn |
| Gửi thông báo khi có sự kiện | SNS | Fan-out đến Email + SMS + Lambda |
| Đặt hàng → thông báo + inventory + analytics cùng lúc | SNS → SQS (Fan-out) | Broadcast rồi queue từng luồng |
| Tích hợp với Salesforce / Zendesk | EventBridge | Partner event bus sẵn có |
| Xử lý log server, clickstream | Kinesis Data Streams | High throughput, replay, real-time |
| Đẩy log vào S3 hàng giờ | Kinesis Firehose | Near-real-time delivery, không cần viết consumer |
| Saga Pattern cho order → payment → shipping | Step Functions | Orchestration, error handling, compensation |
| Migrate từ RabbitMQ on-premises | Amazon MQ | Tương thích protocol AMQP |
| GraphQL API với realtime subscription | AWS AppSync | Managed GraphQL, WebSocket tích hợp |
| Audit trail — cần replay event lịch sử | EventBridge Archive hoặc Kinesis | Chỉ hai dịch vụ này hỗ trợ replay |

---

## 📐 Giới Hạn Quan Trọng (Service Limits)

### Amazon SQS

| Giới Hạn | Giá Trị |
|---|---|
| Message size tối đa | 256 KB (có thể dùng S3 pointer cho lớn hơn) |
| Message retention tối đa | 14 ngày |
| Visibility timeout tối đa | 12 giờ |
| Batch size tối đa | 10 messages |
| Standard Queue — throughput | Không giới hạn |
| FIFO Queue — throughput | 300 TPS (có thể tăng lên 3000 TPS với batching) |
| Long polling wait time tối đa | 20 giây |
| In-flight messages (Standard) | 120,000 |
| In-flight messages (FIFO) | 20,000 |

### Amazon SNS

| Giới Hạn | Giá Trị |
|---|---|
| Message size tối đa | 256 KB |
| Số subscriber per topic | 12,500,000 |
| Topics per account | 100,000 |
| Standard Topic — throughput | Không giới hạn |
| FIFO Topic — throughput | 300 TPS |
| Message delivery retry | Phụ thuộc protocol (HTTP: 3 lần mặc định) |

### Amazon Kinesis Data Streams

| Giới Hạn | Giá Trị |
|---|---|
| Write per shard | 1 MB/s hoặc 1,000 records/s |
| Read per shard (standard) | 2 MB/s chia sẻ cho tất cả consumer |
| Read per shard (Enhanced Fan-Out) | 2 MB/s riêng cho mỗi consumer |
| Record size tối đa | 1 MB |
| Retention mặc định | 24 giờ |
| Retention tối đa | 365 ngày |
| Shard tối đa per stream | Không giới hạn cứng (soft limit 500) |

### Amazon EventBridge

| Giới Hạn | Giá Trị |
|---|---|
| Event size tối đa | 256 KB |
| Rules per event bus | 300 (có thể tăng) |
| Targets per rule | 5 |
| Throughput | Không giới hạn |
| Event age tối đa (retry) | 24 giờ |

### AWS Step Functions

| Giới Hạn | Giá Trị |
|---|---|
| Execution duration (Standard) | Tối đa 1 năm |
| Execution duration (Express) | Tối đa 5 phút |
| State machine definition size | 1 MB |
| History events per execution | 25,000 |
| Throughput (Standard) | 2,000 executions/giây |
| Throughput (Express) | 100,000 executions/giây |

---

## 🔄 Delivery Guarantees (Đảm Bảo Giao Vận)

| Dịch Vụ | Delivery Guarantee | Ghi Chú |
|---|---|---|
| SQS Standard | At-least-once (ít nhất một lần) | Có thể nhận duplicate — phải idempotent |
| SQS FIFO | Exactly-once (đúng một lần) | Deduplication trong 5 phút |
| SNS Standard | At-least-once | Subscriber phải xử lý duplicate |
| SNS FIFO | Exactly-once | Chỉ hỗ trợ SQS FIFO subscriber |
| Kinesis | At-least-once | Dùng sequence number để dedup |
| EventBridge | At-least-once | Retry tối đa 24 giờ |
| Step Functions | Exactly-once per state | Đảm bảo mỗi state chạy đúng một lần |

---

## 💰 Mô Hình Chi Phí (Cost Model)

| Dịch Vụ | Đơn Vị Tính Phí | Free Tier |
|---|---|---|
| SQS | Per 1 triệu request | 1 triệu request/tháng |
| SNS | Per 1 triệu notification | 1 triệu notification/tháng |
| EventBridge | Per 1 triệu event (custom) | AWS service events miễn phí |
| Kinesis | Per shard-hour + per 1 triệu PUT records | Không có Free Tier |
| Step Functions | Per 1,000 state transitions (Standard) | 4,000 state transitions/tháng |
| Amazon MQ | Per broker-hour + storage | Không có Free Tier |

---

## 🔍 Câu Hỏi Phỏng Vấn Thường Gặp Về So Sánh

**Q: SQS và SNS khác nhau thế nào về cơ bản?**
> SQS là message queue — một người nhận, xử lý xong thì xóa. SNS là pub/sub — một người gửi, nhiều người nhận cùng lúc, không lưu lại.

**Q: Khi nào nên dùng EventBridge thay vì SNS?**
> Dùng EventBridge khi: (1) cần filter theo content của event (không chỉ attributes), (2) cần tích hợp với SaaS partners, (3) cần Schema Registry để quản lý event structure, (4) cần Archive & Replay. SNS vẫn tốt hơn khi chỉ cần fan-out đơn giản với latency thấp.

**Q: Tại sao không dùng Kinesis cho mọi use case thay vì SQS?**
> Kinesis tốn hơn (tính tiền theo shard-hour dù không dùng), phức tạp hơn (phải quản lý shard, consumer offset). SQS hoàn toàn serverless, rẻ hơn cho task queue thông thường. Chỉ dùng Kinesis khi cần: nhiều consumer đọc cùng stream, replay, ordering, hoặc high-throughput analytics.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
