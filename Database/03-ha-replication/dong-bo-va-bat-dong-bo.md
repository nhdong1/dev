# Nhân Bản Đồng Bộ vs Bất Đồng Bộ: Phân Tích Đánh Đổi

Hiểu sự đánh đổi cơ bản giữa độ bền vững và hiệu suất trong nhân bản CSDL.

---

## Đánh Đổi Cốt Lõi

```
ĐỒNG BỘ                              BẤT ĐỒNG BỘ
┌─────────────┐                     ┌─────────────┐
│  Primary    │                     │  Primary    │
│  ghi dữ liệu│                     │  ghi dữ liệu│
└──────┬──────┘                     └──────┬──────┘
       │ chờ xác nhận                      │ tiếp tục ngay
       ↓                                   ↓
┌──────────────┐                    ┌──────────────┐
│   Replica    │                    │   Replica    │
│ nhận + áp    │                    │ nhận + áp    │
│   dụng       │                    │  dụng (lag)  │
└──────────────┘                    └──────────────┘
       │                                   │
   [chậm]                            [nhanh cho primary]
  [an toàn]                           [rủi ro]
```

---

## Ma Trận So Sánh

| Khía cạnh                   | Đồng bộ                | Bất đồng bộ              |
| --------------------------- | ---------------------- | ------------------------ |
| **Độ trễ primary**          | Cao hơn (chặn khi ack) | Thấp hơn (không chờ)     |
| **Throughput**              | Thấp hơn               | Cao hơn                  |
| **RPO**                     | Bằng không (mất 0 dữ liệu) | Khác không (lag phút)  |
| **Độ bền dữ liệu**          | Đảm bảo                | Best-effort              |
| **Nhạy cảm mạng**           | Cao (chặn khi lag)     | Thấp (không chặn)        |
| **Tốt nhất cho**            | Nhất quán mạnh         | Hiệu suất cao            |

---

## 1. Nhân Bản Đồng Bộ

### Những Gì Xảy Ra

```
Primary nhận yêu cầu COMMIT
    ↓
Ghi vào storage local của primary
    ↓
Gửi WAL/thay đổi đến replica
    ↓
Replica nhận và áp dụng thay đổi
    ↓
Replica gửi ACK về
    ↓
Primary nhận ACK
    ↓
Primary trả về "thành công" cho client
    ↓
Primary commit transaction (ACID bền vững)
```

**Tổng thời gian = ghi primary + độ trễ mạng + ghi replica + độ trễ phản hồi**

### Cấu Hình

#### PostgreSQL

```sql
-- Nhân bản đồng bộ: Chờ ACK replica
synchronous_commit = remote_apply

-- Các tùy chọn:
-- OFF: async (mặc định)
-- LOCAL: chờ ghi local (không có replica)
-- REMOTE_WRITE: replica ghi vào storage (chưa áp dụng)
-- REMOTE_APPLY: replica áp dụng (an toàn nhất, chậm nhất)

-- Chỉ định replica nào phải xác nhận
max_wal_senders = 3
synchronous_standby_names = 'replica1,replica2'
```

#### MySQL

```sql
-- MySQL 5.7+: Semisynchronous replication
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- Cài đặt master
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- 10 giây trước khi async

-- Giám sát
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

### Tác Động Hiệu Suất

```
Tình huống: Độ trễ commit của một transaction

Primary ở Hà Nội, Replica ở TP.HCM (độ trễ mạng 50ms)

BẤT ĐỒNG BỘ:
  Ghi primary: 1ms
  Tổng: ~1ms (ngay lập tức)

ĐỒNG BỘ (remote_apply):
  Ghi primary: 1ms
  Gửi đến replica: 50ms
  Ghi replica: 1ms
  ACK replica: 50ms
  Tổng: ~102ms (chậm hơn 100 lần!)
```

### Đảm Bảo Độ Bền Vững

**Đồng bộ = "Tôi sẽ không báo thành công cho đến khi dữ liệu cũng ở trên replica"**

```
Tình huống primary crash:

ĐỒNG BỘ: Tất cả transaction đã ACK ở trên replica
   → Không mất dữ liệu
   → Replica trở thành primary an toàn
   → RPO = 0

