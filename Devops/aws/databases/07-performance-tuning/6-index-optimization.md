# Index Optimization — Tối Ưu Chỉ Mục Database

> Index (chỉ mục) là công cụ tối ưu hiệu năng mạnh nhất trong cơ sở dữ liệu. Thiết kế index tốt có thể giảm query time từ vài phút xuống còn mili-giây. Topic này bao gồm index design cho SQL (RDS, Aurora) và GSI optimization (tối ưu chỉ mục phụ toàn cầu) cho DynamoDB.

## 📚 Mục Lục

1. [Tổng Quan Index](#tổng-quan-index)
2. [Index Types — Các Loại Chỉ Mục (SQL)](#index-types--các-loại-chỉ-mục-sql)
3. [Composite Index — Chỉ Mục Tổng Hợp](#composite-index--chỉ-mục-tổng-hợp)
4. [Covering Index — Chỉ Mục Bao Phủ](#covering-index--chỉ-mục-bao-phủ)
5. [Index Best Practices — Thực Hành Tốt Nhất](#index-best-practices--thực-hành-tốt-nhất)
6. [DynamoDB GSI Optimization — Tối Ưu GSI](#dynamodb-gsi-optimization--tối-ưu-gsi)
7. [Index Maintenance — Bảo Trì Chỉ Mục](#index-maintenance--bảo-trì-chỉ-mục)
8. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Tổng Quan Index

### Index Là Gì và Hoạt Động Thế Nào?

Index là cấu trúc dữ liệu bổ sung (thường là B-Tree — Cây Cân Bằng) cho phép database tìm kiếm nhanh mà không cần đọc toàn bộ bảng.

```
Bảng users (1 triệu rows) — Không có index:
SELECT * FROM users WHERE email = 'alice@example.com'
→ Phải quét 1,000,000 rows  (O(n))
→ Thời gian: ~2-5 giây

Với index trên email:
SELECT * FROM users WHERE email = 'alice@example.com'
→ B-Tree lookup: log2(1,000,000) ≈ 20 bước  (O(log n))
→ Thời gian: ~0.1ms

B-Tree Index Structure:
                    [M-R]
                   /     \
           [A-L]           [S-Z]
          /     \           /   \
       [A-D]  [E-L]     [S-P]  [T-Z]
       ...                        ...
       alice@         ─────────────► Row pointer: page 1234, offset 56
```

### Index Trade-offs — Đánh Đổi

| Lợi Ích | Chi Phí |
|---------|--------|
| Query nhanh hơn (SELECT) | Write chậm hơn (INSERT, UPDATE, DELETE) |
| Tránh full table scan | Chiếm thêm storage |
| Hỗ trợ ORDER BY hiệu quả | Cần maintain (fragmentation — phân mảnh) |
| Unique constraint enforcement | Lock contention khi tạo index trên bảng lớn |

---

## 📊 Index Types — Các Loại Chỉ Mục (SQL)

### 1. Primary Key Index (Chỉ Mục Khóa Chính)

Mỗi bảng chỉ có 1 primary key — tự động tạo clustered index (InnoDB) hoặc unique index.

```sql
-- Clustered index (InnoDB): dữ liệu bảng được sắp xếp theo primary key
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    total DECIMAL(10,2),
    created_at DATETIME
);

-- SELECT theo primary key: cực kỳ nhanh
SELECT * FROM orders WHERE id = 12345;
-- Execution: B-Tree lookup trực tiếp vào data page
```

### 2. Secondary Index — Chỉ Mục Phụ

```sql
-- Non-unique index (chỉ mục không duy nhất)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Unique index (chỉ mục duy nhất)
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Partial index — PostgreSQL only (chỉ mục một phần)
-- Chỉ index rows thỏa mãn condition → nhỏ hơn, nhanh hơn
CREATE INDEX idx_orders_pending ON orders(customer_id)
WHERE status = 'pending';

-- Xem indexes hiện có (MySQL)
SHOW INDEXES FROM orders;

-- Xem indexes (PostgreSQL)
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

### 3. Full-Text Index — Chỉ Mục Toàn Văn

```sql
-- MySQL Full-Text Index
ALTER TABLE products ADD FULLTEXT INDEX ft_description(name, description);

-- Dùng trong query
SELECT * FROM products
WHERE MATCH(name, description) AGAINST('laptop gaming' IN BOOLEAN MODE);

-- PostgreSQL Full-Text Search
CREATE INDEX idx_products_fts ON products
USING GIN(to_tsvector('english', name || ' ' || description));

SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description)
      @@ to_tsquery('english', 'laptop & gaming');
```

### 4. Functional/Expression Index — Chỉ Mục Biểu Thức

```sql
-- MySQL 8.0+ và PostgreSQL: index trên expression
-- Hữu ích khi cần query theo function result

-- PostgreSQL
CREATE INDEX idx_users_lower_email ON users(lower(email));
-- Query dùng được index:
SELECT * FROM users WHERE lower(email) = 'alice@example.com';

-- MySQL 8.0+
CREATE INDEX idx_year_created ON orders((YEAR(created_at)));
-- Query dùng được index:
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
```

### 5. Spatial Index — Chỉ Mục Không Gian

```sql
-- MySQL — GeoSpatial index
CREATE SPATIAL INDEX idx_location ON stores(location);

-- Query theo khoảng cách
SELECT *, ST_Distance(location, ST_GeomFromText('POINT(106.8 10.8)')) AS dist
FROM stores
ORDER BY dist
LIMIT 10;
```

---

## 🔗 Composite Index — Chỉ Mục Tổng Hợp

### Nguyên Tắc Leftmost Prefix — Tiền Tố Bên Trái

Composite index (chỉ mục nhiều cột) hoạt động theo nguyên tắc leftmost prefix: query chỉ dùng được index nếu bắt đầu từ cột đầu tiên của index.

```sql
-- Index tổng hợp: (status, customer_id, created_at)
CREATE INDEX idx_orders_compound ON orders(status, customer_id, created_at);

-- ✅ DÙNG ĐƯỢC INDEX đầy đủ
SELECT * FROM orders
WHERE status = 'pending' AND customer_id = 42 AND created_at > '2024-01-01';

-- ✅ DÙNG ĐƯỢC INDEX (chỉ 2 cột đầu)
SELECT * FROM orders
WHERE status = 'pending' AND customer_id = 42;

-- ✅ DÙNG ĐƯỢC INDEX (chỉ cột đầu)
SELECT * FROM orders WHERE status = 'pending';

-- ❌ KHÔNG DÙNG ĐƯỢC INDEX (bỏ qua cột đầu)
SELECT * FROM orders WHERE customer_id = 42;
-- → Full table scan hoặc dùng index khác

-- ❌ KHÔNG DÙNG ĐƯỢC INDEX (bỏ qua cột giữa)
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';
-- → Chỉ dùng được prefix "status", phải filter "created_at" thủ công
```

### Thứ Tự Cột Trong Composite Index

**Quy tắc chung:**
1. Equality conditions (điều kiện bằng nhau) trước, range conditions (điều kiện khoảng) sau
2. Cột có selectivity (độ chọn lọc) cao trước, thấp sau (gây tranh cãi — xem bên dưới)
3. Cột trong ORDER BY ở cuối nếu cần avoid filesort

```sql
-- Query phổ biến:
SELECT * FROM orders
WHERE status = 'pending'       -- Equality — ít giá trị (3 trạng thái)
AND customer_id = 42           -- Equality — nhiều giá trị (millions)
AND created_at > '2024-01-01'  -- Range
ORDER BY created_at DESC;

-- Index tối ưu:
-- status trước (equality), customer_id sau (equality), created_at cuối (range + sort)
CREATE INDEX idx ON orders(status, customer_id, created_at);

-- Thứ tự cột và ORDER BY:
-- created_at trong index giúp tránh filesort khi ORDER BY created_at
```

### Ví Dụ Thực Tế: E-commerce Orders

```sql
-- Bảng orders với các access patterns phổ biến:

-- Pattern 1: Xem orders của 1 customer
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
→ Index: (customer_id, created_at)

-- Pattern 2: Admin xem orders theo status và ngày
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';
→ Index: (status, created_at)

-- Pattern 3: Report theo region và status
SELECT COUNT(*), SUM(total) FROM orders
WHERE region = 'VN' AND status = 'completed'
AND created_at BETWEEN '2024-01-01' AND '2024-12-31';
→ Index: (region, status, created_at)

-- Cần tạo 3 indexes riêng — mỗi index phục vụ 1 pattern
-- Không thể dùng 1 index cho tất cả
```

---

## 🎯 Covering Index — Chỉ Mục Bao Phủ

### Covering Index Là Gì?

Covering index (chỉ mục bao phủ) là index chứa tất cả columns mà query cần, cho phép database trả lời query **chỉ từ index** mà không cần đọc table (table access = I/O tốn kém nhất).

```
Thông thường (Secondary Index):
Query → Index lookup → Row pointer → Table access → Return data
                                     (Expensive I/O)

Với Covering Index:
Query → Index lookup → Return data (from index itself)
        (No table access needed!)
```

```sql
-- Query cần: id, name, price
-- Không có covering index:
SELECT id, name, price FROM products WHERE category = 'electronics';
EXPLAIN: type=ref, Extra=Using where
-- Index lookup → table access cho mỗi row matched

-- Với covering index (category + tất cả cột cần):
CREATE INDEX idx_products_covering ON products(category, id, name, price);
SELECT id, name, price FROM products WHERE category = 'electronics';
EXPLAIN: type=ref, Extra=Using index  ← Chỉ dùng index!
-- Không cần table access

-- PostgreSQL — Index Only Scan
EXPLAIN (ANALYZE) SELECT id, name, price FROM products WHERE category = 'electronics';
-- Output: Index Only Scan using idx_products_covering on products
--         Heap Fetches: 0  ← Không đọc table
```

### Include Columns (SQL Server / PostgreSQL 11+)

```sql
-- PostgreSQL 11+: thêm "included columns" vào index
-- Cột include không ảnh hưởng sort order nhưng được stored trong index
CREATE INDEX idx_orders_status_include ON orders(status)
INCLUDE (customer_id, total, created_at);

-- Giúp query:
SELECT customer_id, total, created_at FROM orders WHERE status = 'pending';
-- → Index Only Scan, không cần table access
-- Nhưng không hỗ trợ: ORDER BY customer_id (customer_id chỉ là included, không phải key)
```

---

## ✅ Index Best Practices — Thực Hành Tốt Nhất

### Khi Nào Thêm Index

```sql
-- ✅ Thêm index khi:
-- 1. Cột xuất hiện thường xuyên trong WHERE clause
-- 2. Cột dùng trong JOIN condition
-- 3. Cột dùng trong ORDER BY hoặc GROUP BY
-- 4. Cột có high cardinality (nhiều unique values)
-- 5. EXPLAIN cho thấy Full Table Scan trên bảng lớn

-- ❌ Không thêm index khi:
-- 1. Bảng nhỏ (< 1000 rows) — full scan thường nhanh hơn
-- 2. Cột low cardinality — Boolean, gender (index không selective)
-- 3. Bảng write-heavy — index làm chậm INSERT/UPDATE/DELETE
-- 4. Cột trong query ít dùng
-- 5. Đã có covering index bao gồm cột này
```

### Audit Index Usage (Kiểm Tra Index Được Dùng Không)

```sql
-- MySQL — Tìm unused indexes
SELECT *
FROM sys.schema_unused_indexes
WHERE object_schema = 'myapp'
ORDER BY object_name, index_name;

-- MySQL — Index usage statistics
SELECT object_schema, object_name, index_name,
       count_read, count_write
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'myapp'
ORDER BY count_read ASC;  -- count_read = 0 → potentially unused

-- PostgreSQL — Index scan stats
SELECT schemaname, tablename, indexname,
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- idx_scan = 0 → possibly unused

-- PostgreSQL — Indexes không được dùng
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND NOT indexname LIKE '%pkey%'  -- Loại trừ primary keys
ORDER BY schemaname, tablename;
```

### Index Bloat — Phân Mảnh Chỉ Mục

```sql
-- PostgreSQL — Kiểm tra index bloat
SELECT
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild fragmented index (PostgreSQL)
-- Không lock table:
REINDEX CONCURRENTLY INDEX idx_orders_customer_id;

-- MySQL — Analyze và optimize
ANALYZE TABLE orders;
OPTIMIZE TABLE orders;  -- Cẩn thận: lock table trên MyISAM
```

---

## 🗄️ DynamoDB GSI Optimization — Tối Ưu GSI

### GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) Là Gì?

GSI cho phép query DynamoDB theo partition key khác với table's primary key. Đây là cơ chế chính để hỗ trợ nhiều access patterns trong DynamoDB.

```
Table: Orders
PK: order_id (UUID)
SK: customer_id

Không có GSI:
- Lấy order_id biết → O(1)  ✅
- Lấy orders của customer_id → Scan toàn bảng  ❌

Với GSI (gsi_customer_orders):
GSI PK: customer_id
GSI SK: created_at

Bây giờ:
- Query "tất cả orders của customer_id=42" → O(log n)  ✅
```

### Thiết Kế GSI

```python
# Tạo table với GSI
import boto3

dynamodb = boto3.resource('dynamodb')

table = dynamodb.create_table(
    TableName='Orders',
    KeySchema=[
        {'AttributeName': 'order_id', 'KeyType': 'HASH'},
    ],
    AttributeDefinitions=[
        {'AttributeName': 'order_id', 'AttributeType': 'S'},
        {'AttributeName': 'customer_id', 'AttributeType': 'S'},
        {'AttributeName': 'created_at', 'AttributeType': 'S'},
        {'AttributeName': 'status', 'AttributeType': 'S'},
    ],
    GlobalSecondaryIndexes=[
        {
            'IndexName': 'gsi-customer-orders',
            'KeySchema': [
                {'AttributeName': 'customer_id', 'KeyType': 'HASH'},
                {'AttributeName': 'created_at', 'KeyType': 'RANGE'},
            ],
            'Projection': {'ProjectionType': 'ALL'},
            'BillingMode': 'PAY_PER_REQUEST'
        },
        {
            'IndexName': 'gsi-status-created',
            'KeySchema': [
                {'AttributeName': 'status', 'KeyType': 'HASH'},
                {'AttributeName': 'created_at', 'KeyType': 'RANGE'},
            ],
            'Projection': {
                'ProjectionType': 'INCLUDE',
                # Chỉ include cột cần cho query này (tiết kiệm storage & write cost)
                'NonKeyAttributes': ['order_id', 'customer_id', 'total']
            }
        }
    ],
    BillingMode='PAY_PER_REQUEST'
)
```

### GSI Projection Types — Loại Chiếu GSI

| Projection | Nội Dung | Khi Nào Dùng |
|-----------|---------|-------------|
| `KEYS_ONLY` | Chỉ PK/SK của table và GSI | Khi chỉ cần key để fetch từ table |
| `INCLUDE` | KEYS_ONLY + các cột chỉ định | Cần một số cột nhất định, tiết kiệm chi phí |
| `ALL` | Toàn bộ item | Query hoàn chỉnh từ GSI mà không cần table fetch |

**Chi phí Write Projection:**
```
Mỗi item được ghi vào table → được ghi lại vào tất cả GSI

Ví dụ: 100 WCU/giây vào table với 3 GSI ALL projection:
→ Thực tế dùng 400 WCU/giây (table + 3 GSI copies)
→ Chi phí ghi tăng 4×

Tối ưu:
- Dùng INCLUDE thay ALL khi chỉ cần một số cột
- Xóa GSI không cần thiết
- Chọn projection nhỏ nhất đủ dùng cho query
```

### GSI Overloading — Tái Sử Dụng GSI

Kỹ thuật sử dụng 1 GSI cho nhiều access patterns bằng cách "overload" (nạp chồng) attribute values:

```python
# GSI "gsi-type-id" được dùng cho nhiều mục đích:
# GSI PK: entity_type  (ví dụ: "ORDER", "INVOICE", "CUSTOMER")
# GSI SK: entity_id

# Orders:
table.put_item(Item={
    'pk': 'ORDER#001',
    'sk': 'METADATA',
    'entity_type': 'ORDER',     # GSI PK
    'entity_id': '2024-01-15',  # GSI SK — có thể là date, status, etc.
    'total': 1500000,
    'customer_id': 'CUST#42'
})

# Invoices:
table.put_item(Item={
    'pk': 'INVOICE#001',
    'sk': 'METADATA',
    'entity_type': 'INVOICE',   # GSI PK
    'entity_id': '2024-01-15',  # GSI SK
    'amount': 1500000,
})

# Query: "Tất cả ORDERS trong ngày 2024-01-15"
response = table.query(
    IndexName='gsi-type-id',
    KeyConditionExpression=Key('entity_type').eq('ORDER')
        & Key('entity_id').begins_with('2024-01-15')
)
```

### LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ) vs GSI

| | LSI | GSI |
|-|-----|-----|
| Partition key | Giống table | Khác table |
| Tạo khi nào | Chỉ khi tạo table | Bất cứ lúc nào |
| Consistency | Strong hoặc eventually | Eventually only |
| Storage limit | 10 GB/partition key | Không giới hạn |
| Capacity | Dùng chung với table | Capacity riêng |
| Dùng khi | Cần sort theo attribute khác trong cùng partition | Cần query theo partition key khác |

```python
# LSI — query trong cùng partition key (customer_id)
# nhưng sort theo order_total thay vì created_at

# Tạo LSI khi create table:
GlobalSecondaryIndexes=[...],  # GSI
LocalSecondaryIndexes=[
    {
        'IndexName': 'lsi-by-total',
        'KeySchema': [
            {'AttributeName': 'customer_id', 'KeyType': 'HASH'},  # Giống table PK
            {'AttributeName': 'order_total', 'KeyType': 'RANGE'},  # Khác sort key
        ],
        'Projection': {'ProjectionType': 'ALL'}
    }
],
```

### Sparse Index — Chỉ Mục Thưa

DynamoDB chỉ đưa item vào GSI khi item có attribute của GSI PK. Dùng để tạo "filtered view":

```python
# Chỉ orders có trạng thái cần xử lý được đưa vào GSI
# Khi order hoàn thành, xóa attribute "needs_processing"

# Order chưa xử lý:
table.put_item(Item={
    'pk': 'ORDER#001',
    'sk': 'METADATA',
    'needs_processing': 'true',  # ← Xuất hiện trong GSI "gsi-needs-processing"
    'total': 1500000
})

# Sau khi xử lý xong: xóa attribute → item tự động bị xóa khỏi GSI
table.update_item(
    Key={'pk': 'ORDER#001', 'sk': 'METADATA'},
    UpdateExpression='REMOVE needs_processing'
)

# GSI "gsi-needs-processing" chỉ chứa orders chưa xử lý:
# → Query GSI để tìm việc cần làm
response = table.query(
    IndexName='gsi-needs-processing',
    KeyConditionExpression=Key('needs_processing').eq('true')
)
```

---

## 🔧 Index Maintenance — Bảo Trì Chỉ Mục

### Index Health Check (MySQL)

```sql
-- Kiểm tra duplicate indexes (chỉ mục trùng)
SELECT *
FROM sys.schema_redundant_indexes
WHERE table_schema = 'myapp';

-- Ví dụ redundant:
-- idx_a: (customer_id)
-- idx_b: (customer_id, created_at)  ← idx_a là redundant, vì idx_b cover idx_a

-- Tìm large indexes
SELECT
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE stat_name = 'size'
AND database_name = 'myapp'
ORDER BY size_mb DESC;
```

### Index Health Check (PostgreSQL)

```sql
-- Unused indexes có thể xóa
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
       idx_scan AS number_of_scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND NOT indisprimary
  AND NOT indisunique
ORDER BY pg_relation_size(indexrelid) DESC;

-- Bloated indexes (cần REINDEX)
SELECT indexrelid::regclass AS index_name,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_index
WHERE pg_relation_size(indexrelid) > 1000000  -- > 1MB
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild indexes với bloat
-- CONCURRENT — không lock reads/writes
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;

-- Hoặc VACUUM để reclaim space
VACUUM (ANALYZE) orders;
```

### DynamoDB GSI Capacity Planning

```bash
# Xem GSI capacity usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions \
    Name=TableName,Value=Orders \
    Name=GlobalSecondaryIndexName,Value=gsi-customer-orders \
  --period 3600 \
  --statistics Sum,Maximum \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z

# Nếu GSI thường xuyên throttle:
# 1. Tăng capacity riêng cho GSI
# 2. Xem xét On-Demand mode cho cả table
# 3. Redesign GSI projection (giảm data size)
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Khi nào nên tạo composite index thay vì nhiều single-column indexes?

```
Composite index tốt hơn khi:
1. Query thường xuyên dùng cả hai columns cùng lúc:
   WHERE status = 'active' AND customer_id = 42
   → Composite (status, customer_id) tốt hơn 2 index riêng

2. Cần support cả WHERE và ORDER BY:
   WHERE status = 'active' ORDER BY created_at
   → Composite (status, created_at) tránh filesort

3. Covering index: tất cả cột cần trong 1 index

Single-column indexes tốt hơn khi:
1. Queries dùng từng cột độc lập nhau
2. Cần linh hoạt — DB optimizer có thể combine bằng index merge
3. Mỗi cột có access pattern riêng

Quy tắc ngón tay cái:
- 5-7 indexes tối đa cho bảng OLTP write-heavy
- Prioritize indexes cho WHERE/JOIN/ORDER BY của queries quan trọng nhất
- Mỗi index cần justify bằng slow query data — không tạo "phòng ngừa"
```

### Q: DynamoDB có bao nhiêu GSI tối đa và strategy để giảm số GSI?

```
Giới hạn DynamoDB:
- Tối đa 20 GSI mỗi table (có thể request tăng)
- Tối đa 5 LSI mỗi table (không thay đổi được sau khi tạo)

Strategies để giảm số GSI:

1. GSI Overloading (phổ biến nhất):
   - 1 GSI với generic attribute names: pk2, sk2
   - Nhiều entity types dùng chung 1 GSI
   - pk2 = "USER#42", pk2 = "ORDER#001" — cùng 1 GSI

2. Sparse Index:
   - Chỉ items relevant được đưa vào GSI
   - Giảm storage và write cost

3. Single-table design với overloaded PK/SK:
   - Thiết kế cẩn thận PK/SK để support nhiều patterns
   - Giảm nhu cầu GSI

4. Write-time denormalization (phi chuẩn hóa khi ghi):
   - Ghi dữ liệu theo nhiều access patterns vào cùng bảng
   - Ví dụ: ghi event vào pk="USER#42|2024-01" và pk="GLOBAL|2024-01"

5. DynamoDB Streams + pre-computed aggregates:
   - Tính trước các aggregates và lưu vào separate items
   - Thay vì GSI để scan/aggregate, đọc pre-computed value
```

### Q: Covering index giúp ích như thế nào và tradeoff là gì?

```
Lợi ích:
1. Index Only Scan / Using index:
   - Database đọc chỉ từ index, không cần table I/O
   - Nhanh hơn 5-10× so với phải đọc table

2. Giảm I/O dramatically:
   - Index thường nhỏ hơn table nhiều (chỉ chứa indexed columns)
   - Fit vào cache tốt hơn (buffer pool / page cache)

3. Parallel scan friendly:
   - Index-only scans có thể parallel hơn

Trade-offs:
1. Index lớn hơn:
   - Mỗi cột thêm vào = thêm storage trong index
   - Ảnh hưởng write amplification

2. Write overhead:
   - Mỗi INSERT/UPDATE/DELETE phải update index với tất cả included columns
   - Càng nhiều covered columns → write chậm hơn

3. Index maintenance:
   - Covering index phải được rebuild khi schema thay đổi

Practical advice:
- Covering index hiệu quả nhất cho read-heavy, infrequently updated data
- Report queries, analytics: covering index rất có giá trị
- OLTP với high write rate: cân nhắc kỹ trước khi thêm covered columns
- Đo thực tế trước và sau khi thêm covering index
```

---

**Hoàn Thành Topic 07:** [README.md](README.md) — Quay lại tổng quan

**Chủ Đề Liên Quan:**
- [08-migration](../08-migration/) — Database Migration
- [09-monitoring](../09-monitoring/) — Monitoring & Observability
- [03-dynamodb](../03-dynamodb/3-indexes.md) — DynamoDB Index Design chi tiết

**Cập Nhật Lần Cuối:** 2026-05-15
