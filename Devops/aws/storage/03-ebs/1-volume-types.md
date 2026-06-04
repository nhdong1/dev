# EBS Volume Types — Các Loại Volume EBS

> Chọn đúng loại EBS volume là quyết định kiến trúc quan trọng — ảnh hưởng trực tiếp đến hiệu suất, chi phí và khả năng mở rộng. AWS cung cấp 6 loại volume với đặc tính hoàn toàn khác nhau.

---

## 📊 Bảng So Sánh Tổng Quan

| Loại | Công Nghệ | Dung Lượng | IOPS Tối Đa | Throughput Tối Đa | Giá (us-east-1) | Boot Vol |
|------|-----------|------------|-------------|-------------------|-----------------|----------|
| **gp3** | SSD | 1 GiB – 16 TiB | 16.000 | 1.000 MB/s | $0.08/GB-month | ✅ |
| **gp2** | SSD | 1 GiB – 16 TiB | 16.000 | 250 MB/s | $0.10/GB-month | ✅ |
| **io2 Block Express** | SSD NVMe | 4 GiB – 64 TiB | 256.000 | 4.000 MB/s | $0.125/GB + IOPS | ✅ |
| **io1** | SSD | 4 GiB – 16 TiB | 64.000 | 1.000 MB/s | $0.125/GB + IOPS | ✅ |
| **st1** | HDD | 125 GiB – 16 TiB | 500 | 500 MB/s | $0.045/GB-month | ❌ |
| **sc1** | HDD | 125 GiB – 16 TiB | 250 | 250 MB/s | $0.015/GB-month | ❌ |

---

## 🔵 SSD-Based Volumes (Volume Dựa Trên SSD)

SSD (Solid-State Drive — Ổ Đĩa Thể Rắn) phù hợp với workload có nhiều I/O nhỏ, ngẫu nhiên (random I/O) như database và ứng dụng transactional.

---

### gp3 — General Purpose SSD v3 (SSD Đa Dụng Thế Hệ 3)

**Đây là loại volume mặc định và được khuyến nghị cho hầu hết workload.**

#### Đặc Điểm Kỹ Thuật

```
Dung lượng:    1 GiB – 16 TiB
IOPS baseline: 3.000 IOPS (miễn phí, không phụ thuộc dung lượng)
IOPS tối đa:   16.000 IOPS (có thể provision thêm, tính phí)
Throughput:    125 MB/s baseline → 1.000 MB/s (provisioned)
Latency:       single-digit millisecond (< 1ms điển hình)
Durability:    99.8% – 99.9%
```

#### Điểm Mạnh So Với gp2

| Tính Năng | gp3 | gp2 |
|-----------|-----|-----|
| IOPS cơ bản | 3.000 (cố định) | 3 × GB (biến đổi) |
| Throughput | tối đa 1.000 MB/s | tối đa 250 MB/s |
| IOPS & Throughput | **Độc lập với dung lượng** | Phụ thuộc dung lượng |
| Giá | **$0.08/GB** (rẻ hơn 20%) | $0.10/GB |
| Burst | Không cần burst | Có burst credits |

#### Use Cases (Trường Hợp Sử Dụng)

- ✅ Boot volumes (volume khởi động) — mặc định cho mọi EC2
- ✅ Ứng dụng web và API server
- ✅ Development và test environments
- ✅ MySQL, PostgreSQL với workload vừa (< 16.000 IOPS)
- ✅ Container workloads (Docker, ECS, EKS)
- ✅ Virtual desktops

#### Ví Dụ Provision

```bash
# Tạo gp3 volume 100GB với 5.000 IOPS và 500 MB/s throughput
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --iops 5000 \
  --throughput 500 \
  --availability-zone us-east-1a \
  --encrypted

# Chi phí tháng:
# Storage: 100 GB × $0.08 = $8.00
# IOPS: (5.000 - 3.000) × $0.005 = $10.00  ← chỉ tính phần vượt baseline
# Throughput: (500 - 125) × $0.04 = $15.00  ← chỉ tính phần vượt baseline
# Tổng: $33.00/tháng
```

---

### gp2 — General Purpose SSD v2 (SSD Đa Dụng Thế Hệ 2) — Legacy

**Không khuyến nghị cho volume mới. Dùng gp3 thay thế.**

