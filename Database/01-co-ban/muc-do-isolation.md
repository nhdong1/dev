# Mức Độ Isolation (Transaction Isolation Levels)

## Tổng Quan

Mức độ isolation kiểm soát sự cân bằng giữa **đồng thời** (nhiều transaction cùng lúc) và **an toàn** (ngăn ngừa các bất thường). Isolation cao hơn = an toàn hơn nhưng chậm hơn.

---

## Bốn Mức Độ Isolation

### 1. Read Uncommitted (Đọc Chưa Commit) ❌

**Định nghĩa:** Transaction có thể đọc dữ liệu từ transaction chưa commit.

**Các vấn đề cho phép:**
- ✅ Dirty reads (đọc bẩn)
- ✅ Non-repeatable reads
- ✅ Phantom reads

**Ví dụ:**

```
Transaction A (Ghi)              Transaction B (Đọc)
BEGIN;
UPDATE balance = 0 WHERE id=1;
                                 BEGIN;
                                 SELECT balance;  -- Trả về 0 (đọc bẩn!)
ROLLBACK;  -- Không bao giờ xảy ra!
                                 SELECT balance;  -- Trả về 100 (giá trị gốc)
```

**Khi nào dùng:**
- ❌ Hầu như không bao giờ (quá nguy hiểm)
- Chỉ cho phân tích không quan trọng với CSDL cũ

**Hiệu suất:** Nhanh nhất (không có khóa)

---

### 2. Read Committed (Chỉ Đọc Đã Commit) ✓

**Định nghĩa:** Chỉ đọc dữ liệu đã được commit. Mặc định phổ biến nhất.

**Vấn đề được ngăn chặn:**
- ✓ Dirty reads

**Vấn đề vẫn còn:**
- ✅ Non-repeatable reads
- ✅ Phantom reads

**Ví dụ:**

```
Transaction A (Ghi)              Transaction B (Đọc)
BEGIN;
UPDATE balance = 0 WHERE id=1;
                                 BEGIN;
                                 SELECT balance;  -- Chờ! Đợi A
COMMIT;  -- Bây giờ đã commit
SELECT balance;  -- Trả về 0 (chỉ dữ liệu đã commit)
                                 ROLLBACK;
```

**Khi nào dùng:**
- ✓ Mặc định cho MySQL, SQL Server
- ✓ Hầu hết ứng dụng web
- ✓ Cân bằng tốt giữa hiệu suất + an toàn

**Mặc định trong PostgreSQL** và SQL Server.

---

### 3. Repeatable Read (Đọc Lặp Lại Được) 🔒

**Định nghĩa:** Cùng truy vấn trả về cùng dữ liệu trong suốt transaction. Không có hàng mới xuất hiện/biến mất.

**Vấn đề được ngăn chặn:**
- ✓ Dirty reads
- ✓ Non-repeatable reads

**Vấn đề vẫn còn:**
- ✅ Phantom reads (theo chuẩn SQL)
- ⚠️ PostgreSQL không cho phép phantom reads (triển khai khác nhau)

**Ví dụ — Non-repeatable Read (Read Committed có vấn đề này):**

```
Transaction A (Sửa đổi)        Transaction B (Đọc)
                                BEGIN;
                                SELECT * FROM orders WHERE user_id = 1;
                                -- Trả về: order1, order2
BEGIN;
INSERT INTO orders (user_id = 1);
COMMIT;
                                SELECT * FROM orders WHERE user_id = 1;
                                -- Trả về: order1, order2, order3 (PHANTOM!)
                                COMMIT;
```

**Với Repeatable Read:**

```
                                BEGIN REPEATABLE READ;
                                SELECT * FROM orders;  -- Snapshot được chụp
BEGIN;
INSERT INTO orders;
COMMIT;
                                SELECT * FROM orders;  -- Vẫn thấy snapshot cũ
                                COMMIT;
```

**Khi nào dùng:**
- ✓ Transaction tài chính
- ✓ Báo cáo cần nhất quán
- ✓ Mặc định trong MySQL

---

### 4. Serializable (Tuần Tự Hóa) 🔐

