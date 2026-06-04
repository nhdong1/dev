# EBS vs Instance Store — Lưu Trữ Bền Vững vs Lưu Trữ Tạm Thời

> Đây là câu hỏi phỏng vấn kinh điển. EBS (Elastic Block Store — Lưu Trữ Khối Linh Hoạt) là **persistent storage** (lưu trữ bền vững) tồn tại độc lập với EC2 instance, trong khi Instance Store (Lưu Trữ Đính Kèm Instance) là **ephemeral storage** (lưu trữ tạm thời) gắn liền vật lý với host server — dữ liệu mất khi instance stop hoặc terminate.

---

## 🔍 So Sánh Tổng Quan

| Đặc Điểm | EBS | Instance Store |
|----------|-----|----------------|
| **Loại** | Network-attached (gắn qua mạng) | Physically attached (gắn vật lý) |
| **Persistence** | Bền vững — tồn tại khi stop/restart | Tạm thời — mất khi stop/terminate |
| **Latency** | < 1ms (SSD), 2-5ms (HDD) | **< 0.1ms** (cực kỳ thấp) |
| **IOPS** | tối đa 256.000 (io2) | **Hàng triệu** (NVMe local SSD) |
| **Throughput** | tối đa 4.000 MB/s (io2) | **Hàng chục GB/s** |
| **Dung lượng** | tối đa 64TB (io2) | Tùy instance type (thường 1–60TB) |
| **Resize** | Có thể tăng khi đang chạy | Không thể thay đổi |
| **Snapshot** | ✅ Hỗ trợ | ❌ Không hỗ trợ |
| **Encryption** | ✅ AES-256 với KMS | ❌ Không hỗ trợ natively |
| **Backup** | ✅ Dễ dàng qua snapshot | ❌ Phải tự backup lên S3/EBS |
| **Giá** | Tính riêng theo GB và IOPS | Bao gồm trong giá instance |
| **Boot volume** | ✅ Hỗ trợ (mặc định) | ❌ Không hỗ trợ |

---

## 🧱 Kiến Trúc So Sánh

### EBS — Network-Attached Block Storage

```
┌─────────────────────────────────────────────────────────┐
│                  AWS Data Center                         │
│                                                         │
│  ┌───────────────┐         ┌───────────────────────┐   │
│  │  EC2 Instance │         │     EBS Volume         │   │
│  │  (Host Server)│ ◄──────►│  (Separate Hardware)   │   │
│  │               │  EBS    │  Replicated within AZ  │   │
│  └───────────────┘ Network └───────────────────────┘   │
│                                                         │
│  EC2 stop → EBS vẫn tồn tại                             │
│  EC2 terminate → EBS có thể giữ (tùy delete-on-         │
│                  termination setting)                    │
└─────────────────────────────────────────────────────────┘
```

### Instance Store — Physically Attached

```
┌─────────────────────────────────────────────────────────┐
│                  AWS Data Center                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              EC2 Host Server                     │   │
│  │                                                  │   │
│  │  ┌───────────────┐  ┌──────────────────────┐   │   │
│  │  │  EC2 Instance │  │   Instance Store      │   │   │
│  │  │  (vCPU, RAM)  │◄─►│   NVMe SSD local    │   │   │
│  │  │               │  │   Direct PCIe path   │   │   │
│  │  └───────────────┘  └──────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  EC2 stop → Instance Store DỮ LIỆU BỊ XÓA              │
│  EC2 hibernate → Instance Store DỮ LIỆU BỊ XÓA          │
│  EC2 terminate → Instance Store DỮ LIỆU BỊ XÓA          │
│  EC2 reboot → Instance Store VẪN CÒN                    │
└─────────────────────────────────────────────────────────┘
```

---

## ⚡ Hiệu Suất Instance Store

### Instance Types Có Instance Store NVMe

```
Instance Family    | Instance Store Size | IOPS         | Throughput
──────────────────────────────────────────────────────────────────────
i3.large           | 1 × 475 GB NVMe     | ~100.000     | ~3 GB/s
i3.xlarge          | 1 × 950 GB NVMe     | ~200.000     | ~6 GB/s
i3.4xlarge         | 2 × 1.9 TB NVMe     | ~800.000     | ~12 GB/s
i3.16xlarge        | 8 × 1.9 TB NVMe     | ~3.300.000   | ~48 GB/s
i3en.24xlarge      | 8 × 7.5 TB NVMe     | ~7.500.000   | ~100 GB/s
d3.xlarge          | 3 × 2 TB HDD        | ~800         | ~285 MB/s (HDD)
r5d.large          | 1 × 75 GB NVMe      | ~200.000     | ~6 GB/s (temp)
m5d.large          | 1 × 75 GB NVMe      | ~200.000     | ~6 GB/s (temp)
```