#### Cơ Chế IOPS Của gp2

gp2 dùng mô hình **burst credit** (tín dụng burst — bùng phát tạm thời):

```
IOPS baseline = 3 × GB
  → 100 GB volume: 300 IOPS baseline
  → 5.334 GB: đạt 16.000 IOPS (tối đa)

Burst Credits:
  - Tích lũy khi dùng < baseline
  - Cho phép burst lên 3.000 IOPS (volume < 1.000 GB)
  - Burst credit pool = 5.4 triệu credits
  - Hết credit → bị giới hạn về baseline IOPS
```

#### Vấn Đề Của gp2

```
Volume 100 GB:
  Baseline IOPS = 300 (quá thấp cho hầu hết ứng dụng)
  Burst = 3.000 IOPS (chỉ tạm thời, hết credit sẽ throttle)

→ Để có 3.000 IOPS ổn định cần volume >= 1.000 GB
→ Lãng phí: trả tiền cho 900 GB chỉ để có IOPS
→ gp3 không có vấn đề này: 3.000 IOPS ở bất kỳ dung lượng nào
```

---

### io2 Block Express — Provisioned IOPS SSD — Tối Cao

**Loại volume hiệu suất cao nhất của AWS, dành cho database mission-critical.**

#### Đặc Điểm Kỹ Thuật

```
Dung lượng:  4 GiB – 64 TiB
IOPS:        tối đa 256.000 IOPS
Throughput:  tối đa 4.000 MB/s
Latency:     sub-millisecond (< 0.1ms)
Durability:  99.999% (5 chín) — cao nhất AWS
Multi-Attach: ✅ Có hỗ trợ (tối đa 16 Nitro EC2)
```

#### Tỷ Lệ IOPS/GB

```
Tỷ lệ tối đa: 1.000 IOPS : 1 GB
  → 32 GB volume: có thể provision tối đa 32.000 IOPS
  → 256 GB volume: có thể provision tối đa 256.000 IOPS
```

#### Pricing (Giá)

```
Storage:   $0.125/GB-month
IOPS:      $0.065/IOPS-month (cho 32.000 IOPS đầu tiên)
           $0.046/IOPS-month (từ 32.001 – 64.000)
           $0.032/IOPS-month (từ 64.001 – 160.000)
           $0.023/IOPS-month (từ 160.001 trở lên)

Ví dụ: 500GB, 40.000 IOPS
  Storage: 500 × $0.125 = $62.50
  IOPS: (32.000 × $0.065) + (8.000 × $0.046) = $2.080 + $368 = $2.448
  Tổng: ~$2.510/tháng
```

#### Use Cases

- ✅ SAP HANA, Oracle RAC — database mission-critical
- ✅ Microsoft SQL Server AlwaysOn
- ✅ MongoDB, Cassandra workload cao
- ✅ EBS Multi-Attach (chia sẻ volume giữa nhiều EC2)
- ✅ Application yêu cầu < 1ms latency nhất quán

---

### io1 — Provisioned IOPS SSD — Thế Hệ Cũ

**io1 là phiên bản tiền nhiệm của io2. Dùng io2 cho volume mới.**

```
Dung lượng:  4 GiB – 16 TiB (ít hơn io2)
IOPS:        tối đa 64.000 IOPS (ít hơn io2 4 lần)
Throughput:  tối đa 1.000 MB/s (ít hơn io2 4 lần)
Durability:  99.8% – 99.9% (thấp hơn io2)
Multi-Attach: ✅ Có hỗ trợ
Giá:         Tương đương io2
```

> **Khuyến nghị:** Chuyển sang io2 Block Express — hiệu suất cao hơn, durability tốt hơn, cùng giá.

---

## 🟠 HDD-Based Volumes (Volume Dựa Trên Ổ Cứng Cơ)

HDD (Hard Disk Drive — Ổ Đĩa Cơ) phù hợp với workload có I/O tuần tự (sequential I/O) lớn, ít transaction nhỏ. **Không thể dùng làm boot volume.**

---

### st1 — Throughput Optimized HDD (HDD Tối Ưu Thông Lượng)

#### Đặc Điểm Kỹ Thuật