**Định nghĩa:** Transaction thực thi như thể chúng tuần tự (từng cái một). Isolation nghiêm ngặt nhất.

**Vấn đề được ngăn chặn:**
- ✓ Dirty reads
- ✓ Non-repeatable reads
- ✓ Phantom reads
- ✓ Serialization anomalies

**Ví dụ — Serialization Anomaly:**

```
Transaction A                  Transaction B
BEGIN;
SELECT SUM(balance) FROM accounts;
-- Kết quả: 1000
                               BEGIN;
                               INSERT INTO accounts (100);
                               COMMIT;
SELECT SUM(balance);
-- Kết quả: 1100
-- A thấy trạng thái không nhất quán!
COMMIT;
```

**Với Serializable:**
- Một transaction chặn cho đến khi transaction kia commit
- Dữ liệu hoàn toàn nhất quán

**Khi nào dùng:**
- ✓ Hệ thống tài chính quan trọng
- ✓ Yêu cầu quy định
- ⚠️ Kỳ vọng tác động hiệu suất
- ✓ Dùng khôn ngoan (không phải tất cả transaction đều cần)

---

## Mặc Định & Khả Năng Theo Nền Tảng

| Nền tảng       | Mặc định            | Hỗ trợ cả 4 | Ghi chú                      |
| -------------- | ------------------- | ------------ | ---------------------------- |
| **PostgreSQL** | Read Committed      | ✅ Có         | SSI cho Serializable         |
| **MySQL**      | Repeatable Read     | ⚠️ Hạn chế   | Thiếu Serializable thực sự   |
| **SQL Server** | Read Committed      | ✅ Có         | Snapshot isolation có sẵn    |
| **Oracle**     | Read Committed      | ✅ Có         | Multi-versioning             |

---

## Các Bất Thường Được Ngăn Chặn

### Dirty Read (Đọc Bẩn)

```sql
-- Read Uncommitted cho phép điều này:
Transaction A:
  UPDATE users SET credit = 0 WHERE id = 1;

Transaction B:
  SELECT credit FROM users WHERE id = 1;  -- Thấy 0 (chưa commit!)

Transaction A:
  ROLLBACK;  -- Không bao giờ xảy ra!

-- Kết quả: B đọc dữ liệu không bao giờ được commit
```

**Được sửa bởi:** Read Committed+

---

### Non-Repeatable Read (Đọc Không Lặp Lại)

```sql
-- Read Committed cho phép điều này:
Transaction A:
  SELECT salary FROM employees WHERE id = 1;  -- Trả về 50000

Transaction B:
  UPDATE employees SET salary = 60000 WHERE id = 1;
  COMMIT;

Transaction A:
  SELECT salary FROM employees WHERE id = 1;  -- Trả về 60000!

-- Kết quả: Cùng SELECT cho kết quả khác nhau
```

**Được sửa bởi:** Repeatable Read+

---

### Phantom Read (Đọc Bóng Ma)

```sql
-- Repeatable Read (theo chuẩn SQL) cho phép điều này:
Transaction A:
  SELECT * FROM employees WHERE salary > 50000;  -- 5 hàng

Transaction B:
  INSERT INTO employees (salary = 60000);
  COMMIT;

Transaction A:
  SELECT * FROM employees WHERE salary > 50000;  -- 6 hàng!

-- Kết quả: Hàng bóng ma xuất hiện
```

**Được sửa bởi:** Serializable

**Lưu ý:** PostgreSQL thực tế ngăn điều này trong Repeatable Read nhờ phát hiện xung đột.

---

## Chọn Mức Độ Phù Hợp

### Ma Trận Quyết Định

```
Bạn cần:
  1. Tốc độ? (Nhiều transaction đồng thời)
     → Read Committed

  2. Chính xác? (Tài chính, kiểm toán)
     → Repeatable Read hoặc Serializable

  3. Tuân thủ pháp lý? (Không được có bất kỳ bất thường)
     → Serializable

  4. Mặc định hoạt động tốt? (An toàn không có chi phí)
     → Repeatable Read
```

### Theo Trường Hợp Sử Dụng

