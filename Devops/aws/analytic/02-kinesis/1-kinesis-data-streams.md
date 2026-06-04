# 🌊 Kinesis Data Streams — KDS (Luồng Dữ Liệu Kinesis)

> KDS — Kinesis Data Streams — là dịch vụ streaming có độ trễ thấp (sub-second latency — Độ Trễ Dưới Giây), cho phép nhiều consumer (Người Tiêu Thụ) đọc cùng stream đồng thời với khả năng tùy chỉnh cao và khả năng replay (Phát Lại) dữ liệu.

## 📚 Mục Lục

1. [Kiến Trúc KDS](#kiến-trúc-kds)
2. [Shard — Đơn Vị Throughput](#shard--đơn-vị-throughput)
3. [Producer — Gửi Dữ Liệu Vào Stream](#producer--gửi-dữ-liệu-vào-stream)
4. [Consumer — Đọc Dữ Liệu Từ Stream](#consumer--đọc-dữ-liệu-từ-stream)
5. [Partition Key & Ordering](#partition-key--ordering)
6. [Retention — Lưu Giữ Dữ Liệu](#retention--lưu-giữ-dữ-liệu)
7. [Scaling — Tăng Giảm Quy Mô](#scaling--tăng-giảm-quy-mô)
8. [Bảo Mật & Mã Hóa](#bảo-mật--mã-hóa)
9. [Tích Hợp Với AWS Services](#tích-hợp-với-aws-services)
10. [Giám Sát & Troubleshooting](#giám-sát--troubleshooting)
11. [Chi Phí](#chi-phí)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏗️ Kiến Trúc KDS

### Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                  Kinesis Data Stream                            │
│                                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │  │ Shard N │          │
│  │ [r1,r3] │  │ [r2,r6] │  │ [r4,r5] │  │  [...]  │          │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘          │
└───────┼────────────┼────────────┼────────────┼────────────────┘
        │            │            │            │
     (Producers gửi records — PartitionKey quyết định shard)
        │
        ▼
┌───────────────────────────────────────────────┐
│              Consumers (Người Tiêu Thụ)        │
│  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │
│  │  Lambda  │  │  Flink   │  │  EC2 App    │  │
│  │  (KCL)   │  │  (KDA)   │  │  (KCL/SDK) │  │
│  └──────────┘  └──────────┘  └─────────────┘  │
└───────────────────────────────────────────────┘
```

### Thành Phần Chính

| Thành Phần      | Mô Tả                                                                  |
| --------------- | ---------------------------------------------------------------------- |
| **Stream**      | Tên logic đại diện cho toàn bộ luồng dữ liệu                          |
| **Shard**       | Đơn vị throughput — mỗi shard có giới hạn read/write riêng            |
| **Record**      | Đơn vị dữ liệu gửi vào stream (tối đa 1 MB/record)                   |
| **SequenceNumber** | ID duy nhất tự sinh cho mỗi record trong shard                    |
| **PartitionKey** | Chuỗi quyết định shard nào nhận record                               |
| **ShardIterator** | Con trỏ xác định vị trí đọc trong shard                            |

### Vòng Đời Record

```
1. Producer.putRecord(data, partitionKey)
       ↓
2. KDS hash(partitionKey) → chọn shard
       ↓
3. Record được lưu trong shard (append-only log)
       ↓
4. Consumer đọc record qua ShardIterator
       ↓
5. Record bị xóa sau khi hết Retention Period
```

---

## 📦 Shard — Đơn Vị Throughput

### Giới Hạn Mỗi Shard

| Chiều            | Giới Hạn                                     | Lưu Ý                                   |
| ---------------- | -------------------------------------------- | --------------------------------------- |
| **Write (Ghi)**  | 1 MB/s hoặc 1,000 records/giây             | Vượt quá → `ProvisionedThroughputExceededException` |
| **Read (Đọc)**   | 2 MB/s (shared — Chia Sẻ Giữa Consumers)   | Standard Consumer: 5 GetRecords/s/shard |
| **Enhanced Fan-out** | 2 MB/s per consumer (riêng biệt)        | Mỗi consumer nhận băng thông riêng      |

### Tính Số Shard Cần Thiết

```
Số shard = max(
    ceil(write_MB_per_sec / 1.0),   ← giới hạn write
    ceil(read_MB_per_sec  / 2.0)    ← giới hạn read (standard)
)

Ví dụ thực tế:
- Có 500 IoT sensors, mỗi sensor gửi 5 KB/s
- Write throughput = 500 × 5KB = 2,500 KB/s = 2.5 MB/s
- Cần ít nhất: ceil(2.5 / 1.0) = 3 shards

- 3 consumer ứng dụng mỗi cái đọc 1.5 MB/s
- Read throughput = 3 × 1.5 = 4.5 MB/s (standard: chia sẻ 2MB/s)
- Cần: ceil(4.5 / 2.0) = 3 shards (nếu dùng standard)
- Hoặc dùng Enhanced Fan-out: mỗi consumer có riêng 2MB/s → đủ rồi
```

### Hai Chế Độ Capacity

#### 1. On-demand Mode (Chế Độ Theo Nhu Cầu)

- KDS tự động scale shards dựa trên throughput thực tế
- Không cần chỉ định số shard
- Phí cao hơn provisioned cho workload có thể dự đoán
- **Phù hợp khi:** Traffic không đều, không biết trước nhu cầu

#### 2. Provisioned Mode (Chế Độ Cấp Phát)

- Bạn chỉ định số shard cụ thể
- Trả phí theo shard-giờ (kể cả khi không dùng hết)
- **Phù hợp khi:** Traffic dự đoán được, muốn kiểm soát chi phí

```python
# Tạo stream với Provisioned Mode
kinesis = boto3.client('kinesis')
kinesis.create_stream(
    StreamName='my-iot-stream',
    ShardCount=3,
    StreamModeDetails={
        'StreamMode': 'PROVISIONED'  # hoặc 'ON_DEMAND'
    }
)
```

---

## 📤 Producer — Gửi Dữ Liệu Vào Stream

### Các Cách Gửi Dữ Liệu

#### 1. AWS SDK — PutRecord (Gửi Từng Record)

```python
import boto3
import json
import time

kinesis = boto3.client('kinesis', region_name='ap-southeast-1')

def send_event(user_id: str, event_type: str, data: dict):
    record = {
        'user_id': user_id,
        'event_type': event_type,
        'data': data,
        'timestamp': int(time.time() * 1000)
    }
    
    response = kinesis.put_record(
        StreamName='user-events-stream',
        Data=json.dumps(record).encode('utf-8'),
        PartitionKey=user_id  # Tất cả event của cùng user → cùng shard
    )
    
    print(f"Record sent to shard: {response['ShardId']}")
    print(f"Sequence number: {response['SequenceNumber']}")
    return response

# Gọi hàm
send_event(
    user_id='user-123',
    event_type='page_view',
    data={'page': '/products', 'session_id': 'sess-456'}
)
```

#### 2. AWS SDK — PutRecords (Gửi Hàng Loạt — Khuyến Nghị)

```python
def send_events_batch(events: list):
    records = []
    for event in events:
        records.append({
            'Data': json.dumps(event).encode('utf-8'),
            'PartitionKey': event['user_id']
        })
    
    # PutRecords gửi tối đa 500 records hoặc 5MB mỗi lần
    response = kinesis.put_records(
        StreamName='user-events-stream',
        Records=records
    )
    
    failed = response['FailedRecordCount']
    if failed > 0:
        print(f"Cảnh báo: {failed} records thất bại — cần retry")
        # Xử lý retry (Thử Lại) cho records thất bại
        handle_failed_records(response['Records'], records)
    
    return response

def handle_failed_records(results, original_records):
    failed_records = []
    for i, result in enumerate(results):
        if 'ErrorCode' in result:
            failed_records.append(original_records[i])
    
    if failed_records:
        # Retry với exponential backoff (Thử Lại Với Thời Gian Chờ Tăng Dần)
        time.sleep(1)
        send_events_batch(failed_records)
```

#### 3. Kinesis Agent (Tác Nhân Kinesis)

```yaml
# /etc/aws-kinesis/agent.json
# Kinesis Agent tự động đọc log file và gửi vào KDS
{
  "cloudwatch.emitMetrics": true,
  "kinesis.endpoint": "",
  "flows": [
    {
      "filePattern": "/var/log/nginx/access.log*",
      "kinesisStream": "nginx-access-logs",
      "partitionKeyOption": "RANDOM",
      "dataProcessingOptions": [
        {
          "optionName": "LOGTOJSON",
          "logEntryFormat": "COMMONAPACHELOG"
        }
      ]
    }
  ]
}
```

#### 4. Amazon Kinesis Producer Library — KPL (Thư Viện Producer Kinesis)

```java
// KPL tự động gom nhiều records nhỏ thành 1 PutRecords call
// → Tăng throughput, giảm số API calls
KinesisProducer producer = new KinesisProducer();

ByteBuffer data = ByteBuffer.wrap(eventJson.getBytes("UTF-8"));
ListenableFuture<UserRecordResult> future = producer.addUserRecord(
    "user-events-stream",
    partitionKey,
    data
);

// KPL features:
// - Aggregation: gom nhiều records nhỏ vào 1 Kinesis record
// - Batching: dùng PutRecords API tự động
// - Retry: tự động retry khi thất bại
// - Metrics: tích hợp CloudWatch
```

### Lỗi Phổ Biến Ở Producer

| Lỗi                                      | Nguyên Nhân                          | Cách Xử Lý                                   |
| ---------------------------------------- | ------------------------------------ | -------------------------------------------- |
| `ProvisionedThroughputExceededException` | Vượt 1MB/s hoặc 1000 records/s/shard | Retry với exponential backoff; thêm shard    |
| `ResourceNotFoundException`              | Stream không tồn tại                 | Kiểm tra tên stream, region                 |
| Record quá lớn                           | Record > 1MB                         | Nén dữ liệu hoặc chia nhỏ record            |

---

## 📥 Consumer — Đọc Dữ Liệu Từ Stream

### Hai Loại Consumer

#### 1. Standard Consumer (Consumer Tiêu Chuẩn)

```
Mỗi shard: 2 MB/s được chia sẻ giữa TẤT CẢ consumers
GetRecords API: tối đa 5 lần gọi/giây/shard
```

**Cách dùng đơn giản với AWS SDK:**

```python
def read_from_shard(stream_name: str, shard_id: str):
    # Lấy ShardIterator — con trỏ vị trí đọc
    response = kinesis.get_shard_iterator(
        StreamName=stream_name,
        ShardId=shard_id,
        ShardIteratorType='TRIM_HORIZON'  # Đọc từ đầu stream
        # LATEST: chỉ đọc records mới
        # AT_SEQUENCE_NUMBER: đọc từ sequence number cụ thể
        # AFTER_SEQUENCE_NUMBER: đọc sau sequence number
        # AT_TIMESTAMP: đọc từ timestamp
    )
    
    shard_iterator = response['ShardIterator']
    
    while True:
        records_response = kinesis.get_records(
            ShardIterator=shard_iterator,
            Limit=100  # Tối đa 10,000 records mỗi lần
        )
        
        records = records_response['Records']
        for record in records:
            data = json.loads(record['Data'])
            process_event(data)
        
        # Cập nhật iterator cho lần đọc tiếp
        shard_iterator = records_response['NextShardIterator']
        
        if not records:
            time.sleep(1)  # Tránh polling quá nhanh
```

#### 2. Enhanced Fan-out Consumer — EFO (Consumer Fan-out Nâng Cao)

```
Mỗi consumer: 2 MB/s RIÊNG BIỆT (không chia sẻ)
Sử dụng HTTP/2 push (server đẩy dữ liệu đến consumer — giảm độ trễ)
```

```python
def create_enhanced_fanout_consumer():
    # Đăng ký consumer
    response = kinesis.register_stream_consumer(
        StreamARN='arn:aws:kinesis:ap-southeast-1:123456789:stream/my-stream',
        ConsumerName='my-analytics-consumer'
    )
    consumer_arn = response['Consumer']['ConsumerARN']
    
    # Subscribe và nhận records qua HTTP/2
    response = kinesis.subscribe_to_shard(
        ConsumerARN=consumer_arn,
        ShardId='shardId-000000000000',
        StartingPosition={
            'Type': 'LATEST'
        }
    )
    
    # Xử lý events được push xuống
    for event in response['EventStream']:
        if 'SubscribeToShardEvent' in event:
            records = event['SubscribeToShardEvent']['Records']
            for record in records:
                process_event(json.loads(record['Data']))
```

#### 3. Kinesis Consumer Library — KCL (Thư Viện Consumer Kinesis)

KCL (Kinesis Consumer Library) tự động xử lý nhiều vấn đề phức tạp:

- **Shard assignment:** Phân chia shards giữa các worker instances
- **Checkpointing:** Lưu tiến độ vào DynamoDB để không xử lý lại khi restart
- **Shard splitting/merging:** Tự động phát hiện thay đổi shard
- **Error handling:** Retry và failover tự động

```python
# Ví dụ KCL với Python (amazon-kclpy)
from amazon_kclpy import kcl
from amazon_kclpy.v3 import processor

class MyRecordProcessor(processor.RecordProcessorBase):
    
    def initialize(self, initialize_input):
        self._shard_id = initialize_input.shard_id
        self._checkpoint_error_count = 0
    
    def process_records(self, process_records_input):
        for record in process_records_input.records:
            data = json.loads(record.binary_data)
            self.process_event(data)
        
        # Checkpoint sau mỗi batch để tránh xử lý lại
        process_records_input.checkpointer.checkpoint()
    
    def lease_lost(self, lease_lost_input):
        # Shard được tái phân công cho worker khác
        pass
    
    def shard_ended(self, shard_ended_input):
        # Shard đã kết thúc (sau split/merge)
        shard_ended_input.checkpointer.checkpoint()
    
    def shutdown_requested(self, shutdown_requested_input):
        shutdown_requested_input.checkpointer.checkpoint()
    
    def process_event(self, data):
        # Business logic của bạn ở đây
        print(f"Processing event: {data}")
```

#### 4. AWS Lambda Consumer (Lambda Là Consumer)

```python
# Lambda function tự động được trigger khi có records mới
# Cấu hình: Event source mapping từ KDS đến Lambda

def lambda_handler(event, context):
    for record in event['Records']:
        # Record data được base64 encode
        payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')
        data = json.loads(payload)
        
        shard_id = record['eventID'].split(':')[0]
        sequence_number = record['kinesis']['sequenceNumber']
        
        process_event(data)
    
    return {'statusCode': 200}

# Event source mapping config (via AWS Console hoặc CDK):
# - Batch size: 100 records (tối đa 10,000)
# - Starting position: LATEST hoặc TRIM_HORIZON
# - Bisect on error: True (tách batch đôi khi lỗi để tìm record lỗi)
# - Maximum retry attempts: 3
# - Destination on failure: SQS queue để lưu records thất bại
```

---

## 🔑 Partition Key & Ordering

### Tại Sao Thứ Tự Quan Trọng

Kinesis **đảm bảo thứ tự trong một shard**, nhưng **không đảm bảo thứ tự giữa các shard**:

```
Shard 1: [event_A_t1, event_A_t3, event_A_t5]  ← thứ tự đúng cho user A
Shard 2: [event_B_t2, event_B_t4]               ← thứ tự đúng cho user B

Nếu event_A đi vào 2 shard khác nhau → thứ tự không đảm bảo
```

### Chọn Partition Key Tốt

```python
# KHÔNG NÊN: Partition key cố định → tất cả records vào 1 shard (hot shard)
partition_key = "all_events"  # Tệ!

# KHÔNG NÊN: Partition key quá ít giá trị → phân phối không đều
partition_key = event_type  # Chỉ có 5 loại event → 5 shard nóng

# NÊN: Cardinality cao, phân phối đều
partition_key = user_id      # Hàng triệu users → phân phối tốt
partition_key = session_id   # Mỗi session khác nhau

# NÊN: Nếu cần thứ tự theo order_id
partition_key = order_id     # Tất cả event của order → cùng shard → đúng thứ tự

# NÊN: Nếu không cần thứ tự, tối đa throughput
import uuid
partition_key = str(uuid.uuid4())  # Random → phân phối hoàn toàn đều
```

### Vấn Đề Hot Shard (Mảnh Nóng) Và Cách Xử Lý

```
Hot Shard xảy ra khi:
- Partition key có phân phối lệch (skewed distribution)
- Một vài giá trị partition key chiếm phần lớn traffic

Ví dụ: Event source là "system" → tất cả records → 1 shard → throttling

Giải pháp 1: Thêm random suffix
partition_key = f"{original_key}_{random.randint(0, 99)}"
→ Phân phối đều hơn nhưng mất ordering (thứ tự)

Giải pháp 2: Tăng số shard
kinesis.update_shard_count(
    StreamName='my-stream',
    TargetShardCount=20,
    ScalingType='UNIFORM_SCALING'
)

Giải pháp 3: Dùng On-demand mode
→ KDS tự động điều chỉnh theo traffic thực tế
```

---

## ⏱️ Retention — Lưu Giữ Dữ Liệu

### Các Mức Retention

| Mức Retention               | Thời Gian   | Giá Phí Thêm                      | Khi Nào Dùng                        |
| --------------------------- | ----------- | ---------------------------------- | ----------------------------------- |
| Default Retention           | 24 giờ      | Không thêm                         | Production thông thường              |
| Extended Retention          | 7 ngày      | ~$0.02/shard-giờ                  | Cần replay, disaster recovery        |
| Long-term Retention         | 7-365 ngày  | ~$0.023/GB (dữ liệu lưu thêm)    | Audit, compliance, long debugging    |

```python
# Tăng retention lên 7 ngày
kinesis.increase_stream_retention_period(
    StreamName='my-stream',
    RetentionPeriodHours=168  # 7 ngày × 24 giờ
)

# Giảm retention
kinesis.decrease_stream_retention_period(
    StreamName='my-stream',
    RetentionPeriodHours=24  # Về lại 24 giờ
)
```

### Use Case Replay (Phát Lại)

```
Tình huống: Consumer bị lỗi trong 6 giờ, cần xử lý lại dữ liệu đã bỏ qua

1. Retention = 24h → vẫn còn dữ liệu 6 giờ trước
2. Đặt ShardIterator về AT_TIMESTAMP của 6 giờ trước
3. Consumer đọc lại từ đó

kinesis.get_shard_iterator(
    StreamName='my-stream',
    ShardId='shardId-000000000000',
    ShardIteratorType='AT_TIMESTAMP',
    Timestamp=datetime(2026, 5, 17, 10, 0, 0)  # 6 giờ trước
)
```

---

## 📈 Scaling — Tăng Giảm Quy Mô

### Resharding (Tái Phân Mảnh)

**Shard Splitting — Tách Shard:** Chia 1 shard thành 2 → tăng throughput

```python
# Tách shard để tăng capacity
kinesis.split_shard(
    StreamName='my-stream',
    ShardToSplit='shardId-000000000000',
    NewStartingHashKey='170141183460469231731687303715884105728'
    # Hash key ở giữa range của shard hiện tại
)
```

**Shard Merging — Gộp Shard:** Gộp 2 shard liền kề thành 1 → giảm chi phí

```python
# Gộp 2 shard để giảm chi phí khi traffic thấp
kinesis.merge_shards(
    StreamName='my-stream',
    ShardToMerge='shardId-000000000000',
    AdjacentShardToMerge='shardId-000000000001'
)
```

**UpdateShardCount — Cập Nhật Số Shard:** Cách đơn giản nhất

```python
# Scale lên 10 shards (auto-splits hoặc merges)
kinesis.update_shard_count(
    StreamName='my-stream',
    TargetShardCount=10,
    ScalingType='UNIFORM_SCALING'
)
```

**Lưu ý:** Trong quá trình resharding, các shard cũ chuyển sang trạng thái CLOSED nhưng vẫn readable cho đến hết retention period.

### Auto Scaling với CloudWatch

```python
# CloudWatch Alarm khi gần hết throughput
cloudwatch.put_metric_alarm(
    AlarmName='KinesisWriteThrottling',
    MetricName='WriteProvisionedThroughputExceeded',
    Namespace='AWS/Kinesis',
    Dimensions=[{'Name': 'StreamName', 'Value': 'my-stream'}],
    Period=60,
    EvaluationPeriods=5,
    Threshold=0,  # Ngay khi có throttling
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=['arn:aws:sns:...']  # Trigger SNS → Lambda để tăng shard
)
```

---

## 🔐 Bảo Mật & Mã Hóa

### Encryption at Rest (Mã Hóa Khi Lưu Trữ)

```python
# Bật server-side encryption khi tạo stream
kinesis.create_stream(
    StreamName='encrypted-stream',
    ShardCount=2
)

# Sau khi tạo: bật SSE (Server-Side Encryption — Mã Hóa Phía Server)
kinesis.start_stream_encryption(
    StreamName='encrypted-stream',
    EncryptionType='KMS',
    KeyId='arn:aws:kms:ap-southeast-1:123456789:key/your-key-id'
    # Hoặc dùng alias: 'alias/aws/kinesis' (AWS managed key)
)
```

### Encryption in Transit (Mã Hóa Khi Truyền)

- Tất cả API calls đến Kinesis đều sử dụng **HTTPS/TLS** mặc định
- Không cần cấu hình thêm

### IAM Permissions (Quyền IAM)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProducerPermissions",
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:DescribeStream"
      ],
      "Resource": "arn:aws:kinesis:ap-southeast-1:123456789:stream/my-stream"
    },
    {
      "Sid": "ConsumerPermissions",
      "Effect": "Allow",
      "Action": [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListShards",
        "kinesis:SubscribeToShard"
      ],
      "Resource": "arn:aws:kinesis:ap-southeast-1:123456789:stream/my-stream"
    }
  ]
}
```

---

## 🔗 Tích Hợp Với AWS Services

### Kinesis → Lambda (Event-driven Processing)

```
KDS ─────[Event source mapping]────► Lambda
          Batch size: 1-10,000
          Starting position: LATEST/TRIM_HORIZON
          Bisect on error: True (chia batch đôi khi lỗi)
```

**Cấu hình Lambda Event Source Mapping (via CDK):**

```python
from aws_cdk import aws_lambda as lambda_
from aws_cdk import aws_kinesis as kinesis
from aws_cdk import aws_lambda_event_sources as sources

stream = kinesis.Stream(self, "MyStream", shard_count=3)

fn = lambda_.Function(
    self, "ProcessorFn",
    runtime=lambda_.Runtime.PYTHON_3_12,
    handler="handler.lambda_handler",
    code=lambda_.Code.from_asset("lambda")
)

fn.add_event_source(
    sources.KinesisEventSource(
        stream,
        starting_position=lambda_.StartingPosition.TRIM_HORIZON,
        batch_size=100,
        bisect_batch_on_error=True,
        retry_attempts=3,
        on_failure=sources.SqsDlq(dead_letter_queue)  # DLQ cho records lỗi
    )
)
```

### Kinesis → Kinesis Analytics (Flink)

```python
# KDA Flink application đọc từ KDS
# (Cấu hình qua AWS Console hoặc CDK)
{
    "ApplicationName": "fraud-detection",
    "Inputs": [
        {
            "NamePrefix": "SOURCE_SQL_STREAM",
            "KinesisStreamsInput": {
                "ResourceARN": "arn:aws:kinesis:...:stream/transactions",
                "RoleARN": "arn:aws:iam::...:role/KDARole"
            },
            "InputSchema": {
                "RecordFormat": {"RecordFormatType": "JSON"},
                "RecordColumns": [
                    {"Name": "user_id", "SqlType": "VARCHAR(64)"},
                    {"Name": "amount", "SqlType": "DOUBLE"},
                    {"Name": "timestamp", "SqlType": "BIGINT"}
                ]
            }
        }
    ]
}
```

### Kinesis → S3 (Lưu Trữ Dài Hạn)

Thay vì consumer tự ghi vào S3, nên dùng **Kinesis Firehose** làm trung gian:

```
KDS → Firehose → S3
      (Buffer 60s / 5MB, định dạng Parquet, partition theo ngày)
```

---

## 📊 Giám Sát & Troubleshooting

### Metrics Quan Trọng Trên CloudWatch

| Metric                                  | Ý Nghĩa                                    | Ngưỡng Cảnh Báo          |
| --------------------------------------- | ------------------------------------------- | ------------------------- |
| `GetRecords.IteratorAgeMilliseconds`    | Độ trễ consumer (ms) — cao = consumer chậm | > 60,000 ms (1 phút)     |
| `WriteProvisionedThroughputExceeded`    | Số records bị throttle ở write             | > 0 (ngay khi có throttle)|
| `ReadProvisionedThroughputExceeded`     | Số records bị throttle ở read              | > 0                       |
| `PutRecord.Success`                     | Tỉ lệ ghi thành công                       | < 99%                     |
| `GetRecords.Success`                    | Tỉ lệ đọc thành công                       | < 99%                     |
| `IncomingRecords`                       | Số records đến mỗi shard                   | Gần 1000/s → cần thêm shard |
| `IncomingBytes`                         | Bytes đến mỗi shard                        | Gần 1MB/s → cần thêm shard  |

### Sự Cố Phổ Biến Và Cách Xử Lý

#### Vấn Đề 1: Consumer Bị Lag (Chậm Trễ)

```
Triệu chứng: IteratorAgeMilliseconds tăng liên tục
Nguyên nhân:
  - Consumer xử lý quá chậm so với tốc độ ghi
  - Số consumer instances không đủ

Giải pháp:
1. Tăng số shard → nhiều Lambda instances chạy song song
2. Tăng batch size của Lambda (giảm overhead per call)
3. Tối ưu logic xử lý trong consumer
4. Dùng Enhanced Fan-out nếu có nhiều consumers
```

#### Vấn Đề 2: ProvisionedThroughputExceededException

```
Triệu chứng: Producer nhận lỗi throttling
Nguyên nhân:
  - Write vượt 1MB/s hoặc 1000 records/s trên 1 shard
  - Hot partition key

Giải pháp:
1. Retry với exponential backoff (1s → 2s → 4s → 8s)
2. Tăng số shard
3. Cải thiện partition key để phân phối đều hơn
4. Chuyển sang On-demand mode
```

#### Vấn Đề 3: Xử Lý Trùng Lặp (Duplicate Processing)

```
Nguyên nhân:
  - Consumer restart mà chưa checkpoint kịp
  - Lambda retry khi có lỗi

Giải pháp (At-least-once → Exactly-once):
1. Idempotent processing: dùng sequence number làm dedup key
2. Lưu processed sequence numbers vào DynamoDB
3. Kiểm tra trước khi xử lý:

def process_record(record):
    seq_num = record['kinesis']['sequenceNumber']
    
    # Kiểm tra đã xử lý chưa
    if is_already_processed(seq_num):
        return  # Skip duplicate
    
    # Xử lý
    do_processing(record)
    
    # Đánh dấu đã xử lý
    mark_as_processed(seq_num)
```

---

## 💰 Chi Phí

### Mô Hình Tính Phí (Region ap-southeast-1 — Singapore)

| Thành Phần                          | Giá                         |
| ----------------------------------- | --------------------------- |
| **Shard giờ** (Provisioned)         | ~$0.015/shard/giờ           |
| **PUT Payload Units**               | ~$0.014/1 triệu units       |
| **Extended Data Retention**         | ~$0.02/shard/giờ (7 ngày)  |
| **Long-term Retention**             | ~$0.023/GB lưu thêm         |
| **Enhanced Fan-out Consumer**       | ~$0.015/shard/giờ + $0.013/GB |
| **On-demand mode**                  | ~$0.04/1M records + $0.04/GB |

### Tính Chi Phí Mẫu

```
Tình huống: 5 shards, 30 ngày, 100GB dữ liệu/ngày, 24h retention

Chi phí shards: 5 × $0.015 × 24 × 30 = $54/tháng
Chi phí PUT: 100GB × 1000 MB/GB × 1024 KB/MB / 25KB × $0.014/1M
           = khoảng $0.06/GB → $6/ngày → $180/tháng
Tổng: ~$234/tháng

So sánh: Nếu dùng Extended Retention (7 ngày):
Chi phí thêm: 5 × $0.02 × 24 × 30 = $72/tháng
```

### Tối Ưu Chi Phí

1. **Dùng PutRecords thay vì PutRecord:** Giảm số API calls → giảm chi phí PUT
2. **KPL Aggregation:** Gom nhiều records nhỏ thành 1 → tiết kiệm PUT units
3. **Giảm số shard khi traffic thấp:** Merge shards ban đêm/cuối tuần
4. **On-demand chỉ khi traffic không đều:** Nếu traffic ổn định, Provisioned rẻ hơn
5. **Không tăng retention quá cần thiết:** Extended retention tính phí thêm

---

## 🎯 Câu Hỏi Phỏng Vấn

### Q1: Giải thích Shard trong Kinesis — cách tính số shard cần thiết?

**A:** Shard là đơn vị throughput cơ bản: 1 MB/s write, 2 MB/s read, 1000 records/s. Công thức tính:

```
Số shard = max(
    ceil(write_MB_per_s / 1.0),
    ceil(total_read_MB_per_s / 2.0)  # standard consumer
)
```

Nếu có 3 MB/s write và 6 MB/s read → cần max(3, 3) = 3 shards.

### Q2: Standard Consumer vs Enhanced Fan-out — khi nào dùng cái nào?

**A:**
- **Standard Consumer:** Nhiều consumers chia sẻ 2MB/s/shard. Phù hợp khi có ít consumers hoặc không cần low-latency. Rẻ hơn.
- **Enhanced Fan-out:** Mỗi consumer nhận riêng 2MB/s/shard qua HTTP/2 push. Độ trễ thấp hơn (~70ms vs ~200ms). Phù hợp khi có nhiều consumers cần throughput cao. Tốn phí thêm.

### Q3: Làm sao đảm bảo thứ tự xử lý events với Kinesis?

**A:** Sử dụng cùng **Partition Key** cho tất cả events liên quan (ví dụ: cùng `user_id` hoặc `order_id`). Tất cả records với cùng partition key sẽ vào cùng một shard, và Kinesis đảm bảo thứ tự trong một shard. Lưu ý: thứ tự chỉ đảm bảo trong 1 shard, không đảm bảo cross-shard.

### Q4: Exactly-once processing với Kinesis — có làm được không?

**A:** KDS đảm bảo **at-least-once delivery** (Giao Hàng Ít Nhất Một Lần) — có thể có duplicate. Để đạt exactly-once processing (Xử Lý Chính Xác Một Lần), consumer cần implement idempotency (Xử Lý Trùng Lặp An Toàn):
1. Dùng sequence number làm dedup key (Khóa Loại Bỏ Trùng Lặp)
2. Lưu danh sách đã xử lý vào DynamoDB
3. Kiểm tra trước khi xử lý mỗi record

### Q5: Điều gì xảy ra khi shard bị CLOSED sau khi merge/split?

**A:** Shard CLOSED vẫn readable cho đến hết retention period. Consumer cần đọc hết records từ shard cũ trước khi chuyển sang shard mới. KCL xử lý tự động; nếu dùng SDK thủ công, cần kiểm tra `ChildShards` trong response và chuyển iterator sang shards con.

---

## 📊 Tóm Tắt Nhanh

```
KDS — Kinesis Data Streams
├── Shard: 1MB/s write, 2MB/s read, 1000 records/s
├── Retention: 24h → 365 ngày
├── Consumer: Standard (shared) vs Enhanced Fan-out (riêng)
├── Ordering: Đảm bảo trong shard, không đảm bảo cross-shard
├── Partition Key: Quyết định shard, chọn high-cardinality
└── Scale: Split/Merge shards, UpdateShardCount, On-demand mode

Lỗi phổ biến:
├── ProvisionedThroughputExceeded → thêm shard, retry
├── Hot shard → partition key tốt hơn
├── Consumer lag → tăng shard, tối ưu consumer
└── Duplicate records → idempotent processing
```

---

## 🔗 Điều Hướng

| Tiếp Theo                                          | Quay Lại                              |
| -------------------------------------------------- | ------------------------------------- |
| [2-kinesis-firehose.md](./2-kinesis-firehose.md)   | [README.md](./README.md) — Tổng quan |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** 1-kinesis-data-streams.md
**Trạng Thái:** ✅ Hoàn thành