### So Sánh Thực Tế

```
Bài test fio random read 4KB:

EBS io2 Block Express (256.000 IOPS):
  → IOPS: 250.000
  → Latency P99: 0.8ms

Instance Store i3.8xlarge (i3 NVMe):
  → IOPS: 1.650.000
  → Latency P99: 0.08ms (nhanh hơn 10 lần!)
  
→ Instance Store thắng tuyệt đối về raw performance
→ Nhưng mất dữ liệu khi stop/terminate
```

---

## 📋 Khi Nào Dùng Instance Store?

### Use Cases Phù Hợp (Data Ephemeral OK)

#### 1. Cache Layer (Tầng Bộ Nhớ Đệm)

```
Redis Cluster / Memcached:
  → Data in-memory, cache có thể rebuild từ database
  → Instance Store lưu snapshot định kỳ để warm-up nhanh
  → Mất cache → cold start, performance giảm tạm thời, không mất data nghiệp vụ
  
  Pattern: Redis persistence (AOF/RDB) lưu vào EBS
           Working set → Instance Store (cực nhanh)
```

#### 2. Temporary Processing (Xử Lý Tạm Thời)

```
Apache Spark / Hadoop:
  → Shuffle data (dữ liệu xáo trộn): tạm thời trong job execution
  → Input từ S3, output lên S3
  → Instance Store cho intermediate data → tăng job performance đáng kể
  
  i3 instances phổ biến cho EMR (Elastic MapReduce) clusters
```

#### 3. High-Performance Database Buffer

```
Elasticsearch / OpenSearch:
  → Instance Store cho warm data index
  → Replica shards có thể rebuild từ primary
  → Snapshot lên S3 định kỳ để DR

MySQL với replicated setup:
  → Instance Store cho replica slaves
  → Nếu slave crash → rebuild từ master nhanh hơn restore snapshot
```

#### 4. Rendering / HPC Jobs (Điện Toán Hiệu Suất Cao)

```
Video rendering, genomic analysis:
  → Job stateless, output lưu lên S3
  → Instance Store cho scratch space (không gian làm việc tạm)
  → Tốc độ I/O cực cao → rút ngắn thời gian job
```

---

## 🔒 Khi Nào KHÔNG Dùng Instance Store?

### Data Phải Persistent

```
❌ Database primary với data không replicate đủ:
   MySQL single instance → stop để maintenance → mất data

❌ Application state (trạng thái ứng dụng):
   Session data lưu local → user logout khi instance restart

❌ File uploads của user:
   Ảnh, documents → mất khi instance replace

❌ Audit logs, compliance records:
   Yêu cầu lưu giữ 7 năm → không thể dùng ephemeral storage

Nguyên tắc: Nếu mất dữ liệu sẽ ảnh hưởng nghiệp vụ → KHÔNG dùng Instance Store
```

---

## 🏗️ Architecture Patterns (Kiểu Kiến Trúc)

### Pattern 1: Instance Store + EBS Kết Hợp

```
                    ┌────────────────────────────┐
                    │      EC2 Instance (i3)     │
                    │                            │
                    │  Instance Store NVMe       │
                    │  /tmp, /var/cache          │
                    │  Temporary working files   │
                    │                            │
                    │  EBS gp3 (Root)            │
                    │  OS, Application binaries  │
                    │                            │
                    │  EBS io2 (Data)            │
                    │  Persistent database files │
                    └────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  S3 Bucket          │
                    │  Backups, archives  │
                    │  Final outputs      │
                    └─────────────────────┘
```

### Pattern 2: Stateless App Với Instance Store Cache

```
                         ┌──────────┐
                         │    ALB   │
                         │  (Load   │
                         │ Balancer)│
                         └────┬─────┘
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ EC2 + IS     │  │ EC2 + IS     │  │ EC2 + IS     │
    │ Local cache  │  │ Local cache  │  │ Local cache  │
    │ (ephemeral)  │  │ (ephemeral)  │  │ (ephemeral)  │
    └──────────────┘  └──────────────┘  └──────────────┘
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                    ┌──────────────────┐
                    │  RDS Database    │
                    │  (Source of      │
                    │   Truth)         │
                    └──────────────────┘

→ Cache miss → fetch từ RDS
→ Instance replace → cache rebuild từ RDS (cold start tạm thời)
→ Acceptable pattern vì mất cache không mất data
```

---

## 💡 Xử Lý Instance Store Với Auto Scaling

Khi dùng Auto Scaling Group (Nhóm Tự Động Mở Rộng) với Instance Store:

