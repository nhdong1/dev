# Schema Registry — Quản Lý Lược Đồ Sự Kiện

> **Schema Registry** (Kho Lưu Lược Đồ) là dịch vụ trong EventBridge cho phép khám phá, quản lý và chia sẻ schema (lược đồ) của sự kiện — giải quyết vấn đề "contract" (hợp đồng) giữa producer (nhà sản xuất sự kiện) và consumer (người tiêu dùng sự kiện).

---

## 📚 Mục Lục

1. [Vấn Đề Schema Registry Giải Quyết](#1-vấn-đề-schema-registry-giải-quyết)
2. [Schema Discovery — Tự Động Khám Phá Schema](#2-schema-discovery--tự-động-khám-phá-schema)
3. [Schema Versioning — Quản Lý Phiên Bản](#3-schema-versioning--quản-lý-phiên-bản)
4. [Code Binding — Sinh Mã Tự Động](#4-code-binding--sinh-mã-tự-động)
5. [Schema Registry API](#5-schema-registry-api)
6. [Best Practices](#6-best-practices)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Schema Registry Giải Quyết

### Tình Huống Thực Tế Không Dùng Schema Registry

```
Team Orders:                          Team Inventory:
Gửi sự kiện:                         Nhận sự kiện:
{                                     Mong đợi:
  "orderId": "ORD-001",               {
  "amount": 99.99,                      "order_id": "...",    ← Khác tên!
  "items": [...]                        "total": ...,         ← Khác tên!
}                                       "products": [...]     ← Khác tên!
                                      }

Kết quả: Consumer bị lỗi runtime, mất dữ liệu, debug rất khó
```

### Schema Registry Là Giải Pháp

```
Producer ──▶ Event Bus ──▶ Schema Registry ──▶ Consumer
                               │
                    Lưu trữ schema chuẩn:
                    - Tên trường chính xác
                    - Kiểu dữ liệu
                    - Trường bắt buộc / tùy chọn
                    - Version history
                    - Code binding tự động
```

**Lợi ích cốt lõi:**
1. **Single source of truth** (Nguồn Sự Thật Duy Nhất) cho event schema
2. **Auto-discovery** (Tự Động Khám Phá) — không cần viết schema tay
3. **Code generation** (Sinh Mã Tự Động) — tạo class/type từ schema
4. **Version control** (Kiểm Soát Phiên Bản) — theo dõi thay đổi schema
5. **IDE integration** (Tích Hợp IDE) — auto-complete khi code

---

## 2. Schema Discovery — Tự Động Khám Phá Schema

### Bật Schema Discovery

```bash
# Bật discovery trên event bus
aws schemas create-discoverer \
  --source-arn "arn:aws:events:ap-southeast-1:123456789:event-bus/orders-bus" \
  --description "Auto-discover schemas for orders-bus"

# Kiểm tra danh sách discoverers
aws schemas list-discoverers
```

Sau khi bật, EventBridge **tự động phân tích** các sự kiện đến và tạo schema:

```
Sự kiện đến:
{
  "source": "com.mycompany.orders",
  "detail-type": "Order Placed",
  "detail": {
    "orderId": "ORD-001",
    "amount": 99.99,
    "status": "PENDING"
  }
}

↓ Schema Registry tự tạo schema:

{
  "$schema": "https://json-schema.org/draft-04/schema#",
  "title": "Order Placed",
  "type": "object",
  "properties": {
    "orderId": { "type": "string" },
    "amount": { "type": "number" },
    "status": { "type": "string" }
  }
}
```

### Chi Phí Discovery

```
$0.10 / triệu sự kiện được phân tích
→ Nên tắt sau khi schema đã ổn định
```

### Tìm Schema Trong Registry

```bash
# List tất cả schemas
aws schemas list-schemas \
  --registry-name "discovered-schemas"

# Tìm schema theo tên
aws schemas search-schemas \
  --registry-name "discovered-schemas" \
  --keywords "Order Placed"

# Xem chi tiết schema
aws schemas describe-schema \
  --registry-name "discovered-schemas" \
  --schema-name "com.mycompany.orders@OrderPlaced"
```

---

## 3. Schema Versioning — Quản Lý Phiên Bản

### Cấu Trúc Version

```
Schema: com.mycompany.orders@OrderPlaced
├── Version 1 (2026-01-01) — Initial schema
│     { orderId, amount, status }
├── Version 2 (2026-03-15) — Thêm field customerId
│     { orderId, amount, status, customerId }
└── Version 3 (2026-05-18) — Thêm field items[]
      { orderId, amount, status, customerId, items[] }
```

### Tạo Và Cập Nhật Schema

```python
import boto3
import json

schemas_client = boto3.client('schemas')

# Tạo schema mới (version 1)
schema_content = {
    "$schema": "https://json-schema.org/draft-04/schema#",
    "title": "OrderPlaced",
    "type": "object",
    "properties": {
        "orderId": {
            "type": "string",
            "description": "Mã đơn hàng duy nhất"
        },
        "customerId": {
            "type": "string",
            "description": "Mã khách hàng"
        },
        "amount": {
            "type": "number",
            "description": "Tổng giá trị đơn hàng (VND)"
        },
        "status": {
            "type": "string",
            "enum": ["PENDING", "CONFIRMED", "PROCESSING"],
            "description": "Trạng thái đơn hàng"
        },
        "items": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "productId": { "type": "string" },
                    "quantity": { "type": "integer" },
                    "price": { "type": "number" }
                },
                "required": ["productId", "quantity", "price"]
            }
        }
    },
    "required": ["orderId", "customerId", "amount", "status"]
}

response = schemas_client.create_schema(
    RegistryName='mycompany-schemas',
    SchemaName='com.mycompany.orders@OrderPlaced',
    Type='JSONSchemaDraft4',
    Content=json.dumps(schema_content),
    Description='Schema cho sự kiện đặt hàng'
)

print(f"Schema version: {response['SchemaVersion']}")

# Cập nhật schema (tạo version mới)
updated_schema = schema_content.copy()
updated_schema['properties']['discountCode'] = {
    "type": "string",
    "description": "Mã giảm giá (tùy chọn)"
}

schemas_client.update_schema(
    RegistryName='mycompany-schemas',
    SchemaName='com.mycompany.orders@OrderPlaced',
    Type='JSONSchemaDraft4',
    Content=json.dumps(updated_schema)
)
```

### List Versions Của Một Schema

```bash
aws schemas list-schema-versions \
  --registry-name "mycompany-schemas" \
  --schema-name "com.mycompany.orders@OrderPlaced"
```

---

## 4. Code Binding — Sinh Mã Tự Động

**Code Binding** (Ràng Buộc Mã) cho phép EventBridge tự động sinh code từ schema — có thể dùng ngay trong IDE.

### Ngôn Ngữ Hỗ Trợ

| Ngôn Ngữ | Runtime | Hỗ Trợ |
|---|---|---|
| **Java** | JVM | ✅ |
| **Python** | CPython | ✅ |
| **TypeScript** | Node.js | ✅ |
| **Go** | Go runtime | ✅ |

### Sinh Code Tự Động

```bash
# Tạo code binding cho Python
aws schemas get-code-binding-source \
  --registry-name "mycompany-schemas" \
  --schema-name "com.mycompany.orders@OrderPlaced" \
  --language "Python36" \
  --schema-version "1" \
  --output text > order_placed_schema.zip
```

### Code Python Được Sinh Ra

```python
# Ví dụ code Python được sinh từ Schema Registry
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    price: float

@dataclass
class OrderPlaced:
    order_id: str
    customer_id: str
    amount: float
    status: str
    items: List[OrderItem]
    discount_code: Optional[str] = None

    @staticmethod
    def from_event(event: dict) -> 'OrderPlaced':
        """Parse EventBridge event thành OrderPlaced object."""
        detail = event['detail']
        return OrderPlaced(
            order_id=detail['orderId'],
            customer_id=detail['customerId'],
            amount=detail['amount'],
            status=detail['status'],
            items=[
                OrderItem(
                    product_id=item['productId'],
                    quantity=item['quantity'],
                    price=item['price']
                )
                for item in detail.get('items', [])
            ],
            discount_code=detail.get('discountCode')
        )

# Dùng trong Lambda handler
def lambda_handler(event, context):
    order = OrderPlaced.from_event(event)
    print(f"Processing order {order.order_id} for customer {order.customer_id}")
    print(f"Total: {order.amount:,.0f} VND")

    if order.discount_code:
        print(f"Discount code applied: {order.discount_code}")

    # Type-safe processing — IDE biết kiểu dữ liệu
    for item in order.items:
        print(f"  - Product {item.product_id}: {item.quantity} x {item.price:,.0f} VND")
```

### Code TypeScript Được Sinh Ra

```typescript
// Ví dụ code TypeScript được sinh từ Schema Registry
export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

export interface OrderPlaced {
  orderId: string;
  customerId: string;
  amount: number;
  status: 'PENDING' | 'CONFIRMED' | 'PROCESSING';
  items: OrderItem[];
  discountCode?: string;
}

// Lambda handler với type safety đầy đủ
export const handler = async (event: AWSEvent<OrderPlaced>): Promise<void> => {
  const order = event.detail;

  console.log(`Processing order ${order.orderId}`);

  // IDE auto-complete hoạt động đầy đủ nhờ type binding
  order.items.forEach(item => {
    console.log(`  - ${item.productId}: ${item.quantity} × ${item.price}`);
  });
};
```

---

## 5. Schema Registry API

### Registry Management (Quản Lý Registry)

```python
import boto3

client = boto3.client('schemas')

# Tạo custom registry
client.create_registry(
    RegistryName='mycompany-schemas',
    Description='Schemas cho tất cả sự kiện của MyCompany'
)

# List registries
registries = client.list_registries()

# Registries mặc định của AWS:
# - aws.events         — schemas cho AWS service events
# - discovered-schemas — schemas được auto-discovered
```

### Schema Search (Tìm Kiếm Schema)

```python
# Tìm tất cả schemas liên quan đến "Order"
results = client.search_schemas(
    RegistryName='mycompany-schemas',
    Keywords='Order'
)

for schema in results['Schemas']:
    print(f"Schema: {schema['SchemaName']}")
    print(f"  ARN: {schema['SchemaArn']}")
    print(f"  Version: {schema['SchemaVersion']}")
    print(f"  Last modified: {schema['LastModified']}")
```

### Export Schema (Xuất Schema)

```bash
# Xuất schema dưới dạng OpenAPI 3.0
aws schemas export-schema \
  --registry-name "mycompany-schemas" \
  --schema-name "com.mycompany.orders@OrderPlaced" \
  --type "OpenApi3" \
  --output json
```

---

## 6. Best Practices

### Quy Ước Đặt Tên Schema

```
Format: <source>@<detail-type>
Ví dụ:
  com.mycompany.orders@OrderPlaced
  com.mycompany.orders@OrderCancelled
  com.mycompany.payments@PaymentProcessed
  com.mycompany.inventory@StockUpdated
```

### Backward Compatibility (Tương Thích Ngược)

```
✅ Thay đổi AN TOÀN (backward compatible):
  - Thêm trường mới tùy chọn (optional)
  - Thêm giá trị mới vào enum (consumer phải handle unknown values)

❌ Thay đổi BREAKING (phá vỡ tương thích ngược):
  - Xóa trường bắt buộc
  - Đổi tên trường
  - Đổi kiểu dữ liệu (string → number)
  - Thêm constraint mới vào trường hiện có
```

### Schema Evolution Strategy (Chiến Lược Tiến Hóa Schema)

```
Version 1: { orderId, amount }
  ↓
Version 2: { orderId, amount, customerId? }     ← Thêm optional field — SAFE
  ↓
Version 3: { orderId, amount, customerId, items? } ← Thêm optional array — SAFE
  ↓
Nếu cần thay đổi breaking:
  → Tạo event type mới: OrderPlacedV2 (không sửa schema cũ)
  → Migrate consumers sang event type mới
  → Deprecated event type cũ sau khi migrate xong
```

### Tắt Discovery Sau Khi Ổn Định

```bash
# Discovery tốn phí $0.10/triệu events — tắt khi schema đã confirmed
aws schemas stop-discoverer \
  --discoverer-id "arn:aws:schemas:ap-southeast-1:123456789:discoverer/events-policy-..."
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Schema Registry giải quyết vấn đề gì trong event-driven architecture?**

> Schema Registry giải quyết vấn đề "schema contract" (hợp đồng lược đồ) giữa producer và consumer. Trong distributed systems (hệ thống phân tán), producer và consumer thường do các team khác nhau phát triển — nếu không có schema registry, consumer phải đoán cấu trúc event, dẫn đến lỗi runtime khó debug. Schema Registry cung cấp: (1) single source of truth cho event structure, (2) version history để theo dõi thay đổi, (3) code binding để IDE biết kiểu dữ liệu.

**Q: Sự khác biệt giữa Schema Discovery và Schema Creation thủ công?**

> Schema Discovery (tự động): EventBridge phân tích sự kiện thực tế và suy luận schema. Nhanh để bắt đầu, nhưng schema có thể không hoàn chỉnh (không biết enum values, không rõ required vs optional). Schema Creation thủ công: Team định nghĩa schema trước khi code → design-first approach, chính xác hơn, hỗ trợ validation. Best practice: Dùng discovery để khởi tạo nhanh, rồi chỉnh sửa thủ công cho chính xác.

**Q: Khi schema thay đổi breaking change, làm thế nào để không break consumers?**

> Không bao giờ sửa schema hiện có theo cách breaking. Thay vào đó: (1) Tạo detail-type mới (`OrderPlacedV2`) với schema mới, (2) Producer gửi cả hai versions trong thời gian chuyển tiếp, (3) Migrate từng consumer sang version mới, (4) Khi tất cả consumers đã migrate, dừng gửi version cũ. Đây gọi là "two-speed migration" (di chuyển hai tốc độ).

---

**Liên Kết:** [2-event-rules-patterns.md](2-event-rules-patterns.md) | [4-archive-replay.md](4-archive-replay.md)
