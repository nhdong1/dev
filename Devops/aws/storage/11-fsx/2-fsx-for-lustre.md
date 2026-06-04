# FSx for Lustre — Hệ Thống Tệp Hiệu Suất Cao Cho HPC và ML

> FSx for Lustre cung cấp hệ thống tệp song song (parallel file system) hiệu suất cực cao, được thiết kế cho HPC — High Performance Computing — Điện Toán Hiệu Suất Cao, Machine Learning — Học Máy, và các workload đòi hỏi băng thông hàng trăm GB/s với độ trễ sub-millisecond.

---

## Mục Lục

1. [Lustre là gì?](#lustre-là-gì)
2. [Kiến Trúc Lustre](#kiến-trúc-lustre)
3. [Deployment Types](#deployment-types)
4. [Tích Hợp Với S3](#tích-hợp-với-s3)
5. [Hiệu Suất](#hiệu-suất)
6. [Kết Nối và Mounting](#kết-nối-và-mounting)
7. [Chi Phí](#chi-phí)
8. [Use Cases Thực Tế](#use-cases-thực-tế)
9. [Điểm Kiểm Tra Phỏng Vấn](#điểm-kiểm-tra-phỏng-vấn)

---

## Lustre là gì?

**Lustre** (Linux + cluster) là hệ thống tệp song song mã nguồn mở, ban đầu được phát triển cho các siêu máy tính. Amazon FSx for Lustre là phiên bản được quản lý hoàn toàn bởi AWS.

### Tại Sao Lustre Khác Biệt?

```
Hệ thống tệp thông thường (NFS, SMB):
Client → Server → Đĩa
→ Bottleneck (thắt cổ chai) tại Server
→ Throughput giới hạn bởi một server

Lustre (song song):
          ┌─→ OSS-1 → Đĩa 1 ─┐
Client ───┼─→ OSS-2 → Đĩa 2 ─┼─→ Dữ liệu được phân mảnh
          └─→ OSS-3 → Đĩa 3 ─┘   và đọc/ghi song song
→ Throughput tỉ lệ thuận với số OSS
→ Có thể đạt hàng trăm GB/s
```

**OSS** — Object Storage Server — Máy Chủ Lưu Trữ Đối Tượng: Các server lưu trữ thực tế trong cụm Lustre.

### Chỉ Dùng Cho Linux

FSx for Lustre yêu cầu **Lustre client** được cài trên máy. Chỉ hỗ trợ Linux. Không hỗ trợ Windows.

---

## Kiến Trúc Lustre

### Các Thành Phần

```
FSx for Lustre Cluster (Cụm FSx Lustre):
├── MDT — Metadata Target — Đích Siêu Dữ Liệu
│   └── Lưu trữ: Tên file, quyền, timestamps, vị trí data
│       (Không lưu nội dung file)
└── OST — Object Storage Target — Đích Lưu Trữ Đối Tượng
    ├── OST-1: Lưu block 0-N của tất cả files
    ├── OST-2: Lưu block N+1-M của tất cả files
    └── OST-N: ...
```

### Striping — Phân Mảnh Dữ Liệu

**Striping** là kỹ thuật chia file thành nhiều mảnh và lưu trên nhiều OST song song.

```
File lớn (100 GB):
┌────────────────────────────────────┐
│ stripe 1 │ stripe 2 │ stripe 3 │  │
│  (OST-1) │  (OST-2) │  (OST-3) │  │
└────────────────────────────────────┘

Đọc file:
→ OST-1, OST-2, OST-3 đọc song song
→ Tốc độ = tổng băng thông của 3 OST
```

```bash
# Xem cấu hình stripe của file
lfs getstripe /mnt/fsx/myfile.dat

# Tạo file với stripe width 4
lfs setstripe -c 4 /mnt/fsx/bigfile.dat

# Tạo thư mục với mặc định stripe 8
lfs setstripe -c 8 /mnt/fsx/training-data/
```

---

## Deployment Types

### 1. Scratch File Systems (Hệ Thống Tệp Tạm Thời)

```
Đặc điểm:
├── Không nhân bản dữ liệu
├── Không tồn tại lâu dài — mất khi instance xóa
├── Throughput cao nhất (không overhead nhân bản)
└── Rẻ nhất

Dùng khi:
├── Xử lý dữ liệu tạm thời (intermediate results)
├── Transform dữ liệu rồi ghi kết quả về S3
└── Thời gian xử lý ngắn (giờ đến ngày)
```

**Scratch 1**: Baseline throughput
**Scratch 2**: Throughput cao hơn 6x so với Scratch 1, redundancy tốt hơn

### 2. Persistent File Systems (Hệ Thống Tệp Bền Vững)

```
Đặc điểm:
├── Dữ liệu được nhân bản trong cùng AZ
├── Tồn tại dài hạn
├── Tự động thay thế server lỗi
└── Phù hợp workload chạy dài

Dùng khi:
├── Training set được dùng lặp lại nhiều lần
├── Shared storage cho nhiều compute cluster
└── Cần data persistence khi EC2 instances bị terminate
```

### So Sánh Deployment Types

| Tiêu Chí | Scratch 1 | Scratch 2 | Persistent 1 | Persistent 2 |
|---------|-----------|-----------|--------------|--------------|
| Durability (Độ bền) | Thấp | Trung bình | Cao | Cao nhất |
| Throughput/TB | 200 MB/s | 500 MB/s | 50/100/200 MB/s | 125/250/500/1000 MB/s |
| Giá tương đối | Thấp nhất | Trung bình | Trung bình | Cao nhất |
| Use case | Temporary | Short-term | Long-term | Long-term/HA |

---

## Tích Hợp Với S3

Đây là tính năng độc đáo nhất của FSx for Lustre — native S3 integration (Tích hợp S3 gốc).

### Lazy Loading — Tải Dữ Liệu Theo Yêu Cầu

```
S3 Bucket                    FSx for Lustre
┌────────────────┐           ┌────────────────┐
│ dataset/       │           │ dataset/       │
│ ├── file1.csv  │           │ ├── file1.csv  │ ← Chỉ metadata
│ ├── file2.csv  │           │ ├── file2.csv  │ ← Chỉ metadata
│ └── file3.csv  │           │ └── file3.csv  │ ← Chỉ metadata
└────────────────┘           └────────────────┘

Khi process đọc file1.csv:
→ FSx tự động tải từ S3 về local storage
→ Những lần đọc tiếp theo: serve từ local (nhanh hơn)
→ Các file chưa đọc vẫn chỉ là metadata
```

### Data Repository Association — Liên Kết Kho Dữ Liệu

```bash
# Tạo FSx Lustre với S3 backend
aws fsx create-file-system \
    --file-system-type LUSTRE \
    --storage-capacity 1200 \
    --subnet-ids subnet-12345 \
    --lustre-configuration \
        ImportPath=s3://my-training-data/dataset/, \
        ExportPath=s3://my-results/output/, \
        ImportedFileChunkSize=1024
```

**ImportPath**: S3 prefix dùng làm nguồn dữ liệu
**ExportPath**: S3 prefix để ghi kết quả về
**ImportedFileChunkSize**: Kích thước mảnh khi import (MB)

### Export Kết Quả Về S3

```bash
# Sau khi training xong, export kết quả về S3
nohup find /mnt/fsx/output/ -type f \
    | xargs -P 8 -I {} \
    aws s3 cp {} s3://my-results/ &

# Hoặc dùng Lustre HSM (Hierarchical Storage Management)
lfs hsm_archive /mnt/fsx/output/model.pt
```

**HSM — Hierarchical Storage Management — Quản Lý Lưu Trữ Phân Cấp**: Tự động di chuyển file từ Lustre về S3 khi không cần truy cập nữa.

### Auto Import và Auto Export

```
Auto Import Policy (Chính Sách Tự Động Nhập):
├── NEW: Import files mới tạo trong S3
├── CHANGED: Import khi file S3 thay đổi
└── DELETED: Xóa khi file bị xóa khỏi S3

Auto Export Policy (Chính Sách Tự Động Xuất):
└── NEW_CHANGED_DELETED: Tự động đồng bộ về S3
```

---

## Hiệu Suất

### Số Liệu Thực Tế

| Metric | Scratch 2 | Persistent 2 (1000 MB/s) |
|--------|-----------|--------------------------|
| Throughput tối đa | 500 MB/s / TiB | 1.000 MB/s / TiB |
| IOPS (đọc) | Hàng triệu | Hàng triệu |
| Latency (độ trễ) | Sub-millisecond | Sub-millisecond |
| Dung lượng tối đa | Petabytes | Petabytes |

### Benchmark Thực Tế (ML Training)

```
Training dataset: 1 TB ảnh (ImageNet)
Cluster: 8x p3.16xlarge (128 GPU)

EFS:          ~1.5 giờ để đọc dataset lần đầu
FSx Lustre:   ~12 phút để đọc dataset lần đầu
              → 7.5x nhanh hơn
```

### Tips Tối Ưu Hiệu Suất

```bash
# 1. Sử dụng stripe_count phù hợp
# Với file lớn (> 1GB): stripe nhiều OST
lfs setstripe -c -1 /mnt/fsx/large-files/  # -1 = tất cả OST có thể

# 2. Prefetch (Tải Trước) với lfs-preheat
lfs hsm_restore /mnt/fsx/dataset/*.csv     # Kích hoạt prefetch

# 3. Parallel IO (Đọc Ghi Song Song) — dùng với MPI
mpirun -np 128 python train.py --data /mnt/fsx/dataset/

# 4. Tránh small random IO — Lustre tối ưu cho large sequential IO
# ❌ Đọc nhiều file nhỏ (< 1 MB) → hiệu quả thấp
# ✅ Đọc file lớn (> 100 MB) → hiệu quả cao
```

---

## Kết Nối và Mounting

### Cài Đặt Lustre Client

```bash
# Amazon Linux 2
sudo amazon-linux-extras install -y lustre2.10
sudo yum install -y lustre-client

# Ubuntu 20.04
wget -O - https://fsx-lustre-client-repo-public-keys.s3.amazonaws.com/fsx-ubuntu-public-key.asc | sudo apt-key add -
sudo apt-get install lustre-client-modules-$(uname -r)
```

### Mount FSx Lustre

```bash
# Mount cơ bản
sudo mkdir -p /mnt/fsx
sudo mount -t lustre \
    -o relatime,flock \
    fs-0123456789abcdef.fsx.us-east-1.amazonaws.com@tcp:/fsx \
    /mnt/fsx

# Mount qua /etc/fstab (tự động khi khởi động)
echo "fs-0123456789abcdef.fsx.us-east-1.amazonaws.com@tcp:/fsx \
    /mnt/fsx lustre defaults,relatime,flock,_netdev 0 0" >> /etc/fstab
```

### Kiểm Tra Kết Nối

```bash
# Kiểm tra file system đã mount
df -h /mnt/fsx
# → fs-xxx: 1.2T total, 0 used, 1.2T available

# Kiểm tra throughput thực tế (benchmark nhanh)
dd if=/dev/zero of=/mnt/fsx/test-write bs=1M count=10000 \
    oflag=direct conv=fdatasync
# → 10 GB write — đo throughput thực tế
```

### Dùng Với ECS / EKS

```yaml
# EKS — Kubernetes StorageClass cho FSx Lustre
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fsx-lustre
provisioner: fsx.csi.aws.com
parameters:
  subnetId: subnet-0abcdef1234567890
  securityGroupIds: sg-0abcdef1234567890
  s3ImportPath: s3://my-training-data/
  s3ExportPath: s3://my-results/
  deploymentType: SCRATCH_2
---
# PVC — Persistent Volume Claim — Yêu Cầu Volume Bền Vững
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fsx-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: fsx-lustre
  resources:
    requests:
      storage: 1200Gi
```

---

## Chi Phí

### Bảng Giá (Tham Khảo)

| Deployment Type | Giá/GB-month |
|----------------|--------------|
| Scratch 1 | $0.14 |
| Scratch 2 | $0.14 |
| Persistent 1 (50 MB/s) | $0.145 |
| Persistent 1 (100 MB/s) | $0.19 |
| Persistent 1 (200 MB/s) | $0.28 |
| Persistent 2 (125 MB/s) | $0.19 |
| Persistent 2 (1000 MB/s) | $0.36 |

### Lưu Ý Về Chi Phí

**Minimum storage**: 1.2 TB (Scratch), 2.4 TB (Persistent)
**Tính theo GiB, không phải GB**: 1 GiB = 1.073 GB

### Ví Dụ Chi Phí ML Training

```
Training set: 5 TB, dùng 3 ngày, Scratch 2

Chi phí: 5.000 GB × $0.14 × (3/30) = $70
→ So với EFS: 5.000 GB × $0.30 × (3/30) = $150
→ FSx Lustre rẻ hơn 53% VÀ nhanh hơn 7x
```

---

## Use Cases Thực Tế

### 1. ML Training — Huấn Luyện Mô Hình Máy Học

```
Luồng làm việc:
S3 (raw data) → FSx Lustre → GPU Cluster → FSx (results) → S3 (models)

Kiến trúc:
├── S3 bucket: 10 TB training data
├── FSx Lustre (Scratch 2): 12 TB (nhỏ hơn S3 do lazy loading)
├── EC2 Spot Instances: 8x p4d.24xlarge (512 A100 GPUs)
└── S3 bucket: Lưu checkpoint và final model

Khi training chạy:
→ GPU đọc batch từ FSx Lustre (sub-ms latency)
→ Không bao giờ đọc trực tiếp S3 (latency ~100ms)
→ GPU utilization: > 95%
```

### 2. Genomics — Phân Tích Bộ Gen

```
Pipeline:
FASTQ files (S3) → FSx (processing) → VCF files (S3)

Tool: GATK — Genome Analysis Toolkit — Bộ Công Cụ Phân Tích Bộ Gen
→ Cần đọc/ghi nhiều file lớn song song
→ FSx Lustre cung cấp POSIX interface đầy đủ
→ 100x nhanh hơn đọc thẳng từ S3
```

### 3. CFD — Computational Fluid Dynamics — Động Lực Học Chất Lỏng Tính Toán

```
Mô phỏng khí động học máy bay:
├── Input: Mesh files (50 GB)
├── Solver: OpenFOAM chạy trên 256 cores
├── Output: Time steps (mỗi step 1 GB, 1000 steps = 1 TB)
└── Visualization: ParaView đọc kết quả

FSx Lustre:
→ Tất cả 256 cores đọc/ghi đồng thời
→ Throughput: 200 GB/s với Persistent 2
→ Wall time giảm 10x so với NFS
```

### 4. Financial Risk Modeling (Mô Hình Hóa Rủi Ro Tài Chính)

```
Monte Carlo simulation (Mô Phỏng Monte Carlo):
├── Input: Market data 1 TB
├── Compute: 1.000 simulation paths song song
└── Output: 500 GB risk metrics

Thực hiện:
→ FSx Lustre: latency < 1ms cho random reads
→ Kết quả trong giờ thay vì ngày
→ Dùng Spot Fleet để tối ưu chi phí
```

---

## Điểm Kiểm Tra Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: FSx Lustre khác EFS như thế nào?**
> A: FSx Lustre dùng giao thức Lustre (không phải NFS), chỉ hỗ trợ Linux, throughput hàng trăm GB/s, có S3 native integration, phù hợp HPC/ML. EFS dùng NFS, đơn giản hơn, throughput thấp hơn, serverless (trả theo dùng). FSx Lustre cần provision storage capacity trước.

**Q: Khi nào nên dùng Scratch vs Persistent?**
> A: Scratch — khi xử lý temporary data, thời gian ngắn, cần giá thấp nhất. Persistent — khi training data dùng lặp lại, shared giữa nhiều cluster, cần data survive sau khi cluster xóa.

**Q: Làm sao FSx Lustre tích hợp với S3?**
> A: Khi mount FSx với S3 ImportPath, file trong S3 hiện ra như file local nhưng chỉ là metadata. Khi đọc file, FSx tự tải từ S3 (lazy loading). Sau khi xử lý, có thể export kết quả về S3 qua ExportPath hoặc HSM commands.

**Q: Có thể dùng FSx Lustre với Kubernetes không?**
> A: Có, qua FSx CSI Driver. StorageClass provisioner là `fsx.csi.aws.com`. Pod dùng PVC với access mode `ReadWriteMany` — nhiều pods cùng mount một FSx volume.

**Q: Minimum storage của FSx Lustre là bao nhiêu?**
> A: 1.2 TB cho Scratch, 2.4 TB cho Persistent. Dung lượng phải là bội số của 1.2 TB (Scratch) hoặc 2.4 TB (Persistent).

### Bảng Tóm Tắt Nhanh

| Câu Hỏi | Trả Lời |
|---------|---------|
| Protocol | Lustre client (Linux only) |
| Throughput tối đa | Hàng trăm GB/s (tùy cấu hình) |
| Latency | Sub-millisecond |
| S3 integration | Có (native, lazy loading) |
| Windows support | Không |
| Multi-AZ | Không (single AZ) |
| Minimum storage | 1.2 TB (Scratch) |
| Giá | $0.14 – $0.36/GB-month |
| Use case chính | HPC, ML, genomics, rendering |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Trạng Thái:** ✅ Hoàn thành