```bash
# User Data script để initialize Instance Store khi instance launch
#!/bin/bash

# Format và mount instance store
mkfs.xfs /dev/nvme1n1
mkdir -p /var/cache/app
mount /dev/nvme1n1 /var/cache/app

# Warm-up cache từ S3 (nếu có warm data)
aws s3 cp s3://my-cache-warmup-bucket/snapshot/ /var/cache/app/ --recursive

# Hoặc chạy cache warming script
/opt/scripts/warm-cache.sh

echo "Instance Store initialized and warmed"
```

---

## 📊 Decision Framework — Khung Quyết Định

```
Dữ liệu có thể mất không cần lo?
├── Có (temporary/cache/scratch) → Instance Store
│     ↓
│   Cần hiệu suất tối đa?
│   ├── Có → i3/i3en instances (NVMe SSD)
│   └── Không → m5d/r5d (nhỏ hơn, rẻ hơn)
│
└── Không (business data) → EBS
      ↓
    Workload pattern?
    ├── Random I/O (database) → gp3 hoặc io2
    ├── Sequential (analytics) → st1
    └── Cold archive → sc1
```

---

## ⚠️ Rủi Ro Instance Store Và Cách Giảm Thiểu

### Rủi Ro: Mất Dữ Liệu Khi Host Failure

```
Host server vật lý của EC2 bị lỗi:
  → AWS tự động migrate EC2 sang host khác
  → Instance Store data BỊ MẤT HOÀN TOÀN
  → EBS data vẫn an toàn (lưu riêng biệt)

Giảm thiểu:
  → Periodic backup lên S3: aws s3 sync /instance-store/ s3://backup/
  → Application-level replication (Redis RDB/AOF)
  → Cluster replication (Elasticsearch replicas)
  → Checkpoint pattern cho Spark/ML jobs
```

### Rủi Ro: Spot Instance Interruption (Gián Đoạn Spot Instance)

```
Spot Instance bị AWS thu hồi → 2 phút cảnh báo → terminate
  → Instance Store mất toàn bộ
  → Không có cơ hội đồng bộ kịp nếu không chuẩn bị

Giảm thiểu:
  → Spot interruption handler: listen CloudWatch event → trigger sync lên S3
  → Dùng Spot + On-Demand mixed trong Auto Scaling Group
  → Stateless design: không lưu critical data trên Spot
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt cơ bản nhất giữa EBS và Instance Store?**
> EBS là network-attached persistent storage — dữ liệu tồn tại khi stop/terminate EC2. Instance Store là physically-attached ephemeral storage — dữ liệu mất khi instance stop, terminate hoặc host failure. EBS phù hợp hầu hết workload; Instance Store chỉ dùng cho data tạm thời như cache, shuffle data, scratch space.

**Q: Khi nào bạn chọn Instance Store thay vì EBS?**
> Khi cần hiệu suất I/O cực cao (> 1 triệu IOPS) và chấp nhận mất dữ liệu tạm thời. Use cases: Redis/Memcached cache, Spark shuffle data, Elasticsearch warm storage, rendering scratch files. Luôn kết hợp với backup mechanism lên S3.

**Q: Instance Store có thể dùng làm boot volume không?**
> Không. AWS yêu cầu boot volume phải là EBS để đảm bảo tính bền vững. OS cần tồn tại qua stop/start cycles. Instance Store chỉ dùng cho data volumes (non-root).

**Q: Nếu EC2 reboot, Instance Store có mất dữ liệu không?**
> Không — đây là điểm hay bị nhầm lẫn. Reboot (khởi động lại) không làm mất Instance Store. Chỉ khi stop, terminate, hoặc host hardware failure thì Instance Store mới mất. Hibernation cũng làm mất Instance Store.

**Q: Làm sao bảo vệ dữ liệu quan trọng trên Instance Store?**
> Không nên lưu dữ liệu quan trọng trên Instance Store. Nếu bắt buộc, dùng: (1) Application-level replication; (2) Periodic sync lên S3; (3) Cluster replication (nhiều node, mỗi node 1 replica). Best practice: thiết kế stateless — Instance Store chỉ là performance cache.

---

## 🔗 Điều Hướng

- **Trước:** [5-encryption-with-kms.md](./5-encryption-with-kms.md) — Encryption With KMS
- **Tiếp theo:** [../04-efs/README.md](../04-efs/README.md) — EFS — Elastic File System
- **Liên quan:** [../04-efs/5-efs-vs-ebs-vs-s3.md](../04-efs/5-efs-vs-ebs-vs-s3.md) — Bảng So Sánh EFS vs EBS vs S3

---

**Cập Nhật Lần Cuối:** 2026-05-15