BẤT ĐỒNG BỘ: Chỉ các transaction ở primary (lag)
   → Tối đa [thời_gian_lag] dữ liệu bị mất
   → Replica có thể bị chậm
   → RPO = lag_nhân_bản
```

### Nhân Bản Đồng Bộ Có Điều Kiện

#### PostgreSQL Quorum Sync

```sql
-- Cần 1 trong 2 replica xác nhận (k-safe)
synchronous_standby_names = 'ANY 1 (replica1, replica2)'

-- vs cần cả hai
synchronous_standby_names = 'replica1, replica2'
```

### Best Practices cho Nhân Bản Đồng Bộ

1. **Giữ primary và replica gần nhau** (cùng region)
   - Độ trễ phải < 10ms

2. **Giám sát trạng thái đồng bộ**

   ```sql
   -- PostgreSQL
   SELECT * FROM pg_stat_replication;
   ```

3. **Có chiến lược fallback**
   - Timeout → chế độ async
   - Cảnh báo ops → điều tra mạng

4. **Không đồng bộ nhiều replica hơn cần thiết**
   - Mỗi replica thêm độ trễ
   - Quorum sync (ANY 1 of 3) tốt hơn tất cả 3

---

## 2. Nhân Bản Bất Đồng Bộ

### Những Gì Xảy Ra

```
Primary nhận yêu cầu COMMIT
    ↓
Ghi vào storage local của primary
    ↓
Primary trả về "thành công" cho client
    ↓
Primary commit transaction (ACID trên primary)
    ↓
Primary gửi WAL/thay đổi đến replica (background)
    ↓
Replica nhận và áp dụng (trong background)
    ↓
[Không có ACK về cho primary]
```

**Tổng thời gian = chỉ ghi primary (~1-5ms)**

### Cấu Hình

#### PostgreSQL

```sql
-- Nhân bản bất đồng bộ (mặc định)
synchronous_commit = OFF

-- Hoặc rõ ràng hơn
synchronous_commit = LOCAL
synchronous_standby_names = ''
```

#### MySQL

```sql
-- Nhân bản bất đồng bộ (mặc định)
-- Không cần cấu hình đặc biệt

-- Xác minh async đang được dùng
SHOW SLAVE STATUS\G
-- Kiểm tra: Seconds_Behind_Master
```

### Lag Nhân Bản

```
Lag nhân bản async = thời gian giữa ghi trên primary và áp dụng trên replica

                       thời gian
Primary:  T1: INSERT   T2: DELETE   T3: UPDATE
          ←──────────────────────→
Replica:             T1: INSERT   T2: DELETE   T3: UPDATE

Lag = T2 (trên replica) - T1 (trên primary)
Thường: 0-5ms trong hệ thống khỏe, vài giây khi có tải
```

### Rủi Ro Mất Dữ Liệu

```
Tình huống: Primary crash trong khi nhân bản bất đồng bộ

Thời gian  Primary                 Replica
0ms:    Commit ghi thành công    [hàng đợi async]
1ms:    Ghi 2                    [vẫn trong hàng đợi]
5ms:    CRASH! ← Primary thất bại
10ms:   Dữ liệu cũ bị mất       Replica vẫn đang áp dụng

Kết quả: Dữ liệu đã commit BỊ MẤT
         Ứng dụng đã được thông báo "thành công"
         Nhưng dữ liệu chưa được nhân bản

RPO = mạng + thời gian ghi primary ≈ 0-50ms điển hình
```

### Giám Sát Nhân Bản Bất Đồng Bộ

```sql
-- PostgreSQL: Kiểm tra lag
SELECT
    client_addr,
    write_lsn,
    replay_lsn,
    (write_lsn - replay_lsn) as bytes_behind
FROM pg_stat_replication;

-- MySQL: Kiểm tra lag
SHOW SLAVE STATUS\G
-- Tìm: Seconds_Behind_Master
```

---

## 3. Hybrid: Semisynchronous Replication

### Khái Niệm

"Đồng bộ mặc định, async khi cần"

```
Hoạt động bình thường:
  Primary chờ ≥1 replica ACK (ĐỒNG BỘ)

Nếu replica thất bại/chậm:
  Primary timeout (ví dụ: 10 giây)
  Chuyển về BẤT ĐỒNG BỘ
  Kích hoạt cảnh báo
  Ops điều tra
