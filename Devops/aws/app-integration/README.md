# 🔗 AWS Application Integration Services — Lộ Trình Học Tập

> Hướng dẫn toàn diện về các dịch vụ tích hợp ứng dụng trên AWS, bao gồm messaging (nhắn tin), eventing (sự kiện), workflow (luồng công việc) và streaming (truyền dữ liệu thời gian thực).

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Dịch Vụ](#tổng-quan-dịch-vụ)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Kiến trúc Event-Driven (Hướng Sự Kiện) và Messaging Patterns (Mẫu Nhắn Tin)
- [ ] Amazon SQS — Simple Queue Service (Dịch Vụ Hàng Đợi Đơn Giản)
- [ ] Amazon SNS — Simple Notification Service (Dịch Vụ Thông Báo Đơn Giản)
- [ ] So sánh Queue vs Pub/Sub vs Event Bus

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–5)**

- [ ] Amazon EventBridge — Event Bus (Xe Buýt Sự Kiện)
- [ ] AWS Step Functions — Workflow Orchestration (Điều Phối Luồng Công Việc)
- [ ] Amazon Kinesis — Real-time Streaming (Truyền Dữ Liệu Thời Gian Thực)
- [ ] Dead Letter Queue — DLQ (Hàng Đợi Thư Chết — Xử Lý Tin Nhắn Lỗi)

### **Giai Đoạn 3: Nâng Cao (Tuần 6–8)**

- [ ] Amazon MQ — Managed Message Broker (Môi Giới Tin Nhắn Được Quản Lý)
- [ ] AWS AppSync — Managed GraphQL API
- [ ] Saga Pattern (Mẫu Saga) trong Microservices
- [ ] Idempotency (Tính Bất Biến) và Exactly-Once Delivery (Giao Vận Đúng Một Lần)

### **Giai Đoạn 4: Chuyên Sâu (Tuần 9+)**

- [ ] Fan-out Pattern (Mẫu Khuếch Tán) với SNS + SQS
- [ ] Event Sourcing (Nguồn Sự Kiện) và CQRS (Command Query Responsibility Segregation)
- [ ] Cross-account và Cross-region Messaging (Nhắn Tin Liên Tài Khoản / Liên Vùng)
- [ ] Cost Optimization (Tối Ưu Chi Phí) và Capacity Planning (Lập Kế Hoạch Năng Lực)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực | Ưu Tiên | Thời Gian | Trạng Thái |
|---|---|---|---|
| **Amazon SQS — Hàng Đợi Tin Nhắn** | ⭐⭐⭐ | 1 tuần | - |
| **Amazon SNS — Pub/Sub Thông Báo** | ⭐⭐⭐ | 1 tuần | - |
| **Amazon EventBridge — Sự Kiện** | ⭐⭐⭐ | 1,5 tuần | - |
| **AWS Step Functions — Luồng Công Việc** | ⭐⭐⭐ | 1,5 tuần | - |
| **Amazon Kinesis — Data Streaming** | ⭐⭐⭐ | 2 tuần | - |
| **Dead Letter Queue — Xử Lý Lỗi** | ⭐⭐⭐ | 0,5 tuần | - |
| **Amazon MQ — Message Broker** | ⭐⭐ | 1 tuần | - |
| **AWS AppSync — GraphQL API** | ⭐⭐ | 1 tuần | - |
| **Idempotency & Delivery Guarantees** | ⭐⭐⭐ | 1 tuần | - |
| **Saga Pattern trong Microservices** | ⭐⭐ | 1 tuần | - |

---

## 🗂️ Tổng Quan Dịch Vụ

### 📁 **1. Amazon SQS — Simple Queue Service** (`01-sqs/`)

**SQS** là dịch vụ hàng đợi tin nhắn được quản lý hoàn toàn, cho phép tách rời (decouple) các thành phần ứng dụng:

- **Standard Queue** vs **FIFO Queue** (First-In, First-Out — Vào Trước Ra Trước)
- **Visibility Timeout** (Thời Gian Ẩn Tin Nhắn) — tránh xử lý trùng lặp
- **Long Polling** vs **Short Polling** (Lấy Tin Nhắn Chờ Dài vs Ngắn)
- **Message Retention Period** (Chu Kỳ Lưu Giữ Tin Nhắn) — tối đa 14 ngày
- **Dead Letter Queue — DLQ** (Hàng Đợi Thư Chết) — xử lý tin nhắn thất bại
- **Delay Queue** (Hàng Đợi Trễ) và **Message Timer** (Hẹn Giờ Tin Nhắn)
- **Batch Processing** (Xử Lý Theo Lô) — tối đa 10 tin nhắn mỗi lần

