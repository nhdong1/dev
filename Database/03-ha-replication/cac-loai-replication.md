# Các Loại Nhân Bản (Replication): Kiến Trúc & Đánh Đổi

Nắm vững các kiến trúc nhân bản cơ bản được sử dụng trong CSDL hiện đại và hiểu khi nào dùng mỗi loại.

---

## Tổng Quan

Các loại nhân bản xác định cách dữ liệu di chuyển từ nguồn đến các replica. Mỗi kiến trúc có đảm bảo nhất quán, độ phức tạp vận hành và khả năng failover khác nhau.

**So Sánh Nhanh:**

| Loại              | Mẫu Ghi              | Failover        | Độ phức tạp | Rủi ro mất dữ liệu | Tốt nhất cho |
| ----------------- | -------------------- | --------------- | ------------ | ------------------- | ------------ |
| **Single-Leader** | Chỉ Primary          | Thăng cấp replica | Thấp       | Thấp (với sync)     | Hầu hết ứng dụng |
| **Multi-Leader**  | Bất kỳ leader        | Tự động         | Cao          | Trung bình          | Multi-region |
| **Leaderless**    | Bất kỳ node (quorum) | Tự động         | Rất cao      | Thấp (điều chỉnh được) | Phân tán cao |
| **Cascading**     | Primary→Replica→Replica | Thủ công     | Trung bình   | Thấp                | Nhiều replica |

---

## 1. Single-Leader (Master-Slave) Replication

### Kiến Trúc

```
         [PRIMARY/MASTER]
              ↓ (luồng write-ahead log)
         [Replica 1]
         [Replica 2]
         [Replica 3]
```

**Cách Hoạt Động:**

1. Tất cả ghi đi đến primary
2. Primary ghi vào WAL local
3. Các entry WAL được stream đến replica
4. Replicas áp dụng thay đổi bất đồng bộ hoặc đồng bộ
5. Đọc có thể đi đến primary hoặc replica

### Ưu Điểm

✅ **Đơn giản & Trực quan** — Vai trò primary/replica rõ ràng
✅ **Hỗ trợ gốc** — Tất cả CSDL đều hỗ trợ
✅ **Dự đoán được** — Nhân bản xác định (phát lại cùng log)
✅ **Độ trễ thấp** — Nhân bản async có tác động tối thiểu
✅ **Failover linh hoạt** — Dễ thăng cấp bất kỳ replica
✅ **Mở rộng đọc** — Phân phối đọc qua các replica

### Nhược Điểm

❌ **Bottleneck ghi đơn** — Tất cả ghi qua primary
❌ **Đọc cũ** — Replicas có thể lag (vấn đề nhất quán tiềm ẩn)
❌ **Failover thủ công** — Không có orchestration bên ngoài
❌ **Tài nguyên replica** — Phải xử lý full dataset

### Hỗ Trợ Theo CSDL

- **PostgreSQL**: Streaming replication (WAL level)
- **MySQL**: Binary log replication
- **MongoDB**: Replica sets (primary-secondary)
- **SQL Server**: Log shipping

### Ví Dụ Cấu Hình

#### PostgreSQL

```sql
-- Trên primary
synchronous_commit = remote_apply    -- Chờ replica
max_wal_senders = 3
synchronous_standby_names = 'replica1,replica2'

-- Trên replica
primary_conninfo = 'host=primary dbname=postgres'
hot_standby = on  -- Cho phép đọc trên replica
```

#### MySQL

```sql
-- Trên primary
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
max_binlog_size = 1G
binlog-format = ROW

-- Trên replica
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin
read-only = ON
```

### Khi Nào Dùng

✅ **Lựa chọn mặc định** cho hầu hết ứng dụng
✅ **Hệ thống OLTP** cần mở rộng đọc
✅ **Throughput ghi giới hạn** (primary đơn chấp nhận được)

### Quy Trình Failover

```
1. Primary ngừng hoạt động
2. Monitor phát hiện lỗi (thường 3-5 giây)
3. Bầu chọn: Chọn replica khỏe nhất
4. Thăng cấp: Replica trở thành primary mới
5. Hạ cấp: Primary cũ gia nhập lại làm replica
6. Cập nhật DNS/Connection
7. Ứng dụng kết nối lại

RTO điển hình: 30 giây - 2 phút
RPO: 0 (sync) hoặc vài giây (async)
```

