# Lambda Architecture — Kiến Trúc Lambda: Batch + Speed + Serving

> Lambda Architecture (Kiến Trúc Lambda) là mẫu kiến trúc dữ liệu do Nathan Marz đề xuất nhằm giải quyết bài toán cần vừa xử lý dữ liệu lịch sử chính xác vừa phân tích dữ liệu mới nhất với độ trễ thấp. Tên "Lambda" không liên quan đến AWS Lambda Functions mà lấy từ ký hiệu toán học λ biểu trưng cho sự kết hợp hai luồng tính toán.

## 📚 Mục Lục

1. [Vấn Đề Lambda Giải Quyết](#vấn-đề-lambda-giải-quyết)
2. [Ba Lớp Cốt Lõi](#ba-lớp-cốt-lõi)
3. [Triển Khai trên AWS](#triển-khai-trên-aws)
4. [Ưu Điểm và Nhược Điểm](#ưu-điểm-và-nhược-điểm)
5. [Khi Nào Dùng Lambda Architecture](#khi-nào-dùng-lambda-architecture)
6. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Vấn Đề Lambda Giải Quyết

### Bài Toán Kinh Điển

Một công ty thương mại điện tử cần dashboard phân tích doanh thu. Họ có hai yêu cầu mâu thuẫn:

**Yêu cầu 1 — Độ chính xác:** Báo cáo hàng tháng phải chính xác tuyệt đối. Mọi giao dịch từ 3 năm trước phải được tính đúng sau khi phát hiện lỗi logic trong code ETL (Extract Transform Load — Trích Xuất Chuyển Đổi Tải).

**Yêu cầu 2 — Tốc độ:** Dashboard real-time phải hiển thị số liệu với độ trễ < 30 giây để team sales theo dõi trong ngày.

**Vấn đề cốt lõi:** Hai yêu cầu này mâu thuẫn nhau:
- Batch processing (Xử lý theo lô) cho độ chính xác cao nhưng cần giờ/ngày để hoàn thành
- Stream processing (Xử lý luồng) cho tốc độ cao nhưng khó reprocess (xử lý lại) dữ liệu lịch sử

Lambda Architecture giải quyết bằng cách chạy **hai pipeline song song** và hợp nhất kết quả.

---

## 🏗️ Ba Lớp Cốt Lõi

```
                    ┌─────────────────────────────────────────────┐
                    │              DATA SOURCES                    │
                    │    (Events, Logs, Transactions, Sensors)     │
                    └──────────────────┬──────────────────────────┘
                                       │
                         ┌─────────────┴─────────────┐
                         │                           │
                         ▼                           ▼
          ┌──────────────────────┐     ┌─────────────────────────┐
          │     BATCH LAYER      │     │      SPEED LAYER        │
          │   (Lớp Theo Lô)      │     │    (Lớp Tốc Độ)         │
          │                      │     │                         │
          │ • Lưu toàn bộ dữ liệu│     │ • Chỉ xử lý dữ liệu mới│
          │   thô (immutable)    │     │ • Kết quả gần đúng      │
          │ • Tính toán batch    │     │ • Độ trễ vài giây       │
          │   views định kỳ      │     │ • Tự động xóa khi batch │
          │ • Độ trễ giờ/ngày    │     │   view bắt kịp          │
          └──────────┬───────────┘     └──────────┬──────────────┘
                     │                            │
                     │   Batch Views              │   Real-time Views
                     │   (chính xác)              │   (gần đúng)
                     └──────────────┬─────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │        SERVING LAYER          │
                    │       (Lớp Phục Vụ)           │
                    │                               │
                    │ • Merge batch + real-time views│
                    │ • Phục vụ query với low latency│
                    │ • Client chỉ query 1 nơi      │
                    └───────────────────────────────┘
```

### Batch Layer (Lớp Theo Lô)

**Mục đích:** Đây là "source of truth" — nguồn sự thật. Lưu trữ toàn bộ dữ liệu thô bất biến (immutable) và tính toán batch views (khung nhìn theo lô) định kỳ.

**Đặc điểm:**
- Dữ liệu thô **không bao giờ bị xóa hay sửa** — chỉ append (thêm vào)
- Chạy batch job định kỳ (hàng giờ, hàng ngày) để tính toán các aggregate (tổng hợp)
- Kết quả cuối cùng có độ chính xác 100% vì xử lý toàn bộ dataset
- Có thể **recompute** (tính toán lại) toàn bộ nếu phát hiện lỗi logic

**AWS Implementation:**
```
S3 (lưu raw data) ──→ AWS Glue / EMR Spark ──→ S3 (batch views) ──→ Redshift / Athena
```

### Speed Layer (Lớp Tốc Độ)

**Mục đích:** Bù đắp độ trễ của Batch Layer bằng cách xử lý dữ liệu mới nhất trong thời gian thực.

**Đặc điểm:**
- Chỉ xử lý dữ liệu **từ lần batch job gần nhất đến hiện tại** — không xử lý lại lịch sử
- Kết quả có thể **gần đúng** (approximate) vì không có đủ ngữ cảnh đầy đủ
- Khi Batch Layer hoàn thành và "bắt kịp", Speed Layer tự **xóa dữ liệu cũ**
- Yêu cầu hệ thống stream processing có khả năng xử lý nhanh

**AWS Implementation:**
```
Kinesis Data Streams ──→ Kinesis Data Analytics (Flink) ──→ DynamoDB / ElastiCache / Redshift
```

### Serving Layer (Lớp Phục Vụ)

**Mục đích:** Hợp nhất kết quả từ Batch Layer và Speed Layer để phục vụ query từ client.

**Đặc điểm:**
- Client **không biết** dữ liệu đến từ batch hay stream — chỉ query một nơi
- Serving layer thực hiện **merge logic**: kết hợp batch views (chính xác) + real-time views (mới nhất)
- Tối ưu cho low-latency reads (đọc độ trễ thấp)

**AWS Implementation:**
```
Redshift (batch views) + DynamoDB (real-time views) ──→ Application Layer
```

---

## ☁️ Triển Khai trên AWS

### Kiến Trúc Tham Khảo AWS

```
                    ┌──────────────────────────────────┐
                    │        DATA SOURCES              │
                    │  (IoT, Clickstream, Transactions) │
                    └────────┬──────────────┬──────────┘
                             │              │
                    ┌────────▼──────┐  ┌────▼──────────────┐
                    │ BATCH LAYER   │  │   SPEED LAYER      │
                    │               │  │                    │
                    │ S3 (raw data) │  │ Kinesis Data       │
                    │      │        │  │ Streams (KDS)      │
                    │      ▼        │  │      │             │
                    │ AWS Glue ETL  │  │      ▼             │
                    │ hoặc EMR Spark│  │ Kinesis Data       │
                    │      │        │  │ Analytics (Flink)  │
                    │      ▼        │  │      │             │
                    │ S3 (Parquet   │  │      ▼             │
                    │  batch views) │  │ DynamoDB /         │
                    └────────┬──────┘  │ ElastiCache        │
                             │         └────────┬───────────┘
                             │                  │
                    ┌────────▼──────────────────▼───────┐
                    │           SERVING LAYER            │
                    │                                    │
                    │  Redshift Spectrum (batch views)   │
                    │  +                                 │
                    │  DynamoDB (real-time views)        │
                    │  +                                 │
                    │  API Gateway / Application         │
                    └────────────────────────────────────┘
```

### Ví Dụ Thực Tế: Phân Tích Đơn Hàng E-Commerce

**Batch Layer:**
```python
# AWS Glue Job — chạy mỗi giờ
# Tính revenue (doanh thu) theo sản phẩm cho toàn bộ lịch sử
spark.sql("""
    SELECT
        product_id,
        DATE_TRUNC('hour', order_time) AS hour,
        SUM(revenue)                   AS total_revenue,
        COUNT(*)                       AS order_count
    FROM s3_raw_orders
    GROUP BY product_id, hour
""").write.parquet("s3://data-lake/batch-views/revenue-by-product/")
```

**Speed Layer:**
```python
# Kinesis Data Analytics — xử lý trong thời gian thực
# Tính revenue trong 5 phút gần nhất (tumbling window)
SELECT
    product_id,
    TUMBLE_START(rowtime, INTERVAL '5' MINUTE) AS window_start,
    SUM(revenue)                               AS revenue_last_5min
FROM kinesis_order_stream
GROUP BY product_id, TUMBLE(rowtime, INTERVAL '5' MINUTE)
```

**Serving Layer — Merge Logic:**
```python
def get_product_revenue(product_id, start_time, end_time):
    # Lấy dữ liệu chính xác từ batch views (cho khoảng thời gian đã được batch xử lý)
    batch_data = redshift.query(f"""
        SELECT SUM(total_revenue)
        FROM batch_views.revenue_by_product
        WHERE product_id = '{product_id}'
          AND hour BETWEEN '{start_time}' AND '{last_batch_time}'
    """)

    # Lấy dữ liệu real-time cho khoảng thời gian batch chưa kịp xử lý
    realtime_data = dynamodb.get_item(
        Key={'product_id': product_id, 'window': 'last_5min'}
    )

    # Hợp nhất: batch (chính xác) + realtime (mới nhất)
    return batch_data['revenue'] + realtime_data['revenue_last_5min']
```

---

## ⚖️ Ưu Điểm và Nhược Điểm

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Tính chính xác cao** | Batch Layer xử lý toàn bộ dataset, không bỏ sót event nào |
| **Khả năng reprocessing** | Khi phát hiện lỗi logic, chỉ cần chạy lại batch job trên dữ liệu thô bất biến |
| **Fault tolerance** (Khả năng chịu lỗi) | Speed Layer lỗi không ảnh hưởng đến tính chính xác lịch sử |
| **Low latency** cho dữ liệu mới | Speed Layer cung cấp kết quả trong vài giây |
| **Mature ecosystem** | Nhiều công cụ và pattern đã được kiểm chứng qua nhiều năm |

### ❌ Nhược Điểm

| Nhược Điểm | Giải Thích |
|-----------|-----------|
| **Hai codebase song song** | Logic tính toán phải implement hai lần (batch + stream) → dễ diverge (phân kỳ) |
| **Complexity cao** | Hai pipeline riêng biệt, merge logic phức tạp tại Serving Layer |
| **Chi phí vận hành** | Phải maintain hai hệ thống khác nhau, nhân đôi nguồn lực DevOps |
| **Inconsistency** (Không nhất quán) | Trong thời gian batch chưa hoàn thành, Speed Layer có thể cho kết quả khác Batch Layer |
| **Khó debug** | Khi kết quả sai, phải kiểm tra cả hai pipeline để tìm nguồn gốc lỗi |

---

## 🎯 Khi Nào Dùng Lambda Architecture

### ✅ Phù Hợp Khi

```
✓ Cần KẾT HỢP batch + streaming — không thể chọn một trong hai
✓ Dữ liệu lịch sử cần độ chính xác tuyệt đối (tài chính, y tế, pháp lý)
✓ Có team đủ lớn để vận hành hai pipeline
✓ Yêu cầu reprocessing (tính toán lại) dữ liệu lịch sử khi logic thay đổi
✓ Real-time view có thể chấp nhận gần đúng (approximate)
```

### ❌ Không Phù Hợp Khi

```
✗ Team nhỏ, ít nguồn lực DevOps
✗ Chỉ cần streaming hoặc chỉ cần batch (không cần cả hai)
✗ Budget hạn chế (chi phí vận hành cao)
✗ Stream processing engine đủ mạnh để xử lý cả historical replay
  → Dùng Kappa Architecture thay thế
✗ Tổ chức lớn với nhiều domain riêng biệt
  → Xem xét Data Mesh
```

### 🔄 So Sánh với Kappa Architecture

| Tiêu Chí | Lambda | Kappa |
|----------|--------|-------|
| **Số pipeline** | 2 (batch + stream) | 1 (chỉ stream) |
| **Độ phức tạp code** | Cao (2 codebase) | Thấp (1 codebase) |
| **Reprocessing** | Chạy lại batch job | Replay stream từ đầu |
| **Historical accuracy** (Độ chính xác lịch sử) | Rất cao | Cao (nếu stream retention đủ dài) |
| **Operational overhead** (Chi phí vận hành) | Cao | Thấp hơn |
| **Streaming platform yêu cầu** | Bất kỳ | Mạnh (Apache Flink/Kafka) |

---

## 💡 Best Practices trên AWS

### 1. Tách Biệt Storage cho Batch và Speed Layer

```bash
# Cấu trúc S3 cho Lambda Architecture
s3://my-data-lake/
├── raw/                    # Batch Layer — immutable raw data
│   ├── year=2024/month=01/
│   └── year=2024/month=02/
├── batch-views/            # Batch Layer output
│   ├── revenue-by-product/
│   └── user-metrics/
└── speed-views/            # Speed Layer output (TTL-managed)
    └── last-5min-revenue/
```

### 2. Sử Dụng AWS Glue Workflow để Orchestrate (Điều Phối) Batch Layer

```python
# Glue Workflow orchestrate batch pipeline
# Trigger: mỗi giờ hoặc khi S3 có file mới
glue_trigger = {
    "Type": "SCHEDULED",
    "Schedule": "cron(0 * * * ? *)",  # mỗi giờ
    "Actions": [
        {"JobName": "raw-to-batch-view-job"},
        {"JobName": "validate-batch-view-job"}
    ]
}
```

### 3. DynamoDB TTL (Time-to-Live — Thời Gian Tồn Tại) cho Speed Layer

```python
# Auto-expire speed layer records sau khi batch view cập nhật
dynamodb_item = {
    'product_id': product_id,
    'revenue_last_5min': revenue,
    'ttl': int(time.time()) + 3600  # Xóa sau 1 giờ (khi batch job mới chạy xong)
}
dynamodb.put_item(Item=dynamodb_item)
```

### 4. Monitoring với CloudWatch

```python
# Metrics quan trọng cần theo dõi
cloudwatch_metrics = [
    "BatchLayerLag",        # Độ trễ của batch job so với thực tế
    "SpeedLayerLatency",    # Thời gian xử lý của speed layer
    "ServingLayerQPS",      # Queries per second tại serving layer
    "MergeErrorRate",       # Tỷ lệ lỗi khi merge batch + real-time views
]
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Câu 1: "Lambda Architecture là gì và tại sao cần tới 3 lớp?"

**Gợi ý trả lời:**

Lambda Architecture là kiến trúc xử lý dữ liệu dùng ba lớp để cân bằng giữa tốc độ và độ chính xác:

- **Batch Layer** lưu toàn bộ dữ liệu thô bất biến và tính toán batch views chính xác. Độ trễ cao (giờ/ngày) nhưng kết quả 100% chính xác và có thể tính lại khi cần.

- **Speed Layer** xử lý dữ liệu mới nhất với độ trễ thấp (giây). Kết quả gần đúng vì không có đầy đủ ngữ cảnh lịch sử. Bù đắp khoảng thời gian batch chưa cập nhật.

- **Serving Layer** hợp nhất kết quả từ hai lớp trên để client nhận được cả dữ liệu chính xác lẫn real-time.

Ba lớp là cần thiết vì không có công nghệ đơn lẻ nào đáp ứng được cả hai yêu cầu: xử lý lại toàn bộ lịch sử với độ chính xác cao **và** cung cấp kết quả trong vài giây.

---

### Câu 2: "Nhược điểm lớn nhất của Lambda Architecture là gì? Bạn sẽ giải quyết thế nào?"

**Gợi ý trả lời:**

Nhược điểm lớn nhất là **code duplication** (sao chép code) — phải implement cùng một business logic hai lần, một cho batch (thường dùng Spark/SQL) và một cho stream (dùng Flink/KDA). Hai implementation dễ bị diverge theo thời gian dẫn đến kết quả không nhất quán.

Các hướng giải quyết:
1. **Unified API**: Dùng framework như Apache Beam (hỗ trợ cả batch và stream với một codebase)
2. **Chuyển sang Kappa Architecture** nếu stream platform đủ mạnh để replay lịch sử
3. **Shared transformation library**: Tách logic nghiệp vụ thành library dùng chung cho cả hai pipeline
4. **Delta Lake / Apache Iceberg**: Cho phép upsert (cập nhật + chèn) trong data lake, giảm nhu cầu speed layer riêng biệt

---

### Câu 3: "Khi nào bạn chọn Lambda thay vì Kappa Architecture?"

**Gợi ý trả lời:**

Tôi chọn Lambda Architecture khi:

1. **Yêu cầu reprocessing nghiêm ngặt**: Hệ thống tài chính cần recompute (tính lại) toàn bộ lịch sử khi phát hiện lỗi logic — Batch Layer với dữ liệu bất biến là lý tưởng cho điều này.

2. **Stream platform không hỗ trợ long retention**: Nếu tổ chức chỉ có Kinesis với retention 7 ngày, không thể replay để reprocess dữ liệu 1 năm trước — cần Batch Layer riêng.

3. **Có sự khác biệt lớn về computation**: Batch job cần tính toán rất phức tạp (ML feature engineering, complex joins nhiều bảng) mà stream processing engine không xử lý hiệu quả.

Ngược lại, nếu stream platform hỗ trợ long retention (Kafka với retention nhiều tháng/năm) và đủ mạnh để replay, tôi sẽ ưu tiên Kappa Architecture để giảm độ phức tạp vận hành.

---

### Câu 4: "Thiết kế Lambda Architecture cho hệ thống fraud detection (phát hiện gian lận) real-time"

**Gợi ý trả lời — Thiết kế cấp cao:**

```
Batch Layer:
- S3 lưu toàn bộ transaction history (lịch sử giao dịch)
- EMR Spark chạy hàng đêm: train ML model, tính user behavior profiles
- Batch views: risk scores theo merchant, địa lý, giờ trong ngày

Speed Layer:
- Kinesis Data Streams nhận transaction events thời gian thực
- Kinesis Data Analytics (Flink): tính velocity (tần suất giao dịch trong 1 phút), 
  phát hiện bất thường so với pattern lịch sử trong batch views

Serving Layer:
- ElastiCache Redis: cache risk scores để lookup nhanh (< 10ms)
- Fraud decision engine: kết hợp batch risk profile + real-time velocity
- Phán quyết: Allow / Flag / Block với confidence score
```

---

## 📊 Tổng Kết

```
Lambda Architecture = Batch Layer + Speed Layer + Serving Layer

Batch Layer   → Chính xác, chậm, cho lịch sử
Speed Layer   → Gần đúng, nhanh, cho hiện tại
Serving Layer → Hợp nhất hai nguồn, phục vụ client

Dùng khi: Cần cả batch lẫn stream, yêu cầu reprocessing, team đủ lớn
Không dùng khi: Team nhỏ, chỉ cần một loại processing, stream platform đủ mạnh
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [2-kappa-architecture.md](./2-kappa-architecture.md)
