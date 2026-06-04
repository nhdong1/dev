# EBS Performance Tuning — Tối Ưu Hiệu Suất IOPS, Throughput, Latency

> Hiểu và tối ưu hiệu suất EBS là kỹ năng vận hành quan trọng. Hiệu suất EBS bị ảnh hưởng bởi nhiều yếu tố: loại volume, EC2 instance type, cấu hình OS, và workload pattern (kiểu tải công việc). Bài này phân tích từng yếu tố và cách tối ưu.

---

## 📐 Ba Chiều Kích Của Hiệu Suất

```
                    Hiệu Suất EBS
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       IOPS          Throughput      Latency
(Thao tác I/O/giây) (Dữ liệu MB/s)  (Độ trễ ms)
          │              │              │
  Đo số lượng    Đo lượng data    Đo thời gian
  thao tác nhỏ  truyền/giây      mỗi thao tác
          │              │              │
   Database         Analytics        Cache,
   OLTP             Streaming        Real-time
```

### Mối Quan Hệ IOPS ↔ Throughput

```
Throughput = IOPS × Block Size (Kích Thước Khối)

EBS tính IOPS với block size 16KB (mặc định):
  - 3.000 IOPS × 16KB = 48 MB/s throughput
  - 16.000 IOPS × 16KB = 256 MB/s throughput

Thực tế: ứng dụng có thể dùng block size khác
  - Database: 4KB – 16KB → IOPS-bound (bị giới hạn IOPS)
  - Analytics: 256KB – 1MB → Throughput-bound (bị giới hạn throughput)
```

---

## 🚧 Bottleneck Analysis — Phân Tích Điểm Nghẽn

### Các Tầng Có Thể Gây Bottleneck

```
Luồng I/O:
  Application
      │
      ▼
  OS Buffer Cache (RAM)
      │
      ▼  ← Bottleneck 1: OS I/O scheduler
  Block Device Driver
      │
      ▼  ← Bottleneck 2: EC2 instance bandwidth
  EBS Network
      │
      ▼  ← Bottleneck 3: Volume IOPS/Throughput limit
  EBS Volume
      │
      ▼
  Physical Storage (SSD/HDD)
```

### Xác Định Bottleneck Với CloudWatch

```bash
# Kiểm tra IOPS đang dùng vs giới hạn
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time 2026-05-15T00:00:00Z \
  --end-time 2026-05-15T01:00:00Z \
  --period 60 \
  --statistics Average

# Metrics quan trọng cần theo dõi:
# VolumeReadOps / VolumeWriteOps  → IOPS đang sử dụng
# VolumeReadBytes / VolumeWriteBytes → Throughput đang dùng
# VolumeTotalReadTime / VolumeTotalWriteTime → Latency
# VolumeQueueLength → Số I/O requests đang chờ
# BurstBalance (chỉ gp2, st1, sc1) → Còn bao nhiêu burst credit
```

---

## 📊 CloudWatch Metrics — Chỉ Số Theo Dõi

### VolumeQueueLength — Độ Dài Hàng Chờ

```
VolumeQueueLength = số I/O requests đang chờ xử lý

Lý tưởng: < 1 (gần 0)
Cảnh báo:  > 1 (volume đang bị overloaded)
Nguy hiểm: > 10 (nghiêm trọng, latency sẽ tăng vọt)

Nếu VolumeQueueLength cao:
→ IOPS limit bị hit → cần tăng provisioned IOPS
→ Hoặc volume type không phù hợp
→ Hoặc EC2 instance bandwidth bị saturate
```

### BurstBalance — Số Dư Burst Credit (Chỉ Có Ở gp2, st1, sc1)

```
BurstBalance = 100% → Đầy burst credits
BurstBalance đang giảm → Đang burst (dùng nhiều hơn baseline)
BurstBalance = 0% → Hết credit, bị throttle về baseline IOPS

Alert rule:
  Cảnh báo khi BurstBalance < 20%
  Critical khi BurstBalance < 5%
```

### VolumeIdleTime — Thời Gian Nhàn Rỗi

```
VolumeIdleTime cao → Volume không được sử dụng nhiều
  → Có thể dowgrade về loại rẻ hơn (sc1)
  → Hoặc xóa nếu không dùng (snapshot trước khi xóa)
```

---

## ⚡ EC2 Instance Bandwidth — Băng Thông EC2

EBS không chỉ bị giới hạn bởi volume, mà còn bởi **EC2 instance EBS bandwidth**.

