# So Sánh Dịch Vụ: SQS vs SNS vs EventBridge vs Kinesis

> Bảng so sánh toàn diện 4 dịch vụ messaging AWS, cây quyết định lựa chọn, và ví dụ kiến trúc thực tế.

---

## 📚 Mục Lục

1. [Tổng Quan Nhanh](#tổng-quan-nhanh)
2. [Bảng So Sánh Chi Tiết](#bảng-so-sánh-chi-tiết)
3. [Khi Nào Dùng Cái Nào?](#khi-nào-dùng-cái-nào)
4. [Cây Quyết Định](#cây-quyết-định)
5. [Use Case Thực Tế](#use-case-thực-tế)
6. [Kết Hợp Nhiều Dịch Vụ](#kết-hợp-nhiều-dịch-vụ)
7. [Pricing Tóm Tắt](#pricing-tóm-tắt)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Nhanh

| Dịch Vụ | Mô Hình | Mục Đích Chính | Từ Khóa |
|---|---|---|---|
| **Amazon SQS** | Queue (Hàng Đợi) — Point-to-Point | Task queuing, decoupling (tách rời) | Pull-based, durability, at-least-once |
| **Amazon SNS** | Pub/Sub (Nhà Phát/Người Đăng Ký) | Notifications, fan-out (khuếch tán) | Push-based, multi-protocol, broadcast |
| **Amazon EventBridge** | Event Bus (Xe Buýt Sự Kiện) | Event-driven architecture (kiến trúc hướng sự kiện) | Routing, filtering, schema, SaaS |
| **Amazon Kinesis** | Streaming (Luồng Dữ Liệu) | Real-time data processing (xử lý dữ liệu thời gian thực) | High-throughput, ordered, replay |

---

## Bảng So Sánh Chi Tiết

### Mô Hình Giao Tiếp

| Đặc Điểm | SQS | SNS | EventBridge | Kinesis |
|---|---|---|---|---|
| **Mô hình** | Queue (P2P) | Pub/Sub | Event Bus | Streaming |
| **Delivery** | Pull (kéo) | Push (đẩy) | Push | Pull |
| **Consumer** | Một consumer xử lý một message | Nhiều subscriber nhận cùng lúc | Nhiều target theo rule | Nhiều consumer đọc độc lập |
| **Message ordering (Thứ tự)** | FIFO Queue: có; Standard: không | SNS FIFO: có; Standard: không | Không đảm bảo | Có (trong cùng shard) |
| **Message retention (Lưu giữ)** | Tối đa 14 ngày | Không lưu (fire-and-forget) | Lưu qua Archive (tối đa vô hạn) | 1–365 ngày |
| **Replay (Phát lại)** | Không | Không | Có (EventBridge Archive) | Có |

### Hiệu Năng & Giới Hạn

| Đặc Điểm | SQS | SNS | EventBridge | Kinesis |
|---|---|---|---|---|
| **Throughput** | Không giới hạn (SQS Standard) | Không giới hạn | 10,000 events/s mặc định (có thể tăng) | 1MB/s hoặc 1,000 records/s per shard |
| **Message size** | Tối đa 256KB | Tối đa 256KB | Tối đa 256KB | Tối đa 1MB per record |
| **Latency (Độ trễ)** | Vài ms–vài giây | Vài ms | Vài ms–giây (near-real-time) | Dưới 70ms (Enhanced Fan-Out) |
| **Batch processing** | Tối đa 10 messages/batch | Không | Không (mỗi event riêng lẻ) | Tối đa 500 records/batch |

### Tính Năng Đặc Trưng

| Tính Năng | SQS | SNS | EventBridge | Kinesis |
|---|---|---|---|---|
| **Dead Letter Queue — DLQ** | ✅ Native | ✅ (qua SQS) | ✅ (Archive) | ✅ (Enhanced Error Handling) |
| **Message filtering (Lọc)** | ❌ | ✅ (Filter Policy) | ✅ (Event Pattern — chi tiết hơn) | ❌ (lọc ở consumer) |
| **Schema Registry (Kho Lược Đồ)** | ❌ | ❌ | ✅ (tích hợp sẵn) | ❌ |
| **Event routing (Định tuyến)** | ❌ | Hạn chế | ✅ (rule-based — dựa trên quy tắc) | ❌ |
| **SaaS integration** | ❌ | ❌ | ✅ (Salesforce, Zendesk...) | ❌ |
| **Visibility Timeout** | ✅ | ❌ | ❌ | ❌ |
| **Long polling** | ✅ | ❌ | ❌ | ✅ |
| **Cross-account** | ✅ (với policy) | ✅ | ✅ (Event Bus policy) | ✅ (với IAM) |

### Chi Phí Mô Hình

| Dịch Vụ | Tính Theo | Free Tier (Cấp Miễn Phí) |
|---|---|---|
| **SQS** | Số request (yêu cầu) | 1 triệu request/tháng |
| **SNS** | Số request + delivery endpoint | 1 triệu publish/tháng |
| **EventBridge** | Số event | 1 triệu event/tháng (custom bus) |
| **Kinesis** | Shard-hour (giờ-mảnh) + PUT payload | Không có Free Tier (có giá cố định) |

---

## Khi Nào Dùng Cái Nào?

### Dùng Amazon SQS Khi...

```
✅ Cần decouple (tách rời) producer và consumer
✅ Consumer cần xử lý ở tốc độ riêng (rate leveling — san phẳng tốc độ)
✅ Cần đảm bảo mỗi message chỉ xử lý bởi một worker
✅ Cần visibility timeout để tránh duplicate processing (xử lý trùng)
✅ Task queuing — background job processing (xử lý công việc nền)
✅ Cần buffer trước service có tốc độ xử lý giới hạn
✅ Load leveling (san phẳng tải) — hấp thụ traffic spike (đỉnh lưu lượng)
```

**Ví dụ:** Email sending queue, image resizing jobs, order fulfillment tasks.

### Dùng Amazon SNS Khi...

```
✅ Cần broadcast (phát sóng) tin nhắn đến nhiều subscriber cùng lúc
✅ Fan-out pattern — một event, nhiều downstream system (hệ thống hạ nguồn)
✅ Multi-protocol delivery (HTTP, SQS, Lambda, Email, SMS, Mobile Push)
✅ Cần push notification (thông báo đẩy) đến mobile/email
✅ Simple pub/sub không cần routing logic phức tạp
✅ Alert notification (thông báo cảnh báo) đến nhiều channel
```

**Ví dụ:** Order placed → notify inventory + shipping + email + analytics cùng lúc.

### Dùng Amazon EventBridge Khi...

```
✅ Cần event routing (định tuyến sự kiện) dựa trên nội dung event
✅ Event-driven microservices architecture
✅ Tích hợp với SaaS third-party (Salesforce, Shopify, Stripe...)
✅ Cần Schema Registry để quản lý cấu trúc event
✅ Cần Event Archive và Replay cho disaster recovery (khôi phục thảm họa)
✅ Cross-account event routing trong organization
✅ Scheduled events (sự kiện lên lịch) thay cron job
✅ AWS service events (CloudTrail, EC2, S3 events...)
```

**Ví dụ:** Microservices loosely coupled qua events, SaaS webhook routing.

### Dùng Amazon Kinesis Khi...

```
✅ High-throughput (thông lượng cao) real-time data streaming
✅ Cần replay data (phát lại dữ liệu) trong khoảng thời gian
✅ Ordered data processing (xử lý dữ liệu có thứ tự) quan trọng
✅ Multiple consumers (nhiều người tiêu dùng) cần đọc cùng stream độc lập
✅ Log ingestion (thu nạp log), clickstream analytics, IoT data
✅ Real-time anomaly detection (phát hiện bất thường thời gian thực)
✅ Tích hợp với Kinesis Data Analytics (Flink/SQL)
```

**Ví dụ:** Clickstream analysis, financial transaction monitoring, IoT sensor data.

---

## Cây Quyết Định

```
Bạn cần xử lý tin nhắn/sự kiện?
│
├─ Cần high-throughput real-time streaming (>MB/s)?
│   └─→ KINESIS DATA STREAMS
│
├─ Cần gửi đến nhiều recipient cùng lúc (broadcast)?
│   │
│   ├─ Cần routing phức tạp theo nội dung event?
│   │   └─→ EVENTBRIDGE
│   │
│   ├─ Tích hợp SaaS hoặc cần Schema Registry?
│   │   └─→ EVENTBRIDGE
│   │
│   └─ Fan-out đơn giản, multi-protocol (Email/SMS/HTTP)?
│       └─→ SNS
│
├─ Cần một consumer xử lý mỗi message?
│   └─→ SQS
│
├─ Cần decouple producer và consumer (tách rời)?
│   └─→ SQS
│
└─ Cần replay events (phát lại sự kiện)?
    ├─ Replay trong vài giờ/ngày → KINESIS
    └─ Replay dài hạn với Archive → EVENTBRIDGE
```

---

## Use Case Thực Tế

### Use Case 1: E-commerce Order Processing

```
Customer đặt hàng
       │
       ▼
   API Gateway
       │
       ▼
  Order Service ──────────────────────────────────────────────────────┐
       │                                                                │
       │ Publish event                                                  │
       ▼                                                                │
  EventBridge                                                           │
  (order.created)                                                       │
       │                                                                │
       ├──────────────────────────────────┐                            │
       │                                  │                            │
       ▼                                  ▼                            │
  SNS Topic                        Step Functions                       │
  (notification)              (orchestrate fulfillment)                 │
       │                                  │                            │
  ┌────┴────┐                   ┌─────────┼─────────┐                  │
  │         │                   │         │         │                   │
  ▼         ▼                   ▼         ▼         ▼                   │
Email     Push              Inventory  Payment  Shipping               │
Service  Notification       Service   Service   Service                │
                                │                                       │
                                ▼                                       │
                            SQS Queue ◄─────── Warehouse Worker       │
                          (warehouse jobs)                              │
                                │                                       │
                                ▼                                       │
                        Background Worker ─────────────────────────────┘
                         (pack & ship)
```

**Lý do chọn từng dịch vụ:**
- **EventBridge:** Route event từ Order Service đến nhiều downstream — schema registry quản lý `order.created` event
- **SNS:** Fan-out notification đến Email và Push — đơn giản, multi-protocol
- **Step Functions:** Orchestrate fulfillment workflow phức tạp — cần retry, rollback (Saga)
- **SQS:** Buffer warehouse jobs — worker xử lý theo pace riêng, cần visibility timeout

---

### Use Case 2: Real-time Analytics Pipeline

```
IoT Sensors / Clickstream / App Logs
              │
              ▼
    Kinesis Data Streams
    (partitioned by device/user)
              │
        ┌─────┴─────┐
        │           │
        ▼           ▼
  Lambda         Kinesis
  Consumer    Data Analytics
  (alerts)      (Flink)
        │           │
        ▼           ▼
     SNS          S3 / Redshift
  (alert noti)  (data warehouse)
        │
        ▼
  Email/PagerDuty
```

**Lý do chọn Kinesis thay vì SQS:**
- Cần ordered processing (thứ tự) theo device/user
- Multiple consumers đọc độc lập (Lambda + Analytics)
- Replay 24 giờ để reprocess nếu có lỗi
- Throughput cao (GB/s)

---

### Use Case 3: Microservices Event-Driven Architecture

```
Service A ──publish──► EventBridge Custom Bus
                               │
               ┌───────────────┼───────────────┐
               │               │               │
         Rule: user.*    Rule: order.*   Rule: payment.*
               │               │               │
               ▼               ▼               ▼
         Service B        Service C        Service D
         (profile)        (fulfillment)    (accounting)
```

**Lý do chọn EventBridge thay vì SNS:**
- Filter event phức tạp theo content (không chỉ theo topic)
- Các service subscribe theo pattern `user.*` hoặc `order.fulfilled`
- Schema Registry tự generate code từ schema
- Archive và replay cho testing và disaster recovery

---

## Kết Hợp Nhiều Dịch Vụ

### Pattern SNS + SQS Fan-out (Phổ Biến Nhất)

```
Publisher ──► SNS Topic ──┬──► SQS Queue A ──► Service A
                           ├──► SQS Queue B ──► Service B
                           └──► SQS Queue C ──► Service C
```

**Tại sao kết hợp SNS + SQS?**
- SNS: fan-out đến nhiều queue
- SQS: mỗi service xử lý ở tốc độ riêng, có DLQ, có retry

### Pattern EventBridge + SQS

```
Event Source ──► EventBridge ──► Rule ──► SQS ──► Lambda/Worker
```

**Khi nào dùng:** Cần routing logic phức tạp, nhưng consumer cần queue buffer.

### Pattern Kinesis + Lambda

```
Data Source ──► Kinesis ──► Lambda (trigger) ──► DynamoDB / S3
                      │
                      └──► Firehose ──► S3 (analytics)
```

---

## Pricing Tóm Tắt

### Amazon SQS

```
Standard Queue:
- $0.40 / 1 triệu requests
- First 1 triệu/tháng: miễn phí

FIFO Queue:
- $0.50 / 1 triệu requests
- Deduplication tính thêm
```

### Amazon SNS

```
- $0.50 / 1 triệu publish requests
- Delivery: tùy endpoint (Email $2/100k, SMS tính riêng)
- First 1 triệu publish: miễn phí
```

### Amazon EventBridge

```
Custom Event Bus:
- $1.00 / 1 triệu events
- First 1 triệu: miễn phí

Event Archive:
- $0.10 / GB / tháng

Pipes:
- $0.40 / 1 triệu event invocations
```

### Amazon Kinesis Data Streams

```
On-demand mode (Chế Độ Theo Nhu Cầu):
- $0.08 / 1 triệu records
- $0.04 / GB data ingested

Provisioned mode (Chế Độ Được Cấp Phát):
- $0.015 / shard-hour
- $0.08 / 1 triệu PUT payload units
```

**Lưu ý pricing quan trọng:**
- SQS và SNS tính theo request — phù hợp low-to-medium volume
- Kinesis tính shard-hour — có chi phí cố định ngay cả khi không dùng
- EventBridge rẻ cho sparse events (sự kiện thưa thớt), tốn hơn SQS cho high volume

---

## So Sánh Với Kafka (Nguồn Gốc Mở)

| Tính Năng | SQS | EventBridge | Kinesis | Apache Kafka |
|---|---|---|---|---|
| **Quản lý** | Fully managed | Fully managed | Fully managed | Tự quản lý (hoặc MSK) |
| **Ordering** | FIFO Queue | Không | Per-shard | Per-partition |
| **Retention** | 14 ngày | Archive tùy chỉnh | 365 ngày | Tùy chỉnh (không giới hạn) |
| **Replay** | Không | Có (Archive) | Có | Có |
| **Throughput** | Rất cao | 10k events/s (mở rộng được) | Cao (theo shard) | Rất cao |
| **Consumer groups** | Không | Không | Không native | Có (consumer group) |
| **Ecosystem** | AWS | AWS | AWS | Open-source, phong phú |

**Khi nào dùng Kafka (Amazon MSK — Managed Streaming for Kafka):**
- Cần consumer groups với offset management phức tạp
- Retention dài hạn không giới hạn
- Cần ecosystem Kafka (Kafka Connect, Kafka Streams)
- Migration từ on-premises Kafka

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Phân biệt SQS và SNS — khi nào dùng cái nào?

**Trả lời:**
> SQS là hàng đợi (queue) theo mô hình point-to-point — một message chỉ được xử lý bởi một consumer, consumer tự pull message. SNS là pub/sub — một message được push đến tất cả subscriber đồng thời.
>
> Dùng SQS khi cần: task queuing, decouple producer-consumer, mỗi task chỉ xử lý một lần.  
> Dùng SNS khi cần: broadcast đến nhiều hệ thống cùng lúc, fan-out pattern, notification đa kênh.
>
> Thường kết hợp SNS + SQS: SNS fan-out đến nhiều SQS queue, mỗi queue phục vụ một service với tốc độ riêng.

### Câu 2: EventBridge vs SNS — khi nào dùng EventBridge?

**Trả lời:**
> SNS phù hợp fan-out đơn giản — một topic, nhiều subscriber, hỗ trợ nhiều protocol.  
> EventBridge phù hợp event routing phức tạp — filter theo nội dung event (không chỉ theo topic), có Schema Registry, tích hợp SaaS, Archive và Replay.
>
> Rule của EventBridge có thể filter theo bất kỳ field nào trong event JSON — ví dụ chỉ route event `order.status == "fulfilled"` và `amount > 1000` đến service kế toán.  
> SNS Filter Policy đơn giản hơn — chỉ filter theo Message Attributes.

### Câu 3: Khi nào dùng Kinesis thay vì SQS?

**Trả lời:**
> SQS: mỗi message xử lý bởi một consumer, không replay, ordering chỉ với FIFO (tốc độ thấp hơn).  
> Kinesis: nhiều consumer đọc độc lập cùng stream, có replay theo thời gian, ordering trong shard.
>
> Chọn Kinesis khi:
> - Cần nhiều consumer xử lý cùng data stream độc lập (vd: Lambda tạo alert + Firehose lưu S3 cùng lúc)
> - Cần replay dữ liệu (ví dụ: ML model mới cần reprocess 7 ngày data)
> - High-throughput streaming (GB/s)
> - Ordering theo partition key quan trọng (ví dụ: mọi event của user X phải theo thứ tự)

### Câu 4: Làm sao tính chi phí cho SQS vs Kinesis?

**Trả lời:**
> SQS tính theo số request — phù hợp cho variable load (tải thay đổi), không tốn tiền khi không dùng.  
> Kinesis tính theo shard-hour — có chi phí cố định dù có data hay không ($0.015/shard/giờ ≈ $11/shard/tháng).
>
> Rule of thumb:
> - Volume thấp, bursty: SQS rẻ hơn
> - Volume cao, steady: Kinesis rẻ hơn theo throughput
> - Cần replay: chỉ Kinesis (hoặc EventBridge Archive) có tính năng này

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** SQS, SNS, EventBridge, Kinesis, so-sánh, lựa-chọn-dịch-vụ
