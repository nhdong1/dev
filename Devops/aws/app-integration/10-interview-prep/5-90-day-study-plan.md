# Kế Hoạch Học 90 Ngày — AWS Application Integration

> Lộ trình có cấu trúc từ zero đến tự tin phỏng vấn Senior Engineer AWS Integration

---

## Tổng Quan Kế Hoạch

| Giai Đoạn | Tuần | Mục Tiêu | Chủ Đề Chính |
|---|---|---|---|
| **Giai Đoạn 1: Nền Tảng** | 1–3 | Hiểu core concepts | SQS, SNS, cơ bản |
| **Giai Đoạn 2: Kỹ Năng Cốt Lõi** | 4–6 | Làm được trên thực tế | EventBridge, Step Functions, Kinesis |
| **Giai Đoạn 3: Nâng Cao** | 7–9 | Hiểu sâu pattern | Saga, Idempotency, Design Patterns |
| **Giai Đoạn 4: Phỏng Vấn** | 10–13 | Sẵn sàng phỏng vấn | Review, mock interview, storytelling |

**Thời gian học mỗi ngày:** 1–2 giờ (đủ dùng nếu học đều đặn)

---

## Giai Đoạn 1: Nền Tảng (Tuần 1–3)

### Tuần 1: Amazon SQS — Nền Tảng Messaging

**Mục tiêu cuối tuần:** Giải thích được SQS, tạo được queue và gửi/nhận message.

#### Ngày 1–2: Khái Niệm Cơ Bản

**Đọc:**
- [ ] `01-sqs/README.md` — Tổng quan SQS
- [ ] `01-sqs/1-standard-vs-fifo.md` — Standard vs FIFO

**Lab thực hành:**
```bash
# Tạo SQS queue bằng AWS CLI
aws sqs create-queue --queue-name my-test-queue

# Gửi message
aws sqs send-message \
  --queue-url <QUEUE_URL> \
  --message-body "Hello SQS"

# Nhận message
aws sqs receive-message \
  --queue-url <QUEUE_URL> \
  --wait-time-seconds 5

# Xóa message
aws sqs delete-message \
  --queue-url <QUEUE_URL> \
  --receipt-handle <RECEIPT_HANDLE>
```

**Câu hỏi tự kiểm tra:**
- Standard Queue và FIFO Queue khác nhau thế nào?
- Khi nào cần FIFO mặc dù throughput bị giới hạn?

#### Ngày 3–4: Visibility Timeout và DLQ

**Đọc:**
- [ ] `01-sqs/2-visibility-timeout.md`
- [ ] `01-sqs/3-dead-letter-queue.md`

**Lab thực hành:**
```bash
# Tạo DLQ
aws sqs create-queue --queue-name my-dlq

# Tạo main queue với DLQ
aws sqs create-queue --queue-name my-main-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"3\"}",
    "VisibilityTimeout": "30"
  }'
```

**Câu hỏi tự kiểm tra:**
- Điều gì xảy ra nếu Visibility Timeout quá ngắn?
- Tại sao DLQ quan trọng hơn chỉ là "nơi chứa message lỗi"?

#### Ngày 5–7: Long Polling và Batch Processing

**Đọc:**
- [ ] `01-sqs/4-long-polling.md`
- [ ] `01-sqs/5-batch-processing.md`

**Bài tập tổng hợp tuần 1:**

Viết một đoạn code (Python/Node.js/Java) để:
1. Gửi 20 message vào SQS Queue
2. Consumer nhận batch 10 message mỗi lần
3. Xử lý (giả lập: sleep random 100–500ms)
4. Delete message sau khi xử lý thành công
5. Nếu message bị nhận > 3 lần → tự động vào DLQ

---

### Tuần 2: Amazon SNS và Fan-Out Pattern

**Mục tiêu cuối tuần:** Thiết kế và implement Fan-out Pattern với SNS + SQS.

#### Ngày 1–2: SNS Cơ Bản

**Đọc:**
- [ ] `02-sns/README.md`
- [ ] `02-sns/1-topic-subscription.md`

