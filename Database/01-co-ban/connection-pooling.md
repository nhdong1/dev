# Connection Pooling — Gộp Kết Nối

## Tổng Quan

Connection pooling (gộp kết nối) giảm chi phí bằng cách tái sử dụng các kết nối CSDL thay vì tạo mới cho mỗi yêu cầu. Đây là yếu tố then chốt cho hiệu suất trong các hệ thống có nhiều yêu cầu đồng thời.

---

## Tại Sao Connection Pooling Quan Trọng?

### Chi Phí Của Một Kết Nối Mới

```
Tạo kết nối TCP mới:
1. TCP handshake (3 gói)       → ~1 ms
2. Thương lượng SSL/TLS        → ~5 ms
3. Xác thực                   → ~1 ms
4. Khởi tạo CSDL              → ~1 ms
─────────────────────────────────────
Tổng cộng: ~10 ms mỗi kết nối

So với:

Tái sử dụng kết nối từ pool:
1. Lấy từ pool                 → <1 ms
```

**Tác động đến web server:**

```
100 yêu cầu đồng thời × 10 ms mỗi kết nối = 1 giây chi phí thêm
Với pooling: ~50 ms chi phí thêm
→ Cải thiện 20 lần về chi phí kết nối
```

### Cạn Kiệt Tài Nguyên

```
PostgreSQL max_connections = 100
Không có pooling:
  100 yêu cầu đồng thời → 100 kết nối
  Yêu cầu thứ 101 → Connection refused (LỖI)

Với pooling (size=20):
  100 yêu cầu đồng thời → 20 kết nối được gộp (xếp hàng)
  Yêu cầu thứ 101 → Chờ kết nối được giải phóng
```

---

## Cách Connection Pooling Hoạt Động

### Kiến Trúc

```
Nhiều Application Server
        ↓
Connection Pool (ví dụ: PgBouncer)
    [conn1][conn2][conn3]
        ↓
Database Server (số kết nối có hạn)
```

### Trạng Thái Kết Nối

```
Pool: [IDLE] [IDLE] [IDLE] [BUSY] [BUSY]
       Sẵn sàng, đang chờ    Đang được dùng
```

**Luồng xử lý yêu cầu:**

```
1. Ứng dụng yêu cầu kết nối
2. Pool kiểm tra: Có kết nối IDLE không?
   CÓ  → Trả về ngay
   KHÔNG → Xếp yêu cầu vào hàng, chờ
3. Ứng dụng dùng kết nối
4. Ứng dụng đóng kết nối (trả về pool)
5. Pool đánh dấu kết nối IDLE (sẵn sàng cho ứng dụng tiếp theo)
```

---

## Các Loại Connection Pooling

### 1. Pooling ở Tầng Ứng Dụng

**Vị trí:** Trong server ứng dụng (Node.js, Python, Java)

**Ví dụ:**
- HikariCP (Java)
- Node-postgres pool
- SQLAlchemy (Python)
- Django database pool

**Ví dụ (Node.js với node-postgres):**

```javascript
const pool = new Pool({
  user: "postgres",
  password: "secret",
  host: "localhost",
  port: 5432,
  database: "myapp",
  max: 20,                    // Kích thước pool
  idleTimeoutMillis: 30000,   // Đóng kết nối idle sau 30s
  connectionTimeoutMillis: 2000,
});

app.get("/users/:id", async (req, res) => {
  const connection = await pool.connect();
  try {
    const result = await connection.query("SELECT * FROM users WHERE id = $1", [
      req.params.id,
    ]);
    res.json(result.rows[0]);
  } finally {
    connection.release(); // Trả về pool
  }
});
```

**Ưu điểm:**
- ✓ Đơn giản (thư viện xử lý)
- ✓ Độ trễ thấp (local)
- ✓ Hoạt động với mọi CSDL

**Nhược điểm:**
- ❌ Mỗi instance ứng dụng có pool riêng (nhân kết nối)
- ❌ Khó giám sát qua nhiều server
- ❌ Mỗi ứng dụng cần chi phí riêng

---

### 2. Pooling ở Tầng Proxy (Middleware)

**Vị trí:** Giữa ứng dụng và CSDL (dịch vụ riêng)

**Ví dụ:**
- PgBouncer (PostgreSQL)
- ProxySQL (MySQL)
- Pgpool-II (PostgreSQL)

**Kiến trúc:**

```
[App Server 1] ┐
[App Server 2] ├─→ [Connection Pool] → [PostgreSQL]
[App Server 3] ┘      (PgBouncer)
```

