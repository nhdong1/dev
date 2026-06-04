# ⚖️ Kinesis vs Apache Kafka / MSK — So Sánh Toàn Diện

> Kinesis Data Streams và Apache Kafka (qua Amazon MSK — Managed Streaming for Apache Kafka — Kafka Được Quản Lý) đều là nền tảng streaming phân tán hàng đầu. Hiểu rõ trade-offs giúp chọn đúng công cụ cho từng bài toán — đây cũng là câu hỏi phỏng vấn cực kỳ phổ biến.

## 📚 Mục Lục

1. [Tổng Quan Nhanh](#tổng-quan-nhanh)
2. [So Sánh Kiến Trúc](#so-sánh-kiến-trúc)
3. [So Sánh Chi Tiết Từng Khía Cạnh](#so-sánh-chi-tiết-từng-khía-cạnh)
4. [Amazon MSK — Managed Kafka](#amazon-msk--managed-kafka)
5. [Ma Trận Quyết Định](#ma-trận-quyết-định)
6. [Migration Giữa 2 Platform](#migration-giữa-2-platform)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tổng Quan Nhanh

```
┌────────────────────────────────────────────────────────────────────┐
│                    CHỌN NHANH                                       │
├───────────────────────────────┬────────────────────────────────────┤
│   KINESIS DATA STREAMS        │   APACHE KAFKA / MSK               │
├───────────────────────────────┼────────────────────────────────────┤
│ ✅ AWS-native, zero ops       │ ✅ Rich ecosystem, cross-cloud      │
│ ✅ Tích hợp IAM, Lambda, Glue │ ✅ Higher throughput potential      │
│ ✅ Scale tự động (on-demand)  │ ✅ Longer retention (unlimited)     │
│ ✅ Pay-as-you-go linh hoạt    │ ✅ Kafka Streams, ksqlDB, Debezium  │
│ ❌ Chỉ chạy trên AWS          │ ❌ Cần quản lý brokers (MSK vẫn cần)|
│ ❌ 365 ngày retention tối đa  │ ✅ Retention không giới hạn         │
│ ❌ Ít tool ecosystem hơn      │ ❌ Phức tạp hơn để setup đúng       │
└───────────────────────────────┴────────────────────────────────────┘
```

---

## 🏗️ So Sánh Kiến Trúc

### Kinesis Data Streams — Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Kinesis Data Stream                         │
│                                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │ Shard 1 │  │ Shard 2 │  │ Shard N │  ← AWS managed  │
│  │ [──────►│  │ [──────►│  │ [──────►│                 │
│  │ append  │  │ append  │  │ append  │                 │
│  │  only]  │  │  only]  │  │  only]  │                 │
│  └─────────┘  └─────────┘  └─────────┘                 │
│                                                          │
│  Partition Key → Hash → Shard Assignment                │
│  Mỗi shard: 1 MB/s write, 2 MB/s read                  │
│  Replication: Tự động 3 AZs (Availability Zones)        │
└─────────────────────────────────────────────────────────┘
```

### Apache Kafka — Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│                 Kafka Cluster                            │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │  ← User/MSK │
│  │  [Topic: │  │ [Topic:  │  │ [Topic:  │    managed  │
│  │  orders] │  │  orders] │  │  orders] │             │
│  │ Partition│  │ Partition│  │ Partition│             │
│  │  0 (lead)│  │  1 (lead)│  │  2 (lead)│             │
│  └──────────┘  └──────────┘  └──────────┘             │
│                                                          │
│  ZooKeeper / KRaft: Cluster coordination                │
│  Topic: Nhiều partitions, replicated across brokers     │
└─────────────────────────────────────────────────────────┘
```

### Khái Niệm Tương Đồng

| Kinesis               | Kafka                      | Mô Tả                                    |
| --------------------- | -------------------------- | ---------------------------------------- |
| **Stream**            | **Topic** (Chủ Đề)         | Luồng dữ liệu logic                      |
| **Shard**             | **Partition** (Phân Vùng)  | Đơn vị parallelism và throughput          |
| **Partition Key**     | **Message Key** (Khóa Tin) | Quyết định partition nào nhận record      |
| **Sequence Number**   | **Offset** (Vị Trí)        | ID của record trong shard/partition       |
| **Consumer**          | **Consumer Group** (Nhóm Consumer) | Tập hợp consumers đọc stream       |
| **Shard Iterator**    | **Offset Commit** (Xác Nhận Vị Trí) | Theo dõi vị trí đọc              |
| **Retention Period**  | **Retention Period**       | Thời gian giữ dữ liệu                    |

---

## 📊 So Sánh Chi Tiết Từng Khía Cạnh

### 1. Throughput (Thông Lượng)

| Khía Cạnh                   | Kinesis                           | Kafka / MSK                          |
| --------------------------- | --------------------------------- | ------------------------------------ |
| **Giới hạn per shard/partition** | 1 MB/s write, 2 MB/s read     | Tùy cấu hình broker (thường 100+ MB/s) |
| **Tổng throughput**         | Tăng bằng cách thêm shard         | Tăng bằng cách thêm broker/partition |
| **Peak throughput thực tế** | Hàng chục GB/s (nhiều shard)     | Hàng trăm GB/s (nhiều broker)        |

```
Kinesis có giới hạn per-shard cứng (1MB/s write):
→ 1 GB/s cần ít nhất 1000 shards
→ Chi phí: 1000 × $0.015/h = $15/h = $10,800/tháng

Kafka: Throughput tùy broker size, không giới hạn per-partition như Kinesis
→ 1 GB/s có thể đạt với 10 broker i3.4xlarge
→ Chi phí MSK: thấp hơn cho ultra-high throughput
```

**Kết luận:** Kafka có lợi thế ở throughput cực cao (> 1 GB/s liên tục).

### 2. Latency (Độ Trễ)

| Khía Cạnh               | Kinesis                         | Kafka                           |
| ----------------------- | ------------------------------- | ------------------------------- |
| **Producer latency**    | 70-200ms (HTTPS overhead)       | < 10ms (binary protocol)        |
| **Consumer latency**    | ~200ms (standard) / ~70ms (EFO) | 5-20ms                          |
| **End-to-end**          | 200-500ms                       | 10-50ms                         |

**Kết luận:** Kafka có latency thấp hơn đáng kể. Với Kinesis EFO, khoảng cách thu hẹp lại nhưng Kafka vẫn nhanh hơn.

### 3. Retention (Lưu Giữ Dữ Liệu)

| Kinesis                         | Kafka                               |
| ------------------------------- | ----------------------------------- |
| 24h mặc định                    | 7 ngày mặc định                     |
| Tối đa 365 ngày                 | Không giới hạn (cấu hình storage)  |
| Tốn phí thêm cho extended       | Tốn storage cost của broker         |

**Kết luận:** Kafka linh hoạt hơn nhiều về retention. Nếu cần lưu data months/years cho replay → Kafka.

### 4. Consumer Model (Mô Hình Consumer)

#### Kinesis Consumer Model

```python
# Kinesis: Mỗi consumer tự track vị trí (ShardIterator)
# Không có khái niệm Consumer Group built-in
# KCL cung cấp consumer group logic qua DynamoDB

# Standard: 5 GetRecords calls/s/shard, chia sẻ 2MB/s
# Enhanced Fan-out: Mỗi consumer 2MB/s riêng
```

#### Kafka Consumer Group Model

```python
# Kafka: Consumer Group = tập hợp consumers chia partitions
# Group 1: Consumer A nhận partition 0,1 | Consumer B nhận partition 2,3
# Group 2: Consumer C nhận tất cả partitions

# Lợi ích: Tự động rebalancing khi consumer join/leave
# Offset được lưu trong __consumer_offsets topic

from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'transactions',
    bootstrap_servers=['broker1:9092', 'broker2:9092'],
    group_id='fraud-detection-group',  # Consumer group
    auto_offset_reset='latest',
    enable_auto_commit=True,
    auto_commit_interval_ms=5000
)

for message in consumer:
    data = json.loads(message.value)
    process_transaction(data)
```

**Kết luận:** Kafka Consumer Groups đơn giản và mạnh mẽ hơn. Kinesis Enhanced Fan-out gần tương đương nhưng phức tạp hơn.

### 5. Scaling (Tăng Giảm Quy Mô)

| Khía Cạnh                | Kinesis                              | Kafka                               |
| ------------------------ | ------------------------------------ | ----------------------------------- |
| **Scaling unit**         | Shard                                | Partition, Broker                   |
| **Auto-scaling**         | On-demand mode (tự động)            | Không tự động (MSK Serverless: có) |
| **Scale up time**        | Vài phút                             | Vài phút đến vài giờ               |
| **Rebalancing**          | Không cần (AWS managed)             | Partition rebalance khi scale       |
| **Giới hạn**             | 500 shards/stream (tăng theo request)| Không giới hạn về lý thuyết        |

### 6. Ecosystem (Hệ Sinh Thái)

| Kinesis                           | Kafka                                        |
| --------------------------------- | -------------------------------------------- |
| AWS Lambda (built-in trigger)     | Kafka Streams (stream processing library)    |
| Kinesis Data Firehose             | ksqlDB (SQL trên Kafka streams)              |
| Kinesis Data Analytics (Flink)    | Kafka Connect (connector framework — 200+ connectors) |
| AWS Glue                          | Debezium (Change Data Capture — CDC)         |
| Amazon S3, Redshift, OpenSearch   | Schema Registry (Quản Lý Schema)            |
| —                                 | Flink, Spark Structured Streaming, Storm     |

**Kết luận:** Kafka có hệ sinh thái phong phú hơn nhiều. Đặc biệt **Kafka Connect** với 200+ connectors và **Debezium** cho CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu) là lợi thế lớn.

### 7. Bảo Mật (Security)

| Khía Cạnh            | Kinesis                          | Kafka / MSK                      |
| -------------------- | -------------------------------- | -------------------------------- |
| **Authentication**   | IAM — tích hợp hoàn toàn AWS    | SASL/SCRAM, IAM (MSK), TLS certs |
| **Authorization**    | IAM policies, resource policies  | ACLs, IAM (MSK)                  |
| **Encryption rest**  | KMS managed (1 click)            | KMS (MSK), tự cấu hình           |
| **Encryption transit** | HTTPS/TLS mặc định            | TLS (cần cấu hình)               |
| **VPC**              | VPC endpoints                    | VPC native (MSK chạy trong VPC)  |

**Kết luận:** Kinesis đơn giản hơn về bảo mật nhờ tích hợp IAM. MSK cần cấu hình thêm nhưng linh hoạt hơn.

### 8. Chi Phí (Cost)

```
Tình huống so sánh: 100 MB/s throughput, 7 ngày retention, ap-southeast-1

KINESIS DATA STREAMS:
Số shard cần: ceil(100 MB/s / 1 MB/s) = 100 shards
Shard cost: 100 × $0.015 × 24 × 30 = $1,080/tháng
PUT cost: 100 MB/s × 60s × 60m × 24h × 30 days / 25KB × $0.014/1M
        ≈ $580/tháng
Extended retention (7 ngày): 100 × $0.02 × 24 × 30 = $1,440/tháng
TỔNG: ~$3,100/tháng

MSK (Amazon Managed Kafka):
3 brokers kafka.m5.4xlarge: 3 × $0.68 × 24 × 30 = $1,469/tháng
EBS storage: 100 MB/s × 604,800s × 7 / 1024 = ~410 GB × $0.10 = $41/tháng
Data transfer: ~$0.02/GB = minimal
TỔNG: ~$1,510/tháng

→ MSK rẻ hơn đáng kể ở high throughput + long retention
→ Kinesis rẻ hơn ở low throughput + ngắn hạn (ít shard, không cần broker)
```

---

## 🚀 Amazon MSK — Managed Kafka

### MSK Là Gì

**Amazon MSK** (Managed Streaming for Apache Kafka — Kafka Được Quản Lý) là dịch vụ AWS chạy Kafka cluster mà không cần quản lý infrastructure:

- AWS lo: Provisioning brokers, ZooKeeper/KRaft, patching, monitoring
- Bạn lo: Cấu hình topics, producers/consumers, application logic

### Các Tùy Chọn MSK

| Loại                       | Mô Tả                                    | Khi Nào Dùng                    |
| -------------------------- | ---------------------------------------- | ------------------------------- |
| **MSK Provisioned**        | Broker instances cố định bạn chọn        | Production, predictable traffic |
| **MSK Serverless**         | Tự động scale, trả phí theo dùng         | Unpredictable traffic, dev/test |
| **MSK Connect**            | Managed Kafka Connect connectors         | Tích hợp với databases, S3      |

### MSK vs Self-managed Kafka

```
MSK Provisioned:
✅ Không cần quản lý broker, ZooKeeper, OS patching
✅ Multi-AZ built-in (3 AZ mặc định)
✅ Tích hợp CloudWatch, IAM, VPC
❌ Không toàn quyền control (không SSH vào broker)
❌ Không hỗ trợ tất cả Kafka configs
❌ Chỉ chạy trên AWS

Self-managed Kafka on EC2:
✅ Toàn quyền cấu hình
✅ Có thể chạy multi-cloud
❌ Tự quản lý patching, HA, scaling
❌ Ops burden cao
```

### Kafka Concepts Quan Trọng Cho MSK

```python
# Tạo topic với retention và replication factor
from confluent_kafka.admin import AdminClient, NewTopic

admin = AdminClient({'bootstrap.servers': 'broker1:9092,broker2:9092'})

topic = NewTopic(
    topic='user-events',
    num_partitions=12,       # Số partitions → parallelism
    replication_factor=3,    # Mỗi partition có 3 bản sao
    config={
        'retention.ms': str(7 * 24 * 60 * 60 * 1000),  # 7 ngày
        'min.insync.replicas': '2',  # Cần ít nhất 2 replicas đồng bộ để ghi
        'compression.type': 'lz4'   # Nén dữ liệu
    }
)
admin.create_topics([topic])
```

```python
# Producer với acks=all (đảm bảo durability)
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'broker1:9092,broker2:9092',
    'acks': 'all',              # Chờ tất cả ISR (In-Sync Replicas) xác nhận
    'retries': 3,
    'compression.type': 'lz4',
    'linger.ms': 5,             # Chờ 5ms để gom records → tăng throughput
    'batch.size': 65536         # 64KB batch size
})

def delivery_callback(err, msg):
    if err:
        print(f'Delivery failed: {err}')
    else:
        print(f'Delivered to {msg.topic()} [{msg.partition()}]')

producer.produce(
    topic='user-events',
    key=user_id.encode('utf-8'),   # Key → quyết định partition
    value=json.dumps(event).encode('utf-8'),
    callback=delivery_callback
)
producer.flush()
```

```python
# Consumer với consumer group
from confluent_kafka import Consumer

consumer = Consumer({
    'bootstrap.servers': 'broker1:9092,broker2:9092',
    'group.id': 'analytics-consumer-group',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,   # Manual commit để kiểm soát
    'max.poll.interval.ms': 300000
})

consumer.subscribe(['user-events'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            handle_error(msg.error())
            continue
        
        data = json.loads(msg.value().decode('utf-8'))
        process_event(data)
        
        # Manual commit sau khi xử lý thành công
        consumer.commit(msg)
finally:
    consumer.close()
```

---

## 🎯 Ma Trận Quyết Định

### Chọn Kinesis Khi

```
✅ Stack thuần AWS — muốn tích hợp dễ với Lambda, Glue, S3, IAM
✅ Team không có Kafka expertise
✅ Throughput vừa phải (< 1 GB/s, < 100 shard)
✅ Cần setup nhanh, zero-ops
✅ Budget: predictable, pay-per-shard
✅ Retention ngắn (< 7 ngày)
✅ Serverless/event-driven architecture thuần AWS
✅ Cần Kinesis Firehose deliver thẳng vào S3/Redshift
```

### Chọn Kafka / MSK Khi

```
✅ Cần Kafka ecosystem: Connect, Streams, ksqlDB, Debezium
✅ Multi-cloud hoặc hybrid cloud (on-prem + AWS)
✅ Ultra-high throughput (> 1 GB/s)
✅ Cần retention dài (> 365 ngày)
✅ Team đã quen Kafka
✅ Cần Change Data Capture (CDC) từ database
✅ Cần Kafka Streams hoặc ksqlDB
✅ Cost-sensitive ở high throughput
✅ Cần custom Kafka configurations
```

### Có Thể Dùng Cả Hai (Hybrid Pattern)

```
Tình huống: Hệ thống e-commerce lớn

MSK (Kafka):
- Order events, payment events — cần low latency
- CDC từ PostgreSQL (Debezium) → capture database changes
- Kafka Streams: business logic phức tạp

Kinesis:
- Clickstream, pageview events — gửi từ frontend SDK đơn giản
- Firehose → S3 cho data lake
- Lambda trigger cho serverless processing

→ Kafka làm "backbone" streaming
→ Kinesis làm "ingest layer" từ browser/mobile
```

---

## 🔄 Migration Giữa 2 Platform

### Từ Kinesis → Kafka

**Lý do thường gặp:**
- Throughput tăng cao, Kinesis trở nên đắt
- Cần Kafka Streams hoặc ksqlDB
- Cần multi-cloud

**Chiến lược:**

```
Phase 1: Dual-write (Ghi Song Song)
  Producer → KDS (existing)
  Producer → MSK (new, parallel)
  → Verify MSK nhận đủ dữ liệu

Phase 2: Consumer migration
  Từng consumer migrate dần sang MSK
  → Giám sát kỹ consumer lag

Phase 3: Cut-over (Chuyển Đổi)
  Stop writing to KDS
  KDS → chỉ đọc historical data
  
Phase 4: Cleanup
  Chờ KDS retention expire
  Delete KDS stream
```

### Từ Kafka → Kinesis

**Lý do thường gặp:**
- Muốn giảm ops overhead
- Đồng nhất stack về AWS

```python
# Kafka → Kinesis bridge (dùng Kafka Connect Kinesis Sink)
{
    "name": "kinesis-sink",
    "config": {
        "connector.class": "com.amazon.kinesis.kafka.AmazonKinesisSinkConnector",
        "tasks.max": "4",
        "topics": "user-events",
        "region": "ap-southeast-1",
        "streamName": "user-events-kinesis",
        "partitionKey": "${kafka:key}"
    }
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn

### Q1: Kinesis vs Kafka — khi nào chọn cái nào?

**A:** Phụ thuộc vào requirements:
- **Kinesis:** AWS-only stack, zero-ops, throughput < 1 GB/s, cần tích hợp native với Lambda/Glue/S3, team không có Kafka expertise.
- **Kafka/MSK:** Cần Kafka ecosystem (Connect, Streams, ksqlDB, Debezium), multi-cloud, ultra-high throughput, retention > 365 ngày, team đã quen Kafka.

Thực tế: Nhiều công ty dùng cả hai — Kafka làm backbone streaming chính, Kinesis làm ingest layer từ mobile/web.

### Q2: Khái niệm Shard (Kinesis) và Partition (Kafka) khác nhau thế nào?

**A:** Về mặt chức năng khá tương đồng — đều là đơn vị parallelism và ordering. Khác biệt chính:
- **Shard:** Giới hạn cứng 1 MB/s write, 2 MB/s read. AWS quản lý hoàn toàn. Tốn phí theo giờ.
- **Partition:** Throughput phụ thuộc broker capacity — có thể cao hơn nhiều. Bạn quyết định số partition khi tạo topic. Không tốn phí riêng.

Cả hai đều đảm bảo thứ tự trong shard/partition, không đảm bảo cross-shard/partition.

### Q3: Tại sao Kafka latency thấp hơn Kinesis?

**A:** Kafka dùng **binary TCP protocol** (giao thức nhị phân qua TCP) — rất nhanh, overhead thấp. Kinesis dùng **HTTPS/REST API** — overhead TLS handshake, HTTP headers. Về mặt số liệu: Kafka đạt 5-20ms end-to-end, Kinesis standard ~200ms (Enhanced Fan-out ~70ms). Với Kinesis EFO dùng HTTP/2 push, khoảng cách thu hẹp lại nhưng Kafka vẫn nhanh hơn.

### Q4: Consumer Group (Kafka) vs Kinesis Consumer — cái nào tốt hơn?

**A:** Kafka Consumer Groups đơn giản và mạnh mẽ hơn:
- **Kafka:** Mỗi Consumer Group có offset riêng, rebalancing tự động khi consumer thêm/bớt, offset lưu trong Kafka.
- **Kinesis Standard:** Chia sẻ 2MB/s/shard, không có group concept built-in, cần KCL + DynamoDB để manage.
- **Kinesis Enhanced Fan-out:** Mỗi consumer nhận 2MB/s riêng — gần tương đương Kafka, nhưng cần đăng ký và tốn thêm phí.

KCL giải quyết được nhiều vấn đề của Kinesis consumer nhưng thêm dependency DynamoDB.

### Q5: MSK Serverless vs Kinesis On-demand — khác nhau thế nào?

**A:** Cả hai đều auto-scale và tính phí theo dùng:
- **Kinesis On-demand:** Tự động điều chỉnh shard, tính phí theo record + GB. Đơn giản nhất, zero-config.
- **MSK Serverless:** Kafka serverless, tính phí theo throughput (MB/h) + storage. Vẫn cần cấu hình Kafka topics, không hoàn toàn zero-ops như Kinesis On-demand.

---

## 📊 Bảng So Sánh Tổng Kết

| Tiêu Chí                      | Kinesis Data Streams          | Apache Kafka / MSK              |
| ----------------------------- | ----------------------------- | ------------------------------- |
| **Throughput**                | 1 MB/s/shard                  | 100+ MB/s/broker                |
| **Latency**                   | 70-200ms                      | 5-20ms                          |
| **Retention**                 | 24h → 365 ngày                | Không giới hạn                  |
| **Ops complexity**            | Thấp (managed)                | Trung bình (MSK) / Cao (self)   |
| **Ecosystem**                 | AWS native                    | Kafka ecosystem (200+ connectors)|
| **Pricing model**             | Shard-giờ + data              | Instance-giờ + storage          |
| **Multi-cloud**               | Không                         | Có                              |
| **Consumer model**            | ShardIterator / KCL / EFO     | Consumer Groups (đơn giản hơn)  |
| **CDC support**               | Không native                  | Debezium, Kafka Connect JDBC    |
| **AWS integration**           | Xuất sắc                      | Tốt (MSK)                       |
| **Scalability**               | Tốt (on-demand)               | Rất tốt                         |
| **Thích hợp nhất**            | AWS-native, low-medium scale  | High scale, rich ecosystem      |

---

## 🔗 Điều Hướng

| Quay Lại                                              | Đầu Module                          |
| ----------------------------------------------------- | ----------------------------------- |
| [3-kinesis-analytics.md](./3-kinesis-analytics.md)   | [README.md](./README.md)            |
| [../03-glue/README.md](../03-glue/README.md)          | [../10-msk/README.md](../10-msk/README.md) — Chi tiết MSK |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** 4-kinesis-vs-kafka.md
**Trạng Thái:** ✅ Hoàn thành
