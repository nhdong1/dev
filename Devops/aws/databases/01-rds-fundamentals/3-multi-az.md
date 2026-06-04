# 3 — Multi-AZ Deployment — Triển Khai Đa Vùng Sẵn Sàng

> Multi-AZ (Multi Availability Zone — Đa Vùng Sẵn Sàng) là tính năng High Availability (Tính Sẵn Sàng Cao) cốt lõi của RDS. Nó duy trì một standby instance (Máy Chủ Dự Phòng) đồng bộ trong AZ khác, sẵn sàng tiếp quản khi primary gặp sự cố mà ứng dụng không cần thay đổi connection string.

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
2. [Synchronous Replication — Sao Chép Đồng Bộ](#synchronous-replication--sao-chép-đồng-bộ)
3. [Failover Process — Quy Trình Chuyển Đổi Dự Phòng](#failover-process--quy-trình-chuyển-đổi-dự-phòng)
4. [Multi-AZ vs Single-AZ](#multi-az-vs-single-az)
5. [Multi-AZ DB Cluster — Cụm Đa Vùng](#multi-az-db-cluster--cụm-đa-vùng)
6. [Cấu Hình và Bảo Trì](#cấu-hình-và-bảo-trì)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cơ Chế Hoạt Động

### Kiến Trúc Multi-AZ

```
                    AWS Region (us-east-1)
        ┌─────────────────────────────────────────────────┐
        │                                                  │
        │   AZ-a (Availability Zone A)                     │
        │   ┌─────────────────────────────────────┐        │
        │   │  ┌─────────────────────────────┐   │        │
        │   │  │     RDS Primary Instance     │   │        │
        │   │  │  ●  (Đang Nhận Traffic)      │   │        │
        │   │  └──────────────┬──────────────┘   │        │
        │   │                 │ EBS Volume         │        │
        │   └─────────────────│───────────────────┘        │
        │                     │                             │
        │            Synchronous Replication               │
        │          (Sao Chép Đồng Bộ — mọi write)         │
        │                     │                             │
        │   AZ-b              ▼                             │
        │   ┌─────────────────────────────────────┐        │
        │   │  ┌─────────────────────────────┐   │        │
        │   │  │    RDS Standby Instance      │   │        │
        │   │  │  ○  (Không Nhận Traffic)     │   │        │
        │   │  └─────────────────────────────┘   │        │
        │   │                 │ EBS Volume         │        │
        │   └─────────────────────────────────────┘        │
        │                                                  │
        │   DNS Endpoint: mydb.xxxxxx.rds.amazonaws.com   │
        │   (Trỏ đến Primary — tự động đổi khi failover)  │
        └─────────────────────────────────────────────────┘
```

### Điểm Mấu Chốt

1. **Standby KHÔNG phục vụ read requests** — chỉ là hot standby (Chờ Nóng)
2. **Cùng DNS endpoint** — ứng dụng không cần thay connection string
3. **Tự động failover** — không cần can thiệp thủ công
4. **Backups được lấy từ standby** — không ảnh hưởng I/O của primary

---

## Synchronous Replication — Sao Chép Đồng Bộ

### Cơ Chế Write Với Multi-AZ

```
Application → Primary → Ghi vào Primary storage
                      → Ghi đồng thời vào Standby storage
              Primary ← Xác nhận từ Standby
Application ← Xác nhận thành công
```

**Điều này nghĩa là:**
- Mọi commit chỉ được xác nhận sau khi **cả primary và standby** đã ghi thành công
- **Zero data loss** (Không Mất Dữ Liệu) khi failover — standby luôn đồng bộ 100%
- **Write latency tăng nhẹ** do phải chờ xác nhận từ standby (~1-2 ms thêm)

### So Sánh Synchronous vs Asynchronous

| Đặc Điểm                          | Synchronous (Đồng Bộ) — Multi-AZ | Asynchronous (Bất Đồng Bộ) — Read Replica |
| ---------------------------------- | ----------------------------------- | ------------------------------------------ |
| **Thứ tự xác nhận**               | Sau khi cả hai ghi xong             | Sau khi primary ghi xong                  |
| **Data loss khi failover**        | Zero (Không Có)                     | Có thể mất vài giây (replication lag)     |
| **Write latency**                 | Tăng nhẹ (~1-2 ms)                  | Không bị ảnh hưởng                        |
| **Mục đích**                      | HA / DR                             | Scale đọc                                  |
| **Nhận read traffic**             | Không                               | Có                                         |

---

## Failover Process — Quy Trình Chuyển Đổi Dự Phòng

### Các Tình Huống Kích Hoạt Failover

RDS tự động kích hoạt failover khi phát hiện:

1. **Primary instance failure** — máy chủ primary bị lỗi (hardware, OS crash)
2. **AZ failure** — toàn bộ AZ chứa primary gặp sự cố
3. **Network connectivity loss** — mất kết nối mạng đến primary
4. **Storage failure** — lỗi EBS volume của primary
5. **Manual failover** — người dùng chủ động trigger (cho testing, patching)

### Timeline Failover Chi Tiết

```
T+0s   — Primary gặp sự cố, RDS health check phát hiện lỗi
T+10s  — RDS xác nhận primary không phục hồi được
T+20s  — Bắt đầu quy trình failover
T+30s  — Standby được promote thành primary
T+60s  — DNS TTL (Time To Live — Thời Gian Tồn Tại) cập nhật (thường 5s TTL)
T+60-120s — Ứng dụng reconnect và bắt đầu gửi traffic đến primary mới

Tổng thời gian: 60-120 giây (điển hình)
               Có thể lên đến 3-5 phút trong các tình huống phức tạp
```

### Vì Sao Ứng Dụng Cần Xử Lý Failover

Dù DNS tự cập nhật, ứng dụng cần:

```python
# Xấu — không có retry logic
connection = connect(endpoint, timeout=30)

# Tốt — có exponential backoff retry
def connect_with_retry(endpoint, max_retries=5):
    for attempt in range(max_retries):
        try:
            return connect(endpoint, timeout=10)
        except ConnectionError:
            wait_time = 2 ** attempt  # 1, 2, 4, 8, 16 giây
            time.sleep(wait_time)
    raise Exception("Could not connect after failover")
```

**Cần cài đặt:**
- **Connection timeout** thấp (~10s)
- **Retry logic** với exponential backoff (Tăng Lũy Thừa)
- **Connection pool refresh** sau khi reconnect

---

## Multi-AZ vs Single-AZ

### Khi Nào Dùng Multi-AZ

| Tình Huống                                           | Khuyến Nghị  |
| ----------------------------------------------------- | ------------ |
| Production database, business-critical (Quan Trọng Kinh Doanh) | ✅ Bắt buộc Multi-AZ |
| SLA yêu cầu 99.95%+ availability                    | ✅ Bắt buộc Multi-AZ |
| Workload không thể chịu downtime > 2 phút           | ✅ Bắt buộc Multi-AZ |
| Development / Testing                                 | ❌ Single-AZ (tiết kiệm ~50% chi phí) |
| Staging nếu cần giống production                     | ✅ Multi-AZ  |
| Database nhỏ, không quan trọng                       | ❌ Single-AZ |

### Ảnh Hưởng Đến Hiệu Năng

| Tác Vụ                    | Single-AZ | Multi-AZ       | Lý Do           |
| -------------------------- | --------- | -------------- | --------------- |
| Read latency               | Baseline  | Baseline       | Không ảnh hưởng |
| Write latency              | Baseline  | +1-2 ms        | Chờ xác nhận standby |
| Backup I/O impact          | Có        | Ít hơn         | Backup lấy từ standby |
| Patching downtime          | 5-10 phút | 60-120 giây    | Failover trước khi patch |

---

## Multi-AZ DB Cluster — Cụm Đa Vùng

Đây là tính năng **mới hơn** (từ 2022), khác với Multi-AZ deployment truyền thống.

### So Sánh Hai Loại Multi-AZ

| Tính Năng                            | Multi-AZ Instance (Truyền Thống)  | Multi-AZ DB Cluster (Mới)      |
| ------------------------------------ | ---------------------------------- | ------------------------------- |
| **Số instances**                    | 2 (1 primary + 1 standby)          | 3 (1 writer + 2 readers)        |
| **Standby nhận read traffic**       | ❌                                  | ✅ (2 reader instances)         |
| **Failover time**                   | 60-120 giây                        | < 35 giây                       |
| **Replication**                     | Synchronous (block-level)          | Synchronous (semi-sync)         |
| **Engine hỗ trợ**                   | Tất cả engines                     | MySQL 8.0.28+, PostgreSQL 13.4+ |
| **Chi phí**                         | 2× instance                        | 3× instance                     |
| **Endpoints**                        | 1 endpoint                         | 3 endpoints (writer + 2 readers)|

### Khi Nào Dùng Multi-AZ DB Cluster

- Cần **read scaling** kết hợp **HA**
- Cần failover nhanh hơn (< 35 giây)
- Budget cho phép 3 instances
- Không cần Aurora (muốn giữ standard RDS pricing)

---

## Cấu Hình và Bảo Trì

### Bật Multi-AZ

```bash
# AWS CLI — Bật Multi-AZ khi tạo instance
aws rds create-db-instance \
    --db-instance-identifier mydb \
    --engine mysql \
    --db-instance-class db.m7g.large \
    --master-username admin \
    --master-user-password mypassword \
    --allocated-storage 100 \
    --multi-az \          # Bật Multi-AZ
    --region us-east-1

# Bật Multi-AZ cho instance đang chạy
aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --multi-az \
    --apply-immediately   # Hoặc bỏ để áp dụng trong maintenance window
```

### Maintenance Window (Cửa Sổ Bảo Trì)

- AWS thực hiện patching, hardware maintenance trong maintenance window
- Với Multi-AZ: failover trước → patch standby → failover lại → patch old primary (downtime tối thiểu)
- Cấu hình: chọn khung giờ ít traffic nhất (ví dụ: 2-3 AM Chủ Nhật)

### Backup Window (Cửa Sổ Sao Lưu)

```
Với Multi-AZ:
- Automated backups (Sao Lưu Tự Động) được thực hiện từ standby instance
- Không ảnh hưởng I/O của primary
- Không gây thêm latency cho ứng dụng
```

### Testing Failover

Nên test failover định kỳ (gameday exercises — Buổi Diễn Tập):

```bash
# Trigger manual failover để test
aws rds reboot-db-instance \
    --db-instance-identifier mydb \
    --force-failover      # Failover sang standby

# Theo dõi events (Sự Kiện)
aws rds describe-events \
    --source-identifier mydb \
    --source-type db-instance \
    --duration 60
```

---

## Câu Hỏi Phỏng Vấn

**Q: Multi-AZ có giúp ích gì cho read performance không?**

**Không** — Multi-AZ Instance truyền thống, standby chỉ là hot standby, không phục vụ read requests. Để cải thiện read performance, dùng **Read Replicas**. Chỉ **Multi-AZ DB Cluster** (mới) mới cho phép đọc từ reader instances.

**Q: Khi failover xảy ra, ứng dụng cần làm gì?**

Ứng dụng cần:
1. **Detect lỗi** — connection timeout, connection error
2. **Retry với backoff** — không retry liên tục gây thundering herd (Bầy Đàn Sấm Sét)
3. **Refresh DNS** — một số connection pools cache DNS, cần flush cache
4. **Clear old connections** — trả về error cho requests đang chạy, không treo mãi

**Q: Failover RDS Multi-AZ mất bao lâu? Có thể giảm xuống không?**

- Thông thường: 60-120 giây
- Cải thiện bằng cách:
  - Dùng **RDS Proxy** — proxy duy trì connection pool, ứng dụng reconnect qua proxy nhanh hơn
  - Dùng **Multi-AZ DB Cluster** — failover < 35 giây
  - Dùng **Aurora** — failover < 30 giây (kiến trúc khác biệt hoàn toàn)

**Q: Khác biệt giữa Multi-AZ và Aurora trong trường hợp HA?**

| Tiêu Chí           | RDS Multi-AZ           | Aurora Multi-AZ           |
| ------------------ | ---------------------- | ------------------------- |
| Failover time      | 60-120 giây            | < 30 giây                 |
| Replication        | Block-level (EBS)      | Shared storage (không cần replicate data) |
| Reader scaling     | Không (Multi-AZ Instance) | 15 read replicas          |
| Storage            | Per-instance EBS       | Shared distributed storage |

Aurora có HA vượt trội hơn RDS Multi-AZ, nhưng chi phí cao hơn và ít engine hơn.

---

## 🔗 Điều Hướng

- **Trước:** [2-instance-storage.md](2-instance-storage.md) — Instance & Storage
- **Tiếp theo:** [4-read-replicas.md](4-read-replicas.md) — Read Replicas
- **Liên quan:** [../05-ha-backup/README.md](../05-ha-backup/README.md) — HA & Backup

---

**Cập Nhật Lần Cuối:** 2026-05-15