**Cấu hình PgBouncer:**

```ini
[databases]
mydb = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
```

**Ưu điểm:**
- ✓ Pool dùng chung cho tất cả ứng dụng (hiệu quả kết nối)
- ✓ Giám sát tập trung
- ✓ Trong suốt với ứng dụng
- ✓ Kết nối từ nhiều server

**Nhược điểm:**
- ⚠️ Thêm một network hop (độ trễ hơi cao hơn)
- ⚠️ Cần thêm cơ sở hạ tầng
- ⚠️ Độ phức tạp tăng

---

## Kích Thước Pool

### Công Thức

```
Kích thước Pool = ((Số CPU * 2) + Số đĩa hiệu dụng)

Ví dụ:
- CPU 8 nhân, 1 đĩa
- Kích thước Pool = (8 * 2) + 1 = 17
- Dùng 10-15 để an toàn
```

**Quy tắc thực tế hơn:**

```
Kích thước Pool ≈ (Số truy vấn đồng thời dự kiến) + Dự phòng

Ví dụ:
- Lượng truy cập đỉnh: 50 yêu cầu đồng thời
- Thời gian truy vấn DB: 50ms trung bình
- Pool = 50 + 5 (dự phòng) = 55

Thực tế: Bắt đầu với 20-25, giám sát, điều chỉnh
```

### Pool Quá Nhỏ vs Quá Lớn

| Kích thước      | Vấn đề              | Triệu chứng                                  |
| --------------- | ------------------- | -------------------------------------------- |
| Quá nhỏ (5)     | Hàng đợi kết nối tăng | Yêu cầu timeout khi chờ kết nối           |
| Tối ưu (20-25)  | Dùng tài nguyên hiệu quả | Cân bằng độ trễ + tải DB               |
| Quá lớn (200)   | Lãng phí tài nguyên | Nhiều kết nối idle, áp lực bộ nhớ DB       |

**Giám sát:** Theo dõi độ sâu hàng đợi

```
Nếu hàng đợi luôn rỗng: Kích thước quá lớn
Nếu hàng đợi thường xuyên đầy: Kích thước quá nhỏ
```

---

## Cấu Hình Pool

### Các Tham Số Chính

```
connection_timeout    → Thời gian chờ kết nối có sẵn
idle_timeout          → Thời gian giữ kết nối idle
max_idle              → Số kết nối idle tối đa
queue_timeout         → Thời gian yêu cầu chờ trong hàng đợi
validation_query      → Kiểm tra sức khỏe kết nối trước khi dùng
```

### Ví dụ (HikariCP - Java)

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("postgres");
config.setPassword("secret");

// Kích thước pool
config.setMaximumPoolSize(20);           // Số kết nối tối đa
config.setMinimumIdle(5);                // Số idle tối thiểu luôn sẵn sàng

// Timeouts
config.setConnectionTimeout(30000);      // 30 giây
config.setIdleTimeout(600000);           // 10 phút
config.setMaxLifetime(1800000);          // 30 phút

// Kiểm tra sức khỏe
config.setConnectionTestQuery("SELECT 1");
config.setLeakDetectionThreshold(60000); // Cảnh báo nếu mở 60s+

HikariDataSource ds = new HikariDataSource(config);
```

### Các Vấn Đề & Cách Sửa

| Vấn đề                      | Nguyên nhân                    | Cách sửa                                        |
| --------------------------- | ------------------------------ | ----------------------------------------------- |
| Kết nối bị rò rỉ            | Ứng dụng không trả kết nối     | Bật cảnh báo phát hiện rò rỉ                    |
| Timeout hàng đợi            | Pool quá nhỏ                   | Tăng kích thước pool                            |
| Lỗi timeout idle            | Kết nối idle quá lâu           | Giảm idle_timeout hoặc dùng validation_query    |
| Vượt quá số kết nối tối đa  | Pool + kết nối trực tiếp       | Ngừng kết nối trực tiếp                         |

---

## Chế Độ Pool

### Chế Độ Transaction (Khuyến nghị)

**Quy tắc:** Pool một kết nối cho mỗi transaction (không phải mỗi session)

```
Client A: BEGIN → LẤY conn1 → QUERY → COMMIT → TRẢ conn1
Client B: BEGIN → LẤY conn1 (chờ) → QUERY → COMMIT → TRẢ conn1
```

**Ưu điểm:**
- ✓ Tối đa hóa tái sử dụng kết nối
- ✓ Cho phép nhiều client với ít kết nối
- ✓ Giải quyết vấn đề giới hạn kết nối

**Cạm bẫy:**

```sql
-- Không hoạt động với Prepared Statements qua nhiều transaction
PREPARE stmt AS SELECT * FROM users WHERE id = $1;
EXECUTE stmt;
-- Kết nối được trả về pool
EXECUTE stmt;  -- LỖI: Prepared statement không tìm thấy!

