# DynamoDB — Indexes (Chỉ Mục Phụ)

> GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) và LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ) cho phép query DynamoDB theo nhiều chiều khác nhau ngoài primary key.

## 📚 Mục Lục

1. [Tại Sao Cần Secondary Indexes?](#tại-sao-cần-secondary-indexes)
2. [LSI — Local Secondary Index](#lsi--local-secondary-index)
3. [GSI — Global Secondary Index](#gsi--global-secondary-index)
4. [So Sánh LSI vs GSI](#so-sánh-lsi-vs-gsi)
5. [Index Projection — Chiếu Dữ Liệu](#index-projection--chiếu-dữ-liệu)
6. [Sparse Index — Chỉ Mục Thưa](#sparse-index--chỉ-mục-thưa)
7. [GSI Overloading — Tái Sử Dụng GSI](#gsi-overloading--tái-sử-dụng-gsi)
8. [Thiết Kế Index Thực Tế](#thiết-kế-index-thực-tế)
9. [Chi Phí & Hiệu Năng Index](#chi-phí--hiệu-năng-index)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Secondary Indexes?

DynamoDB chỉ cho phép query theo primary key. Để query theo các attribute khác, cần secondary index (chỉ mục phụ):

```
Bảng Orders:
PK = UserId | SK = OrderId | Status | CreatedAt | TotalAmount

Câu hỏi: "Tìm tất cả orders có Status = 'PENDING'"
→ Không thể query trực tiếp (Status không phải PK)
→ SCAN toàn bảng = tốn kém và chậm
→ Giải pháp: Tạo GSI với PK = Status
```

---

## LSI — Local Secondary Index

### Định Nghĩa

LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ) là chỉ mục có cùng **Partition Key** với base table nhưng **Sort Key khác**.

- "Local" = cùng partition với base table
- Phải tạo **lúc tạo bảng** — không thể thêm sau
- Tối đa **5 LSI** mỗi bảng
- Dùng chung **storage** với base table (tính vào table quota)
- Hỗ trợ cả **strongly** và **eventually consistent reads**

### Ví Dụ LSI

```
Base Table: Orders
PK = UserId | SK = OrderId | Status | CreatedAt | TotalAmount

LSI (Index name: "ByCreatedAt"):
PK = UserId | SK = CreatedAt  ← Sort key khác
(Cùng partition với UserId, nhưng sort theo CreatedAt)

Queries được hỗ trợ với LSI:
✅ "Tất cả orders của user-001 từ tháng 1-2024"
   → Query trên LSI: PK=user-001, SK begins_with("2024-01")
```

### Tạo LSI

```python
import boto3
dynamodb = boto3.client('dynamodb')

dynamodb.create_table(
    TableName='Orders',
    AttributeDefinitions=[
        {'AttributeName': 'UserId',    'AttributeType': 'S'},
        {'AttributeName': 'OrderId',   'AttributeType': 'S'},
        {'AttributeName': 'CreatedAt', 'AttributeType': 'S'},  # LSI sort key
    ],
    KeySchema=[
        {'AttributeName': 'UserId',  'KeyType': 'HASH'},
        {'AttributeName': 'OrderId', 'KeyType': 'RANGE'},
    ],
    LocalSecondaryIndexes=[
        {
            'IndexName': 'ByCreatedAt',
            'KeySchema': [
                {'AttributeName': 'UserId',    'KeyType': 'HASH'},   # Giống base table
                {'AttributeName': 'CreatedAt', 'KeyType': 'RANGE'},  # Sort key mới
            ],
            'Projection': {'ProjectionType': 'ALL'}
        }
    ],
    BillingMode='PAY_PER_REQUEST'
)
```

### Query trên LSI

```python
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Query tất cả orders của user-001 trong tháng 1-2024
response = table.query(
    IndexName='ByCreatedAt',  # Chỉ định index name
    KeyConditionExpression=Key('UserId').eq('user-001') &
                            Key('CreatedAt').begins_with('2024-01')
)
```

---

## GSI — Global Secondary Index

### Định Nghĩa

GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) là chỉ mục với **Partition Key và Sort Key hoàn toàn khác** với base table.

- "Global" = span qua tất cả partitions của base table
- Có thể tạo **bất cứ lúc nào** — ngay cả khi bảng đang có dữ liệu
- Tối đa **20 GSI** mỗi bảng
- Có **storage và capacity riêng**
- Chỉ hỗ trợ **eventually consistent reads**
- Data sync bất đồng bộ (async) từ base table → có độ trễ nhỏ

### Ví Dụ GSI

```
Base Table: Orders
PK = UserId | SK = OrderId | Status | CreatedAt | TotalAmount

GSI 1 (Index: "StatusIndex"):
PK = Status | SK = CreatedAt
→ Query: "Tất cả orders PENDING sắp xếp theo thời gian"

GSI 2 (Index: "DateIndex"):
PK = CreatedAt (dạng YYYY-MM-DD) | SK = TotalAmount
→ Query: "Tất cả orders ngày 2024-01-15 sắp xếp theo giá trị"
```

### Tạo GSI

```python
# Thêm GSI vào bảng đang tồn tại
dynamodb.update_table(
    TableName='Orders',
    AttributeDefinitions=[
        {'AttributeName': 'Status',    'AttributeType': 'S'},
        {'AttributeName': 'CreatedAt', 'AttributeType': 'S'},
    ],
    GlobalSecondaryIndexUpdates=[
        {
            'Create': {
                'IndexName': 'StatusIndex',
                'KeySchema': [
                    {'AttributeName': 'Status',    'KeyType': 'HASH'},
                    {'AttributeName': 'CreatedAt', 'KeyType': 'RANGE'},
                ],
                'Projection': {
                    'ProjectionType': 'INCLUDE',
                    'NonKeyAttributes': ['TotalAmount', 'UserId']
                },
                'ProvisionedThroughput': {
                    'ReadCapacityUnits': 5,
                    'WriteCapacityUnits': 5
                }
            }
        }
    ]
)
```

### Query trên GSI

```python
# Tất cả PENDING orders trong tháng 1-2024
response = table.query(
    IndexName='StatusIndex',
    KeyConditionExpression=Key('Status').eq('PENDING') &
                            Key('CreatedAt').begins_with('2024-01'),
    ScanIndexForward=True   # True = ascending, False = descending
)

# Chỉ đọc eventually consistent từ GSI (mặc định)
# Không thể dùng ConsistentRead=True với GSI
```

---

## So Sánh LSI vs GSI

| Tiêu Chí                    | LSI                                    | GSI                                        |
| --------------------------- | -------------------------------------- | ------------------------------------------ |
| **Partition Key**           | Giống base table                       | Có thể khác hoàn toàn                      |
| **Sort Key**                | Khác base table                        | Khác base table, có thể là PK của base     |
| **Thời Điểm Tạo**          | **Chỉ khi tạo bảng**                   | **Bất cứ lúc nào**                         |
| **Số Lượng Tối Đa**        | 5 LSI/bảng                             | 20 GSI/bảng                                |
| **Storage**                 | Dùng chung với base table              | Storage riêng                              |
| **Capacity**                | Dùng chung RCU/WCU với base table      | RCU/WCU riêng (Provisioned)               |
| **Consistency**             | Strongly hoặc Eventually               | **Chỉ Eventually Consistent**              |
| **Strong Consistency Reads** | ✅ Có                                 | ❌ Không                                   |
| **Phù Hợp**                | Sort theo attribute khác trong cùng partition | Query theo entity type hoàn toàn khác |

### Quy Tắc Chọn Lựa

```
Chọn LSI khi:
✅ Cần strongly consistent reads trên index
✅ Query luôn filter theo cùng partition key (UserId, TenantId...)
✅ Biết trước access pattern khi thiết kế bảng
✅ Muốn tiết kiệm (không cần capacity riêng)

Chọn GSI khi:
✅ Cần query theo attribute không liên quan đến PK
✅ Eventually consistent là đủ
✅ Cần thêm index sau khi bảng đã có dữ liệu
✅ Access pattern yêu cầu "scan tất cả items theo status"
```

---

## Index Projection — Chiếu Dữ Liệu

### Các Loại Projection

Projection quyết định attributes nào được copy sang index:

```
KEYS_ONLY — Chỉ copy primary key của base table + index key
├── Ưu điểm: Storage nhỏ nhất, chi phí thấp
└── Nhược điểm: Để đọc non-key attributes phải fetch từ base table (tốn RCU)

INCLUDE — Copy key + danh sách attributes chỉ định
├── Ưu điểm: Cân bằng giữa storage và flexibility
└── Nhược điểm: Cần biết trước cần attributes gì

ALL — Copy tất cả attributes
├── Ưu điểm: Query index không cần fetch thêm
└── Nhược điểm: Tốn storage gấp đôi (data exist ở cả base table và index)
```

### Ví Dụ Thực Tế

```python
# Scenario: Query Status index, chỉ cần hiển thị OrderId + TotalAmount

# KEYS_ONLY: phải fetch thêm từ base table
response = table.query(IndexName='StatusIndex', ...)
for item in response['Items']:
    # item chỉ có: Status, CreatedAt, UserId, OrderId
    # Phải get_item thêm để lấy TotalAmount → N+1 problem!
    detail = table.get_item(Key={'UserId': item['UserId'], 'OrderId': item['OrderId']})

# INCLUDE với NonKeyAttributes=['TotalAmount']:
response = table.query(IndexName='StatusIndex', ...)
for item in response['Items']:
    # item có: Status, CreatedAt, UserId, OrderId, TotalAmount
    # Không cần fetch thêm ✅
    print(item['TotalAmount'])
```

### Chiến Lược Projection

```
Nếu query index luôn cần tất cả attributes → ALL (thuận tiện nhất)
Nếu index rất lớn hoặc nhiều GSI → INCLUDE (cân bằng)
Nếu chỉ cần check existence hoặc count → KEYS_ONLY
```

---

## Sparse Index — Chỉ Mục Thưa

### Khái Niệm

Sparse Index là GSI chỉ index những items **có attribute được dùng làm GSI partition key**. Items không có attribute đó **không xuất hiện** trong index — tạo ra "index thưa" nhỏ hơn nhiều so với base table.

### Ví Dụ: Index Chỉ Orders Chưa Xử Lý

```
Base Table: Orders (100,000 items)
├── 95,000 orders đã xử lý: không có attribute "PendingAt"
└──  5,000 orders chưa xử lý: có attribute "PendingAt" = timestamp

GSI "PendingOrdersIndex":
  PK = PendingAt

→ GSI chỉ chứa 5,000 items (5%) thay vì 100,000 items
→ Query "tất cả orders chưa xử lý" cực kỳ hiệu quả!
```

```python
# Khi tạo order chưa xử lý: thêm PendingAt attribute
table.put_item(Item={
    'UserId': 'user-001',
    'OrderId': 'ord-001',
    'Status': 'PENDING',
    'PendingAt': '2024-01-15T10:00:00Z'  # GSI key
})

# Khi order đã xử lý: xóa PendingAt attribute
table.update_item(
    Key={'UserId': 'user-001', 'OrderId': 'ord-001'},
    UpdateExpression='SET #s = :s REMOVE PendingAt',
    ExpressionAttributeNames={'#s': 'Status'},
    ExpressionAttributeValues={':s': 'PROCESSED'}
)

# Query sparse index: chỉ lấy pending orders
response = table.query(
    IndexName='PendingOrdersIndex',
    KeyConditionExpression=Key('PendingAt').begins_with('2024-01')
)
```

**Sparse Index là một trong những pattern mạnh nhất của DynamoDB.**

---

## GSI Overloading — Tái Sử Dụng GSI

### Vấn Đề

Tối đa 20 GSI mỗi bảng. Trong Single-Table Design (Thiết Kế Đơn Bảng) với nhiều loại entity, 20 GSI có thể không đủ.

### Giải Pháp: GSI Overloading

Dùng **generic attribute names** (tên thuộc tính chung) như `GSI1PK`, `GSI1SK` để một GSI phục vụ nhiều loại entity khác nhau:

```
Base Table: (Single Table Design)
├── Entity: User
│   PK = "USER#user-001"  | SK = "PROFILE"
│   GSI1PK = "USER#user-001" | GSI1SK = "2024-01-01" (join date)
│
├── Entity: Order
│   PK = "USER#user-001"  | SK = "ORDER#ord-001"
│   GSI1PK = "STATUS#PENDING" | GSI1SK = "2024-01-15T10:00:00Z"
│
└── Entity: Product
    PK = "PRODUCT#prod-001" | SK = "DETAIL"
    GSI1PK = "CATEGORY#electronics" | GSI1SK = "prod-001"

GSI1 (PK=GSI1PK, SK=GSI1SK) phục vụ 3 access patterns khác nhau:
✅ Users joined on date
✅ Pending orders by timestamp
✅ Products by category
```

---

## Thiết Kế Index Thực Tế

### Case Study: E-Commerce Platform

**Bảng:** `EcommerceTable` (Single Table)

**Access Patterns:**
1. Lấy thông tin user theo UserId
2. Lấy tất cả orders của user
3. Tìm orders theo status + thời gian
4. Tìm sản phẩm theo category
5. Tìm sessions đang active

```
Base Table:
PK          | SK              | GSI1PK          | GSI1SK     | GSI2PK     | GSI2SK
------------------------------------------------------------------------
USER#u-001  | PROFILE         | -               | -          | -          | -
USER#u-001  | ORDER#o-001     | STATUS#PENDING  | 2024-01-15 | -          | -
USER#u-001  | ORDER#o-002     | STATUS#SHIPPED  | 2024-02-01 | -          | -
PRODUCT#p-1 | DETAIL          | CAT#electronics | p-001      | -          | -
SESSION#s-1 | ACTIVE          | SESSION#ACTIVE  | 2024-01-15 | -          | -

GSI1 (PK=GSI1PK, SK=GSI1SK):
→ Query STATUS#PENDING → đáp ứng pattern #3
→ Query CAT#electronics → đáp ứng pattern #4
→ Query SESSION#ACTIVE → đáp ứng pattern #5

Base Table Query:
→ PK=USER#u-001, SK begins_with("ORDER#") → đáp ứng pattern #2
→ PK=USER#u-001, SK=PROFILE → đáp ứng pattern #1
```

---

## Chi Phí & Hiệu Năng Index

### Chi Phí Ghi

Mỗi lần ghi vào base table, DynamoDB **tự động propagate** sang tất cả indexes:

```
Nếu bảng có 5 GSI + 3 LSI:
1 write vào base table → 1 write + 5 GSI writes + 3 LSI writes = 9 writes!

Ảnh Hưởng Chi Phí:
- Mỗi GSI tăng WCU consumption
- On-Demand: tăng số WRU tính phí
- Provisioned: cần đặt đủ WCU cho cả base table và indexes
```

### Chi Phí Lưu Trữ

```
Storage tính cho:
- Base table storage
- Mỗi LSI: keys + projected attributes (dùng chung với base table quota)
- Mỗi GSI: keys + projected attributes (tính riêng)

Projection ALL: GSI storage ≈ Base table storage
→ 5 GSI với ALL projection = ~6x storage cost!
```

### Giới Hạn & Best Practices

```
✅ Minimize số lượng GSI — chỉ tạo khi thật sự cần
✅ Dùng INCLUDE projection thay vì ALL khi có thể
✅ Dùng Sparse Index để giảm index size
✅ GSI Overloading để dùng ít GSI hơn
✅ Monitor GSI throttling riêng với GSI consumed capacity metrics

❌ Không tạo GSI cho mọi attribute — tốn kém
❌ Không dùng ALL projection cho GSI lớn không cần thiết
```

---

## Câu Hỏi Phỏng Vấn

**Q: GSI vs LSI — khác biệt chính và khi nào dùng loại nào?**

A: LSI (Local Secondary Index) cùng partition key với base table, phải tạo khi tạo bảng, hỗ trợ strongly consistent reads. GSI (Global Secondary Index) có partition key hoàn toàn khác, tạo được bất cứ lúc nào, chỉ hỗ trợ eventually consistent reads. Dùng LSI khi cần strongly consistent reads hoặc query chỉ trong cùng partition. Dùng GSI khi cần query theo chiều hoàn toàn khác (ví dụ: theo status, category) hoặc khi cần thêm index sau khi bảng đã có data.

**Q: Sparse Index là gì và tại sao hữu ích?**

A: Sparse Index là GSI chỉ chứa items có attribute làm GSI partition key — items không có attribute đó không xuất hiện trong index. Hữu ích khi chỉ một phần nhỏ items cần được index theo chiều đó (ví dụ: chỉ pending orders, chỉ premium users). Kết quả là index nhỏ hơn nhiều, query nhanh hơn và tốn ít storage/capacity hơn.

**Q: Tại sao GSI chỉ hỗ trợ eventually consistent reads?**

A: GSI được cập nhật bất đồng bộ (asynchronously) từ base table — khi bạn ghi vào base table, DynamoDB propagate thay đổi sang GSI trong vài mili-giây sau. Do đó, GSI có thể "behind" base table một chút. Strongly consistent reads từ GSI không có ý nghĩa vì data đã được sao chép với độ trễ. Nếu cần strongly consistent reads, phải dùng base table hoặc LSI.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