### Instance Types Và EBS Bandwidth

```
Instance Type  | EBS Bandwidth | Max EBS IOPS
─────────────────────────────────────────────
t3.micro       | 2.085 Mbps   | 11.800
t3.small       | 2.085 Mbps   | 11.800
m5.large       | 4.750 Mbps   | 18.750
m5.xlarge      | 4.750 Mbps   | 18.750
m5.4xlarge     | 4.750 Mbps   | 18.750
m5.8xlarge     | 6.800 Mbps   | 30.000
r5.2xlarge     | 3.500 Mbps   | 18.750
c5.18xlarge    | 19.000 Mbps  | 80.000
i3en.24xlarge  | 19.000 Mbps  | 1.750.000 (NVMe)
```

### Vấn Đề: Instance Bandwidth < Volume IOPS

```
Ví dụ xấu:
  Volume: io2, 50.000 IOPS
  Instance: m5.large, max 18.750 IOPS
  
  → Volume không bao giờ đạt 50.000 IOPS vì instance limit là 18.750
  → Lãng phí tiền provision IOPS cao hơn instance limit

Giải pháp:
  → Chọn instance có EBS bandwidth phù hợp
  → Hoặc giảm provisioned IOPS xuống gần instance limit
```

---

## 🔧 OS-Level Tuning — Tối Ưu Cấp Hệ Điều Hành

### I/O Scheduler (Bộ Lập Lịch I/O)

```bash
# Kiểm tra I/O scheduler hiện tại
cat /sys/block/nvme0n1/queue/scheduler
# [mq-deadline] kyber bfq none

# Với EBS NVMe SSD, dùng "none" (no-op) cho hiệu suất tốt nhất
# Lý do: EBS đã có internal queueing; OS scheduler thêm overhead không cần thiết
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler

# Persistent qua reboot
cat /etc/udev/rules.d/99-ebs.rules
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
```

### Read-Ahead (Đọc Trước)

```bash
# Kiểm tra read-ahead setting
blockdev --getra /dev/nvme0n1
# Output: 256 (sectors = 128KB)

# Tăng read-ahead cho workload sequential (analytics, streaming)
sudo blockdev --setra 4096 /dev/nvme0n1  # 2MB

# Giảm read-ahead cho workload random (database OLTP)
sudo blockdev --setra 128 /dev/nvme0n1   # 64KB
```

### Queue Depth (Độ Sâu Hàng Chờ)

```bash
# Kiểm tra queue depth
cat /sys/block/nvme0n1/queue/nr_requests
# 64 (mặc định)

# Tăng queue depth để parallel I/O nhiều hơn (cho high IOPS workload)
echo 256 | sudo tee /sys/block/nvme0n1/queue/nr_requests
```

### Filesystem Mount Options (Tùy Chọn Mount Hệ Thống Tệp)

```bash
# /etc/fstab cho database (ưu tiên latency, tắt atime)
/dev/nvme1n1 /data ext4 defaults,noatime,nodiratime,data=writeback 0 2

# Giải thích các option:
# noatime      → Không cập nhật access timestamp khi đọc (giảm write overhead)
# nodiratime   → Không cập nhật directory access time
# data=writeback → ext4 writeback mode: ghi metadata sau data (faster, less safe)
# barrier=0    → Tắt write barrier nếu có UPS (tăng write performance)

# Cho workload an toàn (financial):
/dev/nvme1n1 /data ext4 defaults,noatime,data=ordered 0 2
# data=ordered → metadata chỉ ghi sau data → an toàn hơn
```

---

## 📈 Tối Ưu Theo Workload

### OLTP Database (Online Transaction Processing — Xử Lý Giao Dịch Trực Tuyến)

```
Đặc điểm: Nhiều thao tác nhỏ, ngẫu nhiên, latency-sensitive
→ Workload: MySQL, PostgreSQL transactional

Tối ưu:
  Volume:     io2 Block Express hoặc gp3 với cao IOPS
  IOPS:       Provision đủ IOPS (không để BurstBalance về 0)
  Block size: Nhỏ (4–16KB) — phù hợp transaction size
  Read-ahead: Thấp (128 sectors = 64KB)
  Filesystem: XFS hoặc ext4, noatime
  I/O scheduler: none (NVMe) hoặc deadline (SATA)
  
  Benchmark kiểm tra:
  fio --filename=/dev/nvme1n1 --direct=1 --rw=randrw \
    --bs=4k --iodepth=64 --numjobs=4 --runtime=60 \
    --group_reporting --name=oltp-test
```

