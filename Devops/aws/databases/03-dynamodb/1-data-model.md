# DynamoDB — Data Model (Mô Hình Dữ Liệu)

> Hiểu rõ mô hình dữ liệu DynamoDB là nền tảng để thiết kế bảng hiệu quả: Tables (Bảng), Partition Key (Khóa Phân Vùng), Sort Key (Khóa Sắp Xếp), Attributes (Thuộc Tính) và cách chúng ảnh hưởng đến hiệu năng.

## 📚 Mục Lục

1. [Tổng Quan Mô Hình Dữ Liệu](#tổng-quan-mô-hình-dữ-liệu)
2. [Table — Bảng DynamoDB](#table--bảng-dynamodb)
3. [Primary Key — Khóa Chính](#primary-key--khóa-chính)
4. [Partition Key — Khóa Phân Vùng](#partition-key--khóa-phân-vùng)
5. [Sort Key — Khóa Sắp Xếp](#sort-key--khóa-sắp-xếp)
6. [Attributes — Thuộc Tính](#attributes--thuộc-tính)
7. [Item — Mục Dữ Liệu](#item--mục-dữ-liệu)
8. [Cơ Chế Phân Vùng Nội Bộ](#cơ-chế-phân-vùng-nội-bộ)
9. [Data Types — Kiểu Dữ Liệu](#data-types--kiểu-dữ-liệu)
10. [Giới Hạn Quan Trọng](#giới-hạn-quan-trọng)
11. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Mô Hình Dữ Liệu

DynamoDB khác biệt hoàn toàn với RDBMS (Relational Database Management System — Hệ Thống Quản Lý Cơ Sở Dữ Liệu Quan Hệ). Thay vì tables với fixed schema (schema cố định), DynamoDB dùng flexible schema (schema linh hoạt) — mỗi item có thể có tập hợp attributes khác nhau.

```
RDBMS (SQL):                      DynamoDB:
┌────────────────────┐            ┌──────────────────────────────┐
│ Table: Users       │            │ Table: Users                 │
│ id | name | email  │            │ PK=UserId | Attrs (linh hoạt)│
│ 1  | Alice | a@... │            │ "u1" | {name, email, age}     │
│ 2  | Bob   | b@... │            │ "u2" | {name, phone}          │
│ 3  | Carol | c@... │            │ "u3" | {name, email, premium} │
└────────────────────┘            └──────────────────────────────┘
  Cột cố định, mọi row            Mỗi item có thể có attributes
  phải có cùng cột                khác nhau — không cần NULL
```

---

## Table — Bảng DynamoDB

### Đặc Điểm Table

- **Không có schema cố định** — chỉ primary key là bắt buộc và cố định
- **Không giới hạn kích thước** — có thể lưu petabyte dữ liệu
- **Một primary key** — xác định duy nhất mỗi item
- **Tối đa 2500 bảng** mỗi account theo mặc định (có thể tăng)
- **Regional service** — bảng thuộc về một AWS Region cụ thể

### Tạo Table (AWS CLI)

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=UserId,AttributeType=S \
    AttributeName=OrderId,AttributeType=S \
  --key-schema \
    AttributeName=UserId,KeyType=HASH \
    AttributeName=OrderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

- `HASH` = Partition Key (Khóa Phân Vùng)
- `RANGE` = Sort Key (Khóa Sắp Xếp)

---

## Primary Key — Khóa Chính

Primary Key (Khóa Chính) xác định duy nhất mỗi item trong bảng. DynamoDB hỗ trợ 2 loại:

### Loại 1: Simple Primary Key (Khóa Chính Đơn Giản)

Chỉ dùng Partition Key. Mỗi giá trị Partition Key phải là duy nhất trong toàn bảng.

```
Table: Products
┌─────────────────┬──────────────────────────────┐
│ ProductId (PK)  │ Attributes                   │
├─────────────────┼──────────────────────────────┤
│ "p-001"         │ {name: "Laptop", price: 999} │
│ "p-002"         │ {name: "Phone", price: 499}  │
│ "p-003"         │ {name: "Tablet", price: 299} │
└─────────────────┴──────────────────────────────┘
```

**Dùng khi:** Mỗi entity là duy nhất và không cần group hay sort.

### Loại 2: Composite Primary Key (Khóa Chính Tổng Hợp)

Kết hợp Partition Key + Sort Key. Hai items có thể có cùng Partition Key nếu Sort Key khác nhau.

```
Table: Orders
┌────────────┬──────────────────┬──────────────────────────────┐
│ UserId(PK) │ OrderId (SK)     │ Attributes                   │
├────────────┼──────────────────┼──────────────────────────────┤
│ "user-001" │ "2024-01-order1" │ {status: "PAID", total: 150} │
│ "user-001" │ "2024-02-order2" │ {status: "SHIP", total: 80}  │
│ "user-002" │ "2024-01-order3" │ {status: "PAID", total: 200} │
└────────────┴──────────────────┴──────────────────────────────┘
```

**Dùng khi:** Cần lưu nhiều items liên quan dưới cùng một "parent" (cha mẹ).

---

## Partition Key — Khóa Phân Vùng

### Cách DynamoDB Dùng Partition Key

DynamoDB áp dụng hàm hash (hàm băm) lên Partition Key để xác định partition (phân vùng) nào lưu item đó:

```
Partition Key Value
       │
       ▼
  Hash Function
  (Hàm Băm)
       │
       ▼
  Partition ID ──► Storage Node (Nút Lưu Trữ)
```

Quá trình này hoàn toàn tự động và trong suốt với developer.

### Đặc Điểm Partition Key Tốt

```
✅ HIGH CARDINALITY (Số Lượng Giá Trị Cao)
   - UserId (UUID): hàng triệu giá trị khác nhau
   - OrderId (ULID): mỗi order là duy nhất

✅ UNIFORM DISTRIBUTION (Phân Bố Đều)
   - Không có một giá trị nào chiếm > 20% traffic
   - Ví dụ: UUID tự nhiên phân bố đều qua hash

❌ LOW CARDINALITY (Số Lượng Giá Trị Thấp)
   - Status: chỉ "PENDING", "PAID", "SHIPPED" — hot partition!
   - Date (YYYY-MM-DD): chỉ có vài trăm giá trị
   - Boolean: chỉ true/false — cực kỳ tệ!

❌ SKEWED ACCESS (Truy Cập Không Đều)
   - Celebrity item: một key được đọc 1 triệu lần/giây
   - Sequential ID: ID tăng dần → tất cả write vào partition cuối
```

### Chiến Lược Partition Key Phổ Biến

#### 1. UUID / ULID

```python
import uuid
user_id = str(uuid.uuid4())  # "550e8400-e29b-41d4-a716-446655440000"
```

**Ưu điểm:** Phân bố tốt tự nhiên.
**Nhược điểm:** Không có thứ tự thời gian (UUID thuần).

#### 2. Composite Partition Key Pattern

Thêm prefix hoặc suffix để tăng cardinality:

```python
# Thay vì: date = "2024-01-15"  (hot partition)
# Dùng:    shard_key = f"2024-01-15#{random.randint(0, 9)}"
# Kết quả: "2024-01-15#0", "2024-01-15#3", "2024-01-15#7"
```

#### 3. Write Sharding (Chia Nhỏ Ghi)

Khi phải dùng key có cardinality thấp (ví dụ: sensor type):

```python
SHARD_COUNT = 10
shard = hash(item_id) % SHARD_COUNT
partition_key = f"sensor-type-A#{shard}"
# Tạo ra: "sensor-type-A#0" đến "sensor-type-A#9"
```

---

## Sort Key — Khóa Sắp Xếp

### Công Dụng của Sort Key

Sort Key (Khóa Sắp Xếp) — còn gọi là Range Key (Khóa Phạm Vi) — cho phép:

1. **Sắp xếp items** trong cùng partition theo thứ tự từ điển (lexicographic)
2. **Query theo khoảng** — `BETWEEN`, `BEGINS_WITH`, `<`, `>`
3. **Tạo hierarchical data** (dữ liệu phân cấp)

### Query Patterns với Sort Key

```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Lấy TẤT CẢ orders của user-001
table.query(KeyConditionExpression=Key('UserId').eq('user-001'))

# Lấy orders từ tháng 2024-01
table.query(
    KeyConditionExpression=Key('UserId').eq('user-001') &
                            Key('OrderId').begins_with('2024-01')
)

# Lấy orders trong khoảng thời gian
table.query(
    KeyConditionExpression=Key('UserId').eq('user-001') &
                            Key('CreatedAt').between('2024-01-01', '2024-06-30')
)
```

### Sort Key Design Patterns (Mẫu Thiết Kế Sort Key)

#### Pattern 1: Timestamp Sort Key

```
Bảng: UserEvents
PK = UserId | SK = Timestamp (ISO 8601)
"user-001" | "2024-01-15T10:30:00Z" | {event: "LOGIN"}
"user-001" | "2024-01-15T11:00:00Z" | {event: "PURCHASE"}
"user-001" | "2024-02-01T09:00:00Z" | {event: "LOGIN"}

→ Query: "Tất cả events của user-001 trong tháng 1"
```

#### Pattern 2: Hierarchical Sort Key

```
Bảng: Documents
PK = TenantId | SK = "folder/subfolder/file"
"tenant-1" | "docs/reports/q1-2024.pdf"
"tenant-1" | "docs/reports/q2-2024.pdf"
"tenant-1" | "images/logo.png"

→ Query: begins_with("docs/reports/") → lấy tất cả reports
```

#### Pattern 3: Reverse Chronological Sort

Vì DynamoDB sort ascending (tăng dần) theo mặc định, để lấy items mới nhất trước:

```python
# Dùng reverse timestamp: MaxLong - timestamp
MAX_LONG = 9999999999999
sk = str(MAX_LONG - int(time.time() * 1000))
# Item mới nhất có SK nhỏ nhất → đứng đầu khi sort ascending
```

---

## Attributes — Thuộc Tính

### Đặc Điểm Attributes

- **Không bắt buộc** — mỗi item có thể có attributes khác nhau (trừ primary key)
- **Không cần khai báo trước** — thêm attribute mới bất cứ lúc nào không cần migration
- **Nested** — hỗ trợ Map và List lồng nhau đến **32 cấp độ**
- **Case-sensitive** — `"Status"` và `"status"` là hai attributes khác nhau

### Projected Attributes (Thuộc Tính Được Chiếu)

Khi dùng GSI/LSI, cần chỉ định attributes nào được copy sang index:

```
KEYS_ONLY     — Chỉ copy primary key attributes (tiết kiệm storage nhất)
INCLUDE       — Copy key + một số attributes chỉ định
ALL           — Copy tất cả attributes (tốn storage nhất)
```

### Reserved Words (Từ Khóa Dành Riêng)

Một số tên attribute bị cấm dùng trực tiếp trong expressions (biểu thức):

```python
# ❌ Sai — "name", "status", "count" là reserved words
table.update_item(
    UpdateExpression="SET name = :n"  # Lỗi!
)

# ✅ Đúng — dùng Expression Attribute Names (Tên Thuộc Tính Biểu Thức)
table.update_item(
    UpdateExpression="SET #n = :n",
    ExpressionAttributeNames={"#n": "name"},
    ExpressionAttributeValues={":n": "Alice"}
)
```

---

## Item — Mục Dữ Liệu

### Cấu Trúc Item Hoàn Chỉnh

```json
{
  "UserId": "user-001",
  "OrderId": "ord-2024-001",
  "Status": "PAID",
  "TotalAmount": 149.99,
  "Currency": "USD",
  "Items": [
    {
      "ProductId": "prod-001",
      "Name": "Wireless Mouse",
      "Qty": 2,
      "UnitPrice": 29.99
    },
    {
      "ProductId": "prod-002",
      "Name": "USB Hub",
      "Qty": 1,
      "UnitPrice": 90.01
    }
  ],
  "ShippingAddress": {
    "Street": "123 Main St",
    "City": "Ho Chi Minh City",
    "Country": "VN"
  },
  "Tags": ["electronics", "accessories"],
  "CreatedAt": "2024-01-15T10:30:00Z",
  "UpdatedAt": "2024-01-15T14:00:00Z",
  "IsDeleted": false
}
```

### CRUD Operations (Thao Tác Tạo/Đọc/Cập Nhật/Xóa)

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# CREATE (Tạo) — PutItem
table.put_item(Item={
    'UserId': 'user-001',
    'OrderId': 'ord-001',
    'Status': 'PENDING'
})

# READ (Đọc) — GetItem (truy cập theo primary key — nhanh nhất)
response = table.get_item(Key={
    'UserId': 'user-001',
    'OrderId': 'ord-001'
})
item = response.get('Item')

# UPDATE (Cập Nhật) — UpdateItem (chỉ cập nhật attributes chỉ định)
table.update_item(
    Key={'UserId': 'user-001', 'OrderId': 'ord-001'},
    UpdateExpression='SET #s = :s, UpdatedAt = :t',
    ExpressionAttributeNames={'#s': 'Status'},
    ExpressionAttributeValues={
        ':s': 'PAID',
        ':t': '2024-01-15T14:00:00Z'
    }
)

# DELETE (Xóa) — DeleteItem
table.delete_item(Key={
    'UserId': 'user-001',
    'OrderId': 'ord-001'
})

# QUERY (Truy Vấn) — query theo partition key
response = table.query(
    KeyConditionExpression=Key('UserId').eq('user-001')
)
items = response['Items']

# SCAN (Quét Toàn Bảng) — tránh dùng nếu có thể
response = table.scan(
    FilterExpression=Attr('Status').eq('PENDING')
)
```

### Conditional Writes (Ghi Có Điều Kiện)

```python
# Chỉ tạo nếu item chưa tồn tại
table.put_item(
    Item={'UserId': 'user-001', 'OrderId': 'ord-new'},
    ConditionExpression='attribute_not_exists(UserId)'
)

# Chỉ cập nhật nếu version match (Optimistic Locking — Khóa Lạc Quan)
table.update_item(
    Key={'UserId': 'user-001', 'OrderId': 'ord-001'},
    UpdateExpression='SET #s = :new_status, Version = :new_ver',
    ConditionExpression='Version = :expected_ver',
    ExpressionAttributeNames={'#s': 'Status'},
    ExpressionAttributeValues={
        ':new_status': 'SHIPPED',
        ':new_ver': 2,
        ':expected_ver': 1  # Thất bại nếu Version != 1
    }
)
```

---

## Cơ Chế Phân Vùng Nội Bộ

### Cách DynamoDB Phân Vùng Dữ Liệu

```
1. Partition Key → Hash Function → Hash Value (0 đến MaxHash)
2. Hash Value → Partition Assignment (xác định partition)
3. Partition → lưu trên 3 nodes ở 3 AZ (đảm bảo durability)

MaxHashValue / NumberOfPartitions = Range per Partition
```

### Partition Limits (Giới Hạn Partition)

Mỗi partition có giới hạn:
- **3,000 RCU** (Read Capacity Unit — Đơn Vị Năng Lực Đọc) mỗi giây
- **1,000 WCU** (Write Capacity Unit — Đơn Vị Năng Lực Ghi) mỗi giây
- **10 GB** storage

Khi vượt giới hạn, DynamoDB tự động split partition — nhưng capacity không tự tăng theo, dẫn đến throttling nếu không đủ RCU/WCU.

### Hot Partition (Phân Vùng Nóng)

```
Scenario: Bảng Songs, PK = ArtistId
- BTS (nghệ sĩ nổi tiếng): 100,000 reads/s
- Partition cho BTS: 100,000 RCU → vượt limit 3,000 RCU → THROTTLED!

Giải pháp:
1. DAX — cache phía trước để giảm reads đến DynamoDB
2. Write Sharding — thêm random suffix vào ArtistId
3. Application-level caching (ElastiCache)
```

---

## Data Types — Kiểu Dữ Liệu

### Scalar Types (Kiểu Vô Hướng)

| Kiểu    | Ký Hiệu API | Ví Dụ                          | Ghi Chú                                     |
| ------- | ----------- | ------------------------------ | ------------------------------------------- |
| String  | S           | `"hello"`, `"2024-01-15"`      | UTF-8, tối đa 400 KB                        |
| Number  | N           | `42`, `3.14`, `-100`, `1e10`   | Độ chính xác 38 chữ số                      |
| Binary  | B           | `b'\x00\x01\x02'`              | Base64 khi gọi qua API                      |
| Boolean | BOOL        | `true`, `false`                | Không thể dùng làm primary key              |
| Null    | NULL        | `null`                         | Dùng khi attribute có giá trị null rõ ràng  |

### Document Types (Kiểu Tài Liệu)

| Kiểu | Ký Hiệu API | Ví Dụ                              |
| ---- | ----------- | ---------------------------------- |
| List | L           | `[1, "two", true, {"key": "val"}]` |
| Map  | M           | `{"name": "Alice", "age": 30}`     |

**Lồng nhau tối đa 32 cấp:** Map trong List trong Map... đến 32 cấp.

### Set Types (Kiểu Tập Hợp)

| Kiểu       | Ký Hiệu API | Ghi Chú                              |
| ---------- | ----------- | ------------------------------------ |
| String Set | SS          | Tập hợp strings duy nhất, không có thứ tự |
| Number Set | NS          | Tập hợp numbers duy nhất             |
| Binary Set | BS          | Tập hợp binary values duy nhất       |

```python
# Thêm vào Set
table.update_item(
    Key={'UserId': 'user-001', 'OrderId': 'ord-001'},
    UpdateExpression='ADD #tags :new_tag',
    ExpressionAttributeNames={'#tags': 'Tags'},
    ExpressionAttributeValues={':new_tag': {'electronics', 'sale'}}
)

# Xóa khỏi Set
table.update_item(
    Key={'UserId': 'user-001', 'OrderId': 'ord-001'},
    UpdateExpression='DELETE #tags :old_tag',
    ExpressionAttributeNames={'#tags': 'Tags'},
    ExpressionAttributeValues={':old_tag': {'sale'}}
)
```

---

## Giới Hạn Quan Trọng

| Giới Hạn                          | Giá Trị                      |
| --------------------------------- | ---------------------------- |
| Kích thước item tối đa            | **400 KB**                   |
| Partition Key tối đa              | 2 KB                         |
| Sort Key tối đa                   | 1 KB                         |
| Số attributes tối đa mỗi item     | Không giới hạn (trong 400 KB)|
| Độ sâu lồng nhau tối đa           | 32 cấp                       |
| Số LSI tối đa mỗi bảng            | 5                            |
| Số GSI tối đa mỗi bảng            | 20                           |
| Kết quả Query/Scan tối đa         | 1 MB (dùng pagination)       |
| Số tables tối đa mỗi account      | 2,500 (có thể tăng)          |

---

## Thực Hành Tốt Nhất

### Thiết Kế Primary Key

```
✅ Dùng UUID v4 hoặc ULID cho unique IDs
✅ Kết hợp Partition Key + Sort Key để nhóm items liên quan
✅ Encode timestamp vào Sort Key để sort theo thời gian
✅ Dùng prefix trong Sort Key để phân loại: "ORDER#", "PROFILE#"

❌ Tránh partition key là status, date, boolean
❌ Tránh sequential integers làm PK (gây hot partition)
❌ Không dùng email làm PK nếu email có thể thay đổi
```

### Thiết Kế Attribute

```
✅ Dùng camelCase hoặc PascalCase nhất quán
✅ Lưu timestamps dạng ISO 8601: "2024-01-15T10:30:00Z"
✅ Dùng Number type cho amounts, không dùng String
✅ Dùng TTL (Time to Live — Thời Gian Sống) cho data tạm thời

❌ Không duplicate data nếu có thể tránh được
❌ Không dùng reserved words làm attribute names
❌ Tránh item > 100 KB để có latency tốt nhất
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao không nên dùng sequential integer làm Partition Key?**

A: Sequential integers (số nguyên tăng dần) như `1, 2, 3, ...` khi qua hash function của DynamoDB tạo ra hash values không đều — các giá trị liên tiếp có thể map đến cùng một partition, gây hot partition. Hơn nữa, pattern "write cuối bảng" khiến mọi insert đều hit cùng partition. Dùng UUID hoặc ULID để đảm bảo phân bố đều.

**Q: Khi nào dùng Composite Primary Key (PK + SK) thay vì Simple Primary Key (chỉ PK)?**

A: Dùng Composite Key khi: (1) cần nhóm nhiều items liên quan dưới một "parent" (ví dụ: tất cả orders của một user); (2) cần sort hay filter items trong cùng group; (3) cần hierarchical data (dữ liệu phân cấp). Simple Key đủ khi mỗi entity hoàn toàn độc lập và không cần group.

**Q: Item size limit 400 KB ảnh hưởng thiết kế như thế nào?**

A: Với items lớn (ảnh, document), không lưu binary data trực tiếp — thay vào đó lưu metadata trong DynamoDB và binary data trong S3, chỉ lưu S3 key trong DynamoDB. Với nested data phức tạp, cân nhắc flatten (làm phẳng) structure hoặc tách thành nhiều items.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
