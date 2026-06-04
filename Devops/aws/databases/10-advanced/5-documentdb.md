# 5. Amazon DocumentDB — Cơ Sở Dữ Liệu Tài Liệu Tương Thích MongoDB

> Amazon DocumentDB (with MongoDB compatibility) — Cơ Sở Dữ Liệu Tài Liệu Tương Thích MongoDB là dịch vụ **document database** (cơ sở dữ liệu tài liệu) được quản lý hoàn toàn, cung cấp khả năng tương thích với MongoDB API trong khi tận dụng kiến trúc lưu trữ phân tán của AWS.

## 📚 Mục Lục

1. [DocumentDB là Gì?](#documentdb-là-gì)
2. [Kiến Trúc DocumentDB](#kiến-trúc-documentdb)
3. [MongoDB Compatibility — Tương Thích MongoDB](#mongodb-compatibility--tương-thích-mongodb)
4. [Document Model — Mô Hình Tài Liệu](#document-model--mô-hình-tài-liệu)
5. [CRUD Operations — Thao Tác CRUD](#crud-operations--thao-tác-crud)
6. [Indexes — Chỉ Mục Trong DocumentDB](#indexes--chỉ-mục-trong-documentdb)
7. [Aggregation Pipeline — Đường Ống Tổng Hợp](#aggregation-pipeline--đường-ống-tổng-hợp)
8. [Migration Từ MongoDB](#migration-từ-mongodb)
9. [High Availability & Scaling](#high-availability--scaling)
10. [So Sánh DocumentDB vs Các Lựa Chọn Khác](#so-sánh-documentdb-vs-các-lựa-chọn-khác)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DocumentDB là Gì?

### Document Database Model (Mô Hình Cơ Sở Dữ Liệu Tài Liệu)

Document database lưu dữ liệu dưới dạng **documents** (tài liệu) — thường là JSON hoặc BSON:

```json
// Một document trong collection "orders"
{
  "_id": "order-001",
  "customer": {
    "id": "cust-123",
    "name": "Nguyen Van A",
    "email": "nguyenvana@example.com"
  },
  "items": [
    {"product_id": "prod-001", "name": "Laptop", "quantity": 1, "price": 25000000},
    {"product_id": "prod-002", "name": "Mouse",  "quantity": 2, "price": 500000}
  ],
  "total": 26000000,
  "status": "shipped",
  "created_at": "2024-01-15T10:30:00Z",
  "shipping_address": {
    "street": "123 Nguyen Hue",
    "city": "Ho Chi Minh",
    "country": "Vietnam"
  }
}
```

### Ưu Điểm So Với Relational Database

| Tiêu Chí | Relational (SQL) | Document (DocumentDB) |
|----------|------------------|-----------------------|
| **Schema** | Fixed schema (schema cố định) | Flexible schema (schema linh hoạt) |
| **Nested data** | Phải JOIN nhiều bảng | Lồng trực tiếp trong document |
| **Scale out** | Khó horizontal scale | Thiết kế cho horizontal scaling |
| **Development speed** | Schema migration phức tạp | Thêm field không cần migration |
| **Array support** | Phải dùng junction table | Native array trong document |

---

## Kiến Trúc DocumentDB

### Giống Aurora, Không Phải MongoDB

DocumentDB **không** là MongoDB chạy trên AWS. Đây là dịch vụ AWS viết lại từ đầu với:
- **MongoDB-compatible API** — ứng dụng dùng MongoDB driver vẫn hoạt động
- **Aurora-like storage** — distributed, self-healing, 6 copies qua 3 AZs

```
┌─────────────────────────────────────────────────────────────────┐
│                  DOCUMENTDB CLUSTER                              │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐│
│  │  Primary Instance│  │    Up to 15 Read Replicas            ││
│  │  (Nút Ghi Chính) │  │    (Các Nút Đọc — Tối Đa 15)        ││
│  │                  │  │                                       ││
│  │  Port: 27017     │  │  Port: 27017                         ││
│  │  (MongoDB port)  │  │  (MongoDB port)                      ││
│  └──────────────────┘  └──────────────────────────────────────┘│
│            │                          ↑ Replication             │
│            └──────────────────────────┘                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │     Cluster Storage (64 TB max, auto-grow)              │   │
│  │     6 copies across 3 AZs, self-healing                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Cluster Endpoint Types

```
Primary endpoint:
  my-cluster.cluster-abc123.us-east-1.docdb.amazonaws.com:27017

Reader endpoint (Load balanced — Cân Bằng Tải):
  my-cluster.cluster-ro-abc123.us-east-1.docdb.amazonaws.com:27017

Instance endpoint (Trực Tiếp Đến Instance):
  my-instance.abc123.us-east-1.docdb.amazonaws.com:27017
```

---

## MongoDB Compatibility — Tương Thích MongoDB

### Mức Độ Tương Thích

DocumentDB tương thích với **MongoDB 3.6, 4.0, 5.0** (tuỳ phiên bản DocumentDB). Hầu hết MongoDB drivers và tools hoạt động không cần sửa code.

### Kết Nối Bằng MongoDB Driver

```python
from pymongo import MongoClient

# Kết nối DocumentDB — cú pháp y hệt MongoDB
client = MongoClient(
    'mongodb://username:password@my-cluster.cluster-abc123.us-east-1.docdb.amazonaws.com:27017',
    tls=True,
    tlsCAFile='rds-combined-ca-bundle.pem',  # TLS certificate (chứng chỉ TLS)
    retryWrites=False  # DocumentDB không hỗ trợ retryable writes
)

db = client['myapp']
collection = db['orders']
```

### Những Gì KHÔNG Được Hỗ Trợ

| Tính Năng MongoDB | DocumentDB |
|------------------|------------|
| **Transactions** (multi-document) | Hỗ trợ từ DocumentDB 4.0 |
| **$lookup** (JOIN) | Hỗ trợ giới hạn |
| **Change Streams** | Hỗ trợ |
| **Full-text search** | Hạn chế — dùng OpenSearch thay thế |
| **GridFS** | Không hỗ trợ |
| **Capped collections** | Không hỗ trợ |
| **Map-Reduce** | Không hỗ trợ (dùng Aggregation Pipeline) |

---

## Document Model — Mô Hình Tài Liệu

### Collections và Documents

```
Database: myapp
  ├── Collection: users      (Bộ Sưu Tập: người dùng)
  │     ├── Document: {_id: "u1", name: "Alice", ...}
  │     ├── Document: {_id: "u2", name: "Bob", ...}
  │     └── ...
  ├── Collection: orders     (Bộ Sưu Tập: đơn hàng)
  └── Collection: products   (Bộ Sưu Tập: sản phẩm)
```

### Flexible Schema (Schema Linh Hoạt)

```python
# Document 1 trong users — có address
db.users.insert_one({
    "_id": "u1",
    "name": "Alice",
    "email": "alice@example.com",
    "address": {"city": "Hanoi", "country": "Vietnam"}
})

# Document 2 trong CÙNG collection users — không có address, có thêm fields
db.users.insert_one({
    "_id": "u2",
    "name": "Bob",
    "email": "bob@example.com",
    "phone": "0901234567",     # Field không có ở document 1
    "company": "TechCorp"      # Field không có ở document 1
})
# → Hoàn toàn hợp lệ, không cần ALTER TABLE
```

---

## CRUD Operations — Thao Tác CRUD

### Create (Tạo)

```python
# Insert một document
result = db.orders.insert_one({
    "customer_id": "cust-123",
    "items": [
        {"product_id": "prod-001", "quantity": 2, "price": 150000}
    ],
    "total": 300000,
    "status": "pending"
})
print(result.inserted_id)

# Insert nhiều documents cùng lúc
db.orders.insert_many([
    {"customer_id": "cust-124", "total": 500000},
    {"customer_id": "cust-125", "total": 750000}
])
```

### Read (Đọc)

```python
# Tìm theo điều kiện
orders = db.orders.find({"status": "pending"})

# Filter phức tạp — đơn hàng > 100,000 VND của customer cụ thể
orders = db.orders.find({
    "customer_id": "cust-123",
    "total": {"$gt": 100000}   # $gt = greater than (lớn hơn)
})

# Tìm trong nested document (tài liệu lồng nhau)
orders = db.orders.find({"shipping_address.city": "Hanoi"})

# Tìm trong array (mảng)
orders = db.orders.find({"items.product_id": "prod-001"})
```

### Update (Cập Nhật)

```python
# Update một document
db.orders.update_one(
    {"_id": "order-001"},
    {"$set": {"status": "shipped"}}
)

# Thêm element vào array
db.orders.update_one(
    {"_id": "order-001"},
    {"$push": {"tracking_events": {"status": "delivered", "time": "2024-01-16"}}}
)

# Increment (tăng) giá trị
db.products.update_one(
    {"_id": "prod-001"},
    {"$inc": {"stock_count": -1}}  # Giảm stock đi 1
)
```

### Delete (Xóa)

```python
# Xóa một document
db.orders.delete_one({"_id": "order-001"})

# Xóa nhiều documents theo điều kiện
db.orders.delete_many({"status": "cancelled", "created_at": {"$lt": "2023-01-01"}})
```

---

## Indexes — Chỉ Mục Trong DocumentDB

### Tạo Index

```python
# Single field index (chỉ mục một trường)
db.orders.create_index("customer_id")

# Compound index (chỉ mục kết hợp)
db.orders.create_index([("customer_id", 1), ("created_at", -1)])
# 1 = ascending (tăng dần), -1 = descending (giảm dần)

# Index trên nested field (trường lồng nhau)
db.orders.create_index("shipping_address.city")

# Index trên array element
db.orders.create_index("items.product_id")
```

### Unique Index (Chỉ Mục Duy Nhất)

```python
# Đảm bảo email là duy nhất
db.users.create_index("email", unique=True)
```

### TTL Index — Tự Động Xóa Documents

```python
# Tự động xóa sessions sau 24 giờ
db.sessions.create_index(
    "created_at",
    expireAfterSeconds=86400  # 24 giờ = 86400 giây
)
```

---

## Aggregation Pipeline — Đường Ống Tổng Hợp

Aggregation pipeline (đường ống tổng hợp) xử lý documents qua nhiều stages:

```python
# Tính doanh thu theo tháng
pipeline = [
    # Stage 1: Filter — chỉ lấy đơn hàng đã hoàn thành
    {"$match": {"status": "completed"}},
    
    # Stage 2: Group — nhóm theo tháng
    {"$group": {
        "_id": {
            "year":  {"$year": "$created_at"},
            "month": {"$month": "$created_at"}
        },
        "total_revenue": {"$sum": "$total"},
        "order_count":   {"$sum": 1}
    }},
    
    # Stage 3: Sort — sắp xếp theo tháng mới nhất
    {"$sort": {"_id.year": -1, "_id.month": -1}},
    
    # Stage 4: Limit — lấy 12 tháng gần nhất
    {"$limit": 12}
]

results = list(db.orders.aggregate(pipeline))
```

---

## Migration Từ MongoDB

### Quy Trình Migration (Di Chuyển)

```
Bước 1: Đánh giá compatibility
  → Chạy mongodump để xem collections và indexes
  → Kiểm tra features đang dùng (có GridFS, capped collections không?)
  → Sửa code nếu dùng features không hỗ trợ

Bước 2: Migration offline (downtime ngắn)
  mongodump → mongorestore trực tiếp vào DocumentDB

Bước 3: Migration online (không downtime)
  → Dùng AWS DMS với MongoDB source + DocumentDB target
  → Bật CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu)
  → Migrate initial load, sau đó sync ongoing changes
  → Cutover khi lag = 0

Bước 4: Validation
  → So sánh document count
  → Kiểm tra sample records
  → Test application queries
```

### Công Cụ Migration

```bash
# Export từ MongoDB
mongodump --host mongodb-host --db myapp --out /backup

# Import vào DocumentDB (cần TLS)
mongorestore \
  --host my-cluster.cluster-abc123.us-east-1.docdb.amazonaws.com \
  --port 27017 \
  --username admin \
  --password mypassword \
  --tls \
  --tlsCAFile rds-combined-ca-bundle.pem \
  --db myapp \
  /backup/myapp
```

---

## High Availability & Scaling

### Automated Failover (Chuyển Đổi Dự Phòng Tự Động)

- Failover tự động < 30 giây khi primary instance fail
- Read replicas được promote tự động
- Connection string dùng cluster endpoint — không cần thay đổi

### Read Scaling (Mở Rộng Đọc)

```python
from pymongo import MongoClient, ReadPreference

client = MongoClient(
    'mongodb://admin:pass@my-cluster.cluster-ro-abc123.us-east-1.docdb.amazonaws.com:27017',
    tls=True,
    tlsCAFile='rds-combined-ca-bundle.pem',
    # Đọc từ replica (secondary) để giảm tải primary
    readPreference='secondaryPreferred'
)
```

---

## So Sánh DocumentDB vs Các Lựa Chọn Khác

### DocumentDB vs MongoDB Atlas

| Tiêu Chí | DocumentDB | MongoDB Atlas |
|----------|------------|---------------|
| **Tương thích** | Tương thích một phần | 100% MongoDB |
| **Features** | Giới hạn một số features | Đầy đủ MongoDB features |
| **Quản lý** | AWS managed | MongoDB managed |
| **Pricing** | Tính theo instance + storage | Tính theo cluster tier |
| **Integration** | Tốt với AWS ecosystem | Multi-cloud |
| **Chọn khi** | Đã ở AWS, muốn managed | Cần full MongoDB features |

### DocumentDB vs DynamoDB

| Tiêu Chí | DocumentDB | DynamoDB |
|----------|------------|----------|
| **Query model** | Flexible queries, MongoDB API | Cần biết access patterns trước |
| **Schema** | Flexible, document-oriented | Flexible nhưng cần partition key |
| **Scale** | Vertical + read replicas | Unlimited horizontal scale |
| **Latency** | Low latency | Single-digit millisecond |
| **Chọn khi** | Migrate từ MongoDB, complex queries | Massive scale, predictable access patterns |

---

## Câu Hỏi Phỏng Vấn

### Q1: DocumentDB có thực sự là MongoDB không?

> **Trả lời:** Không. DocumentDB là dịch vụ AWS viết riêng với MongoDB-compatible API. Tương tự Aurora là MySQL/PostgreSQL compatible nhưng không phải MySQL/PostgreSQL. DocumentDB dùng Aurora-like distributed storage, không phải MongoDB storage engine (WiredTiger). Một số features MongoDB không được hỗ trợ (GridFS, capped collections, map-reduce). Tuy nhiên, hầu hết ứng dụng MongoDB có thể chạy trên DocumentDB với ít thay đổi.

### Q2: Khi nào nên chọn DocumentDB thay vì MongoDB Atlas?

> **Trả lời:** Chọn DocumentDB khi:
> - Team đã **all-in trên AWS**, muốn tận dụng IAM, VPC, CloudWatch ecosystem
> - Không cần full MongoDB features (GridFS, etc.)
> - Cần **managed service**, không muốn quản lý MongoDB
>
> Chọn MongoDB Atlas khi:
> - Cần **100% MongoDB compatibility**
> - Đang dùng MongoDB-specific features không có trên DocumentDB
> - Cần multi-cloud hoặc on-premise hybrid

### Q3: Document database khác gì relational database?

> **Trả lời:** Document database lưu dữ liệu dưới dạng documents (JSON/BSON) với flexible schema — không cần định nghĩa trước cấu trúc bảng. Data liên quan được **embed** (nhúng) trong một document thay vì normalize ra nhiều bảng. Tốt cho: dữ liệu hierarchical (phân cấp), schema thay đổi thường xuyên, dev speed nhanh. Không tốt cho: complex relationships, multi-entity transactions, ad-hoc joins.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
