# AWS Application Integration Services — Mục Lục Toàn Diện

> Điều hướng nhanh toàn bộ kiến thức về dịch vụ tích hợp ứng dụng AWS

## 📁 Cấu Trúc Thư Mục

```
app-integration/
├── README.md                                   [BẮT ĐẦU ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    Mục lục này — điều hướng nhanh
│
├── 01-sqs/                                     Amazon SQS — Simple Queue Service
│   ├── README.md                               Tổng quan SQS, khi nào dùng
│   ├── 1-standard-vs-fifo.md                   So sánh Standard và FIFO Queue
│   ├── 2-visibility-timeout.md                 Visibility Timeout & Idempotency
│   ├── 3-dead-letter-queue.md                  DLQ — Hàng Đợi Thư Chết
│   ├── 4-long-polling.md                       Long Polling vs Short Polling
│   └── 5-batch-processing.md                   Xử lý theo lô với SQS
│
├── 02-sns/                                     Amazon SNS — Simple Notification Service
│   ├── README.md                               Tổng quan SNS, Pub/Sub model
│   ├── 1-topic-subscription.md                 Topic, Subscription, Protocol
│   ├── 2-message-filtering.md                  Message Filtering — Lọc Tin Nhắn
│   ├── 3-fanout-pattern.md                     Fan-out với SNS + SQS
│   └── 4-fifo-topic.md                         FIFO Topic — Thứ Tự & Exactly-Once
│
├── 03-eventbridge/                             Amazon EventBridge — Event Bus
│   ├── README.md                               Tổng quan EventBridge, event-driven
│   ├── 1-event-bus.md                          Default, Custom, Partner Event Bus
│   ├── 2-event-rules-patterns.md               Rule & Pattern — Lọc và Định Tuyến Sự Kiện
│   ├── 3-schema-registry.md                    Schema Registry — Quản Lý Lược Đồ
│   ├── 4-archive-replay.md                     Archive & Replay — Lưu Trữ và Phát Lại
│   └── 5-eventbridge-pipes.md                  EventBridge Pipes — Đường Ống Sự Kiện
│
├── 04-step-functions/                          AWS Step Functions — Workflow Orchestration
│   ├── README.md                               Tổng quan Step Functions, state machine
│   ├── 1-state-types.md                        Các loại State: Task, Choice, Parallel...
│   ├── 2-standard-vs-express.md                Standard vs Express Workflow
│   ├── 3-error-handling.md                     Catch & Retry — Xử Lý Lỗi
│   ├── 4-callback-pattern.md                   Callback Pattern với waitForTaskToken
│   └── 5-saga-orchestration.md                 Saga Pattern với Step Functions
│
├── 05-kinesis/                                 Amazon Kinesis — Real-time Streaming
│   ├── README.md                               Tổng quan Kinesis, so sánh các loại
│   ├── 1-data-streams.md                       Kinesis Data Streams — KDS, Shard
│   ├── 2-firehose.md                           Kinesis Firehose — Near-realtime Delivery
│   ├── 3-data-analytics.md                     Kinesis Data Analytics — SQL/Flink
│   ├── 4-shard-management.md                   Quản Lý Shard, tính toán capacity
│   └── 5-enhanced-fanout.md                    Enhanced Fan-Out — Khuếch Tán Nâng Cao
│
├── 06-amazon-mq/                               Amazon MQ — Managed Message Broker
│   ├── README.md                               Tổng quan Amazon MQ, khi nào dùng
│   ├── 1-activemq-vs-rabbitmq.md               ActiveMQ vs RabbitMQ — So Sánh
│   └── 2-migration-guide.md                    Di Chuyển Từ On-Premises sang MQ
│
├── 07-appsync/                                 AWS AppSync — Managed GraphQL
│   ├── README.md                               Tổng quan AppSync, GraphQL model
│   ├── 1-schema-resolvers.md                   Schema, Resolver, Data Sources
│   ├── 2-real-time-subscriptions.md            Real-time Subscription qua WebSocket
│   └── 3-caching-strategy.md                   Caching Strategy — Chiến Lược Bộ Nhớ Đệm
│
├── 08-patterns/                                Messaging Patterns — Mẫu Tích Hợp
│   ├── README.md                               Tổng quan các mẫu, khi nào dùng gì
│   ├── 1-service-comparison.md                 So Sánh: SQS vs SNS vs EventBridge vs Kinesis
│   ├── 2-saga-pattern.md                       Saga Pattern — Choreography vs Orchestration
│   ├── 3-idempotency.md                        Idempotency — Tính Bất Biến
│   ├── 4-outbox-pattern.md                     Outbox Pattern — Nhất Quán DB + Messaging
│   ├── 5-event-sourcing.md                     Event Sourcing — Nguồn Sự Kiện
│   ├── 6-cqrs.md                               CQRS — Phân Tách Lệnh và Truy Vấn
│   ├── 7-circuit-breaker.md                    Circuit Breaker — Cầu Dao Ngắt Lỗi
│   └── 8-competing-consumers.md                Competing Consumers — Scale Song Song
│
├── 09-monitoring/                              Monitoring & Observability — Giám Sát
│   ├── README.md                               Tổng quan monitoring cho integration
│   ├── 1-key-metrics.md                        Các Metrics Quan Trọng Theo Dịch Vụ
│   ├── 2-cloudwatch-alarms.md                  Cấu Hình CloudWatch Alarms
│   ├── 3-xray-tracing.md                       X-Ray Distributed Tracing — Theo Dõi Phân Tán
│   └── 4-dlq-monitoring.md                     Giám Sát DLQ — Phát Hiện Tin Nhắn Lỗi
│
└── 10-interview-prep/                          Chuẩn Bị Phỏng Vấn
    ├── README.md                               Tổng quan, chiến lược phỏng vấn
    ├── 1-INTERVIEW_GUIDE.md                    Top 25 câu hỏi + gợi ý trả lời
    ├── 2-service-comparison.md                 Bảng So Sánh Dịch Vụ Đầy Đủ
    ├── 3-system-design-scenarios.md            Bài Toán Thiết Kế Hệ Thống
    ├── 4-star-stories.md                       Mẫu Câu Chuyện Theo Phương Pháp STAR
    └── 5-90-day-study-plan.md                  Kế Hoạch Học 90 Ngày
```