---

## 2. Multi-Leader (Active-Active) Replication

### Kiến Trúc

```
[Leader A] ←→ [Leader B] ←→ [Leader C]
   ↓ ↓          ↓ ↓          ↓ ↓
Replicas     Replicas      Replicas
```

**Cách Hoạt Động:**

1. Mỗi leader chấp nhận ghi độc lập
2. Ghi được nhân bản đến tất cả leader khác
3. Các leader hoạt động như replica của nhau
4. Giải quyết xung đột xử lý ghi đồng thời

### Ưu Điểm

✅ **Không bottleneck ghi** — Ghi đến bất kỳ datacenter
✅ **Độ trễ local** — Người dùng ghi đến leader gần nhất
✅ **Tính sẵn sàng cao** — Không có điểm lỗi đơn
✅ **Phân tán địa lý** — Hoàn hảo cho multi-region

### Nhược Điểm

❌ **Phức tạp xung đột** — Ghi đồng thời vào cùng bản ghi
❌ **Khó vận hành** — Nhiều moving parts
❌ **Độ trễ cao hơn** — Nhân bản giữa các leader
❌ **Thách thức nhất quán** — Có thể cần eventual consistency

### Các Tình Huống Xung Đột

```
Thời gian  Datacenter A                 Datacenter B
1:00       UPDATE user SET age=30       UPDATE user SET age=31
           WHERE id=1                   WHERE id=1

Độ trễ nhân bản: 100ms

Cả hai leader áp dụng locally:
- A: age = 30
- B: age = 31

Sau nhân bản:
- A nhận: age = 31
- B nhận: age = 30

XUNG ĐỘT! Cái nào đúng?
```

### Chiến Lược Giải Quyết Xung Đột

1. **Last Write Wins (LWW)**
   ```
   Dùng timestamp trên mỗi lần ghi
   Timestamp cao hơn thắng
   Rủi ro: Lựa chọn tùy tiện, có thể mất dữ liệu
   ```

2. **Multi-Version Concurrency Control (MVCC)**
   ```
   Giữ cả hai phiên bản
   Ứng dụng chọn khi đọc
   Rủi ro: Logic merge phức tạp
   ```

3. **Logic Tùy Chỉnh**
   ```
   Xác định quy tắc nghiệp vụ mỗi trường
   age: lấy giá trị lớn nhất
   name: lấy từ region chính
   status: merge flags theo bit
   ```

### Khi Nào Dùng

✅ **Multi-datacenter** (primary + standby ở region khác nhau)
✅ **Ứng dụng toàn cầu** (ghi local ở mỗi region)
✅ **Mở rộng throughput ghi** (phân tán tải ghi)
❌ **Nhất quán mạnh** quan trọng
❌ **Nhóm mới về hệ thống phân tán**

---

## 3. Leaderless (Quorum-Based) Replication

### Kiến Trúc

```
Client gửi ghi đến 3 node (W trong N)
Client gửi đọc đến 3 node (R trong N)

Nhất quán đảm bảo nếu: W + R > N

Ví dụ: N=5, W=3, R=3
Ghi đến 3: Đảm bảo đọc sẽ gặp ≥2 đã ghi
```

### Ưu Điểm

✅ **Tính sẵn sàng cao** — Không có lỗi leader đơn
✅ **Không cần failover** — Phục hồi tự động
✅ **Nhất quán điều chỉnh được** — Đánh đổi qua W và R
✅ **Ghi có thể mở rộng** — Tất cả node chấp nhận ghi
✅ **Không split-brain** — Quorum ngăn xung đột

### Nhược Điểm

❌ **Ngữ nghĩa phức tạp** — Toán quorum không trực quan
❌ **Độ trễ đọc cao hơn** — Phải chờ quorum
❌ **Hỗ trợ CSDL hạn chế** — Chỉ hệ thống chuyên biệt

### Toán Quorum

