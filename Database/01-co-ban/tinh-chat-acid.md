# Tính Chất ACID

## Tổng Quan

ACID là viết tắt của bốn tính chất quan trọng đảm bảo độ tin cậy của các transaction trong cơ sở dữ liệu. Hiểu rõ ACID là nền tảng để thiết kế các hệ thống bền vững.

## Bốn Tính Chất ACID

### 1. Tính Nguyên Tử (Atomicity) — Tất cả hoặc Không có gì

**Định nghĩa:** Một transaction hoặc được thực hiện hoàn toàn, hoặc bị hủy bỏ hoàn toàn. Không thể thực hiện một phần.

**Ví dụ thực tế:**

```sql
-- Chuyển tiền ngân hàng: Atomicity đảm bảo cả hai thao tác xảy ra cùng lúc
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Alice bị trừ 100$
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Bob được cộng 100$
COMMIT;
```

Nếu hệ thống bị sập giữa hai câu lệnh UPDATE:
- CSDL sẽ khôi phục về trạng thái trước transaction
- Alice vẫn giữ 100$, Bob không nhận được tiền
- Không có trạng thái nào bị "mất tiền"

**Chi Tiết Triển Khai:**

- **Write-Ahead Logging (WAL):** Các thay đổi được ghi vào log trước khi commit vào đĩa
- **Khả năng Rollback:** Undo log được lưu trữ cho mọi thay đổi
- **2-Phase Commit:** Đảm bảo tất cả-hoặc-không-gì trên nhiều hệ thống

**Lỗi Thường Gặp:**

```
Ứng dụng xử lý SQL nhưng bị crash trước khi COMMIT
→ CSDL rollback, nhưng ứng dụng nghĩ đã thành công

Giải pháp: Luôn xử lý COMMIT/ROLLBACK trong cùng một ngữ cảnh transaction
```

---

### 2. Tính Nhất Quán (Consistency) — Trạng Thái Hợp Lệ

**Định nghĩa:** CSDL chuyển từ trạng thái hợp lệ này sang trạng thái hợp lệ khác. Không có tham chiếu mồ côi, vi phạm ràng buộc hay dữ liệu bị hỏng.

**Ví dụ:**

```sql
-- Ràng buộc khóa ngoại đảm bảo tính nhất quán
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL NOT NULL CHECK (balance >= 0)
);

CREATE TABLE transactions (
  id INT PRIMARY KEY,
  account_id INT REFERENCES accounts(id),
  amount DECIMAL NOT NULL
);
```

**Các Quy Tắc Nhất Quán:**

1. **Ràng buộc được thực thi:**
   - Khóa chính (không trùng lặp)
   - Khóa ngoại (không có orphan)
   - Check constraint (quy tắc nghiệp vụ)
   - Unique constraint

2. **Trigger** có thể thực thi nhất quán phức tạp:

```sql
CREATE TRIGGER balance_check BEFORE INSERT ON transactions
BEGIN
  IF (SELECT balance FROM accounts WHERE id = NEW.account_id) < NEW.amount
  THEN RAISE ERROR 'Số dư không đủ';
  END IF;
END;
```

---

### 3. Tính Cô Lập (Isolation) — Độc Lập Đồng Thời

**Định nghĩa:** Các transaction đồng thời không can thiệp vào nhau. Mỗi transaction thấy một snapshot nhất quán.

**Các Vấn Đề Mà Isolation Ngăn Chặn:**

| Vấn đề                   | Mô tả                        | Ví dụ                                                                    |
| ------------------------ | ---------------------------- | ------------------------------------------------------------------------ |
| **Dirty Read**           | Đọc thay đổi chưa commit     | Transaction A đọc bản cập nhật chưa commit của B trước khi B commit     |
| **Non-Repeatable Read**  | Dữ liệu thay đổi giữa chừng  | Cùng SELECT trả về kết quả khác ở đầu và cuối transaction               |
| **Phantom Read**         | Hàng mới xuất hiện/biến mất  | Mệnh đề WHERE khớp với các hàng khác nhau trong suốt transaction         |
| **Lost Update**          | Cập nhật đồng thời bị ghi đè | Hai transaction đọc `x=10`, cả hai cộng 1, ghi `x=11` thay vì `x=12`   |

**Mức Độ Isolation (Từ thấp đến cao):**

```
Read Uncommitted → Read Committed → Repeatable Read → Serializable
    ↓                    ↓                ↓                 ↓
 (không có)       (chặn dirty reads) (chặn non-repeat)  (khóa hoàn toàn)
 Hiệu suất cao    Chuẩn             Mặc định (PostgreSQL) An toàn nhất
 Kém an toàn       ✓ Production      ✓ Phổ biến nhất        Chậm
```

---

### 4. Tính Bền Vững (Durability) — Lưu Trữ Vĩnh Viễn

**Định nghĩa:** Dữ liệu đã commit tồn tại sau mọi sự cố (crash, mất điện, hỏng đĩa).

**Cơ Chế:**

1. **Write-Ahead Logging (WAL):**

   ```
   Bước 1: Ghi vào log trên đĩa
   Bước 2: Cập nhật buffer trong RAM
   Bước 3: Trả về thành công cho client
   Bước 4: Định kỳ flush buffer xuống đĩa
   ```

2. **Đảm bảo Fsync:**
   - `fsync=on` (PostgreSQL): Mỗi COMMIT chờ ghi đĩa
   - `fsync=off`: Nhanh hơn nhiều nhưng nguy hiểm khi crash

3. **Replication:** Dữ liệu trên nhiều server

**Timeline Ví dụ:**

