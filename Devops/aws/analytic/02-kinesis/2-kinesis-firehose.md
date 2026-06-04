# 🚒 Kinesis Data Firehose — KDF (Vòi Dữ Liệu Kinesis)

> KDF — Kinesis Data Firehose — là dịch vụ **fully managed** (Được Quản Lý Hoàn Toàn) để **deliver** (Chuyển Phát) streaming data đến các điểm đến như S3, Redshift, OpenSearch mà không cần quản lý consumer code hay shard — chỉ cần cấu hình, Firehose tự lo phần còn lại.

## 📚 Mục Lục

1. [Tổng Quan KDF](#tổng-quan-kdf)
2. [Buffering — Cơ Chế Đệm Dữ Liệu](#buffering--cơ-chế-đệm-dữ-liệu)
3. [Các Điểm Đến Được Hỗ Trợ](#các-điểm-đến-được-hỗ-trợ)
4. [Data Transformation — Biến Đổi Dữ Liệu](#data-transformation--biến-đổi-dữ-liệu)
5. [Dynamic Partitioning — Phân Vùng Động](#dynamic-partitioning--phân-vùng-động)
6. [Cấu Hình Thực Tế](#cấu-hình-thực-tế)
7. [Xử Lý Lỗi & Dead Letter](#xử-lý-lỗi--dead-letter)
8. [Giám Sát](#giám-sát)
9. [Chi Phí](#chi-phí)
10. [KDS vs KDF — Quyết Định Nhanh](#kds-vs-kdf--quyết-định-nhanh)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tổng Quan KDF

### KDF Làm Gì

```
┌────────────────────────────────────────────────────────────┐
│                  Kinesis Data Firehose                      │
│                                                             │
│  Sources          Transform          Destinations           │
│  ──────           ─────────          ────────────           │
│  Direct PUT  ──►  Lambda     ──►    Amazon S3               │
│  KDS         ──►  (tùy chọn) ──►   Amazon Redshift         │
│  MSK         ──►             ──►   Amazon OpenSearch        │
│  CloudWatch  ──►             ──►   Splunk                   │
│  IoT Core    ──►             ──►   HTTP Endpoint            │
│  EventBridge ──►             ──►   Snowflake                │
│                              ──►   MongoDB                  │
└────────────────────────────────────────────────────────────┘
```

### KDF Khác KDS Ở Điểm Nào

| Khía Cạnh                  | KDS (Data Streams)              | KDF (Firehose)                  |
| -------------------------- | ------------------------------- | ------------------------------- |
| **Managed**                | Partially managed               | Fully managed                   |
| **Consumer code**          | Bạn tự viết consumer            | Không cần — KDF tự xử lý        |
| **Shard**                  | Bạn tự quản lý shard            | Không có shard concept          |
| **Độ trễ**                 | Real-time (sub-second)          | Near real-time (60s - 15 phút)  |
| **Destinations**           | Tùy ý (Lambda, EC2, ...)        | Các destination cố định         |
| **Replay**                 | Có (retention 1-365 ngày)       | Không                           |
| **Scaling**                | Thủ công (update shard count)   | Tự động                         |
| **Phù hợp**                | Custom processing, low-latency  | Ingestion pipeline đơn giản     |

### Khi Nào Chọn KDF

✅ **Dùng KDF khi:**
- Cần đẩy dữ liệu vào S3, Redshift, OpenSearch đơn giản
- Không cần xử lý real-time sub-second
- Muốn zero-ops — không quản lý infrastructure
- Data pipeline ít phức tạp (collect → transform nhẹ → store)

❌ **Không dùng KDF khi:**
- Cần nhiều consumers đọc cùng stream
- Cần replay dữ liệu
- Cần xử lý trong vài millisecond
- Cần custom routing phức tạp

---

## ⏱️ Buffering — Cơ Chế Đệm Dữ Liệu

### Tại Sao Cần Buffering

Thay vì ghi từng record một vào S3 (tốn kém, tạo ra hàng triệu file nhỏ), KDF **gom dữ liệu lại** (buffer) rồi ghi một lần:

```
Record 1 ──►
Record 2 ──►  [Buffer 5 MB]  ──► flush ──► S3 file (5MB)
Record 3 ──►      hoặc
...       ──►  [Buffer 60s]  ──► flush ──► S3 file
```

### Cấu Hình Buffer

| Tham Số                   | Phạm Vi          | Mặc Định  | Ghi Chú                              |
| ------------------------- | ---------------- | --------- | ------------------------------------ |
| **Buffer size** (kích thước đệm) | 1 MB – 128 MB  | 5 MB      | Flush khi đạt ngưỡng                 |
| **Buffer interval** (khoảng thời gian đệm) | 60s – 900s | 300s | Flush khi đạt ngưỡng thời gian       |

**Nguyên tắc flush:** Điều kiện nào đến trước thì flush trước.

```
Ví dụ: size=5MB, interval=60s
- Nếu 5MB tích lũy trong 30s → flush ngay (không chờ 60s)
- Nếu 60s trôi qua chỉ có 2MB → flush ngay (không chờ đủ 5MB)
```

### Tối Ưu Buffer Cho Use Case

```
Data Volume thấp (< 1 MB/phút):
→ interval nhỏ (60s) để không chờ quá lâu
→ size nhỏ (1MB)

Data Volume cao (> 100 MB/phút):
→ size lớn hơn (64-128 MB) để giảm số file S3
→ interval lớn hơn (300-900s)

Latency-sensitive (cần dữ liệu S3 sớm):
→ interval nhỏ (60s), size nhỏ
→ Nhưng chấp nhận nhiều file nhỏ → tốn query cost hơn khi Athena đọc
```

---

## 📍 Các Điểm Đến Được Hỗ Trợ

### 1. Amazon S3

**Use case phổ biến nhất.** Lưu trữ dữ liệu thô hoặc đã transform để query bằng Athena, xử lý bằng Glue, v.v.

```
KDF → S3: s3://bucket/prefix/YYYY/MM/DD/HH/filename
```

**Cấu hình định dạng:**

| Định Dạng       | Ưu Điểm                              | Khi Nào Dùng                     |
| --------------- | ------------------------------------ | -------------------------------- |
| **JSON**        | Đơn giản, dễ debug                   | Dev/test, dữ liệu đa dạng schema |
| **Parquet**     | Columnar — query nhanh, nén tốt      | Production, query thường xuyên   |
| **ORC**         | Columnar — tương tự Parquet          | Khi dùng Hive/Spark             |

```python
# Cấu hình KDF ghi Parquet ra S3 (thông qua Glue Data Catalog)
{
    "ExtendedS3DestinationConfiguration": {
        "BucketARN": "arn:aws:s3:::my-data-lake",
        "Prefix": "events/",
        "DataFormatConversionConfiguration": {
            "Enabled": True,
            "InputFormatConfiguration": {
                "Deserializer": {"OpenXJsonSerDe": {}}  # Input là JSON
            },
            "OutputFormatConfiguration": {
                "Serializer": {"ParquetSerDe": {}}  # Output là Parquet
            },
            "SchemaConfiguration": {
                "DatabaseName": "analytics_db",
                "TableName": "user_events",
                "RoleARN": "arn:aws:iam::...:role/FirehoseRole"
                # Lấy schema từ Glue Data Catalog
            }
        }
    }
}
```

### 2. Amazon Redshift

Đặc biệt hơn: KDF **không ghi thẳng vào Redshift** mà đi theo 2 bước:
1. Ghi vào S3 trung gian
2. Chạy lệnh `COPY` để load từ S3 vào Redshift

```
Producer → KDF → S3 (intermediate) → COPY command → Redshift
```

```python
{
    "RedshiftDestinationConfiguration": {
        "ClusterJDBCURL": "jdbc:redshift://cluster.region.redshift.amazonaws.com:5439/db",
        "CopyCommand": {
            "DataTableName": "user_events",
            "CopyOptions": "JSON 'auto' TIMEFORMAT 'epochmillisecs'"
        },
        "Username": "firehose_user",
        "Password": "...",
        "S3Configuration": {
            "BucketARN": "arn:aws:s3:::my-redshift-staging",
            "Prefix": "staging/",
            "BufferingHints": {"SizeInMBs": 128, "IntervalInSeconds": 300}
        }
    }
}
```

**Lưu ý:** Staging S3 files cần được cleanup sau khi COPY xong để tránh tích lũy.

### 3. Amazon OpenSearch Service

```
Producer → KDF → OpenSearch (index documents)
```

```python
{
    "ElasticsearchDestinationConfiguration": {
        "DomainARN": "arn:aws:es:...:domain/my-opensearch",
        "IndexName": "logs-{yyyy.MM.dd}",  # Index theo ngày
        "TypeName": "_doc",
        "BufferingHints": {
            "IntervalInSeconds": 60,
            "SizeInMBs": 5
        },
        "RetryOptions": {"DurationInSeconds": 300},
        "S3BackupMode": "FailedDocumentsOnly"  # Lưu documents lỗi vào S3
    }
}
```

### 4. HTTP Endpoint (Điểm Cuối HTTP Tùy Chỉnh)

Gửi dữ liệu đến bất kỳ HTTP endpoint nào (Datadog, New Relic, MongoDB Atlas, v.v.):

```python
{
    "HttpEndpointDestinationConfiguration": {
        "EndpointConfiguration": {
            "Url": "https://my-api.example.com/ingest",
            "AccessKey": "my-api-key"
        },
        "BufferingHints": {
            "SizeInMBs": 5,
            "IntervalInSeconds": 60
        },
        "RetryOptions": {"DurationInSeconds": 300}
    }
}
```

---

## 🔄 Data Transformation — Biến Đổi Dữ Liệu

### Transformation Với Lambda

KDF có thể gọi Lambda để transform mỗi batch records **trước khi ghi**:

```
Producer → KDF Buffer → Lambda Transform → KDF → S3/Redshift/...
```

**Lambda nhận và trả về:**

```python
def lambda_handler(event, context):
    output = []
    
    for record in event['records']:
        # Decode dữ liệu đầu vào (base64)
        payload = base64.b64decode(record['data']).decode('utf-8')
        data = json.loads(payload)
        
        # ==== Transformation logic ====
        # 1. Lọc records không hợp lệ
        if 'user_id' not in data:
            output.append({
                'recordId': record['recordId'],
                'result': 'Dropped',  # Bỏ qua record này
                'data': record['data']
            })
            continue
        
        # 2. Enrich data — thêm trường mới
        data['processed_at'] = datetime.utcnow().isoformat()
        data['source_region'] = 'ap-southeast-1'
        
        # 3. Normalize — chuẩn hóa
        data['user_id'] = data['user_id'].lower().strip()
        data['event_type'] = data.get('event_type', 'unknown')
        
        # 4. Thêm newline (quan trọng cho S3 JSON files)
        transformed = json.dumps(data) + '\n'
        
        output.append({
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': base64.b64encode(transformed.encode('utf-8')).decode('utf-8')
        })
    
    return {'records': output}

# Kết quả Lambda:
# - 'Ok': Record được ghi vào destination
# - 'Dropped': Record bị bỏ qua (không ghi, không báo lỗi)
# - 'ProcessingFailed': Record lỗi → ghi vào S3 backup bucket
```

### Compression (Nén Dữ Liệu)

KDF hỗ trợ nén dữ liệu trước khi ghi vào S3:

```python
{
    "S3DestinationConfiguration": {
        "CompressionFormat": "GZIP"   # GZIP, SNAPPY, ZIP, Hadoop-Compatible SNAPPY
        # Parquet/ORC tự nén nội bộ → không cần đặt CompressionFormat
    }
}
```

---

## 🗂️ Dynamic Partitioning — Phân Vùng Động

### Vấn Đề Với Static Prefix

```
Cách cũ (static prefix):
s3://bucket/events/2026/05/17/...
                   ↑ Tất cả events → cùng folder → không thể query theo region
```

### Dynamic Partitioning Là Gì

Dynamic Partitioning (Phân Vùng Động) cho phép KDF **tự động tạo S3 prefix** dựa trên nội dung record — giúp phân vùng dữ liệu thông minh hơn:

```
s3://bucket/region=ap-southeast-1/event_type=purchase/date=2026-05-17/...
s3://bucket/region=us-east-1/event_type=page_view/date=2026-05-17/...
```

### Cấu Hình Dynamic Partitioning

```python
{
    "ExtendedS3DestinationConfiguration": {
        "BucketARN": "arn:aws:s3:::my-data-lake",
        
        # Dynamic prefix dùng JQ expressions trích xuất từ record
        "Prefix": "region=!{partitionKeyFromQuery:region}/event_type=!{partitionKeyFromQuery:event_type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
        "ErrorOutputPrefix": "errors/!{firehose:error-output-type}/",
        
        "DynamicPartitioningConfiguration": {
            "Enabled": True,
            "RetryOptions": {"DurationInSeconds": 300}
        },
        
        "ProcessingConfiguration": {
            "Enabled": True,
            "Processors": [
                {
                    "Type": "MetadataExtraction",
                    "Parameters": [
                        {
                            "ParameterName": "MetadataExtractionQuery",
                            "ParameterValue": "{region:.region, event_type:.event_type}"
                            # JQ query trích xuất field từ JSON record
                        },
                        {
                            "ParameterName": "JsonParsingEngine",
                            "ParameterValue": "JQ-1.6"
                        }
                    ]
                }
            ]
        }
    }
}
```

**Kết quả:** Athena có thể query theo partition cực kỳ hiệu quả:

```sql
SELECT *
FROM user_events
WHERE region = 'ap-southeast-1'
  AND event_type = 'purchase'
  AND year = '2026'
  AND month = '05'
-- → Athena chỉ quét đúng folder cần → giảm chi phí đáng kể
```

---

## 🛠️ Cấu Hình Thực Tế

### Tạo Delivery Stream Với AWS CDK

```python
from aws_cdk import (
    aws_kinesisfirehose as firehose,
    aws_kinesisfirehose_alpha as firehose_alpha,  # L2 constructs
    aws_s3 as s3,
    aws_iam as iam,
    aws_lambda as lambda_
)

# S3 bucket đích
bucket = s3.Bucket(self, "AnalyticsBucket",
    versioned=True,
    lifecycle_rules=[
        s3.LifecycleRule(
            transitions=[
                s3.Transition(
                    storage_class=s3.StorageClass.INTELLIGENT_TIERING,
                    transition_after=Duration.days(30)
                )
            ]
        )
    ]
)

# Lambda transformer
transformer_fn = lambda_.Function(self, "TransformerFn",
    runtime=lambda_.Runtime.PYTHON_3_12,
    handler="transformer.lambda_handler",
    code=lambda_.Code.from_asset("lambda/transformer"),
    timeout=Duration.minutes(3)  # KDF timeout tối đa 3 phút
)

# IAM role cho Firehose
firehose_role = iam.Role(self, "FirehoseRole",
    assumed_by=iam.ServicePrincipal("firehose.amazonaws.com")
)
bucket.grant_read_write(firehose_role)
transformer_fn.grant_invoke(firehose_role)

# Tạo Delivery Stream
delivery_stream = firehose.CfnDeliveryStream(
    self, "UserEventsFirehose",
    delivery_stream_name="user-events-firehose",
    delivery_stream_type="DirectPut",  # Hoặc KinesisStreamAsSource
    extended_s3_destination_configuration=firehose.CfnDeliveryStream.ExtendedS3DestinationConfigurationProperty(
        bucket_arn=bucket.bucket_arn,
        role_arn=firehose_role.role_arn,
        prefix="events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
        error_output_prefix="errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/",
        buffering_hints=firehose.CfnDeliveryStream.BufferingHintsProperty(
            size_in_m_bs=64,
            interval_in_seconds=300
        ),
        compression_format="GZIP",
        processing_configuration=firehose.CfnDeliveryStream.ProcessingConfigurationProperty(
            enabled=True,
            processors=[
                firehose.CfnDeliveryStream.ProcessorProperty(
                    type="Lambda",
                    parameters=[
                        firehose.CfnDeliveryStream.ProcessorParameterProperty(
                            parameter_name="LambdaArn",
                            parameter_value=transformer_fn.function_arn
                        ),
                        firehose.CfnDeliveryStream.ProcessorParameterProperty(
                            parameter_name="BufferSizeInMBs",
                            parameter_value="3"
                        ),
                        firehose.CfnDeliveryStream.ProcessorParameterProperty(
                            parameter_name="BufferIntervalInSeconds",
                            parameter_value="60"
                        )
                    ]
                )
            ]
        )
    )
)
```

### Gửi Dữ Liệu Vào KDF

```python
import boto3
import json

firehose = boto3.client('firehose', region_name='ap-southeast-1')

# Gửi 1 record
def send_event(event: dict):
    response = firehose.put_record(
        DeliveryStreamName='user-events-firehose',
        Record={
            'Data': (json.dumps(event) + '\n').encode('utf-8')
            # Thêm '\n' để mỗi record trên 1 dòng trong file S3
        }
    )
    return response

# Gửi batch (tối đa 500 records hoặc 4MB)
def send_events_batch(events: list):
    records = [
        {'Data': (json.dumps(e) + '\n').encode('utf-8')}
        for e in events
    ]
    
    response = firehose.put_record_batch(
        DeliveryStreamName='user-events-firehose',
        Records=records
    )
    
    if response['FailedPutCount'] > 0:
        # Retry records thất bại
        failed_indices = [
            i for i, r in enumerate(response['RequestResponses'])
            if 'ErrorCode' in r
        ]
        failed_records = [events[i] for i in failed_indices]
        send_events_batch(failed_records)  # Đệ quy retry
    
    return response
```

### KDS Làm Nguồn Cho KDF

Thay vì gửi trực tiếp vào KDF, có thể dùng KDS làm nguồn:

```python
# Cấu hình KDF đọc từ KDS
{
    "DeliveryStreamType": "KinesisStreamAsSource",
    "KinesisStreamSourceConfiguration": {
        "KinesisStreamARN": "arn:aws:kinesis:ap-southeast-1:...:stream/my-kds-stream",
        "RoleARN": "arn:aws:iam::...:role/FirehoseKDSRole"
    }
}
```

**Lợi ích pattern này:**
- KDS xử lý real-time (Lambda, EC2)
- KDF đồng thời lưu trữ vào S3 → historical analysis
- Không cần gửi dữ liệu 2 lần

---

## ⚠️ Xử Lý Lỗi & Dead Letter

### Các Loại Lỗi

```
1. Lambda transformation lỗi → 'ProcessingFailed'
2. Destination không tiếp nhận → KDF retry
3. Record không hợp lệ (quá lớn, sai format) → ghi vào error prefix

Khi nào KDF ghi vào Error Output Prefix:
- Lambda trả về 'ProcessingFailed'
- S3 put lỗi sau hết retry
- Record vượt giới hạn kích thước
- Destination trả về lỗi 4xx
```

### Cấu Hình Error Handling

```python
{
    "ExtendedS3DestinationConfiguration": {
        "Prefix": "events/",
        "ErrorOutputPrefix": "errors/!{firehose:error-output-type}/!{timestamp:yyyy}/!{timestamp:MM}/",
        # error-output-type: processing-failed, S3-access-denied, S3-connection-timeout, etc.
        
        "S3BackupMode": "Enabled",         # Backup ALL records vào S3 riêng
        # Hoặc "FailedDataOnly"            # Chỉ backup records lỗi
        
        "S3BackupConfiguration": {
            "BucketARN": "arn:aws:s3:::my-backup-bucket",
            "Prefix": "backup/",
            "BufferingHints": {"SizeInMBs": 5, "IntervalInSeconds": 60}
        }
    }
}
```

### Retry Logic (Logic Thử Lại) Của KDF

```
KDF retry behavior:
- Destination không available → retry trong tối đa [retry_duration] giây
- Mặc định: 300 giây (5 phút)
- Tối đa: 7200 giây (2 giờ)
- Sau khi hết retry → ghi vào S3 error prefix
- Dữ liệu KHÔNG bị mất — chỉ bị delay hoặc chuyển sang error

Lưu ý: KDF không có DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) như Lambda
→ Phải dùng error S3 prefix + alert để xử lý
```

---

## 📊 Giám Sát

### CloudWatch Metrics Quan Trọng

| Metric                              | Ý Nghĩa                                   | Ngưỡng Cảnh Báo          |
| ----------------------------------- | ------------------------------------------ | ------------------------- |
| `DeliveryToS3.Success`              | Số bytes ghi vào S3 thành công             | Giảm đột ngột             |
| `DeliveryToS3.DataFreshness`        | Thời gian data "cũ nhất" trong buffer (s) | > buffer_interval × 2    |
| `ThrottledRecords`                  | Số records bị throttle                     | > 0                       |
| `DeliveryToS3.Records`              | Số records ghi vào S3                      | Monitoring trend           |
| `FailedConversion.Records`          | Records lỗi khi convert sang Parquet/ORC   | > 0                       |
| `ExecuteProcessing.Duration`        | Thời gian Lambda transform                 | Gần 3 phút (timeout limit)|

```python
# CloudWatch Alarm cho DataFreshness
cloudwatch.put_metric_alarm(
    AlarmName='FirehoseDataStaleness',
    MetricName='DeliveryToS3.DataFreshness',
    Namespace='AWS/Firehose',
    Dimensions=[{'Name': 'DeliveryStreamName', 'Value': 'user-events-firehose'}],
    Period=60,
    EvaluationPeriods=3,
    Threshold=600,  # Cảnh báo nếu data > 10 phút chưa deliver
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=['arn:aws:sns:...:firehose-alerts']
)
```

---

## 💰 Chi Phí

### Mô Hình Tính Phí

| Thành Phần                              | Giá (ap-southeast-1)        |
| --------------------------------------- | --------------------------- |
| **Data ingestion** (Thu Nạp Dữ Liệu)   | ~$0.029/GB                  |
| **Dynamic Partitioning**                | ~$0.02/GB thêm              |
| **Format conversion** (Parquet/ORC)     | ~$0.018/GB                  |
| **VPC delivery** (nếu cần)             | ~$0.01/GB + VPC endpoint    |

### So Sánh Chi Phí KDF vs KDS + Custom Consumer

```
Tình huống: 100 GB/ngày, ghi vào S3

KDF:
  Data ingestion: 100GB × $0.029 = $2.9/ngày = ~$87/tháng
  (Không tốn phí shard, không tốn phí compute cho consumer)

KDS + Lambda Consumer:
  KDS shard giờ: 3 shards × $0.015 × 24 = $1.08/ngày
  Lambda invocations: (tùy throughput) ~$0.5-2/ngày
  Tổng: ~$1.6-3/ngày = ~$50-90/tháng

→ Chi phí tương đương, nhưng KDF đơn giản hơn nhiều
→ KDS mạnh hơn khi cần real-time processing + nhiều consumer
```

### Tối Ưu Chi Phí KDF

1. **Tăng buffer size:** Ít files S3 hơn → giảm S3 PUT requests (rẻ)
2. **Dùng Parquet:** Nén tốt hơn JSON → ít GB hơn → rẻ hơn khi query Athena
3. **Dynamic Partitioning thận trọng:** Tốn $0.02/GB thêm — chỉ dùng khi thực sự cần
4. **Compress:** Gzip giảm kích thước 60-80% → giảm S3 storage cost

---

## ⚖️ KDS vs KDF — Quyết Định Nhanh

```
┌─────────────────────────────────────────────────────────────┐
│                   CÂU HỎI QUYẾT ĐỊNH                        │
└─────────────────────────────────────────────────────────────┘

Cần nhiều consumers đọc cùng stream?
    Có → KDS
    Không ↓

Cần replay dữ liệu?
    Có → KDS
    Không ↓

Cần xử lý real-time (< 1 giây)?
    Có → KDS
    Không ↓

Chỉ cần đẩy data vào S3/Redshift/OpenSearch?
    Có → KDF ✅ (fully managed, zero ops)
    Không → KDS (custom consumer)
```

### Kết Hợp KDS + KDF (Pattern Phổ Biến)

```
Producer
    │
    ▼
Kinesis Data Streams (KDS)
    │
    ├──► Lambda (real-time processing, alerts, fraud detection)
    │
    ├──► KDA/Flink (windowed aggregations, ML inference)
    │
    └──► Kinesis Firehose (KDF)
              │
              └──► S3 (lưu trữ historical data)
                       │
                       └──► Athena, Glue, EMR (batch analytics)
```

**Ưu điểm:** KDS xử lý real-time, KDF đồng thời lưu trữ — gửi data 1 lần, nhận 2 giá trị.

---

## 🎯 Câu Hỏi Phỏng Vấn

### Q1: KDF deliver dữ liệu vào Redshift như thế nào?

**A:** KDF không ghi thẳng vào Redshift mà theo 2 bước:
1. Buffer và ghi records vào S3 (staging bucket)
2. Tự động chạy lệnh `COPY` để load từ S3 vào bảng Redshift

Điều này có nghĩa là: (1) Redshift phải có quyền đọc S3 staging; (2) Có độ trễ thêm từ S3 COPY operation; (3) Nên cleanup staging S3 files sau khi load.

### Q2: Lambda transform trong KDF hoạt động thế nào, có thể Drop records không?

**A:** KDF gọi Lambda với batch records. Lambda trả về từng record với status:
- `"Ok"` → ghi vào destination
- `"Dropped"` → bỏ qua (không ghi, không báo lỗi) — dùng để filter records
- `"ProcessingFailed"` → ghi vào error S3 prefix

Giới hạn: Lambda timeout tối đa 3 phút; payload tối đa 6 MB.

### Q3: Dynamic Partitioning trong KDF mang lại lợi ích gì?

**A:** Dynamic Partitioning tự động tạo S3 prefix dựa trên nội dung record (ví dụ: partition theo `region`, `event_type`, `date`). Lợi ích:
- Athena query chỉ quét đúng partition cần → giảm chi phí (tính theo TB quét)
- Không cần Glue ETL job để re-partition dữ liệu
- Data nằm đúng chỗ ngay khi arrive

Chi phí thêm: ~$0.02/GB — đáng dùng nếu query pattern rõ ràng.

### Q4: Khi nào dùng Direct PUT vs KDS-as-source cho Firehose?

**A:**
- **Direct PUT:** Producer gửi thẳng vào KDF. Đơn giản, ít latency overhead. Phù hợp khi chỉ cần lưu data.
- **KDS-as-source:** KDF đọc từ KDS stream. Phù hợp khi đã có KDS cho real-time processing và muốn đồng thời lưu vào S3 — tránh phải gửi data 2 lần. KDF đọc từ KDS không tốn thêm chi phí KDS read.

---

## 📊 Tóm Tắt Nhanh

```
KDF — Kinesis Data Firehose
├── Fully managed — không cần quản lý infrastructure
├── Near real-time (60s – 15 phút tùy buffer)
├── Destinations: S3, Redshift, OpenSearch, HTTP, Snowflake...
├── Transform: Lambda (filter, enrich, convert format)
├── Format: JSON → Parquet/ORC (dùng Glue Catalog cho schema)
├── Dynamic Partitioning: tạo S3 prefix từ record content
└── Error handling: error S3 prefix (không có DLQ)

KDF vs KDS:
├── KDF: zero-ops, near real-time, 1 destination per stream
└── KDS: custom consumer, real-time, multi-consumer, replay
```

---

## 🔗 Điều Hướng

| Tiếp Theo                                              | Quay Lại                                        |
| ------------------------------------------------------ | ----------------------------------------------- |
| [3-kinesis-analytics.md](./3-kinesis-analytics.md)     | [1-kinesis-data-streams.md](./1-kinesis-data-streams.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** 2-kinesis-firehose.md
**Trạng Thái:** ✅ Hoàn thành