---

## ✅ Trạng Thái Tạo Nội Dung

| Chủ Đề | File / Thư Mục | Trạng Thái | Chất Lượng |
|---|---|---|---|
| **Tổng Quan & Lộ Trình** | README.md | ✅ | Toàn diện |
| **Mục Lục** | INDEX.md | ✅ | Toàn diện |
| **Amazon SQS** | 01-sqs/ (6 files) | ✅ Hoàn thành | Toàn diện |
| **Amazon SNS** | 02-sns/ (5 files) | ✅ Hoàn thành | Toàn diện |
| **Amazon EventBridge** | 03-eventbridge/ (6 files) | ✅ Hoàn thành | Toàn diện |
| **AWS Step Functions** | 04-step-functions/ (6 files) | ✅ Hoàn thành | Toàn diện |
| **Amazon Kinesis** | 05-kinesis/ (6 files) | ✅ Hoàn thành | Toàn diện |
| **Amazon MQ** | 06-amazon-mq/ (3 files) | ✅ Hoàn thành | Toàn diện |
| **AWS AppSync** | 07-appsync/ (4 files) | ✅ Hoàn thành | Toàn diện |
| **Messaging Patterns** | 08-patterns/ (9 files) | ✅ Hoàn thành | Toàn diện |
| **Monitoring** | 09-monitoring/ (5 files) | ✅ Hoàn thành | Toàn diện |
| **Phỏng Vấn Prep** | 10-interview-prep/ (6 files) | ✅ Hoàn thành | Toàn diện |

---

## 🎯 Ưu Tiên Tạo Nội Dung

### Ưu Tiên Cao — Kỹ Năng Cốt Lõi

- [x] `01-sqs/README.md` — Nền tảng SQS, thường hỏi nhất ✅
- [x] `01-sqs/1-standard-vs-fifo.md` — Câu hỏi phỏng vấn phổ biến ✅
- [x] `01-sqs/3-dead-letter-queue.md` — DLQ quan trọng trong production ✅
- [x] `02-sns/3-fanout-pattern.md` — Fan-out pattern thực tế ✅
- [x] `08-patterns/1-service-comparison.md` — Bảng so sánh dịch vụ ✅
- [x] `10-interview-prep/1-INTERVIEW_GUIDE.md` — Top 25 câu hỏi ✅

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao

- [x] `03-eventbridge/README.md` — EventBridge ngày càng quan trọng ✅
- [x] `04-step-functions/5-saga-orchestration.md` — Saga là pattern hot ✅
- [x] `05-kinesis/1-data-streams.md` — Streaming architecture ✅
- [x] `08-patterns/2-saga-pattern.md` — Distributed transactions ✅
- [x] `08-patterns/3-idempotency.md` — Luôn được hỏi trong phỏng vấn ✅
- [x] `09-monitoring/1-key-metrics.md` — Giám sát production ✅

### Ưu Tiên Thấp — Tham Khảo Nâng Cao

- [ ] `06-amazon-mq/2-migration-guide.md` — Migration use case
- [x] `07-appsync/2-real-time-subscriptions.md` — GraphQL subscriptions ✅
- [x] `05-kinesis/5-enhanced-fanout.md` — Advanced Kinesis ✅
- [x] `10-interview-prep/3-system-design-scenarios.md` — Thiết kế hệ thống ✅
- [x] `10-interview-prep/5-90-day-study-plan.md` — Kế hoạch học ✅

