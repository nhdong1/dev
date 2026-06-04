# Các Loại Cơ Sở Dữ Liệu & Lựa Chọn Phù Hợp

## Tổng Quan

Các loại CSDL khác nhau phục vụ cho các mục đích khác nhau. Việc chọn đúng loại là điều then chốt cho hiệu suất và khả năng mở rộng hệ thống.

---

## Phân Loại CSDL

### 1. Quan Hệ (RDBMS)

**Đặc điểm:**
- Dữ liệu có cấu trúc (bảng, hàng, cột)
- Đảm bảo ACID
- Ngôn ngữ SQL
- JOIN giữa các bảng
- Index cho hiệu suất

**Tốt nhất cho:**
- ✓ Hệ thống giao dịch (ngân hàng, đơn hàng)
- ✓ Truy vấn phức tạp với JOIN
- ✓ Tính toàn vẹn dữ liệu là quan trọng
- ✓ Schema được định nghĩa rõ ràng, ổn định

**Các CSDL phổ biến:**

| CSDL           | Ưu điểm                                    | Nhược điểm                   | Tốt nhất cho             |
| -------------- | ------------------------------------------ | ----------------------------- | ------------------------ |
| **PostgreSQL** | Tính năng nâng cao, JSONB, tìm kiếm toàn văn | Ghi hơi chậm hơn          | Truy vấn phức tạp        |
| **MySQL**      | Đọc nhanh, đơn giản, phổ biến              | Tính năng hạn chế             | Ứng dụng web             |
| **SQL Server** | Tính năng doanh nghiệp, tích hợp Windows   | Giấy phép đắt tiền            | Doanh nghiệp lớn         |
| **Oracle**     | Mở rộng và hiệu suất tối đa                | Rất đắt, phức tạp             | Tổ chức tài chính        |

**Ví dụ (PostgreSQL):**

```sql
-- Schema có cấu trúc
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  total DECIMAL(10, 2),
  created_at TIMESTAMP
);

-- Transaction ACID
BEGIN;
  INSERT INTO users VALUES (DEFAULT, 'john@example.com', NOW());
  INSERT INTO orders VALUES (DEFAULT, 1, 99.99, NOW());
COMMIT;

-- JOIN phức tạp
SELECT u.email, COUNT(o.id) as so_don_hang
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING COUNT(o.id) > 5;
```

---

### 2. NoSQL — Tài Liệu (MongoDB, Firebase)

**Đặc điểm:**
- Schema linh hoạt (tài liệu JSON)
- Mở rộng ngang
- Dữ liệu phi chuẩn hóa
- Không có ACID ban đầu, ACID một tài liệu (4.0+)

**Tốt nhất cho:**
- ✓ Schema linh hoạt/thay đổi thường xuyên
- ✓ Dữ liệu bán cấu trúc
- ✓ Mở rộng ngang
- ✓ Ứng dụng real-time

**Ví dụ MongoDB:**

```javascript
// Schema linh hoạt
db.users.insertOne({
  _id: ObjectId("..."),
  email: "john@example.com",
  profile: {
    firstName: "John",
    lastName: "Doe",
  },
  tags: ["premium", "early-adopter"],
  settings: {
    notifications: true,
    theme: "dark",
  },
});

// Truy vấn với aggregation
db.users.aggregate([
  { $match: { "profile.firstName": "John" } },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]);
```

**Ưu điểm:**
- ✓ Linh hoạt schema (tốt cho phát triển nhanh)
- ✓ Mở rộng ngang
- ✓ Đọc nhanh (dữ liệu phi chuẩn hóa)

**Nhược điểm:**
- ❌ Dữ liệu trùng lặp
- ❌ Không có JOIN (phải phi chuẩn hóa)
- ❌ Vấn đề eventual consistency
- ❌ Transaction đa tài liệu phức tạp

---

### 3. NoSQL — Key-Value (Redis, Memcached)

**Đặc điểm:**
- Ánh xạ key → value đơn giản
- In-memory (thường)
- Rất nhanh
- Khả năng truy vấn hạn chế
- Hỗ trợ TTL

**Tốt nhất cho:**
- ✓ Caching
- ✓ Lưu session
- ✓ Rate limiting
- ✓ Dữ liệu real-time (bảng xếp hạng, bộ đếm)
- ✓ Message queues

**Ví dụ Redis:**

```bash
# Key-value đơn giản
SET user:1 '{"name":"John","email":"john@example.com"}'
GET user:1

# Bộ đếm
INCR luot_xem
GET luot_xem  # 1
INCR luot_xem
GET luot_xem  # 2

# Hết hạn (cache)
SET session:abc123 '{...}' EX 3600  # Hết hạn sau 1 giờ

# Cấu trúc dữ liệu
LPUSH queue:jobs '{"task":"email"}'
RPOP queue:jobs

# Bảng xếp hạng
ZADD leaderboard 100 user:1
ZADD leaderboard 150 user:2
ZRANGE leaderboard 0 -1 WITHSCORES
```

---

### 4. NoSQL — Time-Series (InfluxDB, TimescaleDB)

**Đặc điểm:**
- Tối ưu cho dữ liệu có timestamp
- Nén dữ liệu
- Aggregation nhanh
- Truy vấn dựa trên thời gian đặc biệt

**Tốt nhất cho:**
- ✓ Metrics (CPU, bộ nhớ)
- ✓ Dữ liệu giám sát
- ✓ Biểu đồ tài chính
- ✓ Dữ liệu cảm biến IoT

