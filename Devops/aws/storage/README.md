# ☁️ AWS Storage Services — Lộ Trình Học Tập

> Hướng dẫn toàn diện về các dịch vụ lưu trữ AWS, từ kiến thức nền tảng đến vận hành nâng cao trong môi trường production.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Dịch Vụ](#tổng-quan-dịch-vụ)
4. [Chủ Đề Chi Tiết](#chủ-đề-chi-tiết)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] Mô hình lưu trữ AWS — Object, Block, File Storage (Lưu trữ Đối tượng, Khối, Tệp)
- [ ] S3 — Simple Storage Service — Dịch vụ Lưu trữ Đơn giản: bucket, object, ACL
- [ ] EBS — Elastic Block Store — Lưu trữ Khối Linh hoạt: volume types, snapshots
- [ ] Các khái niệm về Durability (Độ bền) và Availability (Tính sẵn sàng)

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)**

- [ ] S3 nâng cao: Versioning (Quản lý phiên bản), Lifecycle Policies (Chính sách vòng đời), Replication (Sao chép)
- [ ] EFS — Elastic File System — Hệ thống Tệp Linh hoạt: NFS, shared storage
- [ ] Bảo mật lưu trữ: IAM, Bucket Policies, Encryption (Mã hóa) at-rest và in-transit
- [ ] Storage Classes (Lớp lưu trữ) và tối ưu chi phí

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Performance Optimization (Tối ưu hiệu suất) cho S3 và EBS
- [ ] Disaster Recovery (Khôi phục thảm họa) và Cross-Region Replication (Sao chép liên vùng)
- [ ] Monitoring & Observability (Giám sát & Quan sát) với CloudWatch
- [ ] Storage Gateway — Cổng Lưu trữ Hybrid Cloud (Đám mây Lai)

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] FSx — Amazon File System (Hệ thống Tệp Amazon): Windows, Lustre, NetApp
- [ ] Snow Family — Thiết bị Di chuyển Dữ liệu: Snowcone, Snowball, Snowmobile
- [ ] DataSync — Đồng bộ dữ liệu tự động
- [ ] Thiết kế kiến trúc lưu trữ hỗn hợp

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                              | Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------- | ------- | --------- | ---------- |
| **S3 Fundamentals** (Kiến thức nền)   | ⭐⭐⭐  | 2 tuần    | -          |
| **EBS & Storage Types** (Loại lưu trữ)| ⭐⭐⭐  | 1 tuần    | -          |
| **Bảo Mật & Mã Hóa**                  | ⭐⭐⭐  | 2 tuần    | -          |
| **Lifecycle & Cost Optimization**     | ⭐⭐⭐  | 1 tuần    | -          |
| **Replication & HA** (Sao chép & Cao khả dụng) | ⭐⭐⭐ | 2 tuần | -      |
| **Monitoring & Alerting** (Giám sát)  | ⭐⭐⭐  | 1 tuần    | -          |
| **EFS & Shared Storage** (Lưu trữ chia sẻ) | ⭐⭐ | 1 tuần  | -          |
| **Storage Gateway**                   | ⭐⭐    | 1 tuần    | -          |
| **Snow Family & DataSync**            | ⭐⭐    | 1 tuần    | -          |
| **FSx & Advanced File Systems**       | ⭐      | 2 tuần    | -          |

---

## 🗺️ Tổng Quan Dịch Vụ

### Object Storage (Lưu Trữ Đối Tượng)

| Dịch Vụ | Mô Tả | Use Case (Trường hợp dùng) |
| ------- | ------ | -------------------------- |
| **S3 Standard** | Độ bền 11 chín (99.999999999%), truy cập thường xuyên | Web assets, backup nóng |
| **S3 Standard-IA** | Infrequent Access — Truy cập Không Thường Xuyên | Backup, log dài hạn |
| **S3 Glacier Instant** | Lưu trữ lạnh, truy xuất tức thì | Archive truy cập thỉnh thoảng |
| **S3 Glacier Flexible** | Truy xuất trong vài phút đến vài giờ | Compliance archive |
| **S3 Glacier Deep Archive** | Lưu trữ lạnh nhất, chi phí thấp nhất | Lưu trữ dài hạn 7–10 năm |
| **S3 Intelligent-Tiering** | Tự động di chuyển giữa các tầng lưu trữ | Dữ liệu không dự đoán được pattern |