---

## 🚀 Cách Dùng Knowledge Base Này

### Tự Học

```
1. Bắt đầu từ README.md
2. Chọn lộ trình phù hợp (Mới Bắt Đầu / Trung Cấp / Nâng Cao)
3. Đi qua từng thư mục theo thứ tự 01 → 10
4. Thực hành trên AWS Console với Free Tier
5. Xây dựng project demo cá nhân
```

### Chuẩn Bị Phỏng Vấn

```
1. Đọc 10-interview-prep/1-INTERVIEW_GUIDE.md
2. Nắm vững 08-patterns/1-service-comparison.md
3. Học sâu SQS (01-sqs/) và SNS (02-sns/) — luôn được hỏi
4. Học EventBridge (03-eventbridge/) và Step Functions (04-step-functions/)
5. Chuẩn bị câu chuyện tình huống theo phương pháp STAR
6. Tự thực hành giải thích trade-off bằng lời
```

### Áp Dụng Thực Tế

```
Dùng như tài liệu tham khảo:
- Chọn dịch vụ: Đọc 08-patterns/1-service-comparison.md
- Xử lý lỗi: Đọc 01-sqs/3-dead-letter-queue.md
- Thiết kế saga: Đọc 08-patterns/2-saga-pattern.md
- Tối ưu hiệu năng: Đọc 09-monitoring/1-key-metrics.md
- Tuân thủ idempotency: Đọc 08-patterns/3-idempotency.md
```

### Thiết Kế Hệ Thống

```
1. Đọc 08-patterns/1-service-comparison.md để chọn đúng dịch vụ
2. Áp dụng messaging pattern phù hợp từ 08-patterns/
3. Thiết kế error handling với DLQ và retry
4. Cài đặt monitoring từ 09-monitoring/
5. Đảm bảo idempotency ở consumer layer
```

---

## 📊 Ước Tính Thời Gian Học

| Phần | Thời Gian | Độ Khó | Ưu Tiên |
|---|---|---|---|
| Amazon SQS | 4–6 giờ | ⭐⭐ | Bắt buộc |
| Amazon SNS | 3–4 giờ | ⭐⭐ | Bắt buộc |
| Amazon EventBridge | 5–7 giờ | ⭐⭐⭐ | Bắt buộc |
| AWS Step Functions | 6–8 giờ | ⭐⭐⭐ | Bắt buộc |
| Amazon Kinesis | 8–10 giờ | ⭐⭐⭐ | Bắt buộc |
| Messaging Patterns | 6–8 giờ | ⭐⭐⭐ | Bắt buộc |
| Amazon MQ | 3–4 giờ | ⭐⭐ | Nên biết |
| AWS AppSync | 4–5 giờ | ⭐⭐ | Nên biết |
| Monitoring | 3–4 giờ | ⭐⭐ | Nên biết |
| Advanced Patterns | 8–10 giờ | ⭐⭐⭐ | Tốt hơn |

**Tổng cộng: 50–66 giờ để có kiến thức toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Mới Bắt Đầu (0–1 năm kinh nghiệm)

- [ ] Khái niệm Queue (Hàng Đợi) và Message (Tin Nhắn)
- [ ] SQS Standard Queue cơ bản
- [ ] SNS Topic và Subscription
- [ ] Dead Letter Queue — DLQ
- [ ] Phân biệt SQS và SNS

**Thời gian để nắm vững:** 2–3 tuần

### Trung Cấp (1–3 năm kinh nghiệm)

- [ ] Fan-out pattern (Mẫu Khuếch Tán)
- [ ] EventBridge rule và event pattern
- [ ] Step Functions state machine
- [ ] Kinesis Shard và tính throughput
- [ ] Idempotency trong consumer

**Thời gian để nâng cao:** 3–4 tuần

### Nâng Cao (3–5+ năm kinh nghiệm)

