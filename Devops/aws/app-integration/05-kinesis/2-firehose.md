# Kinesis Data Firehose — Giao Vận Dữ Liệu Gần Thời Gian Thực

> **Kinesis Data Firehose** (trước đây là Amazon Kinesis Data Firehose, nay đổi tên thành **Amazon Data Firehose**) là dịch vụ fully-managed (được quản lý hoàn toàn) để giao vận data streaming vào các dịch vụ lưu trữ và phân tích như S3, Redshift, OpenSearch, không cần viết consumer code.

---

## 📚 Mục Lục

1. [Tổng Quan Firehose](#tổng-quan-firehose)
2. [Kiến Trúc Và Luồng Dữ Liệu](#kiến-trúc-và-luồng-dữ-liệu)
3. [Sources — Nguồn Dữ Liệu](#sources--nguồn-dữ-liệu)
4. [Destinations — Đích Đến](#destinations--đích-đến)
5. [Buffering — Cơ Chế Đệm Dữ Liệu](#buffering--cơ-chế-đệm-dữ-liệu)
6. [Data Transformation — Biến Đổi Dữ Liệu](#data-transformation--biến-đổi-dữ-liệu)
7. [Format Conversion — Chuyển Đổi Định Dạng](#format-conversion--chuyển-đổi-định-dạng)
8. [Error Handling — Xử Lý Lỗi](#error-handling--xử-lý-lỗi)
9. [KDS vs Firehose — So Sánh](#kds-vs-firehose--so-sánh)
10. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Firehose

### Firehose Giải Quyết Bài Toán Gì?

Bài toán: Bạn có log server cần lưu vào S3 để phân tích sau, nhưng:
- Không muốn viết và duy trì consumer code
- Không cần xử lý từng record riêng lẻ
- Chấp nhận độ trễ vài chục giây đến vài phút

→ **Firehose** là giải pháp: chỉ cần cấu hình destination, Firehose tự lo phần còn lại.

### Đặc Điểm Nổi Bật

```
✅ Fully managed — không cần quản lý server, shard, consumer
✅ Near-real-time — buffering 60–900 giây (không phải real-time như KDS)
✅ Auto-scaling — tự động scale theo throughput
✅ Built-in transformation — tích hợp Lambda để transform record
✅ Format conversion — tự động chuyển JSON sang Parquet/ORC
✅ Error backup — lưu record lỗi vào S3 riêng
✅ Compression — GZIP, Snappy, Zip tự động
```

---

## Kiến Trúc Và Luồng Dữ Liệu

### Sơ Đồ Tổng Quan

```
Source (Nguồn)
──────────────
Direct PUT API          ┌─────────────────────────────────────────┐
Kinesis Data Streams    │         Kinesis Data Firehose           │
Amazon MSK        ────▶ │                                         │
AWS IoT           ────▶ │  Buffer         Transform    Destination│
CloudWatch Logs   ────▶ │  (Đệm)   ─────▶ (Lambda) ─▶ (Đích)    │
AWS WAF                 │  60–900s         Optional    S3         │
EventBridge             │                              Redshift   │
                        │                    Backup   OpenSearch  │
                        │                     S3 ◀── HTTP Endpoint│
                        └─────────────────────────────────────────┘
```

### Quá Trình Giao Vận

```
1. Records đến Firehose từ source
2. Firehose buffer (đệm) records theo size hoặc time
3. (Optional) Gọi Lambda để transform records
4. (Optional) Convert format (JSON → Parquet)
5. Deliver (giao vận) batch vào destination
6. Ghi backup records lỗi vào S3 error bucket
```

---

## Sources — Nguồn Dữ Liệu

### Direct PUT (Ghi Trực Tiếp)

```python
import boto3, json
from datetime import datetime

firehose_client = boto3.client('firehose', region_name='ap-southeast-1')

def send_to_firehose(delivery_stream_name: str, records: list):
    """Gửi batch records vào Firehose."""
    formatted = [
        {'Data': (json.dumps(record) + '\n').encode('utf-8')}  # '\n' để S3 dễ parse
        for record in records
    ]
    
    response = firehose_client.put_record_batch(
        DeliveryStreamName=delivery_stream_name,
        Records=formatted
    )
    
    failed_count = response['FailedPutCount']
    if failed_count > 0:
        print(f"Cảnh báo: {failed_count} records ghi thất bại")
    
    return response

# Ghi log events vào Firehose
send_to_firehose('application-logs', [
    {'level': 'INFO', 'service': 'order-api', 'message': 'Order created', 'orderId': '123'},
    {'level': 'ERROR', 'service': 'payment-api', 'message': 'Payment failed', 'code': 'CARD_DECLINED'},
])
```

### Từ Kinesis Data Streams

```
Firehose có thể dùng KDS làm source:
- KDS thu thập real-time data
- Firehose đọc từ KDS, buffer, rồi giao vào storage
- Ưu điểm: vừa có real-time consumer (Lambda), vừa có storage consumer (Firehose)
- Một KDS stream có thể có nhiều Firehose delivery streams
```

---

## Destinations — Đích Đến

### Amazon S3

```
Cấu hình lưu trữ phổ biến nhất:
- Tự động tạo prefix theo thời gian: s3://bucket/year=2026/month=05/day=18/hour=10/
- Hỗ trợ compression: GZIP, Snappy, ZIP, HADOOP_SNAPPY
- Hỗ trợ encryption (mã hóa) với AWS KMS
- Thường dùng làm data lake (hồ dữ liệu)
```

```json
{
  "S3Configuration": {
    "BucketARN": "arn:aws:s3:::my-data-lake",
    "Prefix": "events/!{partitionKeyFromQuery:year}/!{partitionKeyFromQuery:month}/",
    "ErrorOutputPrefix": "errors/!{firehose:error-output-type}/",
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "GZIP"
  }
}
```

### Amazon Redshift

```
Quy trình giao vận vào Redshift:
1. Firehose ghi file vào S3 trung gian
2. Firehose chạy COPY command để load vào Redshift table
3. Xóa file S3 trung gian (hoặc giữ lại nếu cấu hình)

Lưu ý: Redshift cần có Redshift cluster sẵn — không serverless tự động như S3
```

### Amazon OpenSearch Service

```
Dùng để full-text search và dashboard (Kibana/OpenSearch Dashboards):
- Tự động tạo index theo ngày: logs-2026.05.18
- Rotation policy: mỗi ngày, tuần, tháng tạo index mới
- Backup S3: luôn bật backup để tránh mất dữ liệu nếu OpenSearch lỗi
```

### HTTP Endpoint (Datadog, Splunk, New Relic...)

```
Giao vận tới bất kỳ HTTP endpoint nào:
- Content-Type: application/json
- Hỗ trợ access key cho authentication
- Retry với exponential backoff khi endpoint lỗi
- Backup records thất bại sang S3
```

---

## Buffering — Cơ Chế Đệm Dữ Liệu

### Buffer Size và Buffer Interval

```
Firehose giao vận khi ĐẠT MỘT TRONG HAI điều kiện:

┌──────────────────────────────────┐
│  Buffer Size (Kích Thước Đệm)    │  → Đạt trước  ─┐
│  Ví dụ: 128 MB                   │                  ├──▶ DELIVER
│                                  │                  │
│  Buffer Interval (Khoảng Đệm)    │  → Đạt trước  ─┘
│  Ví dụ: 300 giây (5 phút)        │
└──────────────────────────────────┘
```

### Giới Hạn Buffer Theo Destination

| Destination | Buffer Size | Buffer Interval |
|---|---|---|
| Amazon S3 | 1–128 MB | 60–900 giây |
| Amazon Redshift | 1–128 MB | 60–900 giây |
| Amazon OpenSearch | 1–100 MB | 60–900 giây |
| Splunk | 1–5 MB | 60–900 giây |
| HTTP Endpoint | 1–128 MB | 60–900 giây |

### Tại Sao Không Phải Real-Time?

```
Firehose PHẢI buffer tối thiểu 60 giây
→ Đây là giới hạn thiết kế, không thể thay đổi

Nếu cần real-time (<1 giây):
→ Dùng Kinesis Data Streams + Lambda

Nếu chấp nhận ~1–15 phút:
→ Firehose đủ tốt và đơn giản hơn nhiều
```

---

## Data Transformation — Biến Đổi Dữ Liệu

### Lambda Transformation (Biến Đổi Bằng Lambda)

```
Records → Firehose → Lambda (transform) → Destination

Lambda nhận batch records, xử lý, trả về transformed records.
Firehose giao vận kết quả đã transform.
```

```python
# Lambda function để transform Firehose records
import base64, json

def lambda_handler(event, context):
    output_records = []
    
    for record in event['records']:
        # Decode record từ base64
        payload = base64.b64decode(record['data']).decode('utf-8')
        data = json.loads(payload)
        
        # Thực hiện transformation
        transformed = transform_record(data)
        
        # Encode lại và trả về
        # Status: 'Ok', 'Dropped', 'ProcessingFailed'
        output_records.append({
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(
                (json.dumps(transformed) + '\n').encode('utf-8')
            ).decode('utf-8')
        })
    
    return {'records': output_records}

def transform_record(data: dict) -> dict:
    """Ví dụ: thêm trường, đổi tên field, lọc field nhạy cảm."""
    return {
        'timestamp': data.get('ts'),
        'event_type': data.get('type'),
        'user_id': data.get('uid'),
        'amount': data.get('amt'),
        # Loại bỏ các field nhạy cảm như PII (Personal Identifiable Information)
    }
```

### Record Transformation Outcomes (Kết Quả Biến Đổi)

| Kết Quả | Ý Nghĩa |
|---|---|
| `Ok` | Record đã transform thành công, giao vận |
| `Dropped` | Record bị bỏ qua có chủ ý, không giao vận |
| `ProcessingFailed` | Lambda lỗi khi xử lý → backup sang S3 error |

---

## Format Conversion — Chuyển Đổi Định Dạng

### JSON → Apache Parquet / Apache ORC

```
Lợi ích của Parquet/ORC so với JSON:
- Nén tốt hơn: giảm 70–85% kích thước file
- Columnar storage: query nhanh hơn 10–100x trên Amazon Athena
- Tương thích tốt với AWS Glue, Athena, EMR, Redshift Spectrum

Yêu cầu:
- Cần AWS Glue Data Catalog để định nghĩa schema
- Glue Crawler tự động phát hiện schema, hoặc tự tạo table
```

```
JSON format (1 GB):
{"userId": "u1", "event": "click", "page": "/home", "ts": 1716000000}
{"userId": "u2", "event": "purchase", "page": "/checkout", "ts": 1716000001}
...

→ Sau format conversion (Parquet):
Kích thước: ~150 MB (giảm 85%)
Query Athena: SELECT COUNT(*) WHERE event='purchase'
Chỉ đọc cột 'event' — nhanh hơn nhiều so với JSON
```

---

## Error Handling — Xử Lý Lỗi

### Backup S3 — Sao Lưu Vào S3

```
Hai chế độ backup:
1. FailedDataOnly: chỉ backup records giao vận thất bại
2. AllData: backup tất cả records (trước khi giao vận đến destination)

Nên dùng chế độ nào?
→ AllData cho môi trường production — đảm bảo không mất dữ liệu
→ FailedDataOnly tiết kiệm chi phí S3 hơn
```

### Retry Behavior (Hành Vi Thử Lại)

```
Khi destination không available:
- Firehose retry theo cấu hình (ví dụ: 0–7200 giây)
- Trong thời gian retry, data vẫn buffer trong Firehose
- Sau retry timeout → record được backup vào S3 error bucket
- Độ trễ giao vận tăng lên nhưng không mất dữ liệu

Error prefix ví dụ trong S3:
s3://bucket/errors/processing-failed/2026/05/18/10/
s3://bucket/errors/splunk-failed/2026/05/18/10/
```

---

## KDS vs Firehose — So Sánh

| Tiêu Chí | Kinesis Data Streams | Kinesis Data Firehose |
|---|---|---|
| **Latency (Độ Trễ)** | Real-time (~200ms) | Near-real-time (60–900s) |
| **Consumer** | Tự viết (Lambda, KCL, custom) | Fully managed, không cần code |
| **Destinations** | Bất kỳ (code tự xử lý) | S3, Redshift, OpenSearch, HTTP, Splunk |
| **Replay** | Có (trong retention period) | Không |
| **Scaling** | Tự quản lý shard (hoặc On-Demand) | Auto-scaling tự động |
| **Transformation** | Tự code | Lambda integration built-in |
| **Format conversion** | Không | JSON → Parquet/ORC |
| **Chi phí** | Theo shard-hour + PUT | Theo GB ingested |
| **Complexity** | Cao hơn | Thấp hơn |
| **Dùng khi nào** | Custom processing, real-time | Storage delivery, analytics pipeline |

### Kết Hợp KDS + Firehose

```
KDS Stream
    │
    ├──▶ Lambda Consumer (real-time alerts, fraud detection)
    │
    └──▶ Firehose Delivery Stream
              │
              ├──▶ S3 (data lake)
              ├──▶ Redshift (data warehouse)
              └──▶ OpenSearch (search & dashboard)

→ Vừa có real-time processing, vừa có persistent storage
→ Đây là pattern phổ biến trong production
```

---

## Ví Dụ Thực Tế

### Pipeline Thu Thập Application Logs

```python
# Ví dụ: thu thập access log từ web servers
import boto3, json, time
from datetime import datetime

firehose = boto3.client('firehose', region_name='ap-southeast-1')

class LogCollector:
    def __init__(self, delivery_stream: str, batch_size: int = 500):
        self.stream = delivery_stream
        self.batch_size = batch_size
        self.buffer = []
    
    def add_log(self, log_entry: dict):
        log_entry['ingested_at'] = datetime.utcnow().isoformat()
        self.buffer.append({'Data': (json.dumps(log_entry) + '\n').encode()})
        
        if len(self.buffer) >= self.batch_size:
            self.flush()
    
    def flush(self):
        if not self.buffer:
            return
        
        response = firehose.put_record_batch(
            DeliveryStreamName=self.stream,
            Records=self.buffer[:500]  # Tối đa 500 records/call
        )
        
        failed = response['FailedPutCount']
        if failed > 0:
            print(f"WARN: {failed}/{len(self.buffer)} logs ghi thất bại")
        
        self.buffer = self.buffer[500:]

# Sử dụng
collector = LogCollector('application-logs-firehose')
collector.add_log({'method': 'GET', 'path': '/api/orders', 'status': 200, 'duration_ms': 45})
collector.add_log({'method': 'POST', 'path': '/api/payment', 'status': 500, 'duration_ms': 1200})
collector.flush()
```

### Pipeline IoT → S3 Data Lake

```
Kiến trúc:
IoT Device ──▶ IoT Core ──▶ Firehose ──▶ Lambda (enrich data) ──▶ S3 (Parquet)
                                              │
                                              ▼
                                         Glue Catalog ──▶ Athena (SQL query)

Lợi ích:
- IoT Core xử lý kết nối từ thiết bị
- Firehose buffer và transform
- Lambda thêm metadata (device type, location)
- S3 Parquet → query rẻ với Athena (trả tiền theo TB scanned)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Firehose khác KDS như thế nào?

**Trả lời:**
- KDS: real-time, cần viết consumer, hỗ trợ replay, nhiều consumer song song
- Firehose: near-real-time (60s+), fully managed, chỉ giao vào storage destinations cố định, không replay
- Chọn Firehose khi mục tiêu là lưu vào S3/Redshift/OpenSearch mà không muốn code consumer

### Q2: Tại sao Firehose không thể real-time?

**Trả lời:**
- Firehose PHẢI buffer tối thiểu 60 giây trước khi giao vận
- Đây là trade-off có chủ ý: đổi latency lấy simplicity và throughput tốt hơn
- Nếu cần <1 giây, phải dùng KDS với Lambda

### Q3: Làm thế nào để debug khi Firehose mất dữ liệu?

**Trả lời:**
- Kiểm tra S3 error bucket (backup prefix)
- Xem CloudWatch Metrics: `DeliveryToS3.Records`, `DeliveryToS3.DataFreshness`
- Check Lambda transformation errors nếu có dùng transform
- `DeliveryToS3.Success` và `DeliveryToS3.DataFreshness` là metrics quan trọng nhất

### Q4: Khi nào nên dùng format conversion (Parquet/ORC)?

**Trả lời:**
- Khi dữ liệu S3 sẽ được query thường xuyên bằng Athena hoặc Redshift Spectrum
- Parquet giảm 75–85% kích thước → giảm chi phí lưu trữ và query (Athena tính tiền theo TB scanned)
- Cần setup Glue Data Catalog trước — không thể tự động hoàn toàn

---

**Tiếp Theo:** [3-data-analytics.md](./3-data-analytics.md) — Kinesis Data Analytics, xử lý streaming với SQL và Apache Flink.
