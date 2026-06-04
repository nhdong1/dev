# EFS vs EBS vs S3 — Bảng So Sánh Đầy Đủ

> Ba dịch vụ lưu trữ chính của AWS phục vụ các nhu cầu rất khác nhau. Hiểu rõ sự khác biệt giúp chọn đúng công cụ cho từng use case — đây là câu hỏi thường gặp trong phỏng vấn AWS.

---

## 📊 Bảng So Sánh Chính

| Tiêu Chí | Amazon EFS | Amazon EBS | Amazon S3 |
|----------|-----------|-----------|----------|
| **Loại lưu trữ** | File Storage (Lưu trữ Tệp) | Block Storage (Lưu trữ Khối) | Object Storage (Lưu trữ Đối tượng) |
| **Giao thức** | NFS v4.1 (Network File System) | iSCSI (block device) | HTTP REST API |
| **Truy cập** | Nhiều instances đồng thời | 1 instance (Multi-Attach chỉ io1/io2) | Bất kỳ client nào qua Internet/VPC |
| **Hệ điều hành** | Linux only | Linux & Windows | Tất cả (không cần OS) |
| **Dung lượng** | Tự động scale (petabytes) | Cố định — phải provision (max 64 TiB) | Vô hạn (unlimited) |
| **Tính bền vững** | 99.999999999% (đa-AZ) | 99.999% trong một AZ | 99.999999999% (đa-AZ) |
| **Tính sẵn sàng** | 99.99% | 99.99% (trong AZ) | 99.99% |
| **Latency** | ~1ms (General Purpose) | ~0.1–1ms (SSD volumes) | ~10–100ms (HTTP) |
| **Throughput** | Lên đến 3 GiB/s | Lên đến 4 GiB/s (io2 Block Express) | Phụ thuộc prefix, multipart |
| **Chi phí** | ~$0.30/GiB | ~$0.08–0.10/GiB (gp3) | ~$0.023/GiB (Standard) |
| **Serverless** | ✅ (không cần provision) | ❌ (phải provision volume) | ✅ (không cần provision) |

---

## 🔍 So Sánh Chi Tiết Theo Từng Khía Cạnh

### 1. Mô Hình Dữ Liệu (Data Model)

```
EFS:
  - Cây thư mục phân cấp (hierarchical directory tree)
  - File, thư mục, symlinks, hardlinks, permissions POSIX
  - Giống hệt một filesystem Unix
  - Path: /mnt/efs/data/logs/app.log

EBS:
  - Raw block device — thiết bị khối thô
  - Cần format với filesystem (ext4, xfs, NTFS) trước khi dùng
  - Client thấy như local disk (/dev/xvdf)
  - Sau khi format mới có thể tạo file/thư mục

S3:
  - Flat namespace — không gian phẳng (không có thư mục thật)
  - Mọi object có Key (là một string) và Value (nội dung)
  - "Thư mục" chỉ là prefix convention trong key name
  - Key: "data/logs/2026/05/16/app.log" (đây là key, không phải path)
```

### 2. Truy Cập Đồng Thời (Concurrent Access)

```
EFS:
  ✅ Hàng nghìn EC2, ECS tasks, EKS pods, Lambda đồng thời
  ✅ Shared file system — đọc/ghi concurrent native (POSIX semantics)
  ✅ File locking (khóa file) qua NFS locks
  Dùng cho: CMS, shared content, home directories

EBS:
  ❌ Mặc định: chỉ 1 instance một lúc (single-attach)
  ⚠️  Multi-Attach: chỉ io1/io2, chỉ trong cùng AZ, Linux only
  ⚠️  Multi-Attach yêu cầu cluster-aware filesystem (GFS2, OCFS2)
  Dùng cho: Database, boot volumes — không cần share

S3:
  ✅ Concurrent reads không giới hạn (đọc đồng thời)
  ⚠️  Concurrent writes cùng key → last-write-wins (ghi cuối thắng)
  ❌ Không phải POSIX filesystem — không có file locking
  Dùng cho: Static files, backup, data lake
```