-- Giải pháp: Dùng parameterized queries mỗi transaction
SELECT * FROM users WHERE id = $1;  -- Prepare mới mỗi lần
```

---

### Chế Độ Session

**Quy tắc:** Pool một kết nối cho mỗi session (toàn bộ vòng đời kết nối)

```
Client A: LẤY conn1 → QUERY → QUERY → QUERY → TRẢ conn1 (khi xong)
Client B: Chờ conn1
```

**Ưu điểm:**
- ✓ Hỗ trợ đầy đủ tính năng PostgreSQL
- ✓ Prepared Statements hoạt động qua nhiều transaction
- ✓ Biến session được giữ nguyên

**Nhược điểm:**
- ❌ Tái sử dụng ít hơn (một kết nối mỗi client)

---

## Giám Sát Connection Pool

### Các Chỉ Số Quan Trọng

```
Kết nối active:    Đang thực thi truy vấn
Kết nối idle:      Đang chờ yêu cầu
Yêu cầu trong hàng: Chờ kết nối
Lỗi kết nối:      Lần lấy thất bại
Số lần rò rỉ:     Kết nối không được trả về
```

### Giám sát PostgreSQL (PgBouncer)

```sql
-- Kết nối đến admin console PgBouncer
psql -U pgbouncer -d pgbouncer -h localhost -p 6432

-- Xem trạng thái pool
SHOW pools;

-- Thống kê theo CSDL
SHOW stats;

-- Chi tiết kết nối
SHOW clients;
SHOW servers;

-- Reload cấu hình
RELOAD;
```

---

## Các Lỗi Thường Gặp

### 1. Rò Rỉ Kết Nối

```javascript
// Sai
app.get('/users', async (req, res) => {
  const conn = await pool.connect();
  if (!user_id) {
    res.send(400);  // Quên release!
  }
  const result = await conn.query(...);
  conn.release();
  res.json(result);
});

// Đúng
app.get('/users', async (req, res) => {
  const conn = await pool.connect();
  try {
    if (!user_id) {
      return res.send(400);
    }
    const result = await conn.query(...);
    res.json(result);
  } finally {
    conn.release();  // Luôn được gọi
  }
});
```

### 2. Transaction Dài Giữ Kết Nối

```python
# Sai
with db.connection() as conn:
    data = conn.query("SELECT * FROM big_table")
    # Xử lý dữ liệu trong ứng dụng (5 giây)
    time.sleep(5)
    # Kết nối được giữ cả thời gian!
    conn.execute("UPDATE table SET processed = true")

# Đúng
data = db.query("SELECT * FROM big_table")
# Xử lý dữ liệu trong ứng dụng (5 giây)
time.sleep(5)
# Không giữ kết nối
with db.connection() as conn:
    conn.execute("UPDATE table SET processed = true")
```

---

## Câu Hỏi Phỏng Vấn

1. **Tại sao connection pooling quan trọng?**
   - Gợi ý: Chi phí mỗi lần tạo kết nối + ngăn cạn kiệt tài nguyên

2. **Sự khác biệt giữa chế độ transaction và session pooling?**
   - Gợi ý: Transaction mode tái sử dụng mỗi transaction, session mode mỗi client

3. **Làm thế nào để xác định kích thước pool?**
   - Gợi ý: Công thức + giám sát độ sâu hàng đợi

4. **Thiết kế chiến lược pooling cho microservices**
   - Gợi ý: Proxy pooling (PgBouncer) vs app-level + các cân nhắc

5. **Điều gì gây ra rò rỉ kết nối và cách ngăn chặn?**
   - Gợi ý: Không release trong đường xử lý lỗi + dùng try/finally

---

## Tham Chiếu Nhanh

```
Chỉ số               Phạm vi tốt      Cảnh báo
────────────────────────────────────────────────
Kích thước Pool      10-25            < 5 hoặc > 100
Độ sâu hàng đợi      Hầu hết là 0     Liên tục > 5
Thời gian lấy kết nối  < 1 ms          > 10 ms
Kết nối Idle         20-50%           < 5% hoặc > 80%
Sự kiện rò rỉ        0                > 0 (cần điều tra)
```
