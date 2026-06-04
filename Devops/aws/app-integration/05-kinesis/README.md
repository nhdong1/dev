# Amazon Kinesis — Nền Tảng Streaming Thời Gian Thực

> **Amazon Kinesis** là tập hợp các dịch vụ xử lý data streaming (luồng dữ liệu) thời gian thực trên AWS, cho phép thu thập, xử lý và phân tích dữ liệu trong vài giây thay vì hàng giờ.

---

## 📚 Mục Lục Module

| File | Nội Dung | Độ Khó |
|---|---|---|
| [1-data-streams.md](./1-data-streams.md) | Kinesis Data Streams — KDS, Shard, Producer, Consumer | ⭐⭐⭐ |
| [2-firehose.md](./2-firehose.md) | Kinesis Data Firehose — Near-realtime Delivery | ⭐⭐ |
| [3-data-analytics.md](./3-data-analytics.md) | Kinesis Data Analytics — SQL và Apache Flink | ⭐⭐⭐ |
| [4-shard-management.md](./4-shard-management.md) | Quản Lý Shard — Tính Toán Capacity và Scaling | ⭐⭐⭐ |
| [5-enhanced-fanout.md](./5-enhanced-fanout.md) | Enhanced Fan-Out — Khuếch Tán Nâng Cao | ⭐⭐⭐ |

---

## 🎯 Tổng Quan Amazon Kinesis

### Kinesis Là Gì?

**Amazon Kinesis** giải quyết bài toán: làm thế nào để xử lý hàng triệu sự kiện mỗi giây từ nhiều nguồn (web logs, IoT sensors, transaction records) một cách real-time thay vì batch?

```
Nguồn Dữ Liệu                    Kinesis                    Đích Đến
─────────────                    ───────                    ────────
Web servers     ─────┐
Mobile apps     ─────┤           ┌─────────────┐            S3, Redshift
IoT sensors     ─────┼──────────▶│  Kinesis    │──────────▶ Lambda
Database CDC    ─────┤           │  Platform   │            OpenSearch
App logs        ─────┘           └─────────────┘            DynamoDB
```

### Bốn Dịch Vụ Kinesis

| Dịch Vụ | Mục Đích | Latency (Độ Trễ) | Quản Lý |
|---|---|---|---|
| **Kinesis Data Streams (KDS)** | Thu thập và xử lý custom | ~200ms | Tự quản lý consumer |
| **Kinesis Data Firehose** | Giao vận đến storage | 60–900 giây | Fully managed |
| **Kinesis Data Analytics** | Phân tích SQL/Flink | Gần thực tế | Fully managed |
| **Kinesis Video Streams** | Streaming video | Biến thiên | Fully managed |

---

## 🔍 Kinesis vs Các Dịch Vụ Khác

### Kinesis Data Streams vs SQS

| Tiêu Chí | Kinesis Data Streams | Amazon SQS |
|---|---|---|
| **Mô hình** | Streaming (nhiều consumer đọc song song) | Queue (một consumer nhận, xóa) |
| **Thứ tự** | Đảm bảo trong từng shard | Chỉ FIFO Queue |
| **Replay (Phát Lại)** | Có — đọc lại trong retention period | Không |
| **Throughput (Thông Lượng)** | 1MB/s write, 2MB/s read per shard | Không giới hạn (managed) |
| **Retention (Lưu Giữ)** | 24h–365 ngày | Tối đa 14 ngày |
| **Consumer tối đa** | Không giới hạn với Enhanced Fan-Out | N/A (nhiều workers cạnh tranh) |
| **Giá** | Theo shard-hour + PUT payload | Theo số request |
| **Dùng khi nào** | Analytics, IoT, log processing | Task queue, background jobs |

### Kinesis Data Streams vs Apache Kafka

| Tiêu Chí | Kinesis Data Streams | Apache Kafka |
|---|---|---|
| **Quản lý** | Fully managed | Tự quản lý (hoặc MSK) |
| **Sharding** | Shard (thêm/bớt thủ công hoặc on-demand) | Partition (cố định hơn) |
| **Retention** | Tối đa 365 ngày | Không giới hạn (disk) |
| **Ecosystem** | AWS-native, tích hợp với Firehose/Analytics | Rich ecosystem (Kafka Connect, KSQL) |
| **Phù hợp** | AWS-first workloads | Multi-cloud, on-premises |

---

## 🏗️ Kiến Trúc Kinesis Data Streams

### Thành Phần Cơ Bản

```
Producer (Nhà Sản Xuất)
    │
    │  PutRecord / PutRecords
    ▼
┌────────────────────────────────────┐
│         Kinesis Data Stream        │
│  ┌──────────┐  ┌──────────┐       │
│  │ Shard 1  │  │ Shard 2  │  ...  │
│  │[record1] │  │[record4] │       │
│  │[record2] │  │[record5] │       │
│  │[record3] │  │[record6] │       │
│  └──────────┘  └──────────┘       │
└────────────────────────────────────┘
    │
    │  GetRecords / SubscribeToShard
    ▼
Consumer (Người Tiêu Dùng)
    ├── Lambda
    ├── KCL Application (Kinesis Client Library)
    ├── Kinesis Data Analytics
    └── Kinesis Data Firehose
```

### Khái Niệm Quan Trọng

**Shard (Mảnh)**
- Đơn vị năng lực cơ bản của Kinesis Data Streams
- **Ingress (Nạp Vào):** 1 MB/giây hoặc 1.000 record/giây
- **Egress (Xuất Ra):** 2 MB/giây (shared across all consumers)
- Số shard quyết định throughput tổng của stream

**Partition Key (Khóa Phân Vùng)**
- Chuỗi xác định record thuộc shard nào
- Kinesis hash partition key → xác định shard
- Cùng partition key → cùng shard → đảm bảo thứ tự