---

### 5. Search Engine (Elasticsearch)

**Đặc điểm:**
- Tối ưu cho tìm kiếm toàn văn
- Truy vấn văn bản phức tạp
- Phân tán
- Real-time indexing

**Tốt nhất cho:**
- ✓ Tìm kiếm toàn văn
- ✓ Phân tích log
- ✓ Autocomplete

---

## Cây Quyết Định Lựa Chọn CSDL

```
Bạn có cần ACID không?
├─ CÓ → RDBMS (PostgreSQL/MySQL/SQL Server)
└─ KHÔNG → Tiếp tục...

Bạn có cần truy vấn phức tạp không?
├─ CÓ → RDBMS hoặc Search (Elasticsearch)
└─ KHÔNG → Tiếp tục...

Dữ liệu có cấu trúc & cố định không?
├─ CÓ → RDBMS
└─ KHÔNG → Tiếp tục...

Cần mở rộng ngang không?
├─ CÓ → NoSQL (MongoDB, Cassandra)
└─ KHÔNG → RDBMS

Dữ liệu là metrics có timestamp không?
├─ CÓ → Time-series DB (InfluxDB)
└─ KHÔNG → Tiếp tục...

Cần tìm kiếm toàn văn không?
├─ CÓ → Elasticsearch
└─ KHÔNG → Tiếp tục...

Caching/session không?
├─ CÓ → Redis
└─ KHÔNG → Đánh giá với đội nhóm
```

---

## RDBMS vs NoSQL — Đánh Đổi

| Khía cạnh          | RDBMS               | NoSQL                |
| ------------------ | ------------------- | -------------------- |
| **Schema**         | Cố định             | Linh hoạt            |
| **Nhất quán**      | ACID                | Eventual             |
| **Mở rộng**        | Dọc (khó)           | Ngang (dễ)           |
| **Join**           | Nhanh               | Thủ công (phi chuẩn hóa) |
| **Transaction**    | Mạnh                | Hạn chế/Không có     |
| **Ngôn ngữ truy vấn** | SQL (tiêu chuẩn) | Khác nhau           |
| **Chi phí**        | Thấp hơn (open-source) | Khác nhau         |

---

## Điểm Mạnh Theo Nền Tảng

### PostgreSQL

```
✓ Quan hệ tốt nhất + hỗ trợ JSON
✓ Tìm kiếm toàn văn (tích hợp sẵn)
✓ Tính năng nâng cao (arrays, ranges, custom types)
✓ Truy vấn phức tạp
✓ Tài liệu xuất sắc

Lý tưởng cho: Ứng dụng web, data warehousing, phân tích phức tạp
```

### MySQL

```
✓ Nhanh, đơn giản, đáng tin cậy
✓ Tốt cho tải nặng đọc
✓ Dễ mở rộng ngang (sharding)
✓ Yêu cầu tài nguyên thấp hơn

Lý tưởng cho: Ứng dụng web lưu lượng cao, đọc nhanh
```

### MongoDB

```
✓ Dễ bắt đầu (schema linh hoạt)
✓ Mở rộng ngang có sẵn
✓ Transaction đa tài liệu (4.0+)
✓ Tốt cho dữ liệu real-time

Lý tưởng cho: Phát triển nhanh, dữ liệu linh hoạt, microservices
```

### Redis

```
✓ Nhanh nhất có thể (in-memory)
✓ Hoàn hảo cho caching/session
✓ Atomic operations
✓ Message queues

Lý tưởng cho: Caching, session, tính năng real-time
```

---

## Kiến Trúc Kết Hợp Phổ Biến

### Kiến Trúc Ứng Dụng Web

```
┌─ PostgreSQL (CSDL chính)
│  Dữ liệu người dùng, đơn hàng, giao dịch
│
├─ Redis (cache)
│  Dữ liệu session, đối tượng truy cập thường xuyên
│
├─ Elasticsearch (tìm kiếm)
│  Tìm kiếm toàn văn trên sản phẩm/nội dung
│
└─ TimescaleDB (metrics)
   Metrics ứng dụng, giám sát
```

### Mẫu Microservices

```
Service A: PostgreSQL (đơn hàng)
Service B: MongoDB (hồ sơ người dùng)
Service C: Cassandra (event log)
Dùng chung: Redis (cache)
```

---

## Câu Hỏi Phỏng Vấn

1. **So sánh RDBMS vs NoSQL. Khi nào dùng mỗi loại?**
   - Gợi ý: ACID + có cấu trúc → RDBMS, linh hoạt + mở rộng → NoSQL

2. **Tại sao chọn MongoDB thay vì PostgreSQL?**
   - Gợi ý: Linh hoạt schema, mở rộng ngang dễ hơn, dữ liệu real-time

3. **Thiết kế lựa chọn CSDL cho một hệ thống cụ thể**
   - Gợi ý: Phân tích yêu cầu, ánh xạ vào điểm mạnh CSDL

4. **Hạn chế của NoSQL là gì?**
   - Gợi ý: Không có join, eventual consistency, không có ACID, transaction phức tạp

---

## Tham Chiếu Nhanh

```
Cần ACID?            → PostgreSQL
Cần mở rộng?         → Cassandra/MongoDB
Cần tốc độ (cache)?  → Redis
Cần full-text?       → Elasticsearch
Cần time-series?     → InfluxDB
Lựa chọn mặc định?  → PostgreSQL + Redis
```
