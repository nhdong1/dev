# MSK vs Kinesis — So Sánh Chi Tiết & Tiêu Chí Lựa Chọn

> So sánh toàn diện giữa Amazon MSK (Managed Streaming for Apache Kafka — Kafka Được Quản Lý) và Amazon Kinesis Data Streams (Luồng Dữ Liệu Kinesis) — hai dịch vụ streaming chính trên AWS. Hiểu rõ trade-offs (đánh đổi) để đưa ra quyết định đúng cho từng use case.

## 📚 Mục Lục

1. [Tổng Quan So Sánh](#tổng-quan-so-sánh)
2. [Kiến Trúc So Sánh](#kiến-trúc-so-sánh)
3. [So Sánh Chi Tiết Từng Khía Cạnh](#so-sánh-chi-tiết-từng-khía-cạnh)
4. [Mô Hình Chi Phí](#mô-hình-chi-phí)
5. [Tích Hợp AWS Native](#tích-hợp-aws-native)
6. [Use Cases Thực Tế](#use-cases-thực-tế)
7. [Decision Tree — Cây Quyết Định](#decision-tree--cây-quyết-định)
8. [Migration Scenarios](#migration-scenarios--kịch-bản-di-chuyển)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan So Sánh

| Khía Cạnh                          | Amazon MSK (Kafka)                              | Kinesis Data Streams                             |
| ---------------------------------- | ----------------------------------------------- | ------------------------------------------------ |
| **Nền tảng công nghệ**             | Apache Kafka (open-source)                      | Proprietary (độc quyền AWS)                      |
| **Mô hình quản lý**                | Managed Kafka — AWS lo infra, bạn lo Kafka config | Fully managed — AWS lo tất cả                  |
| **Đơn vị parallelism**             | Partition (không giới hạn số lượng)             | Shard (Mảnh) — tính phí theo shard              |
| **Throughput tối đa**              | Rất cao — theo số broker và partition           | 1 MB/s write + 2 MB/s read mỗi shard            |
| **Retention tối đa**               | Không giới hạn (cấu hình tùy ý + Tiered Storage)| 365 ngày (Extended Retention tính phí thêm)     |
| **Consumer model**                 | Pull — consumer tự kéo theo offset              | Pull (GetRecords) + Push (Lambda trigger)        |
| **Tích hợp AWS native**            | Trung bình — cần cấu hình thêm                  | Cao — Lambda, Firehose, Glue tích hợp sẵn       |
| **Kafka ecosystem**                | ✅ Đầy đủ (Kafka Streams, Connect, KSQL)        | ❌ Không có                                      |
| **Vendor lock-in**                 | Thấp — Kafka là open-source                     | Cao — API độc quyền của AWS                      |
| **Overhead vận hành**              | Trung bình — vẫn cần tuning Kafka configs       | Thấp — gần như không cần tuning                  |
| **Chi phí khởi điểm**              | Cao hơn — phải có broker instances              | Thấp hơn — pay per shard giờ                    |

---

## Kiến Trúc So Sánh

### Kinesis Data Streams — KDS

```
┌──────────────────────────────────────────────────────────┐
│               Kinesis Data Stream                         │
│                                                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │ Shard-1 │  │ Shard-2 │  │ Shard-3 │  │ Shard-N │    │
│  │1MB/s in │  │1MB/s in │  │1MB/s in │  │1MB/s in │    │
│  │2MB/s out│  │2MB/s out│  │2MB/s out│  │2MB/s out│    │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │
│       │             │             │             │         │
└───────┼─────────────┼─────────────┼─────────────┼────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                              │
                   Consumer (Lambda / KCL App)
                   
KCL — Kinesis Client Library (Thư Viện Client Kinesis)
AWS quản lý: shards, replication, scaling (bạn phải ResplitMerge thủ công)
```

### Amazon MSK — Kafka

```
┌────────────────────────────────────────────────────────────────┐
│                    MSK Cluster                                  │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│  │ Broker-1 │   │ Broker-2 │   │ Broker-3 │                   │
│  │  AZ: 1a  │   │  AZ: 1b  │   │  AZ: 1c  │                   │
│  └──────────┘   └──────────┘   └──────────┘                   │
│                                                                 │
│  Topic "events" — 12 Partitions                                │
│  Part 0,3,6,9  Part 1,4,7,10  Part 2,5,8,11                   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         Consumer-1      Consumer-2      Consumer-3
         (Group: A)      (Group: A)      (Group: A)
              │               │               │
         Consumer-4      Consumer-5
         (Group: B)      (Group: B)
         
Nhiều consumer groups độc lập đọc cùng topic
AWS quản lý: broker infra; bạn quản lý: topic config, consumer groups
```

---

## So Sánh Chi Tiết Từng Khía Cạnh

### 1. Throughput (Thông Lượng)

```
Kinesis Data Streams:
  Mỗi shard:
    - Write: 1 MB/s hoặc 1.000 records/giây (lấy giới hạn nào đến trước)
    - Read: 2 MB/s (chia sẻ giữa TẤT CẢ consumers của shard đó)
    - Enhanced Fan-Out (Phân Phát Mở Rộng): mỗi consumer có 2 MB/s riêng

  Ví dụ: 100 MB/s write = 100 shards
  Chi phí: 100 shards × $0.015/giờ = $1.50/giờ = ~$1.080/tháng (chỉ shard cost)

MSK:
  Mỗi broker (kafka.m5.4xlarge):
    - Network: 10 Gbps (= ~1.25 GB/s tổng)
    - Thực tế sustainable: ~200-400 MB/s per broker

  3 brokers kafka.m5.4xlarge:
    - Tổng throughput: ~600 MB/s - 1 GB/s
  Chi phí: 3 × ~$0.40/giờ = $1.20/giờ = ~$864/tháng (chưa EBS)

Kết luận throughput:
  - KDS phù hợp low-medium throughput (< 500 MB/s) — predictable cost
  - MSK hiệu quả hơn ở high throughput (> 500 MB/s) 
```

### 2. Retention (Lưu Giữ Dữ Liệu)

```
Kinesis Data Streams:
  Mặc định:   24 giờ (miễn phí)
  Extended:   7 ngày (tính phí thêm)
  Long-term: 365 ngày (tính phí thêm nữa)
  
  Replay: Có — theo sequence number, tối đa trong retention window

MSK:
  Mặc định:   7 ngày (cấu hình được)
  Tối đa:     Không giới hạn (cấu hình retention.ms=-1)
  Tiered Storage: Lưu trữ vô thời hạn trên S3 với chi phí thấp

  Replay: Có — theo offset, bất kỳ thời điểm nào trong retention period

Kết luận retention:
  MSK rõ ràng thắng nếu cần retention dài hạn hoặc replay lịch sử
```

### 3. Ordering (Thứ Tự)

```
Kinesis Data Streams:
  - Ordering trong cùng shard: ✅ Đảm bảo
  - Ordering cross-shard: ❌ Không đảm bảo
  - Partition key → quyết định shard (tương tự Kafka)

MSK (Kafka):
  - Ordering trong cùng partition: ✅ Đảm bảo
  - Ordering cross-partition: ❌ Không đảm bảo
  - Message key → quyết định partition

Cả hai đều có cùng ordering semantics (ngữ nghĩa thứ tự):
  Đảm bảo per-key ordering trong cùng partition/shard
```

### 4. Delivery Semantics (Ngữ Nghĩa Giao Hàng)

```
Kinesis Data Streams:
  - At-least-once (Ít Nhất Một Lần) — mặc định
  - Exactly-once (Chính Xác Một Lần) — cần idempotent consumer logic

MSK (Kafka):
  - At-least-once — mặc định
  - At-most-once (Nhiều Nhất Một Lần) — với enable.auto.commit=true (rủi ro)
  - Exactly-once — Kafka Transactions (Giao Dịch Kafka) với:
    * enable.idempotence=true trên producer
    * transactional.id trên producer
    * isolation.level=read_committed trên consumer

MSK thắng về exactly-once nếu dùng Kafka Transactions đúng cách
```

### 5. Scaling (Mở Rộng)

```
Kinesis Data Streams:
  Scale OUT: SplitShard (Tách Mảnh) — tốn 30 giây, mất ordering trong khi reshard
  Scale IN: MergeShards (Gộp Mảnh) — tương tự
  Auto-scaling: Cần AWS Application Auto Scaling hoặc Lambda trigger
  
  Nhược điểm: Không thể scale trong ngày nhiều hơn 2x số shard hiện tại

MSK:
  Scale OUT (Thêm Partition): Không ảnh hưởng ordering của messages hiện tại
  Scale OUT (Thêm Broker): Cần rebalance partitions — có tool tự động
  Scale broker type: Rolling restart (khởi động lại tuần tự) — downtime thấp
  
  MSK Serverless: Auto-scale hoàn toàn

Kinesis KDS Scaling phức tạp và có giới hạn; MSK linh hoạt hơn
```

### 6. Fan-out (Phân Phát Cho Nhiều Consumer)

```
Kinesis Data Streams:
  Standard: 2 MB/s PER SHARD chia sẻ giữa TẤT CẢ consumers
  → 5 consumers đọc 1 shard → mỗi consumer chỉ được 400 KB/s
  
  Enhanced Fan-Out: Mỗi consumer có 2 MB/s riêng
  → 5 consumers đọc 1 shard → mỗi consumer được 2 MB/s đầy đủ
  → Tốn thêm phí: $0.015/shard/giờ + $0.013/GB data retrieved

MSK (Kafka):
  Mỗi consumer group có throughput đầy đủ của partition
  N consumer groups → N × throughput (không giảm lẫn nhau)
  → Hoàn toàn miễn phí, không cần cấu hình đặc biệt

MSK rõ ràng thắng cho multi-consumer scenarios
```

---

## Mô Hình Chi Phí

### Kinesis Data Streams — Chi Phí

```
Thành phần chi phí KDS (us-east-1):

1. Shard giờ:
   $0.015 / shard / giờ
   = $10.80 / shard / tháng

2. PUT Payload Units (Đơn Vị Tải Trọng PUT):
   $0.014 / 1,000,000 PUT units
   (Mỗi PUT unit = 25 KB — record lớn hơn 25KB tốn nhiều units hơn)

3. Extended Data Retention (tùy chọn):
   7 ngày: $0.020 / shard / giờ
   365 ngày: $0.023 / shard / giờ

4. Enhanced Fan-Out (tùy chọn):
   $0.015 / consumer-shard / giờ
   $0.013 / GB data retrieved

Ví dụ thực tế:
  10 MB/s throughput = 10 shards
  Chi phí shard: 10 × $10.80 = $108/tháng
  Data: 10 MB/s × 3600 × 24 × 30 = ~25 TB/tháng
  PUT units: 25 TB / 25 KB = ~1 tỷ units = $14/tháng
  Tổng: ~$122/tháng (không Enhanced Fan-Out, không Extended Retention)
```

### MSK Provisioned — Chi Phí

```
Thành phần chi phí MSK Provisioned (us-east-1, kafka.m5.large):

1. Broker giờ:
   kafka.m5.large: ~$0.21 / broker / giờ = ~$152 / broker / tháng
   3 brokers: ~$456/tháng

2. EBS Storage:
   $0.10 / GB-tháng (gp2)
   3 brokers × 1 TB = $300/tháng

3. Data transfer (Truyền Dữ Liệu):
   $0.011 / GB (inter-AZ data transfer)

4. MSK Connect workers (nếu dùng):
   MCU (MSK Connect Unit — Đơn Vị MSK Connect): $0.11 / MCU / giờ

Chi phí tối thiểu production:
  3 × kafka.m5.large + 3 × 500 GB EBS:
  = $456 + $150 = ~$606/tháng (cơ bản)

So sánh tại 10 MB/s throughput:
  KDS: ~$122/tháng (rẻ hơn đáng kể ở low throughput)
  MSK: ~$606/tháng (overhead lớn)

So sánh tại 500 MB/s throughput:
  KDS: 500 shards × $10.80 + data = ~$5.400 + $700 = ~$6.100/tháng
  MSK: 3x kafka.m5.4xlarge (~$0.40/h) + EBS = ~$864 + $300 = ~$1.164/tháng
  → MSK tiết kiệm hơn ~80% ở high throughput!
```

### MSK Serverless — Chi Phí

```
MSK Serverless (us-east-1):

1. Cluster giờ:
   $0.75 / cluster / giờ = $540 / tháng (cố định, bất kể lượng data)

2. Storage:
   $0.10 / GB-tháng

3. Throughput:
   $0.10 / GB ingress (nhập vào)
   $0.05 / GB egress (xuất ra)

Ví dụ 10 MB/s:
  Cluster: $540/tháng
  Data in: 10 MB/s × 2.592.000 giây/tháng = ~25 TB × $0.10 = $2.500/tháng
  → Tổng ~$3.040/tháng (rất đắt cho throughput này!)

MSK Serverless phù hợp: variable workloads, dev/test, burst traffic
Không phù hợp: steady high-throughput production
```

---

## Tích Hợp AWS Native

### Kinesis Data Streams — Tích Hợp

```
Lambda Trigger (Kích Hoạt Lambda): ✅ Native, zero config
  → Tự động invoke Lambda khi có records
  → Batch size, bisect-on-error, parallelization tùy chỉnh

Kinesis Data Firehose: ✅ Native source
  → KDS → Firehose → S3/Redshift/OpenSearch

AWS Glue Streaming ETL: ✅ Native source
  → Đọc từ KDS, xử lý với Spark, ghi ra S3/JDBC

Kinesis Data Analytics (Flink/SQL): ✅ Native source
  → KDS → KDA → KDS/S3/Redshift

CloudWatch Metrics: ✅ Tự động
  → IncomingBytes, GetRecords.IteratorAgeMilliseconds, ...

EventBridge Pipes: ✅ Source
  → KDS → EventBridge → bất kỳ target
```

### MSK — Tích Hợp

```
Lambda Trigger: ✅ Có (MSK Event Source Mapping — Ánh Xạ Nguồn Sự Kiện MSK)
  → Nhưng cần VPC access, IAM hoặc SASL/SCRAM auth
  → Không seamless bằng KDS

Kinesis Data Firehose: ✅ MSK có thể là nguồn cho Firehose
  → MSK → Firehose → S3/Redshift

AWS Glue Streaming ETL: ✅ Glue 3.0+ hỗ trợ MSK source

MSK Connect (Kafka Connect): ✅ Managed Kafka connectors
  → S3 Sink Connector, JDBC Sink, Elasticsearch Sink, ...

Amazon EMR với Kafka: ✅ Spark Structured Streaming đọc MSK

CloudWatch Metrics: ✅ Tự động
  → BytesInPerSec, MessagesInPerSec, ConsumerLag, ...

AWS DMS (Database Migration Service): ✅ MSK là target
  → RDS → DMS → MSK (CDC streaming)
```

### Điểm Khác Biệt Quan Trọng

```
Kinesis:
  - Lambda integration gần như zero-config
  - Không cần VPC nếu Lambda trong public subnet
  - Firehose tích hợp trực tiếp (không cần connector)

MSK:
  - Lambda trigger cần MSK trong VPC accessible từ Lambda
  - Kafka Connect (MSK Connect) cần cấu hình worker config
  - Không tích hợp trực tiếp với Firehose (cần MSK Connect)
  - Nhưng: Kafka ecosystem rộng hơn (Debezium CDC, JDBC source, ...)
```

---

## Use Cases Thực Tế

### Chọn Kinesis Data Streams Khi

#### Use Case 1: Real-time Web Analytics (Phân Tích Web Thời Gian Thực)

```
Bài toán: Thu thập clickstream từ website, xử lý real-time cho dashboard

Architecture (Kiến Trúc):
  Browser → API Gateway → Lambda → KDS → Lambda (real-time aggregation)
                                       → Firehose → S3 (batch analysis)

Lý do chọn KDS:
  ✅ Lambda trigger native — không cần Kafka consumer code
  ✅ Firehose tích hợp sẵn cho S3 delivery
  ✅ Throughput ~10-50 MB/s — phù hợp KDS cost-effective
  ✅ Retention 24h đủ cho real-time dashboard
  ✅ Không cần Kafka ecosystem
```

#### Use Case 2: IoT Sensor Data (Dữ Liệu Cảm Biến IoT)

```
Bài toán: 10.000 thiết bị IoT gửi telemetry 1 message/giây

Architecture:
  IoT devices → AWS IoT Core → Kinesis Rule Action → KDS
  KDS → Lambda (anomaly detection) → SNS alert
     → Firehose → S3 → Athena analysis

Lý do chọn KDS:
  ✅ AWS IoT Core có native rule action gửi vào KDS
  ✅ Lambda xử lý real-time alert
  ✅ Throughput dự đoán được → dễ sizing shard
```

### Chọn MSK Khi

#### Use Case 3: Event-Driven Microservices (Microservices Dựa Trên Sự Kiện)

```
Bài toán: E-commerce platform với 20+ microservices cần event bus

Architecture:
  order-service → MSK Topic: "order-events" → inventory-service
                                             → payment-service
                                             → notification-service
                                             → analytics-service

  Mỗi service là consumer group riêng biệt

Lý do chọn MSK:
  ✅ 4+ consumer groups độc lập đọc cùng topic — KDS sẽ cần Enhanced Fan-Out đắt tiền
  ✅ Kafka ecosystem — có thể dùng Kafka Streams cho stateful processing
  ✅ Low latency <10ms — Kafka native client hiệu quả hơn
  ✅ Future-proof — không bị vendor lock-in nếu cần hybrid cloud
```

#### Use Case 4: CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu)

```
Bài toán: Streaming changes từ PostgreSQL sang data lake và search index

Architecture:
  PostgreSQL → Debezium (CDC connector) → MSK Topic: "db-changes"
  MSK → Elasticsearch (via MSK Connect Elasticsearch sink)
     → S3 (via MSK Connect S3 sink)
     → Analytics service (via Kafka consumer)

Lý do chọn MSK:
  ✅ Debezium chỉ hỗ trợ Kafka (không có KDS connector)
  ✅ MSK Connect có Elasticsearch sink connector sẵn
  ✅ Retention dài — cần replay history để rebuild index
  ✅ Kafka là tiêu chuẩn công nghiệp cho CDC
```

#### Use Case 5: High-Throughput Log Processing (Xử Lý Log Thông Lượng Cao)

```
Bài toán: 1.000 servers gửi application logs, 500 MB/s, cần phân tích real-time

Architecture:
  App servers → Fluent Bit → MSK Topic: "app-logs" → Spark Structured Streaming (EMR)
                                                    → OpenSearch (MSK Connect)
                                                    → S3 long-term storage

Lý do chọn MSK:
  ✅ 500 MB/s throughput → MSK rẻ hơn KDS ~5x
  ✅ Fluent Bit có Kafka output plugin
  ✅ Spark Structured Streaming đọc Kafka native
  ✅ MSK Connect Elasticsearch sink
```

#### Use Case 6: Hybrid Cloud / Multi-Cloud Migration

```
Bài toán: Đang có Kafka on-premises, migrate lên cloud

Architecture:
  On-premises Kafka → MirrorMaker 2 (Kafka replication tool) → MSK
  MSK → Ứng dụng trên AWS

Lý do chọn MSK:
  ✅ API tương thích 100% với Kafka — zero code change
  ✅ MirrorMaker 2 là tool Kafka native
  ✅ Có thể chạy hybrid: on-premises + MSK song song
  ✅ Dễ rollback nếu cần
```

---

## Decision Tree — Cây Quyết Định

```
Bắt đầu: Bạn cần streaming platform trên AWS

┌─ Bạn đã có Kafka workload/team expertise?
│
├─ CÓ → Dùng MSK (tránh rewrite code, tận dụng expertise)
│
└─ KHÔNG → Tiếp tục...
   │
   ├─ Cần Kafka-specific features?
   │  (Kafka Streams, Kafka Connect, KSQL, Debezium CDC)
   │
   ├─ CÓ → Dùng MSK
   │
   └─ KHÔNG → Tiếp tục...
      │
      ├─ Cần nhiều consumer groups độc lập (> 3)?
      │
      ├─ CÓ → Xem xét MSK (tránh Enhanced Fan-Out cost)
      │
      └─ KHÔNG → Tiếp tục...
         │
         ├─ Throughput > 300 MB/s liên tục?
         │
         ├─ CÓ → MSK tiết kiệm chi phí hơn
         │
         └─ KHÔNG → Tiếp tục...
            │
            ├─ Cần tích hợp sâu với AWS Lambda/Firehose?
            │
            ├─ CÓ → Kinesis Data Streams (zero-config integration)
            │
            └─ Ưu tiên đơn giản, fully managed, AWS-native?
               │
               └─ CÓ → Kinesis Data Streams
                  KHÔNG → MSK
```

### Bảng Quyết Định Nhanh

| Tình Huống                                    | Khuyến Nghị |
| --------------------------------------------- | ----------- |
| Đã có Kafka on-premises, muốn lift-and-shift  | **MSK**     |
| Cần Debezium CDC từ database                  | **MSK**     |
| Cần Kafka Streams stateful processing          | **MSK**     |
| Throughput > 300 MB/s ổn định                 | **MSK**     |
| > 3 consumer groups độc lập                   | **MSK**     |
| Cần retention > 7 ngày hoặc vô thời hạn       | **MSK**     |
| Multi-cloud / Hybrid cloud                     | **MSK**     |
| AWS-native, tích hợp Lambda/Firehose đơn giản | **KDS**     |
| Throughput < 100 MB/s, bắt đầu mới            | **KDS**     |
| IoT với AWS IoT Core                          | **KDS**     |
| Clickstream analytics đơn giản                | **KDS**     |
| Không có Kafka expertise trong team           | **KDS**     |

---

## Migration Scenarios — Kịch Bản Di Chuyển

### Migration Từ Kafka On-Premises Sang MSK

```
Chiến lược:
1. Tạo MSK cluster với cùng Kafka version
2. Cập nhật bootstrap servers trong application config
3. Dùng MirrorMaker 2 để sync dữ liệu
4. Test với consumer group mới trỏ về MSK
5. Cutover (Chuyển Đổi) dần dần

Rủi ro:
  - Message ordering trong quá trình migration
  - Offset không được sync (consumer cần reset)
  - Network latency trong thời gian dual-write
```

### Migration Từ KDS Sang MSK (Khi Scale Lớn)

```
Khi nào nên migrate:
  - KDS cost > MSK cost (thường > 300 MB/s hoặc > 50 shards)
  - Cần Kafka-specific features
  - Multi-consumer fan-out tốn phí Enhanced Fan-Out

Chiến lược:
1. Deploy MSK cluster song song
2. Dual-write: write vào cả KDS và MSK
3. Chuyển consumer sang đọc từ MSK
4. Validate correctness (kiểm tra tính đúng đắn)
5. Tắt KDS sau khi xác nhận

Khó khăn:
  - Kafka consumer code hoàn toàn khác KCL (Kinesis Client Library)
  - Phải rewrite toàn bộ consumer logic
  - Ordering semantics khác (shard key vs partition key)
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: MSK và Kinesis Data Streams khác nhau ở điểm chính nào?**

> MSK là managed Apache Kafka — bạn có toàn bộ Kafka ecosystem (Kafka Streams, Kafka Connect, consumer groups không giới hạn) nhưng vẫn cần quản lý Kafka-specific configurations như topic retention, partition số lượng, consumer group tuning. Kinesis Data Streams là fully managed service của AWS — đơn giản hơn, tích hợp sâu với Lambda/Firehose/S3, nhưng bị giới hạn bởi API độc quyền của AWS và thiếu Kafka ecosystem.

**Q: Ở throughput nào thì MSK tiết kiệm hơn KDS?**

> Tại khoảng **200-300 MB/s** trở lên, MSK thường tiết kiệm hơn KDS. Ở throughput thấp (< 100 MB/s), KDS thường rẻ hơn vì không có overhead chi phí broker instances cố định. Điểm hòa vốn phụ thuộc vào số consumer groups (Enhanced Fan-Out tốn tiền) và retention requirements.

### Câu Hỏi Nâng Cao

**Q: Tại sao multi-consumer là lợi thế của MSK so với KDS?**

> Trong KDS, tất cả consumers của một shard chia sẻ 2 MB/s read bandwidth. Với 5 consumers, mỗi consumer chỉ còn 400 KB/s — phải dùng **Enhanced Fan-Out** ($0.015/shard/giờ thêm) để mỗi consumer có 2 MB/s riêng. Trong MSK, mỗi **consumer group** có throughput đầy đủ hoàn toàn miễn phí — 10 consumer groups đọc cùng topic không ảnh hưởng lẫn nhau. Cho hệ thống microservices với nhiều service cần đọc cùng event stream, MSK tiết kiệm và đơn giản hơn nhiều.

**Q: Khi nào bạn sẽ chọn Kinesis thay vì MSK dù team đã biết Kafka?**

> Ngay cả với Kafka expertise, tôi sẽ chọn Kinesis khi: (1) **Tích hợp Lambda đơn giản là ưu tiên** — KDS Lambda trigger zero-config, không cần VPC plumbing phức tạp; (2) **AWS IoT Core** là data source — có native rule action vào KDS; (3) **Throughput thấp và cost optimization** quan trọng hơn features; (4) **Time-to-market** là ưu tiên — KDS setup nhanh hơn nhiều so với provisioning MSK cluster và cấu hình security. Trade-off là bị lock vào AWS API, nhưng đổi lại operational simplicity.

---

## 🔗 Điều Hướng

| Trước                                    | Tiếp Theo                          |
| ---------------------------------------- | ---------------------------------- |
| [1-msk-architecture.md](1-msk-architecture.md) — Kiến trúc MSK | [3-msk-security.md](3-msk-security.md) — Bảo mật MSK |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Độ Khó:** ⭐⭐ Trung Bình
