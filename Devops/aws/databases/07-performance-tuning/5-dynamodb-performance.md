# DynamoDB Performance — Hot Partitions, Throttling & DAX

> Tối ưu hiệu năng DynamoDB tập trung vào ba vấn đề cốt lõi: phân phối tải đều giữa các partition (phân vùng), xử lý throttling (giới hạn thông lượng), và sử dụng DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB) để giảm latency (độ trễ).

## 📚 Mục Lục

1. [Kiến Trúc Partition DynamoDB](#kiến-trúc-partition-dynamodb)
2. [Hot Partition — Phân Vùng Nóng](#hot-partition--phân-vùng-nóng)
3. [Throttling — Giới Hạn Thông Lượng](#throttling--giới-hạn-thông-lượng)
4. [Write Sharding — Phân Mảnh Ghi](#write-sharding--phân-mảnh-ghi)
5. [DAX — DynamoDB Accelerator](#dax--dynamodb-accelerator)
6. [Adaptive Capacity — Năng Lực Thích Nghi](#adaptive-capacity--năng-lực-thích-nghi)
7. [Monitoring & Observability](#monitoring--observability)
8. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🏗️ Kiến Trúc Partition DynamoDB

### Cách DynamoDB Phân Phối Dữ Liệu

```
DynamoDB Table: "Orders"
                    │
                    ▼
         Hash(partition_key)
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
    Partition 1  Partition 2  Partition 3
    ────────── ─────────── ────────────
    PK: "US"   PK: "VN"   PK: "SG"
    ────────── ─────────── ────────────
    Capacity:  Capacity:  Capacity:
    1000 WCU  1000 WCU  1000 WCU
```

### Giới Hạn Mỗi Partition

| Giới Hạn | Giá Trị |
|---------|--------|
| Dữ liệu tối đa | 10 GB |
| RCU (Read Capacity Units — Đơn Vị Đọc) | 3,000 RCU/giây |
| WCU (Write Capacity Units — Đơn Vị Ghi) | 1,000 WCU/giây |

**Tính số partition:**
```
Số partition = MAX(
  ceil(Table capacity / 3000 RCU hoặc 1000 WCU),
  ceil(Data size GB / 10)
)

Ví dụ: Bảng có 9,000 RCU provision và 3,000 WCU
→ Theo RCU: ceil(9,000 / 3,000) = 3 partitions
→ Theo WCU: ceil(3,000 / 1,000) = 3 partitions
→ 3 partitions, mỗi partition nhận 3,000 RCU và 1,000 WCU
```

---

## 🔥 Hot Partition — Phân Vùng Nóng

### Hot Partition Là Gì?

Hot partition xảy ra khi **phần lớn requests tập trung vào cùng một partition key**, dẫn đến partition đó bị overloaded trong khi các partitions khác idle.

```
Kịch bản xấu: Partition key = Country
                    │
                    ▼ 90% traffic
    ┌────────────────────────────────────┐
    │  Partition "VN"                    │
    │  WCU used: 980/1000 ← GẦN BÁO LỖI │
    │  Traffic: 90% của toàn bảng       │
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │  Partition "US"                    │
    │  WCU used: 50/1000 ← LÃNG PHÍ    │
    │  Traffic: 5% của toàn bảng        │
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │  Partition "SG"                    │
    │  WCU used: 50/1000 ← LÃNG PHÍ    │
    │  Traffic: 5% của toàn bảng        │
    └────────────────────────────────────┘
```

### Dấu Hiệu Hot Partition

```
Triệu chứng:
- ProvisionedThroughputExceededException mặc dù total capacity đủ
- ThrottledRequests tăng nhưng ConsumedCapacity < ProvisionedCapacity
- Một số items/users bị throttle, số khác hoạt động bình thường

Phát hiện qua CloudWatch:
1. WriteThrottleEvents > 0 mặc dù ConsumedWriteCapacityUnits thấp
2. Contributor Insights → Top Partition Keys nhận nhiều traffic nhất

Phát hiện qua CloudWatch Contributor Insights:
Settings → CloudWatch → Contributor Insights → Enable for table
→ Xem "Top N Items by consumed capacity"
→ Xem "Top N Partitions by consumed capacity"
```

### Nguyên Nhân Phổ Biến

| Nguyên Nhân | Ví Dụ | Vấn Đề |
|------------|-------|--------|
| **Low-cardinality partition key** | `status` chỉ có 3 giá trị | Traffic tập trung vào 3 partitions |
| **Time-based keys** | `date` hoặc `year-month` | Mọi người ghi vào "hôm nay" |
| **Popular items** | `product_id` của sản phẩm viral | Flash sale gây hot partition |
| **Sequential keys** | Auto-increment ID hoặc timestamps | Adjacent items cùng partition |
| **Celebrity problem** | `user_id` của Elon Musk trên Twitter | 1 user có triệu followers |

---

## ⚡ Throttling — Giới Hạn Thông Lượng

### Các Loại Throttling

| Loại | Metric | Nguyên Nhân |
|------|--------|------------|
| **Provisioned throughput exceeded** | `ProvisionedThroughputExceededException` | Vượt capacity đã provision |
| **Partition-level throttling** | `ThrottledRequests` cao nhưng capacity còn | Hot partition |
| **GSI throttling** | GSI `WriteThrottleEvents` | GSI capacity không đủ |
| **Burst capacity exhausted** | Sau khi burst capacity hết | Traffic spike kéo dài |

### Burst Capacity — Năng Lực Bùng Phát

DynamoDB tự động dự trữ burst capacity trong tối đa 5 phút:

```
Ví dụ: Table provision 1,000 WCU/giây

Burst capacity pool:
- Tích lũy capacity không dùng (tối đa 300 giây × 1,000 WCU = 300,000 WCU)
- Khi traffic spike, có thể dùng burst pool

Timeline:
Giây 1-60:   Traffic 800 WCU/s → Tích lũy 200 WCU/s × 60s = 12,000 WCU burst
Giây 61-90:  Traffic 2,000 WCU/s → Dùng burst capacity (1,000 extra × 30s = 30,000)
Giây 91+:    Burst pool cạn → Throttling bắt đầu nếu traffic > 1,000 WCU/s
```

### Xử Lý Throttling Trong Application Code

```python
# Python — Exponential Backoff với Jitter
import boto3
import time
import random
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def write_with_retry(item, max_retries=5):
    for attempt in range(max_retries):
        try:
            table.put_item(Item=item)
            return True
        except ClientError as e:
            if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
                if attempt == max_retries - 1:
                    raise  # Hết retry, ném exception

                # Exponential backoff với jitter (độ trễ ngẫu nhiên)
                # Jitter quan trọng: tránh thundering herd — hiệu ứng đàn trâu
                base_delay = 2 ** attempt  # 1, 2, 4, 8, 16 giây
                jitter = random.uniform(0, base_delay)
                sleep_time = base_delay + jitter

                print(f"Throttled, retry {attempt + 1}/{max_retries} sau {sleep_time:.2f}s")
                time.sleep(sleep_time)
            else:
                raise  # Lỗi khác, không retry
```

```python
# Xử lý BatchWriteItem với retry cho unprocessed items
def batch_write_with_retry(items):
    # BatchWriteItem nhận tối đa 25 items mỗi lần
    request_items = {
        'Orders': [{'PutRequest': {'Item': item}} for item in items]
    }

    while request_items:
        response = dynamodb.batch_write_item(RequestItems=request_items)
        unprocessed = response.get('UnprocessedItems', {})

        if not unprocessed:
            break  # Tất cả đã được ghi thành công

        # Retry unprocessed items với backoff
        request_items = unprocessed
        time.sleep(0.1 * (2 ** len(unprocessed)))
```

---

## 🔀 Write Sharding — Phân Mảnh Ghi

### Kỹ Thuật 1 — Random Suffix (Hậu Tố Ngẫu Nhiên)

Thêm số ngẫu nhiên vào partition key để phân tán traffic:

```python
import random

# Giả sử leaderboard với partition key "GLOBAL"
# BAD: Hot partition — tất cả ghi vào "GLOBAL"
table.put_item(Item={
    'pk': 'GLOBAL',
    'sk': user_id,
    'score': score
})

# GOOD: Sharding với 10 shards
shard_count = 10
shard_id = random.randint(0, shard_count - 1)
table.put_item(Item={
    'pk': f'GLOBAL#{shard_id}',  # "GLOBAL#0" đến "GLOBAL#9"
    'sk': user_id,
    'score': score
})

# Đọc: phải query tất cả shards và merge
def get_top_scores(limit=100):
    all_scores = []
    for shard_id in range(shard_count):
        response = table.query(
            KeyConditionExpression=Key('pk').eq(f'GLOBAL#{shard_id}'),
            ScanIndexForward=False,  # Giảm dần
            Limit=limit
        )
        all_scores.extend(response['Items'])

    # Sort và lấy top
    all_scores.sort(key=lambda x: x['score'], reverse=True)
    return all_scores[:limit]
```

### Kỹ Thuật 2 — Calculated Shard (Phân Mảnh Theo Tính Toán)

Dùng hash của attribute để xác định shard:

```python
import hashlib

def get_shard(user_id, shard_count=16):
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    return hash_value % shard_count

# Ghi
shard_id = get_shard(user_id)
table.put_item(Item={
    'pk': f'PROFILE#{shard_id}',  # Ổn định cho cùng user_id
    'sk': user_id,
    'data': profile_data
})

# Đọc item cụ thể (biết user_id → biết shard)
shard_id = get_shard(user_id)
response = table.get_item(Key={
    'pk': f'PROFILE#{shard_id}',
    'sk': user_id
})
```

### Kỹ Thuật 3 — Time-based Partitioning (Phân Vùng Theo Thời Gian)

Thay vì chỉ dùng ngày, thêm bucket:

```python
from datetime import datetime

# BAD: Tất cả events hôm nay vào 1 partition
table.put_item(Item={
    'pk': datetime.now().strftime('%Y-%m-%d'),  # "2024-01-15"
    'sk': event_id,
    'data': event_data
})

# GOOD: Phân tán theo giờ hoặc theo hash
bucket = hash(event_id) % 24  # 24 buckets
table.put_item(Item={
    'pk': f"{datetime.now().strftime('%Y-%m-%d')}#{bucket:02d}",  # "2024-01-15#07"
    'sk': event_id,
    'data': event_data
})

# Đọc events của 1 ngày (query 24 buckets):
def get_events_for_date(date_str):
    events = []
    for bucket in range(24):
        response = table.query(
            KeyConditionExpression=Key('pk').eq(f'{date_str}#{bucket:02d}')
        )
        events.extend(response['Items'])
    return events
```

---

## ⚡ DAX — DynamoDB Accelerator

### DAX Là Gì?

DAX (DynamoDB Accelerator — Bộ Nhớ Đệm DynamoDB) là in-memory cache (bộ nhớ đệm trong RAM) fully-managed, được tối ưu cho DynamoDB, cung cấp microsecond latency (độ trễ micro-giây) so với millisecond của DynamoDB.

```
Không có DAX:
Application ──────────────────────► DynamoDB
                              Latency: 1-10ms

Với DAX:
Application ──► DAX Cluster ──► DynamoDB
         Cache hit │           Cache miss
         ~microsecond           1-10ms + populate cache
```

### Kiến Trúc DAX

```
┌──────────────────────────────────────────────────────────────────┐
│  VPC                                                             │
│                                                                   │
│  Application          DAX Cluster               DynamoDB         │
│                                                                   │
│  ┌─────────┐         ┌────────────────┐                         │
│  │         │         │  Primary Node  │◄─────────────────────── │
│  │  App 1  ├────────►│  (Read+Write)  │    Cache miss/write     │
│  │         │         └────────┬───────┘    ──────────────────►  │
│  └─────────┘                  │ Replication                      │
│                               ▼                                  │
│  ┌─────────┐         ┌────────────────┐                         │
│  │         │         │  Read Replica  │                         │
│  │  App 2  ├────────►│  Node 1        │                         │
│  │         │         └────────────────┘                         │
│  └─────────┘                                                     │
│                      ┌────────────────┐                         │
│  ┌─────────┐         │  Read Replica  │                         │
│  │         ├────────►│  Node 2        │                         │
│  │  App 3  │         └────────────────┘                         │
│  └─────────┘                                                     │
└──────────────────────────────────────────────────────────────────┘
```

### Hai Loại Cache Trong DAX

| Cache | Lưu Gì | TTL Mặc Định | Invalidation — Vô Hiệu Hóa |
|-------|--------|-------------|---------------------------|
| **Item Cache** | Kết quả của `GetItem`, `BatchGetItem` | 5 phút | Khi item được ghi/xóa |
| **Query Cache** | Kết quả của `Query`, `Scan` | 5 phút | Không tự động — cần TTL |

**Lưu ý quan trọng về Query Cache:**
- Query Cache KHÔNG được invalidate khi dữ liệu thay đổi
- Nếu write một item → Item Cache được invalidate
- Nhưng Query Cache vẫn giữ kết quả cũ đến khi TTL hết
- Đây là **eventual consistency** (nhất quán cuối cùng) ở cấp cache

### DAX Request Flow — Luồng Request

```
GetItem Request:

1. App gửi request tới DAX endpoint (không phải DynamoDB endpoint)
2. DAX kiểm tra Item Cache:
   → Cache HIT: Trả về ngay (~microseconds), không cần tới DynamoDB
   → Cache MISS: Chuyển tiếp tới DynamoDB, lưu kết quả vào cache, trả về

Write Request (PutItem, UpdateItem, DeleteItem):
1. DAX ghi vào DynamoDB trước (synchronous)
2. DAX cập nhật/xóa Item Cache (invalidate entry tương ứng)
3. Trả về kết quả cho app

→ DAX là write-through cache (ghi xuyên suốt) cho write operations
```

### Khi Nào Nên Dùng DAX?

| Phù Hợp | Không Phù Hợp |
|---------|--------------|
| Read-heavy workloads (tải đọc nhiều) | Write-heavy workloads |
| Cùng items được đọc nhiều lần | Mỗi item chỉ đọc 1 lần |
| Cần microsecond latency | Strong consistency (nhất quán mạnh) bắt buộc |
| Ứng dụng game, social media | Financial transactions cần real-time accuracy |
| Hot item problem | Mỗi read có data khác nhau |

### Cấu Hình DAX Cluster

```python
# Python — Kết nối DAX
import amazondax
import boto3

# Tạo DAX client (thay thế hoàn toàn DynamoDB client)
dax_client = amazondax.AmazonDaxClient(
    endpoints=['myapp-dax.xxxx.dax-clusters.ap-southeast-1.amazonaws.com'],
    region_name='ap-southeast-1'
)

# Sử dụng giống DynamoDB hoàn toàn
response = dax_client.get_item(
    TableName='Orders',
    Key={
        'pk': {'S': 'ORDER#001'},
        'sk': {'S': 'METADATA'}
    },
    ConsistentRead=False  # Eventually consistent — tận dụng DAX cache
)

# Lưu ý: ConsistentRead=True → DAX bypass cache và đọc trực tiếp DynamoDB
```

**Terraform — DAX Cluster:**
```hcl
resource "aws_dax_cluster" "main" {
  cluster_name       = "myapp-dax"
  iam_role_arn       = aws_iam_role.dax.arn
  node_type          = "dax.r4.large"
  replication_factor = 3  # 1 primary + 2 replicas
  subnet_group_name  = aws_dax_subnet_group.main.name
  security_group_ids = [aws_security_group.dax.id]

  server_side_encryption {
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}

resource "aws_dax_parameter_group" "main" {
  name = "myapp-dax-params"

  parameters {
    name  = "query-ttl-millis"
    value = "60000"  # 1 phút TTL cho Query Cache
  }

  parameters {
    name  = "record-ttl-millis"
    value = "300000"  # 5 phút TTL cho Item Cache
  }
}
```

### DAX Node Types — Loại Node

| Node Type | vCPU | RAM | Network | Phù Hợp |
|-----------|------|-----|---------|---------|
| `dax.t2.small` | 1 | 2 GB | Thấp | Dev/test |
| `dax.r4.large` | 2 | 15.25 GB | Trung bình | Small production |
| `dax.r4.xlarge` | 4 | 30.5 GB | Cao | Medium production |
| `dax.r4.4xlarge` | 16 | 122 GB | Rất cao | Large production |

---

## 🔄 Adaptive Capacity — Năng Lực Thích Nghi

### Adaptive Capacity Là Gì?

AWS tự động kích hoạt Adaptive Capacity để isolate (cô lập) hot items và phân bổ thêm capacity cho chúng.

```
Không có Adaptive Capacity:
- Partition bị hot → throttle ngay khi vượt partition limit
- Dù table tổng thể còn nhiều capacity

Với Adaptive Capacity (DynamoDB Standard):
- DynamoDB phát hiện hot partition
- Tự động tăng capacity cho partition đó
- Lấy từ capacity "mượn" của các partitions ít dùng
- Không cần thay đổi gì từ phía ứng dụng

Lưu ý:
- Xử lý spike ngắn hạn
- Không thay thế partition key design tốt
- Có thể mất vài phút để activate
```

---

## 📊 Monitoring & Observability

### CloudWatch Metrics Quan Trọng

```bash
# Kiểm tra throttling
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name WriteThrottleEvents \
  --dimensions Name=TableName,Value=Orders \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 60 \
  --statistics Sum

# So sánh consumed vs provisioned
# Nếu consumed thấp nhưng throttle cao → hot partition
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --period 60 \
  --statistics Sum,Maximum
```

### Bật CloudWatch Contributor Insights

```bash
# Bật Contributor Insights để xem top partition keys
aws dynamodb enable-kinesis-streaming-destination \
  --table-name Orders \
  --stream-arn arn:aws:kinesis:...

# Hoặc qua CLI
aws dynamodb describe-contributor-insights \
  --table-name Orders

aws dynamodb update-contributor-insights \
  --table-name Orders \
  --contributor-insights-action ENABLE
```

### Alarm Quan Trọng

```bash
# Alarm khi có throttling
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-Orders-WriteThrottling" \
  --metric-name WriteThrottleEvents \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=Orders \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:...
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Partition key tốt cho DynamoDB là gì?

```
Tiêu chí của partition key tốt:

1. High cardinality (đặc thù cao):
   - Nhiều giá trị khác nhau → traffic phân tán đều
   - BAD: status (pending/active/done) — chỉ 3 giá trị
   - GOOD: user_id (UUID) — hàng triệu giá trị

2. Phân tán đều traffic:
   - Không có "popular items" chiếm đa số requests
   - BAD: product_id khi có sản phẩm viral
   - GOOD: order_id (mỗi order chỉ được xử lý 1 lần)

3. Uniform access pattern:
   - Không tạo temporal hot spots
   - BAD: date (hôm nay bị hot, hôm qua idle)
   - GOOD: UUID tạo từ timestamp + random component

4. Avoid sequential keys:
   - Auto-increment → adjacent items cùng partition
   - Timestamp → tất cả writes vào partition mới nhất

Best practices:
- Dùng UUID v4 làm partition key
- Combine: user_id#date thay vì chỉ date
- Thêm random suffix nếu cần: product_id#(hash%N)
```

### Q: DAX khác ElastiCache như thế nào khi dùng với DynamoDB?

```
DAX:
- Fully-managed, được tối ưu riêng cho DynamoDB
- API compatible: thay DAX endpoint vào code, không cần sửa gì
- Write-through cache: tự động invalidate khi write
- Item cache + Query cache tách biệt
- Chỉ dùng được với DynamoDB
- Giới hạn: không hỗ trợ strong consistent reads (đọc nhất quán mạnh)
- Chi phí: ~$0.24/node/giờ (r4.large)

ElastiCache Redis + DynamoDB:
- Cần code thêm cache layer thủ công
- Linh hoạt hơn: custom TTL, custom key design
- Hỗ trợ nhiều data structures hơn (sorted sets cho leaderboard)
- Cache invalidation: phải tự xử lý (tricky)
- Có thể dùng cho nhiều data sources
- Chi phí: thường rẻ hơn DAX cho same memory

Khi nào dùng gì:
- DAX: Read-heavy, đơn giản, không muốn manage cache logic
- ElastiCache: Cần cache phức tạp, cross-service caching, sorted sets
```

### Q: Xử lý Celebrity problem trong DynamoDB như thế nào?

```
Celebrity problem: 1 user (celebrity) có hàng triệu followers
→ Khi celebrity post, hàng triệu fans đọc cùng lúc
→ hot partition trên celebrity's post/profile

Giải pháp 1 — DAX (read-heavy):
- Cache celebrity profiles/posts với TTL dài
- Cache hit rate cao → giảm DynamoDB load

Giải pháp 2 — Write fanout (fan-out ghi):
- Khi celebrity post, ghi vào timeline của tất cả followers
- Trade-off: write amplification (khuếch đại ghi)
- Instagram/Twitter cũ dùng hybrid approach

Giải pháp 3 — Application-level cache:
- Cache celebrity posts trong ElastiCache
- CDN cache public posts
- Celebrity check: if followers > 1M → always cache

Giải pháp 4 — Partitioned timeline:
- User timeline pk = user_id + shard_number
- Phân tán reads khi fan cần aggregate tất cả shards

Trong phỏng vấn, đề xuất:
1. DAX cho read-heavy celebrity profiles
2. On-demand capacity mode cho spike không đoán trước
3. Application-level caching với ElastiCache cho popular content
```

---

**Tiếp Theo:** [6-index-optimization.md](6-index-optimization.md) — Index Design & GSI Optimization

**Cập Nhật Lần Cuối:** 2026-05-15
