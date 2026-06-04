# AMI & Storage — Ảnh Máy Ảo và Lưu Trữ EC2

> **AMI — Amazon Machine Image — Ảnh Máy Ảo** là template chứa OS, cấu hình, và dữ liệu để khởi tạo EC2 instance. **EBS — Elastic Block Store — Lưu Trữ Khối Mạng** là giải pháp lưu trữ bền vững chính của EC2.

## 📚 Mục Lục

1. [AMI — Amazon Machine Image](#ami--amazon-machine-image)
2. [EBS — Elastic Block Store](#ebs--elastic-block-store)
3. [Instance Store — Lưu Trữ Tạm Thời](#instance-store--lưu-trữ-tạm-thời)
4. [EBS Volume Types](#ebs-volume-types)
5. [EBS Snapshots](#ebs-snapshots)
6. [So Sánh Loại Lưu Trữ](#so-sánh-loại-lưu-trữ)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## AMI — Amazon Machine Image

### AMI Là Gì?

**AMI** là "bản in" (template) của một máy chủ ảo, bao gồm:
- **Root volume snapshot** — Ảnh chụp ổ đĩa hệ điều hành
- **Launch permissions** — Ai được phép dùng AMI này
- **Block device mappings** — Định nghĩa volumes gắn vào instance

```
AMI  →  EC2 Instance
         ├── Root Volume (EBS hoặc Instance Store)
         ├── Optional Data Volumes
         └── OS + Installed Software
```

### Các Loại AMI

#### Theo Nguồn Gốc

| Loại                | Mô Tả                                              | Tin Cậy |
| ------------------- | -------------------------------------------------- | ------- |
| **AWS-provided**    | Amazon Linux 2/2023, Ubuntu, Windows chính thức   | ✅ Cao  |
| **AWS Marketplace** | Vendor-provided (Red Hat, SUSE, commercial apps)  | ✅ Tốt  |
| **Community AMIs**  | Do cộng đồng đăng, không kiểm duyệt               | ⚠️ Cẩn thận |
| **Custom AMIs**     | Bạn tự tạo từ instance hiện có                    | ✅ Kiểm soát hoàn toàn |

#### Theo Root Device Type

| Loại                    | Root Device     | Boot Time | Phổ Biến |
| ----------------------- | --------------- | --------- | -------- |
| **EBS-backed AMI**      | EBS Volume      | ~30-60s   | 99%      |
| **Instance Store-backed** | Instance Store | ~5-10 phút | < 1%   |

> **Hầu hết AMI hiện đại đều EBS-backed.** Instance Store-backed AMI gần như không còn được dùng trong production.

### Tạo Custom AMI

Quy trình chuẩn khi bake AMI (đưa ứng dụng vào AMI để deploy nhanh):

```
1. Launch base instance (từ Amazon Linux 2023 hoặc Ubuntu)
2. Cài đặt dependencies (Java, Python, runtime...)
3. Cấu hình application
4. Test hoạt động đúng
5. Create Image (AWS Console / CLI)
6. AMI được tạo với snapshot của root volume
7. Dùng AMI mới cho Auto Scaling Group hoặc deploy mới
```

```bash
# Tạo AMI từ instance đang chạy
aws ec2 create-image \
  --instance-id i-0abc123def456789 \
  --name "my-app-v1.2.0-$(date +%Y%m%d)" \
  --description "Application version 1.2.0 with Java 21" \
  --no-reboot  # instance tiếp tục chạy khi tạo AMI

# Kiểm tra trạng thái AMI
aws ec2 describe-images \
  --image-ids ami-0abc123 \
  --query 'Images[0].State'

# Copy AMI sang region khác (cho multi-region deployment)
aws ec2 copy-image \
  --source-image-id ami-0abc123 \
  --source-region us-east-1 \
  --region ap-southeast-1 \
  --name "my-app-v1.2.0-ap-southeast-1"
```

### AMI Encryption (Mã Hóa AMI)

```bash
# Tạo encrypted AMI từ unencrypted AMI
aws ec2 copy-image \
  --source-image-id ami-0abc123 \
  --source-region us-east-1 \
  --region us-east-1 \
  --name "my-app-encrypted" \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/my-key
```

> **Best Practice:** Luôn encrypt AMI cho production. Encrypted AMI kế thừa key từ snapshot, đảm bảo data-at-rest encryption.

### AMI Lifecycle

```
Tạo AMI
  │
  ├── Sử dụng để launch instances
  │
  ├── Deprecate AMI (đánh dấu lỗi thời, nhưng vẫn dùng được)
  │   aws ec2 enable-image-deprecation --image-id ami-xxx --deprecate-at "2027-01-01T00:00:00Z"
  │
  └── Deregister AMI (xóa AMI, nhưng snapshot vẫn còn)
      aws ec2 deregister-image --image-id ami-xxx
      # Phải xóa snapshot thủ công sau đó:
      aws ec2 delete-snapshot --snapshot-id snap-xxx
```

---

## EBS — Elastic Block Store

### EBS Là Gì?

**EBS — Elastic Block Store** là dịch vụ lưu trữ dạng khối (block storage) qua mạng, tương tự như ổ đĩa cứng nhưng kết nối qua network (NVMe-over-Fabrics trong Nitro instances).

```
Đặc điểm quan trọng:
  ✅ Persistent    — Dữ liệu tồn tại qua stop/start/reboot
  ✅ Network-based — Có thể detach và attach sang instance khác
  ✅ Snapshot      — Sao lưu lên S3 theo điểm thời gian
  ✅ Encrypted     — Mã hóa AES-256 (tùy chọn hoặc mặc định)
  ⚠️ AZ-bound     — Volume chỉ dùng được trong cùng AZ với instance
  ⚠️ 1 attachment  — Mỗi volume chỉ attach 1 instance (trừ EBS Multi-Attach io2)
```

### EBS vs Local Disk

```
Instance (AWS host)
│
├── EBS Volume (qua network)
│   Latency: ~0.1-1ms
│   IOPS: 16,000-256,000 (io2 Block Express)
│   Size: 1GB - 64TB
│   Persist: ✅ Qua stop/start
│
└── Instance Store (NVMe gắn trực tiếp)
    Latency: ~0.01-0.05ms (10x nhanh hơn)
    IOPS: Lên đến 3.3M IOPS (i3en.24xlarge)
    Size: Cố định theo instance type
    Persist: ❌ Mất khi stop/terminate
```

---

## Instance Store — Lưu Trữ Tạm Thời

### Đặc Điểm Instance Store

**Instance Store** (còn gọi là ephemeral storage — lưu trữ tạm thời) là NVMe SSD gắn trực tiếp vào host vật lý nơi instance chạy.

```
Dữ liệu trên Instance Store BỊ MẤT khi:
  ❌ Instance được stop
  ❌ Instance bị terminate
  ❌ Host vật lý bị lỗi

Dữ liệu KHÔNG bị mất khi:
  ✅ Instance reboot (khởi động lại, vẫn trên cùng host)
```

### Use Cases Phù Hợp

```
✅ Phù hợp (dữ liệu tạm thời):
  - Buffer / Cache tạm thời của ứng dụng
  - Scratch space cho batch processing
  - Replicated databases (Cassandra, MongoDB replica)
  - Temp files trong quá trình xử lý
  - Hadoop/Spark intermediate data

❌ Không phù hợp:
  - Root volume (OS)
  - Application data cần bền vững
  - Database primary storage (trừ khi có external replication)
  - Bất kỳ data nào không thể tái tạo
```

### Instances Có Instance Store

```
Instance Type   | NVMe Volumes | Total Size
----------------|--------------|------------------
i3.large        | 1 × 475GB    | 475 GB NVMe SSD
i3.xlarge       | 1 × 950GB    | 950 GB NVMe SSD
i3.4xlarge      | 2 × 1.9TB    | 3.8 TB NVMe SSD
i3en.large      | 1 × 1.25TB   | 1.25 TB NVMe SSD
d3.xlarge       | 3 × 2TB HDD  | 6 TB HDD
c5d.large       | 1 × 50GB     | 50 GB NVMe SSD (nhỏ, cho scratch)
m5d.large       | 1 × 75GB     | 75 GB NVMe SSD
```

---

## EBS Volume Types

### Tổng Quan Các Loại EBS

```
General Purpose SSD:
  gp3 — Mới nhất, linh hoạt, default choice
  gp2 — Thế hệ cũ, sắp bị thay thế hoàn toàn bởi gp3

Provisioned IOPS SSD (IOPS cam kết):
  io2 Block Express — Cao cấp nhất, sub-millisecond latency
  io2               — Production database
  io1               — Thế hệ cũ, dùng io2 thay thế

Throughput Optimized HDD (Băng Thông Cao):
  st1 — Streaming workload (Kafka, big data)

Cold HDD (Chi Phí Thấp Nhất):
  sc1 — Ít truy cập, lưu trữ dài hạn
```

### Chi Tiết Từng Loại

#### gp3 — General Purpose SSD (Mặc Định Hiện Tại)

```
Dung lượng:   1 GB – 16 TB
IOPS:         3,000 (mặc định) – 16,000 (có thể tăng độc lập với size)
Throughput:   125 MB/s (mặc định) – 1,000 MB/s
Latency:      < 1 ms (single-digit millisecond)
Chi phí:      $0.08/GB/tháng + IOPS riêng nếu > 3,000
Use case:     Boot volumes, web server, dev/test, virtual desktop
```

> **gp3 vs gp2:** gp3 cho phép tăng IOPS và throughput độc lập với dung lượng. gp2 buộc tăng dung lượng để có thêm IOPS (3 IOPS/GB). gp3 thường rẻ hơn gp2 cho cùng hiệu năng.

#### io2 Block Express — Provisioned IOPS SSD Cao Cấp

```
Dung lượng:   4 GB – 64 TB
IOPS:         Đến 256,000 IOPS (tỉ lệ 1,000:1 với GB)
Throughput:   Đến 4,000 MB/s
Latency:      Sub-millisecond (< 0.5ms)
Durability:   99.999% (so với 99.8% của gp3)
Chi phí:      $0.125/GB/tháng + $0.065/provisioned IOPS
Use case:     Oracle RAC, SAP HANA, latency-sensitive production DB
```

#### io2 — Provisioned IOPS SSD

```
Dung lượng:   4 GB – 16 TB
IOPS:         Đến 64,000 IOPS
Throughput:   Đến 1,000 MB/s
Durability:   99.999%
Multi-Attach: Có (attach tới 16 Nitro instances trong cùng AZ)
Use case:     Production MySQL/PostgreSQL, MongoDB, Elasticsearch
```

#### io1 — Thế Hệ Cũ

> Dùng io2 thay thế — hiệu năng tốt hơn với cùng giá.

#### st1 — Throughput Optimized HDD (Ổ Cứng Tối Ưu Băng Thông)

```
Dung lượng:   125 GB – 16 TB
IOPS:         Tối đa 500 IOPS
Throughput:   Tối đa 500 MB/s (burst), 40 MB/s/TB baseline
Chi phí:      $0.045/GB/tháng (rẻ hơn SSD nhiều)
Use case:     Kafka, EMR/Hadoop, data lake, log storage
Lưu ý:        KHÔNG thể dùng làm boot volume
```

#### sc1 — Cold HDD (Ổ Cứng Lạnh Chi Phí Thấp Nhất)

```
Dung lượng:   125 GB – 16 TB
IOPS:         Tối đa 250 IOPS
Throughput:   Tối đa 250 MB/s (burst)
Chi phí:      $0.015/GB/tháng (rẻ nhất)
Use case:     Backup archive, data infrequently accessed
Lưu ý:        KHÔNG thể dùng làm boot volume
```

### Bảng So Sánh Nhanh EBS Volume Types

| Type  | Max IOPS    | Max Throughput | Latency    | Chi Phí/GB | Boot? |
| ----- | ----------- | -------------- | ---------- | ---------- | ----- |
| gp3   | 16,000      | 1,000 MB/s     | < 1ms      | $0.08      | ✅    |
| gp2   | 16,000      | 250 MB/s       | < 1ms      | $0.10      | ✅    |
| io2bx | 256,000     | 4,000 MB/s     | < 0.5ms    | $0.125+    | ✅    |
| io2   | 64,000      | 1,000 MB/s     | < 1ms      | $0.125+    | ✅    |
| st1   | 500         | 500 MB/s       | few ms     | $0.045     | ❌    |
| sc1   | 250         | 250 MB/s       | few ms     | $0.015     | ❌    |

---

## EBS Snapshots

### Snapshot Là Gì?

**EBS Snapshot** là bản sao lưu tại một thời điểm (point-in-time backup) của EBS volume, lưu trữ trên S3 (quản lý bởi AWS, không thấy trong S3 bucket của bạn).

```
Đặc điểm snapshot:
  ✅ Incremental   — Chỉ lưu data thay đổi kể từ snapshot trước
  ✅ Cross-AZ      — Dùng để tạo volume ở AZ khác trong cùng region
  ✅ Cross-Region  — Copy snapshot sang region khác
  ✅ Encrypted     — Snapshot của encrypted volume tự động encrypted
  ✅ AMI Source    — AMI được tạo từ snapshot
```

### Snapshot Lifecycle

```bash
# Tạo snapshot thủ công
aws ec2 create-snapshot \
  --volume-id vol-0abc123 \
  --description "Before deployment $(date +%Y%m%d-%H%M)"

# Tạo volume từ snapshot (có thể khác AZ)
aws ec2 create-volume \
  --snapshot-id snap-0abc123 \
  --availability-zone us-east-1b \
  --volume-type gp3 \
  --size 100  # Có thể lớn hơn snapshot

# Copy snapshot sang region khác
aws ec2 copy-snapshot \
  --source-snapshot-id snap-0abc123 \
  --source-region us-east-1 \
  --destination-region ap-southeast-1 \
  --description "DR copy"

# Xóa snapshot
aws ec2 delete-snapshot --snapshot-id snap-0abc123
```

### Amazon DLM — Data Lifecycle Manager — Quản Lý Vòng Đời Dữ Liệu

**DLM** tự động hóa việc tạo, giữ lại, và xóa snapshots theo policy.

```json
// Ví dụ DLM Policy — snapshot mỗi ngày, giữ 7 ngày
{
  "ResourceTypes": ["VOLUME"],
  "TargetTags": [{"Key": "Backup", "Value": "daily"}],
  "Schedules": [{
    "Name": "daily-backup",
    "CreateRule": {
      "Interval": 24,
      "IntervalUnit": "HOURS",
      "Times": ["03:00"]
    },
    "RetainRule": {"Count": 7},
    "CopyTags": true
  }]
}
```

### Fast Snapshot Restore — FSR (Khôi Phục Snapshot Nhanh)

Bình thường, volume được khôi phục từ snapshot cần "warm-up" — IOPS đầy đủ chỉ có sau khi từng block được đọc lần đầu. **FSR** pre-initialize volume để IOPS đầy đủ ngay từ đầu.

```bash
# Bật FSR cho snapshot trong AZ cụ thể
aws ec2 enable-fast-snapshot-restores \
  --availability-zones us-east-1a us-east-1b \
  --source-snapshot-ids snap-0abc123
```

> **Chi phí FSR:** Tính theo số snapshot × số AZ × giờ. Dùng cho production DB volume cần restore nhanh trong disaster recovery.

---

## So Sánh Loại Lưu Trữ

```
                    EBS gp3        EBS io2        Instance Store    S3
Persistence         ✅ Bền vững    ✅ Bền vững    ❌ Tạm thời       ✅ Bền vững
Performance         Tốt            Rất cao        Cực cao           Chậm (network)
Latency             ~0.5-1ms       ~0.1-0.5ms     ~0.01-0.05ms      ms-giây
Max IOPS            16,000         256,000        Hàng triệu        N/A
Durability          99.8%          99.999%        Mất khi stop      99.999999999%
AZ restriction      Có             Có             Theo instance     Không (global)
Snapshot            ✅             ✅             ❌                ✅ (native)
Cost                Vừa            Cao            Miễn phí (bundled) Thấp/GB
Use case            General        Critical DB    Cache/Scratch     Object storage
```

---

## Best Practices

### AMI Best Practices

```
1. Đặt tên rõ ràng với version và ngày: "myapp-v2.3.1-20260514"
2. Tag đầy đủ: Name, Version, Environment, Owner, Project
3. Xóa AMI và snapshot cũ định kỳ (tốn tiền)
4. Encrypt AMI trong production
5. Dùng AMI Builder (Packer, EC2 Image Builder) thay vì tạo thủ công
6. Kiểm tra AMI mới trước khi dùng trong production ASG
7. Copy AMI sang DR region
```

### EBS Best Practices

```
1. Dùng gp3 làm default (không dùng gp2)
2. Chỉ dùng io2 khi thực sự cần > 16,000 IOPS hoặc < 1ms latency
3. Enable EBS encryption by default ở account level
4. Dùng DLM để tự động backup
5. Monitor với CloudWatch: VolumeReadOps, VolumeWriteOps, VolumeThroughput
6. Delete unattached volumes (lãng phí tiền)
7. Dùng gp3 với IOPS/throughput tùy chỉnh thay vì io1 (rẻ hơn)
```

```bash
# Bật EBS encryption by default toàn account
aws ec2 enable-ebs-encryption-by-default --region us-east-1

# Kiểm tra unattached volumes (lãng phí)
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,AvailabilityZone,CreateTime]' \
  --output table
```

---

## Câu Hỏi Phỏng Vấn

**Q: EBS Snapshot incremental hoạt động thế nào?**
> Snapshot đầu tiên là full copy. Mỗi snapshot tiếp theo chỉ lưu các blocks đã thay đổi kể từ snapshot trước. Tuy nhiên, mỗi snapshot là độc lập — bạn có thể xóa snapshot giữa mà không ảnh hưởng đến snapshot sau (AWS tự động consolidate dữ liệu). Chi phí chỉ tính cho dữ liệu duy nhất trong mỗi snapshot.

**Q: Tại sao nên dùng gp3 thay vì gp2?**
> gp3 tách biệt IOPS và throughput khỏi dung lượng. gp2 buộc bạn tăng size để có thêm IOPS (3 IOPS/GB, tối đa 16,000 IOPS cần 5,334 GB). gp3 cho 3,000 IOPS mặc định bất kể size, và có thể tăng lên 16,000 IOPS riêng lẻ. Kết quả: gp3 thường rẻ hơn gp2 20-30% cho hiệu năng tương đương.

**Q: Khi nào dùng io2 thay vì gp3?**
> Dùng io2 khi: (1) Cần > 16,000 IOPS (giới hạn gp3), (2) Cần latency sub-millisecond nhất quán (Oracle DB, SAP HANA), (3) Cần EBS Multi-Attach cho cluster database, (4) Cần durability 99.999% (thay vì 99.8% của gp3).

**Q: Instance Store có thể dùng cho database không?**
> Có thể, nếu database tự replicate dữ liệu (như Cassandra, MongoDB replica set, hoặc Kafka với replication factor ≥ 3). Khi instance stop/terminate, dữ liệu mất nhưng replica ở instance khác vẫn còn. Lợi ích: latency cực thấp và IOPS cao hơn EBS nhiều lần. Cần hiểu rõ replication model trước khi dùng.

---

## Liên Kết

- [← Instance Types](./1-instance-types.md)
- [Tiếp theo: Security & Key Pairs →](./3-security-keypairs.md)
- [AWS EBS Documentation](https://docs.aws.amazon.com/ebs/)
- [EBS Volume Types Comparison](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