```
N = tổng số node
W = kích thước quorum ghi
R = kích thước quorum đọc

Nhất quán mạnh: W + R > N
  - Ví dụ: N=5, W=3, R=3 ✓
  - Bất kỳ ghi nào gặp ≥3, bất kỳ đọc nào gặp ≥3
  - Overlap đảm bảo

Nhất quán yếu: W + R ≤ N
  - Ví dụ: N=5, W=2, R=2 ✗
  - Ghi có thể bỏ lỡ node đọc từ đó
  - Đọc cũ có thể xảy ra
```

### Hỗ Trợ CSDL

- **DynamoDB**: Quorum-based (cấu hình W/R)
- **Cassandra**: Mức nhất quán cấu hình được
- **Riak**: Vector clocks + quorum

### Khi Nào Dùng

✅ **Hệ thống tính sẵn sàng cao** (chịu đựng lỗi)
✅ **Phân tán toàn cầu** (replica qua region)
✅ **Eventual consistency** chấp nhận được (mạng xã hội, caching)
❌ **Hệ thống tài chính** (cần nhất quán mạnh)
❌ **Ngân hàng/tuân thủ** (yêu cầu nhất quán pháp lý)

---

## 4. Cascading (Multi-Level) Replication

### Kiến Trúc

```
[PRIMARY]
    ↓ (nhân bản)
[REPLICA 1]
    ↓ (nhân bản)
[REPLICA 2]
    ↓ (nhân bản)
[REPLICA 3]
```

**Cách Hoạt Động:**

1. Primary nhân bản đến replica 1
2. Replica 1 nhân bản đến replica 2
3. Replica 2 nhân bản đến replica 3
4. Giảm băng thông trên primary
5. Tăng lag nhân bản ở mỗi hop

### Ưu Điểm

✅ **Giảm tải primary** — Primary chỉ nói chuyện với R1
✅ **Hiệu quả băng thông** — Fan-out lớn qua WAN
✅ **Replica có thể mở rộng** — Nhiều replica hơn primary có thể xử lý

### Nhược Điểm

❌ **Lag tăng** — Độ trễ tích lũy ở mỗi level
❌ **Nhiều điểm lỗi** — Nếu R1 thất bại, R2/R3 không cập nhật

### Khi Nào Dùng

✅ **Nhiều replica** (10+) ở các region khác nhau
✅ **Băng thông bị giới hạn** (nhân bản qua WAN)
❌ **Cần lag nhân bản thấp**
❌ **Cần failover tự động**

---

## So Sánh & Cây Quyết Định

### Theo Trường Hợp Sử Dụng

**Dự án mới, OLTP chuẩn?**
→ **Single-Leader** (Đơn giản, đã được chứng minh, đọc có thể mở rộng)

**Multi-datacenter, lưu lượng ghi toàn cầu?**
→ **Multi-Leader** (Chấp nhận phức tạp để ghi local)

**Quy mô cực lớn, eventual consistency ổn?**
→ **Leaderless** (Quy mô Netflix/Facebook)

**Fan-out lớn replica?**
→ **Cascading** (Giữ tải primary thấp)

### Theo Quy Mô

```
0-10 nodes:           Single-Leader (đơn giản, đã được chứng minh)
10-100 nodes:         Multi-Leader hoặc Leaderless
100+ nodes:           Leaderless (được thiết kế cho điều này)
Multi-region (ít):    Multi-Leader
Multi-region (nhiều): Leaderless
```

---

## Checklist Vận Hành

- [ ] Hiểu mô hình nhất quán dữ liệu cho kiến trúc của bạn
- [ ] Biết lag nhân bản trong hệ thống của bạn
- [ ] Kiểm tra quy trình failover thường xuyên
- [ ] Giám sát sức khỏe nhân bản
- [ ] Có runbook cho lag > ngưỡng
- [ ] Tài liệu hóa đường ghi (ghi đi đâu)
- [ ] Xác minh không có ứng dụng nào ghi vào replica
- [ ] Kiểm tra backup + restore độc lập với replication
- [ ] Biết cách dừng replication trong tình huống khẩn cấp
- [ ] Hiểu chiến lược giải quyết xung đột

---

> **Điểm Mấu Chốt:** Loại nhân bản quyết định mô hình vận hành. Single-leader là mặc định; multi-leader và leaderless là giải pháp chuyên biệt cho các vấn đề quy mô/địa lý cụ thể.
