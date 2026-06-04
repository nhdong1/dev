# EBS Multi-Attach — Gắn Kết Nhiều EC2 Cùng Lúc

> EBS Multi-Attach (Gắn Kết Nhiều) cho phép một EBS volume io1 hoặc io2 được gắn đồng thời vào tối đa **16 EC2 Nitro instances** trong **cùng một Availability Zone**. Đây là tính năng nâng cao cho các cluster application (ứng dụng cụm) yêu cầu shared block storage.

---

## 🧠 Multi-Attach Là Gì?

### So Sánh Với Chế Độ Thông Thường

```
Chế Độ Thông Thường (Single Attach):
┌──────────┐     ┌──────────────┐
│ EC2 - A  │────►│  EBS Volume  │
└──────────┘     └──────────────┘
   Độc quyền toàn bộ volume

Multi-Attach:
┌──────────┐     
│ EC2 - A  │────►┐
└──────────┘     │  ┌──────────────┐
┌──────────┐     ├─►│  EBS Volume  │
│ EC2 - B  │────►│  │ (io1/io2)    │
└──────────┘     │  └──────────────┘
┌──────────┐     │
│ EC2 - C  │────►┘
└──────────┘
   Tối đa 16 EC2 trong cùng AZ
```

### Giới Hạn Quan Trọng

```
✅ Hỗ trợ:   io1, io2, io2 Block Express
❌ Không hỗ trợ: gp2, gp3, st1, sc1

✅ Tối đa:   16 Nitro-based EC2 instances
✅ Phạm vi:  Cùng 1 Availability Zone (us-east-1a)
❌ Không thể: Across AZ, across Region

✅ OS hỗ trợ: Linux (cluster-aware filesystem bắt buộc)
❌ Windows:  Không hỗ trợ đầy đủ (cần cấu hình đặc biệt)
```

---

## ⚠️ Cảnh Báo Quan Trọng: Quản Lý Đồng Thời

Multi-Attach **không** cung cấp cơ chế đồng bộ hay locking (khóa) tự động giữa các instances. Mỗi instance đều có quyền đọc/ghi đồng thời.

### Vấn Đề Nếu Dùng Sai

```
EC2-A: write "Hello" vào block 100
EC2-B: đồng thời write "World" vào block 100

Kết quả: DATA CORRUPTION (Hỏng Dữ Liệu) nếu không có locking!

→ Filesystem thông thường (ext4, xfs, NTFS) KHÔNG an toàn với Multi-Attach
→ Phải dùng cluster-aware filesystem hoặc ứng dụng tự quản lý locking
```

### Giải Pháp: Cluster-Aware Filesystem

```
Cluster-Aware Filesystems (Hệ Thống Tệp Hỗ Trợ Cụm):
  - GFS2 (Global File System 2) — Linux
  - OCFS2 (Oracle Cluster File System 2)
  - GPFS (IBM General Parallel File System)
  - Oracle ASM (Automatic Storage Management)

Ứng dụng tự quản lý locking:
  - Oracle RAC (Real Application Clusters — Cụm Ứng Dụng Thực)
  - Custom distributed systems với explicit locking
```

---

## 🏗️ Kiến Trúc Điển Hình

### Pattern 1: Oracle RAC

```
                    ┌─────────────────────────┐
                    │   Multi-Attach io2 Vol  │
                    │   500GB, 40.000 IOPS    │
                    │   Oracle ASM managed    │
                    └──────────┬──────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │ EC2-A    │     │ EC2-B    │     │ EC2-C    │
       │ Oracle   │     │ Oracle   │     │ Oracle   │
       │ RAC Node1│     │ RAC Node2│     │ RAC Node3│
       └──────────┘     └──────────┘     └──────────┘
              │                │                │
              └────────────────┴────────────────┘
                    Private Network (Mạng Nội Bộ)
                    Oracle Cache Fusion Protocol
```

### Pattern 2: High Availability Failover

```
Active-Passive với shared storage:

┌──────────┐
│ EC2-A    │ ←── Active (đang write)
│ Primary  │
└──────────┘
     │        ┌──────────────────┐
     ├────────►│ Multi-Attach     │
     │        │ io2 Volume       │
┌────▼─────┐  └──────────────────┘
│ EC2-B    │
│ Standby  │ ←── Passive (chờ failover)
└──────────┘

Khi EC2-A fail → EC2-B mount volume và trở thành Primary
→ Nhanh hơn snapshot restore (không cần copy data)
→ RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) < 30 giây
```

---

## ⚙️ Cấu Hình Multi-Attach

### Bật Multi-Attach Khi Tạo Volume

```bash
# Tạo io2 volume với Multi-Attach enabled
aws ec2 create-volume \
  --volume-type io2 \
  --size 100 \
  --iops 10000 \
  --availability-zone us-east-1a \
  --multi-attach-enabled \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[
    {Key=Name,Value=shared-cluster-volume},
    {Key=MultiAttach,Value=enabled}
  ]'
```

### Attach Volume Vào Nhiều EC2