```
Dung lượng:       125 GiB – 16 TiB
Throughput:       40 MB/s per TiB baseline
                  250 MB/s per TiB burst (tối đa 500 MB/s)
IOPS:             tối đa 500 IOPS (không phải thế mạnh)
Giá:              $0.045/GB-month
Boot volume:      ❌ Không hỗ trợ
```

#### Cơ Chế Burst của st1

```
Baseline throughput = 40 MB/s × dung lượng (TiB)
  → 2 TiB: 80 MB/s baseline, 500 MB/s burst
  → 8 TiB: 320 MB/s baseline, 500 MB/s burst

Burst bucket size = 1 TiB × dung lượng (TiB)
  → Tích lũy khi dùng dưới baseline
  → Xả khi cần throughput cao
```

#### Use Cases

- ✅ Big data analytics — Apache Spark, Hadoop — xử lý dữ liệu lớn
- ✅ Data warehouse — Amazon Redshift, data warehousing
- ✅ Log processing — đọc log theo thứ tự tuần tự
- ✅ ETL (Extract, Transform, Load — Trích Xuất, Biến Đổi, Tải) pipelines
- ✅ Kafka log storage — lưu trữ message queue
- ❌ Database — random I/O → không phù hợp

---

### sc1 — Cold HDD (HDD Lạnh)

**Loại volume rẻ nhất, dành cho dữ liệu hiếm khi truy cập.**

#### Đặc Điểm Kỹ Thuật

```
Dung lượng:       125 GiB – 16 TiB
Throughput:       12 MB/s per TiB baseline
                  80 MB/s per TiB burst (tối đa 250 MB/s)
IOPS:             tối đa 250 IOPS
Giá:              $0.015/GB-month (rẻ nhất trong EBS)
Boot volume:      ❌ Không hỗ trợ
```

#### Use Cases

- ✅ Cold data — dữ liệu lưu trữ lâu dài ít truy cập
- ✅ Backup secondary storage — lưu trữ backup tiết kiệm
- ✅ Compliance archive — lưu trữ theo yêu cầu pháp lý
- ✅ Dữ liệu scan định kỳ (monthly/quarterly)

---

## 🔍 So Sánh Chi Tiết Theo Kịch Bản

### Kịch Bản 1: MySQL Production Database

```
Yêu cầu: 500GB dữ liệu, 8.000 IOPS, latency < 5ms

→ Chọn: gp3
  - Dung lượng: 500 GB
  - Provision IOPS: 8.000 (vượt baseline 3.000)
  - Chi phí:
    Storage: 500 × $0.08 = $40
    IOPS: (8.000 - 3.000) × $0.005 = $25
    Tổng: $65/tháng

→ KHÔNG chọn io2: $0.125 × 500 + IOPS cost = $400+/tháng — đắt gấp 6 lần
```

### Kịch Bản 2: Oracle Database Mission-Critical

```
Yêu cầu: 2TB dữ liệu, 50.000 IOPS, SLA 99.999% durability, Multi-Attach

→ Chọn: io2 Block Express
  - Lý do: gp3 chỉ có 16.000 IOPS tối đa
  - io2 đáp ứng: 50.000 IOPS, 99.999% durability, Multi-Attach
  - gp3 KHÔNG có Multi-Attach
```

### Kịch Bản 3: Hadoop Data Lake

```
Yêu cầu: 50TB dữ liệu, đọc/ghi tuần tự, cost-sensitive

→ Chọn: st1
  - 50TB × $0.045 = $2.250/tháng
  - vs gp3: 50TB × $0.08 = $4.000/tháng
  - Tiết kiệm: $1.750/tháng (44%)
```

### Kịch Bản 4: Compliance Archive

```
Yêu cầu: 100TB dữ liệu cũ, truy cập hàng quý, cost tối thiểu

→ Chọn: sc1
  - 100TB × $0.015 = $1.500/tháng
  - vs st1: 100TB × $0.045 = $4.500/tháng
  - Tiết kiệm: $3.000/tháng (67%)
```

---

## 🔄 Chuyển Đổi Giữa Volume Types

### Online Modification (Thay Đổi Khi Đang Chạy)

EBS cho phép thay đổi loại volume, IOPS, Throughput mà **không cần dừng EC2** (không có downtime):