```

### Cấu Hình (MySQL)

```sql
-- Cài đặt plugin semisync
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

-- Trên master
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- 10 giây timeout

-- Trên slave
SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- Giám sát
SHOW STATUS WHERE variable_name LIKE 'Rpl_semi_sync%';
```

### Khi Nào Dùng Semisync

✅ **Muốn an toàn** nhưng không phải toàn bộ chi phí độ trễ
✅ **Có thể chấp nhận async thỉnh thoảng** khi replica ngừng
✅ **Muốn cảnh báo** khi có sự cố
✅ **Hầu hết hệ thống production** được hưởng lợi từ điều này
❌ **Đảm bảo RPO = 0** (một số giai đoạn async)

---

## Tác Động Hiệu Suất Thực Tế

### Test: 1000 transaction, 4KB mỗi cái

```
Tình huống: Primary (10ms CPU) + Replica (20ms đi, 5ms CPU)

BẤT ĐỒNG BỘ:
  Độ trễ primary: ~10ms
  Throughput: ~100 txn/sec
  RPO: ~100ms (lag điển hình)

ĐỒNG BỘ (remote_apply):
  Độ trễ primary: ~50-60ms
  Throughput: ~15 txn/sec
  RPO: 0

SEMISYNC (1 replica, 10 giây timeout):
  Độ trễ bình thường: ~50-60ms
  Khi tải nặng: Giảm ~10ms (fallback async)
  RPO: Thường 0, thỉnh thoảng ~100ms
```

---

## Khung Quyết Định

### Chọn BẤT ĐỒNG BỘ nếu:

- Hiệu suất > an toàn
- RPO vài giây đến phút chấp nhận được
- Ghi có khối lượng cao
- Kết hợp với backup thường xuyên
- Read replica quan trọng hơn failover

### Chọn ĐỒNG BỘ nếu:

- Mất dữ liệu không thể chấp nhận (giao dịch tài chính, tuân thủ)
- Độ trễ < 50ms chấp nhận được
- Cluster nhỏ (1-2 replica)
- Replica cùng region

### Chọn SEMISYNC nếu:

- **Hầu hết hệ thống!** (cân bằng tốt)
- Muốn an toàn với fallback
- Chịu đựng async thỉnh thoảng
- Có thể xử lý độ trễ tăng đột biến 50ms

---

## Cạm Bẫy Vận Hành

### Primary Bị Chặn Do Đồng Bộ

```
Vấn đề: Replica ngừng, primary chặn khi chờ

pg_stat_replication: không có hàng (replica ngắt kết nối)
SELECT * FROM pg_stat_activity;  -- Tất cả truy vấn bị chặn!

Giải pháp:
1. Sửa kết nối replica
2. Hoặc đặt synchronous_standby_names = '' (xóa replica)
3. Hoặc tăng rpl_semi_sync_master_timeout
```

### Khoảng Trống Nhân Bản Bất Đồng Bộ Trong Failover

```
Vấn đề: Replica được thăng cấp thiếu transaction gần đây

Primary có: INSERT 1, INSERT 2, INSERT 3
Replica có: INSERT 1, INSERT 2
Failover: Replica được thăng cấp
Kết quả: INSERT 3 bị mất

Giải pháp: Phát hiện failover tốt hơn
```

---

## Checklist Cấu Hình

**Cho Hệ Thống Production Tính Sẵn Sàng Cao:**

- [ ] Quyết định: Async vs Sync vs Semisync
- [ ] Tài liệu hóa quyết định và lý do
- [ ] Cấu hình timeout cho sync (nếu dùng)
- [ ] Giám sát lag nhân bản liên tục
- [ ] Cảnh báo khi lag > ngưỡng
- [ ] Kiểm tra failover hàng quý
- [ ] Xác minh chiến lược backup độc lập với nhân bản
- [ ] Biết điều gì xảy ra khi replica ngắt kết nối
- [ ] Kiểm tra disaster recovery với lag async
- [ ] Tài liệu hóa RPO chấp nhận được

---

> **Điểm Mấu Chốt:** Đồng bộ = an toàn với chi phí độ trễ. Bất đồng bộ = hiệu suất với chi phí an toàn. Semisync = kết hợp tốt nhất cả hai với fallback. Chọn dựa trên SLA của bạn, không phải trực giác.
