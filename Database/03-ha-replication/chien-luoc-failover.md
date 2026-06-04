# Chiến Lược Failover: Từ Thủ Công Đến Hoàn Toàn Tự Động

Nắm vững quy trình failover để giảm thiểu thời gian chết và ngăn ngừa split-brain.

---

## Tổng Quan

**Failover = Phục hồi sau khi primary CSDL thất bại bằng cách thăng cấp replica lên primary**

### Các Chỉ Số Quan Trọng

| Chỉ số   | Định nghĩa                      | Mục tiêu                                           |
| -------- | ------------------------------- | -------------------------------------------------- |
| **RTO**  | Recovery Time Objective         | Bao lâu sau thất bại thì ghi khả dụng?            |
| **RPO**  | Recovery Point Objective        | Có thể mất bao nhiêu dữ liệu?                     |
| **MTTR** | Mean Time To Repair             | Mất bao lâu để khôi phục hoàn toàn dự phòng?     |

### Ví dụ: SLA Uptime 99.99%

```
Ngân sách downtime: 52 phút/năm ≈ 4 phút/tháng

Nếu RTO = 1 phút:
  → ~52 lỗi/năm có thể chấp nhận
  → Khả thi với failover bán tự động

Nếu RTO = 30 giây:
  → ~104 lỗi/năm có thể chấp nhận
  → Cần phát hiện + thăng cấp tự động

Nếu RTO = 5 giây:
  → ~625 lỗi/năm có thể chấp nhận
  → Cần failover tự động ngay lập tức
  → Không thực tế với sự tham gia của con người
```

---

## 1. Failover Thủ Công

### Quy Trình

```
1. Giám sát phát hiện primary ngừng hoạt động
   └─ Kết nối CSDL thất bại
   └─ Health check timeout
   └─ Cảnh báo gửi đến DBA on-call

2. DBA xác minh primary thực sự ngừng
   └─ SSH đến primary, thử kết nối
   └─ Kiểm tra logs: crash? hung process?
   └─ Quyết định: thực sự ngừng? hay network blinks tạm thời?

3. DBA xác minh replica đã bắt kịp
   └─ Kiểm tra lag nhân bản
   └─ Nếu lag > 0: phải quyết định (mất dữ liệu? chờ?)

4. DBA thăng cấp replica lên primary
   └─ Chạy script thăng cấp
   └─ Dừng nhân bản
   └─ Đặt chế độ read-write
   └─ Xác minh primary mới chấp nhận ghi

5. DBA cập nhật connection strings
   └─ Cập nhật config ứng dụng
   └─ Cập nhật DNS
   └─ Cập nhật mục tiêu giám sát

6. DBA giám sát vấn đề
   └─ Xác minh ứng dụng đã kết nối
   └─ Kiểm tra hiệu suất
   └─ Giám sát ổn định hệ thống
```

### Thời Gian Điển Hình

```
Lỗi CSDL thông thường:
5s   → Phát hiện vấn đề (health check timeout)
30s  → DBA nhận cảnh báo
60s  → DBA xác nhận lỗi
120s → DBA chạy script thăng cấp
180s → Ứng dụng nhận DNS mới (TTL=60s)
300s → Hệ thống đã phục hồi, giám sát ổn định

Tổng RTO: ~5 phút
```

### Ưu Điểm

✅ **Phán đoán con người** — Có thể xử lý edge cases
✅ **An toàn** — Xác minh rõ ràng trước khi thăng cấp
✅ **Đơn giản** — Cần ít tự động hóa
✅ **Phanh khẩn cấp** — DBA có thể dừng thăng cấp

### Nhược Điểm

❌ **Chậm** — RTO tính bằng phút
❌ **Chi phí on-call** — Cần DBA trực tiếp
❌ **Lỗi con người** — Nhầm lệnh trong sự cố
❌ **Không mở rộng** — 10 lỗi/ngày = kiệt sức

### Mẫu Runbook Failover Thủ Công

