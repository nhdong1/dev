# 🌊 Amazon Kinesis — Real-time Data Streaming (Luồng Dữ Liệu Thời Gian Thực)

> Amazon Kinesis là nền tảng streaming data của AWS, cho phép thu thập, xử lý và phân tích dữ liệu thời gian thực ở bất kỳ quy mô nào — từ vài KB/s đến hàng TB/giờ.

## 📚 Mục Lục Module

1. [Tổng Quan Kinesis](#tổng-quan-kinesis)
2. [Các Dịch Vụ Kinesis](#các-dịch-vụ-kinesis)
3. [Khi Nào Dùng Kinesis](#khi-nào-dùng-kinesis)
4. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
5. [Các File Trong Module](#các-file-trong-module)
6. [Câu Hỏi Phỏng Vấn Nhanh](#câu-hỏi-phỏng-vấn-nhanh)

---

## 🎯 Tổng Quan Kinesis

**Amazon Kinesis** là họ dịch vụ AWS dành cho **Stream Processing** (Xử Lý Luồng) — xử lý dữ liệu liên tục ngay khi phát sinh, thay vì chờ tích lũy rồi xử lý theo lô (Batch Processing).

### Tại Sao Cần Streaming?

| Tình Huống                              | Batch (Theo Lô)              | Streaming (Thời Gian Thực)         |
| --------------------------------------- | ----------------------------- | ------------------------------------ |
| Phát hiện gian lận thẻ tín dụng        | Phát hiện sau vài giờ        | Phát hiện trong vòng giây           |
| Dashboard doanh thu theo thời gian thực | Cập nhật mỗi đêm             | Cập nhật liên tục                   |
| Cảnh báo cảm biến IoT bất thường        | Xử lý sau, có thể quá muộn  | Cảnh báo ngay lập tức               |
| Personalization (Cá Nhân Hóa) nội dung | Dựa trên hành vi hôm qua     | Dựa trên hành vi trong vài phút qua |

### Vị Trí Kinesis Trong Hệ Sinh Thái AWS Analytics

```
Data Sources (Nguồn Dữ Liệu)
    │
    ▼
┌─────────────────────────────────────────────────┐
│              INGESTION LAYER (Tầng Thu Nạp)      │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Kinesis Data    │  │   Kinesis Data       │  │
│  │  Streams (KDS)   │  │   Firehose (KDF)     │  │
│  │  (Tùy chỉnh cao) │  │  (Fully managed)     │  │
│  └────────┬─────────┘  └──────────┬───────────┘  │
└───────────┼────────────────────────┼──────────────┘
            │                        │
            ▼                        ▼
┌───────────────────┐    ┌───────────────────────┐
│  PROCESSING LAYER │    │   STORAGE LAYER       │
│  (Tầng Xử Lý)    │    │   (Tầng Lưu Trữ)     │
│  ┌─────────────┐  │    │  S3, Redshift,        │
│  │  Kinesis    │  │    │  OpenSearch,          │
│  │  Analytics  │  │    │  DynamoDB             │
│  │  (KDA/Flink)│  │    └───────────────────────┘
│  └─────────────┘  │
│  Lambda, EC2      │
└───────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│  SERVING LAYER (Tầng Phục Vụ)    │
│  QuickSight, OpenSearch, API GW   │
└───────────────────────────────────┘
```

---

## 🗂️ Các Dịch Vụ Kinesis

Amazon Kinesis gồm **4 dịch vụ chính**:

### 1. Kinesis Data Streams — KDS (Luồng Dữ Liệu Kinesis)

**Định nghĩa:** Dịch vụ streaming có độ trễ thấp, cho phép nhiều consumer (người tiêu thụ) đọc dữ liệu cùng lúc với khả năng tùy chỉnh cao.

```
Producer → KDS → Consumer 1 (Lambda)
               → Consumer 2 (EC2 application)
               → Consumer 3 (Kinesis Analytics)
```

**Đặc điểm nổi bật:**
- Dữ liệu được lưu trữ 24 giờ (mặc định) đến 365 ngày
- Đơn vị scaling: **Shard** (Mảnh) — mỗi shard xử lý 1 MB/s đọc vào, 2 MB/s đọc ra
- Hỗ trợ nhiều consumer đồng thời
- Thứ tự bản ghi được đảm bảo trong một shard

### 2. Kinesis Data Firehose — KDF (Vòi Dữ Liệu Kinesis)

**Định nghĩa:** Dịch vụ **fully managed** (Được Quản Lý Hoàn Toàn) để deliver (Chuyển Phát) streaming data vào các điểm đến như S3, Redshift, OpenSearch — không cần viết consumer code.

```
Producer → KDF → Transformation (Lambda tùy chọn)
               → Destination (S3 / Redshift / OpenSearch / Splunk)
```

**Đặc điểm nổi bật:**
- Không cần quản lý shard hay consumer
- Tự động scaling
- Buffering (Đệm) dữ liệu trước khi ghi (60s hoặc 5MB)
- Tích hợp sẵn với Lambda để transform (Biến Đổi) dữ liệu

### 3. Kinesis Data Analytics — KDA (Phân Tích Dữ Liệu Kinesis)

**Định nghĩa:** Dịch vụ cho phép chạy **Apache Flink** hoặc **SQL** trực tiếp trên streaming data mà không cần quản lý cluster.

```
KDS / KDF → KDA (Flink Application) → Destination
                                      (S3, KDS, Lambda, ...)
```

**Đặc điểm nổi bật:**
- Hỗ trợ Apache Flink (Xử Lý Luồng Phân Tán) — windowing, joins, aggregations
- Serverless — tự động scale
- Tích hợp với KDS và KDF làm nguồn đầu vào

### 4. Kinesis Video Streams — KVS (Luồng Video Kinesis)

**Định nghĩa:** Dịch vụ chuyên biệt để streaming **video** từ thiết bị kết nối vào AWS để xử lý, phân tích và lưu trữ.

**Use case:** Camera an ninh, xe tự hành, thiết bị IoT có camera.

> **Lưu ý phỏng vấn:** KVS ít được hỏi hơn 3 dịch vụ còn lại. Tập trung vào KDS, KDF, KDA.

---

## 🎯 Khi Nào Dùng Kinesis

### So Sánh Nhanh KDS vs KDF

| Tiêu Chí                        | KDS (Data Streams)             | KDF (Firehose)                  |
| ------------------------------- | ------------------------------ | ------------------------------- |
| **Độ trễ**                      | Real-time (< 1 giây)           | Near real-time (60s - 15 phút) |
| **Quản lý**                     | Tự quản lý shard               | Fully managed                   |
| **Consumer**                    | Nhiều consumer tùy chỉnh       | 1 destination per stream        |
| **Lưu trữ dữ liệu**             | 1-365 ngày (replay được)       | Không lưu, chỉ deliver          |
| **Chi phí**                     | Theo shard-giờ                 | Theo dung lượng dữ liệu (GB)   |
| **Khi nào dùng**                | Cần custom processing          | Cần đưa data vào S3/Redshift   |

### Quyết Định Nhanh

```
Cần xử lý từng event ngay lập tức?
├── Có → Kinesis Data Streams (KDS)
└── Không → Kinesis Data Firehose (KDF)

Cần nhiều consumer khác nhau đọc cùng stream?
├── Có → KDS (hỗ trợ fan-out)
└── Không → KDF (simpler, cheaper)

Cần replay dữ liệu từ quá khứ?
├── Có → KDS (retention 1-365 ngày)
└── Không → KDF

Cần transform dữ liệu phức tạp (windowing, join)?
└── → Kinesis Data Analytics (KDA/Flink)
```

---

## 🏗️ Kiến Trúc Tổng Thể

### Pattern 1: Real-time Event Processing (Xử Lý Sự Kiện Thời Gian Thực)

```
[Web App / Mobile App]
        │ PutRecord API
        ▼
[Kinesis Data Streams]
        │
    ┌───┴────┐
    │        │
    ▼        ▼
[Lambda] [EC2 Consumer]
    │        │
    ▼        ▼
[DynamoDB] [Fraud Detection Service]
```

**Use case:** Phát hiện gian lận, personalization, gaming leaderboard.

### Pattern 2: Log & Metrics Ingestion (Thu Nạp Log & Chỉ Số)

```
[EC2 Instances / ECS Tasks]
   │ CloudWatch Agent / Kinesis Agent
   ▼
[Kinesis Data Firehose]
   │ Transform (Lambda)
   ▼
[S3] → [Athena] → [QuickSight]
```

**Use case:** Phân tích log ứng dụng, infrastructure metrics.

### Pattern 3: Lambda Architecture (Kiến Trúc Lambda — Kết Hợp Batch và Streaming)

```
[Data Source]
     │
     ├──────────────────────────┐
     ▼                          ▼
[Kinesis Data Streams]    [S3 Raw Data]
     │                          │
     ▼                          ▼
[Lambda / KDA]           [Glue ETL Job]
(Speed Layer —            (Batch Layer —
 Tầng Tốc Độ)             Tầng Theo Lô)
     │                          │
     ▼                          ▼
[DynamoDB]               [Redshift / S3]
     │                          │
     └──────────┬───────────────┘
                ▼
          [Serving Layer]
         (Tầng Phục Vụ)
```

### Pattern 4: IoT Data Pipeline (Đường Ống Dữ Liệu IoT)

```
[IoT Sensors / Devices]
       │ IoT Core Rules
       ▼
[Kinesis Data Streams]
       │
   ┌───┴──────────┐
   ▼              ▼
[KDA — Flink]  [KDF → S3]
(Real-time     (Historical
 Alerting)      Analysis)
   │
   ▼
[SNS → Email/SMS Alert]
```

---

## 📁 Các File Trong Module

| File                          | Nội Dung                                            | Độ Quan Trọng |
| ----------------------------- | ---------------------------------------------------- | ------------- |
| `README.md`                   | Tổng quan module này (file hiện tại)                 | ⭐⭐⭐        |
| `1-kinesis-data-streams.md`   | KDS — Shard, Consumer, Retention, Scaling            | ⭐⭐⭐        |
| `2-kinesis-firehose.md`       | KDF — Delivery, Buffering, Transformation            | ⭐⭐⭐        |
| `3-kinesis-analytics.md`      | KDA — Apache Flink, SQL, Windowing                   | ⭐⭐          |
| `4-kinesis-vs-kafka.md`       | Kinesis vs MSK/Kafka — So sánh chi tiết             | ⭐⭐⭐        |

---

## 🔑 Khái Niệm Cốt Lõi Cần Nắm

### Shard (Mảnh)

**Shard** là đơn vị throughput (Thông Lượng) cơ bản của Kinesis Data Streams:

- **Write capacity:** 1 MB/s hoặc 1,000 records/giây
- **Read capacity:** 2 MB/s (shared across all consumers)
- Thứ tự bản ghi được đảm bảo **trong một shard**
- Dữ liệu được routing vào shard dựa trên **Partition Key** (Khóa Phân Vùng)

### Partition Key (Khóa Phân Vùng)

```python
# Producer gửi record với partition key
kinesis.put_record(
    StreamName='my-stream',
    Data=json.dumps(event),
    PartitionKey=user_id  # Tất cả event của cùng user → cùng shard
)
```

**Lưu ý:** Chọn partition key có **cardinality cao** (nhiều giá trị khác nhau) để phân phối đều tải giữa các shard. Tránh "hot shard" (Mảnh Nóng) — một shard nhận quá nhiều tải.

### Consumer Types (Loại Consumer — Người Tiêu Thụ)

| Loại Consumer                         | Mô Tả                               | Max Throughput      |
| ------------------------------------- | ----------------------------------- | ------------------- |
| **Shared Throughput Consumer**        | Nhiều consumer chia sẻ 2MB/s/shard  | 2 MB/s per shard    |
| **Enhanced Fan-out Consumer**         | Mỗi consumer nhận riêng 2MB/s/shard | 2 MB/s per consumer |

### Retention (Lưu Giữ Dữ Liệu)

- **Mặc định:** 24 giờ
- **Tối đa:** 365 ngày (Extended Retention — Lưu Giữ Mở Rộng)
- Cho phép **replay** (Phát Lại) — đọc lại dữ liệu từ quá khứ

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

### Q: Kinesis Data Streams vs Kinesis Firehose — khi nào dùng cái nào?

**A:** Dùng KDS khi cần custom consumer code, nhiều consumer đọc cùng stream, hoặc cần replay dữ liệu. Dùng KDF khi chỉ cần deliver data vào S3/Redshift/OpenSearch mà không cần viết consumer — fully managed, ít code hơn, giá rẻ hơn cho use case đơn giản.

### Q: Tính số Shard cần thiết như thế nào?

**A:**
```
Số shard = max(
    ceil(max_write_throughput_MB / 1),   # dựa trên write (1 MB/s/shard)
    ceil(max_read_throughput_MB / 2)      # dựa trên read (2 MB/s/shard)
)

Ví dụ: 10 MB/s write, 15 MB/s read
→ max(ceil(10/1), ceil(15/2)) = max(10, 8) = 10 shards
```

### Q: Kinesis vs SQS — khác nhau thế nào?

| Khía Cạnh            | Kinesis Data Streams           | SQS (Simple Queue Service)      |
| -------------------- | ------------------------------ | ------------------------------- |
| **Mô hình**          | Pub/sub với multiple consumers | Queue — 1 message, 1 consumer   |
| **Thứ tự**           | Đảm bảo trong shard            | FIFO Queue tùy chọn             |
| **Replay**           | Có (retention 1-365 ngày)      | Không (sau khi consumed → xóa) |
| **Use case**         | Analytics, real-time pipeline  | Task queue, microservices       |

### Q: Hot Shard là gì và cách xử lý?

**A:** Hot Shard (Mảnh Nóng) xảy ra khi một shard nhận quá nhiều dữ liệu, vượt quá limit 1MB/s, gây lỗi `ProvisionedThroughputExceededException`. Cách xử lý:
1. Chọn partition key có cardinality cao hơn
2. Thêm random suffix vào partition key
3. Tăng số shard (Resharding — Tái Phân Mảnh)

---

## 📊 Tóm Tắt Nhanh Cho Phỏng Vấn

```
Kinesis = Họ dịch vụ streaming AWS

KDS  = Real-time, custom consumer, nhiều consumer, replay được
KDF  = Fully managed, deliver vào S3/Redshift/OpenSearch, near real-time
KDA  = SQL/Flink trên streaming data, windowing, join
KVS  = Streaming video

Shard = Đơn vị throughput (1MB/s write, 2MB/s read)
Partition Key = Quyết định shard nào nhận record
Retention = 24h mặc định, tối đa 365 ngày

KDS vs KDF: Cần custom? → KDS. Chỉ cần deliver? → KDF
KDS vs MSK: AWS native? → KDS. Kafka ecosystem? → MSK
```

---

## 🔗 Điều Hướng

| Tiếp Theo                                              | Quay Lại                                      |
| ------------------------------------------------------ | --------------------------------------------- |
| [1-kinesis-data-streams.md](./1-kinesis-data-streams.md) | [../README.md](../README.md) — Tổng Quan Analytics |
| [2-kinesis-firehose.md](./2-kinesis-firehose.md)       | [../01-fundamentals/](../01-fundamentals/)     |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Module:** 02-kinesis
**Trạng Thái:** ✅ Hoàn thành