```
08:00:00.000 - Client bắt đầu transaction
08:00:00.100 - Gửi INSERT/UPDATE đến server
08:00:00.200 - Server ghi vào WAL trên đĩa
08:00:00.300 - Client nhận phản hồi thành công
08:00:00.500 - CRASH! Server mất điện

Phục hồi: CSDL đọc WAL, phát lại các transaction đã commit
Kết quả: Dữ liệu được khôi phục và bền vững
```

---

## Vi Phạm ACID Trong Thực Tế

### Khi ACID Bị Phá Vỡ

**Lỗi ở tầng ứng dụng:**

```python
def chuyen_tien(tu_id, den_id, so_tien):
    # Sai: Không dùng transaction
    db.execute(f"UPDATE accounts SET balance = balance - {so_tien} WHERE id = {tu_id}")

    if random() > 0.99:  # 1% thời gian
        raise Exception("Lỗi mạng")  # Bị rollback, nhưng...

    db.execute(f"UPDATE accounts SET balance = balance + {so_tien} WHERE id = {den_id}")
```

**Cách đúng:**

```python
def chuyen_tien(tu_id, den_id, so_tien):
    with db.transaction():  # Đúng
        db.execute(f"UPDATE accounts SET balance = balance - {so_tien} WHERE id = {tu_id}")
        db.execute(f"UPDATE accounts SET balance = balance + {so_tien} WHERE id = {den_id}")
        # Cả hai thành công cùng nhau hoặc cả hai thất bại
```

---

## Hỗ Trợ ACID Theo Nền Tảng

### PostgreSQL

- ✅ ACID đầy đủ ở cấp transaction
- ✅ Hỗ trợ Serializable isolation
- ✅ Replication đồng bộ đảm bảo tính bền vững
- ✅ An toàn khi crash với WAL

### MySQL (InnoDB)

- ✅ ACID đầy đủ (engine InnoDB)
- ⚠️ Mặc định Repeatable Read (không phải Serializable)
- ✅ Group Commit đảm bảo tính bền vững
- ⚠️ Engine MyISAM ≠ ACID (không còn dùng nữa)

### MongoDB

- ✅ ACID ở cấp single document (4.0+)
- ✅ ACID đa tài liệu (4.0+)
- ⚠️ Không có ACID cross-shard
- ⚠️ Tính bền vững phụ thuộc vào `writeConcern`

---

## Đánh Đổi ACID

| Tính chất         | Lợi ích                     | Chi phí                    | Mặc định |
| ----------------- | --------------------------- | -------------------------- | -------- |
| **Atomicity**     | Không có cập nhật một phần  | Chi phí rollback           | Bật      |
| **Consistency**   | Không có trạng thái không hợp lệ | Kiểm tra ràng buộc   | Bật      |
| **Isolation**     | Không có dữ liệu bẩn        | Tranh chấp khóa            | Thay đổi |
| **Durability**    | Lưu trữ dữ liệu             | Độ trễ I/O đĩa             | Bật      |

**Điều chỉnh hiệu suất:**

```sql
-- PostgreSQL: Giảm tính bền vững để tăng tốc (CHỈ cho testing!)
SET synchronous_commit = off;  -- Rủi ro: mất dữ liệu khi crash
SET fsync = off;               -- Rủi ro: hỏng dữ liệu khi crash

-- Cho production: Giữ tất cả tính chất ACID
SET synchronous_commit = on;
```

---

## Câu Hỏi Phỏng Vấn

1. **Giải thích ACID và tại sao nó quan trọng**
   - Gợi ý: Định nghĩa từng tính chất + tác động thực tế

2. **Chuyện gì xảy ra nếu mất điện giữa chừng một transaction?**
   - Gợi ý: Giải thích WAL → quá trình phục hồi

3. **Có thể có ACID mà không cần khóa không?**
   - Gợi ý: Giải thích MVCC (PostgreSQL/MySQL) ngăn nhiều khóa

4. **Thiết kế hệ thống không có ACID. Điều gì sẽ bị phá vỡ?**
   - Gợi ý: Hệ thống phân tán, đánh đổi eventual consistency

---

## Bài Tập Thực Hành

```sql
-- Test 1: Atomicity
BEGIN;
  INSERT INTO accounts VALUES (1, 100);
  INSERT INTO accounts VALUES (1, 200);  -- Khóa trùng! Sẽ thất bại
COMMIT;  -- Toàn bộ transaction rollback

SELECT * FROM accounts;  -- Rỗng (cả hai inserts đều bị rollback)

-- Test 2: Isolation
-- Terminal 1:
BEGIN;
  UPDATE accounts SET balance = 500 WHERE id = 1;
  -- Chưa commit

-- Terminal 2:
SELECT * FROM accounts WHERE id = 1;  -- Vẫn thấy giá trị cũ (không có dirty read)

-- Test 3: Durability
INSERT INTO accounts VALUES (1, 100);
COMMIT;
-- Server crash ngay sau đó

-- Sau khi khởi động lại: Dữ liệu vẫn còn (bền vững)
SELECT * FROM accounts;  -- Row 1 vẫn còn
```

---

## Điểm Mấu Chốt

- ✅ **Atomicity**: Transaction tất cả-hoặc-không-gì ngăn ngừa hỏng dữ liệu
- ✅ **Consistency**: Ràng buộc giữ CSDL ở trạng thái hợp lệ
- ✅ **Isolation**: Các transaction đồng thời không can thiệp nhau
- ✅ **Durability**: WAL đảm bảo phục hồi sau crash
- ⚠️ **Đánh đổi**: ACID nghiêm ngặt có chi phí hiệu suất; điều chỉnh dựa trên mức độ chấp nhận rủi ro
- ⚠️ **Không tự động**: Phải sử dụng transaction đúng cách trong code ứng dụng