```markdown
# Runbook Failover CSDL Production

## Phát Hiện Lỗi

- Hệ thống cảnh báo kích hoạt: "Primary ngừng"
- Xác nhận cảnh báo trong PagerDuty
- Mở war room (kênh Slack)

## Xác Minh (30 giây)

```bash
# 1. Xác minh primary thực sự ngừng
ping -c 3 db-primary-prod.internal
timeout 5 psql -h db-primary-prod.internal -U postgres -c "SELECT 1" || echo "THẤT BẠI"

# 2. Kiểm tra trạng thái replica
ssh db-replica-prod.internal
psql -U postgres -c "SELECT now() - pg_last_xact_replay_timestamp() as replication_lag;"
```

## Thăng Cấp (2 phút)

```bash
# 3. Trên replica, thăng cấp lên primary
ssh db-replica-prod.internal
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main

# 4. Xác minh đã là primary
psql -U postgres -c "SELECT pg_is_in_recovery();"
# Phải trả về: f (false = KHÔNG trong recovery = là primary)

# 5. Xác minh có thể chấp nhận ghi
psql -U postgres -c "CREATE TABLE test_table (id INT); DROP TABLE test_table;"
```

## Cập Nhật Kết Nối (1 phút)

```bash
# 6. Cập nhật DNS
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch "file://update-dns.json"

# 7. Khởi động lại ứng dụng (hoặc kích hoạt kết nối lại)
kubectl set env deployment/api \
  DB_HOST=db-replica-prod.internal \
  ROLLOUT_RESTART=true
```
```

---

## 2. Failover Bán Tự Động

### Quy Trình

```
Tự động hóa phát hiện lỗi + chạy script thăng cấp
DBA phê duyệt thăng cấp
Ứng dụng được thông báo + kết nối lại
```

### Thời Gian

```
5s   → Health check thất bại (đồng thuận)
15s  → Bắt đầu thăng cấp replica
30s  → Thăng cấp hoàn thành
45s  → DNS/service discovery được cập nhật
60s  → Ứng dụng bắt đầu kết nối lại
90s  → Hầu hết kết nối đã di chuyển

Tổng RTO: ~2 phút
```

### Ưu Điểm

✅ **Nhanh hơn thủ công** (RTO: 1-2 phút)
✅ **Có thể lặp lại** — Tự động hóa giảm lỗi
✅ **24/7 không cần on-call** — Chạy tự động
✅ **Vẫn có an toàn** — Cần trigger rõ ràng

### Các Công Cụ Bán Tự Động

- **PostgreSQL**: Patroni + etcd, pg_auto_failover
- **MySQL**: Orchestrator, MHA (MySQL HA)
- **MongoDB**: Built-in replica set auto-failover
- **Cloud RDS**: AWS RDS Multi-AZ automatic failover
- **Kubernetes**: Operators (CrunchyData PostgreSQL Operator)

---

## 3. Failover Hoàn Toàn Tự Động

### Quy Trình

```
Quorum phát hiện lỗi
Replica tự động được thăng cấp
Ứng dụng kết nối lại trong suốt
Không cần sự can thiệp của con người
```

### Kiến Trúc

```
┌─────────────────────────────────────────────────┐
│             Tầng Ứng Dụng                        │
│  (Connection pooling, tự động kết nối lại)       │
└────────────┬─────────────────────────────────┬──┘
             │                                 │
    ┌────────▼────────┐              ┌─────────▼────────┐
    │  Read Replica 1 │              │ Read Replica 2   │
    └────────┬────────┘              └─────────┬────────┘
             └───────────┬───────────────────┘
                         │ Nhân bản
                         ▼
              ┌──────────────────────┐
              │   Primary (Active)   │
              │   [với watchdog]     │
              └──────────┬───────────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
           ▼             ▼            ▼
       [Replica1]   [Replica2]   [Replica3]
     (standby)      (standby)    (standby)
        │              │           │
        └──────────────┴───────────┘
               Đồng thuận quorum
         (etcd/Zookeeper/khác)
```

### Cách Hoạt Động