### 3. Hiệu Suất (Performance)

```
Latency (Độ trễ):
  EBS gp3/io2:  0.1–1ms    ← Thấp nhất, gần local disk
  EFS GP mode:  ~1ms        ← Tốt, phù hợp hầu hết apps
  EFS Max I/O:  5–10ms+     ← Cho hàng nghìn clients
  S3:           10–200ms    ← HTTP overhead không thể tránh

Throughput tối đa:
  EBS io2 Block Express: 4 GiB/s, 256K IOPS
  EFS Elastic:           3 GiB/s đọc / 1 GiB/s ghi
  S3:                    Không giới hạn (phân tán theo prefix)

IOPS:
  EBS io2:  256.000 IOPS/volume
  EBS gp3:  16.000 IOPS/volume
  EFS GP:   ~35.000 IOPS tổng
  S3:       3.500 PUT/s / 5.500 GET/s per prefix
```

### 4. Chi Phí (Pricing — us-east-1, tháng 5/2026 ước tính)

| | EFS Standard | EBS gp3 | EBS io2 | S3 Standard |
|-|-------------|---------|---------|-------------|
| **Lưu trữ** | $0.30/GiB | $0.08/GiB | $0.125/GiB | $0.023/GiB |
| **IOPS** | Included | 3.000 free, $0.005/IOPS | $0.065/IOPS-month | $0.0004/1K requests |
| **Throughput** | Elastic: $0.06/GiB write | $0.040/MiB/s (>125 MiB/s) | Included | $0.09/GB transfer out |
| **Không dùng cũng trả** | Không (pay per use) | Có (trả theo provision) | Có | Không |

```
Ví dụ: 1 TiB storage, 1 tháng

EFS Standard: 1.024 × $0.30 = $307
EBS gp3:      1.024 × $0.08 = $82 (nếu provision đủ)
S3 Standard:  1.024 × $0.023 = $24
```

> **EFS đắt hơn EBS và S3** nhưng cung cấp shared access mà hai dịch vụ kia không có hoặc phức tạp hơn.

### 5. Bảo Mật (Security)

| | EFS | EBS | S3 |
|-|-----|-----|----|
| **Mã hóa at-rest** | KMS ✅ | KMS ✅ | SSE-S3, SSE-KMS, SSE-C ✅ |
| **Mã hóa in-transit** | TLS (mount helper) ✅ | Không (block protocol) N/A | HTTPS ✅ |
| **IAM Authorization** | ✅ (IAM + file system policy) | IAM (attach/detach volume) | ✅ (bucket policy + IAM) |
| **POSIX permissions** | ✅ (Unix rwx) | ✅ (sau khi format) | ❌ (không có) |
| **Access Points** | ✅ (per-app identity) | N/A | ❌ |
| **VPC Endpoint** | ✅ (Interface endpoint) | N/A (trong VPC) | ✅ (Gateway endpoint) |

### 6. Vòng Đời & Tự Động Hóa (Lifecycle)

```
EFS:
  - Intelligent-Tiering: Standard → IA tự động (N ngày không truy cập)
  - AWS Backup tích hợp
  - Không có versioning (quản lý phiên bản) tích hợp

EBS:
  - Snapshots (ảnh chụp) → lưu trong S3 (incremental)
  - DLM — Data Lifecycle Manager: tự động snapshot + retention
  - Không có auto-tiering

S3:
  - Lifecycle rules: chuyển tầng storage class tự động
  - Versioning: giữ nhiều phiên bản của object
  - Object Lock: WORM — Write Once Read Many (bất biến)
  - Replication: CRR/SRR cross-region
```

---

## 🗺️ Decision Tree — Cây Quyết Định Chọn Dịch Vụ

