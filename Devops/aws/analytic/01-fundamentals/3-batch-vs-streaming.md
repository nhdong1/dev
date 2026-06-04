# 3 — Batch vs Streaming: Xử Lý Theo Lô vs Luồng Thời Gian Thực

> Lựa chọn giữa Batch Processing (Xử Lý Theo Lô) và Stream Processing (Xử Lý Luồng) là quyết định kiến trúc quan trọng nhất khi thiết kế data pipeline. Mỗi cách có trade-off riêng và phù hợp với bài toán khác nhau.

## 📚 Mục Lục

1. [Định Nghĩa Cơ Bản](#định-nghĩa-cơ-bản)
2. [Batch Processing — Xử Lý Theo Lô](#batch-processing--xử-lý-theo-lô)
3. [Stream Processing — Xử Lý Luồng](#stream-processing--xử-lý-luồng)
4. [So Sánh Chi Tiết](#so-sánh-chi-tiết)
5. [Micro-batch — Giải Pháp Trung Gian](#micro-batch--giải-pháp-trung-gian)
6. [Quyết Định Kiến Trúc — Khi Nào Chọn Cái Nào](#quyết-định-kiến-trúc)
7. [AWS Services Cho Mỗi Loại](#aws-services-cho-mỗi-loại)
8. [Ví Dụ Kiến Trúc Thực Tế](#ví-dụ-kiến-trúc-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Định Nghĩa Cơ Bản

### Batch Processing — Xử Lý Theo Lô
```
Tích lũy dữ liệu trong một khoảng thời gian → Xử lý toàn bộ cùng một lúc

[==data==data==data==data==] → PROCESS → [results]
 (thu thập 1 giờ)              (chạy)     (output)
```

### Stream Processing — Xử Lý Luồng
```
Xử lý từng sự kiện ngay khi nó đến

event → PROCESS → result
event → PROCESS → result
event → PROCESS → result
(liên tục, real-time)
```

### Micro-batch — Kết Hợp
```
Tích lũy dữ liệu trong khoảng thời gian rất ngắn (giây) → Xử lý

[=data=] → PROCESS → [results]   (mỗi 30 giây)
[=data=] → PROCESS → [results]
[=data=] → PROCESS → [results]
```

---

## Batch Processing — Xử Lý Theo Lô

### Cách Hoạt Động

```
Timeline:

00:00                    01:00                    02:00
  │                        │                        │
  ├──── Thu thập data ─────┤                        │
  │     (1 giờ)            │                        │
  │                        ├──── Batch Job runs ────┤
  │                        │    (xử lý 1 giờ data)  │
  │                        │                        │
  │                        └──→ Output ready ~02:30  │

Latency = 1h (collection) + 30min (processing) = ~1.5h từ event đến result
```

### Đặc Điểm Kỹ Thuật

| Đặc Điểm           | Mô Tả                                                        |
| ------------------ | ------------------------------------------------------------ |
| **Latency**        | Cao — phút đến hàng giờ, tùy schedule                       |
| **Throughput**     | Cao — xử lý volume lớn hiệu quả                             |
| **Complexity**     | Thấp — đơn giản hơn stream processing                       |
| **Cost**           | Thấp hơn — chạy on-demand, không cần hạ tầng liên tục       |
| **Error handling** | Dễ — có thể retry cả job, reprocess từ đầu                  |
| **State management** | Đơn giản — không cần quản lý state phức tạp               |

### Use Cases Phù Hợp

**Batch xử lý tốt nhất khi:**
- Báo cáo end-of-day, end-of-month
- ETL pipeline nạp dữ liệu vào data warehouse
- Xử lý file lớn (log files, CSV uploads)
- ML model training trên historical data
- Billing và invoicing (hóa đơn) hàng tháng
- Data archiving và compliance reports
- Backfilling (lấp đầy) dữ liệu lịch sử

**Ví dụ thực tế:**
```
✅ "Mỗi đêm lúc 2:00 AM, tổng hợp doanh thu ngày và load vào Redshift"
✅ "Hàng tuần, train lại recommendation model với data 7 ngày gần nhất"
✅ "Mỗi tháng, generate báo cáo compliance cho toàn bộ transactions"
✅ "Crawl giá sản phẩm đối thủ mỗi 6 tiếng"
```

### Lợi Thế Của Batch

1. **Đơn giản hơn** — Không cần xử lý late data, out-of-order events
2. **Chi phí thấp hơn** — Chỉ chạy khi cần, tắt khi xong (Spot Instances)
3. **Dễ debug** — Log đầy đủ, có thể reproduce lỗi
4. **Throughput cao** — Xử lý GB/TB data hiệu quả
5. **Tích hợp dễ** — Standard SQL, PySpark quen thuộc

### Hạn Chế Của Batch

1. **Latency cao** — Insights cũ khi đến tay người dùng
2. **Không phù hợp real-time** — Không thể detect fraud ngay lập tức
3. **Resource spikes** — Load cao khi batch job chạy
4. **Reprocessing tốn kém** — Phải chạy lại toàn bộ job nếu có lỗi

---

## Stream Processing — Xử Lý Luồng

### Cách Hoạt Động

```
Event stream liên tục:

t=0s  → [event1] → process → result1
t=1s  → [event2] → process → result2
t=2s  → [event3] → process → result3
...
(Không bao giờ dừng)

Latency = milliseconds đến seconds (tùy implementation)
```

### Các Khái Niệm Quan Trọng Trong Stream Processing

#### 1. Event Time vs Processing Time (Thời Gian Sự Kiện vs Thời Gian Xử Lý)
```
Event Time:      Khi sự kiện thực sự xảy ra (timestamp trong data)
Processing Time: Khi hệ thống xử lý sự kiện đó

Vấn đề: Network delay, mobile offline → events đến muộn (late events)

Event:    [t=10:00] ─────────────────────────→ Arrives at t=10:05
                                                 (5 phút trễ)
```

#### 2. Windows — Cửa Sổ Thời Gian
Stream processing thường cần aggregate theo khoảng thời gian:

```
Tumbling Window (Cửa Sổ Lăn — không overlap):
├─[10:00-10:05]─┤─[10:05-10:10]─┤─[10:10-10:15]─┤
  (đóng, process)  (đóng, process)  (đang mở)

Sliding Window (Cửa Sổ Trượt — có overlap):
├──────────[10:00-10:10]──────────┤
         ├──────────[10:05-10:15]──────────┤
                  ├──────────[10:10-10:20]──────────┤

Session Window (Cửa Sổ Phiên — theo activity):
├──[user active]──┤ (inactive 30min gap) ├──[user active]──┤
  session 1                               session 2
```

#### 3. Watermarks — Dấu Thời Gian Nước
```
Watermark = thời điểm hệ thống "tuyên bố" đã nhận đủ events đến một thời điểm nhất định

Ví dụ: Watermark = event_time - 5 phút
→ Khi watermark = 10:00, đóng window [9:55-10:00]
→ Events đến trễ hơn 5 phút sẽ bị drop hoặc xử lý riêng
```

#### 4. Exactly-once Semantics (Xử Lý Chính Xác Một Lần)
```
At-most-once:  Có thể mất dữ liệu (0 hoặc 1 lần xử lý)
At-least-once: Không mất dữ liệu, có thể duplicate (1+ lần)
Exactly-once:  Mỗi event xử lý đúng 1 lần — khó nhất, đắt nhất

Kinesis + Lambda: At-least-once (cần idempotent consumer)
Apache Flink:    Exactly-once với checkpointing
```

### Đặc Điểm Kỹ Thuật

| Đặc Điểm           | Mô Tả                                                        |
| ------------------ | ------------------------------------------------------------ |
| **Latency**        | Thấp — milliseconds đến seconds                              |
| **Throughput**     | Cao — nhưng cần scale đúng cách                             |
| **Complexity**     | Cao — state management, late events, exactly-once           |
| **Cost**           | Cao hơn — cần hạ tầng chạy liên tục 24/7                   |
| **Error handling** | Phức tạp — partial failures, dead letter queues             |
| **State management** | Phức tạp — cần distributed state store                    |

### Use Cases Phù Hợp

**Stream processing tốt nhất khi:**
- Fraud detection (phát hiện gian lận) real-time
- Real-time monitoring và alerting
- Clickstream analysis (phân tích luồng click) của user
- IoT sensor data processing
- Live leaderboard (bảng xếp hạng trực tiếp)
- Stock price alerts
- Real-time recommendation (gợi ý thời gian thực)

**Ví dụ thực tế:**
```
✅ "Chặn giao dịch gian lận trong < 100ms"
✅ "Alert khi error rate vượt 5% trong 60 giây qua"
✅ "Cập nhật bảng xếp hạng game mỗi 5 giây"
✅ "Hiện thị số người đang xem video live"
✅ "Phát hiện thiết bị IoT bị lỗi ngay lập tức"
```

### Lợi Thế Của Stream Processing

1. **Low latency** — Insights ngay lập tức
2. **Real-time actions** — Phản ứng ngay (block fraud, send alert)
3. **Continuous insights** — Data luôn fresh
4. **Event-driven** — Trigger actions tự động

### Hạn Chế Của Stream Processing

1. **Phức tạp hơn** — Late events, out-of-order, state management
2. **Chi phí cao hơn** — Hạ tầng chạy 24/7
3. **Khó debug** — Stateful, distributed, không reproducible dễ dàng
4. **Exactly-once khó** — Đòi hỏi thiết kế cẩn thận

---

## So Sánh Chi Tiết

| Tiêu Chí               | Batch Processing         | Stream Processing               |
| ---------------------- | ------------------------ | ------------------------------- |
| **Latency**            | Phút đến giờ             | Milliseconds đến giây           |
| **Throughput**         | Rất cao (TB+)            | Cao (MB-GB/s)                   |
| **Complexity**         | Thấp                     | Cao                             |
| **Cost**               | Thấp (on-demand)         | Cao hơn (always-on)             |
| **Use case**           | Báo cáo, ETL, training   | Fraud, monitoring, real-time    |
| **Data size**          | Tốt với data lớn         | Tốt với event-driven data       |
| **Error recovery**     | Dễ (re-run job)          | Phức tạp (checkpoint/replay)    |
| **Data completeness**  | Đầy đủ trước khi process | Có thể thiếu (late events)      |
| **AWS Service**        | Glue, EMR                | Kinesis, MSK, Kinesis Analytics |

---

## Micro-batch — Giải Pháp Trung Gian

**Micro-batch** là pattern kết hợp — tích lũy dữ liệu trong khoảng thời gian rất ngắn (5-60 giây) rồi xử lý như batch nhỏ.

```
Micro-batch với 30-second windows:

t=0s  ┌─────────────────────────┐
      │ events arriving...      │
t=30s └─────────────────────────┘ → PROCESS → results
      ┌─────────────────────────┐
      │ events arriving...      │
t=60s └─────────────────────────┘ → PROCESS → results
```

**Khi nào dùng micro-batch:**
- Cần "gần real-time" nhưng không cần ms latency
- Muốn đơn giản hơn full streaming
- Kinesis Firehose buffer (60s-15min)
- Spark Structured Streaming

**AWS Services:** Kinesis Firehose (buffering), Spark Structured Streaming trên EMR

---

## Quyết Định Kiến Trúc

### Decision Framework (Khung Quyết Định)

```
Câu hỏi 1: Latency requirement (yêu cầu độ trễ) là bao nhiêu?
  │
  ├──→ < 1 giây          → STREAMING (Kinesis, MSK + Flink)
  ├──→ 1 giây - 5 phút   → MICRO-BATCH (Kinesis Firehose, Spark)
  └──→ > 5 phút          → BATCH (Glue, EMR)

Câu hỏi 2: Actions có cần trigger ngay lập tức không?
  ├──→ Có (fraud block, alert) → STREAMING
  └──→ Không                   → BATCH hoặc MICRO-BATCH

Câu hỏi 3: Budget và team complexity?
  ├──→ Nhỏ/trung bình     → BATCH (đơn giản, rẻ hơn)
  └──→ Lớn, có expertise  → STREAMING (nếu business cần)
```

### Bảng Quyết Định Nhanh

| Scenario                                      | Khuyến Nghị      | Lý Do                                |
| --------------------------------------------- | ---------------- | ------------------------------------ |
| Báo cáo doanh thu hàng ngày                   | **Batch**        | Latency không quan trọng             |
| Phát hiện gian lận thanh toán                 | **Streaming**    | Cần phản hồi trong < 100ms           |
| ETL từ CRM vào data warehouse                 | **Batch**        | Dữ liệu chỉ cần fresh hàng ngày      |
| Monitoring server health                      | **Streaming**    | Alert ngay khi CPU > 90%             |
| ML model training hàng tuần                   | **Batch**        | Historical data, không cần real-time |
| Real-time leaderboard game                    | **Streaming**    | User thấy kết quả ngay               |
| Log aggregation cho compliance                | **Batch**        | Accuracy > speed                     |
| IoT sensor anomaly detection                  | **Streaming**    | Phát hiện sự cố ngay lập tức         |
| Recommendation cho trang chủ                  | **Micro-batch**  | Fresh đủ nếu cập nhật mỗi 5 phút     |

---

## AWS Services Cho Mỗi Loại

### Batch Processing Stack

```
Orchestration:   AWS Step Functions / MWAA (Airflow)
                          │
Ingestion:       AWS DMS / S3 / AWS DataSync
                          │
Processing:      AWS Glue (PySpark) / Amazon EMR
                          │
Storage:         S3 (Parquet) / Amazon Redshift
                          │
Serving:         Athena / Redshift / QuickSight
```

### Stream Processing Stack

```
Sources:         Applications / IoT / Mobile
                          │
Ingestion:       Amazon Kinesis Data Streams / Amazon MSK
                          │
Processing:      Kinesis Data Analytics (Flink)
                 OR Lambda (for simple transforms)
                 OR EMR Flink / Spark Streaming
                          │
Storage:         S3 / DynamoDB / OpenSearch / Redshift
                          │
Serving:         OpenSearch Dashboards / Lambda API / QuickSight
```

### Service Selection Guide

| Batch Service        | Khi Nào Dùng                                          |
| -------------------- | ----------------------------------------------------- |
| **AWS Glue**         | ETL đơn giản đến trung bình, không muốn quản lý cluster |
| **Amazon EMR**       | Big data lớn, cần Spark/Hive/Presto, kiểm soát cluster |
| **AWS Batch**        | Docker-based batch jobs, HPC workloads                |
| **Lambda**           | Lightweight processing, event-driven triggers         |

| Stream Service            | Khi Nào Dùng                                        |
| ------------------------- | --------------------------------------------------- |
| **Kinesis Data Streams**  | AWS-native streaming, custom consumers, replay      |
| **Kinesis Firehose**      | Managed delivery, không cần code consumer           |
| **Kinesis Analytics**     | SQL/Flink trên streams, không quản lý cluster       |
| **Amazon MSK**            | Kafka-compatible, multi-consumer, open ecosystem    |

---

## Ví Dụ Kiến Trúc Thực Tế

### Ví Dụ 1: Ride-sharing App (Ứng Dụng Đặt Xe)

```
STREAMING (cần real-time):
Driver location updates → Kinesis → Lambda → DynamoDB
(cập nhật vị trí tài xế mỗi 5 giây)

BATCH (có thể chậm):
Trip data → S3 → Glue ETL (hàng đêm) → Redshift → QuickSight
(báo cáo doanh thu, phân tích tuyến đường phổ biến)
```

### Ví Dụ 2: E-commerce Platform

```
STREAMING:
User clicks → Kinesis → Kinesis Analytics → Real-time
                                            Recommendations

Purchase events → Kinesis → Lambda → Fraud Score API
                                     → Block if suspicious

BATCH:
Orders DB → Glue ETL (2:00 AM daily) → Redshift
                                        → QuickSight Reports
                                        → SageMaker Training Data
```

### Ví Dụ 3: DevOps Log Analytics

```
STREAMING (alerts):
App logs → CloudWatch Logs → Kinesis → Lambda → PagerDuty
                                       (error rate > threshold)

BATCH (investigation):
App logs → S3 → Athena (ad-hoc query sự cố)
               → OpenSearch (full-text search logs)
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Khi nào bạn chọn Batch, khi nào chọn Streaming?**

> **Trả lời:** Tôi đặt câu hỏi về latency requirement trước: nếu business có thể chờ vài phút đến vài giờ để có insights, Batch là lựa chọn tốt hơn vì đơn giản hơn và rẻ hơn. Nếu cần phản ứng ngay lập tức (fraud detection < 100ms, alert khi server down), phải dùng Streaming. Với nhiều hệ thống thực tế, tôi dùng cả hai: Streaming cho operational decisions, Batch cho analytical reports.

**Q2: Giải thích late events trong Stream Processing và cách xử lý?**

> **Trả lời:** Late events (sự kiện đến muộn) xảy ra khi event có timestamp cũ nhưng đến processor sau — ví dụ mobile app offline 5 phút rồi mới sync. Xử lý bằng: (1) **Watermarks** — chờ đến khi watermark vượt qua event_time + allowed_lateness mới đóng window; (2) **Allowed lateness** trong Apache Flink — ví dụ chấp nhận events trễ đến 5 phút; (3) **Dead letter queue** — events trễ quá mức gửi vào SQS DLQ để xử lý riêng; (4) **Idempotent writes** — nếu late event làm update, đảm bảo không duplicate trong target.

**Q3: Exactly-once trong Kinesis có thể đạt được không?**

> **Trả lời:** Kinesis Data Streams native là at-least-once. Để đạt exactly-once semantics, cần: (1) **Idempotent consumers** — dùng sequence number của Kinesis để detect và skip duplicates; (2) **Conditional writes** — DynamoDB conditional put để tránh duplicate inserts; (3) **Kinesis Data Analytics với Apache Flink** — Flink có built-in exactly-once với checkpointing; (4) Hoặc dùng **MSK (Apache Kafka)** với Kafka transactions nếu cần exactly-once end-to-end.

**Q4: Thiết kế real-time fraud detection system xử lý 10,000 transactions/giây?**

> **Trả lời:**
> - **Ingestion:** Kinesis Data Streams với 10 shards (1,000 TPS/shard × 10 = 10,000 TPS)
> - **Processing:** Kinesis Data Analytics (Flink) — sliding window 5 phút, compute features: transaction frequency, geographic anomaly, amount deviation
> - **ML Inference:** Lambda gọi SageMaker real-time endpoint (< 20ms p99)
> - **Action:** Nếu score > threshold → DynamoDB write (block list) + SNS notification
> - **Monitoring:** CloudWatch metrics cho latency < 100ms end-to-end
> - **Fallback:** Nếu ML endpoint down → rule-based fallback trong Lambda

---

## 💡 Key Takeaways

1. **Latency là tiêu chí quan trọng nhất** — xác định rõ SLA trước khi chọn kiến trúc
2. **Batch đơn giản và rẻ hơn** — không nên over-engineer với streaming nếu không cần
3. **Streaming phức tạp hơn** — late events, state management, exactly-once đều là thách thức thực tế
4. **Nhiều hệ thống dùng cả hai** — Lambda Architecture: Batch cho accuracy, Streaming cho speed
5. **Micro-batch là middle ground** — Kinesis Firehose, Spark Structured Streaming
6. **AWS có đủ services** — Glue/EMR cho Batch, Kinesis/MSK cho Streaming

---

**← [2-data-pipeline-concepts.md](./2-data-pipeline-concepts.md)** | **→ [4-data-lake-vs-warehouse.md](./4-data-lake-vs-warehouse.md)**