**Lab:**
```bash
# Tạo SNS topic
aws sns create-topic --name my-test-topic

# Subscribe SQS vào SNS
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol sqs \
  --notification-endpoint <SQS_QUEUE_ARN>

# Gửi message qua SNS
aws sns publish \
  --topic-arn <TOPIC_ARN> \
  --message "Hello from SNS"
```

#### Ngày 3–4: Message Filtering và FIFO Topic

**Đọc:**
- [ ] `02-sns/2-message-filtering.md`
- [ ] `02-sns/4-fifo-topic.md`

**Lab:** Cấu hình Message Filtering để một SQS Queue chỉ nhận message có attribute `event_type = "ORDER_CREATED"`.

#### Ngày 5–7: Fan-Out Pattern — Project Thực Hành

**Đọc:**
- [ ] `02-sns/3-fanout-pattern.md`

**Project tuần 2:** Xây dựng hệ thống thông báo mini:
```
POST /orders → SNS Topic
                ├── SQS Queue A → Lambda → Print "Email sent to {email}"
                └── SQS Queue B → Lambda → Print "SMS sent to {phone}"
```

Kiểm tra: Kill Lambda A — SNS + SQS B vẫn hoạt động bình thường.

---

### Tuần 3: Amazon EventBridge

**Mục tiêu cuối tuần:** Tạo được EventBridge rule với pattern filtering và target.

#### Ngày 1–3: Event Bus và Rules

**Đọc:**
- [ ] `03-eventbridge/README.md`
- [ ] `03-eventbridge/1-event-bus.md`
- [ ] `03-eventbridge/2-event-rules-patterns.md`

**Lab:** Tạo Custom Event Bus, tạo Rule với pattern lọc, target là Lambda.

#### Ngày 4–5: Schema Registry và Archive

**Đọc:**
- [ ] `03-eventbridge/3-schema-registry.md`
- [ ] `03-eventbridge/4-archive-replay.md`

**Lab:** Enable Schema Discovery, xem schema tự động được tạo sau khi gửi event.

#### Ngày 6–7: EventBridge Pipes

**Đọc:**
- [ ] `03-eventbridge/5-eventbridge-pipes.md`

**Tổng kết tuần 3:** So sánh được SNS vs EventBridge — khi nào dùng cái nào.

---

## Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 4–6)

### Tuần 4: AWS Step Functions

**Mục tiêu cuối tuần:** Xây dựng được state machine với error handling.

#### Ngày 1–2: Khái Niệm Cơ Bản

**Đọc:**
- [ ] `04-step-functions/README.md`
- [ ] `04-step-functions/1-state-types.md`

**Lab:** Tạo state machine đơn giản:
```
Start → Wait 5s → Call Lambda → Log Result → End
```

#### Ngày 3–4: Standard vs Express và Error Handling

**Đọc:**
- [ ] `04-step-functions/2-standard-vs-express.md`
- [ ] `04-step-functions/3-error-handling.md`

**Lab:** Thêm Catch và Retry vào state machine. Test bằng cách inject lỗi trong Lambda.

#### Ngày 5–7: Callback Pattern và Saga

**Đọc:**
- [ ] `04-step-functions/4-callback-pattern.md`
- [ ] `04-step-functions/5-saga-orchestration.md`

**Project tuần 4:** Saga đơn giản cho Order Flow:
```
ReserveInventory → ChargePayment → CreateShipment
     Catch → ReleaseInventory → CancelOrder
```

---

### Tuần 5: Amazon Kinesis

**Mục tiêu cuối tuần:** Hiểu shard, partition key, tính được số shard cần thiết.

#### Ngày 1–3: Kinesis Data Streams

**Đọc:**
- [ ] `05-kinesis/README.md`
- [ ] `05-kinesis/1-data-streams.md`
- [ ] `05-kinesis/4-shard-management.md`

**Bài tập tính shard:**
```
Scenario 1: 5,000 records/giây, 2KB/record, 2 consumer
Scenario 2: 1,000 records/giây, 500KB/record, 1 consumer
Scenario 3: 10,000 records/giây, 100 bytes/record, 5 consumer (Enhanced Fan-Out)
```