### Analytics / Data Warehouse

```
Đặc điểm: I/O tuần tự, file lớn, throughput quan trọng
→ Workload: Spark, Presto, Redshift local

Tối ưu:
  Volume:     st1 (throughput optimized HDD) hoặc gp3 (nếu cần SSD)
  Block size: Lớn (256KB–1MB) — scan toàn bộ file
  Read-ahead: Cao (2MB–4MB)
  Filesystem: XFS (tốt hơn ext4 cho large files)
  Striping:   RAID 0 nếu cần throughput > 1 volume

  Benchmark kiểm tra:
  fio --filename=/dev/nvme1n1 --direct=1 --rw=read \
    --bs=1M --iodepth=8 --numjobs=1 --runtime=60 \
    --name=analytics-test
```

### Write-Heavy Logging (Ghi Log Nhiều)

```
Đặc điểm: Tuần tự append-only, throughput quan trọng
→ Workload: Kafka log storage, audit logs, CDC streams

Tối ưu:
  Volume:     st1 hoặc gp3
  Write mode: Sequential append (không random)
  Filesystem: XFS với bigalloc cho file lớn
  Sync:       Batch fsync thay vì per-record sync
```

---

## 🔀 RAID — Redundant Array of Independent Disks

Có thể dùng nhiều EBS volumes cùng lúc với RAID để tăng IOPS hoặc throughput:

### RAID 0 — Striping (Phân Dải) — Tăng Hiệu Suất

```
          ┌──────────────┐   ┌──────────────┐
          │  EBS Vol 1   │   │  EBS Vol 2   │
          │  gp3, 16K IOPS│  │  gp3, 16K IOPS│
          └──────┬───────┘   └──────┬───────┘
                 └────────┬─────────┘
                          ▼
                    RAID 0 Array
                    32.000 IOPS total
                    2.000 MB/s throughput

Khi nào dùng:
  → Cần IOPS > 16.000 nhưng không muốn trả giá io2
  → Cần Throughput > 1.000 MB/s
  
Nhược điểm:
  → Không có redundancy (dự phòng): 1 volume fail = mất dữ liệu
  → Phải kết hợp với EBS Snapshots để bảo vệ dữ liệu
```

```bash
# Tạo RAID 0 với 2 gp3 volumes (Linux mdadm)
sudo mdadm --create /dev/md0 \
  --level=0 \       # RAID 0
  --raid-devices=2 \
  /dev/nvme1n1 /dev/nvme2n1

# Format và mount
sudo mkfs.xfs /dev/md0
sudo mkdir -p /data
sudo mount /dev/md0 /data

# Persistent config
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

### RAID 1 — Mirroring (Phản Chiếu) — Không Cần Cho EBS

```
RAID 1 nhân đôi dữ liệu giữa 2 volumes:
  → Không cần thiết với EBS vì EBS đã tự nhân bản trong AZ
  → RAID 1 chỉ hữu ích cho on-premises hardware RAID
  → Với EBS: dùng Snapshots + Multi-AZ deployment thay vì RAID 1
```

---

## 📊 EBS Optimized Instances

**EBS-Optimized** (Tối Ưu EBS) là tính năng cấp EC2 tạo network path riêng cho EBS traffic, tách biệt với network thông thường:

```
Không EBS-Optimized:
  EC2 Network ─────────── EBS Traffic + App Traffic (tranh chấp)
  
EBS-Optimized:
  EC2 Network ─────────── App Traffic (network interface)
  EBS Network ─────────── EBS Traffic (dedicated path)
  
→ Tránh I/O contention (tranh chấp I/O) giữa ứng dụng và storage
```

```bash
# Kiểm tra instance có EBS-Optimized không
aws ec2 describe-instance-attribute \
  --instance-id i-xxx \
  --attribute ebsOptimized

# Bật EBS-Optimized khi launch (hầu hết instance types hiện đại tự bật)
aws ec2 run-instances \
  --image-id ami-xxx \
  --instance-type m5.xlarge \
  --ebs-optimized  # Explicit flag
```

> **Lưu ý:** Hầu hết instance types từ năm 2019 trở đi đã **mặc định EBS-Optimized**. Không cần bật thủ công.

---

## 🔍 Benchmark Và Monitoring

### Dùng fio (Flexible I/O Tester — Công Cụ Kiểm Tra I/O Linh Hoạt)

```bash
# Cài đặt
sudo yum install -y fio

