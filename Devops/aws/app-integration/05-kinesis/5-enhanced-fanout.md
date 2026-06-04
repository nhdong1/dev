# Enhanced Fan-Out — Khuếch Tán Nâng Cao

> **Enhanced Fan-Out (EFO) — Khuếch Tán Nâng Cao** là tính năng của Kinesis Data Streams cho phép mỗi consumer (người tiêu dùng) được cấp phát riêng **2 MB/giây bandwidth per shard**, thay vì chia sẻ chung như Standard Consumer. Đây là giải pháp khi nhiều consumers cần đọc cùng stream với tốc độ cao mà không ảnh hưởng lẫn nhau.

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Standard vs Enhanced Fan-Out — So Sánh](#standard-vs-enhanced-fan-out--so-sánh)
3. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
4. [Đăng Ký Enhanced Fan-Out Consumer](#đăng-ký-enhanced-fan-out-consumer)
5. [Enhanced Fan-Out Với Lambda](#enhanced-fan-out-với-lambda)
6. [Enhanced Fan-Out Với KCL](#enhanced-fan-out-với-kcl)
7. [Giới Hạn Và Chi Phí](#giới-hạn-và-chi-phí)
8. [Khi Nào Dùng Enhanced Fan-Out](#khi-nào-dùng-enhanced-fan-out)
9. [Kiến Trúc Fan-Out Thực Tế](#kiến-trúc-fan-out-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Vấn Đề Của Standard Consumer

```
Stream: 5 shards × 2 MB/s = 10 MB/s tổng bandwidth read

Standard Consumer — chia sẻ bandwidth:
                    ┌──────────────────────────────────────┐
Shard 1 (2 MB/s) ──▶│ Consumer A (analytics)              │
                    │ Consumer B (fraud detection)  ├─ chia  2 MB/s ÷ 3 = 0.67 MB/s
Shard 2 (2 MB/s) ──▶│ Consumer C (audit log)              │ per consumer
                    └──────────────────────────────────────┘
                    
Kết quả: mỗi consumer chỉ đọc được 0.67 MB/s từ một shard
→ Consumer bị lag khi traffic gần đến giới hạn
→ Thêm consumer → mỗi consumer càng chậm hơn
```

### Giải Pháp Enhanced Fan-Out

```
Enhanced Fan-Out — bandwidth riêng mỗi consumer:

                    ┌──────────────────────────────────────────┐
Shard 1 (2 MB/s) ──▶│ Consumer A (analytics)    2 MB/s riêng  │
                    │ Consumer B (fraud detect.) 2 MB/s riêng  │
                    │ Consumer C (audit log)     2 MB/s riêng  │
                    └──────────────────────────────────────────┘

Kinesis mở rộng tổng bandwidth cho shard đó lên 2 MB/s × 3 = 6 MB/s
→ Mỗi consumer luôn có 2 MB/s bất kể số consumers khác
→ Thêm consumer không ảnh hưởng tốc độ đọc của consumer khác
```

---

## Standard vs Enhanced Fan-Out — So Sánh

| Tiêu Chí | Standard Consumer | Enhanced Fan-Out |
|---|---|---|
| **Mô hình** | Pull — tự gọi `GetRecords` | Push — Kinesis đẩy data qua HTTP/2 |
| **Bandwidth per shard** | 2 MB/s **chia sẻ** tất cả consumers | 2 MB/s **riêng** cho từng consumer |
| **API** | `GetRecords` (polling) | `SubscribeToShard` (streaming) |
| **Độ trễ** | ~200ms (polling interval) | ~70ms (near real-time push) |
| **Giới hạn consumers** | Không giới hạn cứng | Tối đa 20 consumers đăng ký/stream |
| **GetRecords limit** | 5 calls/s per shard | Không áp dụng (push model) |
| **Chi phí thêm** | Không | $0,015/shard-hour + $0,013/GB |
| **Phù hợp** | Ít consumers, không cần tốc độ cao | Nhiều consumers, low latency, high throughput |

---

## Cơ Chế Hoạt Động

### HTTP/2 Server-Push

```
Standard Consumer (Polling — Kéo):
Consumer ──▶ GetRecords ──▶ Kinesis
         ◀──── Records ◀────
         
         (Lặp lại mỗi 200ms–1s)

Enhanced Fan-Out (Push — Đẩy):
Consumer ──▶ SubscribeToShard ──▶ Kinesis
         ◀───────────────────────────────
                Records liên tục qua HTTP/2
                (Kinesis chủ động đẩy, không cần polling)
```

### Subscription Lifecycle (Vòng Đời Đăng Ký)

```
1. Consumer đăng ký với stream (RegisterStreamConsumer)
   → Trạng thái: CREATING → ACTIVE (mất khoảng 10–15 giây)

2. Consumer gọi SubscribeToShard
   → Mở HTTP/2 streaming connection
   → Kinesis push records theo thời gian thực

3. Subscription hết hạn sau 5 phút
   → Consumer phải gọi lại SubscribeToShard

4. Consumer có thể deregister (DeregisterStreamConsumer)
   → Trạng thái: DELETING → xóa
```

---

## Đăng Ký Enhanced Fan-Out Consumer

### Bước 1 — Đăng Ký Consumer

```python
import boto3
import time

kinesis = boto3.client('kinesis', region_name='ap-southeast-1')

def register_efo_consumer(stream_name: str, consumer_name: str) -> str:
    """
    Đăng ký Enhanced Fan-Out consumer.
    Chỉ cần đăng ký một lần — consumer tồn tại đến khi deregister.
    """
    stream_arn = get_stream_arn(stream_name)
    
    try:
        response = kinesis.register_stream_consumer(
            StreamARN=stream_arn,
            ConsumerName=consumer_name
        )
        consumer_arn = response['Consumer']['ConsumerARN']
        consumer_status = response['Consumer']['ConsumerStatus']
        
        print(f"Đã đăng ký consumer: {consumer_name}")
        print(f"ConsumerARN: {consumer_arn}")
        print(f"Status: {consumer_status}")
        
        # Chờ consumer trở thành ACTIVE
        while True:
            desc = kinesis.describe_stream_consumer(ConsumerARN=consumer_arn)
            status = desc['ConsumerDescription']['ConsumerStatus']
            if status == 'ACTIVE':
                print("Consumer đã ACTIVE!")
                return consumer_arn
            elif status == 'CREATING':
                print(f"Đang tạo consumer, chờ...")
                time.sleep(5)
            else:
                raise Exception(f"Consumer ở trạng thái không mong đợi: {status}")
                
    except kinesis.exceptions.ResourceInUseException:
        # Consumer đã tồn tại — lấy ARN
        print(f"Consumer {consumer_name} đã tồn tại, lấy ARN...")
        return get_existing_consumer_arn(stream_arn, consumer_name)

def get_stream_arn(stream_name: str) -> str:
    response = kinesis.describe_stream_summary(StreamName=stream_name)
    return response['StreamDescriptionSummary']['StreamARN']

def get_existing_consumer_arn(stream_arn: str, consumer_name: str) -> str:
    response = kinesis.describe_stream_consumer(
        StreamARN=stream_arn,
        ConsumerName=consumer_name
    )
    return response['ConsumerDescription']['ConsumerARN']
```

### Bước 2 — Subscribe và Đọc Records

```python
import threading
import json
import base64
from botocore.eventstream import EventStreamError

def subscribe_and_read(consumer_arn: str, shard_id: str, handler_func):
    """
    Subscribe vào shard và xử lý records theo push model.
    """
    while True:  # Loop để tự động reconnect khi subscription hết hạn
        try:
            response = kinesis.subscribe_to_shard(
                ConsumerARN=consumer_arn,
                ShardId=shard_id,
                StartingPosition={
                    'Type': 'LATEST'  # Hoặc TRIM_HORIZON, AT_TIMESTAMP, AT_SEQUENCE_NUMBER
                }
            )
            
            # EventStream — Kinesis sẽ push records liên tục
            event_stream = response['EventStream']
            
            for event in event_stream:
                if 'SubscribeToShardEvent' in event:
                    shard_event = event['SubscribeToShardEvent']
                    records = shard_event.get('Records', [])
                    
                    for record in records:
                        data = json.loads(record['Data'].decode('utf-8'))
                        sequence_number = record['SequenceNumber']
                        handler_func(data, sequence_number)
                    
                    # Lưu continuation sequence number để reconnect sau
                    continuation_seq = shard_event.get('ContinuationSequenceNumber')
                    
        except (EventStreamError, Exception) as e:
            print(f"Subscription bị ngắt: {e}. Reconnect sau 1 giây...")
            time.sleep(1)
            # Loop tiếp → gọi lại subscribe_to_shard

# Ví dụ sử dụng
def process_order_event(data: dict, seq_number: str):
    print(f"Nhận event: {data['eventType']} cho order {data['orderId']}")
    # Xử lý business logic tại đây

# Chạy mỗi shard trong một thread riêng
consumer_arn = register_efo_consumer('order-events', 'fraud-detection-consumer')
shards = ['shardId-000000000000', 'shardId-000000000001']

threads = [
    threading.Thread(
        target=subscribe_and_read,
        args=(consumer_arn, shard_id, process_order_event),
        daemon=True
    )
    for shard_id in shards
]

for t in threads:
    t.start()
```

---

## Enhanced Fan-Out Với Lambda

### Cấu Hình Event Source Mapping

```python
import boto3

lambda_client = boto3.client('lambda')

def create_efo_event_source_mapping(
    function_name: str,
    stream_arn: str,
    consumer_arn: str,
    batch_size: int = 100
):
    """
    Tạo Event Source Mapping với Enhanced Fan-Out cho Lambda.
    Lambda tự động sử dụng EFO khi cấu hình StartingPosition + ConsumerARN.
    """
    response = lambda_client.create_event_source_mapping(
        FunctionName=function_name,
        EventSourceArn=stream_arn,
        StartingPosition='LATEST',
        BatchSize=batch_size,
        
        # Bật Enhanced Fan-Out
        SelfManagedEventSource={},  # Không dùng với Kinesis
        # Dùng consumer ARN để kích hoạt EFO
        AmazonManagedKafkaEventSourceConfig={},  # Không dùng
        
        # Cấu hình EFO chính xác
        FilterCriteria={
            'Filters': []
        },
        BisectBatchOnFunctionError=True,  # Chia batch khi Lambda lỗi
        MaximumRetryAttempts=3,
        DestinationConfig={
            'OnFailure': {
                'Destination': 'arn:aws:sqs:...:kinesis-dlq'  # DLQ cho records lỗi
            }
        }
    )
    return response

# Cấu hình qua AWS Console hoặc CloudFormation:
# EventSourceArn: <stream-arn>
# Consumer: <consumer-arn>  ← Chỉ định consumer ARN là bật EFO
# StartingPosition: LATEST
```

### Lambda Handler Với EFO

```python
# Lambda function được trigger bởi EFO không khác gì Standard
# Kinesis đẩy records, Lambda xử lý theo batch

import json
import base64

def lambda_handler(event, context):
    """
    Với EFO: Lambda nhận records được push từ Kinesis,
    không cần polling. Độ trễ thấp hơn (~70ms vs ~200ms).
    """
    successful = []
    failed = []
    
    for record in event['Records']:
        sequence_number = record['kinesis']['sequenceNumber']
        try:
            data = json.loads(
                base64.b64decode(record['kinesis']['data']).decode('utf-8')
            )
            
            # Xử lý record
            process_event(data)
            successful.append(sequence_number)
            
        except Exception as e:
            print(f"Lỗi xử lý {sequence_number}: {e}")
            failed.append({'itemIdentifier': sequence_number})
    
    print(f"Thành công: {len(successful)}, Thất bại: {len(failed)}")
    
    # Partial batch response — chỉ retry các records thất bại
    return {'batchItemFailures': failed}
```

---

## Enhanced Fan-Out Với KCL

### KCL v2 — Tự Động Dùng EFO

```java
// KCL version 2 hỗ trợ EFO tự động
// Chỉ cần cấu hình ConsumerArn là KCL tự dùng SubscribeToShard

ConfigsBuilder configsBuilder = new ConfigsBuilder(
    streamName,
    applicationName,
    kinesisClient,
    dynamoClient,
    cloudWatchClient,
    UUID.randomUUID().toString(),
    new OrderEventProcessorFactory()
);

// Cấu hình EFO
PollingConfig pollingConfig = configsBuilder.retrievalConfig()
    .retrievalSpecificConfig(
        new FanOutConfig(kinesisClient)
            .streamName(streamName)
            .consumerName("order-processor-consumer")
            // KCL tự đăng ký consumer nếu chưa tồn tại
    );

Scheduler scheduler = new Scheduler(
    configsBuilder.checkpointConfig(),
    configsBuilder.coordinatorConfig(),
    configsBuilder.leaseManagementConfig(),
    configsBuilder.lifecycleConfig(),
    configsBuilder.metricsConfig(),
    configsBuilder.processorConfig(),
    configsBuilder.retrievalConfig()
        .retrievalSpecificConfig(pollingConfig)
);

// Chạy scheduler
Thread schedulerThread = new Thread(scheduler);
schedulerThread.setDaemon(true);
schedulerThread.start();
```

---

## Giới Hạn Và Chi Phí

### Giới Hạn Enhanced Fan-Out

```
Per Stream:
- Tối đa 20 registered consumers per stream
- Tối đa 5 SubscribeToShard calls/giây/consumer (reconnect limit)

Per Shard per Consumer:
- 2 MB/s bandwidth (không chia sẻ với consumer khác)
- Không giới hạn số records (chỉ giới hạn bandwidth)

Subscription:
- Hết hạn sau 5 phút (phải reconnect)
- ConsumerStatus CREATING → ACTIVE: ~10–15 giây
```

### Chi Phí Enhanced Fan-Out

```
Chi phí thêm ngoài chi phí KDS thông thường:

1. Shard-Hour của EFO:
   $0,015 per shard-hour
   (Đây là chi phí cho shard khi có EFO consumer, không phải cho consumer)
   
2. Data Retrieval:
   $0,013 per GB của data được đọc qua EFO
   
Ví dụ tính chi phí:
- Stream: 5 shards, 3 EFO consumers
- Traffic: 1 GB/giờ (write vào stream)
- Mỗi consumer đọc toàn bộ: 1 GB/giờ × 3 = 3 GB/giờ

Chi phí EFO per tháng (720 giờ):
  Shard-hour: 5 shards × 720h × $0,015 = $54
  Data retrieval: 3 GB/h × 720h × $0,013 = $28,08
  Tổng EFO: $82,08/tháng

So sánh với Standard Consumer:
  Nếu không cần EFO: không tốn $82,08 này
  Nhưng với Standard, mỗi consumer chỉ được 2÷3 = 0,67 MB/s/shard
```

---

## Khi Nào Dùng Enhanced Fan-Out

### Nên Dùng EFO Khi

```
✅ Có từ 3+ consumers đọc cùng stream
   → Mỗi consumer cần đủ bandwidth để không lag

✅ Cần low latency (<100ms)
   → Fraud detection, real-time dashboard, trading system

✅ Consumer xử lý nặng và chậm
   → Nếu consumer xử lý 1.5 MB/s data mà bandwidth chỉ có 0.67 MB/s → lag liên tục

✅ Không muốn consumers ảnh hưởng lẫn nhau
   → Một consumer chậm không ảnh hưởng consumer khác
```

### Không Cần EFO Khi

```
❌ Chỉ có 1–2 consumers
   → Standard đủ, tiết kiệm chi phí

❌ Traffic thấp, bandwidth không phải bottleneck (nút cổ chai)
   → 100 records/s × 200 bytes = 20 KB/s << 2 MB/s → Standard thừa sức

❌ Consumers không cần real-time, chấp nhận vài giây lag
   → Standard polling đủ

❌ Chi phí là ưu tiên cao nhất
   → Standard không tốn thêm
```

### Decision Tree (Cây Quyết Định)

```
Có bao nhiêu consumers cùng đọc stream?
├── 1–2 consumers → STANDARD (đủ bandwidth)
└── 3+ consumers
    ├── Latency yêu cầu < 100ms? → EFO
    ├── Consumer có thể lag? → Standard nếu lag chấp nhận được
    └── Traffic cao (> 1.5 MB/s per shard)? → EFO (bandwidth không đủ chia)
```

---

## Kiến Trúc Fan-Out Thực Tế

### E-commerce Real-Time System

```
Kiến trúc với Enhanced Fan-Out:

Order Events Stream (10 shards)
    │
    ├──▶ Consumer 1: Fraud Detection (EFO — 2MB/s/shard)
    │         └── Lambda → Fraud scoring model → Block hoặc Allow
    │
    ├──▶ Consumer 2: Inventory Update (EFO — 2MB/s/shard)
    │         └── Lambda → DynamoDB (cập nhật tồn kho)
    │
    ├──▶ Consumer 3: Customer Notification (EFO — 2MB/s/shard)
    │         └── Lambda → SNS → Email/Push notification
    │
    ├──▶ Consumer 4: Analytics (Standard — chia bandwidth)
    │         └── Kinesis Analytics → Real-time metrics dashboard
    │         (Analytics chấp nhận độ trễ cao hơn → không cần EFO)
    │
    └──▶ Consumer 5: Audit Log (Standard — chia bandwidth)
              └── Kinesis Firehose → S3 (lưu trữ lâu dài)
              (Firehose buffer 60s, độ trễ không quan trọng → Standard đủ)

Tóm tắt:
- EFO: dùng cho consumers cần low latency và high throughput
- Standard: dùng cho consumers chấp nhận lag (analytics, logging)
- Kết hợp cả hai → tối ưu chi phí và performance
```

### Monitoring Enhanced Fan-Out

```python
def create_efo_monitoring_alarms(stream_name: str, consumer_name: str):
    """Tạo alarm giám sát EFO consumer."""
    
    cloudwatch = boto3.client('cloudwatch')
    
    # Metric quan trọng nhất: SubscribeToShard latency
    cloudwatch.put_metric_alarm(
        AlarmName=f'efo-{stream_name}-{consumer_name}-latency',
        MetricName='GetRecords.IteratorAgeMilliseconds',
        Namespace='AWS/Kinesis',
        Statistic='Maximum',
        Dimensions=[
            {'Name': 'StreamName', 'Value': stream_name},
            {'Name': 'ConsumerName', 'Value': consumer_name}
        ],
        Period=60,
        EvaluationPeriods=3,
        Threshold=5_000,  # 5 giây — EFO phải gần real-time, >5s là bất thường
        ComparisonOperator='GreaterThanThreshold',
        AlarmActions=['arn:aws:sns:...:kinesis-alerts']
    )
    
    # Số lần subscription bị reset
    cloudwatch.put_metric_alarm(
        AlarmName=f'efo-{stream_name}-{consumer_name}-reconnects',
        MetricName='SubscribeToShard.RateExceeded',
        Namespace='AWS/Kinesis',
        Statistic='Sum',
        Period=300,
        EvaluationPeriods=1,
        Threshold=5,
        ComparisonOperator='GreaterThanThreshold',
        AlarmActions=['arn:aws:sns:...:kinesis-alerts']
    )
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Enhanced Fan-Out khác Standard Consumer như thế nào?

**Trả lời:**
- **Model:** Standard = pull (GetRecords polling mỗi ~200ms); EFO = push (SubscribeToShard, HTTP/2 server-push)
- **Bandwidth:** Standard = 2MB/s chia sẻ tất cả consumers/shard; EFO = 2MB/s riêng mỗi consumer/shard
- **Latency:** Standard ~200ms; EFO ~70ms
- **Chi phí:** Standard không tốn thêm; EFO thêm $0,015/shard-hour + $0,013/GB read
- **Giới hạn:** Standard không giới hạn consumer; EFO tối đa 20 consumers/stream

### Q2: Khi nào nên dùng Enhanced Fan-Out?

**Trả lời:**
- 3+ consumers cùng đọc một stream — bandwidth không đủ nếu chia sẻ
- Latency yêu cầu dưới 100ms — fraud detection, real-time dashboard
- Consumers không muốn ảnh hưởng lẫn nhau — độc lập về throughput
- Không nên dùng khi: ít consumers (<3), chi phí ưu tiên, consumers chấp nhận lag

### Q3: Subscription EFO hết hạn sau 5 phút — xử lý thế nào?

**Trả lời:**
- Mỗi `SubscribeToShard` connection hết hạn sau 5 phút
- Consumer phải gọi lại `SubscribeToShard` trước hoặc ngay khi hết hạn
- KCL v2 tự động xử lý reconnect
- Khi tự code: dùng `ContinuationSequenceNumber` từ event cuối để tiếp tục đúng vị trí, không bị bỏ sót hay đọc trùng

### Q4: Làm thế nào để tích hợp EFO với Lambda?

**Trả lời:**
1. Đăng ký consumer bằng `RegisterStreamConsumer` → lấy ConsumerARN
2. Tạo Lambda Event Source Mapping với ConsumerARN thay vì StreamARN
3. Lambda tự động được trigger qua push model — không cần code SubscribeToShard
4. Handler code giống hệt Standard Consumer (decode base64, xử lý records)
5. Sự khác biệt chỉ ở cấu hình Event Source Mapping — code Lambda giống nhau

---

## Tổng Kết Module 05-Kinesis

```
Amazon Kinesis — Bức Tranh Tổng Thể

Kinesis Data Streams (KDS)
    ├── Shard: đơn vị throughput (1MB/s write, 2MB/s read)
    ├── Partition Key: routing + ordering
    ├── Producer: PutRecord, PutRecords, KPL
    ├── Consumer Standard: GetRecords, polling, chia sẻ bandwidth
    ├── Consumer EFO: SubscribeToShard, push, bandwidth riêng
    ├── Retention: 24h → 365 ngày, hỗ trợ replay
    └── Mode: Provisioned (quản lý shard) | On-Demand (tự động)

Kinesis Data Firehose
    ├── Fully managed delivery
    ├── Destinations: S3, Redshift, OpenSearch, HTTP
    ├── Buffer: 60–900s (near-real-time)
    ├── Transform: Lambda
    └── Format: JSON → Parquet/ORC

Kinesis Data Analytics
    ├── SQL Application: window queries đơn giản
    ├── Apache Flink: stateful, complex, exactly-once
    ├── Windows: Tumbling, Sliding, Session
    └── Anomaly Detection: RANDOM_CUT_FOREST

Shard Management
    ├── Tính toán: MAX(write_records/1000, write_MB/1, read_bandwidth_need)
    ├── Hot Shard: partition key skewed → split + fix key
    ├── Split: tăng throughput
    ├── Merge: giảm chi phí
    └── Iterator Age: metric quan trọng nhất (consumer lag indicator)

Enhanced Fan-Out
    ├── 2MB/s per consumer per shard (không chia sẻ)
    ├── Push model (~70ms latency)
    ├── Max 20 consumers/stream
    └── Dùng khi: 3+ consumers, low-latency yêu cầu
```

---

**Module Hoàn Thành!** Tiếp tục với [06-amazon-mq/](../06-amazon-mq/) hoặc xem [08-patterns/1-service-comparison.md](../08-patterns/1-service-comparison.md) để so sánh Kinesis với SQS, SNS, EventBridge.
