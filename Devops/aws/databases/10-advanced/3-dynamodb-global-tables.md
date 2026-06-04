# 3. DynamoDB Global Tables — Bảng Toàn Cầu Đa Vùng

> DynamoDB Global Tables (Bảng Toàn Cầu DynamoDB) là giải pháp **multi-master, multi-region** (đa chủ, đa vùng) cho DynamoDB, cho phép ứng dụng **đọc và ghi từ bất kỳ AWS Region** nào với **replication tự động** và **sub-second latency** (độ trễ dưới 1 giây).

## 📚 Mục Lục

1. [Tổng Quan & Kiến Trúc](#tổng-quan--kiến-trúc)
2. [So Sánh Global Tables v1 vs v2](#so-sánh-global-tables-v1-vs-v2)
3. [Replication & Eventual Consistency](#replication--eventual-consistency)
4. [Conflict Resolution — Giải Quyết Xung Đột](#conflict-resolution--giải-quyết-xung-đột)
5. [Strongly Consistent Reads trong Global Tables](#strongly-consistent-reads-trong-global-tables)
6. [Thiết Kế Ứng Dụng Cho Global Tables](#thiết-kế-ứng-dụng-cho-global-tables)
7. [Capacity Planning — Lập Kế Hoạch Năng Lực](#capacity-planning--lập-kế-hoạch-năng-lực)
8. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan & Kiến Trúc

### Khái Niệm Cốt Lõi

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DYNAMODB GLOBAL TABLES                           │
│                                                                     │
│   US-East-1            EU-West-1           AP-Southeast-1          │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐           │
│  │  Table   │◄────────│  Table   │────────►│  Table   │           │
│  │ Replica  │─────────►  Replica │◄────────│ Replica  │           │
│  └──────────┘         └──────────┘         └──────────┘           │
│        ▲                   ▲                    ▲                   │
│        │                   │                    │                   │
│    App US             App EU              App Asia                 │
│  (reads+writes)     (reads+writes)       (reads+writes)           │
│                                                                     │
│  → Mỗi region là master (chủ) — không có primary/secondary        │
│  → Replication bi-directional (hai chiều) tự động                 │
│  → ~1 giây để replica lan truyền                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Tạo Global Table

```python
import boto3

dynamodb = boto3.client('dynamodb', region_name='us-east-1')

# Tạo table với Global Tables enabled
dynamodb.create_table(
    TableName='UserProfiles',
    KeySchema=[
        {'AttributeName': 'user_id', 'KeyType': 'HASH'},
    ],
    AttributeDefinitions=[
        {'AttributeName': 'user_id', 'AttributeType': 'S'},
    ],
    BillingMode='PAY_PER_REQUEST',
    # Chỉ định replication regions
    ReplicationGroup=[
        {'RegionName': 'us-east-1'},
        {'RegionName': 'eu-west-1'},
        {'RegionName': 'ap-southeast-1'},
    ]
)
```

### Thêm Region Vào Global Table Đã Có

```bash
aws dynamodb update-table \
  --table-name UserProfiles \
  --replica-updates '[{"Create": {"RegionName": "sa-east-1"}}]' \
  --region us-east-1
```

---

## So Sánh Global Tables v1 vs v2

| Tiêu Chí | Global Tables v1 (2017) | Global Tables v2 (2019) — Khuyên Dùng |
|----------|------------------------|----------------------------------------|
| **Tạo table** | Phải tạo table riêng ở mỗi region | Chỉ tạo 1 lần, chọn regions |
| **Streams** | Phải bật DynamoDB Streams thủ công | Tự động quản lý |
| **Capacity** | Phải cấu hình mỗi region riêng | Quản lý tập trung |
| **Khả năng mở rộng** | Thêm region phức tạp | Thêm region dễ dàng |
| **On-demand mode** | Không hỗ trợ | Hỗ trợ đầy đủ |
| **Terraform** | `aws_dynamodb_global_table` | `aws_dynamodb_table` với `replica` blocks |

---

## Replication & Eventual Consistency

### Cơ Chế Replication

Global Tables dùng **DynamoDB Streams** (Luồng Dữ Liệu DynamoDB) để replication:

```
1. App ghi vào Region A
   PUT Item: {user_id: "alice", status: "active", updated_at: "2024-01-01T10:00:00Z"}

2. DynamoDB Streams ghi lại thay đổi
   [Stream record: NEW_IMAGE của item]

3. Replication process đọc stream và apply vào Region B, C
   ↓
   (khoảng 100ms - 1 giây, thường dưới 500ms)

4. Region B, C có item mới nhất
```

### Eventual Consistency (Nhất Quán Cuối Cùng)

**Vấn đề:** Sau khi ghi vào Region A, trong ~500ms, Region B chưa có data mới:

```python
# Region A: Ghi
dynamodb_us.put_item(
    TableName='UserProfiles',
    Item={'user_id': {'S': 'alice'}, 'status': {'S': 'premium'}}
)

# Ngay sau đó, Region B: Đọc
response = dynamodb_eu.get_item(
    TableName='UserProfiles',
    Key={'user_id': {'S': 'alice'}}
)
# → Có thể trả về 'status': 'free' (dữ liệu cũ) vì replication chưa đến
```

**Giải pháp thiết kế:**
- Nếu cần read-your-own-writes: **Đọc và ghi trong cùng 1 region**
- Nếu chấp nhận eventual consistency: Thiết kế ứng dụng phù hợp

---

## Conflict Resolution — Giải Quyết Xung Đột

### Last-Write-Wins (Ghi Sau Thắng)

DynamoDB Global Tables dùng **Last-Write-Wins** (LWW) dựa trên **timestamp** để giải quyết conflict (xung đột):

```
Scenario: Hai regions ghi vào cùng item trong cùng ~1 giây

Region US:   PUT {user_id: "alice", plan: "premium", timestamp: T1}
Region EU:   PUT {user_id: "alice", plan: "enterprise", timestamp: T2}

Nếu T2 > T1 → "enterprise" thắng, "premium" bị ghi đè
Nếu T1 > T2 → "premium" thắng, "enterprise" bị ghi đè
```

### Vấn Đề Với Last-Write-Wins

Clock skew (Lệch Đồng Hồ) giữa các servers có thể gây kết quả không mong muốn:

```
Server US:  timestamp 10:00:00.100 UTC
Server EU:  timestamp 10:00:00.050 UTC (đồng hồ lệch 50ms)

US ghi sau nhưng có timestamp thấp hơn → EU "thắng" mặc dù US ghi sau
```

**Giải pháp:** DynamoDB dùng **hybrid logical clocks** (đồng hồ logic kết hợp) để giảm thiểu clock skew issues.

### Patterns Tránh Conflict

#### Pattern 1: Partition By Region (Phân Vùng Theo Khu Vực)

```python
# Mỗi user chỉ được ghi ở 1 region cố định (home region)
def get_home_region(user_id: str) -> str:
    # Hash user_id để xác định region
    hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    regions = ['us-east-1', 'eu-west-1', 'ap-southeast-1']
    return regions[hash_val % len(regions)]

# Ghi luôn về home region
home_region = get_home_region('alice')
dynamodb_client = boto3.client('dynamodb', region_name=home_region)
dynamodb_client.put_item(...)
```

#### Pattern 2: Conditional Writes (Ghi Có Điều Kiện)

```python
# Chỉ ghi nếu version không đổi (optimistic locking — khóa lạc quan)
try:
    dynamodb.put_item(
        TableName='UserProfiles',
        Item={
            'user_id': {'S': 'alice'},
            'plan': {'S': 'premium'},
            'version': {'N': '2'}   # Version mới
        },
        ConditionExpression='version = :expected_version',
        ExpressionAttributeValues={':expected_version': {'N': '1'}}
    )
except ConditionalCheckFailedException:
    # Item đã bị thay đổi bởi write khác → retry
    retry_update()
```

#### Pattern 3: Append-Only Writes (Chỉ Thêm Mới)

```python
# Thay vì UPDATE item, dùng timestamp-keyed items (không thể conflict)
dynamodb.put_item(
    TableName='UserEvents',
    Item={
        'user_id':    {'S': 'alice'},
        'event_time': {'S': '2024-01-01T10:00:00Z'},  # Sort key
        'event_type': {'S': 'plan_upgrade'},
        'new_plan':   {'S': 'premium'}
    }
)
# Mỗi event là unique → không có conflict
```

---

## Strongly Consistent Reads trong Global Tables

**Mặc định:** Reads trong Global Tables là **eventually consistent** (nhất quán cuối cùng).

**Strongly consistent reads** (đọc nhất quán mạnh) chỉ khả dụng **trong cùng region** nơi item được ghi:

```python
# Strongly consistent read — chỉ đọc từ region đã ghi vào
response = dynamodb.get_item(
    TableName='UserProfiles',
    Key={'user_id': {'S': 'alice'}},
    ConsistentRead=True  # Chỉ có giá trị nếu đọc từ region vừa ghi
)
```

**Lưu ý:** `ConsistentRead=True` ở secondary region (chưa nhận replica) vẫn có thể trả về dữ liệu cũ nếu replication chưa hoàn tất.

---

## Thiết Kế Ứng Dụng Cho Global Tables

### Nguyên Tắc Thiết Kế

```
1. "Own your data" — Mỗi entity thuộc về 1 region chính để viết
2. Dùng conditional writes với version attribute để detect conflicts
3. Thiết kế cho eventual consistency — không assume strong consistency cross-region
4. Monitor ReplicationLatency — nếu cao, cần điều tra
5. Dùng TTL (Time to Live — Thời Gian Sống) để tự động xóa dữ liệu hết hạn
```

### Session Management Pattern

```python
class GlobalSessionStore:
    def __init__(self):
        # Mỗi region có client riêng
        self.clients = {
            'us-east-1': boto3.resource('dynamodb', region_name='us-east-1'),
            'eu-west-1': boto3.resource('dynamodb', region_name='eu-west-1'),
        }
    
    def create_session(self, session_id: str, user_id: str, home_region: str):
        # Ghi session vào home region của user
        table = self.clients[home_region].Table('Sessions')
        table.put_item(Item={
            'session_id': session_id,
            'user_id':    user_id,
            'region':     home_region,
            'created_at': datetime.utcnow().isoformat(),
            'ttl':        int(time.time()) + 3600  # TTL 1 giờ
        })
    
    def validate_session(self, session_id: str, current_region: str):
        # Đọc từ local region (eventually consistent, chấp nhận được cho session)
        table = self.clients[current_region].Table('Sessions')
        response = table.get_item(Key={'session_id': session_id})
        return response.get('Item')
```

### TTL (Time to Live) & Global Tables

TTL hoạt động **độc lập** ở mỗi region — không đồng bộ TTL deletion:

```
Region US: Item TTL expired → xóa sau vài phút
Region EU: Item vẫn tồn tại cho đến khi TTL deletion xảy ra ở EU

→ Thời gian xóa có thể chênh nhau vài phút giữa các regions
→ Không nên dùng TTL để xóa data nhạy cảm ngay lập tức
```

---

## Capacity Planning — Lập Kế Hoạch Năng Lực

### Provisioned Capacity trong Global Tables

```
Nếu dùng Provisioned mode:
  - Mỗi region cần capacity riêng
  - AWS không tự đồng bộ capacity giữa regions

Best practice: Dùng Auto Scaling (Tự Động Co Giãn) ở mỗi region
Hoặc: Dùng On-Demand mode để tránh quản lý phức tạp
```

### Replication Cost (Chi Phí Sao Chép)

Global Tables tính phí thêm cho replication:

```
Chi phí = (Chi phí ghi thông thường) × (Số regions - 1)

Ví dụ: Ghi 1 item vào 3 regions
  → 1 WCU (Write Capacity Unit — Đơn Vị Năng Lực Ghi) ghi thực tế
  → 2 WCU replication (sang EU và AP)
  → Tổng: 3 WCU tương đương
```

---

## Monitoring & Troubleshooting

### CloudWatch Metrics Quan Trọng

```
ReplicationLatency       — Độ trễ sao chép giữa regions (mục tiêu < 1000ms)
PendingReplicationCount  — Số items chờ replication
SystemErrors             — Lỗi hệ thống trong replication
```

### Alarm Cho Replication Lag

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "GlobalTable-ReplicationLatency-High" \
  --namespace "AWS/DynamoDB" \
  --metric-name "ReplicationLatency" \
  --dimensions Name=TableName,Value=UserProfiles \
               Name=ReceivingRegion,Value=eu-west-1 \
  --period 60 \
  --threshold 5000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts
```

### Troubleshooting Replication Issues

```
Triệu chứng: ReplicationLatency cao bất thường
→ Kiểm tra: PendingReplicationCount có tăng không?
→ Kiểm tra: Throttling ở region nguồn hoặc đích?
→ Kiểm tra: Có hot partition (phân vùng nóng) không?
→ Giải pháp: Scale up capacity, fix hot partition, review access patterns
```

---

## Câu Hỏi Phỏng Vấn

### Q1: DynamoDB Global Tables giải quyết conflict như thế nào?

> **Trả lời:** Dùng **Last-Write-Wins** (LWW) dựa trên timestamp. Khi hai regions ghi vào cùng item trong ~1 giây, item có timestamp lớn hơn (ghi sau) sẽ thắng. DynamoDB dùng hybrid logical clocks để giảm ảnh hưởng của clock skew. Để tránh conflict, có thể dùng: (1) partition-by-region — mỗi entity có home region để ghi, (2) conditional writes với version attribute, (3) append-only pattern cho event data.

### Q2: Strongly consistent read hoạt động thế nào với Global Tables?

> **Trả lời:** `ConsistentRead=True` chỉ đảm bảo consistency **trong cùng region**. Đọc strongly consistent ở Region EU không đảm bảo thấy data vừa ghi ở Region US. Nếu cần read-your-own-writes, phải đọc từ cùng region vừa ghi vào.

### Q3: Global Tables khác gì Aurora Global Database?

> **Trả lời:**
> - **DynamoDB Global Tables:** Multi-master (tất cả regions đều ghi), NoSQL key-value, eventual consistency, last-write-wins conflict resolution
> - **Aurora Global Database:** Single-master (chỉ 1 region ghi), SQL quan hệ, strongly consistent ở primary, secondary chỉ đọc (trừ Write Forwarding)
>
> Chọn Global Tables khi cần true active-active với NoSQL; chọn Aurora Global khi cần SQL với một primary ghi.

### Q4: Giải thích chi phí của Global Tables

> **Trả lời:** Chi phí ghi nhân với số regions. Ghi 1 item vào table có 3 regions tốn ~3 WCU (1 ghi thực tế + 2 replication). Đọc vẫn là 1 RCU (Read Capacity Unit — Đơn Vị Năng Lực Đọc). Nên dùng On-Demand mode để tránh quản lý phức tạp và cho variable workload.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