# Test random read IOPS (database pattern)
sudo fio \
  --filename=/dev/nvme1n1 \
  --direct=1 \
  --rw=randread \
  --bs=4k \
  --iodepth=256 \
  --numjobs=4 \
  --runtime=60 \
  --group_reporting \
  --name=random-read-iops

# Test sequential read throughput (analytics pattern)
sudo fio \
  --filename=/dev/nvme1n1 \
  --direct=1 \
  --rw=read \
  --bs=1M \
  --iodepth=16 \
  --numjobs=1 \
  --runtime=60 \
  --name=seq-read-throughput

# Test mixed read/write (production-like)
sudo fio \
  --filename=/dev/nvme1n1 \
  --direct=1 \
  --rw=randrw \
  --rwmixread=70 \
  --bs=16k \
  --iodepth=64 \
  --numjobs=4 \
  --runtime=120 \
  --group_reporting \
  --name=mixed-workload
```

### CloudWatch Dashboard

```bash
# Tạo CloudWatch alarm cho VolumeQueueLength
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-QueueLength-High" \
  --alarm-description "EBS queue length > 1 for 5 minutes" \
  --metric-name VolumeQueueLength \
  --namespace AWS/EBS \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --period 300 \
  --evaluation-periods 3 \
  --statistic Average \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:xxx:storage-alerts
```

---

## 📋 Performance Checklist — Danh Sách Kiểm Tra

### Trước Khi Deploy

- [ ] Volume type phù hợp với workload pattern (random vs sequential)?
- [ ] Provisioned IOPS đủ? (Kiểm tra công thức: VolumeQueueLength < 1)
- [ ] EC2 instance EBS bandwidth >= provisioned IOPS × 16KB?
- [ ] EBS-Optimized instance?
- [ ] I/O scheduler đã chỉnh về "none" cho NVMe?
- [ ] Filesystem mount options đã tối ưu (noatime, data mode)?

### Khi Gặp Vấn Đề Hiệu Suất

1. Kiểm tra VolumeQueueLength > 1? → Tăng IOPS
2. Kiểm tra BurstBalance = 0? → Tăng size (gp2) hoặc provision IOPS (gp3)
3. So sánh IOPS dùng thực tế vs provisioned?
4. Kiểm tra EC2 bandwidth limit?
5. Kiểm tra OS I/O scheduler?
6. Có Network contention không? (EBS-Optimized chưa?)

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa IOPS và Throughput?**
> IOPS (Input/Output Operations Per Second — Thao Tác Vào/Ra Mỗi Giây) đo số lượng thao tác I/O nhỏ mỗi giây — phù hợp database OLTP với nhiều transaction 4–16KB. Throughput đo lượng dữ liệu MB/giây — phù hợp analytics và streaming với I/O lớn tuần tự 256KB–1MB.

**Q: VolumeQueueLength cao có ý nghĩa gì?**
> VolumeQueueLength > 1 nghĩa là có I/O requests đang chờ volume xử lý — volume bị overloaded. Cần tăng provisioned IOPS, nâng volume type lên io2, hoặc xem xét EC2 instance bandwidth limit.

**Q: Tại sao nên dùng I/O scheduler "none" cho EBS NVMe?**
> EBS SSD NVMe đã có internal queueing và scheduling. Thêm OS scheduler tạo overhead không cần thiết và thậm chí có thể làm giảm hiệu suất. "none" (no-op) đưa requests trực tiếp tới device queue, phù hợp nhất cho SSD.

**Q: Khi nào dùng RAID 0 với EBS?**
> Khi cần IOPS hoặc throughput vượt quá giới hạn 1 volume (gp3 max 16.000 IOPS, 1.000 MB/s). RAID 0 ghép nhiều gp3 volumes để đạt hiệu suất cao hơn với chi phí thấp hơn io2. Nhược điểm: cần snapshot cả array để backup.

---

## 🔗 Điều Hướng

- **Trước:** [3-ebs-multi-attach.md](./3-ebs-multi-attach.md) — Multi-Attach
- **Tiếp theo:** [5-encryption-with-kms.md](./5-encryption-with-kms.md) — Mã Hóa KMS
- **Liên quan:** [../07-monitoring/2-cloudwatch-ebs-metrics.md](../07-monitoring/2-cloudwatch-ebs-metrics.md) — EBS Metrics

---

**Cập Nhật Lần Cuối:** 2026-05-15