```bash
# Gắn vào EC2 thứ nhất
aws ec2 attach-volume \
  --volume-id vol-multi-attach-xxx \
  --instance-id i-instance-A \
  --device /dev/sdf

# Gắn vào EC2 thứ hai (cùng volume!)
aws ec2 attach-volume \
  --volume-id vol-multi-attach-xxx \
  --instance-id i-instance-B \
  --device /dev/sdf

# Kiểm tra trạng thái
aws ec2 describe-volumes \
  --volume-ids vol-multi-attach-xxx \
  --query "Volumes[*].Attachments[*].[InstanceId,State,Device]" \
  --output table
```

### Cấu Hình GFS2 Cluster Filesystem (Linux)

```bash
# Trên tất cả node trong cluster, cài đặt
sudo yum install -y gfs2-utils dlm-corosync

# Tạo GFS2 filesystem trên shared volume
sudo mkfs.gfs2 \
  -p lock_dlm \
  -t cluster1:shared-vol \
  -j 3 \  # 3 journals cho 3 nodes
  /dev/nvme1n1

# Mount trên mỗi node
sudo mount -t gfs2 /dev/nvme1n1 /mnt/shared

# Filesystem bây giờ tự động quản lý concurrent access
```

---

## 📊 Multi-Attach vs Các Giải Pháp Thay Thế

| Tiêu Chí | EBS Multi-Attach | EFS | FSx for Lustre | S3 |
|----------|-----------------|-----|----------------|-----|
| **Protocol** | Block (iSCSI-like) | NFS | Lustre | HTTP |
| **Latency** | < 1ms | ~1-5ms | < 1ms | 10-100ms |
| **IOPS** | 256.000 | Không đo | Rất cao | Không áp dụng |
| **Multi-AZ** | ❌ | ✅ | ✅ | ✅ |
| **Concurrent writes** | Cần cluster FS | ✅ Native | ✅ Native | ✅ |
| **Use case** | Oracle RAC, cluster DB | Shared files | HPC/ML | Object storage |
| **Phức tạp** | Cao (cần cluster FS) | Thấp | Trung bình | Thấp |

---

## 💰 Chi Phí

Multi-Attach không tính thêm phí so với io1/io2 thông thường:
```
Chi phí = io2 pricing (volume + IOPS)
       = $0.125/GB + $0.065/IOPS (cho 32.000 IOPS đầu)

Lưu ý:
  - Mỗi EC2 attach đều tiêu thụ IOPS từ cùng 1 volume
  - Nếu 3 EC2 đều write nặng → tranh chấp IOPS
  - Cần provision IOPS cao đủ cho tổng nhu cầu của tất cả instances
```

---

## ✅ Khi Nào Dùng Multi-Attach?

### Nên Dùng

- **Oracle RAC** (Real Application Clusters — Cụm Ứng Dụng Thực) — use case chính thức
- **High-availability failover** không muốn downtime snapshot restore
- **Custom cluster database** với locking mechanism riêng
- **SAP** cluster configurations với shared block storage

### Không Nên Dùng

- Thay thế cho EFS (quá phức tạp, cần cluster FS)
- Shared storage đơn giản giữa nhiều ứng dụng → dùng EFS
- Dữ liệu đọc chung, ghi riêng → dùng snapshots hoặc AMI
- Multi-AZ workload → EFS hoặc shared database service

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: EBS Multi-Attach là gì và khi nào dùng?**
> Multi-Attach cho phép một io1/io2 volume gắn vào tối đa 16 EC2 Nitro instances trong cùng AZ đồng thời. Use case chính là Oracle RAC và HA failover clusters. Quan trọng: phải dùng cluster-aware filesystem như GFS2 hoặc Oracle ASM để tránh data corruption.

**Q: Tại sao Multi-Attach chỉ hỗ trợ io1/io2?**
> io1/io2 được thiết kế cho IOPS cao và workload critical. Việc shared block storage yêu cầu IOPS nhất quán và high durability — đây là điểm mạnh của Provisioned IOPS. gp3 không có Multi-Attach vì use case của nó không cần shared block access.

**Q: Multi-Attach có tự xử lý concurrent writes không?**
> Không. EBS chỉ cung cấp shared block access. Quản lý concurrent writes là trách nhiệm của tầng trên — cluster filesystem (GFS2, OCFS2) hoặc ứng dụng (Oracle ASM, RAC). Nếu dùng ext4/xfs bình thường với Multi-Attach sẽ gây data corruption.

**Q: Multi-Attach khác EFS như thế nào cho shared storage?**
> Multi-Attach: block storage, latency < 1ms, cần cluster FS, chỉ cùng AZ. EFS: NFS file storage, tự quản lý concurrent access, multi-AZ, dễ dùng hơn nhiều. Chọn Multi-Attach khi cần block-level performance; chọn EFS cho shared files đơn giản.

---

## 🔗 Điều Hướng

- **Trước:** [2-snapshots-and-lifecycle.md](./2-snapshots-and-lifecycle.md) — Snapshots & DLM
- **Tiếp theo:** [4-performance-tuning.md](./4-performance-tuning.md) — Performance Tuning
- **Liên quan:** [1-volume-types.md](./1-volume-types.md) — Volume Types (io1/io2 details)

---

**Cập Nhật Lần Cuối:** 2026-05-15