#### Ngày 4–5: Kinesis Firehose

**Đọc:**
- [ ] `05-kinesis/2-firehose.md`

**Lab:** Tạo Kinesis Firehose delivery stream → S3. Gửi data và kiểm tra file trong S3.

#### Ngày 6–7: Enhanced Fan-Out và Kinesis Analytics

**Đọc:**
- [ ] `05-kinesis/5-enhanced-fanout.md`
- [ ] `05-kinesis/3-data-analytics.md`

---

### Tuần 6: Amazon MQ và AWS AppSync

**Mục tiêu:** Hiểu khi nào dùng MQ, cơ bản về AppSync GraphQL.

#### Ngày 1–3: Amazon MQ

**Đọc:**
- [ ] `06-amazon-mq/README.md`
- [ ] `06-amazon-mq/1-activemq-vs-rabbitmq.md`
- [ ] `06-amazon-mq/2-migration-guide.md`

**Câu hỏi tự kiểm tra:**
- Tại sao không dùng Amazon MQ cho dự án mới?
- Khi nào việc migrate từ RabbitMQ sang SQS là hợp lý?

#### Ngày 4–7: AWS AppSync

**Đọc:**
- [ ] `07-appsync/README.md`
- [ ] `07-appsync/2-real-time-subscriptions.md`
- [ ] `07-appsync/3-caching-strategy.md`

**Lab:** Tạo AppSync API với một query và một subscription. Test với Postman hoặc AppSync console.

---

## Giai Đoạn 3: Nâng Cao (Tuần 7–9)

### Tuần 7: Messaging Patterns — Phần 1

**Mục tiêu:** Hiểu sâu và giải thích được Saga, Idempotency, Outbox Pattern.

#### Ngày 1–2: Service Comparison và Saga Pattern

**Đọc:**
- [ ] `08-patterns/1-service-comparison.md`
- [ ] `08-patterns/2-saga-pattern.md`

**Bài tập:** Vẽ sơ đồ Choreography Saga cho Order Flow với EventBridge.

#### Ngày 3–4: Idempotency

**Đọc:**
- [ ] `08-patterns/3-idempotency.md`

**Bài tập code:**
```python
# Implement idempotency key check
# 1. Nhận message từ SQS
# 2. Check Redis / DynamoDB xem message_id đã xử lý chưa
# 3. Nếu rồi: skip và delete message
# 4. Nếu chưa: xử lý, lưu message_id, delete message
```

#### Ngày 5–7: Outbox Pattern và Event Sourcing

**Đọc:**
- [ ] `08-patterns/4-outbox-pattern.md`
- [ ] `08-patterns/5-event-sourcing.md`

---

### Tuần 8: Messaging Patterns — Phần 2

#### Ngày 1–2: CQRS

**Đọc:**
- [ ] `08-patterns/6-cqrs.md`

**Bài tập thiết kế:** Thiết kế kiến trúc CQRS cho hệ thống quản lý inventory với separate read model và write model.

#### Ngày 3–4: Circuit Breaker và Competing Consumers

**Đọc:**
- [ ] `08-patterns/7-circuit-breaker.md`
- [ ] `08-patterns/8-competing-consumers.md`

#### Ngày 5–7: Tổng Hợp Patterns

**Bài tập lớn:** Thiết kế kiến trúc Event-Driven Microservices cho hệ thống booking (đặt chỗ) khách sạn:
- 5 service: Hotel, Booking, Payment, Email, Analytics
- Apply: Saga, Fan-out, Idempotency, Outbox
- Vẽ sơ đồ và giải thích từng quyết định

---

### Tuần 9: Monitoring và Observability

**Mục tiêu:** Biết cách monitor và debug messaging system trong production.

#### Ngày 1–3: Key Metrics và CloudWatch Alarms

**Đọc:**
- [ ] `09-monitoring/README.md`
- [ ] `09-monitoring/1-key-metrics.md`
- [ ] `09-monitoring/2-cloudwatch-alarms.md`

**Lab:** Tạo CloudWatch Dashboard với metrics từ SQS và Lambda. Cấu hình alarm khi DLQ có message.

