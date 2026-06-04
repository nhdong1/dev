# DAX — DynamoDB Accelerator (Bộ Tăng Tốc DynamoDB)

> DAX (DynamoDB Accelerator — Bộ Tăng Tốc DynamoDB) là in-memory cache (bộ nhớ đệm trong RAM) fully managed, được thiết kế đặc biệt cho DynamoDB, giảm độ trễ từ millisecond (mili-giây) xuống microsecond (micro-giây) cho read-heavy workloads.

## 📚 Mục Lục

1. [DAX Là Gì?](#dax-là-gì)
2. [Kiến Trúc DAX](#kiến-trúc-dax)
3. [Item Cache vs Query Cache](#item-cache-vs-query-cache)
4. [Write-Through Behavior](#write-through-behavior)
5. [Cấu Hình & Triển Khai](#cấu-hình--triển-khai)
6. [DAX vs ElastiCache](#dax-vs-elasticache)
7. [Khi Nào Dùng DAX](#khi-nào-dùng-dax)
8. [Giới Hạn DAX](#giới-hạn-dax)
9. [Chi Phí DAX](#chi-phí-dax)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DAX Là Gì?

DAX là caching layer (lớp bộ nhớ đệm) nằm giữa ứng dụng và DynamoDB, **transparent** (trong suốt) với code — ứng dụng chỉ cần đổi endpoint từ DynamoDB sang DAX.

```
Không có DAX:
Application ──────────────────► DynamoDB
                                 Latency: 1-10 ms

Có DAX:
Application ──► DAX Cluster ──► DynamoDB (khi cache miss)
                Cache Hit:       Cache Miss:
                Latency: ~microseconds     Latency: 1-10 ms
```

### Vấn Đề DAX Giải Quyết

```
Scenario: Product catalog với 10 triệu reads/ngày

Không có DAX:
- 10 triệu × $0.00025/read = $2,500/tháng (On-Demand)
- Mỗi read: 1-3 ms

Với DAX (cache hit rate 90%):
- 1 triệu reads đến DynamoDB (10% cache miss)
- 9 triệu reads từ DAX cache
- Chi phí DynamoDB giảm 90%
- Latency cho cached reads: ~microseconds
```

---

## Kiến Trúc DAX

### Cluster Architecture (Kiến Trúc Cụm)

```
                 ┌────────────────────────────────┐
                 │         DAX Cluster             │
                 │                                 │
  Application   │  ┌──────────┐  ┌──────────┐    │
  ─────────────►│  │  Primary │  │  Replica │    │
                │  │  Node    │  │  Node 1  │    │
                │  │ (Write   │  │(Read only│    │
                │  │ + Read)  │  │+ failover│    │
                │  └────┬─────┘  └──────────┘    │
                │       │        ┌──────────┐    │
                │       │        │  Replica │    │
                │       │        │  Node 2  │    │
                │  ┌────▼────────────────────┐   │
                │  │   Cluster Endpoint      │   │
                │  │  (Load Balancer)        │   │
                │  └─────────────────────────┘   │
                └────────────────────────────────┘
                              │
                              ▼
                       DynamoDB Table
```

### Thành Phần

| Thành Phần                    | Mô Tả                                                               |
| ----------------------------- | ------------------------------------------------------------------- |
| **Primary Node**              | Xử lý reads và writes, quản lý cluster state                        |
| **Replica Nodes**             | Chỉ xử lý reads, automatic failover nếu primary fail               |
| **Cluster Endpoint**          | DNS endpoint duy nhất cho toàn cluster                              |
| **Item Cache**                | Cache kết quả của GetItem/BatchGetItem                              |
| **Query Cache**               | Cache kết quả của Query/Scan                                        |

### Multi-AZ Deployment (Triển Khai Đa Vùng Sẵn Sàng)

```
Khuyến nghị: ít nhất 3 nodes, mỗi node ở một AZ khác nhau

AZ-a: Primary Node
AZ-b: Replica Node 1
AZ-c: Replica Node 2

→ Chịu được failure (sự cố) của 1 AZ
→ Nếu primary fail: automatic failover (<30 giây) sang replica
```

---

## Item Cache vs Query Cache

DAX có **hai loại cache** riêng biệt với TTL (Time-to-Live — Thời Gian Sống) có thể cấu hình độc lập:

### Item Cache (Bộ Nhớ Đệm Item)

```
Lưu kết quả của: GetItem, BatchGetItem
TTL mặc định: 5 phút
Invalidation: khi item được ghi (write-through)

Cache Key = Primary Key của item
Cache Value = toàn bộ item

GetItem flow:
1. App gọi GetItem(UserId=u001, OrderId=o001)
2. DAX kiểm tra Item Cache → HIT → trả về ngay (~microseconds)
                           → MISS → forward đến DynamoDB
                                  → lưu vào cache với TTL
                                  → trả về kết quả
```

### Query Cache (Bộ Nhớ Đệm Truy Vấn)

```
Lưu kết quả của: Query, Scan
TTL mặc định: 5 phút
Invalidation: KHÔNG tự động khi data thay đổi!

Cache Key = Query parameters (TableName, KeyCondition, FilterExpression, ...)
Cache Value = result set (tập kết quả)

⚠️ QUAN TRỌNG: Query Cache không bị invalidate khi items thay đổi!
→ Có thể trả về stale data (dữ liệu cũ) đến hết TTL
→ Dùng TTL ngắn hơn cho data thay đổi thường xuyên
→ Hoặc bypass DAX cho queries cần fresh data
```

### Ví Dụ TTL Configuration

```python
import amazondax

# Cấu hình TTL khác nhau cho item cache và query cache
dax_client = amazondax.AmazonDaxClient(
    endpoints=['dax-cluster.abc123.dax-clusters.us-east-1.amazonaws.com:8111'],
    request_timeout=1000,   # ms
    item_cache_ttl=300,     # 5 phút cho item cache
    query_cache_ttl=60      # 1 phút cho query cache (data thay đổi nhanh hơn)
)
```

---

## Write-Through Behavior

### Cách DAX Xử Lý Writes

```
Write Flow (Luồng Ghi):

1. App gọi PutItem/UpdateItem/DeleteItem
2. DAX forward đến DynamoDB ngay lập tức
3. DynamoDB xác nhận ghi thành công
4. DAX cập nhật Item Cache với data mới
5. DAX trả về success cho App

→ Writes luôn đi qua DynamoDB (không cache writes)
→ Item cache được cập nhật tự động sau mỗi write
→ Query cache KHÔNG được cập nhật tự động
```

### Write-Through vs Write-Around

```
DAX dùng Write-Through (Ghi Xuyên Suốt):
✅ Item cache luôn consistent (nhất quán) với DynamoDB
✅ Reads sau write ngay lập tức thấy data mới
❌ Writes phải đi qua DAX và DynamoDB → thêm latency nhỏ

Write-Around (bỏ qua cache khi ghi):
✅ Viết thẳng vào DynamoDB, không update cache
❌ Đọc sau write phải đến DynamoDB (cache miss) cho đến khi TTL hết
→ DAX không hỗ trợ Write-Around natively (phải implement ở app layer)
```

---

## Cấu Hình & Triển Khai

### Yêu Cầu Network

```
DAX phải được đặt trong VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)
Ứng dụng và DAX cluster phải trong cùng VPC và subnet
Security Group phải cho phép port 8111 (DAX) từ ứng dụng

Architecture:
┌─────────────────────────────┐
│           VPC               │
│  ┌────────────────────────┐ │
│  │   Application Subnet   │ │
│  │   ┌──────────────┐     │ │
│  │   │  EC2 / ECS   │     │ │
│  │   └──────┬───────┘     │ │
│  └──────────┼─────────────┘ │
│  ┌──────────▼─────────────┐ │
│  │    DAX Cluster Subnet  │ │
│  │   ┌──────────────┐     │ │
│  │   │  DAX Cluster │     │ │
│  │   └──────┬───────┘     │ │
│  └──────────┼─────────────┘ │
└─────────────┼───────────────┘
              ▼
         DynamoDB (AWS Managed)
```

### Minimal Code Changes (Thay Đổi Code Tối Thiểu)

```python
# Trước (dùng DynamoDB trực tiếp)
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Products')
response = table.get_item(Key={'ProductId': 'p-001'})

# Sau (dùng DAX — chỉ thay đổi client!)
import amazondax
dax = amazondax.AmazonDaxClient.resource(
    endpoints=['dax-cluster.abc123.dax-clusters.us-east-1.amazonaws.com:8111']
)
table = dax.Table('Products')
response = table.get_item(Key={'ProductId': 'p-001'})
# Code giống hệt, chỉ đổi client object

# Bypass DAX để đọc fresh data
dynamodb = boto3.resource('dynamodb')  # Direct DynamoDB client
fresh_response = dynamodb.Table('Products').get_item(
    Key={'ProductId': 'p-001'}
)
```

---

## DAX vs ElastiCache

| Tiêu Chí                      | DAX                                    | ElastiCache Redis                          |
| ----------------------------- | -------------------------------------- | ------------------------------------------ |
| **Tích Hợp**                  | Chỉ DynamoDB                           | Bất kỳ database/service nào               |
| **Code Changes**              | Minimal — chỉ đổi client              | Cần thêm cache logic trong app             |
| **Cache Key**                 | Tự động dựa trên DynamoDB request     | Developer tự quản lý cache keys            |
| **Invalidation**              | Tự động (item cache) qua write-through | Developer tự implement                     |
| **Data Structures**           | Chỉ DynamoDB items/query results       | String, Hash, List, Set, Sorted Set...     |
| **Protocols**                 | DynamoDB API                           | Redis protocol                             |
| **Use Cases**                 | DynamoDB acceleration (tăng tốc)      | Session, leaderboard, pub/sub, general cache |
| **Latency**                   | Microseconds                           | Microseconds (sub-millisecond)             |
| **Consistency**               | Write-through (item cache)            | Developer controlled                       |

### Khi Nào Dùng DAX vs ElastiCache

```
DAX phù hợp khi:
✅ Ứng dụng chỉ dùng DynamoDB
✅ Muốn caching minh bạch, không muốn thay đổi app logic
✅ Cần reduce DynamoDB costs và latency tự động
✅ Read-heavy workload với hot items

ElastiCache phù hợp khi:
✅ Cần cache cho nhiều databases hoặc services
✅ Cần advanced data structures (sorted sets cho leaderboard)
✅ Cần pub/sub messaging
✅ Session management
✅ Cần control sepenuhnya logic invalidation
```

---

## Khi Nào Dùng DAX

### Phù Hợp

```
✅ Hot keys (Khóa Nóng) — một số items được đọc rất nhiều
   Ví dụ: product catalog, configuration, popular content

✅ Read-heavy workloads (Workload Đọc Nhiều)
   Tỷ lệ read:write > 80:20

✅ Latency-sensitive applications (Ứng Dụng Nhạy Cảm Độ Trễ)
   Game servers, trading platforms, real-time APIs

✅ Burst reads (Đọc Đột Biến)
   Flash sales, viral content — DAX hấp thụ read spikes

✅ Cost reduction (Giảm Chi Phí)
   Reduce DynamoDB RCU consumption khi cache hit rate cao
```

### Không Phù Hợp

```
❌ Write-heavy workloads — DAX không giúp ích cho writes
❌ Large items > 400 KB — vượt giới hạn DynamoDB item size
❌ Strongly consistent reads bắt buộc — DAX chỉ eventually consistent
   (hoặc phải bypass DAX để read directly từ DynamoDB)
❌ Ứng dụng không trong VPC
❌ Queries cần fresh data real-time (với TTL ngắn hơn cũng không phải lý tưởng)
```

---

## Giới Hạn DAX

```
┌───────────────────────────────────────────────────────┐
│                   DAX Limits                          │
├──────────────────────────────┬────────────────────────┤
│ Nodes tối đa mỗi cluster     │ 10 nodes               │
│ Item size tối đa cache       │ 400 KB (như DynamoDB)  │
│ Query result tối đa cache    │ 1 MB                   │
│ TTL item cache               │ 0-∞ giây (mặc định 5m) │
│ TTL query cache              │ 0-∞ giây (mặc định 5m) │
│ Strongly consistent reads    │ ❌ Không hỗ trợ qua DAX│
│ Transactions qua DAX         │ ❌ Không hỗ trợ        │
│ Cross-region                 │ ❌ Không hỗ trợ        │
└──────────────────────────────┴────────────────────────┘
```

**Quan trọng:** DAX không hỗ trợ:
- `TransactGetItems` / `TransactWriteItems` — phải gọi trực tiếp DynamoDB
- `ConsistentRead=True` — DAX chỉ trả về eventually consistent data
- Lambda functions trong VPC cần cấu hình đúng subnet/security group

---

## Chi Phí DAX

### Node Types (Loại Node)

```
dax.r4.large:  2 vCPU,  13.5 GB RAM → $0.269/giờ
dax.r4.xlarge: 4 vCPU,  27 GB RAM   → $0.538/giờ
dax.r5.large:  2 vCPU,  16 GB RAM   → $0.269/giờ
dax.r5.2xlarge: 8 vCPU, 64 GB RAM   → $1.076/giờ
(Giá tham khảo us-east-1, có thể thay đổi)
```

### ROI Calculation (Tính Toán Lợi Nhuận Đầu Tư)

```
Scenario: 100 million reads/ngày, item 4 KB, eventually consistent

Không có DAX:
100M × $0.25/million RRU = $25/ngày = $750/tháng (On-Demand)

Với DAX (cache hit rate 95%):
- DynamoDB: 5M reads × $0.25/million = $1.25/ngày = $37.5/tháng
- DAX (3 × dax.r4.large): 3 × $0.269 × 24h × 30 = $580/tháng
- Tổng: $617.5/tháng

Tiết kiệm: $750 - $617.5 = $132.5/tháng (18% savings)
+ Latency cải thiện đáng kể
+ Giải quyết hot partition issues
```

---

## Câu Hỏi Phỏng Vấn

**Q: DAX là gì và khi nào nên dùng?**

A: DAX (DynamoDB Accelerator) là in-memory cache fully managed cho DynamoDB, giảm latency từ milliseconds xuống microseconds. Tích hợp transparent — chỉ cần đổi client endpoint, không cần thay đổi logic. Nên dùng khi: có hot keys (items được đọc rất nhiều), read-heavy workloads (>80% reads), cần giảm DynamoDB RCU costs, hoặc latency-sensitive applications. Không phù hợp khi cần strongly consistent reads (phải bypass DAX), write-heavy workloads, hoặc cần DynamoDB transactions.

**Q: DAX Item Cache vs Query Cache khác nhau như thế nào?**

A: Item Cache lưu kết quả của GetItem/BatchGetItem theo primary key, được tự động cập nhật (invalidate) khi item được ghi qua DAX (write-through). Query Cache lưu kết quả của Query/Scan theo query parameters, **không tự động invalidate** khi data thay đổi — chỉ expire sau TTL. Điều này có nghĩa Query Cache có thể trả về stale data (dữ liệu cũ). Giải pháp: dùng TTL ngắn hơn cho Query Cache, hoặc bypass DAX cho queries cần fresh data.

**Q: DAX so với ElastiCache Redis — khi nào chọn cái nào?**

A: DAX chuyên biệt cho DynamoDB với tích hợp transparent và write-through tự động — lý tưởng khi chỉ cần tăng tốc DynamoDB mà không muốn thêm cache logic. ElastiCache Redis linh hoạt hơn — hỗ trợ nhiều data structures (sorted sets cho leaderboard), pub/sub, session management, và cache cho bất kỳ service nào. Nếu chỉ cần accelerate DynamoDB reads → DAX. Nếu cần general-purpose cache, session management, hoặc cache nhiều services → ElastiCache.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