### 📁 **2. Amazon SNS — Simple Notification Service** (`02-sns/`)

**SNS** là dịch vụ Pub/Sub (Publisher/Subscriber — Nhà Phát/Người Đăng Ký) được quản lý:

- **Topic** (Chủ Đề) và **Subscription** (Đăng Ký Nhận)
- **Push-based** delivery (Giao Vận Theo Kiểu Đẩy) — khác SQS pull-based
- **Fan-out Pattern** (Mẫu Khuếch Tán) — một topic, nhiều subscriber
- **Message Filtering** (Lọc Tin Nhắn) — subscriber chỉ nhận tin phù hợp
- **FIFO Topic** — đảm bảo thứ tự và chính xác một lần giao vận
- Hỗ trợ endpoint: SQS, Lambda, HTTP/S, Email, SMS, Mobile Push
- **Message Attributes** (Thuộc Tính Tin Nhắn) cho filtering

### 📁 **3. Amazon EventBridge** (`03-eventbridge/`)

**EventBridge** là serverless event bus (xe buýt sự kiện không máy chủ) cho kiến trúc hướng sự kiện:

- **Event Bus** — Default, Custom, và Partner event buses
- **Event Rule** (Quy Tắc Sự Kiện) với **Event Pattern** (Mẫu Sự Kiện)
- **Scheduled Rules** (Quy Tắc Lịch Trình) — cron/rate expressions
- **Schema Registry** (Kho Lưu Lược Đồ) — quản lý cấu trúc sự kiện
- **Event Archive** (Lưu Trữ Sự Kiện) và **Replay** (Phát Lại)
- **Cross-account Routing** (Định Tuyến Liên Tài Khoản)
- **Pipes** (Đường Ống) — kết nối source và target với transformation (biến đổi)
- Phân biệt với SNS: EventBridge dành cho event-driven architecture phức tạp hơn

### 📁 **4. AWS Step Functions** (`04-step-functions/`)

**Step Functions** là dịch vụ điều phối workflow (luồng công việc) dưới dạng state machine (máy trạng thái):

- **Standard Workflow** vs **Express Workflow** — trade-off chi phí và độ bền
- **State Types** (Loại Trạng Thái): Task, Choice, Wait, Parallel, Map, Pass, Succeed, Fail
- **Task Integration** (Tích Hợp Tác Vụ) — Lambda, ECS, DynamoDB, SQS, SNS...
- **Error Handling** (Xử Lý Lỗi) với `Catch` và `Retry` (Bắt Lỗi và Thử Lại)
- **Callback Pattern** (Mẫu Gọi Lại) với `waitForTaskToken`
- **Activity Worker** (Công Nhân Hoạt Động) cho long-running tasks
- **Execution History** (Lịch Sử Thực Thi) — debug toàn bộ luồng
- Saga Pattern (Mẫu Saga) — quản lý distributed transactions (giao dịch phân tán)

### 📁 **5. Amazon Kinesis** (`05-kinesis/`)

**Kinesis** là nền tảng xử lý data streaming (luồng dữ liệu) thời gian thực:

- **Kinesis Data Streams — KDS** — Luồng Dữ Liệu Kinesis
  - **Shard** (Mảnh) — đơn vị năng lực throughput (thông lượng)
  - **Partition Key** (Khóa Phân Vùng) — định tuyến bản ghi vào shard
  - **Sequence Number** (Số Thứ Tự) — thứ tự trong shard
  - Retention (Lưu Giữ): 24 giờ mặc định, tối đa 365 ngày
  - **Enhanced Fan-Out** (Khuếch Tán Nâng Cao) — 2MB/s per consumer per shard
- **Kinesis Data Firehose** — Firehose Dữ Liệu Kinesis
  - Near-real-time delivery (Giao Vận Gần Thời Gian Thực) đến S3, Redshift, OpenSearch
  - Built-in transformation với Lambda
  - Không cần quản lý consumer (người tiêu dùng)
- **Kinesis Data Analytics** — Phân Tích Dữ Liệu Kinesis
  - SQL hoặc Apache Flink để xử lý streaming
  - Anomaly detection (Phát Hiện Bất Thường) thời gian thực

### 📁 **6. Amazon MQ** (`06-amazon-mq/`)

