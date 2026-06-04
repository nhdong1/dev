# 4 — Read Replicas — Bản Sao Đọc

> Read Replicas (Bản Sao Đọc) cho phép scale out (Mở Rộng Theo Chiều Ngang) khả năng đọc của RDS bằng cách tạo ra các bản sao read-only của database chính. Đây là giải pháp tự nhiên khi ứng dụng có read-heavy workload (Khối Lượng Công Việc Nặng Về Đọc).

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
2. [Asynchronous Replication — Sao Chép Bất Đồng Bộ](#asynchronous-replication--sao-chép-bất-đồng-bộ)
3. [Số Lượng và Loại Read Replicas](#số-lượng-và-loại-read-replicas)
4. [Cross-Region Read Replicas — Bản Sao Xuyên Vùng](#cross-region-read-replicas--bản-sao-xuyên-vùng)
5. [Promoting Read Replica — Nâng Cấp Bản Sao Thành Chính](#promoting-read-replica--nâng-cấp-bản-sao-thành-chính)
6. [Use Cases — Trường Hợp Sử Dụng](#use-cases--trường-hợp-sử-dụng)
7. [Replication Lag — Độ Trễ Sao Chép](#replication-lag--độ-trễ-sao-chép)
8. [Read Replicas vs Multi-AZ](#read-replicas-vs-multi-az)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cơ Chế Hoạt Động

### Kiến Trúc Read Replicas

```
                      AWS Region (us-east-1)
        ┌──────────────────────────────────────────────────────┐
        │                                                       │
        │  ┌─────────────────────────────────────────────┐    │
        │  │            Primary RDS Instance              │    │
        │  │         (Nhận đọc VÀ ghi)                   │    │
        │  └────────────┬─────────────────────────────────┘   │
        │               │                                       │
        │    Async Replication (Bất Đồng Bộ, Binary Log)      │
        │               │                                       │
        │    ┌──────────┼──────────┐                           │
        │    ▼          ▼          ▼                           │
        │  ┌───────┐  ┌───────┐  ┌───────┐                   │
        │  │Read   │  │Read   │  │Read   │                   │
        │  │Replica│  │Replica│  │Replica│                   │
        │  │#1     │  │#2     │  │#3     │                   │
        │  │(Read) │  │(Read) │  │(Read) │                   │
        │  └───────┘  └───────┘  └───────┘                   │
        └──────────────────────────────────────────────────────┘

Application routing:
  Write requests  → Primary endpoint
  Read requests   → Read Replica endpoints (round-robin hoặc custom logic)
```

### Điểm Khác Biệt Với Multi-AZ

- Read Replicas có **endpoint riêng** — ứng dụng phải chỉ định tường minh
- Dữ liệu có thể **không hoàn toàn đồng bộ** với primary (replication lag)
- Phục vụ được **read traffic thực tế**

---

## Asynchronous Replication — Sao Chép Bất Đồng Bộ

### Cơ Chế Binary Log Replication

Với MySQL/MariaDB:

```
Primary ghi transaction → Ghi vào Binary Log (Nhật Ký Nhị Phân)
                                │
                                ▼ (bất đồng bộ, không chờ)
Replica IO thread ← Kéo Binary Log events từ Primary
        │
        ▼
Replica SQL thread áp dụng events vào local database

Thời gian trễ = Thời gian IO thread kéo + Thời gian SQL thread áp dụng
```

Với PostgreSQL: sử dụng **WAL (Write-Ahead Log — Nhật Ký Ghi Trước) Streaming Replication**.

### Hệ Quả Của Asynchronous Replication

**Eventual Consistency (Nhất Quán Cuối Cùng):** Data trên replica có thể chậm hơn primary vài millisecond đến vài giây.

**Thiết kế ứng dụng cần lưu ý:**

```python
# Xấu — đọc ngay sau khi ghi từ replica
def create_order(user_id, items):
    order_id = db_primary.insert_order(user_id, items)
    order = db_replica.get_order(order_id)  # Có thể chưa thấy!
    return order

# Tốt — đọc từ primary ngay sau khi ghi
def create_order(user_id, items):
    order_id = db_primary.insert_order(user_id, items)
    order = db_primary.get_order(order_id)  # Luôn thấy
    return order

# Tốt — đọc từ replica cho non-critical reads
def get_product_list():
    return db_replica.get_all_products()  # Ổn nếu hơi cũ vài giây
```

---

## Số Lượng và Loại Read Replicas

### Giới Hạn Theo Engine

| Engine          | Số Replica Tối Đa | Replica của Replica | Storage Type Yêu Cầu |
| --------------- | ----------------- | -------------------- | --------------------- |
| **MySQL**       | 5                 | ✅ (cascading)       | Không hạn chế         |
| **PostgreSQL**  | 5                 | ❌                   | Không hạn chế         |
| **MariaDB**     | 5                 | ✅                   | Không hạn chế         |
| **Oracle**      | 5                 | ❌                   | Không hạn chế         |
| **SQL Server**  | 5                 | ❌                   | Chỉ hỗ trợ Enterprise |

> **Lưu ý:** Aurora hỗ trợ **15 read replicas** — một lý do lớn để chọn Aurora khi cần scale đọc nhiều.

### Instance Class của Read Replica

- Read replica có thể dùng **instance class khác** với primary
- Thường dùng instance nhỏ hơn cho replicas phục vụ reporting (vì queries lớn hơn nhưng ít hơn)
- Hoặc instance lớn hơn nếu replica chạy queries phức tạp

```bash
# Tạo read replica với instance class khác
aws rds create-db-instance-read-replica \
    --db-instance-identifier mydb-replica-1 \
    --source-db-instance-identifier mydb-primary \
    --db-instance-class db.r7g.large \  # Khác với primary
    --region us-east-1
```

---

## Cross-Region Read Replicas — Bản Sao Xuyên Vùng

### Kiến Trúc Cross-Region

```
us-east-1 (Vùng Chính — Primary Region)
├── Primary RDS Instance (Nhận mọi ghi)
│   │
│   └── Async replication (qua AWS backbone network)
│
ap-southeast-1 (Singapore — Vùng Phụ)
├── Read Replica #1 (Phục vụ users châu Á)
│
eu-west-1 (Ireland — Vùng Châu Âu)
└── Read Replica #2 (Phục vụ users châu Âu)
```

### Use Cases Cross-Region Replica

1. **Read performance cho users địa lý xa** — giảm latency bằng cách đọc gần người dùng hơn
2. **Disaster Recovery (Khôi Phục Thảm Họa)** — có sẵn dữ liệu tại vùng khác nếu vùng chính gặp sự cố
3. **Data compliance (Tuân Thủ Dữ Liệu)** — yêu cầu dữ liệu phải tồn tại ở một vùng địa lý cụ thể

### Chi Phí Cross-Region Replica

- **Data transfer** (Truyền Dữ Liệu) cross-region tốn phí ($0.02/GB từ us-east-1 sang ap-southeast-1)
- Instance chạy ở region khác tính phí region đó
- Tổng chi phí đáng kể — chỉ dùng khi thực sự cần

---

## Promoting Read Replica — Nâng Cấp Bản Sao Thành Chính

### Khi Nào Promote

1. **Disaster Recovery** — primary region gặp sự cố toàn khu vực
2. **Database migration** — tạo replica ở region/AZ mới, rồi promote
3. **Blue/green deployment** — tạo replica của production, upgrade, rồi promote và cutover

### Quy Trình Promote

```
1. Dừng replication (replication lag về 0 nếu có thể)
2. Promote replica thành standalone DB
   - Replica không còn nhận data từ primary
   - Replica trở thành read/write database
3. Cập nhật application connection string sang endpoint mới
4. Xóa hoặc archive primary cũ
```

```bash
# Promote read replica thành standalone DB
aws rds promote-read-replica \
    --db-instance-identifier mydb-replica-1 \
    --backup-retention-period 7

# Kiểm tra trạng thái
aws rds describe-db-instances \
    --db-instance-identifier mydb-replica-1 \
    --query 'DBInstances[0].DBInstanceStatus'
```

### Lưu Ý Khi Promote

- **Không thể undo** — một khi đã promote, không thể thêm replica trở lại thành replication target
- **Replication lag** tại thời điểm promote = data loss tiềm ẩn
- **Backup retention** cần cấu hình lại sau promote

---

## Use Cases — Trường Hợp Sử Dụng

### 1. Tách Read/Write Traffic (Read/Write Splitting)

```
Phù hợp cho: E-commerce, social platforms — nhiều reads hơn writes nhiều lần

Architecture:
  Writes:          → Primary (1 instance)
  Product reads:   → Replica #1 (instance nhỏ)
  User analytics:  → Replica #2 (instance lớn cho queries phức tạp)
  Reports/exports: → Replica #3 (instance lớn, isolated)
```

### 2. Reporting Database (Cơ Sở Dữ Liệu Báo Cáo)

```
Vấn đề: Reports chạy OLAP queries phức tạp, table scans lớn
→ Làm chậm primary serving OLTP traffic

Giải pháp: Dedicated Read Replica cho reporting
- Replica chạy instance class lớn hơn (nhiều RAM hơn)
- Reports chạy trên replica — không ảnh hưởng production
- Có thể delay replication nhẹ — OK vì reports không cần real-time
```

### 3. Phát Triển & Testing (Development & Testing)

```
Dev team cần access database production-like:
- Tạo read replica
- Dev kết nối vào replica để test queries
- Không rủi ro ảnh hưởng production
- Có thể promote thành standalone DB cho staging environment
```

### 4. Cross-Region DR (Disaster Recovery — Khôi Phục Thảm Họa)

```
RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục): Vài giây đến vài phút
RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục): 15-30 phút (promote + DNS change)

Tốt hơn không có gì, nhưng Aurora Global Database cho RPO/RTO tốt hơn nhiều.
```

---

## Replication Lag — Độ Trễ Sao Chép

### Đo Lường Replication Lag

```
CloudWatch Metric: ReplicaLag
  - Đơn vị: giây
  - Cảnh báo khi lag > 30 giây (thông thường)
  - Ngưỡng nguy hiểm: > 5 phút

MySQL: SHOW REPLICA STATUS\G
  → Seconds_Behind_Source: 0    # Hoàn hảo
  → Seconds_Behind_Source: 30   # Bình thường dưới tải
  → Seconds_Behind_Source: 300  # Cần điều tra
```

### Nguyên Nhân Lag Tăng Cao

1. **Write-heavy primary** — replica không apply kịp tốc độ ghi
2. **Long-running queries trên replica** — block SQL thread
3. **Replica instance quá nhỏ** — không đủ CPU/RAM để apply replication
4. **Network latency** — đặc biệt với cross-region replica
5. **DDL operations** (ALTER TABLE, CREATE INDEX) — block replication

### Giảm Replication Lag

```
1. Nâng cấp instance class của replica
2. Tối ưu hóa writes trên primary (ít writes = ít phải apply trên replica)
3. Dùng parallel replication (Sao Chép Song Song):
   MySQL: innodb_parallel_read_threads, slave_parallel_workers
4. Tách long-running queries sang dedicated replica
5. Chuyển sang Aurora nếu lag là vấn đề thường xuyên
```

---

## Read Replicas vs Multi-AZ

Đây là câu hỏi phỏng vấn rất phổ biến — cần thuộc lòng:

| Tiêu Chí                          | Read Replicas                    | Multi-AZ                         |
| ---------------------------------- | --------------------------------- | --------------------------------- |
| **Mục đích chính**                | Scale read performance            | High Availability (HA)            |
| **Replication type**              | Asynchronous (Bất Đồng Bộ)       | Synchronous (Đồng Bộ)             |
| **Phục vụ read traffic**          | ✅ Có                             | ❌ Không                          |
| **Data loss khi failover**        | Có thể (replication lag)         | Không (zero data loss)            |
| **Failover**                      | Thủ công (promote)               | Tự động (60-120s)                 |
| **Số lượng**                      | Tối đa 5 (MySQL)                 | 1 standby                         |
| **Cross-region**                  | ✅ Có                             | ❌ Không                          |
| **Chi phí thêm**                  | Instance cost × số replicas      | ~2× instance cost                 |
| **Endpoints**                     | Mỗi replica 1 endpoint riêng    | Dùng chung 1 endpoint với primary |

### Dùng Cả Hai Cùng Nhau

Trong production thực tế, thường dùng **cả hai**:

```
Primary (Multi-AZ) → HA, zero data loss
    ├── Standby (trong Multi-AZ) → HA failover
    ├── Read Replica #1 → Traffic đọc thông thường
    └── Read Replica #2 → Reporting & analytics

Kết quả: HA + Read Scaling cùng lúc
```

---

## Câu Hỏi Phỏng Vấn

**Q: Nếu primary gặp sự cố, Read Replica có tự động trở thành primary không?**

**Không** — với RDS Multi-AZ Instance, chỉ standby mới tự động promote. Read Replica phải được promote **thủ công**. Với Aurora thì khác — Reader Instances có thể tự động promote.

**Q: Read Replica có thể ở trong Multi-AZ setup không?**

**Có** — một Read Replica có thể có Multi-AZ của chính nó (replica cũng có standby riêng). Điều này cho cả HA lẫn scale đọc nhưng chi phí cao hơn.

**Q: Bao nhiêu Read Replica là quá nhiều?**

Cần cân nhắc:
- Mỗi replica nhận **toàn bộ replication stream** từ primary → primary phải xử lý nhiều replica connections
- MySQL: 5 replicas là giới hạn trên RDS
- Nếu cần nhiều hơn → **Aurora** (15 replicas) hoặc kiến trúc khác (ProxySQL, horizontal sharding)

**Q: Có thể scale tự động số lượng Read Replicas không?**

Với RDS thuần: không có auto-scaling replicas tự động. Phải tự tạo/xóa thủ công.
Với Aurora: có **Aurora Auto Scaling** — tự động thêm/bớt reader instances dựa trên CPU/connections.

---

## 🔗 Điều Hướng

- **Trước:** [3-multi-az.md](3-multi-az.md) — Multi-AZ Deployment
- **Tiếp theo:** [5-parameter-groups.md](5-parameter-groups.md) — Parameter Groups
- **Liên quan:** [../02-aurora/README.md](../02-aurora/README.md) — Aurora (15 replicas, auto-scaling)

---

**Cập Nhật Lần Cuối:** 2026-05-15