#### Ngày 4–5: X-Ray Distributed Tracing

**Đọc:**
- [ ] `09-monitoring/3-xray-tracing.md`

**Lab:** Enable X-Ray tracing cho Lambda + SQS. Gửi vài message, xem Service Map trong X-Ray console.

#### Ngày 6–7: DLQ Monitoring

**Đọc:**
- [ ] `09-monitoring/4-dlq-monitoring.md`

**Bài tập:** Thiết lập monitoring đầy đủ cho hệ thống từ tuần trước (booking):
- Dashboard: SQS depth, Lambda errors, Iterator Age
- Alarms: DLQ > 0, SQS age > SLA, Step Functions failures

---

## Giai Đoạn 4: Phỏng Vấn (Tuần 10–13)

### Tuần 10: Review và Củng Cố

#### Ngày 1–2: Service Comparison Deep Dive

- [ ] Đọc lại `10-interview-prep/2-service-comparison.md`
- [ ] Làm bài tập: Cho 10 use case, chọn dịch vụ phù hợp và giải thích
- [ ] Học thuộc service limits quan trọng (SQS max size, FIFO TPS, Kinesis shard capacity)

#### Ngày 3–4: Top 25 Câu Hỏi

- [ ] Đọc `10-interview-prep/1-INTERVIEW_GUIDE.md`
- [ ] Câu 1–12: Che lời giải, tự trả lời
- [ ] Ghi lại những điểm bỏ sót

#### Ngày 5–7: Câu Hỏi Còn Lại

- [ ] Câu 13–25: Tự trả lời
- [ ] Tập trung vào câu hỏi thiết kế hệ thống (câu 19–25)

---

### Tuần 11: System Design Practice

#### Ngày 1–2: Bài 1 và Bài 2

- [ ] Đọc `10-interview-prep/3-system-design-scenarios.md`
- [ ] Bài 1 (E-commerce): Tự vẽ sơ đồ trước — sau đó so sánh
- [ ] Bài 2 (Clickstream): Tự tính số shard — kiểm tra lại

#### Ngày 3–4: Bài 3 và Bài 4

- [ ] Bài 3 (Microservices Saga): Thiết kế Step Functions state machine
- [ ] Bài 4 (Notification): Thiết kế với EventBridge

#### Ngày 5–7: Bài 5 và Tổng Hợp

- [ ] Bài 5 (Multi-Region DR): Thiết kế failover strategy
- [ ] Review tất cả 5 bài — tổng hợp điểm thường bị bỏ sót

---

### Tuần 12: Câu Chuyện STAR và Mock Interview

#### Ngày 1–3: Chuẩn Bị Câu Chuyện Cá Nhân

- [ ] Đọc `10-interview-prep/4-star-stories.md`
- [ ] Viết ra 3 câu chuyện từ kinh nghiệm cá nhân theo template STAR
- [ ] Đảm bảo mỗi câu chuyện có: số liệu cụ thể, quyết định kỹ thuật, bài học

#### Ngày 4–5: Luyện Nói

- [ ] Nói to từng câu chuyện — đặt đồng hồ 3 phút
- [ ] Record lại và nghe lại — cải thiện phần chưa rõ
- [ ] Nhờ bạn bè hỏi mock và cho feedback

#### Ngày 6–7: Mock Interview

Tìm partner để mock interview, mỗi người hỏi:
- 3 câu hỏi kỹ thuật cơ bản
- 1 câu hỏi system design
- 1 câu hỏi hành vi (STAR)

---

### Tuần 13: Tuần Cuối — Tinh Chỉnh

#### Ngày 1–3: Weak Spots (Điểm Yếu)

Xác định 5 chủ đề bạn chưa tự tin nhất → đọc lại + practice thêm.

**Chủ đề thường yếu:**
- Kinesis shard calculation (tính toán shard)
- EventBridge Archive & Replay
- Saga compensation design
- Cost optimization
- Multi-region architecture

#### Ngày 4–5: Final Review