**Amazon MQ** là message broker được quản lý tương thích với các protocol tiêu chuẩn:

- Hỗ trợ **Apache ActiveMQ** và **RabbitMQ**
- Protocol: **AMQP** (Advanced Message Queuing Protocol), **MQTT**, **STOMP**, **OpenWire**
- **Broker Deployment** (Triển Khai Môi Giới): Single-instance và Active/Standby HA
- Trường hợp dùng: Migration (di chuyển) từ on-premises message broker
- So sánh với SQS/SNS: MQ dành cho legacy systems (hệ thống cũ) cần protocol riêng

### 📁 **7. AWS AppSync** (`07-appsync/`)

**AppSync** là dịch vụ GraphQL API được quản lý, tự động đồng bộ dữ liệu:

- **Schema** (Lược Đồ) — định nghĩa types, queries, mutations, subscriptions
- **Resolver** (Bộ Giải Quyết) — kết nối field với data source
- **Data Sources** (Nguồn Dữ Liệu): DynamoDB, Lambda, RDS, OpenSearch, HTTP
- **Real-time Subscriptions** (Đăng Ký Thời Gian Thực) qua WebSocket
- **Offline Support** (Hỗ Trợ Ngoại Tuyến) với Amplify DataStore
- **Caching** (Bộ Nhớ Đệm) ở cấp độ resolver

### 📁 **8. Messaging Patterns** (`08-patterns/`)

Các mẫu kiến trúc tích hợp quan trọng:

- **Saga Pattern** (Mẫu Saga) — Choreography (Vũ Đạo) vs Orchestration (Điều Phối)
- **Idempotency** (Tính Bất Biến) — đảm bảo xử lý an toàn khi nhận trùng
- **At-least-once vs Exactly-once Delivery** (Giao Vận Ít Nhất Một Lần vs Đúng Một Lần)
- **Outbox Pattern** (Mẫu Hộp Thư Đi) — đảm bảo tính nhất quán DB + messaging
- **Event Sourcing** (Nguồn Sự Kiện) — lưu trạng thái qua chuỗi sự kiện
- **CQRS** — Command Query Responsibility Segregation (Phân Tách Trách Nhiệm Lệnh và Truy Vấn)
- **Circuit Breaker** (Cầu Dao) — ngăn cascade failure (lỗi dây chuyền)
- **Competing Consumers** (Người Tiêu Dùng Cạnh Tranh) — scale xử lý song song

### 📁 **9. Monitoring & Observability** (`09-monitoring/`)

Giám sát hệ thống tích hợp:

- **CloudWatch Metrics** (Số Liệu CloudWatch) cho SQS, SNS, EventBridge, Kinesis
- **CloudWatch Alarms** (Cảnh Báo CloudWatch) — ngưỡng quan trọng
- **AWS X-Ray** — Distributed Tracing (Theo Dõi Phân Tán) toàn luồng
- **CloudWatch Logs Insights** — phân tích log
- Key metrics: `ApproximateNumberOfMessages`, `Age of Oldest Message`, `Iterator Age`
- **Dead Letter Queue Monitoring** (Giám Sát Hàng Đợi Thư Chết)
- **Step Functions Execution Metrics** (Số Liệu Thực Thi Step Functions)

### 📁 **10. Phỏng Vấn & Thiết Kế Hệ Thống** (`10-interview-prep/`)

- Top 25 câu hỏi phỏng vấn về AWS Integration
- Bài toán thiết kế hệ thống: Order Processing (Xử Lý Đơn Hàng), Event-Driven Microservices
- Câu hỏi tình huống và đánh đổi (trade-off)
- So sánh dịch vụ: SQS vs SNS vs EventBridge vs Kinesis

---

## 🎓 Theo Dịch Vụ AWS

### **Amazon SQS**

```
Điểm mạnh: Decoupling (Tách Rời), Durability (Độ Bền), Auto Scaling
Phù hợp: Task queuing, background jobs, load leveling (san phẳng tải)
Học trong: 01-sqs/, 08-patterns/competing-consumers.md
```

### **Amazon SNS**

```
Điểm mạnh: Fan-out (Khuếch Tán), Push delivery (Giao Vận Đẩy), Multi-protocol
Phù hợp: Notifications (Thông Báo), alerts, event distribution (phân phối sự kiện)
Học trong: 02-sns/, 08-patterns/fan-out.md
```

### **Amazon EventBridge**

