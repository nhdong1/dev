# Kinesis Data Streams (KDS) — Luồng Dữ Liệu Kinesis

> **Kinesis Data Streams — KDS** là dịch vụ thu thập và xử lý data streaming (luồng dữ liệu) thời gian thực, cho phép nhiều consumer (người tiêu dùng) đọc cùng dữ liệu song song và có khả năng replay (phát lại) dữ liệu trong retention period (khoảng thời gian lưu giữ).

---

## 📚 Mục Lục

1. [Kiến Trúc Cốt Lõi](#kiến-trúc-cốt-lõi)
2. [Shard — Đơn Vị Throughput](#shard--đơn-vị-throughput)
3. [Producer — Nhà Sản Xuất Dữ Liệu](#producer--nhà-sản-xuất-dữ-liệu)
4. [Consumer — Người Tiêu Dùng Dữ Liệu](#consumer--người-tiêu-dùng-dữ-liệu)
5. [Partition Key và Ordering](#partition-key-và-ordering)
6. [Retention và Replay](#retention-và-replay)
7. [Capacity Mode — Chế Độ Năng Lực](#capacity-mode--chế-độ-năng-lực)
8. [Mã Lỗi Thường Gặp](#mã-lỗi-thường-gặp)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc Cốt Lõi

### Luồng Dữ Liệu Tổng Quan

```
Producer (Nhà Sản Xuất)              Stream                Consumer (Người Tiêu Dùng)
───────────────────                  ──────                ──────────────────────────
Web server                           Shard 1               Lambda Function A
Mobile app       PutRecord ────────▶ [rec1]  ◀──────────── KCL Application B
IoT device       PutRecords          [rec2]                Kinesis Data Analytics
Database CDC                         [rec3]                Kinesis Data Firehose
                                     
                                     Shard 2               Lambda Function A (same)
                         ──────────▶ [rec4]  ◀──────────── KCL Application B (same)
                                     [rec5]
                                     [rec6]
```

### Vòng Đời Của Một Record (Bản Ghi)

```
1. Producer gọi PutRecord(partitionKey, data)
2. Kinesis hash(partitionKey) → xác định shard
3. Record được ghi vào shard với sequence number
4. Record lưu trong retention period (24h–365 ngày)
5. Consumer đọc record từ shard theo sequence number
6. Sau retention period, record tự động bị xóa
```

---

## Shard — Đơn Vị Throughput

### Định Nghĩa

**Shard (Mảnh)** là đơn vị năng lực cơ bản của Kinesis Data Streams. Mỗi shard cung cấp:

| Hướng | Giới Hạn |
|---|---|
| **Write (Ghi)** | 1 MB/giây HOẶC 1.000 records/giây (đạt giới hạn nào trước thì bị throttle) |
| **Read (Đọc)** | 2 MB/giây (chia sẻ giữa tất cả consumers dùng standard) |
| **Read Enhanced** | 2 MB/giây PER CONSUMER per shard (Enhanced Fan-Out) |

### Shard Giới Hạn Chi Tiết

```
Một Shard:
┌─────────────────────────────────────┐
│  Write capacity:                    │
│    1.000 records/s                  │
│    OR 1 MB/s                        │
│    (whichever hits first)           │
│                                     │
│  Read capacity (Standard):          │
│    5 GetRecords calls/s             │
│    2 MB/s total across consumers    │
│                                     │
│  Read capacity (Enhanced Fan-Out):  │
│    2 MB/s per registered consumer   │
└─────────────────────────────────────┘
```

### Cấu Trúc Record Trong Shard

```
Shard
├── Record 1
│   ├── PartitionKey: "user-123"
│   ├── SequenceNumber: "49590338271490256608559692540961969032588658621268971522"
│   ├── Data: <base64-encoded-payload>
│   └── ApproximateArrivalTimestamp: 2026-05-18T10:00:00Z
├── Record 2
│   ├── PartitionKey: "user-456"
│   ├── SequenceNumber: "49590338271490256608559692540961969032588658621268971523"
│   └── ...
└── ...
```

---

## Producer — Nhà Sản Xuất Dữ Liệu

### Các Cách Ghi Dữ Liệu Vào KDS

#### 1. AWS SDK — PutRecord (Ghi Một Record)

```python
import boto3
import json

kinesis_client = boto3.client('kinesis', region_name='ap-southeast-1')

def put_record(stream_name: str, partition_key: str, data: dict):
    response = kinesis_client.put_record(
        StreamName=stream_name,
        Data=json.dumps(data).encode('utf-8'),
        PartitionKey=partition_key
    )
    
    print(f"ShardId: {response['ShardId']}")
    print(f"SequenceNumber: {response['SequenceNumber']}")
    return response

# Gọi hàm
put_record(
    stream_name='order-events',
    partition_key='order-12345',       # Cùng order → cùng shard → đảm bảo thứ tự
    data={'orderId': '12345', 'status': 'PLACED', 'amount': 99.99}
)
```

#### 2. AWS SDK — PutRecords (Ghi Nhiều Record, Tối Đa 500 Records / Lần)

```python
def put_records_batch(stream_name: str, records: list):
    """
    Ghi nhiều records một lần — hiệu quả hơn, giảm số API call.
    Lưu ý: PutRecords không đảm bảo tất cả thành công cùng lúc.
    """
    formatted_records = [
        {
            'Data': json.dumps(record['data']).encode('utf-8'),
            'PartitionKey': record['partition_key']
        }
        for record in records
    ]
    
    response = kinesis_client.put_records(
        StreamName=stream_name,
        Records=formatted_records
    )
    
    failed_count = response['FailedRecordCount']
    if failed_count > 0:
        # Xử lý retry cho các records thất bại
        failed_records = [
            records[i] for i, r in enumerate(response['Records'])
            if 'ErrorCode' in r
        ]
        print(f"Cần retry {failed_count} records")
        return failed_records
    
    return []
```

#### 3. Kinesis Producer Library (KPL) — Thư Viện Sản Xuất Kinesis

```java
// KPL tự động batching, retry, và compression
KinesisProducer kinesis = new KinesisProducer(config);

ListenableFuture<UserRecordResult> f = kinesis.addUserRecord(
    streamName,
    partitionKey,
    ByteBuffer.wrap(data.getBytes())
);

// KPL gộp nhiều record nhỏ thành PutRecords call (aggregation)
// Tăng throughput đáng kể cho record nhỏ
```

### ProvisionedThroughputExceededException — Lỗi Vượt Throughput

```python
import time
from botocore.exceptions import ClientError

def put_record_with_retry(stream_name, partition_key, data, max_retries=5):
    """Retry với exponential backoff khi gặp throttling."""
    for attempt in range(max_retries):
        try:
            return kinesis_client.put_record(
                StreamName=stream_name,
                Data=json.dumps(data).encode('utf-8'),
                PartitionKey=partition_key
            )
        except ClientError as e:
            if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
                wait_time = (2 ** attempt) * 0.1  # 0.1s, 0.2s, 0.4s, 0.8s, 1.6s
                print(f"Throttled, retry sau {wait_time}s (lần {attempt + 1})")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Đã hết số lần retry")
```

---

## Consumer — Người Tiêu Dùng Dữ Liệu

### Hai Loại Consumer

#### Standard Consumer (Consumer Tiêu Chuẩn) — Pull Model

```
Đặc điểm:
- GetRecords API call (polling — kéo dữ liệu)
- 2 MB/s chia sẻ giữa TẤT CẢ consumers trên cùng shard
- Tối đa 5 GetRecords calls/giây per shard
- Độ trễ: 200ms–1 giây
- Chi phí: chỉ tính PUT payload và storage
```

#### Enhanced Fan-Out Consumer (Consumer Khuếch Tán Nâng Cao) — Push Model

```
Đặc điểm:
- SubscribeToShard API (streaming push — đẩy dữ liệu)
- 2 MB/s RIÊNG cho từng consumer (không chia sẻ)
- Độ trễ: ~70ms
- Tối đa 20 registered consumers per stream
- Chi phí: tính thêm phí Enhanced Fan-Out
```

### Consumer Với AWS Lambda

```python
# Lambda tự động được trigger bởi Kinesis (Event Source Mapping)
# Lambda nhận một batch records từ shard

def lambda_handler(event, context):
    """
    event['Records']: danh sách records từ shard
    Mỗi record đã được decode từ base64 tự động.
    """
    for record in event['Records']:
        # Decode dữ liệu từ base64
        import base64
        payload = base64.b64decode(record['kinesis']['data']).decode('utf-8')
        data = json.loads(payload)
        
        # Metadata quan trọng
        shard_id = record['eventID'].split(':')[0]
        sequence_number = record['kinesis']['sequenceNumber']
        partition_key = record['kinesis']['partitionKey']
        arrival_time = record['kinesis']['approximateArrivalTimestamp']
        
        print(f"Shard: {shard_id}, Seq: {sequence_number}")
        process_record(data)
    
    # Lambda xử lý thành công → tự động advance checkpoint
    return {'batchItemFailures': []}
```

### Lambda Bisect-on-Error (Chia Đôi Khi Lỗi)

```python
def lambda_handler(event, context):
    """
    BisectBatchOnFunctionError: true → khi Lambda lỗi, Kinesis
    tự động chia batch làm đôi và retry từng nửa.
    Giúp cô lập record gây lỗi mà không block toàn bộ shard.
    """
    failed_items = []
    
    for record in event['Records']:
        try:
            payload = base64.b64decode(record['kinesis']['data'])
            process_record(json.loads(payload))
        except Exception as e:
            # Báo cáo failure cho từng record — partial batch response
            failed_items.append({
                'itemIdentifier': record['kinesis']['sequenceNumber']
            })
    
    return {'batchItemFailures': failed_items}
```

### Kinesis Client Library (KCL) — Thư Viện Client Kinesis

```java
// KCL xử lý shard assignment, checkpointing, failover tự động
public class OrderEventProcessor implements ShardRecordProcessor {
    
    @Override
    public void processRecords(ProcessRecordsInput processRecordsInput) {
        for (KinesisClientRecord record : processRecordsInput.records()) {
            // Decode và xử lý record
            String data = StandardCharsets.UTF_8.decode(record.data()).toString();
            processOrderEvent(data);
        }
        
        // Checkpoint — lưu vị trí đã xử lý vào DynamoDB
        // KCL tự động tạo DynamoDB table để track checkpoint
        try {
            processRecordsInput.checkpointer().checkpoint();
        } catch (ShutdownException | ThrottlingException | KinesisClientLibDependencyException e) {
            // Xử lý checkpoint failure
        }
    }
}
```

---

## Partition Key và Ordering

### Cơ Chế Phân Vùng

```
partition_key → MD5 hash → số 128-bit → ánh xạ vào shard

Ví dụ với 3 shards:
Hash range [0, 42949672960)           → Shard 1
Hash range [42949672960, 85899345920) → Shard 2
Hash range [85899345920, ...)         → Shard 3

"user-001" → hash 15... → Shard 1 (luôn vào Shard 1)
"user-002" → hash 55... → Shard 2 (luôn vào Shard 2)
"user-003" → hash 15... → Shard 1 (cùng shard với user-001)
```

### Đảm Bảo Thứ Tự Trong Shard

```
✅ Đảm bảo thứ tự: các records có cùng partition key trong một shard
❌ Không đảm bảo thứ tự: records ở các shards khác nhau

Ví dụ đặt hàng:
PUT(partitionKey="order-123", data="PLACED")
PUT(partitionKey="order-123", data="PAYMENT_CONFIRMED")
PUT(partitionKey="order-123", data="SHIPPED")

→ Ba events này luôn vào cùng shard → consumer đọc đúng thứ tự
```

### Hot Shard Problem (Vấn Đề Shard Nóng)

```
Vấn đề: Partition key không đồng đều → một shard nhận quá nhiều traffic

Ví dụ xấu:
partitionKey = "static-key"  → TẤT CẢ records vào một shard
partitionKey = user_country  → "VN" nhận 80% traffic → hot shard

Giải pháp 1: Random Suffix (Hậu Tố Ngẫu Nhiên)
import random
partition_key = f"user-{user_id}-{random.randint(0, 9)}"
# Lưu ý: mất strict ordering khi dùng suffix ngẫu nhiên

Giải pháp 2: Composite Key (Khóa Tổng Hợp)
partition_key = f"{region}-{user_id}"
# Phân tán đều hơn theo region

Giải pháp 3: On-Demand Mode
# Kinesis tự động scale shard khi traffic tăng
```

---

## Retention và Replay

### Cấu Hình Retention

```python
# Mặc định: 24 giờ
# Mở rộng lên 7 ngày (không tính phí thêm ở một số tier)
# Mở rộng lên 365 ngày (tính phí thêm)

kinesis_client.increase_stream_retention_period(
    StreamName='order-events',
    RetentionPeriodHours=168  # 7 ngày
)

# Xem retention hiện tại
response = kinesis_client.describe_stream_summary(StreamName='order-events')
retention = response['StreamDescriptionSummary']['RetentionPeriodHours']
```

### Replay Use Case (Trường Hợp Phát Lại Dữ Liệu)

```
Khi nào cần replay:
1. Consumer gặp bug → fix bug → replay lại toàn bộ events để tính toán lại
2. Thêm consumer mới → cần xử lý lại lịch sử
3. Reprocessing sau khi thay đổi business logic

Cách replay:
- Đặt ShardIterator về TRIM_HORIZON (đầu retention window)
- Hoặc AT_TIMESTAMP (từ thời điểm cụ thể)
```

```python
def get_shard_iterator_from_timestamp(stream_name, shard_id, timestamp):
    """Lấy iterator từ thời điểm cụ thể để replay."""
    response = kinesis_client.get_shard_iterator(
        StreamName=stream_name,
        ShardId=shard_id,
        ShardIteratorType='AT_TIMESTAMP',
        Timestamp=timestamp
    )
    return response['ShardIterator']

# Đọc records từ thời điểm đó
shard_iterator = get_shard_iterator_from_timestamp(
    'order-events',
    'shardId-000000000000',
    '2026-05-01T00:00:00Z'
)

while shard_iterator:
    response = kinesis_client.get_records(
        ShardIterator=shard_iterator,
        Limit=100
    )
    records = response['Records']
    process_records(records)
    shard_iterator = response.get('NextShardIterator')
```

### ShardIterator Types (Các Loại Trình Lặp Shard)

| Loại | Ý Nghĩa | Dùng Khi |
|---|---|---|
| `TRIM_HORIZON` | Từ record cũ nhất trong retention | Replay toàn bộ |
| `LATEST` | Chỉ records mới sau thời điểm get | Consumer mới, chỉ cần dữ liệu mới |
| `AT_SEQUENCE_NUMBER` | Từ sequence number cụ thể | Tiếp tục từ checkpoint |
| `AFTER_SEQUENCE_NUMBER` | Sau sequence number cụ thể | Tiếp tục sau checkpoint |
| `AT_TIMESTAMP` | Từ timestamp cụ thể | Replay từ thời điểm nhất định |

---

## Capacity Mode — Chế Độ Năng Lực

### Provisioned Mode (Chế Độ Dự Phòng)

```
- Bạn chỉ định số shard cố định
- Trả tiền theo shard-hour
- Cần tự scale (split/merge shard)
- Phù hợp: traffic có thể dự đoán, cần kiểm soát chi phí
```

### On-Demand Mode (Chế Độ Theo Nhu Cầu)

```
- Kinesis tự động scale số shard
- Trả tiền theo GB throughput thực tế
- Không cần quản lý shard thủ công
- Phù hợp: traffic biến động, workload mới chưa biết pattern

Lưu ý: On-Demand có thể đắt hơn Provisioned nếu traffic cao ổn định
```

```python
# Tạo stream với On-Demand mode
kinesis_client.create_stream(
    StreamName='order-events-v2',
    StreamModeDetails={
        'StreamMode': 'ON_DEMAND'  # hoặc 'PROVISIONED'
    }
)

# Chuyển từ Provisioned sang On-Demand
kinesis_client.update_stream_mode(
    StreamARN='arn:aws:kinesis:...',
    StreamModeDetails={'StreamMode': 'ON_DEMAND'}
)
```

---

## Mã Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Giải Pháp |
|---|---|---|
| `ProvisionedThroughputExceededException` | Vượt giới hạn write/read throughput | Retry với exponential backoff, tăng shard |
| `ExpiredIteratorException` | ShardIterator hết hạn (5 phút không dùng) | Lấy ShardIterator mới |
| `ResourceNotFoundException` | Stream không tồn tại | Kiểm tra tên stream |
| `LimitExceededException` | Vượt giới hạn API call | Giảm tần suất GetRecords call |
| `InvalidArgumentException` | Data quá 1MB hoặc partition key rỗng | Kiểm tra kích thước và format |

---

## Ví Dụ Thực Tế

### Hệ Thống Thu Thập Order Events

```python
# producer.py — thu thập order events từ API
import boto3, json, uuid
from datetime import datetime

kinesis = boto3.client('kinesis', region_name='ap-southeast-1')
STREAM_NAME = 'order-events'

def publish_order_event(order_id: str, event_type: str, payload: dict):
    event = {
        'eventId': str(uuid.uuid4()),
        'eventType': event_type,
        'orderId': order_id,
        'timestamp': datetime.utcnow().isoformat(),
        'payload': payload
    }
    
    # Dùng order_id làm partition key
    # → tất cả events của cùng order vào cùng shard
    # → đảm bảo thứ tự xử lý
    response = kinesis.put_record(
        StreamName=STREAM_NAME,
        Data=json.dumps(event).encode('utf-8'),
        PartitionKey=order_id
    )
    
    return response['SequenceNumber']

# Sử dụng
publish_order_event('order-789', 'ORDER_PLACED', {'items': [...], 'total': 299.99})
publish_order_event('order-789', 'PAYMENT_RECEIVED', {'method': 'CARD', 'amount': 299.99})
publish_order_event('order-789', 'ORDER_SHIPPED', {'tracking': 'VNP123456'})
```

```python
# consumer_lambda.py — Lambda xử lý order events
import json, base64

def lambda_handler(event, context):
    failed_records = []
    
    for record in event['Records']:
        try:
            # Decode dữ liệu
            raw_data = base64.b64decode(record['kinesis']['data'])
            order_event = json.loads(raw_data.decode('utf-8'))
            
            # Xử lý theo loại event
            handler = EVENT_HANDLERS.get(order_event['eventType'])
            if handler:
                handler(order_event)
            else:
                print(f"Unknown event type: {order_event['eventType']}")
                
        except Exception as e:
            print(f"Lỗi xử lý record: {e}")
            # Báo lỗi để Lambda bisect batch
            failed_records.append({
                'itemIdentifier': record['kinesis']['sequenceNumber']
            })
    
    return {'batchItemFailures': failed_records}

EVENT_HANDLERS = {
    'ORDER_PLACED':      handle_order_placed,
    'PAYMENT_RECEIVED':  handle_payment,
    'ORDER_SHIPPED':     handle_shipping,
}
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Kinesis Data Streams đảm bảo thứ tự xử lý như thế nào?

**Trả lời:**
- Thứ tự chỉ được đảm bảo **trong một shard**
- Records có cùng partition key luôn vào cùng shard → ordered
- Giữa các shards khác nhau không có ordering guarantee
- **Thiết kế:** Chọn partition key sao cho records cần ordered nằm cùng shard (ví dụ: `order_id`)

### Q2: Tại sao cần cẩn thận với partition key?

**Trả lời:**
- Partition key không đa dạng → **hot shard** — một shard nhận quá nhiều traffic, bị throttle
- Ví dụ: dùng `"default"` cho tất cả → toàn bộ traffic vào 1 shard, lãng phí các shard còn lại
- Giải pháp: partition key phân tán đều (user_id, order_id) hoặc thêm random suffix

### Q3: Sự khác biệt giữa Standard Consumer và Enhanced Fan-Out?

**Trả lời:**
- **Standard:** 2MB/s chia sẻ; polling; độ trễ ~200ms; không tính phí thêm
- **Enhanced Fan-Out:** 2MB/s riêng mỗi consumer; push (HTTP/2 streaming); độ trễ ~70ms; tính phí thêm
- Dùng Enhanced khi cần nhiều consumer đọc tốc độ cao mà không ảnh hưởng lẫn nhau

### Q4: Khi nào nên dùng On-Demand Mode thay vì Provisioned Mode?

**Trả lời:**
- **On-Demand:** traffic không thể dự đoán, workload mới, muốn zero-ops
- **Provisioned:** traffic dự đoán được, cần kiểm soát chi phí, cần tự quản lý shard split/merge
- On-Demand có thể đắt hơn 2–3x so với Provisioned ở high-sustained traffic

---

**Tiếp Theo:** [2-firehose.md](./2-firehose.md) — Kinesis Data Firehose, giao vận near-real-time vào S3, Redshift, OpenSearch.
