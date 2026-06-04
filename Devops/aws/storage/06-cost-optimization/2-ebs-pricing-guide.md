# Bảng Giá EBS — gp3 vs io2 vs st1 — Tính Toán

> EBS — Elastic Block Store — Lưu Trữ Khối Linh Hoạt tính phí theo dung lượng đã provision (cấp phát trước), không phải dung lượng thực dùng. Hiểu cấu trúc giá để chọn đúng loại volume và tránh overprovision.

## 📚 Mục Lục

1. [Cấu Trúc Chi Phí EBS](#1-cấu-trúc-chi-phí-ebs)
2. [So Sánh Giá Các Loại Volume](#2-so-sánh-giá-các-loại-volume)
3. [Phân Tích gp3 — General Purpose SSD](#3-phân-tích-gp3)
4. [Phân Tích io2 — Provisioned IOPS SSD](#4-phân-tích-io2)
5. [Phân Tích st1 và sc1 — HDD](#5-phân-tích-st1-và-sc1)
6. [Chi Phí Snapshot — Ảnh Chụp Nhanh](#6-chi-phí-snapshot)
7. [Tính Toán Chi Phí Thực Tế](#7-tính-toán-chi-phí-thực-tế)
8. [Chiến Lược Giảm Chi Phí EBS](#8-chiến-lược-giảm-chi-phí-ebs)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Cấu Trúc Chi Phí EBS

### Nguyên Tắc Tính Phí

```
Chi phí EBS KHÔNG phụ thuộc vào:
  ❌ Dữ liệu thực sự lưu trên volume
  ❌ Số lần đọc/ghi (với gp3, st1, sc1)
  ❌ Volume có được attach EC2 hay không

Chi phí EBS PHỤ THUỘC vào:
  ✅ Dung lượng đã provision (GB)
  ✅ IOPS đã provision (với io1/io2)
  ✅ Throughput đã provision (MB/s, với gp3 nếu vượt base)
  ✅ Snapshot dung lượng thực tế lưu (incremental)
```

### Công Thức Chi Phí EBS

```
Tổng Chi Phí EBS = Storage + IOPS (nếu có) + Throughput (nếu có) + Snapshots

gp3: Cost = (GB × $0.08) + (IOPS_extra × $0.005) + (MB/s_extra × $0.04)
  Trong đó: IOPS_extra = max(0, IOPS_provision - 3,000)
            MB/s_extra = max(0, throughput_provision - 125)

io2: Cost = (GB × $0.125) + (IOPS × $0.065)
  Không có "free tier IOPS" — tất cả IOPS đều tính phí

Snapshot: Cost = unique_data_GB × $0.05
  (Chỉ tính phần dữ liệu thay đổi, incremental — tăng dần)
```

---

## 2. So Sánh Giá Các Loại Volume

### Bảng Giá EBS (us-east-1, 2026)

| Loại Volume | Giá Storage | Giá IOPS | Giá Throughput | Ghi Chú |
|------------|------------|---------|---------------|---------|
| **gp3** (General Purpose SSD) | $0.08/GB-month | Miễn phí đến 3,000 IOPS; $0.005/IOPS/month sau đó | Miễn phí đến 125 MB/s; $0.04/MB/s/month sau đó | Tốt nhất cho hầu hết workload |
| **gp2** (General Purpose SSD — cũ) | $0.10/GB-month | Tính theo GB (3 IOPS/GB, max 16,000) | Tự động theo IOPS | Không khuyến nghị, dùng gp3 |
| **io1** (Provisioned IOPS SSD) | $0.125/GB-month | $0.065/IOPS/month | Không tách biệt | Thế hệ cũ |
| **io2** (Provisioned IOPS SSD — mới) | $0.125/GB-month | $0.065/IOPS/month (đến 32,000); $0.046/IOPS sau đó | Không tách biệt | Durability cao hơn io1 |
| **st1** (Throughput Optimized HDD) | $0.045/GB-month | Không tách biệt | Tự động (max 500 MB/s) | Big data, log |
| **sc1** (Cold HDD) | $0.015/GB-month | Không tách biệt | Tự động (max 250 MB/s) | Truy cập ít, lưu trữ rẻ |

### Khi Nào Dùng Loại Nào

```
gp3: 90% workload thông thường
  → Web servers, boot volumes, dev/test, small databases

io2: Cần IOPS cực cao (>16,000) và độ bền cao hơn
  → Oracle, SQL Server, SAP HANA, critical databases
  → io2 Block Express: đến 256,000 IOPS, dành cho SAP

st1: Throughput cao, sequential read (đọc tuần tự)
  → Hadoop, Kafka, log processing, ETL — Extract Transform Load

sc1: Ít truy cập nhất, cần lưu trữ giá rẻ
  → Cold data, backup local, archival cần block storage
```

---

## 3. Phân Tích gp3

### gp3 — Thế Hệ Mới Thay Thế gp2

```
gp3 Advantages (Lợi Thế) so với gp2:
  1. Rẻ hơn 20% ($0.08 vs $0.10/GB-month)
  2. IOPS và throughput độc lập — không ràng buộc vào dung lượng
  3. 3,000 IOPS baseline miễn phí (gp2 chỉ 3 IOPS/GB)
  4. Throughput baseline 125 MB/s (gp2 chỉ 128 MB/s gắn với IOPS)

Migration gp2 → gp3:
  - Không downtime — thay đổi live
  - Tiết kiệm 20% ngay lập tức
  - IOPS baseline tốt hơn cho volume nhỏ
```

### Tính Chi Phí gp3

```
Ví dụ 1: Volume 100 GB, 3,000 IOPS, 125 MB/s (base defaults)
  Storage: 100 × $0.08 = $8/month
  IOPS: Trong 3,000 free → $0
  Throughput: Trong 125 MB/s free → $0
  Tổng: $8/month

Ví dụ 2: Volume 500 GB, 10,000 IOPS, 500 MB/s
  Storage: 500 × $0.08 = $40
  IOPS extra: (10,000 - 3,000) × $0.005 = 7,000 × $0.005 = $35
  Throughput extra: (500 - 125) × $0.04 = 375 × $0.04 = $15
  Tổng: $90/month

Ví dụ 3: Volume 1 TB, 16,000 IOPS, 1,000 MB/s
  Storage: 1,000 × $0.08 = $80
  IOPS extra: (16,000 - 3,000) × $0.005 = 13,000 × $0.005 = $65
  Throughput extra: (1,000 - 125) × $0.04 = 875 × $0.04 = $35
  Tổng: $180/month
```

### Khi Nào gp3 Đủ

```
gp3 đủ cho:
  ✅ Boot volume EC2 (thường 20–100 GB, 3,000 IOPS dư dả)
  ✅ Web/app server (thường cần <3,000 IOPS)
  ✅ Dev/test environments
  ✅ Small-medium databases (<3,000 IOPS)
  ✅ Bất kỳ workload nào cần <16,000 IOPS

gp3 KHÔNG đủ khi:
  ❌ Cần >16,000 IOPS → Dùng io2
  ❌ Cần latency <1ms nhất quán → Dùng io2
  ❌ SAP HANA, critical Oracle → Dùng io2 Block Express
```

---

## 4. Phân Tích io2

### io2 — Provisioned IOPS SSD — SSD IOPS Được Cấp Phát Sẵn

```
io2 Characteristics (Đặc Điểm):
  - IOPS max: 64,000 (io2), 256,000 (io2 Block Express)
  - Throughput max: 4,000 MB/s (io2 Block Express)
  - Durability: 99.999% (io2), so với 99.8–99.9% cho gp3
  - Multi-Attach: Được hỗ trợ (gắn vào nhiều EC2 cùng lúc)
  - Ratio IOPS:GB = 1:1 đến 50:1 (io2), 1:1 đến 1,000:1 (io2 Block Express)
```

### Tính Chi Phí io2

```
Ví dụ 1: Database server — 200 GB, 20,000 IOPS
  Storage: 200 × $0.125 = $25
  IOPS: 20,000 × $0.065 = $1,300
  Tổng: $1,325/month

→ So với gp3 (nếu gp3 đủ đáp ứng):
  gp3: 200 × $0.08 + (20,000-3,000) × $0.005 = $16 + $85 = $101
  io2: $1,325 (gấp 13 lần!)

→ Kết luận: Chỉ dùng io2 khi thực sự cần reliability/latency cao hơn.
  Đừng dùng io2 chỉ vì "an toàn hơn" — chi phí rất cao.
```

### So Sánh io2 vs gp3 Theo IOPS

```
IOPS   | gp3 Cost | io2 Cost | Nên Dùng
-------|----------|----------|----------
3,000  | $8 (100GB)| $220 (100GB+3K IOPS) | gp3
10,000 | $43 (100GB)| $674 (100GB+10K IOPS)| gp3
16,000 | $73 (100GB)| $1,050 (100GB+16K IOPS)| gp3 (max)
20,000 | Không hỗ trợ | $1,300 (200GB) | io2 bắt buộc
64,000 | Không hỗ trợ | $4,185 (200GB) | io2

→ gp3 rẻ hơn đáng kể ở mọi mức IOPS nó hỗ trợ (<16,000)
→ io2 chỉ khi cần >16,000 IOPS hoặc durability/latency đặc biệt
```

---

## 5. Phân Tích st1 và sc1

### st1 — Throughput Optimized HDD — HDD Tối Ưu Thông Lượng

```
st1 phù hợp với:
  ✅ Sequential workloads (đọc/ghi tuần tự)
  ✅ Throughput cao (đến 500 MB/s)
  ✅ Dữ liệu lớn: Hadoop, Kafka, log streaming
  ✅ Khi cần throughput nhưng không cần IOPS cao

st1 không phù hợp với:
  ❌ Random I/O (đọc/ghi ngẫu nhiên) — latency cao
  ❌ Boot volume — không hỗ trợ
  ❌ Database OLTP — cần random I/O
  ❌ Volume <125 GB (kích thước tối thiểu)

Chi phí st1:
  Volume 2 TB: 2,000 GB × $0.045 = $90/month
  So với gp3 2 TB: 2,000 × $0.08 = $160/month
  → st1 tiết kiệm 44% với workload sequential!
```

### sc1 — Cold HDD — HDD Lạnh

```
sc1 rẻ nhất trong EBS ($0.015/GB-month) nhưng hạn chế nhiều:
  - Throughput max: 250 MB/s (còn st1 500 MB/s)
  - Không thể boot volume
  - Dành cho dữ liệu ít truy cập nhất cần block storage

Chi phí sc1:
  Volume 5 TB: 5,000 GB × $0.015 = $75/month
  So với st1 5 TB: 5,000 × $0.045 = $225/month
  → sc1 tiết kiệm 67% so với st1!

Khi nào sc1 hơn S3:
  - Cần POSIX filesystem interface
  - Cần block-level access
  - Dữ liệu lớn không muốn trả transfer fee S3
  Nhưng S3 Glacier ($0.004/GB) vẫn rẻ hơn sc1 ($0.015/GB) → Cân nhắc kỹ!
```

---

## 6. Chi Phí Snapshot

### Cơ Chế Incremental Snapshot — Snapshot Tăng Dần

```
Snapshot 1 (Full — Đầy Đủ):
  Volume 100 GB, dữ liệu thực 80 GB → Tính phí 80 GB

Snapshot 2 (sau 1 ngày, thay đổi 5 GB):
  Chỉ lưu 5 GB thay đổi → Tính phí 5 GB

Snapshot 3 (sau 1 ngày nữa, thay đổi 3 GB):
  Chỉ lưu 3 GB → Tính phí 3 GB

Tổng phí snapshot = (80 + 5 + 3) GB × $0.05 = $4.40/month
Nếu xóa snapshot 2 → snapshot 1 và 3 vẫn độc lập đủ
```

### Chi Phí Snapshot Thực Tế

```
Chiến lược snapshot phổ biến:
  - Daily (hàng ngày) 7 ngày gần nhất
  - Weekly (hàng tuần) 4 tuần gần nhất
  - Monthly (hàng tháng) 12 tháng gần nhất

Volume 100 GB với 5 GB thay đổi/ngày:
  Daily 7: 80 + 5×6 = 110 GB snapshot data
  Weekly 4: 5×7×3 = 105 GB (tuần 2, 3, 4)
  Monthly 12: 5×7×52 ≈ ảnh hưởng thấp do incremental dùng chung
  
  Ước tính: ~250–300 GB snapshot
  Chi phí: 300 × $0.05 = $15/month chỉ cho snapshots
```

### EBS Snapshot Archive — Lưu Trữ Snapshot Cũ

```
EBS Snapshot Archive — Lưu Trữ Snapshot Vào Glacier:
  - Giá: $0.0125/GB-month (thay vì $0.05/GB-month thường)
  - Tiết kiệm 75% cho snapshot hiếm dùng
  - Restore mất 24–72 giờ (tạm thời)
  - Tối thiểu 90 ngày lưu trữ

Khi nào dùng:
  ✅ Snapshot compliance dài hạn (1–7 năm)
  ✅ Snapshot không cần restore nhanh
  ✅ Snapshot tổng hợp cuối năm/quý
```

---

## 7. Tính Toán Chi Phí Thực Tế

### Ví Dụ 1: Web Application Server

```
Cấu hình:
  - Boot volume: gp3 30 GB, 3,000 IOPS (default)
  - Data volume: gp3 200 GB, 5,000 IOPS
  - Snapshot daily 7 ngày, 2 GB thay đổi/ngày

Chi phí:
  Boot volume: 30 × $0.08 = $2.40
  Data volume storage: 200 × $0.08 = $16
  Data volume IOPS extra: (5,000-3,000) × $0.005 = $10
  Snapshots: (30 + 200 + 2×6) GB = 242 GB × $0.05 = $12.10
  
  Tổng: $40.50/month
```

### Ví Dụ 2: Database Server (So Sánh gp3 vs io2)

```
Yêu cầu: Database cần 500 GB, 20,000 IOPS

Với gp3:
  gp3 max 16,000 IOPS → Không đủ yêu cầu!

Với io2:
  Storage: 500 × $0.125 = $62.50
  IOPS: 20,000 × $0.065 = $1,300
  Tổng: $1,362.50/month

Tối ưu: Nếu có thể giảm xuống 15,000 IOPS → Dùng gp3:
  Storage: 500 × $0.08 = $40
  IOPS: (15,000-3,000) × $0.005 = $60
  Tổng: $100/month (tiết kiệm $1,262/month = 93%!)
```

### Ví Dụ 3: Data Pipeline — Hadoop Cluster

```
Cấu hình mỗi node (5 nodes):
  - OS: gp3 20 GB
  - Data: st1 2,000 GB (sequential reads lớn)

Chi phí mỗi node/month:
  OS volume: 20 × $0.08 = $1.60
  Data volume: 2,000 × $0.045 = $90

Tổng 5 nodes: 5 × $91.60 = $458/month

Nếu dùng gp3 cho data: 5 × (20 + 2,000) × $0.08 = $808/month
→ st1 tiết kiệm $350/month = 43%!
```

---

## 8. Chiến Lược Giảm Chi Phí EBS

### 1. Migration gp2 → gp3 (Ưu Tiên Cao)

```bash
# Tìm tất cả gp2 volumes
aws ec2 describe-volumes \
  --filters Name=volume-type,Values=gp2 \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,State:State}' \
  --output table

# Thay đổi sang gp3 (không cần restart, live modification)
aws ec2 modify-volume \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125

# Theo dõi tiến độ
aws ec2 describe-volumes-modifications \
  --volume-ids vol-xxxxxxxxxxxxxxxxx
```

### 2. Xóa Volume Không Gắn (Unattached Volume)

```bash
# Tìm volume trạng thái "available" (không gắn EC2)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Created:CreateTime}' \
  --output table

# Tạo snapshot trước khi xóa (an toàn)
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Backup before deletion $(date)"

# Xóa volume (SAU KHI đã snapshot)
aws ec2 delete-volume --volume-id vol-xxxxxxxxxxxxxxxxx
```

### 3. DLM — Data Lifecycle Manager — Quản Lý Vòng Đời Snapshot Tự Động

```bash
# Tạo DLM policy tự động snapshot và xóa cũ
aws dlm create-lifecycle-policy \
  --description "Daily snapshots, keep 7 days" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::ACCOUNT:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "backup", "Value": "true"}],
    "Schedules": [{
      "Name": "Daily backup",
      "CreateRule": {
        "Interval": 24,
        "IntervalUnit": "HOURS",
        "Times": ["02:00"]
      },
      "RetainRule": {"Count": 7},
      "CopyTags": true
    }]
  }'
```

### 4. Right-Sizing — Điều Chỉnh Đúng Kích Thước

```
Dấu hiệu overprovision (cấp phát thừa):
  - VolumeQueueLength (CloudWatch) luôn ~0
  - VolumeReadOps/WriteOps (CloudWatch) << IOPS đã provision
  - Volume utilization < 20% theo AWS Compute Optimizer

Công cụ:
  - AWS Compute Optimizer → Đề xuất tối ưu EBS
  - CloudWatch → Xem metrics thực tế
  - Cost Explorer → Xem chi phí từng volume
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: gp3 khác gp2 như thế nào về chi phí và hiệu suất?**

> gp3 rẻ hơn gp2 20% ($0.08 vs $0.10/GB-month). gp3 cung cấp 3,000 IOPS baseline miễn phí bất kể dung lượng, trong khi gp2 chỉ có 3 IOPS/GB (volume 100GB chỉ được 300 IOPS baseline). gp3 cho phép cấu hình IOPS và throughput độc lập với dung lượng, còn gp2 gắn chặt IOPS với GB. Không có lý do gì để giữ gp2 — nên migrate hết sang gp3.

**Q: Khi nào nên dùng io2 thay vì gp3?**

> io2 cần thiết khi workload yêu cầu hơn 16,000 IOPS (giới hạn max của gp3), cần latency dưới 1ms nhất quán, cần durability 99.999% (io2 cao hơn gp3 là 99.8–99.9%), hoặc cần Multi-Attach để gắn vào nhiều EC2 đồng thời. io2 đắt hơn gp3 đáng kể ($0.065/IOPS vs $0.005/IOPS extra) nên phải có yêu cầu kỹ thuật rõ ràng mới dùng.

**Q: Chi phí EBS Snapshot được tính như thế nào?**

> EBS Snapshot tính phí theo dung lượng thực tế lưu trữ (không phải dung lượng volume), với giá $0.05/GB-month. Snapshot incremental — chỉ lưu phần dữ liệu thay đổi so với snapshot trước, nên snapshot sau thường nhỏ hơn snapshot đầu tiên rất nhiều. Khi xóa một snapshot, AWS đảm bảo snapshot còn lại vẫn đủ dữ liệu để restore. Dùng DLM để tự động hóa retention policy thay vì quản lý thủ công.

**Q: Làm sao phát hiện EBS volume đang lãng phí chi phí?**

> Ba cách chính: (1) Tìm volume ở trạng thái "available" (không gắn EC2) — vẫn tính phí lưu trữ; (2) Dùng CloudWatch xem VolumeReadOps/WriteOps — nếu gần 0 nghĩa là volume không được dùng; (3) Dùng AWS Compute Optimizer — tự động phân tích và đề xuất right-sizing. Ngoài ra, còn cần review gp2 volumes để migrate sang gp3 tiết kiệm 20% ngay.

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