### Block Storage (Lưu Trữ Khối)

| Dịch Vụ | Mô Tả | Use Case |
| ------- | ------ | --------- |
| **EBS gp3** | General Purpose SSD — SSD Đa Dụng, 16.000 IOPS | Boot volumes, ứng dụng chung |
| **EBS io2 Block Express** | Provisioned IOPS SSD, 256.000 IOPS | Database, latency thấp |
| **EBS st1** | Throughput Optimized HDD — HDD Tối ưu Thông lượng | Data warehouse, log streaming |
| **EBS sc1** | Cold HDD — HDD Lạnh | Dữ liệu truy cập thỉnh thoảng |

### File Storage (Lưu Trữ Tệp)

| Dịch Vụ | Mô Tả | Use Case |
| ------- | ------ | --------- |
| **EFS** | NFS-based, serverless, auto-scaling | Shared storage cho EC2/containers |
| **FSx for Windows** | SMB — Server Message Block, Windows-native | Windows workloads, Active Directory |
| **FSx for Lustre** | HPC — High Performance Computing, parallel FS | ML training, genomics, rendering |
| **FSx for NetApp ONTAP** | Enterprise NFS/SMB | Migration từ on-premises |
| **FSx for OpenZFS** | ZFS-based, snapshots nhanh | Dev/test environments |

---

## 📁 Chủ Đề Chi Tiết

### 📁 **1. S3 Fundamentals** (`01-s3-fundamentals/`)

- Bucket và Object Model (Mô hình Bucket và Object)
- Storage Classes (Lớp Lưu Trữ) — Standard, IA, Glacier, Intelligent-Tiering
- Versioning (Quản lý Phiên Bản) và MFA Delete
- Presigned URLs — URL có Chữ ký Tạm Thời
- Multipart Upload — Tải Lên Nhiều Phần cho file lớn
- S3 Transfer Acceleration — Tăng Tốc Truyền Tải

### 📁 **2. S3 Nâng Cao** (`02-s3-advanced/`)

- **Lifecycle Policies** — Chính Sách Vòng Đời tự động chuyển tầng
- **Cross-Region Replication (CRR)** — Sao Chép Liên Vùng
- **Same-Region Replication (SRR)** — Sao Chép Cùng Vùng
- S3 Object Lock — Khóa Đối Tượng (WORM — Write Once Read Many)
- S3 Event Notifications — Thông Báo Sự Kiện (Lambda, SQS, SNS)
- S3 Batch Operations — Thao Tác Hàng Loạt

### 📁 **3. EBS — Elastic Block Store** (`03-ebs/`)

- Volume Types (Loại Volume) và khi nào dùng loại nào
- Snapshots (Ảnh Chụp Nhanh) và Lifecycle Manager
- EBS Multi-Attach — Gắn Kết Nhiều EC2 (chỉ io1/io2)
- Performance Tuning (Tối Ưu Hiệu Suất): IOPS, Throughput, Latency
- Encryption (Mã Hóa) với AWS KMS — Key Management Service
- EBS vs Instance Store — So sánh lưu trữ tạm thời vs bền vững

### 📁 **4. EFS — Elastic File System** (`04-efs/`)

- NFS — Network File System: mount targets, access points
- Performance Modes (Chế Độ Hiệu Suất): General Purpose vs Max I/O
- Throughput Modes (Chế Độ Thông Lượng): Bursting, Provisioned, Elastic
- EFS Intelligent-Tiering — Phân Tầng Thông Minh
- Security Groups và IAM Authorization (Ủy Quyền)
- EFS vs EBS vs S3 — Khi nào dùng loại nào

