# Shard Management — Quản Lý Shard và Tính Toán Capacity

> **Shard (Mảnh)** là đơn vị năng lực cơ bản của Kinesis Data Streams. Hiểu cách tính toán số shard, khi nào cần split (chia) hay merge (gộp), và làm thế nào để tránh hot shard (shard nóng) là kỹ năng quan trọng trong phỏng vấn và thực tế.

---

## 📚 Mục Lục

1. [Giới Hạn Một Shard](#giới-hạn-một-shard)
2. [Tính Toán Số Shard Cần Thiết](#tính-toán-số-shard-cần-thiết)
3. [Hot Shard Problem — Vấn Đề Shard Nóng](#hot-shard-problem--vấn-đề-shard-nóng)
4. [Split Shard — Chia Shard](#split-shard--chia-shard)
5. [Merge Shard — Gộp Shard](#merge-shard--gộp-shard)
6. [On-Demand Mode — Chế Độ Tự Động](#on-demand-mode--chế-độ-tự-động)
7. [Monitoring Metrics — Chỉ Số Giám Sát](#monitoring-metrics--chỉ-số-giám-sát)
8. [Chiến Lược Partition Key](#chiến-lược-partition-key)
9. [Chi Phí Shard](#chi-phí-shard)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Giới Hạn Một Shard

### Write Limits (Giới Hạn Ghi)

```
Mỗi Shard cho phép:
┌─────────────────────────────────────────────────┐
│  Write (Ghi):                                   │
│  - 1.000 records/giây                           │
│  - 1 MB/giây                                    │
│  → Đạt giới hạn nào trước → ProvisionedThroughput│
│    ExceededException                            │
│                                                 │
│  Read (Đọc) — Standard Consumer:               │
│  - 5 GetRecords calls/giây                      │
│  - 2 MB/giây TỔNG cho tất cả consumers         │
│                                                 │
│  Read (Đọc) — Enhanced Fan-Out Consumer:       │
│  - 2 MB/giây RIÊNG cho từng consumer           │
└─────────────────────────────────────────────────┘
```

### Ví Dụ Tính Nhanh

```
Tình huống:
- Ghi: 500 records/giây, mỗi record trung bình 1.5 KB
- Đọc: 3 consumers, mỗi consumer cần đọc toàn bộ data

Tính Write:
  Theo records: 500 records/s ÷ 1.000 = 0,5 shard
  Theo bandwidth: 500 × 1.5 KB = 750 KB/s ÷ 1.024 KB/MB = 0,73 MB/s ÷ 1 MB = 0,73 shard
  → Lấy max: cần 0,73 shard → làm tròn lên 1 shard

Tính Read (Standard Consumer):
  Mỗi shard: 2 MB/s ÷ 3 consumers = 0,67 MB/s per consumer
  Nhưng ghi là 750 KB/s ≈ 0,73 MB/s > 0,67 MB/s per consumer
  → 3 consumers không đủ bandwidth → cần 2 shards
  Với 2 shards: 2 × 2 MB/s ÷ 3 = 1,33 MB/s per consumer > 0,73 MB/s ✅

Kết luận: cần 2 shards
```

---

## Tính Toán Số Shard Cần Thiết

### Công Thức Tổng Quát

```
Số shard = MAX(
    CEIL(write_records_per_second / 1_000),
    CEIL(write_MB_per_second / 1),
    CEIL(read_MB_per_second × num_consumers / 2)     ← Standard Consumer
)

Với Enhanced Fan-Out:
Số shard = MAX(
    CEIL(write_records_per_second / 1_000),
    CEIL(write_MB_per_second / 1),
    CEIL(read_MB_per_second_per_consumer / 2)         ← mỗi consumer độc lập
)
```

### Bài Tập Tính Shard — Từng Bước

#### Tình Huống 1: IoT Data Collection

```
Yêu cầu:
- 10.000 sensors, mỗi sensor gửi 1 record/giây
- Mỗi record: 200 bytes
- 2 consumers standard (Lambda + Analytics)

Bước 1 — Write throughput:
  Records: 10.000 records/s ÷ 1.000 = 10 shards (giới hạn records)
  Bandwidth: 10.000 × 200 B = 2 MB/s ÷ 1 MB = 2 shards (giới hạn bandwidth)
  → Write cần: MAX(10, 2) = 10 shards

Bước 2 — Read throughput (Standard):
  2 MB/shard ÷ 2 consumers = 1 MB/s/consumer
  Write = 200 KB/s/shard (2 MB/s tổng ÷ 10 shards) → OK

Bước 3 — Kết luận:
  Cần 10 shards
  Throughput: 10.000 records/s = giới hạn write tổng
  
Tip: thêm 20–30% buffer → đặt 13 shards để tránh sát giới hạn
```

#### Tình Huống 2: E-commerce Clickstream

```
Yêu cầu:
- Peak traffic: 50.000 page views/phút
- Mỗi clickstream event: 800 bytes
- 4 consumers standard (Fraud, Analytics, Personalization, Logging)

Bước 1 — Write throughput:
  Records: 50.000 ÷ 60 = 833 records/s ÷ 1.000 = 0,83 shard
  Bandwidth: 833 × 800 B = 0,67 MB/s ÷ 1 = 0,67 shard
  → Write cần: MAX(0,83, 0,67) = 0,83 → 1 shard

Bước 2 — Read throughput (Standard, 4 consumers):
  Bandwidth read = 0,67 MB/s
  Per shard read capacity = 2 MB/s ÷ 4 consumers = 0,5 MB/s/consumer
  0,67 MB/s > 0,5 MB/s → KHÔNG ĐỦ với 1 shard

  Thử 2 shards:
  Read = 2 × 2 MB/s ÷ 4 = 1 MB/s/consumer
  Write = 0,67 MB/s → OK ✅

Bước 3 — Kết luận:
  Cần 2 shards
  Thêm 20% buffer → dùng 3 shards
```

### Script Tính Shard Tự Động

```python
import math

def calculate_required_shards(
    records_per_second: float,
    avg_record_size_bytes: float,
    num_standard_consumers: int = 1,
    buffer_percent: float = 0.2
) -> dict:
    """
    Tính số shard Kinesis cần thiết.
    
    Args:
        records_per_second: Số records mỗi giây (write)
        avg_record_size_bytes: Kích thước trung bình mỗi record (bytes)
        num_standard_consumers: Số consumers đọc (standard mode)
        buffer_percent: % buffer thêm để tránh sát giới hạn
    """
    SHARD_WRITE_RECORDS = 1_000    # records/s
    SHARD_WRITE_BYTES = 1_048_576  # 1 MB
    SHARD_READ_BYTES = 2_097_152   # 2 MB (shared)
    
    write_mb_per_second = records_per_second * avg_record_size_bytes
    
    # Tính shards cần cho write
    shards_by_records = math.ceil(records_per_second / SHARD_WRITE_RECORDS)
    shards_by_write_bandwidth = math.ceil(write_mb_per_second / SHARD_WRITE_BYTES)
    shards_for_write = max(shards_by_records, shards_by_write_bandwidth)
    
    # Tính shards cần cho read (standard consumers)
    # Mỗi consumer cần đọc write_mb_per_second bytes
    # Mỗi shard cung cấp SHARD_READ_BYTES ÷ num_consumers cho mỗi consumer
    if num_standard_consumers > 0:
        total_read_needed = write_mb_per_second * num_standard_consumers
        shards_for_read = math.ceil(total_read_needed / SHARD_READ_BYTES)
    else:
        shards_for_read = 0
    
    base_shards = max(shards_for_write, shards_for_read)
    recommended = math.ceil(base_shards * (1 + buffer_percent))
    
    return {
        'base_shards': base_shards,
        'recommended_shards': recommended,
        'write_throughput_mb': write_mb_per_second / 1_048_576,
        'by_record_limit': shards_by_records,
        'by_write_bandwidth': shards_by_write_bandwidth,
        'by_read_consumers': shards_for_read,
    }

# Ví dụ sử dụng
result = calculate_required_shards(
    records_per_second=5_000,
    avg_record_size_bytes=512,
    num_standard_consumers=3,
    buffer_percent=0.25
)
print(f"Số shard tối thiểu: {result['base_shards']}")
print(f"Số shard khuyến nghị (có buffer): {result['recommended_shards']}")
```

---

## Hot Shard Problem — Vấn Đề Shard Nóng

### Nguyên Nhân Hot Shard

```
Partition Key không đồng đều → một shard nhận quá nhiều traffic

Ví dụ xấu:
partitionKey = "default"           → 100% traffic vào 1 shard
partitionKey = user_country        → "VN" chiếm 80% → hot shard
partitionKey = event_type          → "click" chiếm 95% → hot shard
partitionKey = timestamp_second    → nhiều records cùng giây → hot shard

Ví dụ tốt:
partitionKey = user_id             → phân tán đều nếu user_id đa dạng
partitionKey = order_id            → UUID → phân tán tốt
partitionKey = device_id           → ID duy nhất mỗi thiết bị
```

### Phát Hiện Hot Shard

```python
import boto3

cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

def detect_hot_shards(stream_name: str):
    """
    Kiểm tra metric IncomingRecords per shard.
    Shard có traffic cao hơn 80% giới hạn → cảnh báo hot shard.
    """
    kinesis = boto3.client('kinesis')
    stream_info = kinesis.describe_stream_summary(StreamName=stream_name)
    
    # Lấy danh sách shards
    paginator = kinesis.get_paginator('list_shards')
    shards = []
    for page in paginator.paginate(StreamName=stream_name):
        shards.extend(page['Shards'])
    
    print(f"Stream: {stream_name}, Total shards: {len(shards)}")
    
    # Kiểm tra metrics từ CloudWatch
    for shard in shards:
        shard_id = shard['ShardId']
        response = cloudwatch.get_metric_statistics(
            Namespace='AWS/Kinesis',
            MetricName='IncomingRecords',
            Dimensions=[
                {'Name': 'StreamName', 'Value': stream_name},
                {'Name': 'ShardId', 'Value': shard_id}
            ],
            Period=60,
            Statistics=['Sum'],
            # Lấy 5 phút gần nhất
        )
        
        if response['Datapoints']:
            records_per_minute = response['Datapoints'][-1]['Sum']
            records_per_second = records_per_minute / 60
            utilization = records_per_second / 1_000 * 100
            
            if utilization > 80:
                print(f"⚠️  HOT SHARD: {shard_id} — {utilization:.1f}% utilization")
            else:
                print(f"✅ Normal: {shard_id} — {utilization:.1f}% utilization")
```

### Giải Pháp Hot Shard

```python
import hashlib
import random

# Giải pháp 1: Random Suffix (không cần ordering)
def get_partition_key_random(entity_id: str, num_shards: int = 10) -> str:
    """
    Thêm random suffix để phân tán records.
    Mất strict ordering — dùng khi không cần order.
    """
    suffix = random.randint(0, num_shards - 1)
    return f"{entity_id}-{suffix}"

# Giải pháp 2: Composite Key (cân bằng giữa phân tán và ordering)
def get_partition_key_composite(user_id: str, session_id: str) -> str:
    """
    Dùng composite key để phân tán đều hơn.
    Events cùng session_id vẫn vào cùng shard → ordering trong session.
    """
    return f"{user_id}#{session_id}"

# Giải pháp 3: Deterministic Bucket (phân tán có kiểm soát)
def get_partition_key_bucket(entity_id: str, num_buckets: int = 50) -> str:
    """
    Hash entity_id vào N buckets — phân tán đều, reproducible.
    Không đảm bảo ordering tuyệt đối nhưng cùng bucket → gần nhau.
    """
    hash_val = int(hashlib.md5(entity_id.encode()).hexdigest(), 16)
    bucket = hash_val % num_buckets
    return f"bucket-{bucket:04d}"
```

---

## Split Shard — Chia Shard

### Khi Nào Cần Split

```
Tín hiệu cần split shard:
- WriteProvisionedThroughputExceeded metric tăng cao
- IncomingRecords/s gần 1.000 cho một shard
- IncomingBytes/s gần 1 MB cho một shard
- Consumer ghi nhận nhiều ProvisionedThroughputExceededException
```

### Cách Split Shard

```python
def split_shard(stream_name: str, shard_id: str):
    """
    Split shard thành 2 shard con.
    Lưu ý: tối đa 1 split per second per stream.
    """
    kinesis = boto3.client('kinesis')
    
    # Lấy thông tin shard hiện tại
    response = kinesis.describe_stream_summary(StreamName=stream_name)
    
    # Lấy hash range của shard cần split
    paginator = kinesis.get_paginator('list_shards')
    target_shard = None
    for page in paginator.paginate(StreamName=stream_name):
        for shard in page['Shards']:
            if shard['ShardId'] == shard_id:
                target_shard = shard
                break
    
    if not target_shard:
        raise ValueError(f"Shard {shard_id} không tìm thấy")
    
    # Tính điểm chia giữa hash range
    start_hash = int(target_shard['HashKeyRange']['StartingHashKey'])
    end_hash = int(target_shard['HashKeyRange']['EndingHashKey'])
    mid_hash = str((start_hash + end_hash) // 2)
    
    # Thực hiện split
    response = kinesis.split_shard(
        StreamName=stream_name,
        ShardToSplit=shard_id,
        NewStartingHashKey=mid_hash
    )
    
    print(f"Split shard {shard_id} tại hash {mid_hash}")
    print(f"Đang chờ stream trở lại trạng thái ACTIVE...")
    
    # Đợi stream sẵn sàng
    waiter = kinesis.get_waiter('stream_exists')
    waiter.wait(StreamName=stream_name)
    print("Split hoàn thành!")
```

### Lưu Ý Khi Split

```
1. Stream ở trạng thái UPDATING trong ~30 giây sau split
2. Parent shard vẫn còn trong một thời gian (retention period)
   → Consumer cần đọc hết parent shard trước khi chuyển sang child shards
3. KCL (Kinesis Client Library) tự động xử lý shard splitting/merging
4. Tối đa 10 split operations/giây per account
5. Tổng số shard có giới hạn (mặc định 500/stream, có thể tăng qua quota request)
```

---

## Merge Shard — Gộp Shard

### Khi Nào Cần Merge

```
Tín hiệu cần merge:
- Traffic giảm đáng kể (ví dụ: sau peak season)
- Nhiều shard có utilization < 25%
- Muốn giảm chi phí shard-hour
```

### Cách Merge Shard

```python
def merge_adjacent_shards(stream_name: str, shard_id_1: str, shard_id_2: str):
    """
    Gộp hai shard liền kề (adjacent) thành một.
    CHÚ Ý: Chỉ merge được 2 shard có hash range liền kề.
    """
    kinesis = boto3.client('kinesis')
    
    response = kinesis.merge_shards(
        StreamName=stream_name,
        ShardToMerge=shard_id_1,
        AdjacentShardToMerge=shard_id_2
    )
    
    print(f"Đã merge {shard_id_1} và {shard_id_2}")
    print("Đang chờ hoàn tất...")

# Kiểm tra hai shard có liền kề không
def are_adjacent(shard_a: dict, shard_b: dict) -> bool:
    """Hai shard liền kề khi EndingHashKey của A = StartingHashKey của B - 1."""
    end_a = int(shard_a['HashKeyRange']['EndingHashKey'])
    start_b = int(shard_b['HashKeyRange']['StartingHashKey'])
    return end_a + 1 == start_b
```

---

## On-Demand Mode — Chế Độ Tự Động

### Provisioned vs On-Demand

```
PROVISIONED MODE (Chế Độ Dự Phòng):
✅ Chi phí dự đoán được
✅ Phù hợp traffic ổn định
✅ Kiểm soát chi phí hoàn toàn
❌ Phải tự quản lý split/merge
❌ Phải monitor và điều chỉnh thủ công
Giá: $0,015 per shard-hour

ON-DEMAND MODE (Chế Độ Theo Nhu Cầu):
✅ Tự động scale — không cần quản lý shard
✅ Phù hợp traffic biến động hoặc unpredictable
✅ Zero ops overhead
❌ Đắt hơn Provisioned ~3x ở traffic cao ổn định
❌ Ít kiểm soát hơn
Giá: $0,08 per GB ingested (thay vì shard-hour)
```

### Khi Nào Dùng On-Demand

```
Dùng On-Demand khi:
- Traffic biến động mạnh (ví dụ: bùng nổ cuối tuần, flash sale)
- Workload mới chưa biết pattern
- Team không muốn ops overhead của shard management
- Traffic thấp và không thường xuyên

Dùng Provisioned khi:
- Traffic dự đoán được và ổn định
- Cần tối ưu chi phí tối đa
- Cần kiểm soát chặt throughput
- High-volume stream (>100 MB/s)
```

```python
# Chuyển đổi giữa hai mode
kinesis = boto3.client('kinesis')

# Sang On-Demand
kinesis.update_stream_mode(
    StreamARN='arn:aws:kinesis:ap-southeast-1:123456789:stream/order-events',
    StreamModeDetails={'StreamMode': 'ON_DEMAND'}
)

# Sang Provisioned với số shard cụ thể
kinesis.update_stream_mode(
    StreamARN='arn:aws:kinesis:ap-southeast-1:123456789:stream/order-events',
    StreamModeDetails={'StreamMode': 'PROVISIONED'}
)
# Sau đó điều chỉnh số shard nếu cần
kinesis.update_shard_count(
    StreamName='order-events',
    TargetShardCount=10,
    ScalingType='UNIFORM_SCALING'
)
```

---

## Monitoring Metrics — Chỉ Số Giám Sát

### Metrics Quan Trọng Nhất

```
1. GetRecords.IteratorAgeMilliseconds (Tuổi Iterator Milliseconds)
   ─────────────────────────────────────────────────────────────
   Đo: khoảng cách giữa thời điểm record được ghi và khi consumer đọc
   
   Bình thường: < vài trăm ms
   Cảnh báo: > 1 phút → consumer đang lag (chậm hơn producer)
   Nghiêm trọng: tiến gần retention period → có nguy cơ mất data
   
   Nguyên nhân lag:
   - Consumer quá chậm (xử lý nặng)
   - Không đủ shard (throughput bị throttle)
   - Lambda timeout hoặc lỗi liên tục

2. WriteProvisionedThroughputExceeded
   ────────────────────────────────────
   Đo: số lần write bị từ chối vì vượt throughput
   Bình thường: = 0
   Nếu > 0: cần thêm shard hoặc fix hot shard

3. ReadProvisionedThroughputExceeded
   ────────────────────────────────────
   Đo: số lần read bị từ chối
   Nếu > 0: cần thêm shard hoặc dùng Enhanced Fan-Out

4. IncomingRecords / IncomingBytes
   ────────────────────────────────
   Đo: traffic thực tế đang đến
   Dùng để: tính utilization, phát hiện hot shard, dự báo scaling
```

### CloudWatch Alarm Tham Khảo

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def create_kinesis_alarms(stream_name: str, sns_alarm_arn: str):
    """Tạo các alarm cơ bản cho Kinesis stream."""
    
    # Alarm 1: Iterator Age > 5 phút → consumer đang lag
    cloudwatch.put_metric_alarm(
        AlarmName=f'kinesis-{stream_name}-iterator-age',
        AlarmDescription='Consumer lag quá lớn — có thể mất data nếu kéo dài',
        MetricName='GetRecords.IteratorAgeMilliseconds',
        Namespace='AWS/Kinesis',
        Statistic='Maximum',
        Dimensions=[{'Name': 'StreamName', 'Value': stream_name}],
        Period=300,
        EvaluationPeriods=2,
        Threshold=300_000,  # 5 phút = 300.000 ms
        ComparisonOperator='GreaterThanThreshold',
        AlarmActions=[sns_alarm_arn]
    )
    
    # Alarm 2: Write throttling
    cloudwatch.put_metric_alarm(
        AlarmName=f'kinesis-{stream_name}-write-throttle',
        AlarmDescription='Producer đang bị throttle — cần thêm shard',
        MetricName='WriteProvisionedThroughputExceeded',
        Namespace='AWS/Kinesis',
        Statistic='Sum',
        Dimensions=[{'Name': 'StreamName', 'Value': stream_name}],
        Period=60,
        EvaluationPeriods=3,
        Threshold=10,
        ComparisonOperator='GreaterThanThreshold',
        AlarmActions=[sns_alarm_arn]
    )
    
    print(f"Đã tạo alarms cho stream {stream_name}")
```

---

## Chiến Lược Partition Key

### Nguyên Tắc Chọn Partition Key

```
Nguyên tắc 1: ĐỦ ĐA DẠNG
  - Ít nhất = số shard × 10 giá trị khác nhau
  - Càng nhiều giá trị unique → phân tán càng đều
  - BAD: boolean (2 giá trị) | GOOD: UUID (vô số giá trị)

Nguyên tắc 2: PHÙ HỢP VỚI ORDERING REQUIREMENT
  - Nếu cần order trong đơn hàng → dùng order_id
  - Nếu không cần order → dùng random/UUID

Nguyên tắc 3: PHÂN TÁN ĐỀU (Uniform Distribution)
  - Tránh partition key có skewed distribution (phân phối lệch)
  - Test: đếm số records per partition key — có key nào chiếm >20%?
```

### Bảng Tham Khảo Chọn Partition Key

| Use Case | Partition Key Tốt | Lý Do |
|---|---|---|
| Order events | `order_id` (UUID) | UUID phân tán đều, ordering trong order |
| IoT sensors | `device_id` | Unique per device, ordering per device |
| User activity | `user_id` | Đủ đa dạng, ordering per user |
| Log events | `service_name + random_suffix` | Service name + random tránh hot |
| Financial transactions | `account_id` | Ordering per account quan trọng |
| Clickstream | `session_id` | Ordering trong session |

---

## Chi Phí Shard

### Cấu Trúc Chi Phí Provisioned

```
Provisioned Mode:
- Shard Hour: $0,015/shard/giờ
  → 10 shards × 24h × 30 ngày = $108/tháng
  
- PUT Payload Unit: $0,014 per 1 triệu units
  → 1 unit = 25 KB (hoặc phần nhỏ của 25KB)
  → Record 1 KB = 1 unit | Record 26 KB = 2 units
  
- Extended Retention (> 7 ngày): $0,023/shard/giờ thêm
- Long-Term Retention (> 7 ngày): cộng thêm $0,10/GB thêm mỗi tháng

On-Demand Mode:
- $0,08 per GB ingested
- $0,04 per GB retrieved (consumer read)
- Không tính shard-hour
```

### So Sánh Chi Phí Provisioned vs On-Demand

```
Ví dụ: Stream với 5 GB/ngày ingested, 2 consumers

PROVISIONED (2 shards):
  Shard: 2 × $0,015 × 24h × 30 = $21,60/tháng
  PUT: 5 GB × 30 ÷ 25KB = ~6 triệu units × $0,014 = $0,84/tháng
  Tổng: ~$22,44/tháng

ON-DEMAND:
  Ingestion: 5 GB × 30 × $0,08 = $12/tháng
  Read: 5 GB × 2 consumers × 30 × $0,04 = $12/tháng
  Tổng: ~$24/tháng

→ Provisioned rẻ hơn một chút khi traffic ổn định
→ On-Demand đáng giá hơn khi tiết kiệm ops time quan trọng hơn $1.56/tháng
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Làm thế nào để tính số shard Kinesis cho hệ thống?

**Trả lời (cấu trúc STAR):**
1. Thu thập yêu cầu: records/s, avg record size, số consumers
2. Tính write shards: `MAX(CEIL(records/s ÷ 1000), CEIL(MB/s ÷ 1))`
3. Tính read shards (Standard): cần đủ `2MB/s × num_shards ÷ num_consumers > data_rate`
4. Thêm 20–30% buffer
5. Chọn số cao nhất từ write và read requirements
6. Xem xét dùng On-Demand nếu traffic không đoán được

### Q2: Hot shard xảy ra khi nào và giải pháp?

**Trả lời:**
- **Nguyên nhân:** Partition key không đa dạng → một shard nhận nhiều traffic hơn giới hạn → `ProvisionedThroughputExceededException`
- **Phát hiện:** CloudWatch metric `IncomingRecords` per shard — nếu một shard cao bất thường
- **Giải pháp:**
  - Đổi partition key có cardinality (số lượng giá trị unique) cao hơn
  - Thêm random suffix nếu không cần ordering
  - Split shard nóng ra thành 2
  - Dùng On-Demand mode để Kinesis tự scale

### Q3: Tại sao Iterator Age là metric quan trọng nhất?

**Trả lời:**
- **Iterator Age (Tuổi Iterator)** = thời gian từ khi record được write đến khi consumer đọc
- Nếu tăng dần → consumer đang lag, không theo kịp producer
- Nếu tiến gần `retention period` (ví dụ: 24 giờ) → data sẽ bị xóa trước khi consumer đọc → **mất dữ liệu**
- Alert khi > 1 phút, khẩn cấp khi > 50% retention period

### Q4: Khi nào nên split, khi nào nên merge shard?

**Trả lời:**
- **Split:** khi `WriteProvisionedThroughputExceeded > 0`, hoặc utilization > 70%, hoặc Iterator Age tăng
- **Merge:** khi utilization < 20–25% liên tục, muốn giảm chi phí
- Lưu ý: sau split/merge, stream tạm thời UPDATING ~30s; parent shard tồn tại cho đến hết retention; KCL tự xử lý; không merge quá nhanh vì ảnh hưởng throughput

---

**Tiếp Theo:** [5-enhanced-fanout.md](./5-enhanced-fanout.md) — Enhanced Fan-Out, cơ chế đọc nhanh 2MB/s độc lập cho từng consumer.