```
Câu hỏi 1: Bạn cần truy cập từ nhiều servers/instances đồng thời không?
│
├── KHÔNG — chỉ một instance tại một thời điểm
│   │
│   ├── Cần latency rất thấp (< 1ms) hoặc là boot volume?
│   │   └── → EBS ✅ (gp3 cho general, io2 cho database)
│   │
│   └── Lưu trữ dài hạn, backup, static files?
│       └── → S3 ✅
│
└── CÓ — nhiều instances/containers truy cập đồng thời
    │
    ├── Linux clients cần shared filesystem (POSIX)?
    │   └── → EFS ✅
    │
    ├── Windows clients cần shared filesystem (SMB)?
    │   └── → FSx for Windows File Server ✅
    │
    └── High Performance Computing (HPC), ML training?
        └── → FSx for Lustre ✅ (hoặc EFS Max I/O)
```

```
Câu hỏi 2: Dữ liệu của bạn truy cập như thế nào?
│
├── Đọc/ghi file với path (hierarchical) và cần POSIX?
│   ├── Shared → EFS
│   └── Not shared → EBS
│
├── Object/blob với key-value, HTTP API?
│   └── → S3
│
└── Database, OS boot, low-latency block I/O?
    └── → EBS
```

---

## 📋 Use Cases Điển Hình — Ai Dùng Gì?

### EFS Use Cases

```
✅ CMS — Content Management System (WordPress, Drupal)
   → Nhiều web servers chia sẻ cùng file system (PHP, media)

✅ Home Directories (thư mục home người dùng)
   → Người dùng kết nối từ nhiều servers, cần POSIX permissions

✅ Container Shared Storage (ECS/EKS)
   → Nhiều pods/tasks cần đọc cùng config files, shared cache

✅ Lambda Shared Storage
   → Lambda functions cần state lớn hơn /tmp (512MB–10GB)

✅ Lift-and-Shift Migration (Di chuyển ứng dụng on-premises)
   → Ứng dụng dùng NFS mount → chuyển lên EFS không cần sửa code
```

### EBS Use Cases

```
✅ Database Storage (MySQL, PostgreSQL, MongoDB)
   → Cần latency thấp, IOPS cao, consistent performance

✅ Boot Volumes — Ổ Đĩa Khởi Động
   → Mọi EC2 instance cần boot volume (EBS gp3 là mặc định)

✅ Application Data — Dữ liệu ứng dụng đơn instance
   → File server, local cache, transaction logs

✅ High IOPS Workloads
   → Redis, Elasticsearch khi chạy trên EC2 (io2 Block Express)
```

### S3 Use Cases

```
✅ Static Website Hosting (lưu trữ website tĩnh)
   → HTML, CSS, JS, images phục vụ qua CloudFront/URL

✅ Data Lake (hồ dữ liệu)
   → Lưu trữ raw data cho Athena, EMR, Glue phân tích

✅ Backup và Archive (sao lưu và lưu trữ)
   → Long-term retention với Glacier, tuân thủ compliance

✅ Media Storage (lưu trữ media)
   → Video, images cho applications — phục vụ qua CDN

✅ Log Aggregation (tổng hợp log)
   → Centralized logging từ nhiều services

✅ Big Data & Analytics (dữ liệu lớn)
   → Input/output cho Spark, Hadoop, EMR jobs
```

---

## ⚖️ Trade-offs Tóm Tắt

### Khi EFS Tốt Hơn EBS

| Situation | Lý Do |
|-----------|-------|
| Nhiều EC2 cần share content | EBS không share được (trừ Multi-Attach phức tạp) |
| ECS/EKS với stateful workloads | EFS native với containers |
| Không biết trước dung lượng cần | EFS tự scale, EBS phải resize thủ công |
| Lift-and-shift NFS workloads | Không cần thay đổi ứng dụng |

### Khi EBS Tốt Hơn EFS

| Situation | Lý Do |
|-----------|-------|
| Database (MySQL, Postgres) | Latency thấp hơn EFS; IOPS cao hơn |
| Boot volumes | EBS là lựa chọn duy nhất |
| Windows workloads | EFS không hỗ trợ Windows (dùng FSx) |
| Chi phí thấp hơn | EBS rẻ hơn EFS với cùng dung lượng |