```
Trạng thái bình thường:
Primary: Leader (chấp nhận ghi)
Replicas: Standby (streaming replication)
Quorum: Lưu vị trí leader

Phát hiện Primary Thất Bại:
Heartbeat primary hết hạn
Quorum leader timeout (ví dụ: 30 giây)
Quorum kích hoạt bầu chọn failover

Thăng Cấp Replica:
Replica ưu tiên cao nhất được bầu
Replica dừng nhân bản (trở thành leader)
Replica bắt đầu chấp nhận ghi
Quorum được cập nhật với leader mới

Định Tuyến Lại Kết Nối:
Ứng dụng có smart driver (auto-failover driver)
Driver thông báo với quorum về vị trí primary mới
Driver tự động kết nối lại
Trong suốt với ứng dụng!
```

### Triển Khai (PostgreSQL + Patroni + etcd)

```yaml
# patroni.yml - Bật failover tự động
patroni:
  ttl: 30
  loop_wait: 10
  retry_timeout: 10
  maximum_lag_on_failover: 1000000

  watchdog:
    mode: automatic
    device: /dev/watchdog
    safety_margin: 5

postgresql:
  synchronous_commit: remote_apply
  synchronous_standby_names: "patroni"
```

```bash
# Khởi động Patroni trên mỗi node
patronictl -c patroni.yml start
patronictl members
```

### Thời Gian

```
5s   → Heartbeat primary không được quorum phát hiện
10s  → Quorum nhận ra primary ngừng (TTL hết hạn)
15s  → Failover được kích hoạt, replica được thông báo
20s  → Replica tốt nhất được thăng cấp
25s  → Quorum được cập nhật với leader mới
30s  → Driver ứng dụng phát hiện thay đổi
35s  → Ứng dụng kết nối lại
40s  → Traffic đọc đang chạy

Tổng RTO: ~30-40 giây
Tổng RPO: ~0 (nếu sync replication)
```

### Ưu Điểm

✅ **Nhanh nhất** — RTO < 1 phút
✅ **Không cần con người** — Hoàn toàn hands-off
✅ **Có thể mở rộng** — Hoạt động với 10 lỗi/ngày
✅ **Trong suốt** — Ứng dụng không biết

### Nhược Điểm

❌ **Phức tạp** — Cơ sở hạ tầng đáng kể
❌ **Cần quorum** — Ít nhất 3 node
❌ **Rủi ro phân vùng mạng** — Có thể gây split-brain nếu quorum thất bại
❌ **Khó debug** — Nhiều moving parts

---

## 4. Ngăn Ngừa Split-Brain

### Vấn Đề

```
Tình huống: Primary và replica mất kết nối mạng

Trước: [Primary] ←→ [Replica]
Mạng ngừng: [Primary] ✗ [Replica]

Cả hai nghĩ "bên kia đã chết"

Primary tiếp tục: "Chấp nhận ghi, replica sẽ bắt kịp"
Replica tiếp tục: "Bắt đầu chấp nhận ghi (tôi là primary bây giờ)"

Kết quả: HAI primary với dữ liệu phân kỳ!

[Primary chấp nhận ghi A]
[Replica (nghĩ là primary) chấp nhận ghi B]

Xung đột! Dữ liệu nào đúng?
```

### Chiến Lược Ngăn Ngừa

#### Chiến Lược 1: Quorum (Tốt nhất)

```
Yêu cầu đồng thuận đa số để thăng cấp

Trong cluster 3 node:
Primary + Replica1 + Replica2

Phân vùng mạng: Primary ↔ [Replica1, Replica2]

Phía Primary:
  - Chỉ thấy bản thân
  - Cần 2/3 phiếu để làm primary
  - Chỉ có 1 phiếu → Không thể làm primary
  - CHẶN ghi

Phía Replica:
  - Thấy Replica1 + Replica2
  - Có 2/3 phiếu → CÓ THỂ làm primary
  - Thăng cấp Replica1 lên primary
  - Tiếp tục

Kết quả: Chỉ một phía có thể ghi!
         Không có split-brain!
```

#### Chiến Lược 2: Fencing (Buộc Primary Cũ Offline)

```sql
Sau khi thăng cấp replica:

1. DNS được cập nhật trỏ đến primary mới
2. Chạy lệnh trên PRIMARY CŨ (nếu truy cập được):

-- Trên primary cũ
ALTER SYSTEM SET default_transaction_read_only = ON;
SELECT pg_reload_conf();
-- Bây giờ KHÔNG THỂ chấp nhận ghi
-- Ngay cả khi nó trở lại online
```