### 📁 **5. Bảo Mật Lưu Trữ** (`05-security/`)

- IAM — Identity and Access Management: Policies cho S3/EBS/EFS
- S3 Bucket Policies (Chính Sách Bucket) và ACL — Access Control Lists
- Block Public Access (Chặn Truy Cập Công Khai) — cấu hình và best practices
- Encryption at-rest (Mã hóa Khi Lưu): SSE-S3, SSE-KMS, SSE-C, CSE
- Encryption in-transit (Mã hóa Khi Truyền): TLS/HTTPS enforcement
- VPC Endpoint — Điểm Cuối VPC cho S3 (Gateway và Interface)
- S3 Access Analyzer — Phân Tích Quyền Truy Cập

### 📁 **6. Chi Phí & Lifecycle Optimization** (`06-cost-optimization/`)

- Hiểu Bảng Giá S3, EBS, EFS
- Lifecycle Rules (Quy Tắc Vòng Đời) — tự động hóa chuyển tầng
- S3 Intelligent-Tiering — khi nào nên dùng
- Reserved Storage cho EBS (Cam Kết Trước) vs On-Demand
- Cost Allocation Tags (Thẻ Phân Bổ Chi Phí)
- AWS Cost Explorer và S3 Storage Lens (Kính Phân Tích Lưu Trữ)

### 📁 **7. Monitoring & Observability** (`07-monitoring/`)

- CloudWatch Metrics (Chỉ Số) cho S3, EBS, EFS
- S3 Server Access Logging — Nhật Ký Truy Cập
- AWS CloudTrail — Audit Trail (Dấu Vết Kiểm Toán) cho API calls
- EBS CloudWatch Alarms (Cảnh Báo): BurstBalance, VolumeQueueLength
- S3 Storage Lens — Dashboard phân tích toàn tổ chức
- Cost Anomaly Detection (Phát Hiện Bất Thường Chi Phí)

### 📁 **8. Disaster Recovery & Replication** (`08-disaster-recovery/`)

- RPO — Recovery Point Objective — Mục Tiêu Điểm Khôi Phục
- RTO — Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục
- S3 Cross-Region Replication cho DR
- EBS Snapshots và AMI — Amazon Machine Image sao chép vùng
- AWS Backup — Dịch Vụ Backup Tập Trung
- Backup Vault Lock (Khóa Kho Backup) — bất biến theo WORM

### 📁 **9. Storage Gateway & Hybrid** (`09-storage-gateway/`)

- **File Gateway** — Giao Thức NFS/SMB cho S3
- **Volume Gateway** — iSCSI Block Storage sao lưu lên S3
- **Tape Gateway** — Virtual Tape Library — Thư Viện Băng Ảo
- DataSync — Di Chuyển Dữ Liệu Tự Động (on-premises ↔ AWS)
- Direct Connect vs VPN cho hybrid storage

### 📁 **10. Snow Family & Large-Scale Migration** (`10-snow-family/`)

- **Snowcone** — Thiết bị di động 8TB, edge computing
- **Snowball Edge Storage Optimized** — 80TB, di chuyển lớn
- **Snowball Edge Compute Optimized** — 42TB + vCPU, xử lý tại biên
- **Snowmobile** — Container 100PB, di chuyển datacenter
- AWS DataSync vs Snow Family — Chọn loại nào
- OpsHub — Giao Diện Quản Lý Snow Devices

### 📁 **11. FSx — Advanced File Systems** (`11-fsx/`)

- FSx for Windows File Server — tích hợp Active Directory
- FSx for Lustre — HPC workloads, tích hợp S3
- FSx for NetApp ONTAP — multi-protocol, deduplication
- FSx for OpenZFS — snapshots tức thì, clones
- Hiệu suất và chi phí so sánh

### 📁 **12. Chuẩn Bị Phỏng Vấn** (`12-interview-prep/`)

- Top 20 Câu Hỏi Phỏng Vấn AWS Storage
- System Design Scenarios (Tình Huống Thiết Kế Hệ Thống)
- Incident Response Stories — Phương Pháp STAR
- Câu Hỏi về Chi Phí và Tối Ưu

