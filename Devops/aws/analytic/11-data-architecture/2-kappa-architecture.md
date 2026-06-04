# Kappa Architecture — Kiến Trúc Kappa: Stream-Only Processing

> Kappa Architecture (Kiến Trúc Kappa) do Jay Kreps (đồng sáng lập Apache Kafka) đề xuất năm 2014 như một giải pháp đơn giản hóa Lambda Architecture. Triết lý cốt lõi: "Nếu stream processing engine đủ mạnh, tại sao cần maintain hai hệ thống?" Kappa loại bỏ hoàn toàn Batch Layer, chỉ dùng một Streaming Layer để xử lý cả dữ liệu thời gian thực lẫn dữ liệu lịch sử thông qua replay (phát lại).

## 📚 Mục Lục

1. [Triết Lý Kappa Architecture](#triết-lý-kappa-architecture)
2. [Kiến Trúc Chi Tiết](#kiến-trúc-chi-tiết)
3. [Reprocessing với Stream Replay](#reprocessing-với-stream-replay)
4. [Triển Khai trên AWS](#triển-khai-trên-aws)
5. [Ưu Điểm và Nhược Điểm](#ưu-điểm-và-nhược-điểm)
6. [Lambda vs Kappa — Khi Nào Chọn Gì](#lambda-vs-kappa)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💡 Triết Lý Kappa Architecture

### Nguồn Gốc Vấn Đề

Jay Kreps quan sát thấy điểm yếu chính của Lambda Architecture: **code duplication** (sao chép code). Cùng một business logic phải implement hai lần — một cho Spark batch và một cho stream processing. Hai implementation này thường dần dần cho kết quả khác nhau do lỗi nhỏ tích lũy theo thời gian.

**Câu hỏi Kreps đặt ra:** *"Nếu chúng ta có thể replay stream từ đầu với tốc độ cao, thì batch processing chỉ là streaming với input cố định. Tại sao không dùng một hệ thống duy nhất?"*

### Nguyên Tắc Cốt Lõi

```
Mọi dữ liệu = Sự kiện trong một stream bất biến (immutable log)

Kappa Architecture = Một Streaming Layer duy nhất
                   + Khả năng Replay từ bất kỳ điểm nào trong quá khứ
                   + Serving Layer phục vụ kết quả
```

---

## 🏗️ Kiến Trúc Chi Tiết

```
                ┌──────────────────────────────────────────┐
                │             DATA SOURCES                  │
                │   (Events, CDC, Logs, Sensors)            │
                └────────────────┬─────────────────────────┘
                                 │
                                 ▼
                ┌──────────────────────────────────────────┐
                │         IMMUTABLE LOG STORAGE            │
                │    (Lưu trữ Log Bất Biến — Stream)       │
                │                                          │
                │  Apache Kafka / Amazon Kinesis / MSK     │
                │  • Lưu toàn bộ events theo thứ tự       │
                │  • Retention dài hạn (days → forever)   │
                │  • Không xóa, không sửa — chỉ append    │
                └────────────────┬─────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌──────────────────┐     ┌──────────────────────┐
          │  STREAM JOB v1   │     │   STREAM JOB v2      │
          │  (đang chạy)     │     │   (version mới khi   │
          │                  │     │    cần reprocess)     │
          │ Flink / KDA      │     │                      │
          │ Processing       │     │ Replay từ offset=0   │
          │ real-time events │     │ xử lý lại lịch sử    │
          └────────┬─────────┘     └──────────┬───────────┘
                   │                          │
                   │     Output v1            │   Output v2
                   └────────────┬─────────────┘
                                │
                                ▼
                ┌───────────────────────────────────┐
                │          SERVING LAYER            │
                │        (Lớp Phục Vụ)              │
                │                                   │
                │  Kết quả từ Job v2 thay thế v1   │
                │  sau khi reprocessing hoàn tất   │
                │                                   │
                │  DynamoDB / Redshift / S3         │
                └───────────────────────────────────┘
```

---

## 🔄 Reprocessing với Stream Replay

Đây là tính năng cốt lõi khiến Kappa Architecture khả thi. Khi cần tính toán lại lịch sử:

### Quy Trình Reprocessing (Xử Lý Lại)

```
Bước 1: Phát hiện lỗi hoặc logic thay đổi
         │
         ▼
Bước 2: Deploy stream job VERSION MỚI với consumer group (nhóm người tiêu thụ) riêng
         │  • Job v2 bắt đầu đọc từ offset=0 (đầu stream)
         │  • Ghi kết quả vào bảng/bucket riêng (output v2)
         │
         ▼
Bước 3: Job v2 chạy song song với Job v1 đang production
         │  • v1 tiếp tục phục vụ user
         │  • v2 đang "bắt kịp" với hiện tại (catch up)
         │
         ▼
Bước 4: v2 bắt kịp real-time → atomic swap (hoán đổi nguyên tử)
         │  • Serving layer chuyển từ đọc output v1 → output v2
         │
         ▼
Bước 5: Xóa Job v1 và output v1
         • Quá trình reprocessing hoàn tất, không downtime
```

### Ví Dụ Thực Tế với Kinesis

```python
# Kappa với Kinesis — Reprocessing flow

# Job v1 — đang chạy production, đọc từ LATEST
consumer_v1 = KinesisConsumer(
    stream_name="order-events",
    starting_position="LATEST",  # Chỉ đọc events mới
    output_table="revenue_by_product_v1"
)

# Phát hiện lỗi trong discount calculation logic
# Deploy Job v2 với fix mới

# Job v2 — replay từ đầu, dùng TRIM_HORIZON để đọc từ record cũ nhất
consumer_v2 = KinesisConsumer(
    stream_name="order-events",
    starting_position="TRIM_HORIZON",  # Đọc từ đầu stream
    output_table="revenue_by_product_v2"  # Ghi vào bảng riêng
)

# Sau khi v2 catch up với hiện tại:
# Serving layer switch: READ FROM revenue_by_product_v2
# Xóa revenue_by_product_v1
```

---

## ☁️ Triển Khai trên AWS

### Option 1: Kinesis + Kinesis Data Analytics (KDA) / Managed Flink

```
                    Kinesis Data Streams
                           │
               ┌───────────┴───────────┐
               │                       │
        KDA Flink App v1          KDA Flink App v2
        (production)              (reprocessing)
        StartingPosition:         StartingPosition:
        LATEST                    TRIM_HORIZON
               │                       │
               ▼                       ▼
        DynamoDB table v1       DynamoDB table v2
               │
               └── Swap khi v2 bắt kịp ──→ DynamoDB table v2 (serving)
```

**Lưu ý Kinesis Retention:**
```
Default: 24 giờ
Extended: tối đa 365 ngày (có phí thêm)
→ Với Kappa, bạn cần retention đủ dài để replay toàn bộ dữ liệu cần thiết
→ Dữ liệu cần lưu lâu hơn 365 ngày: consider MSK thay vì Kinesis
```

### Option 2: MSK (Managed Kafka) + AWS Glue Streaming / Flink

```
                    MSK (Kafka Cluster)
                    Retention: Không giới hạn (lưu S3 với Tiered Storage)
                           │
               ┌───────────┴───────────┐
               │                       │
    AWS Glue Streaming Job v1    AWS Glue Streaming Job v2
    Consumer Group: "prod"       Consumer Group: "reprocess"
    Auto.offset.reset: latest    Auto.offset.reset: earliest
               │                       │
               ▼                       ▼
        S3 output v1            S3 output v2
               │
               └── Athena view switch ──→ Point to output v2
```

**MSK Tiered Storage** (Lưu Trữ Phân Tầng MSK):
- Hot storage: Broker disk (truy cập nhanh)
- Cold storage: S3 (giảm chi phí cho dữ liệu cũ)
- Cho phép retention không giới hạn với chi phí tối ưu — ideal cho Kappa

### Option 3: S3 as Immutable Log + EMR Streaming

```
# S3 có thể đóng vai trò immutable log nếu dùng đúng
# Append-only pattern với S3 events

s3://events-log/
├── year=2024/month=01/day=01/hour=00/
│   ├── events-00001.parquet  # Không bao giờ sửa/xóa
│   ├── events-00002.parquet
│   └── ...
└── ...

# Streaming job đọc từ S3 events notification
# Reprocess: scan toàn bộ S3 prefix từ ngày cần reprocess
```

### Kinesis Data Analytics (KDA) — Flink Job Template

```python
# KDA Managed Flink — Kappa Architecture pattern
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import KinesisStreamsSource

env = StreamExecutionEnvironment.get_execution_environment()

# Source: Kinesis stream với starting position
kinesis_source = KinesisStreamsSource.builder() \
    .set_stream_arn("arn:aws:kinesis:us-east-1:123:stream/order-events") \
    .set_starting_position(InitialPosition.TRIM_HORIZON)  # Hoặc LATEST
    .build()

orders = env.from_source(
    kinesis_source,
    WatermarkStrategy.for_bounded_out_of_orderness(Duration.of_seconds(5)),
    "Kinesis Order Events"
)

# Transform: tính revenue với tumbling window (cửa sổ trượt)
revenue = orders \
    .key_by(lambda o: o['product_id']) \
    .window(TumblingProcessingTimeWindows.of(Time.minutes(5))) \
    .aggregate(RevenueAggregator())

# Sink: ghi vào DynamoDB
revenue.add_sink(DynamoDBSink("revenue_table_v2"))

env.execute("Revenue Calculation v2")
```

---

## ⚖️ Ưu Điểm và Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Một codebase duy nhất** | Không còn vấn đề code duplication của Lambda |
| **Đơn giản hơn** | Ít moving parts, dễ hiểu và debug |
| **Operational simplicity** (Đơn giản vận hành) | Chỉ một loại pipeline cần monitor và maintain |
| **Consistent logic** (Logic nhất quán) | Batch và real-time dùng cùng code → không thể diverge |
| **Thiết kế hiện đại** | Phù hợp với event-driven architecture (kiến trúc hướng sự kiện) |

### ❌ Nhược Điểm

| Nhược Điểm | Giải Thích |
|-----------|-----------|
| **Stream platform dependency** | Phụ thuộc hoàn toàn vào khả năng của streaming system |
| **Retention cost** | Lưu stream lâu dài tốn kém (Kinesis Extended Retention có phí) |
| **Reprocess time** | Replay toàn bộ lịch sử lớn có thể mất nhiều giờ/ngày |
| **Complex aggregations** | Một số phép tính batch (window join phức tạp) khó làm trong stream |
| **Backpressure management** | Quản lý áp lực ngược khi job v2 replay nhanh cần thêm kỹ năng |

---

## ⚔️ Lambda vs Kappa — Khi Nào Chọn Gì

### Ma Trận Quyết Định

| Điều Kiện | Chọn Lambda | Chọn Kappa |
|-----------|-------------|------------|
| **Stream retention** cần > 1 năm | ✓ (S3 rẻ hơn) | ✗ (Kinesis/Kafka đắt) |
| **Reprocessing** cần < 1 ngày | ✗ (batch chậm) | ✓ (replay nhanh) |
| **Team size** nhỏ (< 5 data engineers) | ✗ (2 pipeline, phức tạp) | ✓ (1 pipeline) |
| **Computation** cực kỳ phức tạp (ML training, large joins) | ✓ (Spark batch tốt hơn) | ✗ (stream processing hạn chế) |
| **Latency** yêu cầu < 1 giây | ✓ hoặc ✓ | ✓ |
| **Budget** hạn chế | ✗ (2 pipeline = 2x cost) | ✓ |
| **Kafka/Flink expertise** có sẵn | Không quan trọng | ✓ (cần thiết) |

### Câu Hỏi Để Quyết Định

```
1. Stream platform của bạn có support long retention đủ để replay không?
   → Không: Lambda Architecture (cần Batch Layer cho reprocessing)
   → Có:    Kappa Architecture

2. Logic xử lý có thể express (diễn đạt) như stream operations không?
   → Không: Lambda Architecture (batch cho logic phức tạp)
   → Có:    Kappa Architecture

3. Team có khả năng vận hành hai pipeline riêng biệt không?
   → Không: Kappa Architecture (đơn giản hơn)
   → Có:    Cả hai đều khả thi, chọn theo yêu cầu khác

4. Yêu cầu recompute (tính lại) lịch sử nhiều năm thường xuyên không?
   → Có:    Lambda Architecture (S3 rẻ hơn cho lưu trữ lâu dài)
   → Không: Kappa Architecture
```

---

## 🔧 Patterns Thực Tế trên AWS

### Pattern 1: Kappa với Kinesis + Flink + S3

```
Kinesis (7-365 ngày retention)
    ↓
Managed Flink (KDA)
    ↓
S3 (Parquet output, partitioned by time)
    ↓
Athena / Redshift Spectrum (Serving)
```

**Khi reprocess:**
```bash
# 1. Tạo Flink job mới với starting position TRIM_HORIZON
# 2. Ghi vào S3 prefix mới: s3://output/v2/
# 3. Khi v2 bắt kịp, đổi Athena table location
aws glue update-table \
  --database-name analytics \
  --table-input '{"StorageDescriptor": {"Location": "s3://output/v2/"}}'
# 4. Drop S3 prefix v1
```

### Pattern 2: Kappa với MSK + Glue Streaming + Delta Lake

```
MSK (Kafka, Tiered Storage — S3 backing)
    ↓
AWS Glue Streaming Job (PySpark Structured Streaming)
    ↓
Delta Lake on S3 (ACID transactions, time travel)
    ↓
Athena / Redshift Spectrum (Serving)
```

**Lợi thế của Delta Lake trong Kappa:**
```python
# Delta Lake cho phép "time travel" — query dữ liệu tại thời điểm bất kỳ
# Thay thế cho việc lưu nhiều version output
spark.read \
    .format("delta") \
    .option("versionAsOf", 0)  # Đọc version đầu tiên
    .load("s3://delta-lake/revenue/")

# Hoặc query theo thời gian
spark.read \
    .format("delta") \
    .option("timestampAsOf", "2024-01-15")
    .load("s3://delta-lake/revenue/")
```

---

## 💡 Anti-patterns Cần Tránh

### ❌ Anti-pattern 1: Kinesis với Retention Quá Ngắn

```
Vấn đề: Kinesis default retention = 24 giờ
        → Không thể replay để reprocess dữ liệu 1 tuần trước

Giải pháp:
1. Tăng Kinesis retention (tốn phí thêm)
2. Dùng S3 làm immutable backup song song với Kinesis
3. Chuyển sang MSK với Tiered Storage nếu cần retention dài
```

### ❌ Anti-pattern 2: Mutating (Thay Đổi) Stream Events

```
Vấn đề: Sửa event đã publish vào Kafka/Kinesis
        → Phá vỡ tính immutability (bất biến) — nền tảng của Kappa

Giải pháp:
- Events phải immutable
- Nếu cần sửa: publish event mới với type "CORRECTION" (sửa đổi)
- Stream job xử lý CORRECTION event và override kết quả cũ
```

### ❌ Anti-pattern 3: Quá Phụ Thuộc Vào Stateful Processing

```
Vấn đề: Stream job giữ state (trạng thái) rất lớn trong memory
        → Reprocessing từ đầu tốn rất nhiều thời gian để rebuild state

Giải pháp:
- Tối thiểu hóa state được giữ trong stream job
- Dùng external state store (DynamoDB, ElastiCache) cho large state
- Checkpoint (điểm kiểm tra) thường xuyên để recovery nhanh
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Câu 1: "Kappa Architecture khác Lambda Architecture như thế nào?"

**Gợi ý trả lời:**

Kappa Architecture là phiên bản đơn giản hóa của Lambda, do Jay Kreps đề xuất để giải quyết vấn đề code duplication trong Lambda.

**Sự khác biệt chính:**
- Lambda có **hai pipeline song song** (batch + stream), Kappa chỉ có **một streaming pipeline**
- Lambda dùng Batch Layer cho dữ liệu lịch sử; Kappa dùng **stream replay** để reprocess lịch sử
- Lambda phức tạp hơn về vận hành; Kappa đơn giản hơn nhưng đòi hỏi stream platform mạnh

**Khi chọn Kappa:** Stream platform (Kafka, Kinesis) có đủ retention, logic có thể express như stream operations, team nhỏ muốn giảm operational overhead.

**Khi giữ Lambda:** Cần reprocess nhiều năm lịch sử với chi phí thấp, computation phức tạp cần Spark batch, hoặc stream platform chưa đủ mạnh.

---

### Câu 2: "Làm thế nào để reprocess dữ liệu trong Kappa Architecture mà không có downtime?"

**Gợi ý trả lời:**

Quy trình Blue-Green Reprocessing (Tái Xử Lý Xanh-Xanh Lá):

1. **Deploy job mới (v2)** với consumer group riêng, đọc từ đầu stream (offset = 0 / TRIM_HORIZON)
2. **Hai job chạy song song:** v1 phục vụ production, v2 xử lý lại lịch sử
3. **Monitor tiến độ:** v2 dần dần catch up với v1
4. **Atomic swap khi v2 bắt kịp real-time:** chuyển serving layer đọc từ output v2
5. **Cleanup:** xóa job v1 và output v1

Không có downtime vì v1 tiếp tục chạy trong suốt quá trình. Điều kiện tiên quyết: stream phải có đủ retention để replay toàn bộ dữ liệu cần tái xử lý.

---

### Câu 3: "Thách thức lớn nhất khi implement Kappa trên AWS là gì?"

**Gợi ý trả lời:**

Thách thức chính trên AWS là **Kinesis Data Streams retention limit** — tối đa 365 ngày và có phí cho extended retention. Nếu cần reprocess dữ liệu 2-3 năm, Kinesis không đủ.

**Cách giải quyết:**
1. **S3 as backup log:** Song song với Kinesis, ghi raw events vào S3 (rẻ hơn nhiều). Khi cần reprocess, đọc từ S3 thay vì Kinesis.
2. **Amazon MSK với Tiered Storage:** Kafka retention không giới hạn về thời gian, chỉ giới hạn bởi chi phí S3.
3. **Event Sourcing pattern:** Thiết kế hệ thống từ đầu theo hướng events là primary source of truth, lưu vào S3 Glacier cho dữ liệu cũ.

---

## 📊 Tổng Kết

```
Kappa Architecture = Một Streaming Layer duy nhất
                   + Immutable log (Kafka/Kinesis) với retention dài
                   + Stream replay cho reprocessing

Ưu điểm: Đơn giản, một codebase, ít vận hành
Nhược điểm: Phụ thuộc vào stream platform, retention cost cao

Trên AWS:
  → Kinesis + Managed Flink (KDA) cho use case < 1 năm retention
  → MSK (Kafka) + Tiered Storage cho use case cần retention dài
  → S3 backup log như safety net cho reprocessing lịch sử xa
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [3-data-mesh.md](./3-data-mesh.md)
