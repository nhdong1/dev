# 5 — Aurora High Availability & Failover — Tính Sẵn Sàng Cao và Chuyển Đổi Dự Phòng

> Aurora được thiết kế với tính sẵn sàng cao (High Availability) là nguyên tắc cốt lõi, không phải tính năng tùy chọn. Với storage (lưu trữ) phân tán trên 3 AZs (Availability Zones — Vùng Sẵn Sàng) và cơ chế failover (chuyển đổi dự phòng) nhanh, Aurora đảm bảo SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) 99.99% uptime cho Multi-AZ configurations.

## 📚 Mục Lục

1. [Kiến Trúc HA Của Aurora](#kiến-trúc-ha-của-aurora)
2. [Failover Mechanics — Cơ Chế Chuyển Đổi Dự Phòng](#failover-mechanics--cơ-chế-chuyển-đổi-dự-phòng)
3. [Failover Tiers — Thứ Tự Ưu Tiên Failover](#failover-tiers--thứ-tự-ưu-tiên-failover)
4. [Aurora vs RDS — HA Comparison](#aurora-vs-rds--ha-comparison)
5. [Testing Failover — Kiểm Tra Khả Năng Chuyển Đổi](#testing-failover--kiểm-tra-khả-năng-chuyển-đổi)
6. [Application-Level HA — HA Ở Tầng Ứng Dụng](#application-level-ha--ha-ở-tầng-ứng-dụng)
7. [Monitoring HA — Giám Sát Tính Sẵn Sàng Cao](#monitoring-ha--giám-sát-tính-sẵn-sàng-cao)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kiến Trúc HA Của Aurora

### Tầng Storage — Nền Tảng HA

Aurora HA bắt đầu từ storage layer (tầng lưu trữ), không phải compute layer (tầng tính toán):

```
AZ-a          AZ-b          AZ-c
┌──────┐      ┌──────┐      ┌──────┐
│Copy 1│      │Copy 3│      │Copy 5│  ← Storage nodes
│Copy 2│      │Copy 4│      │Copy 6│
└──────┘      └──────┘      └──────┘

Khả Năng Chịu Lỗi (Fault Tolerance):
  ─ 1 storage node lỗi     → Tự sửa chữa (self-healing), không ảnh hưởng
  ─ 2 storage nodes lỗi    → Vẫn hoạt động (4/6 còn lại)
  ─ 1 AZ hoàn toàn lỗi    → Vẫn hoạt động (4 copies ở 2 AZs còn)
  ─ 2 AZs lỗi             → Read-only (3/6 đủ cho read quorum)
  ─ 3 AZs lỗi             → Không khả dụng (không đủ quorum)
```

### Compute Layer HA (Tầng Tính Toán)

```
Trước Failover:
  AZ-a: Writer Instance   ← nhận tất cả writes
  AZ-b: Reader 1          ← nhận reads từ Reader Endpoint
  AZ-c: Reader 2          ← nhận reads từ Reader Endpoint

Sau Failover (Writer AZ-a lỗi):
  AZ-a: (Không có instance)
  AZ-b: Writer Instance   ← PROMOTED, bây giờ nhận writes
  AZ-c: Reader 1          ← tiếp tục nhận reads
  [Aurora tự tạo Reader mới ở AZ-a sau khi recover]
```

---

## Failover Mechanics — Cơ Chế Chuyển Đổi Dự Phòng

### Quy Trình Failover Chi Tiết

```
T=0: Writer instance bị lỗi hoặc không phản hồi

T=0 đến T=10 giây: Aurora phát hiện lỗi
  - Health check (kiểm tra sức khỏe) thất bại liên tiếp
  - Aurora internal monitoring phát hiện Writer down

T=10 đến T=20 giây: Chọn Reader để promote
  - Aurora chọn Reader có tier (tầng) ưu tiên cao nhất
  - Nếu nhiều Readers cùng tier → chọn Reader có lag thấp nhất

T=20 đến T=30 giây: Promote diễn ra
  - Reader được chọn nhận lệnh promote
  - Cluster endpoint DNS được cập nhật trỏ sang Reader mới (Writer mới)
  - Clients nhận DNS TTL mới sau khi refresh

T=30 giây: Writer mới bắt đầu nhận traffic
  → Tổng thời gian: ~30 giây (thường 15–30 giây)
```

### Tại Sao Nhanh Hơn RDS?

```
RDS Multi-AZ Failover (60–120 giây):
  1. Phát hiện Primary lỗi (30 giây)
  2. Standby cần mount EBS volume của Primary
  3. Standby phải apply redo logs pending
  4. DNS cập nhật
  → Phụ thuộc vào recovery của storage

Aurora Failover (15–30 giây):
  1. Phát hiện Writer lỗi (10 giây)
  2. Reader được chọn để promote
  3. KHÔNG cần mount volume mới (shared storage đã sẵn sàng)
  4. KHÔNG cần apply pending logs (storage layer đã có log)
  5. DNS cập nhật
  → Writer mới chỉ cần thay đổi vai trò — storage không thay đổi
```

---

## Failover Tiers — Thứ Tự Ưu Tiên Failover

### Tier System (Hệ Thống Tầng) Của Aurora

Aurora sử dụng **failover priority tiers** (tầng ưu tiên failover) từ 0 đến 15:

```
Tier 0 (Ưu Tiên Cao Nhất) → Tier 15 (Ưu Tiên Thấp Nhất)

Aurora chọn Reader để promote theo thứ tự:
  1. Tier thấp hơn được ưu tiên (Tier 0 được chọn trước Tier 1)
  2. Nếu nhiều Readers cùng tier → chọn Reader có kích thước instance lớn hơn
  3. Nếu cùng tier và cùng size → chọn ngẫu nhiên
```

### Thiết Lập Tier Theo Best Practice (Thực Hành Tốt Nhất)

```
Cluster 1 Writer + 3 Readers:

  Writer:   db.r6g.2xlarge (AZ-a) — Tier N/A (là Writer)
  Reader 1: db.r6g.2xlarge (AZ-b) — Tier 0  ← Failover target (đích chuyển đổi)
  Reader 2: db.r6g.xlarge  (AZ-c) — Tier 1  ← Failover backup
  Reader 3: db.r6g.large   (AZ-a) — Tier 15 ← Analytics — KHÔNG muốn promote

Lý do:
  - Reader 1 cùng size với Writer → performance không thay đổi sau failover
  - Reader 3 dùng cho analytics → KHÔNG muốn nó thành Writer (size nhỏ hơn)
  - Tier 15 đảm bảo Reader 3 không bao giờ được promote tự động
```

### Cấu Hình Tier Qua CLI (Giao Diện Dòng Lệnh)

```bash
# Đặt failover priority cho Reader instance
aws rds modify-db-instance \
    --db-instance-identifier my-aurora-reader-1 \
    --promotion-tier 0

aws rds modify-db-instance \
    --db-instance-identifier my-aurora-analytics \
    --promotion-tier 15
```

---

## Aurora vs RDS — HA Comparison

### So Sánh Toàn Diện

| Khía Cạnh HA                                           | RDS Multi-AZ                    | Aurora Multi-AZ                    |
| ------------------------------------------------------- | ------------------------------- | ---------------------------------- |
| **Failover time** (Thời Gian Chuyển Đổi)               | 60–120 giây                     | 15–30 giây                         |
| **RPO** (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) | 0 (synchronous replication) | 0 (shared storage không mất data) |
| **Standby phục vụ reads**                              | Không                           | Có (là Reader instance)            |
| **Storage failures tự sửa**                            | Không (cần EBS replace)         | Có (peer-to-peer repair)           |
| **AZ failures tolerated** (Chịu Lỗi AZ)               | 1 AZ (standby ở AZ khác)        | 1 AZ hoàn toàn lỗi — tiếp tục     |
| **Backup không ảnh hưởng production**                  | Một phần (lấy từ standby)       | Hoàn toàn (lấy từ storage)         |
| **Maintenance impact** (Ảnh Hưởng Bảo Trì)            | Restart instance (~ phút)       | Rolling restarts (không downtime)  |

### Aurora Rolling Restart (Khởi Động Lại Luân Phiên)

Aurora hỗ trợ **zero-downtime patching** (vá lỗi không gián đoạn) cho một số loại patch:

```
Database patch thông thường:
  1. Readers lần lượt được restart
  2. Sau khi tất cả Readers đã restart
  3. Writer thực hiện failover nhanh sang một Reader
  4. Reader cũ (giờ là Writer mới) được restart
  5. Failover trở lại Writer ban đầu
  Kết quả: Toàn cluster được patch với downtime < 30 giây
```

---

## Testing Failover — Kiểm Tra Khả Năng Chuyển Đổi

### Tại Sao Phải Test Failover

```
"Backup không được test là backup chưa tồn tại"
→ Tương tự: failover không được test là HA chưa tồn tại

Test failover để:
  - Đo thời gian thực tế (không chỉ lý thuyết)
  - Kiểm tra ứng dụng reconnect đúng cách
  - Kiểm tra connection pool behavior
  - Phát hiện hardcoded hostnames thay vì dùng DNS endpoint
  - Đảm bảo alerts hoạt động khi có failover event
```

### Cách Kích Hoạt Failover Thử Nghiệm

**Phương pháp 1: Reboot với failover (AWS Console/CLI)**
```bash
# Trigger failover bằng cách reboot Writer với failover option
aws rds reboot-db-instance \
    --db-instance-identifier my-aurora-writer \
    --force-failover

# Kiểm tra trạng thái
aws rds describe-db-instances \
    --db-instance-identifier my-aurora-writer \
    --query 'DBInstances[0].DBInstanceStatus'
```

**Phương pháp 2: Fault Injection Queries (Truy Vấn Gây Lỗi Cố Ý)**
```sql
-- Chỉ dùng trên Aurora MySQL — gây lỗi storage engine crash
ALTER SYSTEM CRASH INSTANCE;

-- Trên Aurora PostgreSQL
SELECT aurora_inject_crash('instance');
-- Hoặc simulate writer crash
SELECT aurora_inject_crash('dispatcher');
```

**Phương pháp 3: AWS Fault Injection Simulator — FIS (Công Cụ Giả Lập Lỗi)**
```json
{
  "targets": {
    "aurora-cluster": {
      "resourceType": "aws:rds:cluster",
      "resourceArns": ["arn:aws:rds:...:cluster:my-aurora-cluster"]
    }
  },
  "actions": {
    "failover": {
      "actionId": "aws:rds:failover-db-cluster",
      "targets": { "Clusters": "aurora-cluster" }
    }
  }
}
```

### Những Gì Cần Monitor Khi Test

```
Metrics CloudWatch cần theo dõi trong khi failover:
  ─ EngineUptime (Thời Gian Hoạt Động Engine) — sẽ reset về 0 khi failover
  ─ DatabaseConnections (Số Kết Nối Database) — drop rồi recover
  ─ CommitLatency (Độ Trễ Commit) — tăng cao trong thời gian failover
  ─ ReadLatency/WriteLatency (Độ Trễ Đọc/Ghi) — tăng cao
  ─ FailoverTime (Thời Gian Chuyển Đổi Dự Phòng) — metric tùy chỉnh từ log

Application metrics cần theo dõi:
  ─ Error rate (Tỷ Lệ Lỗi) — tăng trong failover window
  ─ P99 latency (Độ Trễ Phần Vị 99) — spike trong failover
  ─ Successful reconnection rate (Tỷ Lệ Kết Nối Lại Thành Công)
```

---

## Application-Level HA — HA Ở Tầng Ứng Dụng

### Connection String Best Practices (Thực Hành Tốt Nhất Chuỗi Kết Nối)

```python
# SAI — hardcoded instance endpoint (điểm truy cập cứng)
db_host = "my-aurora-writer.xxxxxxxx.us-east-1.rds.amazonaws.com"
# → Không tự động reconnect sau failover

# ĐÚNG — dùng cluster endpoint (điểm truy cập cụm)
db_host = "my-aurora.cluster-xxxxxxxx.us-east-1.rds.amazonaws.com"
# → DNS cập nhật sau failover → ứng dụng reconnect đúng host mới

# TỐT NHẤT — dùng RDS Proxy (Proxy Kết Nối)
db_host = "my-aurora.proxy-xxxxxxxx.us-east-1.rds.amazonaws.com"
# → Proxy tự quản lý reconnect, ứng dụng không bị gián đoạn
```

### Retry Logic (Logic Thử Lại) Khi Failover

```python
import time
import psycopg2
from psycopg2 import OperationalError

def connect_with_retry(host, database, user, password, max_retries=5):
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(
                host=host,
                database=database,
                user=user,
                password=password,
                connect_timeout=10  # Timeout kết nối 10 giây
            )
            return conn
        except OperationalError as e:
            if attempt < max_retries - 1:
                wait_time = (2 ** attempt)  # Exponential backoff (Tăng Chờ Lũy Thừa)
                print(f"Kết nối thất bại, thử lại sau {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### DNS TTL và Reconnection (Thời Gian Sống TTL DNS và Kết Nối Lại)

```
Vấn đề DNS Caching:
  - Aurora cluster endpoint TTL = 5 giây
  - Nhưng nhiều JVM, OS, connection pools cache DNS lâu hơn
  - Nếu DNS cache TTL > 5 giây, ứng dụng vẫn kết nối host cũ

Giải pháp:
  Java: Đặt networkaddress.cache.ttl=1 trong JVM security settings
  Node.js: Tự refresh DNS mỗi 5 giây hoặc dùng keepAlive = false
  Python: Phần lớn không cache DNS lâu — không cần can thiệp
  Tốt nhất: Dùng RDS Proxy — loại bỏ hoàn toàn vấn đề DNS caching
```

### Connection Pool Behavior (Hành Vi Gộp Kết Nối)

```
Vấn đề với Connection Pool:
  - Pool giữ connections mở lâu dài
  - Khi failover, các connections cũ bị terminate (kết thúc)
  - Pool phải detect connections bị chết và tạo mới

Best Practices cho Connection Pool:
  ─ Đặt connection timeout < 30 giây
  ─ Enable connection validation (kiểm tra kết nối) trước khi dùng
  ─ Đặt idle connection timeout (thời gian chờ kết nối rảnh) phù hợp
  ─ Dùng RDS Proxy để proxy layer xử lý reconnections

Ví dụ cấu hình HikariCP (Java Connection Pool):
  maximumPoolSize: 10
  connectionTimeout: 30000   # 30 giây
  idleTimeout: 600000        # 10 phút
  maxLifetime: 1800000       # 30 phút
  connectionTestQuery: SELECT 1
  keepaliveTime: 30000       # Giữ kết nối alive mỗi 30 giây
```

---

## Monitoring HA — Giám Sát Tính Sẵn Sàng Cao

### CloudWatch Metrics Quan Trọng

| Metric                                                 | Ngưỡng Cảnh Báo              | Ý Nghĩa                                      |
| ------------------------------------------------------- | ----------------------------- | --------------------------------------------- |
| **EngineUptime** (Thời Gian Hoạt Động)                 | Reset về 0                    | Failover đã xảy ra                           |
| **AuroraReplicaLag** (Độ Trễ Bản Sao)                 | > 100 ms                      | Reader lag cao — ảnh hưởng read consistency   |
| **DatabaseConnections** (Số Kết Nối)                   | > 80% max connections         | Nguy cơ connection exhaustion (hết kết nối)  |
| **CommitThroughput** (Thông Lượng Commit)              | Giảm đột ngột > 50%           | Có thể đang failover hoặc có vấn đề           |
| **DiskQueueDepth** (Độ Sâu Hàng Đợi Đĩa)             | > 1                           | I/O bottleneck                                |
| **FreeableMemory** (Bộ Nhớ Có Thể Giải Phóng)        | < 100 MB                      | Nguy cơ out-of-memory                         |
| **CPUUtilization** (Mức Sử Dụng CPU)                  | > 80% kéo dài                 | Cần scale up hoặc thêm Reader                 |

### Events Cần Subscribe (Đăng Ký Sự Kiện)

```bash
# Tạo SNS topic cho Aurora events
aws rds create-event-subscription \
    --subscription-name aurora-ha-alerts \
    --sns-topic-arn arn:aws:sns:us-east-1:123456789:aurora-alerts \
    --source-type db-cluster \
    --event-categories '["failover","notification","maintenance"]' \
    --source-ids my-aurora-cluster

# Events quan trọng:
#   RDS-EVENT-0049: Failover started (Bắt đầu chuyển đổi dự phòng)
#   RDS-EVENT-0050: Failover completed (Hoàn thành chuyển đổi)
#   RDS-EVENT-0069: Failover to Read Replica started
#   RDS-EVENT-0070: Failover to Read Replica completed
```

### Dashboard HA Cần Có

```
Panel 1: Cluster Status
  - Writer/Reader instance list với trạng thái
  - Current Writer AZ

Panel 2: Availability Metrics
  - EngineUptime (alarm khi reset)
  - DatabaseConnections vs Max
  - AuroraReplicaLag cho mỗi Reader

Panel 3: Performance During Failover
  - CommitLatency timeline
  - ReadLatency timeline
  - Annotation khi failover events xảy ra

Panel 4: Storage Health
  - VolumeBytesUsed vs 128 TB max
  - VolumeReadIOPs, VolumeWriteIOPs
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích quy trình Aurora failover từ đầu đến cuối

**Trả lời:** Khi Aurora Writer instance lỗi: (1) Aurora internal health checks phát hiện Writer không phản hồi sau ~10 giây; (2) Aurora chọn Reader để promote — dựa trên failover priority tier và instance size; (3) Reader được promote thành Writer mới (~10 giây); (4) Aurora cập nhật DNS của cluster endpoint trỏ sang Writer mới (~5 giây); (5) Clients với DNS TTL ngắn kết nối lại với Writer mới. Tổng thời gian ~15–30 giây. Dữ liệu không bị mất vì Writer mới đọc từ cùng shared storage volume và storage đã có đủ data từ quorum writes.

### Q2: Tại sao nên đặt Failover Tier 15 cho Reader dùng cho analytics?

**Trả lời:** Reader analytics thường chạy trên instance nhỏ hơn (tiết kiệm chi phí) và không được tối ưu cho OLTP writes. Nếu nó được promote thành Writer sau failover: (1) Instance nhỏ không đủ capacity xử lý write traffic; (2) Connection pool của ứng dụng OLTP sẽ gửi writes đến instance không đủ mạnh; (3) Có thể gây degraded performance hoặc thậm chí cascade failure. Đặt Tier 15 đảm bảo Aurora sẽ không bao giờ tự động chọn nó làm Writer — nó chỉ được promote nếu KHÔNG còn Reader nào khác.

### Q3: RDS Proxy giúp gì cho Aurora HA?

**Trả lời:** RDS Proxy (Proxy Cơ Sở Dữ Liệu Đám Mây) đứng giữa ứng dụng và Aurora cluster, xử lý connection pooling (gộp kết nối) và automatic reconnection (kết nối lại tự động). Khi Aurora failover xảy ra: (1) RDS Proxy duy trì connections từ ứng dụng — ứng dụng không thấy connection drop; (2) Proxy tự reconnect đến Writer mới sau khi DNS cập nhật; (3) Ứng dụng chỉ thấy brief request timeout (hết thời gian chờ tạm thời) thay vì connection errors. Đặc biệt hữu ích cho Lambda functions và microservices với nhiều instances tạo connection cùng lúc.

### Q4: Làm sao để đảm bảo application không bị impact khi Aurora failover xảy ra trong production?

**Trả lời:** Chiến lược toàn diện: (1) **Infrastructure**: Dùng RDS Proxy để xử lý reconnection tự động; đặt Failover Tier đúng cho Readers; đảm bảo có ít nhất 1 Reader ở AZ khác Writer. (2) **Application code**: Dùng cluster endpoint (không hardcode instance endpoint); implement retry logic với exponential backoff; đặt DNS TTL ngắn trong JVM settings. (3) **Testing**: Test failover định kỳ (ít nhất 1 lần/quý) và đo actual RTO; dùng AWS Fault Injection Simulator để simulate; review CloudWatch dashboard trong và sau khi failover. (4) **Monitoring**: Subscribe RDS events cho failover notifications; set alerts trên EngineUptime reset; measure actual user-facing error rate khi failover.

---

## 🔗 Điều Hướng

| Trước                                          | Tiếp Theo                              |
| ---------------------------------------------- | --------------------------------------- |
| [4-aurora-vs-rds.md](./4-aurora-vs-rds.md)    | [03-dynamodb/](../03-dynamodb/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