```
Điểm mạnh: Event routing (Định Tuyến Sự Kiện), Schema registry, SaaS integration
Phù hợp: Event-driven architecture (Kiến Trúc Hướng Sự Kiện), microservices
Học trong: 03-eventbridge/, 08-patterns/event-sourcing.md
```

### **AWS Step Functions**

```
Điểm mạnh: Visual workflow (Luồng Công Việc Trực Quan), Error handling, Long-running
Phù hợp: Orchestration (Điều Phối), Saga pattern, complex workflows
Học trong: 04-step-functions/, 08-patterns/saga-pattern.md
```

### **Amazon Kinesis**

```
Điểm mạnh: Real-time (Thời Gian Thực), High throughput (Thông Lượng Cao), Replay (Phát Lại)
Phù hợp: Log ingestion (Thu Nạp Log), clickstream, IoT data, analytics
Học trong: 05-kinesis/
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề | Thư Mục | Ưu Tiên |
|---|---|---|
| Bắt đầu ở đây | [README.md](./README.md) | Đọc đầu tiên |
| Tổng mục lục | [INDEX.md](./INDEX.md) | Điều hướng |
| SQS vs SNS vs EventBridge | [08-patterns/service-comparison.md](./08-patterns/service-comparison.md) | Quan trọng |
| Saga Pattern | [08-patterns/saga-pattern.md](./08-patterns/saga-pattern.md) | Phỏng vấn |
| Câu hỏi phỏng vấn | [10-interview-prep/](./10-interview-prep/) | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng

### Mới Bắt Đầu (0–1 năm)

- [ ] Hiểu khái niệm Queue (Hàng Đợi) và Message (Tin Nhắn)
- [ ] Tạo và dùng được SQS Standard Queue
- [ ] Gửi thông báo qua SNS
- [ ] Phân biệt được SQS và SNS
- [ ] Biết cấu hình Dead Letter Queue — DLQ

### Trung Cấp (1–3 năm)

- [ ] Thiết kế Fan-out Pattern với SNS + SQS
- [ ] Xây dựng EventBridge rule và target
- [ ] Tạo Step Functions state machine cơ bản
- [ ] Hiểu Kinesis Shard và Partition Key
- [ ] Implement Idempotency trong consumer (người tiêu dùng)
- [ ] Giám sát DLQ và xử lý poison message (tin nhắn độc hại)

### Nâng Cao (3–5+ năm)

- [ ] Thiết kế Saga Pattern cho distributed transactions
- [ ] Event Sourcing + CQRS architecture
- [ ] Kinesis Enhanced Fan-Out và custom consumer
- [ ] Cross-account EventBridge routing
- [ ] Cost optimization cho high-volume messaging
- [ ] Disaster recovery (Khôi Phục Thảm Họa) cho messaging systems

---

## 🚀 Bắt Đầu Như Thế Nào

### Bước 1: Xác Định Mục Tiêu

```
Chọn hướng của bạn:
- Generalist — nắm tất cả dịch vụ ở mức vừa
- Specialist — đi sâu SQS/SNS/EventBridge cho backend/microservices
- Architect — thiết kế hệ thống event-driven quy mô lớn
```

### Bước 2: Dựng Lab Thực Hành

```bash
# Dùng AWS CLI để tạo SQS queue
aws sqs create-queue --queue-name my-demo-queue

# Tạo SNS topic
aws sns create-topic --name my-demo-topic

# Gửi tin nhắn thử
aws sqs send-message \
  --queue-url <QUEUE_URL> \
  --message-body "Hello from SQS"