| Tình huống                  | Mức độ          | Lý do                     |
| --------------------------- | --------------- | ------------------------- |
| Yêu cầu web (REST API)      | Read Committed  | Tốc độ + an toàn đủ       |
| Chuyển tiền tài chính       | Repeatable Read | Cần nhất quán             |
| Thanh toán ngân hàng        | Serializable    | Không được có bất thường  |
| Báo cáo phân tích           | Repeatable Read | Snapshot nhất quán        |
| Dashboard real-time         | Read Committed  | Mới hơn > chính xác       |

---

## Thiết Lập Mức Độ Isolation

### PostgreSQL

```sql
-- Mức phiên làm việc (áp dụng cho tất cả transaction)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Mức transaction (áp dụng cho transaction tiếp theo)
BEGIN ISOLATION LEVEL REPEATABLE READ;
  SELECT * FROM accounts;
  -- Tất cả truy vấn trong transaction này dùng Repeatable Read
COMMIT;

-- Kiểm tra mức hiện tại
SHOW TRANSACTION ISOLATION LEVEL;
```

### MySQL

```sql
-- Mức phiên làm việc
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Mức toàn cục (ảnh hưởng đến kết nối mới)
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Kiểm tra mức hiện tại
SELECT @@transaction_isolation;
```

### SQL Server

```sql
-- Mức kết nối
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

---

## Tác Động Hiệu Suất

| Mức độ           | Khóa   | Snapshot                | Tốc độ  | An toàn |
| ---------------- | ------ | ----------------------- | ------- | ------- |
| Read Uncommitted | Không  | Không                   | ⚡⚡⚡  | ❌      |
| Read Committed   | Ngắn   | Mỗi câu lệnh            | ⚡⚡    | ✓       |
| Repeatable Read  | Trung  | Mỗi transaction         | ⚡      | ✓✓      |
| Serializable     | Dài    | Phát hiện xung đột đầy đủ | 🐢    | ✓✓✓     |

---

## Bẫy Thường Gặp

### 1. Transaction Dài ở Mức Cao

```sql
-- Sai
BEGIN REPEATABLE READ;
  SELECT * FROM big_table;  -- Giữ snapshot
  -- ... Ứng dụng suy nghĩ 30 phút ...
  UPDATE big_table SET status = 'processed';
COMMIT;
-- Khóa được giữ cả thời gian, chặn transaction khác

-- Đúng
SELECT * FROM big_table;  -- Không có transaction
-- ... Ứng dụng suy nghĩ ...
BEGIN REPEATABLE READ;
  UPDATE big_table SET status = 'processed';
COMMIT;
```

### 2. Bỏ Qua Mặc Định Theo Nền Tảng

```sql
-- MySQL mặc định REPEATABLE READ
-- PostgreSQL mặc định READ COMMITTED
-- SQL Server mặc định READ COMMITTED

-- Ứng dụng chạy tốt trên một nhưng thất bại trên nền tảng khác!
-- Giải pháp: Đặt mức isolation rõ ràng trong code ứng dụng
```

---

## Câu Hỏi Phỏng Vấn

1. **Giải thích các mức isolation và tại sao chúng quan trọng**
   - Gợi ý: Read Uncommitted → Serializable, đánh đổi

2. **Repeatable Read ngăn chặn bất thường nào?**
   - Gợi ý: Dirty reads + non-repeatable reads, nhưng KHÔNG phantoms (theo chuẩn SQL)

3. **Khi nào dùng Serializable?**
   - Gợi ý: Transaction tài chính quan trọng + yêu cầu quy định

4. **PostgreSQL đạt được Repeatable Read như thế nào?**
   - Gợi ý: MVCC + snapshot isolation

---

## Tham Chiếu Nhanh

```
        Ngăn Chặn Bất Thường
Mức độ              Dirty   Non-Rep   Phantom
─────────────────────────────────────────────────
Read Uncommitted      ❌       ❌         ❌
Read Committed        ✓        ❌         ❌
Repeatable Read       ✓        ✓          ⚠️*
Serializable          ✓        ✓          ✓

* PostgreSQL ngăn, chuẩn SQL cho phép
```