### Phát Hiện Split-Brain

```sql
-- PostgreSQL: Sau failover, xác minh primary đơn
SELECT datname, usename, application_name, state
FROM pg_stat_replication;

-- Nên hiển thị:
-- 1. Primary mới có replica đang streaming
-- 2. Primary cũ (nếu up) hiển thị 0 replica
-- 3. Kiểm tra logs cho lỗi FATAL xung đột

-- Nếu thấy cả hai chấp nhận ghi:
-- NGHIÊM TRỌNG: Split-brain đã xảy ra!
-- Dừng một cái ngay:
ALTER SYSTEM SET default_transaction_read_only = ON;
SELECT pg_reload_conf();
```

---

## 5. Chọn Chiến Lược Failover

### Ma Trận Quyết Định

```
Có thể chấp nhận 5-10 phút RTO?
├─ CÓ → Failover thủ công (đơn giản, đã kiểm tra)
└─ KHÔNG → Tiếp tục

Có thể chấp nhận 2-5 phút RTO?
├─ CÓ → Bán tự động (Patroni, Orchestrator)
└─ KHÔNG → Tiếp tục

Có thể chấp nhận <1 phút RTO?
├─ CÓ → Hoàn toàn tự động + smart driver
└─ KHÔNG → Xem xét lại kiến trúc
```

### Theo Quy Mô Tổ Chức

```
Startup:
├─ Failover thủ công ổn (đội ops 1-2 người)
├─ Kiểm tra hàng quý
└─ Downtime 30 phút - 1 giờ chấp nhận được

Tăng trưởng (10-50 kỹ sư):
├─ Bán tự động (Patroni)
├─ Vẫn có sự tham gia on-call
└─ RTO: 5-10 phút

Doanh nghiệp:
├─ Hoàn toàn tự động + smart drivers
├─ Thiết lập multi-region
└─ RTO: <1 phút

Tài chính/Y tế:
├─ Dự phòng cực cao
├─ Nhiều tầng failover
└─ RTO: <30 giây
```

---

## Kiểm Tra Failover

### Diễn Tập Failover Hàng Quý (Game Day)

```bash
#!/bin/bash
# Mô phỏng primary thất bại

set -e

echo "Bắt đầu game day failover..."
echo "1. Dừng PostgreSQL primary"
systemctl stop postgresql

echo "2. Chờ 30 giây để giám sát phát hiện..."
sleep 30

echo "3. Kiểm tra failover có xảy ra tự động không"
psql -h replica-endpoint -c "SELECT pg_is_in_recovery();"
# Phải trả về 'f' (không trong recovery = là primary)

echo "4. Thử ghi vào primary mới"
psql -h replica-endpoint -c "INSERT INTO game_day_log VALUES (now(), 'Test write');"

echo "5. Xác minh ứng dụng đã kết nối lại"
curl http://localhost:8080/health
# Phải trả về healthy

echo "Game day thành công!"
```

---

## Checklist Failover

- [ ] Biết chiến lược failover (thủ công/bán tự/tự động)
- [ ] RTO/RPO được tài liệu hóa và truyền đạt cho nghiệp vụ
- [ ] Script thăng cấp được kiểm tra hàng tháng
- [ ] DNS failover được xác minh hoạt động
- [ ] Ngăn ngừa split-brain được cấu hình
- [ ] Kiểm tra bắt kịp replica (với lag > 0)
- [ ] Cảnh báo giám sát được cấu hình cho primary thất bại
- [ ] Runbook on-call được chuẩn bị và xem xét
- [ ] Nhóm được đào tạo về quy trình failover (game day hàng năm)
- [ ] Thời gian phục hồi từ failover đến HA đầy đủ được tài liệu hóa
- [ ] Chiến lược backup độc lập với failover đã được kiểm tra

---

> **Điểm Mấu Chốt:** Failover thủ công chậm nhưng an toàn (cho đội nhỏ). Bán tự động là điểm ngọt ngào (giảm gánh nặng on-call, dễ kiểm tra). Hoàn toàn tự động cần quorum + smart drivers (phức tạp nhưng xử lý quy mô lớn).