```

### Bước 3: Học + Thực Hành

```
1. Đọc module (30 phút)
2. Dựng lab trên AWS Console (30 phút)
3. Viết code thực tế: producer + consumer (30–60 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Cho mỗi chủ đề, chuẩn bị câu chuyện theo phương pháp STAR:
- Situation (Tình Huống): Bối cảnh dự án
- Task (Nhiệm Vụ): Bạn cần giải quyết gì
- Action (Hành Động): Bạn đã làm gì với dịch vụ AWS
- Result (Kết Quả): Kết quả đạt được, số liệu cụ thể
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Designing Data-Intensive Applications"** — Martin Kleppmann — Kiến trúc hệ thống dữ liệu
- **"Enterprise Integration Patterns"** — Gregor Hohpe — Các mẫu tích hợp kinh điển
- **"Building Microservices"** — Sam Newman — Microservices thực tế
- **"Software Architecture Patterns"** — Mark Richards — Các mẫu kiến trúc phần mềm

### Tài Liệu Chính Thức AWS

- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/sqs/)
- [Amazon SNS Developer Guide](https://docs.aws.amazon.com/sns/)
- [Amazon EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/)
- [AWS Step Functions Developer Guide](https://docs.aws.amazon.com/step-functions/)
- [Amazon Kinesis Developer Guide](https://docs.aws.amazon.com/kinesis/)

### Bài Viết & Blog Quan Trọng

- AWS Architecture Blog — các giải pháp thực tế
- AWS re:Invent sessions về messaging và event-driven architecture
- "The Queue" newsletter — AWS messaging updates

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Danh Mục

#### SQS & SNS

- [ ] Phân biệt SQS Standard và FIFO Queue
- [ ] Tại sao cần Dead Letter Queue — DLQ?
- [ ] Fan-out pattern với SNS + SQS hoạt động như thế nào?
- [ ] Visibility Timeout là gì và cấu hình thế nào?
- [ ] At-least-once delivery (Giao Vận Ít Nhất Một Lần) — hàm ý gì?

#### EventBridge & Step Functions

- [ ] Khi nào dùng EventBridge thay vì SNS?
- [ ] Standard Workflow vs Express Workflow trong Step Functions
- [ ] Cách implement Saga Pattern với Step Functions
- [ ] Schema Registry trong EventBridge giải quyết vấn đề gì?

#### Kinesis

- [ ] Phân biệt Kinesis Data Streams và Kinesis Firehose
- [ ] Shard (Mảnh) là gì, cách tính số shard cần thiết?
- [ ] Tại sao Iterator Age (Tuổi Trình Lặp) là metric quan trọng?
- [ ] Enhanced Fan-Out (Khuếch Tán Nâng Cao) khác gì so với standard?

#### Thiết Kế Hệ Thống

- [ ] Thiết kế hệ thống xử lý đơn hàng (order processing) với messaging
- [ ] Xử lý idempotency trong distributed system (hệ thống phân tán)
- [ ] Chiến lược retry (thử lại) và circuit breaker (cầu dao)
- [ ] Lựa chọn giữa SQS / SNS / EventBridge / Kinesis cho use case cụ thể

Xem `10-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc bắt đầu dự án mới, kiểm tra:

- [ ] Giải thích được sự khác biệt SQS vs SNS vs EventBridge vs Kinesis
- [ ] Thiết kế được fan-out pattern (mẫu khuếch tán) cho một hệ thống cụ thể
- [ ] Giải thích idempotency và cách implement
- [ ] Biết cấu hình DLQ và xử lý poison message
- [ ] Hiểu trade-off giữa Standard và FIFO Queue
- [ ] Thiết kế được Saga Pattern cho distributed transactions
- [ ] Tính được số shard Kinesis cần thiết cho throughput mục tiêu
- [ ] Giải thích tại sao và khi nào dùng Step Functions
- [ ] Hiểu chi phí của từng dịch vụ và cách tối ưu
- [ ] Cấu hình được monitoring và alerting cho messaging system

---

## 📋 Cách Dùng Tài Liệu Này

### Tự Học

1. Bắt đầu từ [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành trên AWS Console và AWS CLI
4. Xây dựng project demo cá nhân

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào [10-interview-prep/](./10-interview-prep/)
2. Nắm vững so sánh các dịch vụ
3. Chuẩn bị câu chuyện tình huống (STAR)
4. Thực hành giải thích trade-off rõ ràng

### Áp Dụng Thực Tế

1. Tham khảo [08-patterns/](./08-patterns/) để chọn đúng pattern
2. Dùng [09-monitoring/](./09-monitoring/) để thiết lập giám sát
3. Dùng [10-interview-prep/service-comparison.md](./10-interview-prep/) khi cần lựa chọn dịch vụ
4. Kiểm tra checklist trước khi deploy lên production

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình phù hợp (Mới Bắt Đầu / Trung Cấp / Nâng Cao)
├─ 3️⃣  Bắt đầu với 01-sqs/ — nền tảng quan trọng nhất
├─ 4️⃣  Dựng lab trên AWS (dùng Free Tier)
├─ 5️⃣  Thực hành từng dịch vụ với code thực tế
├─ 6️⃣  Xây dựng project demo: hệ thống order processing
└─ 7️⃣  Chuẩn bị phỏng vấn với 10-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
