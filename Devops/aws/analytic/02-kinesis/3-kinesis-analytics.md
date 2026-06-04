# 📊 Kinesis Data Analytics — KDA (Phân Tích Dữ Liệu Kinesis)

> KDA — Kinesis Data Analytics — cho phép chạy **Apache Flink** (Xử Lý Luồng Phân Tán) hoặc **SQL** trực tiếp trên streaming data mà không cần quản lý cluster — phân tích, tổng hợp, lọc, và phát hiện bất thường ngay khi dữ liệu đến, trước khi lưu vào storage.

## 📚 Mục Lục

1. [Tổng Quan KDA](#tổng-quan-kda)
2. [Hai Chế Độ Vận Hành](#hai-chế-độ-vận-hành)
3. [Apache Flink Cơ Bản](#apache-flink-cơ-bản)
4. [Windowing — Cửa Sổ Thời Gian](#windowing--cửa-sổ-thời-gian)
5. [KDA Studio — Tương Tác](#kda-studio--tương-tác)
6. [Patterns Phổ Biến](#patterns-phổ-biến)
7. [Tích Hợp Với AWS Services](#tích-hợp-với-aws-services)
8. [Chi Phí](#chi-phí)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tổng Quan KDA

### Vấn Đề KDA Giải Quyết

Khi dùng KDS hoặc KDF đơn thuần, bạn nhận được raw events (Sự Kiện Thô). Nhưng business thường cần:

```
Câu hỏi cần trả lời NGAY TRÊN STREAM (không chờ batch):
- "Tổng doanh thu 5 phút qua là bao nhiêu?"
- "User nào click quá 100 lần trong 10 giây? (Bot detection)"
- "Cảm biến nào có nhiệt độ tăng bất thường trong 30 giây qua?"
- "Session nào có tỉ lệ lỗi > 10% trong cửa sổ 1 phút?"
```

KDA cho phép trả lời những câu hỏi này **trực tiếp trên stream**, trước khi dữ liệu vào S3 hay Redshift.

### Vị Trí KDA Trong Pipeline

```
Sources (Nguồn)       KDA Processing          Destinations (Đích)
──────────────        ───────────────          ───────────────────
KDS Stream     ──►    Filter records    ──►    KDS Stream (mới)
KDF Stream     ──►    Aggregate (5min)  ──►    S3 Bucket
               ──►    Detect anomaly    ──►    Lambda function
               ──►    Join 2 streams    ──►    Firehose delivery
               ──►    ML inference      ──►    DynamoDB
```

---

## 🔧 Hai Chế Độ Vận Hành

### Chế Độ 1: Flink Application (Ứng Dụng Flink)

Viết code Java/Python/Scala dùng Apache Flink API:

```
Phù hợp khi:
✅ Logic phức tạp: join nhiều stream, stateful processing
✅ Cần fine-grained control
✅ Team quen Apache Flink
✅ Production workloads

Không phù hợp khi:
❌ Cần prototype nhanh
❌ Không quen Flink API
```

### Chế Độ 2: KDA Studio (Zeppelin Notebooks)

Notebook tương tác dùng SQL / Python / Scala — phù hợp để khám phá data nhanh:

```
Phù hợp khi:
✅ Data exploration, prototyping
✅ Ad-hoc streaming queries
✅ Business analysts không quen coding
✅ Debug và test Flink logic

Không phù hợp khi:
❌ Production workloads liên tục
❌ Cần high throughput
```

---

## ⚡ Apache Flink Cơ Bản

### Khái Niệm Cốt Lõi

| Khái Niệm           | Giải Thích                                                         |
| ------------------- | ------------------------------------------------------------------ |
| **DataStream API**  | API chính để xử lý stream — map, filter, reduce, window...         |
| **Source**          | Đầu vào: KDS, Kafka, S3...                                         |
| **Sink**            | Đầu ra: KDS, S3, DynamoDB, Firehose...                             |
| **Operator**        | Hàm xử lý: map, flatMap, filter, keyBy, window, aggregate...      |
| **State**           | Trạng thái lưu giữ giữa các events (ví dụ: running sum)           |
| **Checkpoint**      | Snapshot trạng thái để recovery khi lỗi                            |
| **Parallelism**     | Số task chạy song song (tương tự shard trong KDS)                  |

### Ví Dụ: Phát Hiện Gian Lận Theo Thời Gian Thực

```java
// Java Flink Application — phát hiện giao dịch bất thường
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;

public class FraudDetectionJob {
    public static void main(String[] args) throws Exception {
        
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Source: đọc từ KDS
        DataStream<Transaction> transactions = env
            .addSource(new FlinkKinesisConsumer<>(
                "transactions-stream",
                new TransactionDeserializer(),
                kinesisConsumerConfig
            ));
        
        // Xử lý: tổng hợp theo cửa sổ 5 phút, nhóm theo user
        DataStream<AlertEvent> alerts = transactions
            // Nhóm theo user_id
            .keyBy(t -> t.getUserId())
            // Cửa sổ trượt 5 phút (sliding window 5min, bước 1 min)
            .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
            // Aggregate: tính tổng và đếm
            .aggregate(new TransactionAggregator())
            // Filter: chỉ giữ user có tổng > $10,000 trong 5 phút
            .filter(agg -> agg.getTotalAmount() > 10_000.0);
        
        // Sink: gửi cảnh báo vào KDS khác
        alerts.addSink(new FlinkKinesisProducer<>(
            "fraud-alerts-stream",
            new AlertSerializer(),
            kinesisProducerConfig
        ));
        
        env.execute("Fraud Detection Job");
    }
}
```

### Ví Dụ Python (PyFlink)

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kinesis import (
    FlinkKinesisConsumer, FlinkKinesisProducer
)
from pyflink.common.typeinfo import Types
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.common.time import Time

def main():
    env = StreamExecutionEnvironment.get_execution_environment()
    env.set_parallelism(4)  # Chạy 4 task song song
    
    # Cấu hình Kinesis source
    kinesis_source = FlinkKinesisConsumer(
        stream_name="click-events",
        deserializer=ClickEventSchema(),
        config_props={
            "aws.region": "ap-southeast-1",
            "flink.stream.initpos": "LATEST"
        }
    )
    
    # DataStream pipeline
    clicks = env.add_source(kinesis_source)
    
    # Đếm clicks theo user trong cửa sổ 1 phút
    click_counts = (
        clicks
        .key_by(lambda e: e['user_id'])
        .window(TumblingEventTimeWindows.of(Time.minutes(1)))
        .reduce(lambda a, b: {
            'user_id': a['user_id'],
            'click_count': a['click_count'] + b['click_count'],
            'window_end': b['timestamp']
        })
        .filter(lambda e: e['click_count'] > 100)  # Bot detection
    )
    
    # Ghi kết quả
    click_counts.print()
    
    env.execute("Click Count Analytics")

if __name__ == '__main__':
    main()
```

---

## 🕐 Windowing — Cửa Sổ Thời Gian

Windowing (Cửa Sổ Thời Gian) là khái niệm trung tâm của stream processing — chia stream vô hạn thành các đoạn hữu hạn để aggregate.

### Các Loại Window

#### 1. Tumbling Window (Cửa Sổ Lăn — Không Chồng Lấp)

```
Window size: 5 phút, không chồng lấp

|──────── 5min ────────|──────── 5min ────────|──────── 5min ────────|
| events: t0 → t5min   | events: t5 → t10min  | events: t10 → t15min |
|   aggregate = X      |   aggregate = Y      |   aggregate = Z      |

Use case: Tính tổng doanh thu mỗi 5 phút
         Đếm API calls mỗi phút để rate limiting
```

```java
// Flink Tumbling Window
stream
    .keyBy(event -> event.getStoreId())
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .sum("revenue")
```

#### 2. Sliding Window (Cửa Sổ Trượt — Có Chồng Lấp)

```
Window size: 10 phút, slide mỗi 5 phút

|───── 10min ──────────|
           |───── 10min ──────────|
                      |───── 10min ──────────|

Use case: Tính average trong 10 phút qua, cập nhật mỗi 5 phút
         Phát hiện trend tăng/giảm liên tục
```

```java
// Flink Sliding Window
stream
    .keyBy(event -> event.getUserId())
    .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(5)))
    .aggregate(new AvgAggregator())
```

#### 3. Session Window (Cửa Sổ Phiên — Dựa Trên Hoạt Động)

```
Đóng cửa sổ khi không có event trong khoảng thời gian gap (khoảng trống):

User A: [event1] [event2] [event3]        [event4] [event5]
        |──── session 1 ─────────|  gap  |── session 2 ──|

Use case: Phân tích session người dùng, tính session duration
         Web analytics, game session tracking
```

```java
// Flink Session Window (gap = 5 phút không có event → đóng session)
stream
    .keyBy(event -> event.getUserId())
    .window(EventTimeSessionWindows.withGap(Time.minutes(5)))
    .aggregate(new SessionAggregator())
```

### Event Time vs Processing Time

| Loại Thời Gian               | Ý Nghĩa                              | Khi Nào Dùng                        |
| ----------------------------- | ------------------------------------ | ----------------------------------- |
| **Event Time** (Thời Gian Sự Kiện) | Thời điểm event thực sự xảy ra | Production — xử lý đúng thứ tự     |
| **Processing Time** (Thời Gian Xử Lý) | Thời điểm Flink xử lý event | Dev/test — đơn giản, nhanh          |
| **Ingestion Time** (Thời Gian Nạp Vào) | Thời điểm event vào Flink | Khi không có event timestamp         |

```
Tại sao Event Time quan trọng?

Tình huống: Mobile app gửi event khi offline
Event thực sự xảy ra: 10:00 AM
Event được gửi lên server: 10:15 AM (sau khi có mạng)

Nếu dùng Processing Time → event tính vào window 10:15
Nếu dùng Event Time → event tính vào window 10:00 (đúng)
```

### Watermark (Dấu Mốc Thời Gian)

**Watermark** (Dấu Mốc Nước — Ngưỡng Thời Gian) cho Flink biết "dữ liệu đến thời điểm T đã đủ":

```java
// Cho phép late events (Sự Kiện Đến Trễ) tối đa 30 giây
stream
    .assignTimestampsAndWatermarks(
        WatermarkStrategy
            .<ClickEvent>forBoundedOutOfOrderness(Duration.ofSeconds(30))
            .withTimestampAssigner((event, timestamp) -> event.getEventTime())
    )
```

```
Timeline:
                    Watermark: T - 30s
                         ↓
Events: ...t1  t3  t5   t7  [gap]  t8  ...
         └─ trong window ─┘     └─ late event ─┘
                                    ↑
                            Có thể bị bỏ qua nếu quá trễ
```

---

## 🖥️ KDA Studio — Tương Tác

### KDA Studio Là Gì

KDA Studio là **Apache Zeppelin Notebook** được tích hợp vào AWS — cho phép viết Flink SQL hoặc code và thấy kết quả ngay lập tức, phù hợp để:

- Khám phá stream data nhanh
- Prototype logic trước khi viết Flink application đầy đủ
- Debug và kiểm tra

### Flink SQL Trong KDA Studio

```sql
-- Định nghĩa bảng source từ KDS
CREATE TABLE user_clicks (
    user_id      VARCHAR(64),
    page_url     VARCHAR(255),
    click_time   BIGINT,
    country      VARCHAR(10),
    event_time AS TO_TIMESTAMP(FROM_UNIXTIME(click_time / 1000)),
    WATERMARK FOR event_time AS event_time - INTERVAL '30' SECOND
) WITH (
    'connector' = 'kinesis',
    'stream' = 'user-click-events',
    'aws.region' = 'ap-southeast-1',
    'scan.stream.initpos' = 'LATEST',
    'format' = 'json'
);

-- Định nghĩa bảng sink (nơi ghi kết quả)
CREATE TABLE click_counts_output (
    country      VARCHAR(10),
    window_start TIMESTAMP,
    window_end   TIMESTAMP,
    click_count  BIGINT
) WITH (
    'connector' = 'kinesis',
    'stream' = 'click-analytics-output',
    'aws.region' = 'ap-southeast-1',
    'format' = 'json'
);

-- Streaming query: đếm clicks theo country mỗi 5 phút
INSERT INTO click_counts_output
SELECT
    country,
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) AS window_start,
    TUMBLE_END(event_time, INTERVAL '5' MINUTE)   AS window_end,
    COUNT(*) AS click_count
FROM user_clicks
GROUP BY
    country,
    TUMBLE(event_time, INTERVAL '5' MINUTE);
```

```sql
-- Ví dụ thực tế: Phát hiện bot (click quá nhanh)
SELECT
    user_id,
    COUNT(*) AS click_count_1min,
    MIN(event_time) AS first_click,
    MAX(event_time) AS last_click
FROM user_clicks
GROUP BY
    user_id,
    TUMBLE(event_time, INTERVAL '1' MINUTE)
HAVING COUNT(*) > 100  -- Hơn 100 clicks/phút → nghi ngờ bot
```

```sql
-- Sliding window: doanh thu trung bình 10 phút (cập nhật mỗi 5 phút)
SELECT
    store_id,
    HOP_START(order_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE) AS window_start,
    HOP_END(order_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE)   AS window_end,
    SUM(amount) AS revenue_10min,
    AVG(amount) AS avg_order_value
FROM orders_stream
GROUP BY
    store_id,
    HOP(order_time, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE)
```

---

## 🏗️ Patterns Phổ Biến

### Pattern 1: Real-time Aggregation (Tổng Hợp Thời Gian Thực)

```
Use case: Dashboard real-time hiển thị KPIs

KDS (raw events)
    │
    ▼
KDA Flink (tumbling window 1min: sum, count, avg)
    │
    ▼
KDS (aggregated metrics)
    │
    ├──► Lambda → DynamoDB (lưu metrics hiện tại)
    └──► KDF → OpenSearch (hiển thị trên dashboard)
```

### Pattern 2: Anomaly Detection (Phát Hiện Bất Thường)

```
Use case: Phát hiện cảm biến hỏng, gian lận tài chính

KDS (sensor readings / transactions)
    │
    ▼
KDA Flink (sliding window: tính z-score, so sánh với baseline)
    │
    ▼
Filter (chỉ giữ records bất thường)
    │
    ├──► SNS → Email/SMS (cảnh báo ngay)
    └──► KDS → Lambda (trigger automated response)
```

### Pattern 3: Stream Enrichment (Làm Giàu Stream)

```
Use case: Thêm thông tin user vào mỗi click event

KDS (click events: user_id, page_url)
    │
    ▼
KDA Flink
    ├── Join với DynamoDB (lookup user profile)
    ├── Thêm: user_segment, country, subscription_tier
    └── Ghi ra KDS (enriched events)
                │
                ▼
            KDF → S3 (full enriched data cho analytics)
```

### Pattern 4: Multi-stream Join (Kết Hợp Nhiều Stream)

```
Use case: Tính conversion rate từ impression đến purchase

Stream 1: ad_impressions (user_id, ad_id, timestamp)
Stream 2: purchases (user_id, product_id, amount, timestamp)

KDA Flink (Temporal Join — Kết Hợp Theo Thời Gian):
- Join 2 streams theo user_id
- Trong window 30 phút: impression → purchase = conversion
- Tính conversion rate theo ad_id

→ Output: Báo cáo ROI quảng cáo theo thời gian thực
```

---

## 🔗 Tích Hợp Với AWS Services

### Nguồn Đầu Vào (Sources)

```python
# Từ KDS
source_config = {
    "InputStartingPositionConfiguration": {"InputStartingPosition": "NOW"},
    "KinesisStreamsInput": {
        "ResourceARN": "arn:aws:kinesis:...:stream/my-stream"
    }
}

# Từ KDF
source_config = {
    "KinesisFirehoseInput": {
        "ResourceARN": "arn:aws:firehose:...:deliverystream/my-firehose"
    }
}
```

### Các Đầu Ra (Destinations/Sinks)

| Destination        | Use Case                                  |
| ------------------ | ----------------------------------------- |
| **KDS**            | Real-time results cho downstream consumers |
| **KDF**            | Deliver kết quả vào S3/Redshift/OpenSearch |
| **Lambda**         | Trigger action dựa trên kết quả           |
| **S3**             | Lưu streaming analytics results           |

---

## 💰 Chi Phí

### Mô Hình Tính Phí

| Thành Phần              | Giá                                  |
| ----------------------- | ------------------------------------ |
| **KPU — Kinesis Processing Unit** (Đơn Vị Xử Lý Kinesis) | ~$0.11/KPU/giờ |
| 1 KPU = 1 vCPU + 4 GB RAM               | —                |
| **Storage** (cho state checkpointing)    | ~$0.10/GB/tháng  |

```
Ví dụ: Flink app với 4 KPUs chạy 24/7:
Chi phí = 4 KPU × $0.11 × 24 × 30 = $316.8/tháng

→ Đáng để so sánh với EMR Spark Streaming hoặc MSK + Flink
   Nếu workload nhỏ-vừa: KDA thuận tiện hơn, EMR rẻ hơn với scale lớn
```

### Khi Nào Dùng KDA vs EMR Spark Streaming

| Tiêu Chí                 | KDA (Flink)                     | EMR Spark Streaming             |
| ------------------------ | ------------------------------- | ------------------------------- |
| **Ops complexity**       | Thấp — managed service          | Cao — cần quản lý cluster       |
| **Cost** (nhỏ-vừa)      | Vừa phải                        | Đắt hơn (cluster overhead)     |
| **Cost** (large scale)   | Có thể đắt hơn                  | Rẻ hơn với Spot Instances       |
| **Flink ecosystem**      | Đầy đủ                          | Không áp dụng                   |
| **Spark ecosystem**      | Không áp dụng                   | Đầy đủ                          |
| **Latency**              | Sub-second (Flink native)       | Mini-batch (1-30s)              |

---

## 🎯 Câu Hỏi Phỏng Vấn

### Q1: KDA dùng Apache Flink gì? Tại sao Flink tốt cho streaming?

**A:** KDA dùng Apache Flink — engine xử lý stream distributed mạnh mẽ nhất hiện tại. Flink nổi bật vì:
- **True streaming** (không phải micro-batch như Spark Streaming cũ) → latency thấp hơn
- **Stateful processing** — lưu trạng thái giữa events (ví dụ: running sum, session state)
- **Exactly-once semantics** — đảm bảo mỗi event được xử lý đúng 1 lần dù có failure
- **Flexible windowing** — Tumbling, Sliding, Session windows với Event Time / Processing Time

### Q2: Watermark trong Flink là gì?

**A:** Watermark (Dấu Mốc Thời Gian) là cơ chế Flink dùng để xử lý **late-arriving events** (Sự Kiện Đến Muộn). Watermark tại thời điểm T có nghĩa "mọi event có timestamp ≤ T đã đến đủ — có thể đóng window". Ví dụ: `WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(30))` cho phép event đến muộn tối đa 30 giây. Events đến sau watermark đóng window sẽ bị bỏ qua hoặc xử lý vào side output.

### Q3: Sự khác biệt giữa Tumbling, Sliding và Session Window?

**A:**
- **Tumbling** (Lăn): Các cửa sổ liền kề, không chồng lấp. Mỗi event thuộc đúng 1 window. Dùng khi cần aggregate theo khoảng thời gian cố định (mỗi 5 phút).
- **Sliding** (Trượt): Cửa sổ chồng lấp, cập nhật thường xuyên hơn. Dùng khi cần "rolling average" (trung bình trượt) cập nhật liên tục.
- **Session** (Phiên): Đóng window khi không có event trong khoảng gap. Dùng khi cần phân tích session tự nhiên của user.

---

## 📊 Tóm Tắt Nhanh

```
KDA — Kinesis Data Analytics
├── Engine: Apache Flink (trực tiếp stream processing)
├── 2 modes: Flink Application (production) | Studio Notebook (khám phá)
├── Windows: Tumbling | Sliding | Session
├── Time: Event Time (đúng) | Processing Time (đơn giản)
├── Watermark: xử lý late events
└── Chi phí: $0.11/KPU/giờ

Use cases:
├── Real-time aggregation (dashboard metrics)
├── Anomaly detection (fraud, sensor)
├── Stream enrichment (join với reference data)
└── Multi-stream join (conversion tracking)
```

---

## 🔗 Điều Hướng

| Tiếp Theo                                      | Quay Lại                                              |
| ---------------------------------------------- | ----------------------------------------------------- |
| [4-kinesis-vs-kafka.md](./4-kinesis-vs-kafka.md) | [2-kinesis-firehose.md](./2-kinesis-firehose.md)     |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** 3-kinesis-analytics.md
**Trạng Thái:** ✅ Hoàn thành