---

## 🎓 Theo Dịch Vụ AWS

### **Amazon S3**

```
Điểm mạnh: Độ bền 11 chín, vô hạn dung lượng, nhiều storage class
Lý tưởng cho: Backup, static assets, data lake, archive
Chi phí: $0.023/GB (Standard) — thấp hơn khi dùng IA/Glacier
```

### **Amazon EBS**

```
Điểm mạnh: Block storage, low latency, IOPS cao
Lý tưởng cho: Boot volumes, databases, applications yêu cầu latency thấp
Chi phí: $0.10/GB-month (gp3) — tính theo provision
```

### **Amazon EFS**

```
Điểm mạnh: Shared NFS, serverless, tự co giãn
Lý tưởng cho: Shared content, CMS, home directories, containers
Chi phí: $0.30/GB-month — cao hơn EBS nhưng không cần provision
```

### **Amazon FSx**

```
Điểm mạnh: Managed enterprise file systems, nhiều protocol
Lý tưởng cho: Windows workloads, HPC, enterprise migration
Chi phí: Tùy loại — FSx Lustre từ $0.14/GB-month
```

### **AWS Storage Gateway**

```
Điểm mạnh: Kết nối on-premises với AWS storage
Lý tưởng cho: Hybrid cloud, backup lên cloud, tape replacement
Chi phí: Không phí gateway, trả theo dữ liệu đọc/ghi
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                     | Thư Mục                                                                   | Ưu Tiên            |
| -------------------------- | ------------------------------------------------------------------------- | ------------------ |
| Bắt đầu từ đây             | [README.md](./README.md)                                                  | Bắt đầu            |
| Index đầy đủ               | [INDEX.md](./INDEX.md)                                                    | Tổng quan          |
| S3 cơ bản                  | [01-s3-fundamentals/](./01-s3-fundamentals/)                              | Thiết yếu          |
| Bảo mật lưu trữ            | [05-security/](./05-security/)                                            | Quan trọng         |
| Tối ưu chi phí             | [06-cost-optimization/](./06-cost-optimization/)                          | Production         |
| Phỏng vấn                  | [12-interview-prep/](./12-interview-prep/)                                | Trước phỏng vấn    |

---

## 📊 Ma Trận Kỹ Năng

### Beginner (Mới Bắt Đầu — 0–1 năm)

- [ ] Phân biệt S3, EBS, EFS
- [ ] Tạo S3 bucket, upload/download object
- [ ] Tạo EBS volume và gắn vào EC2
- [ ] Hiểu S3 storage classes cơ bản
- [ ] Cấu hình bucket policy đơn giản

### Intermediate (Trung Cấp — 1–3 năm)

- [ ] Thiết kế lifecycle policies
- [ ] Cấu hình Cross-Region Replication
- [ ] Implement encryption at-rest và in-transit
- [ ] Monitoring EBS performance với CloudWatch
- [ ] Tối ưu chi phí với Intelligent-Tiering
- [ ] Thiết kế disaster recovery với S3

### Advanced (Nâng Cao — 3–5+ năm)

- [ ] Kiến trúc data lake trên S3
- [ ] Multi-region storage strategy
- [ ] Storage Gateway cho hybrid cloud
- [ ] FSx cho HPC/enterprise workloads
- [ ] Cost optimization ở quy mô lớn
- [ ] Security hardening toàn diện

---

## 🚀 Bắt Đầu

### Bước 1: Xác Định Mục Tiêu

```
Chọn hướng của bạn:
- Generalist Cloud Engineer (tất cả dịch vụ lưu trữ)
- Specialist (S3 expert — data lake, analytics)
- Operations (backup, DR, monitoring)
- Cost Engineer (tối ưu chi phí lưu trữ)
```

### Bước 2: Thiết Lập Lab

```bash
# Cài đặt AWS CLI
aws configure

# Tạo S3 bucket thử nghiệm
aws s3 mb s3://my-test-bucket-$(date +%s)

