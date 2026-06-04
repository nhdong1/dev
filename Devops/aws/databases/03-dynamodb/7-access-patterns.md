# DynamoDB — Access Patterns & Single-Table Design (Mẫu Truy Cập & Thiết Kế Đơn Bảng)

> Single-Table Design (Thiết Kế Đơn Bảng) là kỹ thuật nâng cao lưu nhiều loại entity trong một bảng DynamoDB, tận dụng composite keys (khóa tổng hợp) và GSI để phục vụ tất cả access patterns hiệu quả — đây là điểm khác biệt cốt lõi giữa DynamoDB và SQL.

## 📚 Mục Lục

1. [Tư Duy Khác Biệt: Query-First Design](#tư-duy-khác-biệt-query-first-design)
2. [Access Pattern Analysis (Phân Tích Mẫu Truy Cập)](#access-pattern-analysis-phân-tích-mẫu-truy-cập)
3. [Single-Table Design (Thiết Kế Đơn Bảng)](#single-table-design-thiết-kế-đơn-bảng)
4. [Key Design Patterns (Mẫu Thiết Kế Khóa)](#key-design-patterns-mẫu-thiết-kế-khóa)
5. [Case Study: E-Commerce](#case-study-e-commerce)
6. [Case Study: Social Media](#case-study-social-media)
7. [Adjacency List Pattern (Mẫu Danh Sách Kề)](#adjacency-list-pattern-mẫu-danh-sách-kề)
8. [Hierarchical Data Pattern (Mẫu Dữ Liệu Phân Cấp)](#hierarchical-data-pattern-mẫu-dữ-liệu-phân-cấp)
9. [Time-Series Pattern (Mẫu Chuỗi Thời Gian)](#time-series-pattern-mẫu-chuỗi-thời-gian)
10. [Multi-Table vs Single-Table Design](#multi-table-vs-single-table-design)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tư Duy Khác Biệt: Query-First Design

### SQL (Entity-First Design — Thiết Kế Ưu Tiên Entity)

```
SQL Approach:
1. Xác định entities (User, Order, Product...)
2. Normalize data (chuẩn hóa, chia thành bảng nhỏ)
3. Thiết kế schema
4. Query sau: SELECT ... JOIN ... WHERE ...

SQL có thể trả lời bất kỳ query nào (flexible queries)
nhưng JOIN = expensive ở quy mô lớn
```

### DynamoDB (Query-First Design — Thiết Kế Ưu Tiên Query)

```
DynamoDB Approach:
1. Xác định tất cả access patterns (queries) cần hỗ trợ
2. Thiết kế schema để phục vụ CHÍNH XÁC những queries đó
3. Chấp nhận data duplication nếu cần

Nguyên tắc: "Start with the question, not the data"
(Bắt đầu từ câu hỏi, không phải từ dữ liệu)
```

### Quy Tắc Vàng DynamoDB

```
"Identify your access patterns before writing a single line of code."
— Alex DeBrie, tác giả "The DynamoDB Book"

(Xác định tất cả access patterns trước khi viết bất kỳ dòng code nào)
```

---

## Access Pattern Analysis (Phân Tích Mẫu Truy Cập)

### Bước 1: Liệt Kê Tất Cả Queries

Trước khi thiết kế schema, liệt kê tất cả câu hỏi hệ thống cần trả lời:

```
E-Commerce System Access Patterns:
┌────┬─────────────────────────────────────────────┬────────────┐
│ AP │ Access Pattern                               │ Frequency  │
├────┼─────────────────────────────────────────────┼────────────┤
│ 1  │ Lấy thông tin user theo UserId               │ Very High  │
│ 2  │ Lấy tất cả orders của một user               │ High       │
│ 3  │ Lấy chi tiết một order                       │ High       │
│ 4  │ Lấy orders theo status (PENDING, PAID...)    │ Medium     │
│ 5  │ Lấy orders của user trong khoảng thời gian   │ Medium     │
│ 6  │ Lấy thông tin product theo ProductId         │ Very High  │
│ 7  │ Lấy products theo category                   │ High       │
│ 8  │ Lấy inventory (tồn kho) của product          │ High       │
│ 9  │ Lấy cart items của user                      │ Very High  │
│ 10 │ Lấy user sessions đang active               │ High       │
└────┴─────────────────────────────────────────────┴────────────┘
```

### Bước 2: Nhóm Theo Entity

```
Entities: User, Order, OrderItem, Product, Category, Inventory, Cart, Session

Relationships:
User ──has many──► Orders
Order ──has many──► OrderItems
OrderItem ──references──► Product
Product ──belongs to──► Category
User ──has one──► Cart
Cart ──has many──► CartItems
User ──has many──► Sessions
```

### Bước 3: Map Access Patterns → Key Design

```
AP1: Get user by UserId
→ PK = "USER#<UserId>", SK = "PROFILE"

AP2: Get all orders by user
→ PK = "USER#<UserId>", SK begins_with("ORDER#")

AP3: Get specific order
→ PK = "USER#<UserId>", SK = "ORDER#<OrderId>"

AP4: Get orders by status
→ GSI: PK = "STATUS#<Status>", SK = CreatedAt

AP5: Get user orders in date range
→ SK between "ORDER#2024-01" and "ORDER#2024-06"
```

---

## Single-Table Design (Thiết Kế Đơn Bảng)

### Nguyên Lý

Single-Table Design lưu **nhiều loại entity khác nhau** trong **một bảng duy nhất**, sử dụng:

1. **Generic attribute names**: PK, SK, GSI1PK, GSI1SK (thay vì UserId, OrderId)
2. **Entity prefix trong keys**: "USER#", "ORDER#", "PRODUCT#"
3. **Overloaded sort key** (Sort Key tái sử dụng): cùng SK pattern phục vụ nhiều queries
4. **GSI Overloading** (Tái Sử Dụng GSI): một GSI phục vụ nhiều entity types

### Cấu Trúc Bảng

```
Table: EcommerceTable
──────────────────────────────────────────────────────────────────
PK              | SK                | GSI1PK         | GSI1SK    | Attributes
──────────────────────────────────────────────────────────────────
USER#u-001      | PROFILE           | -              | -         | {name, email}
USER#u-001      | ORDER#2024-01#001 | STATUS#PENDING | 2024-01-15| {total, items}
USER#u-001      | ORDER#2024-02#002 | STATUS#SHIPPED | 2024-02-01| {total, items}
USER#u-001      | CART#ITEM#p-001   | -              | -         | {qty, price}
USER#u-001      | SESSION#s-001     | SESSION#ACTIVE | 2024-01-15| {token, expire}
PRODUCT#p-001   | DETAIL            | CAT#electronics| p-001     | {name, price}
PRODUCT#p-001   | INVENTORY         | -              | -         | {stock, reserved}
──────────────────────────────────────────────────────────────────

Access Pattern → DynamoDB Query:
AP1: PK="USER#u-001", SK="PROFILE"
AP2: PK="USER#u-001", SK begins_with("ORDER#")
AP3: PK="USER#u-001", SK="ORDER#2024-01#001"
AP4: GSI1: PK="STATUS#PENDING"
AP6: PK="PRODUCT#p-001", SK="DETAIL"
AP7: GSI1: PK="CAT#electronics"
AP9: PK="USER#u-001", SK begins_with("CART#ITEM#")
AP10: GSI1: PK="SESSION#ACTIVE"
```

---

## Key Design Patterns (Mẫu Thiết Kế Khóa)

### Pattern 1: Entity Prefix (Tiền Tố Entity)

```python
# Quy ước: EntityType#EntityId
# Cho phép phân biệt entity type khi query

user_pk = f"USER#{user_id}"          # "USER#u-001"
order_pk = f"ORDER#{order_id}"       # "ORDER#o-001"
product_pk = f"PRODUCT#{product_id}" # "PRODUCT#p-001"
```

**Lợi ích:**
- Tránh key collision (va chạm khóa) giữa các entity types
- Dễ dàng filter theo entity type trong GSI
- Debug dễ hơn (biết ngay đây là entity gì)

### Pattern 2: Reverse Sort Key (Sort Key Ngược)

```python
# Lấy items mới nhất trước
import time

MAX_LONG = 9999999999999
reverse_ts = MAX_LONG - int(time.time() * 1000)
sk = f"ORDER#{reverse_ts:013d}#{order_id}"

# Items mới nhất có reverse_ts NHỎ HƠN
# → Khi sort ascending, items mới nhất đứng đầu
```

### Pattern 3: Composite Sort Key (Sort Key Tổng Hợp)

```python
# Kết hợp nhiều fields vào sort key để support nhiều query conditions
sk = f"STATUS#{status}#DATE#{date}#ID#{order_id}"
# "STATUS#PENDING#DATE#2024-01-15#ID#ord-001"

# Queries được hỗ trợ:
# - All orders:        SK begins_with("STATUS#")
# - By status:         SK begins_with("STATUS#PENDING")
# - By status + date:  SK begins_with("STATUS#PENDING#DATE#2024-01")
# - Exact:             SK = "STATUS#PENDING#DATE#2024-01-15#ID#ord-001"
```

### Pattern 4: Version in Sort Key (Phiên Bản Trong Sort Key)

```python
# Lưu lịch sử thay đổi
sk_current = "ORDER#ord-001#v0"       # Version hiện tại
sk_v1 = "ORDER#ord-001#v1"            # Phiên bản lịch sử 1
sk_v2 = "ORDER#ord-001#v2"            # Phiên bản lịch sử 2

# Query: tất cả versions của order
# SK begins_with("ORDER#ord-001#v")
```

---

## Case Study: E-Commerce

### Thiết Kế Đầy Đủ

```
Table: EcommerceApp (Single Table)

──────────────────────────────────────────────────────────────────────────
Entity  | PK              | SK                 | GSI1PK       | GSI1SK
──────────────────────────────────────────────────────────────────────────
User    | USER#<UserId>   | PROFILE            | EMAIL#<email>| -
Order   | USER#<UserId>   | ORDER#<ts>#<OrdId> | STATUS#<st>  | <ts>
OrdItem | ORDER#<OrdId>   | ITEM#<ProductId>   | -            | -
Product | PRODUCT#<ProdId>| DETAIL             | CAT#<cat>    | <ProdId>
Inventory| PRODUCT#<ProdId>| INVENTORY         | -            | -
CartItem| USER#<UserId>   | CART#<ProductId>   | -            | -
Session | USER#<UserId>   | SESSION#<SessId>   | ACTIVE#      | <expire_ts>
──────────────────────────────────────────────────────────────────────────
```

### Mapping Access Patterns

```python
from boto3.dynamodb.conditions import Key

table = dynamodb.Table('EcommerceApp')

# AP1: Get user profile
user = table.get_item(
    Key={'PK': f'USER#{user_id}', 'SK': 'PROFILE'}
)['Item']

# AP2: Get all orders of a user (newest first)
orders = table.query(
    KeyConditionExpression=Key('PK').eq(f'USER#{user_id}') &
                            Key('SK').begins_with('ORDER#'),
    ScanIndexForward=False  # Newest first (reverse order)
)['Items']

# AP3: Get specific order
order = table.get_item(
    Key={'PK': f'USER#{user_id}', 'SK': f'ORDER#{timestamp}#{order_id}'}
)['Item']

# AP4: Get all PENDING orders (GSI)
pending = table.query(
    IndexName='GSI1',
    KeyConditionExpression=Key('GSI1PK').eq('STATUS#PENDING'),
    ScanIndexForward=False
)['Items']

# AP6: Get product detail
product = table.get_item(
    Key={'PK': f'PRODUCT#{product_id}', 'SK': 'DETAIL'}
)['Item']

# AP7: Get products by category (GSI)
electronics = table.query(
    IndexName='GSI1',
    KeyConditionExpression=Key('GSI1PK').eq('CAT#electronics')
)['Items']

# AP9: Get cart items
cart = table.query(
    KeyConditionExpression=Key('PK').eq(f'USER#{user_id}') &
                            Key('SK').begins_with('CART#')
)['Items']
```

---

## Case Study: Social Media

### Access Patterns

```
1. Get user profile by UserId
2. Get user's posts (newest first)
3. Get a specific post
4. Get comments on a post
5. Get posts by hashtag
6. Get followers of a user
7. Get who a user is following
8. Get user's feed (posts from people they follow)
9. Get likes count on a post
```

### Schema Design

```
Table: SocialApp
──────────────────────────────────────────────────────────────────────────
Entity   | PK              | SK                | GSI1PK      | GSI1SK
──────────────────────────────────────────────────────────────────────────
User     | USER#<UserId>   | PROFILE           | -           | -
Post     | USER#<UserId>   | POST#<rev_ts>     | HASHTAG#<h> | <ts>
Comment  | POST#<PostId>   | COMMENT#<ts>      | -           | -
Follow   | USER#<UserId>   | FOLLOWS#<TargetId>| USER#<Target>| USER#<Source>
Like     | POST#<PostId>   | LIKE#<UserId>     | -           | -
──────────────────────────────────────────────────────────────────────────

AP1: PK=USER#u1, SK=PROFILE
AP2: PK=USER#u1, SK begins_with("POST#"), ScanIndexForward=False
AP4: PK=POST#p1, SK begins_with("COMMENT#")
AP5: GSI1: PK=HASHTAG#aws, SK sorted by timestamp
AP6: GSI1: PK=USER#target → lấy tất cả users follow target (SK=USER#source)
AP7: PK=USER#u1, SK begins_with("FOLLOWS#")
AP9: PK=POST#p1, SK begins_with("LIKE#") → Count
```

---

## Adjacency List Pattern (Mẫu Danh Sách Kề)

### Dùng Cho Dữ Liệu Quan Hệ Nhiều-Nhiều (Many-to-Many)

Adjacency List lưu cả hai entity và relationship trong cùng một bảng:

```
Scenario: Student - Course (sinh viên đăng ký nhiều khóa học)
                              (khóa học có nhiều sinh viên)

Table: University
──────────────────────────────────────────────────────
PK              | SK              | Type     | Attrs
──────────────────────────────────────────────────────
STUDENT#s-001   | STUDENT#s-001   | STUDENT  | {name: "Alice"}
COURSE#c-001    | COURSE#c-001    | COURSE   | {title: "CS101"}
STUDENT#s-001   | COURSE#c-001    | ENROLL   | {enrolledAt, grade}
STUDENT#s-001   | COURSE#c-002    | ENROLL   | {enrolledAt, grade}
STUDENT#s-002   | COURSE#c-001    | ENROLL   | {enrolledAt, grade}

GSI1 (PK=SK, SK=PK):
SK              | PK              | (inverted)
──────────────────────────────────────────────
STUDENT#s-001   | STUDENT#s-001   | → Student profile
COURSE#c-001    | COURSE#c-001    | → Course detail
COURSE#c-001    | STUDENT#s-001   | → Students in course
COURSE#c-002    | STUDENT#s-001   | → Students in course
```

```python
# Lấy tất cả courses của student s-001
courses = table.query(
    KeyConditionExpression=Key('PK').eq('STUDENT#s-001') &
                            Key('SK').begins_with('COURSE#')
)['Items']

# Lấy tất cả students trong course c-001 (dùng GSI đảo ngược)
students = table.query(
    IndexName='GSI1',  # PK=SK, SK=PK
    KeyConditionExpression=Key('GSI1PK').eq('COURSE#c-001') &
                            Key('GSI1SK').begins_with('STUDENT#')
)['Items']
```

---

## Hierarchical Data Pattern (Mẫu Dữ Liệu Phân Cấp)

### Dùng Sort Key để Biểu Diễn Hierarchy

```
Scenario: Organizational chart (Sơ Đồ Tổ Chức) — công ty → phòng ban → nhân viên

PK = ORG#acme | SK = "#"                    → Organization root
PK = ORG#acme | SK = "DEPT#engineering"     → Department
PK = ORG#acme | SK = "DEPT#engineering#EMP#alice"  → Employee in dept
PK = ORG#acme | SK = "DEPT#engineering#EMP#bob"    → Employee in dept
PK = ORG#acme | SK = "DEPT#marketing#EMP#carol"    → Employee in dept

Queries:
"Tất cả departments":
  SK begins_with("DEPT#") AND NOT contains("#EMP#")

"Tất cả employees trong engineering":
  SK begins_with("DEPT#engineering#EMP#")

"Tất cả entities trong organization":
  PK = "ORG#acme"
```

### File System Structure

```
PK = "FS#user-001" | SK = "/"                     → Root directory
PK = "FS#user-001" | SK = "/documents/"           → documents folder
PK = "FS#user-001" | SK = "/documents/report.pdf" → file
PK = "FS#user-001" | SK = "/images/"              → images folder
PK = "FS#user-001" | SK = "/images/logo.png"      → file

# Lấy tất cả files trong /documents/
table.query(
    KeyConditionExpression=Key('PK').eq('FS#user-001') &
                            Key('SK').begins_with('/documents/')
)
```

---

## Time-Series Pattern (Mẫu Chuỗi Thời Gian)

### Vấn Đề: Hot Partition Trong Time-Series

Nếu dùng ngày làm partition key, ngày hiện tại luôn là hot partition:

```
❌ Bad Design:
PK = "2024-01-15" | SK = "sensor-001#10:00:00" | Value = 23.5
PK = "2024-01-15" | SK = "sensor-001#10:00:01" | Value = 23.7
...tất cả reads/writes đều vào partition "2024-01-15" (hot!)
```

### Giải Pháp: Time-Based Sharding

```python
# Phân tán traffic qua nhiều partitions
SHARD_COUNT = 10

def get_partition_key(sensor_id: str, timestamp: str) -> str:
    shard = hash(sensor_id) % SHARD_COUNT
    date = timestamp[:10]  # "2024-01-15"
    return f"SENSOR#{date}#SHARD#{shard:02d}"

# Kết quả:
# "SENSOR#2024-01-15#SHARD#03" (cho sensor-001)
# "SENSOR#2024-01-15#SHARD#07" (cho sensor-002)
# ...traffic được phân tán đều qua 10 partitions

# Để query tất cả sensors trong ngày:
# Cần query 10 partitions riêng và merge kết quả
```

### Rotate Table by Time Period (Xoay Vòng Bảng Theo Kỳ)

```
Cho data với retention policy (chính sách lưu giữ):
Tạo bảng mới mỗi tháng/quý/năm:
  - sensor_data_2024_01
  - sensor_data_2024_02
  - sensor_data_2024_03

Ưu điểm: Drop table cũ thay vì delete từng item (nhanh hơn, rẻ hơn)
Nhược điểm: Query cross-table phức tạp hơn
```

---

## Multi-Table vs Single-Table Design

### Multi-Table Design (Thiết Kế Đa Bảng)

```
Users table:
  PK = UserId | Attributes

Orders table:
  PK = OrderId | SK = UserId | Attributes

Products table:
  PK = ProductId | Attributes
```

**Ưu điểm Multi-Table:**
- Dễ hiểu hơn, giống SQL design
- Dễ query từng entity type
- Dễ set permissions riêng cho từng bảng
- Dễ monitor và debug từng bảng

**Nhược điểm Multi-Table:**
- Cần nhiều queries để lấy related data
- Không thể atomic operations (giao dịch nguyên tử) qua bảng mà không dùng Transactions
- Không tận dụng được composite sort key

### Single-Table Design (Thiết Kế Đơn Bảng)

**Ưu điểm:**
- Fetch related data trong 1 query (thay vì N queries)
- Hiệu quả hơn với DynamoDB billing model
- Tận dụng tối đa DynamoDB capabilities

**Nhược điểm:**
- Phức tạp hơn nhiều để thiết kế
- Khó debug và query ad-hoc
- Không phù hợp khi access patterns chưa rõ ràng
- Onboarding team members khó khăn hơn

### Khuyến Nghị Thực Tế

```
Dùng Single-Table Design khi:
✅ Access patterns đã rõ ràng và ổn định
✅ Team có kinh nghiệm DynamoDB
✅ Cần optimize performance (1 query thay vì N queries)
✅ Cần tiết kiệm chi phí ở scale lớn

Dùng Multi-Table Design khi:
✅ Mới bắt đầu với DynamoDB
✅ Access patterns chưa rõ ràng (sẽ thay đổi)
✅ Team quen với SQL mindset
✅ Cần audit/compliance theo từng entity type
✅ Prototype nhanh
```

---

## Câu Hỏi Phỏng Vấn

**Q: Single-Table Design là gì và tại sao được khuyến nghị cho DynamoDB?**

A: Single-Table Design là kỹ thuật lưu nhiều loại entity (User, Order, Product...) trong một bảng DynamoDB, sử dụng entity prefix trong keys ("USER#", "ORDER#") và GSI overloading. Được khuyến nghị vì: (1) có thể fetch related data trong 1 query thay vì N queries; (2) atomic operations dễ hơn (không cần transactions cross-table); (3) tận dụng composite sort key để support nhiều access patterns. Nhưng đòi hỏi access patterns phải xác định trước — không phù hợp khi requirements chưa rõ.

**Q: Giải thích Access Pattern-First Design trong DynamoDB?**

A: Khác với SQL, DynamoDB không hỗ trợ flexible ad-hoc queries — chỉ query theo primary key hoặc index. Vì vậy phải xác định tất cả access patterns (câu hỏi hệ thống cần trả lời) trước khi thiết kế schema. Với mỗi access pattern, thiết kế primary key hoặc GSI để phục vụ nó hiệu quả. Schema được thiết kế để trả lời những câu hỏi cụ thể đó — thay đổi access patterns sau khi đã có data có thể yêu cầu migration phức tạp.

**Q: Làm sao xử lý many-to-many relationships trong DynamoDB?**

A: Dùng Adjacency List Pattern — lưu cả hai entity và relationship trong cùng một bảng. Ví dụ: Student-Course; lưu `PK=STUDENT#s1, SK=COURSE#c1` cho enrollment. Dùng GSI với PK và SK hoán đổi (GSI: PK=SK, SK=PK) để query từ cả hai chiều: "courses của student" (base table: PK=STUDENT#s1, SK begins_with COURSE#) và "students của course" (GSI: PK=COURSE#c1, SK begins_with STUDENT#).

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