### Khi S3 Tốt Hơn EFS

| Situation | Lý Do |
|-----------|-------|
| Lưu trữ dài hạn / archive | S3 Glacier rẻ hơn nhiều |
| Static content phục vụ qua HTTP | S3 + CloudFront tối ưu cho web |
| Backup destination | S3 + versioning + Object Lock |
| Big data analytics | S3 là data lake chuẩn cho Athena, EMR |
| Cross-region access | S3 accessible từ mọi nơi; EFS chỉ trong VPC |

---

## 🔧 Kết Hợp Các Dịch Vụ — Patterns Thực Tế

### Pattern 1: Web Application Stack

```
Internet → CloudFront → EC2 (EBS boot volume)
                           ↓
                        EFS (shared PHP/media files)
                           ↓
                        EBS (database data volume)
                           ↓
                        S3 (user uploads, static assets, backups)
```

### Pattern 2: Container Platform (EKS)

```
EKS Pods
├── Ephemeral storage: emptyDir (trong pod — mất khi pod xóa)
├── Shared storage: EFS PersistentVolume (ReadWriteMany)
├── DB storage: EBS PersistentVolume (ReadWriteOnce — cao IOPS)
└── Artifacts/output: S3 (qua AWS SDK)
```

### Pattern 3: ML Training Pipeline

```
Data: S3 (raw datasets, cheap storage)
  ↓ (DataSync hoặc direct read)
Training: EFS (shared checkpoints cho distributed training)
  hoặc FSx for Lustre (tích hợp S3, throughput cực cao)
  ↓
Models: S3 (artifacts, versioned model files)
  ↓
Inference: EBS (low-latency model serving)
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: "Khi nào bạn chọn EFS thay vì EBS?"**
A: Khi cần shared file system cho nhiều EC2 instances hoặc containers đồng thời. EFS là NFS-based nên nhiều client mount cùng lúc và thấy cùng dữ liệu ngay lập tức (POSIX semantics). EBS chỉ gắn vào một instance (trừ Multi-Attach phức tạp cho io1/io2).

**Q: "EFS có rẻ hơn EBS không?"**
A: Không — EFS Standard (~$0.30/GiB) đắt hơn EBS gp3 (~$0.08/GiB) khoảng 3–4 lần. Nhưng EFS không cần provision trước, tự scale, và phù hợp cho shared workloads mà EBS không thể làm. EFS cũng có Intelligent-Tiering giúp giảm chi phí xuống $0.016/GiB cho dữ liệu ít truy cập.

**Q: "S3 có thể thay thế EFS không?"**
A: Không trực tiếp — S3 là object storage với HTTP API, không có POSIX semantics. Ứng dụng cần mount filesystem, POSIX permissions, hardlinks, file locking không thể dùng S3. Nhưng với use cases phù hợp (backup, static content, data lake), S3 tốt hơn và rẻ hơn nhiều.

**Q: "Thiết kế storage cho WordPress với auto-scaling EC2 fleet"**
A: EFS cho shared web content (PHP files, themes, plugins, media) + EBS gp3 cho database (nếu self-managed MySQL) hoặc RDS. EC2 Auto Scaling Group mount EFS từ cùng Mount Target trong AZ tương ứng. Static assets (images, videos) nên serve từ S3 + CloudFront.

---

## 🔗 Điều Hướng

- [← 4-efs-intelligent-tiering.md](./4-efs-intelligent-tiering.md) — Phân Tầng Thông Minh
- [← README.md](./README.md) — Tổng Quan EFS
- [→ ../05-security/](../05-security/) — Bảo Mật Lưu Trữ
- [→ ../03-ebs/6-ebs-vs-instance-store.md](../03-ebs/6-ebs-vs-instance-store.md) — So Sánh EBS
