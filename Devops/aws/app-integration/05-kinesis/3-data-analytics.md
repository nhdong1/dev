# Kinesis Data Analytics — Phân Tích Streaming Thời Gian Thực

> **Amazon Kinesis Data Analytics** (nay được chia thành **Managed Service for Apache Flink** và **Kinesis Data Analytics for SQL**) cho phép xử lý và phân tích data streaming theo thời gian thực bằng SQL tiêu chuẩn hoặc Apache Flink, mà không cần quản lý infrastructure (hạ tầng).

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [SQL Application — Ứng Dụng SQL](#sql-application--ứng-dụng-sql)
3. [Apache Flink Application — Ứng Dụng Flink](#apache-flink-application--ứng-dụng-flink)
4. [Window Operations — Phép Toán Cửa Sổ Thời Gian](#window-operations--phép-toán-cửa-sổ-thời-gian)
5. [Reference Data — Dữ Liệu Tham Chiếu](#reference-data--dữ-liệu-tham-chiếu)
6. [Anomaly Detection — Phát Hiện Bất Thường](#anomaly-detection--phát-hiện-bất-thường)
7. [SQL vs Flink — So Sánh](#sql-vs-flink--so-sánh)
8. [Kiến Trúc Thực Tế](#kiến-trúc-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Vấn Đề Cần Giải Quyết

```
Bài toán:
- Có 1 triệu clickstream events/phút từ website
- Cần biết real-time: page nào đang hot? user nào bất thường?
- Cần alert ngay khi error rate vượt 5%

Truyền thống:
→ Load vào data warehouse → chạy query → kết quả sau vài giờ (quá muộn)

Kinesis Data Analytics:
→ Query trực tiếp trên stream → kết quả trong vài giây
```

### Hai Engine Xử Lý

| Engine | Ngôn Ngữ | Độ Phức Tạp | Phù Hợp |
|---|---|---|---|
| **SQL Application** | SQL (ANSI) | Thấp | Aggregation đơn giản, report |
| **Apache Flink** | Java, Python, Scala | Cao | Complex event processing, stateful |

### Kiến Trúc Tổng Quan

```
Input (Đầu Vào)           Analytics                  Output (Đầu Ra)
───────────────           ─────────                  ───────────────
Kinesis Data    ────────▶ SQL / Flink ─────────────▶ Kinesis Data Streams
Streams                   Processing                 Kinesis Firehose
Kinesis                   (Window ops,               Lambda
Firehose                  Aggregation,               S3 (Flink only)
                          Anomaly                    DynamoDB (Flink)
                          Detection)
```

---

## SQL Application — Ứng Dụng SQL

### Cấu Trúc SQL Application

```sql
-- Kinesis Analytics SQL tương tự ANSI SQL nhưng có thêm stream-specific syntax

-- 1. Định nghĩa input stream
CREATE OR REPLACE STREAM "INPUT_SQL_STREAM" (
    "userId"        VARCHAR(50),
    "eventType"     VARCHAR(30),
    "pageUrl"       VARCHAR(200),
    "responseTime"  INTEGER,
    "statusCode"    INTEGER,
    "timestamp"     TIMESTAMP
);

-- 2. Kết nối input stream với Kinesis source
CREATE OR REPLACE PUMP "INPUT_PUMP" AS
    INSERT INTO "INPUT_SQL_STREAM"
    SELECT STREAM
        "userId",
        "eventType",
        "pageUrl",
        "responseTime",
        "statusCode",
        CURRENT_TIMESTAMP
    FROM "SOURCE_SQL_STREAM_001";

-- 3. Tạo output stream với kết quả phân tích
CREATE OR REPLACE STREAM "OUTPUT_SQL_STREAM" (
    "pageUrl"       VARCHAR(200),
    "requestCount"  INTEGER,
    "avgResponse"   DOUBLE,
    "errorRate"     DOUBLE,
    "windowStart"   TIMESTAMP
);
```

### Tumbling Window — Cửa Sổ Lăn (Không Chồng Lấp)

```sql
-- Đếm requests theo từng phút (cửa sổ không chồng lấp)
-- Ví dụ: [10:00–10:01], [10:01–10:02], [10:02–10:03]

CREATE OR REPLACE PUMP "REQUEST_COUNT_PUMP" AS
    INSERT INTO "OUTPUT_SQL_STREAM"
    SELECT STREAM
        "pageUrl",
        COUNT(*) AS "requestCount",
        AVG("responseTime") AS "avgResponse",
        SUM(CASE WHEN "statusCode" >= 500 THEN 1.0 ELSE 0.0 END) / COUNT(*) * 100 AS "errorRate",
        STEP("INPUT_SQL_STREAM".ROWTIME BY INTERVAL '1' MINUTE) AS "windowStart"
    FROM "INPUT_SQL_STREAM"
    GROUP BY "pageUrl",
             STEP("INPUT_SQL_STREAM".ROWTIME BY INTERVAL '1' MINUTE);
```

### Sliding Window — Cửa Sổ Trượt (Chồng Lấp)

```sql
-- Tính response time trung bình trong 5 phút trượt
-- Ví dụ: tại 10:03 → tính data từ [09:58–10:03]

CREATE OR REPLACE PUMP "SLIDING_AVG_PUMP" AS
    INSERT INTO "ALERT_STREAM"
    SELECT STREAM
        "userId",
        AVG("responseTime") AS "avg5minResponse",
        COUNT(*) AS "requestCount",
        ROWTIME AS "calculatedAt"
    FROM "INPUT_SQL_STREAM"
    WHERE ROWTIME BETWEEN
        (CURRENT_ROW - INTERVAL '5' MINUTE) AND CURRENT_ROW
    GROUP BY "userId", FLOOR("INPUT_SQL_STREAM".ROWTIME TO MINUTE);
```

### Stagger Window — Cửa Sổ Lệch Pha

```sql
-- Stagger window: giống tumbling nhưng bắt đầu khi event đầu tiên đến
-- Phù hợp khi data có thể đến trễ (late arrival)

CREATE OR REPLACE PUMP "STAGGER_PUMP" AS
    INSERT INTO "AGGREGATED_STREAM"
    SELECT STREAM
        "userId",
        COUNT(*) AS "sessionEvents",
        MAX("responseTime") AS "maxResponse"
    FROM "INPUT_SQL_STREAM"
    WINDOWED BY STAGGER (
        PARTITION BY "userId" RANGE INTERVAL '10' MINUTE
    );
```

---

## Apache Flink Application — Ứng Dụng Flink

### Tại Sao Dùng Flink?

```
SQL Application: tốt cho aggregation đơn giản theo window
Apache Flink: mạnh hơn khi cần:
- Stateful processing (lưu state giữa các events)
- Complex event patterns (chuỗi events phức tạp)
- Custom windowing logic
- Joins giữa nhiều streams
- Machine learning inference
- Exactly-once processing semantics
```

### Flink Application Với Java

```java
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import software.amazon.kinesis.connectors.flink.FlinkKinesisConsumer;

public class OrderAnalyticsJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Cấu hình checkpointing — bật để đảm bảo exactly-once
        env.enableCheckpointing(60_000L); // checkpoint mỗi 60 giây

        // Đọc từ Kinesis Data Streams
        Properties consumerConfig = new Properties();
        consumerConfig.setProperty(AWSConfigConstants.AWS_REGION, "ap-southeast-1");
        consumerConfig.setProperty(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST");

        DataStream<String> rawStream = env.addSource(
            new FlinkKinesisConsumer<>("order-events", new SimpleStringSchema(), consumerConfig)
        );

        // Parse và xử lý
        DataStream<OrderEvent> orderEvents = rawStream
            .map(json -> objectMapper.readValue(json, OrderEvent.class))
            .filter(event -> event.getEventType().equals("ORDER_PLACED"));

        // Tính tổng revenue theo cửa sổ 1 phút
        DataStream<RevenueMetric> revenuePerMinute = orderEvents
            .keyBy(OrderEvent::getProductCategory)
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))
            .aggregate(new RevenueAggregator());

        // Ghi kết quả vào Kinesis Output Stream
        revenuePerMinute.addSink(
            new FlinkKinesisProducer<>("revenue-metrics", new RevenueMetricSerializer(), producerConfig)
        );

        env.execute("Order Revenue Analytics");
    }
}
```

### Flink — Stateful Processing (Xử Lý Có Trạng Thái)

```java
// Phát hiện user gặp nhiều lỗi liên tiếp — cần lưu state
public class ErrorRateDetector extends KeyedProcessFunction<String, PageEvent, Alert> {

    // State: đếm số lỗi trong sliding window
    private transient ValueState<ErrorState> errorState;

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<ErrorState> descriptor =
            new ValueStateDescriptor<>("errorState", ErrorState.class);
        errorState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(PageEvent event, Context ctx, Collector<Alert> out)
            throws Exception {

        ErrorState state = errorState.value();
        if (state == null) state = new ErrorState();

        if (event.getStatusCode() >= 500) {
            state.incrementErrors();
            state.setLastErrorTime(ctx.timestamp());

            if (state.getErrorCount() >= 10) {
                // Phát cảnh báo: user gặp >= 10 lỗi liên tiếp
                out.collect(new Alert(
                    event.getUserId(),
                    "HIGH_ERROR_RATE",
                    state.getErrorCount()
                ));
                state.reset();
            }
        } else {
            state.reset(); // Reset khi có request thành công
        }

        errorState.update(state);
    }
}
```

### Flink — Complex Event Processing (Xử Lý Sự Kiện Phức Tạp)

```java
// Pattern: phát hiện ORDER_PLACED → PAYMENT_FAILED trong 5 phút
// (khách hàng đặt hàng nhưng thanh toán thất bại)

Pattern<OrderEvent, ?> paymentFailurePattern = Pattern
    .<OrderEvent>begin("placed")
        .where(event -> event.getType().equals("ORDER_PLACED"))
    .followedBy("failed")
        .where(event -> event.getType().equals("PAYMENT_FAILED"))
    .within(Time.minutes(5)); // Hai events phải trong vòng 5 phút

PatternStream<OrderEvent> patternStream =
    CEP.pattern(orderStream.keyBy(OrderEvent::getOrderId), paymentFailurePattern);

// Xử lý khi pattern được tìm thấy
DataStream<Alert> alerts = patternStream.process(
    new PatternProcessFunction<OrderEvent, Alert>() {
        @Override
        public void processMatch(Map<String, List<OrderEvent>> match,
                                  Context ctx, Collector<Alert> out) {
            OrderEvent placed = match.get("placed").get(0);
            out.collect(new Alert("PAYMENT_FAILED_AFTER_ORDER", placed.getOrderId()));
        }
    }
);
```

---

## Window Operations — Phép Toán Cửa Sổ Thời Gian

### Ba Loại Window Chính

```
1. TUMBLING WINDOW (Cửa Sổ Lăn)
   ─────────────────────────────
   [──5min──][──5min──][──5min──]
   Không chồng lấp, mỗi event thuộc đúng một window
   Dùng: aggregation theo khoảng thời gian cố định

2. SLIDING WINDOW (Cửa Sổ Trượt)
   ────────────────────────────────
   [───10min───]
        [───10min───]
             [───10min───]
   Chồng lấp, cập nhật liên tục
   Dùng: moving average, real-time monitoring

3. SESSION WINDOW (Cửa Sổ Phiên)
   ───────────────────────────────
   [─activity─] gap [─activity─]
   Đóng cửa sổ khi không có event trong khoảng gap
   Dùng: user session analytics
```

### Xử Lý Late Arrival (Dữ Liệu Đến Trễ)

```java
// Dữ liệu từ mobile app có thể đến trễ vài phút
// Cần cấu hình allowed lateness (cho phép trễ)

DataStream<OrderEvent> withTimestamps = orderStream
    .assignTimestampsAndWatermarks(
        WatermarkStrategy
            .<OrderEvent>forBoundedOutOfOrderness(Duration.ofMinutes(2))
            // Watermark = max_event_time - 2 phút
            // Kinesis sẽ chờ tối đa 2 phút cho data trễ
            .withTimestampAssigner((event, ts) -> event.getTimestamp())
    );

DataStream<Metric> metrics = withTimestamps
    .keyBy(OrderEvent::getCategory)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .allowedLateness(Time.minutes(1)) // Cho phép trễ thêm 1 phút sau khi cửa sổ đóng
    .sideOutputLateData(lateOutputTag)  // Ghi data đến quá trễ ra side output
    .aggregate(new CountAggregator());
```

---

## Reference Data — Dữ Liệu Tham Chiếu

### Enrich Streaming Data (Làm Giàu Dữ Liệu Streaming)

```sql
-- Kết hợp stream data với lookup table (bảng tra cứu) từ S3
-- Ví dụ: thêm thông tin product category vào clickstream

-- Reference table từ S3 (cập nhật định kỳ)
CREATE OR REPLACE REFERENCE TABLE "ProductCategory" (
    "productId"  VARCHAR(50),
    "category"   VARCHAR(100),
    "brand"      VARCHAR(100)
);

-- Join stream với reference table
CREATE OR REPLACE PUMP "ENRICHED_PUMP" AS
    INSERT INTO "ENRICHED_STREAM"
    SELECT STREAM
        s."userId",
        s."productId",
        p."category",
        p."brand",
        s."eventType",
        s."amount"
    FROM "INPUT_SQL_STREAM" s
    LEFT JOIN "ProductCategory" p
        ON s."productId" = p."productId";
```

---

## Anomaly Detection — Phát Hiện Bất Thường

### RANDOM_CUT_FOREST — Thuật Toán Phát Hiện Bất Thường

```sql
-- Kinesis Analytics SQL tích hợp sẵn thuật toán RANDOM_CUT_FOREST
-- Phát hiện bất thường không cần viết ML code

CREATE OR REPLACE STREAM "ANOMALY_STREAM" (
    "userId"        VARCHAR(50),
    "transactionAmt" DOUBLE,
    "anomalyScore"  DOUBLE,
    "explanation"   VARCHAR(2000)
);

-- Áp dụng RANDOM_CUT_FOREST để phát hiện giao dịch bất thường
CREATE OR REPLACE PUMP "ANOMALY_PUMP" AS
    INSERT INTO "ANOMALY_STREAM"
    SELECT STREAM
        "userId",
        "transactionAmt",
        "anomalyScore",
        "explanation"
    FROM TABLE (
        RANDOM_CUT_FOREST(
            CURSOR(SELECT STREAM "userId", "transactionAmt" FROM "INPUT_SQL_STREAM"),
            100,    -- số cây trong forest (số lượng model components)
            256,    -- kích thước sample mỗi cây
            100000, -- số record training ban đầu trước khi dự đoán
            1       -- số shard partition
        )
    )
    WHERE "anomalyScore" > 3.5; -- Ngưỡng phát hiện bất thường (tuỳ chỉnh)
```

### HOTSPOT Detection — Phát Hiện Điểm Nóng

```sql
-- Phát hiện clustering bất thường trong dữ liệu không gian
-- Ví dụ: phát hiện nhiều giao dịch từ cùng IP/vùng địa lý

SELECT STREAM *
FROM TABLE (
    HOTSPOTS(
        CURSOR(SELECT STREAM "ipLong", "geoLat", "geoLon" FROM "TRANSACTION_STREAM"),
        1000,  -- kích thước window
        20,    -- max number of hotspots
        3600   -- minimum observations per hotspot
    )
);
```

---

## SQL vs Flink — So Sánh

| Tiêu Chí | SQL Application | Apache Flink |
|---|---|---|
| **Ngôn ngữ** | SQL (ANSI + extensions) | Java, Python, Scala |
| **Độ phức tạp** | Thấp | Cao |
| **Stateful processing** | Giới hạn | Đầy đủ |
| **Complex event patterns** | Không | Có (CEP library) |
| **Custom ML** | Chỉ RANDOM_CUT_FOREST | Tích hợp tùy ý |
| **Late data handling** | Cơ bản | Đầy đủ (watermarks) |
| **Exactly-once** | Không đảm bảo | Có với checkpointing |
| **Khả năng debug** | Khó | Tốt hơn (local testing) |
| **Thời gian dev** | Nhanh | Chậm hơn |
| **Dùng khi nào** | Aggregation đơn giản, alert | Complex business logic |

---

## Kiến Trúc Thực Tế

### Hệ Thống Phát Hiện Gian Lận Thời Gian Thực

```
Kiến trúc:

Payment API
    │
    │ PutRecord
    ▼
Kinesis Data Streams (payment-events)
    │
    ├──▶ Managed Apache Flink
    │         │
    │         │ Phân tích:
    │         │ - Pattern: nhiều lần thử thất bại
    │         │ - Velocity: giao dịch nhanh bất thường
    │         │ - Geo: vị trí bất thường
    │         │
    │         ▼
    │    Kinesis Data Streams (fraud-alerts)
    │         │
    │         ▼
    │    Lambda → SNS → Email/SMS Alert
    │
    └──▶ Kinesis Firehose
              │
              ▼
         S3 → Athena (audit log, lịch sử phân tích)
```

### Dashboard Realtime Với Kinesis Analytics

```
Kiến Trúc:

Web/Mobile        KDS           KDA SQL           KDS Output      Dashboard
──────────        ───           ───────           ──────────      ─────────
User events ────▶ Stream ─────▶ Window query ──▶ Results ──────▶ Lambda
                                                   stream          │
                                Per-minute:                        ▼
                                - Page views                   DynamoDB
                                - Error rate                       │
                                - Active users                     ▼
                                - Avg response time           API Gateway
                                                                   │
                                                                   ▼
                                                              WebSocket
                                                           (push to browser)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Khi nào chọn SQL Application vs Apache Flink?

**Trả lời:**
- **SQL:** logic đơn giản (count, avg, sum theo window), team không có Flink experience, cần prototype nhanh
- **Flink:** cần stateful processing, complex event patterns, exactly-once guarantee, custom transformations phức tạp
- Trong production, Flink thường được ưa chuộng hơn vì linh hoạt và có exactly-once

### Q2: Watermark trong Flink là gì và tại sao quan trọng?

**Trả lời:**
- **Watermark (Dấu Nước):** marker trong stream nói rằng "tất cả events trước timestamp X đã đến đầy đủ"
- Quan trọng vì: data có thể đến trễ (network delay, mobile offline)
- Không có watermark → window không biết khi nào đóng → kết quả sai
- Trade-off: watermark lớn → chờ lâu hơn nhưng ít mất data trễ; watermark nhỏ → kết quả nhanh nhưng có thể bỏ sót data trễ

### Q3: Tumbling Window vs Sliding Window — dùng khi nào?

**Trả lời:**
- **Tumbling:** báo cáo theo giờ/phút cố định, aggregation định kỳ (ví dụ: doanh thu mỗi 5 phút)
- **Sliding:** real-time monitoring liên tục (ví dụ: error rate trong 5 phút gần nhất, cập nhật mỗi 30 giây)
- Sliding tốn resource hơn vì mỗi event có thể thuộc nhiều window

### Q4: Kinesis Data Analytics xử lý late-arriving data như thế nào?

**Trả lời:**
- SQL: hỗ trợ giới hạn qua row-time ordering
- Flink: dùng watermarks + allowed lateness
  - Watermark: cho phép chờ N phút để data trễ đến
  - Allowed lateness: sau khi window đóng, vẫn cập nhật kết quả nếu data trễ đến trong khoảng cho phép
  - Data trễ quá giới hạn → side output stream → xử lý riêng hoặc bỏ qua

---

**Tiếp Theo:** [4-shard-management.md](./4-shard-management.md) — Quản lý Shard, tính toán số lượng Shard cần thiết và chiến lược scaling.