- [ ] Đọc lại bảng so sánh dịch vụ (30 phút)
- [ ] Đọc lại Top 25 câu hỏi, chỉ xem câu hỏi — không xem đáp án
- [ ] Tự kiểm tra: Có thể trả lời tự nhiên không cần script?

#### Ngày 6–7: Nghỉ Ngơi và Chuẩn Bị Tinh Thần

- [ ] Không học nặng
- [ ] Review lại checklist trong `10-interview-prep/README.md`
- [ ] Chuẩn bị câu hỏi để hỏi lại người phỏng vấn

---

## 📊 Tracking Tiến Độ

Sao chép và theo dõi hàng tuần:

```
Tuần | Chủ Đề | Đọc | Lab | Tự Test | Ghi Chú
-----|--------|-----|-----|---------|--------
1    | SQS    | [ ] | [ ] | [ ]     |
2    | SNS    | [ ] | [ ] | [ ]     |
3    | EventBridge | [ ] | [ ] | [ ] |
4    | Step Functions | [ ] | [ ] | [ ] |
5    | Kinesis | [ ] | [ ] | [ ]   |
6    | MQ + AppSync | [ ] | [ ] | [ ] |
7    | Patterns 1 | [ ] | [ ] | [ ] |
8    | Patterns 2 | [ ] | [ ] | [ ] |
9    | Monitoring | [ ] | [ ] | [ ] |
10   | Review   | [ ] | —   | [ ]   |
11   | System Design | [ ] | — | [ ] |
12   | STAR + Mock | [ ] | — | [ ]  |
13   | Final Prep | [ ] | — | [ ]  |
```

---

## 💡 Mẹo Học Hiệu Quả

### Học Theo Cách Active (Chủ Động)

| Phương Pháp | Mức Ghi Nhớ |
|---|---|
| Đọc | 10% |
| Nghe | 20% |
| Xem hình ảnh | 30% |
| Demo / làm lab | 75% |
| Dạy lại người khác | 90% |

→ Luôn làm lab. Luôn giải thích lại cho người khác (hoặc giải thích một mình).

### Quy Tắc 3 Lần

Học một khái niệm mới:
1. **Lần 1**: Đọc tài liệu — hiểu concept
2. **Lần 2**: Làm lab — thấy nó hoạt động
3. **Lần 3**: Giải thích lại không cần tài liệu — true mastery (thành thạo thực sự)

### Tận Dụng AWS Free Tier

Hầu hết lab trong 90 ngày này nằm trong AWS Free Tier:
- SQS: 1 triệu request/tháng miễn phí
- SNS: 1 triệu notification/tháng miễn phí
- EventBridge: Tất cả AWS service events miễn phí
- Step Functions: 4,000 state transitions/tháng miễn phí
- Lambda: 1 triệu invocation/tháng miễn phí

**Kinesis và Amazon MQ KHÔNG có Free Tier** — xóa resource ngay sau lab!

### Khi Bị Stuck

1. Đọc lại tài liệu một lần nữa — thường lần đọc thứ hai hiểu hơn
2. Tìm official AWS documentation — nguồn chính xác nhất
3. Thử reproduce vấn đề trong lab nhỏ hơn
4. Hỏi community: AWS re:Post, Stack Overflow

---

## 🎯 Milestone (Cột Mốc) Cần Đạt

### Sau 30 Ngày

- [ ] Giải thích được SQS, SNS, EventBridge không cần ghi chú
- [ ] Tạo Fan-out Pattern thành thục
- [ ] Biết cấu hình DLQ và xử lý poison message

### Sau 60 Ngày

- [ ] Thiết kế được hệ thống event-driven cơ bản
- [ ] Implement Saga Pattern với Step Functions
- [ ] Tính được số Kinesis shard cho một yêu cầu cụ thể

### Sau 90 Ngày

- [ ] Trả lời tự tin 25 câu hỏi phỏng vấn
- [ ] Thiết kế hệ thống Order Processing trong 30 phút
- [ ] Kể 3 câu chuyện STAR chắc chắn về AWS Integration
- [ ] Giải thích trade-off giữa mọi cặp dịch vụ

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