# List buckets
aws s3 ls
```

### Bước 3: Học và Thực Hành

```
1. Đọc một module (30 phút)
2. Thực hành trên AWS Console hoặc CLI (30 phút)
3. Thực hành với AWS Free Tier (30–60 phút)
4. Review checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị theo phương pháp STAR:
- Situation  (Tình huống)
- Task       (Nhiệm vụ)
- Action     (Hành động)
- Result     (Kết quả)
```

---

## 📖 Tài Liệu Tham Khảo

### Tài Liệu Thiết Yếu

- **AWS Storage Documentation** — docs.aws.amazon.com/storage
- **AWS Well-Architected Framework** — Reliability Pillar (Trụ Cột Độ Tin Cậy)
- **"AWS Certified Solutions Architect"** — Sybex — chương Storage
- **"Cloud Native Patterns"** — Cornelia Davis — Data management

### Tài Liệu Chính Thức AWS

- [Amazon S3 User Guide](https://docs.aws.amazon.com/s3/)
- [Amazon EBS User Guide](https://docs.aws.amazon.com/ebs/)
- [Amazon EFS User Guide](https://docs.aws.amazon.com/efs/)
- [AWS Storage Blog](https://aws.amazon.com/blogs/storage/)

### Bài Viết & Blog

- AWS Storage Blog — cập nhật tính năng mới
- re:Invent Storage Sessions (YouTube)
- A Cloud Guru — AWS Storage courses
- Adrian Cantrill — SA Pro course

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Thường Gặp Theo Danh Mục

#### S3 & Object Storage

- [ ] Phân biệt S3 storage classes và khi nào dùng
- [ ] Cách bảo vệ dữ liệu trong S3 (versioning, MFA delete, Object Lock)
- [ ] Thiết kế data lake trên S3
- [ ] S3 presigned URL hoạt động như thế nào

#### Block & File Storage

- [ ] Khi nào dùng EBS vs EFS vs S3
- [ ] Các loại EBS volume và trade-offs
- [ ] EFS performance modes khác nhau như thế nào
- [ ] EBS Multi-Attach là gì và khi nào dùng

#### Bảo Mật

- [ ] Các cách mã hóa dữ liệu trong S3
- [ ] Thiết kế bucket policy cho multi-tenant
- [ ] VPC Endpoint cho S3 hoạt động như thế nào
- [ ] Principle of Least Privilege (Nguyên Tắc Quyền Tối Thiểu) áp dụng vào lưu trữ

#### Kiến Trúc & Vận Hành

- [ ] Thiết kế DR với RPO = 1 giờ, RTO = 30 phút
- [ ] Giảm chi phí S3 cho dữ liệu lâu đời
- [ ] Hybrid storage với Storage Gateway
- [ ] Di chuyển 500TB từ on-premises lên AWS

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc triển khai production, kiểm tra:

- [ ] Có thể giải thích sự khác biệt S3, EBS, EFS từ đầu
- [ ] Có thể thiết kế lifecycle policy cho dữ liệu 7 năm
- [ ] Có thể implement bucket policy an toàn
- [ ] Có thể chọn EBS volume type phù hợp
- [ ] Có thể giám sát storage với CloudWatch
- [ ] Có thể thiết kế DR strategy cho ứng dụng
- [ ] Có thể tính toán và tối ưu chi phí lưu trữ
- [ ] Có thể giải thích encryption options trong S3
- [ ] Có thể xử lý incident: "S3 bucket public" hoặc "EBS volume đầy"
- [ ] Có thể thiết kế hybrid storage với Storage Gateway

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner/Intermediate/Advanced)
├─ 3️⃣  Bắt đầu với 01-s3-fundamentals/
├─ 4️⃣  Thực hành trên AWS Free Tier
├─ 5️⃣  Hoàn thành bài tập mỗi chủ đề
├─ 6️⃣  Xây dựng project portfolio
└─ 7️⃣  Chuẩn bị phỏng vấn với 12-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
