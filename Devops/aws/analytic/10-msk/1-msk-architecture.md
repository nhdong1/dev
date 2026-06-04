# MSK Architecture — Kiến Trúc Amazon MSK

> Hiểu sâu kiến trúc Kafka trên Amazon MSK: Brokers (Máy Chủ Môi Giới), Topics (Chủ Đề), Partitions (Phân Vùng), Consumer Groups (Nhóm Người Tiêu Thụ) và cơ chế replication (sao chép dữ liệu) đảm bảo fault tolerance (khả năng chịu lỗi).

## 📚 Mục Lục

1. [Kafka Core Concepts](#kafka-core-concepts--khái-niệm-cốt-lõi-kafka)
2. [Broker Architecture](#broker-architecture--kiến-trúc-broker)
3. [Topic & Partition Model](#topic--partition-model)
4. [Producer Internals](#producer-internals--cơ-chế-nội-tại-producer)
5. [Consumer Groups](#consumer-groups--nhóm-người-tiêu-thụ)
6. [Replication & Fault Tolerance](#replication--fault-tolerance)
7. [ZooKeeper vs KRaft Mode](#zookeeper-vs-kraft-mode)
8. [MSK Provisioned vs Serverless](#msk-provisioned-vs-serverless)
9. [Storage Architecture](#storage-architecture--kiến-trúc-lưu-trữ)
10. [Performance Tuning](#performance-tuning--tối-ưu-hiệu-suất)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kafka Core Concepts — Khái Niệm Cốt Lõi Kafka

### Mô Hình Publish-Subscribe (Xuất Bản — Đăng Ký)

Kafka là một distributed event streaming platform (nền tảng streaming sự kiện phân tán). Các thành phần cốt lõi:

```
Producer (Nhà Sản Xuất)         Kafka Cluster            Consumer (Người Tiêu Thụ)
┌──────────────────┐           ┌───────────────┐         ┌──────────────────────┐
│  Application A   │──publish──▶│    Topic X    │──read───▶  Consumer Group 1   │
│  (order-service) │           │   Partition 0  │         │  (inventory-service) │
└──────────────────┘           │   Partition 1  │         └──────────────────────┘
                               │   Partition 2  │
┌──────────────────┐           └───────────────┘         ┌──────────────────────┐
│  Application B   │──publish──▶                ──read───▶  Consumer Group 2   │
│  (payment-svc)   │                                      │  (analytics-app)    │
└──────────────────┘                                      └──────────────────────┘
```

**Điểm mạnh của mô hình này:**
- Nhiều consumer group độc lập đọc cùng topic mà không ảnh hưởng nhau
- Consumer tự quản lý offset (vị trí đọc) — có thể replay (phát lại) dữ liệu
- Decoupling (Tách Rời) hoàn toàn producer và consumer

### Các Thành Phần Chính

| Thành Phần          | Định Nghĩa                                                              |
| ------------------- | ----------------------------------------------------------------------- |
| **Broker**          | Máy chủ Kafka — lưu trữ và phục vụ messages (tin nhắn)                 |
| **Topic**           | Kênh logic để phân loại messages theo chủ đề (ví dụ: `orders`, `logs`) |
| **Partition**       | Đơn vị parallelism (song song) và ordering (thứ tự) trong topic        |
| **Offset**          | Số thứ tự tuần tự của message trong một partition                       |
| **Producer**        | Ứng dụng ghi messages vào Kafka topic                                   |
| **Consumer**        | Ứng dụng đọc messages từ Kafka topic                                    |
| **Consumer Group**  | Nhóm consumers cùng đọc một topic — mỗi partition được đọc bởi 1 consumer |
| **Leader**          | Broker chịu trách nhiệm reads/writes cho một partition                  |
| **Follower**        | Broker sao lưu dữ liệu từ Leader                                        |

---

## Broker Architecture — Kiến Trúc Broker

### Vai Trò Của Broker

Mỗi broker trong MSK cluster chạy trên một EC2 instance (máy chủ ảo) riêng biệt trong một Availability Zone (Vùng Sẵn Sàng) khác nhau:

```
┌──────────────────────────────────────────────────────────────────┐
│                     MSK Cluster (3 brokers)                       │
│                                                                   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │   Broker 1    │  │   Broker 2    │  │   Broker 3    │        │
│  │  (AZ: 1a)     │  │  (AZ: 1b)     │  │  (AZ: 1c)     │        │
│  │               │  │               │  │               │        │
│  │ Topic: orders │  │ Topic: orders │  │ Topic: orders │        │
│  │  Part-0 [L]   │  │  Part-1 [L]   │  │  Part-2 [L]   │        │
│  │  Part-1 [R]   │  │  Part-2 [R]   │  │  Part-0 [R]   │        │
│  │  Part-2 [R]   │  │  Part-0 [R]   │  │  Part-1 [R]   │        │
│  │               │  │               │  │               │        │
│  │  EBS: 1TB     │  │  EBS: 1TB     │  │  EBS: 1TB     │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
  [L] = Leader Partition (Phân Vùng Dẫn Đầu)
  [R] = Replica Partition (Phân Vùng Bản Sao)
```

### MSK Broker Instance Types (Loại Instance)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Broker Instance Types                            │
├──────────────────┬────────────┬──────────────┬──────────────────────┤
│ Type             │ vCPU       │ Memory (RAM) │ Network Bandwidth    │
├──────────────────┼────────────┼──────────────┼──────────────────────┤
│ kafka.t3.small   │ 2          │ 2 GB         │ Up to 5 Gbps         │
│ (Dev/Test only)  │            │              │ (Không dùng prod)    │
├──────────────────┼────────────┼──────────────┼──────────────────────┤
│ kafka.m5.large   │ 2          │ 8 GB         │ Up to 10 Gbps        │
│ (Small prod)     │            │              │                      │
├──────────────────┼────────────┼──────────────┼──────────────────────┤
│ kafka.m5.4xlarge │ 16         │ 64 GB        │ Up to 10 Gbps        │
│ (Medium prod)    │            │              │                      │
├──────────────────┼────────────┼──────────────┼──────────────────────┤
│ kafka.m5.24xlarge│ 96         │ 384 GB       │ 25 Gbps              │
│ (Large prod)     │            │              │                      │
└──────────────────┴────────────┴──────────────┴──────────────────────┘
```

**Quy tắc chọn instance type:**
- Dev/Test → `kafka.t3.small`
- Production nhỏ (< 100 MB/s throughput) → `kafka.m5.large`
- Production trung bình (100-500 MB/s) → `kafka.m5.4xlarge`
- High-throughput → `kafka.m5.16xlarge` hoặc `kafka.m5.24xlarge`

---

## Topic & Partition Model

### Topic (Chủ Đề)

Topic là kênh logic chứa một loại message nhất định. Ví dụ:
- `user-events` — sự kiện người dùng (đăng nhập, xem trang)
- `order-created` — đơn hàng mới được tạo
- `payment-processed` — thanh toán đã xử lý
- `application-logs` — log ứng dụng

### Partition (Phân Vùng)

Partition là đơn vị storage (lưu trữ) và parallelism (song song) thực sự của Kafka:

```
Topic: "orders" — 3 Partitions, Replication Factor = 2

Partition 0:  [msg-0] [msg-3] [msg-6] [msg-9] ...  ← offset tăng dần
Partition 1:  [msg-1] [msg-4] [msg-7] [msg-10] ...
Partition 2:  [msg-2] [msg-5] [msg-8] [msg-11] ...
```

**Đặc điểm của partition:**
- **Ordering (Thứ Tự):** Đảm bảo thứ tự trong cùng một partition, KHÔNG đảm bảo cross-partition
- **Immutable (Bất Biến):** Messages không bị xóa/sửa — append-only log
- **Offset:** Mỗi message có offset duy nhất trong partition — consumer tự lưu offset đã đọc
- **Retention (Lưu Giữ):** Dữ liệu giữ theo thời gian (time-based) hoặc dung lượng (size-based)

### Cách Chọn Số Partition

```
Công thức tham khảo:
  Số partition = max(Throughput_target / Throughput_per_partition, Số_consumer_tối_đa)

Throughput per partition (thông lượng mỗi phân vùng):
  - Write: ~10 MB/s mỗi partition
  - Read: ~30 MB/s mỗi partition (có nhiều consumer đọc)

Ví dụ thực tế:
  Target: 100 MB/s write, tối đa 20 consumer instances
  → Partitions = max(100/10, 20) = max(10, 20) = 20 partitions
```

**Lưu ý quan trọng:**
- Tăng partition dễ, GIẢM partition rất khó (phải recreate topic)
- Quá nhiều partition → overhead cho ZooKeeper/KRaft và broker memory
- Quy tắc thực tế: bắt đầu với 10-20x số broker, điều chỉnh sau

### Partition Key (Khóa Phân Vùng)

Producer gán key (khóa) cho message → Kafka dùng hash(key) % num_partitions để chọn partition:

```python
# Python producer example
producer.send(
    topic='orders',
    key=b'customer-123',   # Tất cả đơn hàng của customer-123 → cùng partition
    value=b'{"order_id": "ORD-001", "amount": 99.99}'
)
```

**Tại sao partition key quan trọng:**
- Đảm bảo ordering cho cùng một entity (ví dụ: tất cả events của 1 user đến đúng thứ tự)
- Stateful processing (xử lý có trạng thái) như aggregation theo user/device

---

## Producer Internals — Cơ Chế Nội Tại Producer

### Producer Record Flow (Luồng Record Producer)

```
Application Code
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                  Producer                            │
│                                                      │
│  1. Serialize (Tuần Tự Hóa) key & value             │
│  2. Determine partition (Xác Định Phân Vùng)         │
│     - Có key → hash(key) % num_partitions            │
│     - Không key → Round-robin hoặc sticky            │
│  3. Add to RecordBatch (Thêm Vào Lô Record)          │
│  4. Accumulator buffer (Bộ Đệm Tích Lũy)             │
└──────────────────────┬──────────────────────────────┘
                       │ Flush khi đủ batch.size
                       │ hoặc hết linger.ms timeout
                       ▼
            Broker Leader (Broker Dẫn Đầu)
                       │
                       ▼ Acknowledge (Xác Nhận)
                 acks=0: Không chờ ack
                 acks=1: Chờ leader ghi xong
                 acks=all: Chờ tất cả ISR ghi xong
```

### Cấu Hình Producer Quan Trọng

```properties
# Durability (Độ Bền)
acks=all              # Đảm bảo không mất dữ liệu — chờ tất cả in-sync replicas
min.insync.replicas=2 # Cần ít nhất 2 replicas xác nhận

# Throughput (Thông Lượng)
batch.size=65536      # Batch 64KB — tăng để tăng throughput
linger.ms=5           # Đợi 5ms để gom batch lớn hơn
compression.type=lz4  # Nén dữ liệu — giảm network và storage

# Reliability (Độ Tin Cậy)
retries=3                        # Retry khi gửi thất bại
retry.backoff.ms=100             # Đợi 100ms giữa các lần retry
delivery.timeout.ms=120000       # Timeout toàn bộ quá trình gửi

# Idempotence (Gửi Chính Xác Một Lần)
enable.idempotence=true  # Chống duplicate do retry
```

---

## Consumer Groups — Nhóm Người Tiêu Thụ

### Cơ Chế Consumer Group

Consumer Group cho phép nhiều consumer instance cùng đọc từ một topic, mỗi partition chỉ được đọc bởi **một** consumer trong group tại một thời điểm:

```
Topic: "orders" — 6 Partitions
Consumer Group: "order-processors"

  Part-0  Part-1  Part-2  Part-3  Part-4  Part-5
    │       │       │       │       │       │
    ▼       ▼       ▼       ▼       ▼       ▼
┌───────┐ ┌───────┐ ┌───────────────────────┐
│  C-1  │ │  C-2  │ │         C-3           │
│Part-0 │ │Part-1 │ │  Part-2, Part-3       │
│Part-4 │ │Part-5 │ │  (Không cân bằng!)    │
└───────┘ └───────┘ └───────────────────────┘

3 consumers, 6 partitions → Không đều (2-2-2 tốt hơn)
```

**Quy tắc về số consumer trong group:**
- **Tối ưu:** Số consumer = Số partition
- **Thừa consumer:** Consumer dư thừa nhàn rỗi (idle) — lãng phí tài nguyên
- **Thiếu consumer:** Một consumer xử lý nhiều partition — bottleneck (nút thắt cổ chai)

### Partition Rebalancing (Cân Bằng Lại Phân Vùng)

Khi consumer join/leave group, Kafka thực hiện rebalance để phân phối lại partitions:

```
Trước Rebalance:
  C-1: [Part-0, Part-1]
  C-2: [Part-2, Part-3]
  C-3: [Part-4, Part-5]

C-3 bị crash (ngừng hoạt động) → Rebalance trigger

Sau Rebalance:
  C-1: [Part-0, Part-1, Part-4]
  C-2: [Part-2, Part-3, Part-5]
  C-3: (offline)
```

**Vấn đề với rebalance:**
- Trong thời gian rebalance, không consumer nào đọc được → **stop-the-world pause** (dừng toàn bộ)
- Giải pháp: **Static Group Membership** (Thành Viên Nhóm Tĩnh) với `group.instance.id` — consumer tạm thời ngắt kết nối không trigger rebalance ngay (có `session.timeout.ms` grace period)

### Offset Management (Quản Lý Offset)

```
Topic Partition 0:
  Offset: 0  1  2  3  4  5  6  7  8  9  10  11  ...
  Data:  [A][B][C][D][E][F][G][H][I][J][K ][L ]

Consumer Group "group-1":
  Committed Offset: 7 (đã xử lý đến message G)
  Current Position: 8 (đang đọc message H)

Consumer Group "group-2":
  Committed Offset: 3 (đang đọc chậm hơn)
```

**Cấu hình offset consumer:**

```properties
# Khi consumer mới join — đọc từ đâu?
auto.offset.reset=earliest  # Đọc từ đầu topic
# auto.offset.reset=latest  # Đọc từ message mới nhất

# Tự động commit offset
enable.auto.commit=true          # Tự commit mỗi auto.commit.interval.ms
auto.commit.interval.ms=5000     # Commit mỗi 5 giây

# Hoặc manual commit (kiểm soát chính xác hơn)
enable.auto.commit=false
# Trong code: consumer.commitSync() sau khi xử lý xong
```

### Consumer Lag (Độ Trễ Tiêu Thụ)

Consumer lag = (Latest offset trong partition) - (Committed offset của consumer group)

```
Monitoring consumer lag quan trọng:
  - Lag = 0        → Consumer theo kịp producer (healthy)
  - Lag tăng dần   → Consumer xử lý chậm hơn producer → Cần scale out
  - Lag đột ngột   → Consumer bị crash hoặc xử lý lỗi → Cần investigate

Lệnh kiểm tra lag (trên MSK):
  kafka-consumer-groups.sh --bootstrap-server <broker> \
    --group <group-id> --describe
```

---

## Replication & Fault Tolerance

### Replication Factor (Hệ Số Sao Chép)

```
Topic "orders" — Replication Factor = 3

                  Broker-1  Broker-2  Broker-3
Partition 0:      Leader    Follower  Follower
Partition 1:      Follower  Leader    Follower
Partition 2:      Follower  Follower  Leader
```

**Khuyến nghị production:**
- Replication factor = **3** (AWS khuyến nghị — chịu được 1 broker lỗi)
- `min.insync.replicas` = **2** (producer acks=all phải có ít nhất 2 ISR xác nhận)

### ISR — In-Sync Replicas (Bản Sao Đồng Bộ)

ISR là tập hợp các replicas đang được đồng bộ với leader. Một replica bị loại khỏi ISR khi:
- Không gửi heartbeat (tín hiệu tim mạch) trong `replica.lag.time.max.ms` (mặc định 30s)
- Quá chậm so với leader

```
Tình huống: Broker-2 bị lag (chậm đồng bộ)

ISR ban đầu: {Broker-1(L), Broker-2, Broker-3}
ISR sau khi Broker-2 lag: {Broker-1(L), Broker-3}

Producer với acks=all:
  → Chỉ cần Broker-1 + Broker-3 xác nhận (Broker-2 không trong ISR)
```

### Leader Election (Bầu Chọn Leader)

Khi broker chứa leader partition bị lỗi:

```
Trước: Broker-1 là leader cho Partition-0
       Broker-2 và Broker-3 là followers

Broker-1 crash →

ZooKeeper/KRaft detect → Trigger leader election →
Chọn leader mới từ ISR (ưu tiên broker có lag nhỏ nhất) →
Broker-2 trở thành leader cho Partition-0

Thời gian failover (chuyển dự phòng): thường 10-30 giây
(Trong thời gian này, partition không nhận được writes)
```

---

## ZooKeeper vs KRaft Mode

### ZooKeeper Mode (Chế Độ ZooKeeper) — Legacy

```
Truyền thống Kafka dùng ZooKeeper để:
  - Lưu trữ cluster metadata (siêu dữ liệu cluster)
  - Quản lý leader election
  - Track broker registration (đăng ký broker)

Kiến trúc:
  ZooKeeper Ensemble (Tập Hợp ZooKeeper): 3 nodes
  Kafka Brokers: 3+ nodes
  → 2 hệ thống riêng biệt cần quản lý
```

### KRaft Mode (Chế Độ KRaft) — Kafka Raft Metadata

KRaft loại bỏ ZooKeeper — metadata được lưu trực tiếp trong Kafka log:

```
KRaft Architecture:
  Controller Quorum (Nhóm Controller): 3 brokers đặc biệt
  Combined Mode: Broker vừa là broker vừa là controller

  Ưu điểm:
  ✅ Không cần quản lý ZooKeeper cluster riêng
  ✅ Faster controller failover (chuyển dự phòng controller nhanh hơn)
  ✅ Hỗ trợ nhiều partition hơn (triệu partitions)
  ✅ Đơn giản hóa vận hành

  MSK hỗ trợ: Kafka 3.7+ với KRaft mode
```

---

## MSK Provisioned vs Serverless

### MSK Provisioned (Được Cung Cấp Sẵn)

```
Bạn chọn:
  - Số broker: 2, 3, ... (khuyến nghị ≥ 3 cho HA)
  - Broker instance type: kafka.m5.large, kafka.m5.4xlarge, ...
  - Storage per broker: 1 GB đến 16 TB (EBS)
  - Kafka version: 2.x, 3.x

AWS lo:
  - Provision EC2 instances
  - Cài đặt & cập nhật Kafka
  - Multi-AZ replication
  - Monitoring cơ bản (CloudWatch)

Bạn vẫn phải quản lý:
  - Topic configuration (số partition, retention, compression)
  - Consumer group tuning
  - Capacity planning (lên/xuống instance type khi cần)
```

### MSK Serverless (Không Máy Chủ)

```
Bạn không cần quan tâm đến:
  - Số broker
  - Instance type
  - Storage capacity

MSK Serverless tự động:
  - Scale throughput (mở rộng thông lượng) theo nhu cầu
  - Manage storage (quản lý lưu trữ)
  - Handle failover (xử lý chuyển dự phòng)

Giới hạn MSK Serverless:
  - Max: 200 MB/s ingress (nhập), 400 MB/s egress (xuất) per cluster
  - Không hỗ trợ tất cả Kafka configuration parameters
  - Chỉ IAM authentication (không SASL/SCRAM hay TLS mTLS)

Chi phí:
  - Tính theo dung lượng dữ liệu ($/GB)
  - Không có phí instance cố định
```

### So Sánh Provisioned vs Serverless

| Tiêu Chí                    | Provisioned                      | Serverless                       |
| --------------------------- | -------------------------------- | -------------------------------- |
| **Quản lý capacity**        | Thủ công — bạn chọn instance     | Tự động hoàn toàn                |
| **Throughput tối đa**       | Không giới hạn (theo instance)   | 200 MB/s ingress / cluster       |
| **Chi phí predictability**  | Cao (instance giờ cố định)       | Thấp (trả theo dùng)             |
| **Kafka configuration**     | Đầy đủ                           | Hạn chế                          |
| **Authentication**          | TLS, SASL/SCRAM, IAM             | Chỉ IAM                          |
| **Phù hợp cho**             | Production workload ổn định      | Variable workload, dev/test      |

---

## Storage Architecture — Kiến Trúc Lưu Trữ

### Log Segments (Phân Đoạn Log)

Mỗi partition được lưu trữ dưới dạng log segments trên EBS (Elastic Block Store):

```
Partition 0 trên disk (ổ đĩa):
  /kafka-logs/orders-0/
    ├── 00000000000000000000.log      ← Active segment (phân đoạn đang ghi)
    ├── 00000000000000000000.index    ← Offset index (chỉ mục offset)
    ├── 00000000000000000000.timeindex← Time index (chỉ mục thời gian)
    ├── 00000000001234567890.log      ← Rolled segment (phân đoạn đã đóng)
    └── 00000000001234567890.index
```

### Retention Policies (Chính Sách Lưu Giữ)

```
Theo thời gian (Time-based):
  retention.ms=604800000  # Giữ 7 ngày
  retention.ms=-1          # Giữ vĩnh viễn (cẩn thận với disk)

Theo dung lượng (Size-based):
  retention.bytes=107374182400  # Giữ tối đa 100 GB mỗi partition

Compaction (Nén — giữ message cuối cùng theo key):
  cleanup.policy=compact  # Dùng cho state store, changelog topics
  # Thay vì xóa theo thời gian, giữ lại message mới nhất của mỗi key

Kết hợp:
  cleanup.policy=compact,delete  # Vừa compact vừa xóa cũ
```

### Tiered Storage (Lưu Trữ Phân Tầng) — MSK Feature

MSK hỗ trợ tiered storage: tự động chuyển log segments cũ từ EBS sang S3 để giảm chi phí:

```
Hot data (Dữ Liệu Nóng) → EBS (đọc nhanh)
Cold data (Dữ Liệu Lạnh) → S3 (rẻ hơn ~80%)

Kích hoạt:
  remote.storage.enable=true
  local.retention.ms=86400000  # Giữ 1 ngày trên EBS, phần còn lại lên S3
  retention.ms=2592000000      # Tổng retention 30 ngày (bao gồm S3)
```

---

## Performance Tuning — Tối Ưu Hiệu Suất

### Tuning Cho High Throughput (Thông Lượng Cao)

```properties
# Broker side (cấu hình broker)
num.network.threads=8        # Threads xử lý network requests
num.io.threads=16             # Threads xử lý I/O operations
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# Producer side
batch.size=131072             # Batch 128 KB (tăng từ default 16KB)
linger.ms=10                  # Đợi 10ms để gom batch lớn
compression.type=lz4          # Nén nhanh — tốt cho throughput
buffer.memory=67108864        # Buffer 64 MB

# Consumer side
fetch.min.bytes=50000         # Chờ ít nhất 50KB trước khi trả về
fetch.max.wait.ms=500         # Hoặc chờ tối đa 500ms
max.poll.records=500          # Xử lý 500 records mỗi poll
```

### Tuning Cho Low Latency (Độ Trễ Thấp)

```properties
# Producer
linger.ms=0       # Gửi ngay, không đợi batch (giảm throughput)
acks=1            # Chỉ cần leader ack (giảm durability)
compression.type=none  # Không nén — tiết kiệm CPU

# Consumer
fetch.min.bytes=1         # Trả về ngay khi có 1 byte
fetch.max.wait.ms=0       # Không chờ đợi
```

### JVM Tuning Cho Broker

```bash
# Cấu hình JVM (Java Virtual Machine — Máy Ảo Java) cho Kafka broker
KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"  # Heap size cố định (tránh GC pause)
KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35"
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Kafka partition khác với Kafka topic thế nào?**

> Topic là kênh logic chứa messages theo chủ đề (ví dụ: "orders"). Partition là đơn vị vật lý thực sự — một topic được chia thành N partitions để đạt được parallelism. Mỗi partition là một ordered, immutable log (nhật ký có thứ tự, bất biến). Messages được phân phối vào partitions dựa trên key hash (nếu có key) hoặc round-robin (nếu không có key).

**Q: Tại sao consumer group quan trọng?**

> Consumer group cho phép horizontal scaling (mở rộng theo chiều ngang) của phía consumer. Mỗi partition chỉ được giao cho **một** consumer trong group tại một thời điểm, đảm bảo ordering và tránh duplicate processing. Đồng thời, nhiều consumer groups độc lập có thể đọc cùng topic — ví dụ: group "analytics" và group "billing" cùng đọc topic "orders" mà không ảnh hưởng nhau.

### Nâng Cao

**Q: Giải thích ISR (In-Sync Replicas) và tại sao `min.insync.replicas` quan trọng?**

> ISR là tập hợp các replicas đang theo kịp leader (trong `replica.lag.time.max.ms`). Khi producer dùng `acks=all`, chỉ cần tất cả replicas trong ISR xác nhận, không phải tất cả replicas. `min.insync.replicas=2` đặt ngưỡng tối thiểu — nếu ISR còn ít hơn 2 replicas, producer nhận `NotEnoughReplicasException` thay vì ghi vào cluster không đủ redundancy (dự phòng). Đây là cơ chế trade-off giữa availability (tính sẵn sàng) và durability (độ bền).

**Q: Consumer lag là gì và làm thế nào để giảm?**

> Consumer lag = (Latest log-end offset) - (Consumer committed offset). Lag tăng nghĩa là consumer xử lý chậm hơn producer produce. Để giảm lag:
> 1. **Scale out consumer group** — thêm consumer instances (nhưng không vượt quá số partitions)
> 2. **Tăng số partitions** — để phân phối tải rộng hơn
> 3. **Tối ưu consumer processing** — batch xử lý, async I/O
> 4. **Tăng `max.poll.records`** — xử lý nhiều records mỗi poll
> 5. **Kiểm tra downstream bottleneck** — database, API calls trong consumer logic

**Q: Rebalance gây ra vấn đề gì và cách giảm thiểu?**

> Rebalance gây **stop-the-world** — trong thời gian rebalance, tất cả consumers trong group dừng xử lý. Giải pháp:
> - **Static Group Membership:** Set `group.instance.id` — consumer ngắt kết nối tạm thời không trigger rebalance ngay; chờ hết `session.timeout.ms` mới rebalance
> - **Incremental Cooperative Rebalancing:** Kafka 2.4+ — chỉ revoke (thu hồi) partitions cần thiết, không toàn bộ
> - **Tăng `session.timeout.ms`** — giảm false rebalance do tạm thời mất heartbeat
> - **Tăng `max.poll.interval.ms`** — tránh rebalance khi consumer xử lý lâu

---

## 🔗 Điều Hướng

| Trước                              | Tiếp Theo                       |
| ---------------------------------- | -------------------------------- |
| [README.md](README.md) — Tổng quan | [2-msk-vs-kinesis.md](2-msk-vs-kinesis.md) — So sánh MSK vs Kinesis |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Độ Khó:** ⭐⭐⭐ Nâng Cao