```bash
# Chuyển từ gp2 sang gp3 (không có downtime)
aws ec2 modify-volume \
  --volume-id vol-1234567890abcdef0 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125

# Kiểm tra trạng thái chuyển đổi
aws ec2 describe-volumes-modifications \
  --volume-id vol-1234567890abcdef0

# Trạng thái: modifying → optimizing → completed
# Thời gian: vài phút đến vài giờ tùy dung lượng
```

### Giới Hạn Thay Đổi

```
- Sau khi modify, phải chờ 6 giờ trước khi modify lại
- Không thể giảm dung lượng (chỉ tăng được)
- Có thể tăng IOPS và Throughput khi đang chạy
- Sau khi tăng dung lượng, cần mở rộng filesystem trên OS
```

### Mở Rộng Filesystem Sau Khi Tăng Volume

```bash
# Sau khi tăng EBS volume, kiểm tra partition
lsblk

# Mở rộng partition (nếu cần)
sudo growpart /dev/xvda 1

# Mở rộng filesystem ext4
sudo resize2fs /dev/xvda1

# Hoặc xfs
sudo xfs_growfs /
```

---

## 💰 Tối Ưu Chi Phí

### Chiến Lược 1: Migrating gp2 → gp3

```bash
# Tìm tất cả volume gp2
aws ec2 describe-volumes \
  --filters "Name=volume-type,Values=gp2" \
  --query "Volumes[*].[VolumeId,Size,Iops]" \
  --output table

# Migrate từng volume sang gp3
# Tiết kiệm 20% chi phí storage ngay lập tức
```

### Chiến Lược 2: Right-sizing IOPS

```
Kiểm tra CloudWatch metric: VolumeConsumedReadWriteOps
  → Nếu sử dụng thực tế < 50% IOPS đã provision
  → Giảm provisioned IOPS để tiết kiệm chi phí
```

### Chiến Lược 3: Dùng HDD Cho Cold Data

```
Quy tắc:
  - Truy cập > 1 lần/ngày → SSD (gp3)
  - Truy cập < 1 lần/ngày, tuần tự → st1
  - Truy cập < 1 lần/tuần, archive → sc1 hoặc S3 Glacier
```

---

## 📝 Decision Tree — Cây Quyết Định Chọn Volume

```
Cần làm boot volume?
├── Có → Phải dùng SSD (gp3/gp2/io1/io2)
└── Không ↓

Cần > 16.000 IOPS hoặc Multi-Attach?
├── Có → io2 Block Express
└── Không ↓

Workload có nhiều random I/O nhỏ (database, app)?
├── Có → gp3 (đủ dùng 95% trường hợp)
└── Không ↓

Workload sequential, throughput cao (analytics, logs)?
├── Có → st1
└── Không ↓

Dữ liệu rất ít truy cập, cần rẻ nhất?
└── sc1
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Tại sao gp3 tốt hơn gp2?**
> Ba lý do: (1) IOPS không phụ thuộc dung lượng — 3.000 IOPS ngay cả với 1GB volume; (2) Throughput tối đa 1.000 MB/s thay vì 250 MB/s; (3) Rẻ hơn 20%. Không có lý do nào để chọn gp2 cho volume mới.

**Q: Khi nào cần io2 thay vì gp3?**
> Ba kịch bản: (1) Cần hơn 16.000 IOPS; (2) Cần Multi-Attach (gắn volume vào nhiều EC2 cùng lúc); (3) Cần durability 99.999% (5 chín) cho database mission-critical.

**Q: Tại sao không dùng HDD làm boot volume?**
> HDD (st1, sc1) tối ưu cho sequential I/O với throughput cao. Boot process cần random I/O tốc độ cao để load OS và ứng dụng — đây là điểm mạnh của SSD, không phải HDD.

**Q: IOPS và Throughput khác nhau thế nào?**
> IOPS đo số lượng thao tác I/O/giây (phù hợp database nhiều transaction nhỏ). Throughput đo lượng data MB/giây (phù hợp streaming, analytics, data warehouse với I/O lớn tuần tự).

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-snapshots-and-lifecycle.md](./2-snapshots-and-lifecycle.md) — Snapshots & DLM
- **Liên quan:** [4-performance-tuning.md](./4-performance-tuning.md) — Tối ưu IOPS/Throughput

---

**Cập Nhật Lần Cuối:** 2026-05-15
