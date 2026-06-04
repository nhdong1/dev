# CQRS — Phân Tách Lệnh Và Truy Vấn

> CQRS — Command Query Responsibility Segregation (Phân Tách Trách Nhiệm Lệnh và Truy Vấn) — tách biệt model xử lý write (ghi) và model xử lý read (đọc), cho phép tối ưu độc lập từng phía.

---

## 📚 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#vấn-đề-cần-giải-quyết)
2. [CQRS Là Gì?](#cqrs-là-gì)
3. [Kiến Trúc CQRS](#kiến-trúc-cqrs)
4. [CQRS + Event Sourcing](#cqrs--event-sourcing)
5. [Triển Khai Trên AWS](#triển-khai-trên-aws)
6. [Eventual Consistency Trong CQRS](#eventual-consistency-trong-cqrs)
7. [Ví Dụ Code Thực Tế](#ví-dụ-code-thực-tế)
8. [Khi Nào Dùng và Không Dùng CQRS](#khi-nào-dùng-và-không-dùng-cqrs)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề Cần Giải Quyết

### Cùng Một Model Cho Read Và Write

Trong hệ thống truyền thống, cùng một domain model (mô hình miền) phục vụ cả read và write:

```
Vấn đề:
┌─────────────────────────────────────────────────────────────┐
│                   OrderRepository                            │
│                                                              │
│ save(order)                    ← Write: business logic phức  │
│ update(order)                    tạp, validation, rules      │
│                                                              │
│ findById(id)                   ← Read: cần JOIN nhiều bảng,  │
│ findByCustomerId(id)             aggregation, filtering       │
│ getOrdersWithItems()             performance khác hẳn        │
│ getDashboardStats()                                          │
└─────────────────────────────────────────────────────────────┘
         ↓
    Single Database (Một Cơ Sở Dữ Liệu)
```

**Xung đột về nhu cầu:**

| | Write (Ghi) | Read (Đọc) |
|---|---|---|
| **Ưu tiên** | Consistency (nhất quán), integrity (toàn vẹn) | Performance, flexibility |
| **Schema** | Normalized (chuẩn hóa) — giảm redundancy | Denormalized (phi chuẩn hóa) — optimize query |
| **Index** | Ít index → ghi nhanh | Nhiều index → đọc nhanh |
| **Scale** | Scale up hoặc read replica | Scale out, cache, CDN |
| **Transaction** | ACID transaction cần thiết | Eventual consistency chấp nhận được |
| **Load** | ~10–20% traffic | ~80–90% traffic |

---

## CQRS Là Gì?

**CQRS** tách hệ thống thành hai luồng riêng biệt:

```
Command (Lệnh):
  - Thay đổi state (thêm, sửa, xóa)
  - Không trả về data
  - Phải qua business rules, validation
  - Ví dụ: PlaceOrder, CancelOrder, UpdateQuantity

Query (Truy Vấn):
  - Đọc state, không thay đổi gì
  - Trả về data (DTO, view model)
  - Không có side effects (tác dụng phụ)
  - Ví dụ: GetOrderById, ListOrdersByCustomer, GetDashboardStats
```

### Nguyên Tắc CQS (Command Query Separation — Phân Tách Lệnh Truy Vấn)

CQRS mở rộng nguyên tắc CQS của Bertrand Meyer:

```
CQS (cấp độ method — hàm):
  - Command method: thay đổi state, trả về void
  - Query method: trả về data, không thay đổi state
  - Một method không được vừa thay đổi state vừa trả về data

CQRS (cấp độ hệ thống — system):
  - Tách hẳn thành 2 model và 2 data store (kho dữ liệu)
  - Command Side và Query Side
```

---

## Kiến Trúc CQRS

### CQRS Đơn Giản (Same Database)

```
                    ┌──────────────────────────────┐
Client Request      │         API Gateway           │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │        Command Handler         │
                    │   (Xử Lý Lệnh)                │
                    │                               │
                    │  - Validate (kiểm tra)        │
                    │  - Apply business rules        │
                    │  - Update write model          │
                    └──────────┬───────────────────┘
                               │ write
                    ┌──────────▼───────────────────┐
                    │     Write Database            │
                    │  (normalized schema)          │
                    └──────────┬───────────────────┘
                               │
                    Synchronous update (đồng bộ)
                    hoặc triggers
                               │
                    ┌──────────▼───────────────────┐
                    │     Read Database             │
                    │  (denormalized, optimized)    │
                    └──────────┬───────────────────┘
                               │ read
                    ┌──────────▼───────────────────┐
                    │        Query Handler          │
                    │   (Xử Lý Truy Vấn)           │
                    │                               │
                    │  - No business logic          │
                    │  - Return DTO directly        │
                    └──────────────────────────────┘
```

### CQRS Với Separate Store (Kho Riêng Biệt)

```
Command Side:                    Query Side:
                                 
Client ──► Command Bus ──► Write   Events ──► Event Handler ──► Read Store
                            Model  ───────►                      (optimized)
                            │       (publish)                         │
                         Write DB                           Client ──► Query Handler
                                                                       │
                                                               Return View Model (DTO)
```

---

## CQRS + Event Sourcing

CQRS kết hợp Event Sourcing là pattern phổ biến nhất:

```
Write Side (Command):
  1. Nhận Command
  2. Load Aggregate từ Event Store (replay events)
  3. Apply business logic → tạo new events
  4. Append new events vào Event Store

                    Event Store
                    (append-only)
                         │
                   DynamoDB Streams
                   / Kinesis Stream
                         │
                    ┌────┴────┐
                    │         │
               Projector 1  Projector 2
                    │         │
               Read Model   Read Model
               (DynamoDB)  (OpenSearch)

Read Side (Query):
  1. Nhận Query
  2. Query trực tiếp từ Read Model (đã được tối ưu)
  3. Trả về DTO
```

### Flow Chi Tiết

```
1. User gửi: POST /orders {"items": [...]}
   → Command: PlaceOrderCommand {items, customerId}

2. Command Handler:
   → Load Order Aggregate từ Event Store
   → Validate: customer tồn tại, items có hàng không?
   → Tạo events: [OrderCreated, InventoryReserved]
   → Append vào Event Store

3. Event Store emits (phát ra) events qua DynamoDB Streams

4. Projector Lambda nhận events:
   → Cập nhật Read Model trong DynamoDB (denormalized)
   → Cập nhật OpenSearch index (full-text search)
   → Cập nhật Redis cache (hot data)

5. User query: GET /orders/123
   → Query Handler đọc trực tiếp từ DynamoDB Read Model
   → Trả về OrderDetailsDTO (đã pre-computed)
```

---

## Triển Khai Trên AWS

### Option 1: DynamoDB Write + DynamoDB Read (Hai Bảng)

```
Write Table: orders-write
  PK: order_id
  Data: business data, normalized
  Indexes: ít

Read Table: orders-read
  PK: order_id
  SK: có thể thêm các sort key khác
  Data: denormalized, includes customer info, item details
  GSI (Global Secondary Index): customer_id, status, date

DynamoDB Streams → Lambda (Projector) → cập nhật orders-read
```

### Option 2: DynamoDB Write + OpenSearch Read

```
Write: DynamoDB (source of truth — nguồn sự thật)
Read: OpenSearch (full-text search, complex filtering)

DynamoDB Streams → Lambda → OpenSearch index

Query types:
  - GET /orders/123         → DynamoDB (by PK, fast)
  - GET /orders?search=...  → OpenSearch (full-text)
  - GET /orders?status=...&from=...&to=... → OpenSearch (filters)
```

```python
# Lambda Projector: DynamoDB Streams → OpenSearch

from opensearchpy import OpenSearch

opensearch = OpenSearch(
    hosts=[{'host': 'search-endpoint.us-east-1.es.amazonaws.com', 'port': 443}]
)

def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] in ['INSERT', 'MODIFY']:
            new_image = record['dynamodb']['NewImage']

            # Denormalize: thêm customer info, computed fields
            doc = {
                'order_id': new_image['order_id']['S'],
                'customer_id': new_image['customer_id']['S'],
                'status': new_image['status']['S'],
                'total_amount': float(new_image['total_amount']['N']),
                'created_at': new_image['created_at']['S'],
                # Fetch customer name từ customer service (cache)
                'customer_name': get_customer_name(new_image['customer_id']['S']),
                # Compute (tính toán) item count
                'item_count': len(json.loads(new_image['items']['S']))
            }

            opensearch.index(
                index='orders',
                id=doc['order_id'],
                body=doc
            )
```

### Option 3: RDS Write + ElastiCache Read

```
Write: RDS PostgreSQL (ACID, transactions)
Read: ElastiCache Redis (caching hot queries)

Application: sau mỗi write, invalidate (xóa) Redis cache
Hoặc: Write-through cache — update Redis và DB cùng lúc
```

---

## Eventual Consistency Trong CQRS

### Vấn Đề Read-Your-Write

```
Kịch bản:
1. User A đặt order → Command thành công
2. Projector chưa chạy xong (đang async)
3. User A query GET /orders → chưa thấy order mới ← Vấn đề!

Giải pháp:
1. Optimistic UI (Giao Diện Lạc Quan): 
   Frontend hiển thị ngay order mới trước khi có response
   → User experience tốt hơn

2. Version-based read (Đọc Dựa Trên Phiên Bản):
   Command trả về version_number
   Query thêm: ?min_version=5
   Handler chờ đến khi read model đạt version 5

3. Synchronous projection cho critical paths (đường dẫn quan trọng):
   Một số projection update đồng bộ trong write transaction

4. Accept eventual consistency (Chấp nhận nhất quán cuối cùng):
   Với nhiều use case, độ trễ vài giây là chấp nhận được
```

### Lag Monitoring (Giám Sát Độ Trễ)

```python
# CloudWatch metric: projection lag (độ trễ chiếu)
cloudwatch = boto3.client('cloudwatch')

def track_projection_lag(event_timestamp: str):
    event_time = datetime.fromisoformat(event_timestamp)
    lag_ms = (datetime.utcnow() - event_time).total_seconds() * 1000
    
    cloudwatch.put_metric_data(
        Namespace='CQRS/Projections',
        MetricData=[{
            'MetricName': 'ProjectionLagMs',
            'Value': lag_ms,
            'Unit': 'Milliseconds'
        }]
    )
```

---

## Ví Dụ Code Thực Tế

### Command Handler

```python
class PlaceOrderCommandHandler:
    def __init__(self, order_repo, inventory_service, event_publisher):
        self.order_repo = order_repo
        self.inventory = inventory_service
        self.publisher = event_publisher

    def handle(self, command: dict) -> str:
        customer_id = command['customer_id']
        items = command['items']

        # 1. Validation (kiểm tra)
        for item in items:
            if not self.inventory.check_availability(item['product_id'], item['qty']):
                raise Exception(f"Product {item['product_id']} out of stock")

        # 2. Tạo Order aggregate
        order_id = str(uuid.uuid4())
        total = sum(item['price'] * item['qty'] for item in items)

        # 3. Lưu vào write model
        self.order_repo.save({
            'order_id': order_id,
            'customer_id': customer_id,
            'items': items,
            'total_amount': total,
            'status': 'CREATED',
            'created_at': datetime.utcnow().isoformat()
        })

        # 4. Publish event (Projector sẽ cập nhật read model)
        self.publisher.publish('order.created', {
            'order_id': order_id,
            'customer_id': customer_id,
            'total_amount': total
        })

        # 5. Trả về ID, KHÔNG trả về toàn bộ order
        return order_id
```

### Query Handler

```python
class OrderQueryHandler:
    def __init__(self, read_store):
        self.store = read_store  # DynamoDB read table hoặc OpenSearch

    def get_order_details(self, order_id: str) -> dict:
        """Đọc từ read model — không có business logic."""
        item = self.store.get_item(Key={'order_id': order_id})
        if 'Item' not in item:
            return None
        return self._to_dto(item['Item'])

    def list_orders_by_customer(self, customer_id: str, 
                                 page: int = 1, 
                                 size: int = 20) -> list:
        """Query từ GSI (Global Secondary Index) đã được tối ưu cho read."""
        response = self.store.query(
            IndexName='customer-orders-index',
            KeyConditionExpression='customer_id = :cid',
            ExpressionAttributeValues={':cid': customer_id},
            ScanIndexForward=False,  # Mới nhất trước
            Limit=size
        )
        return [self._to_dto(item) for item in response['Items']]

    def _to_dto(self, item: dict) -> dict:
        """Chuyển đổi từ raw DB record sang DTO (Data Transfer Object)."""
        return {
            'order_id': item['order_id'],
            'status': item['status'],
            'total_amount': float(item['total_amount']),
            'item_count': item.get('item_count', 0),
            'customer_name': item.get('customer_name', ''),
            'created_at': item['created_at']
        }
```

### API Layer

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/orders', methods=['POST'])
def create_order():
    """Command endpoint — chỉ nhận command, trả về ID."""
    command = request.json
    order_id = place_order_handler.handle(command)
    
    # KHÔNG trả về toàn bộ order — chỉ trả về ID và location
    return jsonify({'order_id': order_id}), 202  # 202 Accepted

@app.route('/orders/<order_id>', methods=['GET'])
def get_order(order_id):
    """Query endpoint — đọc từ read model."""
    order = query_handler.get_order_details(order_id)
    if not order:
        return jsonify({'error': 'Not found'}), 404
    return jsonify(order), 200

@app.route('/orders', methods=['GET'])
def list_orders():
    """Query endpoint — list với filter."""
    customer_id = request.args.get('customer_id')
    orders = query_handler.list_orders_by_customer(customer_id)
    return jsonify(orders), 200
```

---

## Khi Nào Dùng Và Không Dùng CQRS

### Nên Dùng CQRS Khi

```
✅ Read và write có nhu cầu scale (mở rộng) rất khác nhau
   (ví dụ: write 100 RPS, read 100,000 RPS)

✅ Read model cần denormalized data phức tạp 
   (nhiều JOIN, aggregation từ nhiều bảng)

✅ Cần nhiều "views" khác nhau của cùng data
   (dashboard, report, search, mobile API)

✅ Kết hợp với Event Sourcing

✅ Team lớn — tách write và read team
```

### Không Nên Dùng CQRS Khi

```
❌ Simple CRUD application (CQRS adds unnecessary complexity)
❌ Read và write có cùng nhu cầu
❌ Team nhỏ, không đủ bandwidth maintain 2 data models
❌ Consistency (nhất quán) tuyệt đối là yêu cầu bắt buộc
   (eventual consistency không chấp nhận được)
❌ Chưa có performance problem (đừng over-engineer sớm)
```

### CQRS Levels (Các Mức Độ)

```
Level 1: Tách Command và Query trong cùng service, cùng database
  → Nhẹ nhàng, ít complexity, vẫn có lợi ích code organization

Level 2: Tách Command và Query service, cùng database
  → Moderate complexity, có thể scale riêng

Level 3: Tách hoàn toàn với separate read store
  → Full CQRS, maximum flexibility, maximum complexity

→ Bắt đầu từ Level 1, nâng lên Level 2-3 khi thực sự cần
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: CQRS là gì và tại sao dùng?

**Trả lời:**
> CQRS — Command Query Responsibility Segregation — tách hệ thống thành hai luồng riêng: Command Side (write — ghi) và Query Side (read — đọc). Mỗi phía có model riêng, thậm chí database riêng, tối ưu cho nhu cầu của nó.
>
> Dùng CQRS khi read và write có nhu cầu rất khác nhau. Ví dụ: write cần ACID transaction, normalized schema; read cần denormalized data, phức tạp JOIN, cần search full-text. Tối ưu cùng một database cho cả hai là bất khả thi.
>
> Nhược điểm chính: eventual consistency giữa write và read model, và độ phức tạp tăng đáng kể.

### Câu 2: CQRS và Event Sourcing khác nhau như thế nào, chúng kết hợp thế nào?

**Trả lời:**
> CQRS và Event Sourcing là hai pattern độc lập, không phụ thuộc nhau, nhưng thường kết hợp tốt với nhau.
>
> CQRS là về tách read/write architecture. Event Sourcing là về cách lưu state (theo chuỗi events thay vì state hiện tại).
>
> Kết hợp: Event Sourcing làm write store tự nhiên, events được publish từ event store → projectors cập nhật read models. Đây là kiến trúc cực kỳ linh hoạt — thêm read model mới không cần migrate data, chỉ cần replay events.

### Câu 3: Giải quyết read-your-write problem trong CQRS?

**Trả lời:**
> Read-your-write problem: user vừa tạo order xong, query ngay nhưng chưa thấy vì projector chưa cập nhật read model.
>
> Giải pháp tùy use case: (1) Optimistic UI — frontend tự hiển thị data ngay trước khi query; (2) Command trả về event version, query poll cho đến khi read model đạt version đó; (3) Một số critical paths dùng synchronous projection; (4) Nhiều trường hợp chấp nhận vài giây lag là OK.

---

**Cập Nhật Lần Cuối:** 2026-05-18  
**Tags:** CQRS, command, query, read-model, write-model, event-sourcing, DynamoDB, OpenSearch
