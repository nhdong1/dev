# 4 — Aurora vs RDS — Trade-offs và So Sánh Chi Phí

> Lựa chọn giữa Aurora và RDS là một trong những quyết định kiến trúc phổ biến nhất trên AWS. Không có câu trả lời đúng tuyệt đối — quyết định phụ thuộc vào workload (khối lượng công việc), requirements (yêu cầu), và budget (ngân sách).

## 📚 Mục Lục

1. [So Sánh Kiến Trúc](#so-sánh-kiến-trúc)
2. [So Sánh Tính Năng Chi Tiết](#so-sánh-tính-năng-chi-tiết)
3. [Hiệu Năng — Performance](#hiệu-năng--performance)
4. [High Availability — Tính Sẵn Sàng Cao](#high-availability--tính-sẵn-sàng-cao)
5. [So Sánh Chi Phí](#so-sánh-chi-phí)
6. [Ma Trận Quyết Định](#ma-trận-quyết-định)
7. [Migration Path — Đường Di Chuyển](#migration-path--đường-di-chuyển)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## So Sánh Kiến Trúc

```
RDS MySQL (Kiến Trúc Truyền Thống):
                  ┌─────────────────────────┐
  Application →   │  RDS Primary Instance   │
                  │  (EBS gp3 Volume)        │
                  └─────────┬───────────────┘
                            │ Synchronous binlog
                            ▼
                  ┌─────────────────────────┐
                  │  Standby Instance       │ (Multi-AZ — không nhận reads)
                  │  (Separate EBS Volume)  │
                  └─────────────────────────┘
                            │ Asynchronous binlog
                            ▼
                  ┌─────────────────────────┐
                  │  Read Replica 1         │ (tối đa 5, lag giây đến phút)
                  └─────────────────────────┘

Aurora MySQL (Kiến Trúc Distributed Storage — Lưu Trữ Phân Tán):
                  ┌──────────────────────────────────────────────┐
                  │           Shared Storage Volume              │
                  │  AZ-a: [copy1][copy2]  AZ-b: [copy3][copy4] │
                  │  AZ-c: [copy5][copy6]                        │
                  └──────────┬─────────────┬──────────┬─────────┘
                             │             │          │
                    ┌────────▼──┐   ┌──────▼──┐  ┌───▼────┐
  Application →     │  Writer   │   │Reader 1 │  │  ...   │
                    └───────────┘   └─────────┘  └────────┘
                                   (tối đa 15, lag mili-giây)
```

---

## So Sánh Tính Năng Chi Tiết

### Tính Năng Engine

| Tính Năng                                                    | RDS MySQL/PostgreSQL | Aurora MySQL | Aurora PostgreSQL |
| ------------------------------------------------------------ | -------------------- | ------------ | ----------------- |
| **MySQL/PostgreSQL compatibility** (Tương Thích)            | 100% (nguyên bản)    | ~99%         | ~99%              |
| **Aurora Serverless v2** (Không Máy Chủ)                    | Không                | Có           | Có                |
| **Global Database** (Cơ Sở Dữ Liệu Toàn Cầu)               | Không                | Có           | Có                |
| **Backtrack** (Quay Lại Theo Thời Gian)                     | Không                | Có (MySQL)   | Không             |
| **Parallel Query** (Truy Vấn Song Song)                     | Không                | Có (MySQL)   | Không             |
| **Machine Learning integration** (Tích Hợp ML)              | Không                | Có           | Có                |
| **RDS Proxy** (Proxy Kết Nối)                               | Có                   | Có           | Có                |
| **Performance Insights** (Thông Tin Hiệu Năng)              | Có                   | Có           | Có                |
| **Max Read Replicas** (Tối Đa Bản Sao Đọc)                  | 5                    | 15           | 15                |
| **Cross-region Read Replica** (Bản Sao Đọc Xuyên Vùng)     | Có                   | Qua Global DB| Qua Global DB     |

### Tính Năng Storage (Lưu Trữ)

| Tính Năng                                              | RDS            | Aurora                      |
| ------------------------------------------------------ | -------------- | --------------------------- |
| **Storage type** (Loại Lưu Trữ)                       | EBS (gp2/gp3/io1/io2) | Shared distributed storage |
| **Max storage** (Lưu Trữ Tối Đa)                      | 64 TB          | 128 TB                      |
| **Storage auto-scaling** (Tự Động Tăng)               | Có (nhưng không giảm) | Hoàn toàn tự động     |
| **Storage replication** (Sao Chép Lưu Trữ)            | 1 bản (Multi-AZ: 2) | 6 bản (3 AZs)          |
| **IOPS provisioning** (Cung Cấp IOPS)                  | Có (io1/io2)   | Tự động (hoặc I/O-Optimized)|
| **I/O Optimized mode** (Chế Độ Tối Ưu I/O)            | Không          | Có (giá cao hơn)            |

---

## Hiệu Năng — Performance

### Benchmark Thực Tế (Điểm Chuẩn Hiệu Năng Thực Tế)

```
Write-heavy OLTP (ví dụ: sysbench write-only):
  RDS MySQL 8.0 (r6g.4xlarge):     ~30,000–50,000 TPS
  Aurora MySQL (r6g.4xlarge):      ~150,000–200,000 TPS
  Cải thiện: ~4–5× (gần với con số AWS công bố "5×")

Read-heavy workload (ví dụ: 80% reads, 20% writes):
  RDS MySQL vs Aurora MySQL:        ~1.5–2× cải thiện
  (Lý do: reads cũng lấy từ storage, overhead nhỏ hơn)

Simple reads (SELECT với index):
  RDS MySQL vs Aurora MySQL:        ~tương đương
  (Lý do: cả hai cache vào buffer pool — không khác biệt)
```

### Latency Comparison (So Sánh Độ Trễ)

| Hoạt Động                                    | RDS MySQL           | Aurora MySQL        |
| -------------------------------------------- | ------------------- | ------------------- |
| **Simple SELECT** (query có index)           | 1–5 ms              | 1–5 ms (tương đương)|
| **Complex JOIN** (nhiều bảng lớn)            | Phụ thuộc query     | Có thể nhanh hơn với Parallel Query |
| **Simple INSERT**                            | 1–5 ms              | 1–5 ms (tương đương)|
| **High-concurrency write** (ghi đồng thời cao)| Bottleneck sớm hơn | Chịu được nhiều hơn |
| **Failover time** (Chuyển Đổi Dự Phòng)     | 60–120 giây         | 15–30 giây          |
| **Read Replica lag** (Độ Trễ Bản Sao Đọc)   | Giây đến phút       | Mili-giây           |

### Khi Nào Hiệu Năng Aurora Không Vượt Trội

```
Aurora KHÔNG nhanh hơn đáng kể khi:
  ✗ Query chủ yếu đọc từ buffer pool (cached reads)
  ✗ Workload nhẹ — single-user dev/test
  ✗ Batch reads từ disk (sequential scan, full table scan)
  ✗ Complex analytics queries — dùng Redshift thay thế

Aurora NHANH HƠN đáng kể khi:
  ✓ High-concurrency writes (nhiều writes đồng thời)
  ✓ Mixed read-write OLTP quy mô lớn
  ✓ Cần nhiều Read Replicas (> 5) với lag thấp
  ✓ Cần failover nhanh (< 30 giây)
```

---

## High Availability — Tính Sẵn Sàng Cao

### So Sánh HA Mechanisms (Cơ Chế Tính Sẵn Sàng Cao)

| Cơ Chế HA                                           | RDS Multi-AZ                    | Aurora Multi-AZ                    |
| ---------------------------------------------------- | ------------------------------- | ---------------------------------- |
| **Failover tự động**                                 | Có (60–120 giây)                | Có (15–30 giây)                    |
| **Standby nhận reads**                               | Không                           | Có (là Reader instance)            |
| **AZs coverage** (Độ Phủ Vùng)                      | 2 AZs                           | 3 AZs (6 storage copies)           |
| **Mất AZ hoàn toàn**                                 | 1 AZ: OK; 2 AZ: lỗi            | 1 AZ: OK; 2 AZ: read-only          |
| **Storage failure** (Lỗi Lưu Trữ)                   | EBS failure → failover          | 1–2 storage nodes lỗi: tự sửa     |
| **Backup impact** (Ảnh Hưởng Backup)                 | Lấy từ Standby (có I/O impact)  | Lấy từ storage (không ảnh hưởng instances) |

---

## So Sánh Chi Phí

### Mô Hình Chi Phí

```
RDS Cost Model (Mô Hình Chi Phí RDS):
  = Instance hours
  + Storage: $0.115/GB-month (gp3) đã provision
  + I/O: $0.20/1M requests (nếu dùng gp2/gp3 với I/O credit)
  + Data transfer
  + Backup storage ngoài retention window

Aurora Cost Model (Mô Hình Chi Phí Aurora):
  = Instance hours (cao hơn RDS ~20–30%)
  + Storage: $0.10/GB-month (thực sự dùng, tự động tăng)
  + I/O: $0.20/1M requests (Standard) hoặc không tính I/O (I/O-Optimized với giá storage cao hơn)
  + Data transfer
  + Backup storage
```

### Bảng Giá Instance Tham Khảo (us-east-1, on-demand)

| Instance Type             | RDS MySQL ($/giờ) | Aurora MySQL ($/giờ) | Aurora đắt hơn |
| ------------------------- | ----------------- | --------------------- | -------------- |
| **db.t3.medium** (2vCPU, 4GB) | $0.068         | $0.082                | +20%           |
| **db.r6g.large** (2vCPU, 16GB)| $0.240         | $0.285                | +19%           |
| **db.r6g.xlarge** (4vCPU, 32GB)| $0.480        | $0.570                | +19%           |
| **db.r6g.2xlarge** (8vCPU, 64GB)| $0.960       | $1.140                | +19%           |
| **db.r6g.4xlarge** (16vCPU, 128GB)| $1.920     | $2.280                | +19%           |

### Tổng Chi Phí So Sánh Thực Tế

**Scenario: Web application — 1 Writer + 2 Readers, db.r6g.xlarge, 500 GB data, Multi-AZ**

```
RDS (1 Writer Multi-AZ + 2 Read Replicas):
  Writer Multi-AZ: $0.480 × 2 (Multi-AZ) × 730h = $700.80/tháng
  Read Replicas: $0.480 × 2 × 730h = $701.60/tháng
  Storage: 500 GB × $0.115 × 3 instances = $172.50/tháng
  I/O (ước tính 10B requests): $0.20 × 10,000 = $2,000/tháng
  Tổng RDS: ~$3,574/tháng

Aurora (1 Writer + 2 Readers, db.r6g.xlarge):
  Instances: $0.570 × 3 × 730h = $1,248.30/tháng
  Storage: 500 GB × $0.10 = $50/tháng (chỉ 1 storage volume dùng chung)
  I/O: $0.20 × 10,000 = $2,000/tháng
  Tổng Aurora: ~$3,298/tháng

→ Aurora RẺ HƠN ~8% trong trường hợp này
→ Aurora đắt hơn ~19% về instance cost, nhưng rẻ hơn storage cost đáng kể
```

### Khi Aurora I/O-Optimized Có Lợi

```
Aurora I/O-Optimized (Tối Ưu I/O):
  Storage: $0.225/GB-month (thay $0.10)
  I/O: Miễn phí

Có lợi khi:
  I/O cost > (0.225 - 0.10) × GB-month
  Tức là: I/O cost > $0.125 × storage_GB_month

Ví dụ 500 GB storage, I/O threshold:
  0.125 × 500 = $62.5/tháng I/O cost
  Nếu I/O cost > $62.5/tháng → dùng I/O-Optimized

Thực tế:
  I/O cost = $0.20/1M requests × N requests
  N > 312.5M requests/tháng → dùng I/O-Optimized
  Tức là > ~10.4M requests/ngày hoặc ~120 requests/giây liên tục
```

---

## Ma Trận Quyết Định

### Chọn Aurora Khi

```
✅ Cần > 5 Read Replicas
✅ Cần failover < 30 giây (SLA yêu cầu)
✅ Write-heavy OLTP quy mô cao (> 30K writes/giây)
✅ Cần Global Database (multi-region với lag < 1 giây)
✅ Cần Aurora Serverless v2 (traffic spiky — không đều)
✅ Cần Backtrack (quay lại thời gian — chỉ MySQL)
✅ Dự án mới không bị ràng buộc chi phí instance
✅ Cần I/O-intensive workload (storage tự scale, không trả theo IOPS)
```

### Chọn RDS Khi

```
✅ Cần Oracle hoặc SQL Server (Aurora không hỗ trợ)
✅ Budget hạn chế — workload nhỏ đến trung bình
✅ Cần tương thích 100% MySQL/PostgreSQL (feature edge cases)
✅ Dev/Test environment đơn giản
✅ Workload ổn định, không cần global replication
✅ Team đã quen với RDS — không muốn học Aurora-specific features
✅ Cần Db2 hoặc MariaDB
```

### Quyết Định Theo Use Case

| Use Case                                        | Khuyến Nghị    | Lý Do                                             |
| ----------------------------------------------- | -------------- | ------------------------------------------------- |
| E-commerce Black Friday (spiky traffic)         | Aurora Serverless v2 | Auto-scale theo traffic, không cần resize   |
| SaaS B2B với nhiều tenants                      | Aurora         | 15 Read Replicas, fast failover                   |
| Legacy enterprise app dùng Oracle               | RDS Oracle     | Aurora không hỗ trợ Oracle                        |
| Global social network                           | Aurora Global DB | Cross-region reads với lag < 1 giây             |
| Small startup MVP                               | RDS Single-AZ  | Tiết kiệm chi phí giai đoạn đầu                  |
| Analytics nặng (OLAP)                           | Redshift       | Cả Aurora lẫn RDS đều không tối ưu cho OLAP      |
| Gaming leaderboard (cực kỳ fast reads)          | ElastiCache + Aurora | Aurora cho persistence, ElastiCache cho speed |

---

## Migration Path — Đường Di Chuyển

### RDS MySQL → Aurora MySQL

```
Phương pháp 1: Snapshot restore (nhanh nhất, có downtime nhỏ)
  1. Tạo RDS snapshot
  2. Restore snapshot thành Aurora cluster
  3. Cập nhật connection string
  Downtime: ~30 phút (tùy DB size)

Phương pháp 2: Read Replica promote (không downtime)
  1. Tạo Aurora Read Replica từ RDS instance
  2. Đợi lag về 0
  3. Stop writes trên RDS (maintenance window ngắn)
  4. Promote Aurora Replica thành standalone cluster
  5. Cập nhật connection string
  Downtime: < 5 phút

Phương pháp 3: AWS DMS (cho migration phức tạp)
  - Dùng khi cần transform data hoặc migration xuyên engine version
```

### RDS PostgreSQL → Aurora PostgreSQL

```
Tương tự MySQL, nhưng:
  - Snapshot restore: Có — dùng Aurora PostgreSQL snapshot migration
  - Version matching: Aurora PostgreSQL hỗ trợ PostgreSQL 11, 12, 13, 14, 15, 16
  - Kiểm tra extension compatibility (tương thích extension) trước migration
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Aurora đắt hơn RDS nhưng tại sao vẫn được chọn?

**Trả lời:** Aurora đắt hơn ~19% về instance cost, nhưng tổng cost of ownership (chi phí sở hữu toàn phần) thường thấp hơn hoặc tương đương vì: (1) Storage cost thấp hơn — Aurora chỉ tính theo GB thực sự dùng, không provision trước; (2) Không tốn chi phí Read Replica storage riêng — tất cả chia sẻ một storage volume; (3) Ít operational overhead hơn — failover tự động 15–30 giây, storage tự scale. Business value cũng cao hơn: 15 Read Replicas thay vì 5, failover nhanh hơn, Global Database, Serverless v2.

### Q2: Khi nào bạn KHÔNG nên migrate từ RDS sang Aurora?

**Trả lời:** Không nên migrate khi: (1) Đang dùng Oracle hoặc SQL Server — Aurora không hỗ trợ; (2) Workload nhỏ, budget hạn chế — overhead instance cost 19% không đáng cho workload đơn giản; (3) Cần 100% MySQL/PostgreSQL feature compatibility — Aurora có một số edge cases không tương thích hoàn toàn (ví dụ: một số storage engine flags, specific system tables); (4) Dev/test environment — RDS Single-AZ rẻ hơn đủ dùng; (5) Risk adverse team — migration cần testing và cẩn thận.

### Q3: Aurora có phải là "MySQL/PostgreSQL cloud" hoàn toàn không?

**Trả lời:** Không hoàn toàn. Aurora MySQL tương thích ~99% với MySQL nhưng có một số điểm khác biệt: Aurora không hỗ trợ một số storage engines như MyISAM hay FEDERATED; một số performance schema (schema hiệu năng) và system tables khác; behavior của một số edge cases trong replication, locking. Trước khi migrate, cần kiểm tra application code và queries với Aurora bằng test environment. Trong thực tế, 99% ứng dụng MySQL/PostgreSQL chạy được trên Aurora mà không cần thay đổi.

### Q4: So sánh Aurora Backtrack với PITR — khi nào dùng cái nào?

**Trả lời:** Aurora Backtrack (Quay Lại Theo Thời Gian — chỉ MySQL) cho phép "rewind" cluster về một thời điểm trong vòng 72 giờ **mà không cần restore** — rất nhanh (thường trong vài giây đến vài phút). PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm) thì restore ra một cluster MỚI từ backup, mất 30 phút đến vài giờ. Dùng Backtrack khi cần undo lỗi logic nhanh (ví dụ: chạy nhầm DELETE); dùng PITR khi cần clone cluster để debug hoặc khi cần restore sau Backtrack window (> 72 giờ trước).

---

## 🔗 Điều Hướng

| Trước                                            | Tiếp Theo                                              |
| ------------------------------------------------ | ------------------------------------------------------ |
| [3-aurora-global.md](./3-aurora-global.md)       | [5-aurora-ha-failover.md](./5-aurora-ha-failover.md)  |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