**Sequence Number (Số Thứ Tự)**
- ID duy nhất của mỗi record trong shard
- Tăng dần theo thời gian
- Dùng để đọc từ vị trí cụ thể (checkpoint)

---

## 🔥 Kinesis Data Firehose — Giao Vận Đơn Giản

### Mô Hình Hoạt Động

```
Source (Nguồn)          Firehose              Destination (Đích)
──────────────          ────────              ──────────────────
Kinesis Data            Buffer (Đệm)          Amazon S3
Streams         ──────▶ Transform ──────────▶ Amazon Redshift
Direct PUT              (Lambda)              Amazon OpenSearch
MSK                                           Splunk / Datadog
```

### Đặc Điểm Chính

- **Near-real-time:** Độ trễ 60–900 giây (buffering interval — khoảng đệm)
- **Fully managed:** Không cần viết consumer code
- **Built-in transformation:** Lambda để transform record trước khi gửi
- **Data format conversion:** JSON → Parquet/ORC tự động
- **Error handling:** Backup dữ liệu lỗi sang S3 tự động

---

## 📊 Kinesis Data Analytics — Phân Tích Streaming

### Hai Engine Xử Lý

**SQL Application**
- Truy vấn SQL tiêu chuẩn trên data stream
- Phù hợp với phân tích đơn giản, aggregation (tổng hợp) theo window (cửa sổ thời gian)
- Ít linh hoạt hơn Flink

**Apache Flink Application**
- Framework xử lý streaming mạnh mẽ
- Stateful processing (Xử Lý Có Trạng Thái), complex event processing
- Hỗ trợ Java và Python
- Fault tolerance (Chịu Lỗi) với checkpointing

### Window Operations (Phép Toán Cửa Sổ)

```sql
-- Tumbling Window (Cửa Sổ Lăn): không chồng lấp
SELECT COUNT(*) AS event_count
FROM stream
GROUP BY STEP(stream.ROWTIME BY INTERVAL '1' MINUTE)

-- Sliding Window (Cửa Sổ Trượt): chồng lấp nhau
SELECT AVG(temperature) AS avg_temp
FROM sensor_stream
WHERE ROWTIME BETWEEN CURRENT_ROW AND INTERVAL '5' MINUTE PRECEDING
```

---

## ⚙️ Khi Nào Dùng Dịch Vụ Nào?

```
Bạn cần real-time processing với nhiều consumer?
    └─▶ Kinesis Data Streams (KDS)
         │
         ├── Cần giao vận vào S3/Redshift mà không cần code?
         │       └─▶ Kinesis Data Firehose
         │
         ├── Cần phân tích SQL/Flink trực tiếp trên stream?
         │       └─▶ Kinesis Data Analytics
         │
         └── Cần nhiều consumer đọc song song tốc độ cao?
                 └─▶ KDS + Enhanced Fan-Out
```

### Ví Dụ Use Case Thực Tế

| Use Case | Dịch Vụ | Lý Do |
|---|---|---|
| Clickstream analytics (Phân Tích Luồng Click) | KDS + Analytics | Real-time, nhiều consumer |
| Log aggregation vào S3 | Firehose | Đơn giản, không cần code |
| IoT sensor monitoring | KDS + Lambda | Custom processing logic |
| Fraud detection (Phát Hiện Gian Lận) | KDS + Analytics (Flink) | Real-time ML inference |
| CDC (Change Data Capture) → Data warehouse | KDS → Firehose → Redshift | Pipeline cuối đến cuối |

---

## 💡 Giới Hạn Quan Trọng Cần Nhớ

| Giới Hạn | Giá Trị | Ghi Chú |
|---|---|---|
| Kích thước record tối đa | 1 MB | Per record |
| Write throughput per shard | 1 MB/s hoặc 1.000 records/s | Cái nào đạt trước |
| Read throughput per shard | 2 MB/s | Shared với tất cả consumers |
| Số shard mặc định tối đa | 500 (có thể tăng) | Per stream |
| Retention tối thiểu | 24 giờ | Mặc định |
| Retention tối đa | 365 ngày | Tính phí thêm |
| Firehose buffer size | 1–128 MB | Tùy destination |
| Firehose buffer interval | 60–900 giây | Tùy destination |

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Phân biệt Kinesis Data Streams và Kinesis Firehose?**
   → KDS: custom consumer, real-time sub-second; Firehose: fully-managed, near-real-time, chỉ giao vào storage

2. **Shard là gì và cách tính số shard cần thiết?**
   → Xem chi tiết: [4-shard-management.md](./4-shard-management.md)

3. **Khi nào dùng Kinesis thay vì SQS?**
   → Kinesis khi cần: replay, nhiều consumer đọc song song, ordering strict, analytics streaming

4. **Enhanced Fan-Out khác gì so với standard consumer?**
   → Xem chi tiết: [5-enhanced-fanout.md](./5-enhanced-fanout.md)

5. **Partition Key ảnh hưởng gì đến hiệu năng?**
   → Hot shard (Shard Nóng) nếu partition key không đa dạng → giải pháp: random suffix hoặc composite key

---

## 🔗 Liên Kết Tham Khảo

- [Kinesis Data Streams vs SQS: Khi Nào Dùng Gì?](../08-patterns/1-service-comparison.md)
- [Monitoring Kinesis: Iterator Age và Metrics](../09-monitoring/1-key-metrics.md)
- [So Sánh Toàn Diện Dịch Vụ AWS](../08-patterns/1-service-comparison.md)

---

**Tiếp Theo:** Bắt đầu với [1-data-streams.md](./1-data-streams.md) — hiểu sâu về Kinesis Data Streams và cơ chế Shard.