- [ ] Saga Pattern trong distributed systems
- [ ] Event Sourcing + CQRS
- [ ] Cross-account messaging architecture
- [ ] Cost optimization cho high-volume
- [ ] Disaster recovery cho messaging

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | Vị Trí |
|---|---|
| Tổng quan lộ trình | [README.md](README.md) |
| So sánh SQS / SNS / EventBridge / Kinesis | [08-patterns/1-service-comparison.md](08-patterns/1-service-comparison.md) |
| Thiết kế Fan-out Pattern | [02-sns/3-fanout-pattern.md](02-sns/3-fanout-pattern.md) |
| DLQ — Xử lý tin nhắn lỗi | [01-sqs/3-dead-letter-queue.md](01-sqs/3-dead-letter-queue.md) |
| Saga Pattern | [08-patterns/2-saga-pattern.md](08-patterns/2-saga-pattern.md) |
| Idempotency | [08-patterns/3-idempotency.md](08-patterns/3-idempotency.md) |
| Kinesis Shard tính toán | [05-kinesis/4-shard-management.md](05-kinesis/4-shard-management.md) |
| Step Functions Saga | [04-step-functions/5-saga-orchestration.md](04-step-functions/5-saga-orchestration.md) |
| Top 25 câu hỏi phỏng vấn | [10-interview-prep/1-INTERVIEW_GUIDE.md](10-interview-prep/1-INTERVIEW_GUIDE.md) |
| Monitoring metrics | [09-monitoring/1-key-metrics.md](09-monitoring/1-key-metrics.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép phần này và theo dõi tiến độ cá nhân:

```markdown
## Tiến Độ Học AWS Application Integration

### Giai Đoạn 1: Nền Tảng (Tuần 1–2)

- [ ] SQS Standard Queue
- [ ] SQS FIFO Queue
- [ ] SNS Topic & Subscription
- [ ] Dead Letter Queue (DLQ)
- [ ] Phân biệt SQS vs SNS

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–5)

- [ ] Fan-out Pattern với SNS + SQS
- [ ] EventBridge Rule & Pattern
- [ ] Step Functions State Machine
- [ ] Kinesis Shard & Partition Key
- [ ] Idempotency trong consumer

### Giai Đoạn 3: Nâng Cao (Tuần 6–8)

- [ ] Saga Pattern — Choreography
- [ ] Saga Pattern — Orchestration với Step Functions
- [ ] Kinesis Enhanced Fan-Out
- [ ] EventBridge Cross-account
- [ ] Outbox Pattern

### Giai Đoạn 4: Chuyên Sâu (Tuần 9+)

- [ ] Event Sourcing + CQRS
- [ ] Cost optimization strategy
- [ ] Disaster Recovery design
- [ ] System design scenarios
- [ ] Mock interview hoàn chỉnh
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích SQS, SNS, EventBridge, Kinesis rõ ràng không cần ghi chú
- [ ] Thiết kế fan-out pattern cho hệ thống thực tế
- [ ] Implement idempotency đúng cách trong consumer
- [ ] Cấu hình DLQ và xử lý poison message
- [ ] Chọn đúng dịch vụ cho từng use case

### ✅ Năng Lực Vận Hành

- [ ] Thiết kế error handling toàn diện với retry và DLQ
- [ ] Thiết lập monitoring và alerting cho messaging system
- [ ] Debug vấn đề trong distributed messaging
- [ ] Tối ưu chi phí cho high-volume messaging
- [ ] Implement Saga Pattern cho distributed transactions

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 25 câu hỏi AWS Integration
- [ ] Giải thích trade-off giữa các dịch vụ AWS
- [ ] Thiết kế hệ thống event-driven quy mô lớn
- [ ] Kể 2–3 câu chuyện tình huống theo phương pháp STAR
- [ ] So sánh AWS messaging với alternatives (RabbitMQ, Kafka)

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng cách làm thực tế:** Tạo queue thật, gửi message thật trên AWS Console
2. **Vẽ sơ đồ kiến trúc:** Mỗi pattern nên vẽ ra trước khi code
3. **Hiểu trade-off:** Biết WHY chọn SQS thay vì SNS, không chỉ HOW dùng
4. **Thực hành giải thích:** Giải thích lại cho người khác — cách học tốt nhất
5. **Theo dõi chi phí:** Luôn nhớ pricing model của từng dịch vụ
6. **Đọc case study AWS:** Blog AWS Architecture có nhiều ví dụ thực tế
7. **Thực hành monitoring:** Cài alarm ngay từ đầu, đừng chờ production
8. **Nhớ limits (giới hạn):** Số message/s, message size — quan trọng khi thiết kế

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn bổ sung nội dung?

Tài liệu này luôn được cập nhật. Chào đón đóng góp:

- [ ] Sửa lỗi nội dung hiện có
- [ ] Bổ sung phần chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm dự án
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Hướng dẫn cho từng dịch vụ cụ thể

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 2.0 (Amazon SQS + Amazon SNS + Amazon EventBridge + AWS Step Functions + Amazon Kinesis + Amazon MQ + AWS AppSync + Messaging Patterns + Monitoring + Interview Prep hoàn thành)
**Trạng Thái:** ✅ README + INDEX + 01-sqs + 02-sns + 03-eventbridge + 04-step-functions + 05-kinesis + 06-amazon-mq + 07-appsync + 08-patterns + 09-monitoring + 10-interview-prep hoàn thành
