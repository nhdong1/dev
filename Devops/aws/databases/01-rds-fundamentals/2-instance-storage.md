# 2 — Instance Classes & Storage — Lớp Máy Chủ & Lưu Trữ

> Lựa chọn đúng instance class và storage type là quyết định quan trọng ảnh hưởng đến cả hiệu năng lẫn chi phí. Undersizing gây bottleneck (Nút Thắt Cổ Chai), oversizing lãng phí tiền.

## 📚 Mục Lục

1. [Instance Classes — Lớp Máy Chủ](#instance-classes--lớp-máy-chủ)
2. [Storage Types — Loại Lưu Trữ](#storage-types--loại-lưu-trữ)
3. [IOPS — Tốc Độ Đọc/Ghi](#iops--tốc-độ-đọcghi)
4. [Storage Auto Scaling — Tự Động Mở Rộng Lưu Trữ](#storage-auto-scaling--tự-động-mở-rộng-lưu-trữ)
5. [Vertical Scaling — Mở Rộng Theo Chiều Dọc](#vertical-scaling--mở-rộng-theo-chiều-dọc)
6. [Chọn Instance và Storage Phù Hợp](#chọn-instance-và-storage-phù-hợp)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Instance Classes — Lớp Máy Chủ

RDS instance classes được chia thành các họ (family) phục vụ các nhu cầu khác nhau.

### Các Họ Instance

#### Standard — Cân Bằng (db.m* family)

Cân bằng giữa CPU, RAM, network — lựa chọn mặc định cho hầu hết workloads.

| Instance         | vCPU | RAM    | Network    | Ghi Chú                        |
| ---------------- | ---- | ------ | ---------- | ------------------------------ |
| `db.m7g.large`   | 2    | 8 GB   | 12.5 Gbps  | Graviton3, cost-effective nhất |
| `db.m7g.xlarge`  | 4    | 16 GB  | 12.5 Gbps  |                                |
| `db.m7g.2xlarge` | 8    | 32 GB  | 15 Gbps    |                                |
| `db.m7g.4xlarge` | 16   | 64 GB  | 15 Gbps    |                                |
| `db.m7g.8xlarge` | 32   | 128 GB | 25 Gbps    |                                |
| `db.m6g.*`       | ...  | ...    | ...        | Graviton2, thế hệ cũ hơn       |
| `db.m6i.*`       | ...  | ...    | ...        | Intel Ice Lake, phổ biến       |

#### Memory Optimized — Tối Ưu Bộ Nhớ (db.r* family)

Tỉ lệ RAM cao hơn CPU — dành cho databases cần buffer pool lớn.

| Instance         | vCPU | RAM    | Ghi Chú                                    |
| ---------------- | ---- | ------ | ------------------------------------------ |
| `db.r7g.large`   | 2    | 16 GB  | 2:1 RAM/vCPU ratio                         |
| `db.r7g.xlarge`  | 4    | 32 GB  |                                            |
| `db.r7g.2xlarge` | 8    | 64 GB  |                                            |
| `db.r7g.4xlarge` | 16   | 128 GB |                                            |
| `db.r7g.8xlarge` | 32   | 256 GB | Phù hợp cho OLAP in-memory workloads       |
| `db.r6i.*`       | ...  | ...    | Intel, phổ biến cho Oracle và SQL Server   |
| `db.x2g.*`       | ...  | ...    | Extreme memory — tối đa 1,952 GB RAM       |

#### Burstable — Burst Hiệu Năng (db.t* family)

Chạy ở hiệu năng thấp thông thường, burst khi cần — dành cho dev/test và workloads nhỏ.

| Instance        | vCPU | RAM   | Baseline CPU | Burst CPU |
| --------------- | ---- | ----- | ------------ | --------- |
| `db.t3.micro`   | 2    | 1 GB  | 10%          | 100%      |
| `db.t3.small`   | 2    | 2 GB  | 20%          | 100%      |
| `db.t3.medium`  | 2    | 4 GB  | 20%          | 100%      |
| `db.t4g.micro`  | 2    | 1 GB  | 10%          | 100%      |
| `db.t4g.medium` | 2    | 4 GB  | 20%          | 100%      |

> ⚠️ **Cảnh báo:** Không dùng t3/t4g cho production databases quan trọng — CPU credits (Tín Chỉ CPU) cạn kiệt gây throttle đột ngột.

### Graviton vs Intel vs AMD

| Kiến Trúc        | Ưu Điểm                                    | Nhược Điểm                               |
| ---------------- | ------------------------------------------ | ---------------------------------------- |
| **Graviton3** (ARM — db.m7g, db.r7g) | Rẻ hơn ~20%, tiêu thụ điện thấp hơn | Không phù hợp Oracle, SQL Server |
| **Intel** (x86 — db.m6i, db.r6i) | Tương thích mọi engine | Đắt hơn Graviton |
| **AMD** (x86 — db.m6a) | Rẻ hơn Intel ~10% | Hiệu năng tương đương Intel |

**Khuyến nghị:** Dùng Graviton cho MySQL/PostgreSQL/MariaDB mới. Dùng Intel/AMD cho Oracle/SQL Server.

---

## Storage Types — Loại Lưu Trữ

### gp3 — General Purpose SSD (SSD Đa Dụng)

**Loại storage được khuyến nghị cho hầu hết trường hợp.**

| Thông Số                   | Giá Trị                                      |
| -------------------------- | -------------------------------------------- |
| IOPS cơ bản (Baseline)    | 3,000 IOPS (miễn phí, không phụ thuộc dung lượng) |
| IOPS tối đa               | 16,000 IOPS                                  |
| Throughput cơ bản         | 125 MB/s                                     |
| Throughput tối đa         | 1,000 MB/s                                   |
| Dung lượng                | 20 GB – 64 TB                                |
| Chi phí IOPS thêm         | $0.02/IOPS/tháng (trên 3,000)               |

**Ưu điểm gp3:**
- IOPS và throughput **độc lập** với dung lượng (khác gp2)
- Rẻ hơn gp2 khoảng 20% với cùng IOPS
- Phù hợp hầu hết OLTP workloads

```
Ví dụ: gp3 với 6,000 IOPS
- 3,000 IOPS cơ bản: miễn phí
- 3,000 IOPS thêm: 3,000 × $0.02 = $60/tháng
```

### io1 / io2 — Provisioned IOPS SSD (SSD IOPS Được Cấp Phát)

**Dành cho I/O intensive workloads — khi gp3 không đủ.**

| Thông Số                   | io1                        | io2 (Block Express)        |
| -------------------------- | -------------------------- | -------------------------- |
| IOPS tối đa               | 64,000 IOPS                | 256,000 IOPS               |
| Tỉ lệ IOPS/GB tối đa     | 50:1                       | 1,000:1                    |
| Throughput tối đa         | 1,000 MB/s                 | 4,000 MB/s                 |
| Dung lượng                | 100 GB – 64 TB             | 4 GB – 64 TB               |
| Durability (Độ Bền)       | 99.8-99.9%                 | 99.999%                    |

**Khi nào dùng io1/io2:**
- Cần >16,000 IOPS (giới hạn của gp3)
- Cần IOPS/storage ratio (Tỉ Lệ IOPS/Dung Lượng) linh hoạt
- Mission-critical OLTP với SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) nghiêm ngặt

### Magnetic (Từ Tính) — Legacy

- HDD (Hard Disk Drive — Ổ Đĩa Cứng Từ Tính) cũ
- ~100 IOPS, throughput thấp
- **Không khuyến nghị** — chỉ còn tồn tại để backward compatibility

### So Sánh Tổng Hợp

```
                gp3          io1/io2       Magnetic
               ──────       ─────────     ─────────
IOPS max:     16,000        256,000         ~100
Throughput:   1 GB/s         4 GB/s         ~50 MB/s
Chi phí:      Thấp          Cao            Rất thấp
Use case:     Hầu hết       I/O heavy      Không dùng
```

---

## IOPS — Tốc Độ Đọc/Ghi

IOPS (Input/Output Operations Per Second — Số Thao Tác Đọc/Ghi Mỗi Giây) là metric quan trọng nhất cho database storage.

### Tính IOPS Cần Thiết

```
Quy tắc tính nhẩm:
- OLTP nhỏ (< 100 concurrent users):    1,000 - 3,000 IOPS → gp3 đủ
- OLTP trung bình (100-500 users):      3,000 - 8,000 IOPS → gp3 với IOPS thêm
- OLTP lớn (500+ users):               8,000 - 30,000 IOPS → io1/io2
- Reporting/OLAP mixed:                 Tùy vào query patterns
```

### Công Thức Tính IOPS Cần Thiết

```
Cách 1 — Từ current workload:
  Đo IOPS hiện tại với CloudWatch → target IOPS = peak × 1.3 (buffer 30%)

Cách 2 — Từ concurrency:
  IOPS ≈ concurrent_users × transactions_per_second × io_per_transaction

Ví dụ:
  500 users × 10 TPS × 5 IO/transaction = 25,000 IOPS → cần io1
```

### Latency vs IOPS

| Storage Type  | Latency (Độ Trễ) P99 | Ghi Chú                          |
| ------------- | ---------------------- | -------------------------------- |
| gp3           | 1-2 ms                 | Đủ cho hầu hết ứng dụng         |
| io2           | < 1 ms (sub-ms)        | Cho latency-sensitive (Nhạy Cảm Độ Trễ) |
| io2 Block Express | < 0.5 ms          | Extreme performance              |

---

## Storage Auto Scaling — Tự Động Mở Rộng Lưu Trữ

RDS hỗ trợ tự động tăng storage khi dung lượng sắp đầy — không cần downtime.

### Cấu Hình Auto Scaling

```
Cài đặt khi tạo hoặc modify instance:
- Enable Storage Auto Scaling: ✅
- Maximum Storage Threshold: 1,000 GB  (giới hạn tối đa)

Điều kiện trigger tự động mở rộng:
- Free storage < 10% của allocated storage
- Hoặc free storage < 5 GB (whichever is greater)
- Và trạng thái này kéo dài ≥ 5 phút
```

### Quy Tắc Scaling

- Tăng tối thiểu 10% hoặc 5 GB (lấy giá trị lớn hơn)
- Không tự động shrink (Thu Nhỏ) — storage chỉ tăng
- Có thể trigger nhiều lần nếu tăng không đủ

### Lưu Ý Quan Trọng

> Không thể giảm storage sau khi đã tăng. Nếu phân bổ quá nhiều, phải snapshot + restore vào instance nhỏ hơn.

---

## Vertical Scaling — Mở Rộng Theo Chiều Dọc

RDS hỗ trợ thay đổi instance class (lên hoặc xuống) với minimal downtime.

### Quy Trình Scale Up/Down

```
1. Modify DB instance → chọn instance class mới
2. Chọn thời điểm áp dụng:
   a. Apply immediately (Áp Dụng Ngay) — ngay khi lưu (có downtime ngắn)
   b. Apply during next maintenance window (Áp Dụng Vào Cửa Sổ Bảo Trì Tiếp Theo)
3. RDS thực hiện:
   - Tạo instance mới với class mới
   - Replicate data
   - Failover (Chuyển Đổi Dự Phòng) — downtime 1-2 phút
```

### Downtime Khi Scale

| Deployment Mode             | Downtime Khi Scale Instance |
| --------------------------- | ----------------------------|
| Single-AZ                   | 5-10 phút (instance restart)|
| Multi-AZ                    | 60-120 giây (failover)      |

> **Thực hành tốt:** Luôn dùng Multi-AZ để scale với minimal downtime.

---

## Chọn Instance và Storage Phù Hợp

### Quy Trình Ra Quyết Định

```
Bước 1 — Tính RAM cần thiết:
  innodb_buffer_pool_size (MySQL) hoặc shared_buffers (PostgreSQL)
  = 75% của database working set (tập dữ liệu hoạt động)
  
  Ví dụ: Database 200 GB, working set 50 GB
  → RAM cần: 50 GB × 1.25 = 62.5 GB
  → Chọn: db.r7g.2xlarge (64 GB RAM)

Bước 2 — Xác định IOPS cần:
  Đo peak IOPS hoặc tính từ transaction volume
  ≤ 3,000 IOPS  → gp3 mặc định
  ≤ 16,000 IOPS → gp3 với custom IOPS
  > 16,000 IOPS → io1/io2

Bước 3 — Dung lượng storage:
  Dung lượng database hiện tại × 2 (buffer tăng trưởng 1 năm)
  Tối thiểu 100 GB để tránh phải resize sớm

Bước 4 — Bật Storage Auto Scaling:
  Maximum = dung lượng dự kiến sau 3 năm
```

### Ví Dụ Thực Tế

**Scenario 1: E-commerce nhỏ (10,000 đơn hàng/ngày)**
```
Database size: 50 GB, peak ~500 TPS
Yêu cầu: IOPS ~2,000, RAM 8 GB đủ

Chọn:
- Instance: db.m7g.large (2 vCPU, 8 GB RAM)
- Storage: gp3, 100 GB, 3,000 IOPS (mặc định)
- Chi phí ước tính: ~$150/tháng
```

**Scenario 2: Fintech OLTP (1 triệu giao dịch/ngày)**
```
Database size: 500 GB, peak ~5,000 TPS
Yêu cầu: IOPS ~15,000, RAM 64 GB

Chọn:
- Instance: db.r7g.2xlarge (8 vCPU, 64 GB RAM)
- Storage: gp3, 1 TB, 15,000 IOPS
- Chi phí ước tính: ~$1,200/tháng
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khác biệt giữa gp2 và gp3 là gì? Tại sao nên dùng gp3?**

- **gp2**: IOPS gắn với dung lượng (3 IOPS/GB) — muốn thêm IOPS phải tăng storage
- **gp3**: IOPS và throughput độc lập — có thể tăng IOPS mà không tăng storage, và rẻ hơn ~20%
- **Khuyến nghị:** Luôn dùng gp3 cho dự án mới

**Q: Khi nào nên chọn db.r* thay db.m*?**

Khi database có `working set` lớn cần cache trong RAM:
- MySQL: `innodb_buffer_pool_size` cần RAM lớn
- PostgreSQL: `shared_buffers` và OS cache cần RAM
- Quy tắc: nếu `RAM cần > 50% RAM của db.m*`, chuyển sang db.r*

**Q: Storage Auto Scaling có ảnh hưởng đến hiệu năng khi mở rộng không?**

Việc mở rộng storage xảy ra online (không downtime). Tuy nhiên, trong quá trình mở rộng có thể có I/O throttle nhẹ. Nên đặt cảnh báo khi free storage < 20% để có thời gian phản ứng thay vì phụ thuộc hoàn toàn vào auto scaling.

**Q: Làm thế nào để giảm storage size của RDS instance?**

Không thể trực tiếp — storage RDS chỉ tăng không giảm. Cách giải quyết:
1. Tạo snapshot từ instance hiện tại
2. Restore snapshot vào instance mới với storage size nhỏ hơn
3. Chuyển traffic sang instance mới
4. Xóa instance cũ

---

## 🔗 Điều Hướng

- **Trước:** [1-engine-types.md](1-engine-types.md) — Engine Types
- **Tiếp theo:** [3-multi-az.md](3-multi-az.md) — Multi-AZ Deployment
- **Liên quan:** [../07-performance-tuning/README.md](../07-performance-tuning/README.md) — Performance Tuning

---

**Cập Nhật Lần Cuối:** 2026-05-15
