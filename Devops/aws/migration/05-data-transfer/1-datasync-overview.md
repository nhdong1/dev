# AWS DataSync — Tổng Quan: Agent, Location, Task, Scheduling

> **AWS DataSync** là dịch vụ truyền và đồng bộ dữ liệu trực tuyến, tự động giữa on-premises storage và AWS storage — hoặc giữa các AWS storage services với nhau. DataSync xử lý tất cả phần phức tạp: mã hóa, kiểm tra toàn vẹn dữ liệu, tối ưu mạng, retry (thử lại khi lỗi), và lên lịch tự động — không cần script tùy chỉnh.

## 📚 Mục Lục (Table of Contents)

1. [DataSync là gì và khi nào dùng?](#datasync-là-gì-và-khi-nào-dùng)
2. [Kiến Trúc DataSync](#kiến-trúc-datasync)
3. [DataSync Agent — Tác Nhân](#datasync-agent--tác-nhân)
4. [Location — Vị Trí Nguồn và Đích](#location--vị-trí-nguồn-và-đích)
5. [Task — Tác Vụ Đồng Bộ](#task--tác-vụ-đồng-bộ)
6. [Task Execution — Thực Thi Tác Vụ](#task-execution--thực-thi-tác-vụ)
7. [Scheduling — Lên Lịch Tự Động](#scheduling--lên-lịch-tự-động)
8. [Monitoring — Theo Dõi](#monitoring--theo-dõi)
9. [Bảo Mật](#bảo-mật)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 DataSync là gì và khi nào dùng?

### DataSync là gì?

```
AWS DataSync = Dịch vụ truyền dữ liệu tự động, nhanh, có kiểm tra toàn vẹn

So sánh với các cách truyền file truyền thống:
┌─────────────────────┬──────────────────────────────────────────────────────┐
│ Phương pháp         │ Vấn đề                                               │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ rsync thủ công      │ Không có retry, không mã hóa tốt, chậm              │
│ scp / sftp          │ Single-threaded, không lên lịch, không monitor       │
│ AWS CLI s3 sync     │ Không hỗ trợ NFS/SMB, cần script phức tạp           │
│ AWS DataSync        │ ✅ Tự động hóa toàn bộ, nhanh 10x, checksum, retry  │
└─────────────────────┴──────────────────────────────────────────────────────┘
```

### Khi nào dùng DataSync?

```
Phù hợp:
├── Di chuyển dữ liệu lớn (TB đến PB) từ on-premises lên AWS
├── Đồng bộ định kỳ: backup nightly (sao lưu hàng đêm), archiving (lưu trữ)
├── Copy dữ liệu giữa các AWS region hoặc giữa S3/EFS/FSx
├── Replicate (sao chép) NAS/SAN on-premises lên EFS hoặc FSx
└── Tạo data lake (hồ dữ liệu) bằng cách tập trung file vào S3

Không phù hợp:
├── Dữ liệu > 100 TB mà mạng chậm → dùng Snow Family
├── Streaming data (dữ liệu luồng real-time) → dùng Kinesis
├── Database migration → dùng DMS
└── SFTP file exchange với đối tác → dùng Transfer Family
```

---

## 🏗️ Kiến Trúc DataSync

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS DataSync Architecture                        │
├───────────────────────────────┬─────────────────────────────────────────┤
│      On-Premises / Edge       │              AWS Cloud                   │
│                               │                                          │
│  ┌──────────┐                 │  ┌──────────────────────────────────┐   │
│  │  NFS     │                 │  │       DataSync Service            │   │
│  │  Share   ├──┐              │  │  ┌──────────┐  ┌─────────────┐  │   │
│  └──────────┘  │              │  │  │ Location │  │    Task     │  │   │
│                │              │  │  │ (Source) │→ │  (Rules &   │  │   │
│  ┌──────────┐  ▼              │  │  └──────────┘  │  Schedule)  │  │   │
│  │  SMB     │ ┌──────────┐    │  │                └──────┬──────┘  │   │
│  │  Share   ├→│DataSync  │────┼─→│                       ▼         │   │
│  └──────────┘ │  Agent   │    │  │  ┌──────────────────────────┐   │   │
│               │  (VM)    │    │  │  │  Location (Destination)  │   │   │
│  ┌──────────┐ └──────────┘    │  │  │  S3 / EFS / FSx          │   │   │
│  │  HDFS    │                 │  │  └──────────────────────────┘   │   │
│  └──────────┘                 │  └──────────────────────────────────┘   │
└───────────────────────────────┴─────────────────────────────────────────┘

Luồng dữ liệu:
Source Storage → DataSync Agent → DataSync Service → Destination Storage
                 (On-premises)     (AWS managed)       (S3/EFS/FSx)
```

### Các thành phần chính

| Thành Phần | Mô Tả |
| ---------- | ------ |
| **Agent** — Tác Nhân | VM (máy ảo) cài tại on-premises, kết nối với source storage |
| **Location** — Vị Trí | Định nghĩa nơi lưu dữ liệu (source hoặc destination) |
| **Task** — Tác Vụ | Định nghĩa luồng copy từ source location → destination location |
| **Task Execution** — Lần Chạy | Một lần chạy cụ thể của task (có thể on-demand hoặc scheduled) |

---

## 🤖 DataSync Agent — Tác Nhân

### Agent là gì?

DataSync Agent là một **virtual machine (máy ảo)** được cài đặt tại on-premises (hoặc tại AWS nếu transfer trong AWS). Agent đóng vai trò là cầu nối giữa source storage và DataSync service trên AWS.

```
Agent deployment options (tùy chọn triển khai agent):

Option 1: VMware ESXi (phổ biến nhất trong enterprise)
└── Deploy OVA (Open Virtual Appliance — File Máy Ảo Mở) trên ESXi host

Option 2: Microsoft Hyper-V
└── Deploy VHDX (Virtual Hard Disk — Đĩa Ảo Hyper-V) trên Hyper-V

Option 3: Linux KVM (Kernel-based Virtual Machine)
└── Deploy QCOW2 (disk image format) trên KVM host

Option 4: Amazon EC2 (khi source/destination đều trong AWS)
└── Launch AMI (Amazon Machine Image) từ AWS Marketplace
└── Không cần on-premises agent cho transfer trong AWS

Option 5: AWS Snowcone
└── DataSync agent pre-installed (cài sẵn) trên Snowcone device
└── Transfer dữ liệu edge → AWS sau khi Snowcone kết nối mạng
```

### Yêu cầu phần cứng Agent

```
Minimum requirements (yêu cầu tối thiểu):
├── vCPU: 4 cores
├── RAM: 32 GB
├── Disk: 80 GB (cho cache và logs)
└── Network: 1 Gbps (khuyến nghị 10 Gbps cho performance cao)

Bandwidth vs Performance:
├── 1 Gbps link → ~100 MB/s throughput thực tế
├── 10 Gbps link → ~900 MB/s throughput
└── Có thể chạy nhiều agent song song để tăng tổng throughput
```

### Kích hoạt Agent (Agent Activation)

```
Quy trình kích hoạt:
1. Download và deploy VM image từ AWS Console
2. Mở port 80 (HTTP) tạm thời để nhận activation key
3. Trong AWS Console → DataSync → Agents → Create agent
4. Nhập IP của agent → AWS gửi request đến agent qua port 80
5. Agent trả về activation key
6. Agent kết nối về AWS DataSync endpoint qua port 443 (HTTPS)
7. Agent được kích hoạt → xuất hiện trong Console ở trạng thái "Online"

Lưu ý bảo mật:
├── Sau khi kích hoạt, đóng port 80 — không còn cần nữa
└── Agent chỉ dùng outbound HTTPS (port 443) về AWS
```

---

## 📍 Location — Vị Trí Nguồn và Đích

### Các loại Location được hỗ trợ

#### On-Premises / Edge (cần Agent)

```
NFS Location — Network File System (Hệ Thống File Mạng):
├── Hỗ trợ: NFS v3, v4, v4.1
├── Cấu hình: Server IP/hostname + mount path
└── Ví dụ: nfs://192.168.1.100/data/exports

SMB Location — Server Message Block (Giao Thức Chia Sẻ File Windows):
├── Hỗ trợ: SMB 2.0, 2.1, 3.0
├── Cần: username, password, domain
└── Ví dụ: smb://fileserver/sharedfolder

HDFS Location — Hadoop Distributed File System (Hệ Thống File Phân Tán Hadoop):
├── Dùng cho Hadoop clusters (cụm Hadoop)
├── Hỗ trợ Kerberos authentication (xác thực Kerberos)
└── Thường dùng để di chuyển data lake từ on-premises lên S3

Object Storage Location (S3-compatible on-premises):
└── Dell EMC ECS, IBM COS, MinIO, Cloudian...
```

#### AWS Storage (không cần Agent khi transfer trong AWS)

```
Amazon S3:
├── Tất cả S3 storage classes: Standard, Standard-IA, One Zone-IA,
│   Intelligent-Tiering, Glacier Instant, Glacier Flexible, Glacier Deep Archive
└── Có thể chọn storage class cho từng file khi transfer

Amazon EFS — Elastic File System (Hệ Thống File Co Giãn):
├── Dùng cho Linux workloads cần POSIX-compliant file system
└── Hỗ trợ transfer giữa hai EFS (ví dụ: copy data giữa regions)

Amazon FSx for Windows File Server:
├── Dùng cho Windows workloads cần SMB share
└── Tích hợp Active Directory

Amazon FSx for Lustre:
├── High-performance computing (HPC — Tính Toán Hiệu Năng Cao)
└── Thường dùng cho ML training data migration

Amazon FSx for OpenZFS và ONTAP:
└── Cho enterprise storage workloads cần ZFS hoặc NetApp ONTAP
```

### Tạo Location trong Console

```
Ví dụ tạo NFS source location:
1. DataSync Console → Locations → Create location
2. Chọn "Network File System (NFS)"
3. Điền:
   ├── Agent: chọn agent đã kích hoạt
   ├── NFS server: 192.168.1.100
   └── Mount path: /mnt/data
4. "Create location" → location sẵn sàng
```

---

## 📋 Task — Tác Vụ Đồng Bộ

### Task là gì?

Task định nghĩa:
- **Source location** (nguồn) → **Destination location** (đích)
- **Options** (tùy chọn): xử lý file metadata, permissions, ownership thế nào
- **Filters** (bộ lọc): file/folder nào include hoặc exclude
- **Schedule** (lịch): chạy khi nào

### Task Options — Tùy Chọn Tác Vụ

```
Transfer mode (chế độ truyền):
├── Changed files only (mặc định):
│   ├── So sánh file tại source và destination
│   ├── Chỉ copy file mới hoặc đã thay đổi (kiểm tra qua mtime + size)
│   └── Hiệu quả cho incremental sync (đồng bộ gia tăng)
└── All files:
    ├── Copy tất cả file bất kể đã có tại destination chưa
    └── Dùng khi cần verify (xác minh) toàn bộ dataset

Verify mode (chế độ xác minh toàn vẹn):
├── Only files transferred (mặc định):
│   └── Verify checksum của file vừa copy trong lần chạy này
├── All files in the destination:
│   └── Verify tất cả file tại destination (tốn thời gian hơn)
└── None:
    └── Không verify (nhanh nhất, nhưng rủi ro dữ liệu hỏng)

POSIX permissions (quyền truy cập POSIX — Portable Operating System Interface):
├── Preserve (mặc định): giữ nguyên owner, group, permissions
└── Do not preserve: bỏ qua permissions (dùng khi cross-platform)

Timestamps (dấu thời gian):
├── Preserve: giữ mtime (modification time — thời gian sửa đổi)
└── Do not preserve: dùng timestamp của lần transfer

Deleted files (file bị xóa tại source):
├── Preserve: giữ file tại destination dù source đã xóa
└── Remove (nguy hiểm): xóa file tại destination nếu source đã xóa
```

### Task Queueing (Hàng Đợi Tác Vụ)

```
Mặc định: Một task chỉ chạy một execution tại một thời điểm.
Nếu schedule trigger khi task đang chạy → execution mới được xếp vào hàng đợi.

Queue behavior options:
├── QUEUED: Chờ execution hiện tại hoàn thành → chạy tiếp
└── NOT_QUEUED: Bỏ qua execution mới nếu task đang bận
```

---

## ▶️ Task Execution — Thực Thi Tác Vụ

### Các giai đoạn của một Task Execution

```
Phase 1: LAUNCHING
└── DataSync chuẩn bị tài nguyên, kết nối agent

Phase 2: PREPARING
└── DataSync quét (scan) source và destination để liệt kê file
    ├── Tạo danh sách file cần transfer
    └── Thống kê: số file, tổng kích thước

Phase 3: TRANSFERRING
├── Truyền dữ liệu thực tế từ source → destination
├── Parallel transfer: nhiều thread đồng thời (tăng throughput)
├── Compression (nén dữ liệu) tự động trong quá trình truyền
└── Hiển thị progress: số file đã transfer, MB/s

Phase 4: VERIFYING (nếu được bật)
├── Tính và so sánh checksum của file vừa transfer
└── Đảm bảo dữ liệu tại destination trùng khớp với source

Phase 5: SUCCESS / ERROR
├── SUCCESS: Tất cả file transfer và verify thành công
└── ERROR: Có file lỗi → xem error report (báo cáo lỗi) trong CloudWatch Logs
```

### Task Execution Status (Trạng thái thực thi)

| Trạng Thái | Ý Nghĩa |
| ---------- | ------- |
| `LAUNCHING` | Đang khởi động |
| `PREPARING` | Đang quét danh sách file |
| `TRANSFERRING` | Đang truyền dữ liệu |
| `VERIFYING` | Đang kiểm tra toàn vẹn |
| `SUCCESS` | Hoàn thành thành công |
| `ERROR` | Có lỗi — xem logs |

---

## 🕒 Scheduling — Lên Lịch Tự Động

### Cách cấu hình Schedule

```
DataSync hỗ trợ cron expression (biểu thức cron — định nghĩa lịch):

Cú pháp: cron(phút giờ ngày tháng ngày-trong-tuần năm)

Ví dụ phổ biến:
├── Hàng đêm lúc 2 AM UTC:      cron(0 2 * * ? *)
├── Hàng giờ:                    cron(0 * * * ? *)
├── Thứ Hai đến Thứ Sáu, 6 AM:  cron(0 6 ? * MON-FRI *)
└── Mỗi 15 phút:                 cron(0/15 * * * ? *)

Lưu ý: AWS dùng cron với 6 fields (6 trường), khác cron Unix 5 fields
```

### Chiến lược lên lịch điển hình

```
Chiến lược 1: Nightly full scan (Quét toàn bộ hàng đêm)
├── Schedule: cron(0 1 * * ? *)  — 1 AM mỗi đêm
├── Task option: "Changed files only"
└── Phù hợp: backup NAS, archive data lên S3 Glacier

Chiến lược 2: Hourly incremental (Đồng bộ gia tăng hàng giờ)
├── Schedule: cron(0 * * * ? *)
├── Phù hợp: dữ liệu thay đổi thường xuyên, cần sync gần real-time
└── Lưu ý: mỗi execution có overhead (chi phí khởi động) ~ vài phút

Chiến lược 3: One-time migration (Di chuyển một lần)
├── Không cần schedule
├── Chạy manually khi cần
└── Phù hợp: initial data load (tải dữ liệu ban đầu) cho migration project
```

---

## 📊 Monitoring — Theo Dõi

### CloudWatch Metrics (Số Liệu CloudWatch)

```
DataSync tự động gửi metrics vào Amazon CloudWatch:

Key metrics (số liệu quan trọng):
├── BytesTransferred: Tổng bytes đã truyền
├── BytesVerified: Tổng bytes đã verify
├── FilesTransferred: Số file đã truyền
├── FilesVerified: Số file đã verify
├── FilesDeleted: Số file đã xóa tại destination (nếu bật remove deleted)
└── TaskExecutionStatus: Trạng thái execution hiện tại

Tạo CloudWatch Alarm (cảnh báo) điển hình:
└── Alarm khi execution kéo dài hơn X giờ → có thể bị stuck
```

### CloudWatch Logs (Nhật Ký CloudWatch)

```
DataSync ghi chi tiết error log vào CloudWatch Logs:
├── File nào bị lỗi (không transfer được)
├── Lý do lỗi: permission denied, file locked, path too long...
└── Có thể dùng Athena (query service) để phân tích log lớn
```

### DataSync Console Dashboard

```
Xem trực tiếp trong AWS Console:
├── Task execution history (lịch sử các lần chạy)
├── Progress real-time (số file, GB đã transfer)
├── Transfer rate (tốc độ truyền): MB/s hiện tại
└── Errors (lỗi): số file lỗi và lý do
```

---

## 🔒 Bảo Mật

### Mã hóa dữ liệu (Encryption)

```
In-transit (khi truyền qua mạng):
└── TLS 1.2 — Transport Layer Security (Bảo Mật Tầng Truyền Tải)
    ├── DataSync Agent ↔ DataSync Service: TLS 1.2
    └── DataSync Service ↔ S3/EFS/FSx: HTTPS

At-rest (khi lưu trữ tại destination):
├── S3: Server-Side Encryption (SSE — Mã Hóa Phía Server)
│   ├── SSE-S3: AWS managed key (khóa do AWS quản lý)
│   ├── SSE-KMS: AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)
│   └── SSE-C: Customer-provided key (khóa khách hàng tự cung cấp)
└── EFS/FSx: Mã hóa được cấu hình tại storage destination
```

### IAM Permissions (Quyền IAM — Identity and Access Management)

```
DataSync cần IAM role với các quyền:
├── Đọc từ source S3 bucket (nếu source là S3)
├── Ghi vào destination S3 bucket
├── Truy cập EFS/FSx
└── Ghi CloudWatch Logs và Metrics

Ví dụ IAM policy cho DataSync task role:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::source-bucket/*", "arn:aws:s3:::source-bucket"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::destination-bucket/*"
    }
  ]
}
```

### Network Security (Bảo Mật Mạng)

```
Agent outbound connections (kết nối ra ngoài của Agent):
├── Port 443 (HTTPS): Agent ↔ DataSync Service endpoints
├── Port 443 hoặc 80: Agent ↔ S3/EFS (khi dùng VPC endpoints)
└── Các port NFS/SMB: Agent ↔ Source storage (on-premises network)

Best practice:
├── Dùng AWS Direct Connect hoặc VPN cho agent traffic
├── Dùng VPC Endpoints (điểm cuối VPC) để không ra internet
└── Security Group của agent: chỉ allow outbound 443, block inbound
```

---

## 🎓 Câu Hỏi Phỏng Vấn

1. **DataSync Agent là gì? Tại sao cần nó?**
   - VM cài tại on-premises, kết nối source NFS/SMB với DataSync service. Cần vì service AWS không thể trực tiếp truy cập storage on-premises.

2. **Phân biệt Location và Task trong DataSync.**
   - Location = nơi lưu dữ liệu (NFS share, S3 bucket); Task = luồng copy từ source location đến destination location với các cấu hình cụ thể.

3. **DataSync verify dữ liệu như thế nào?**
   - Tính checksum của từng file sau khi transfer, so sánh với checksum tại source. Nếu khác nhau → báo lỗi.

4. **Khi nào dùng "All files" thay vì "Changed files only"?**
   - "Changed files only" cho incremental sync hàng ngày. "All files" khi cần verify toàn bộ dataset hoặc rebuild destination từ đầu.

5. **Tính thời gian transfer 10 TB qua đường 100 Mbps.**
   - 10 TB = 10 × 1024 × 1024 MB = 10,485,760 MB. Tốc độ 100 Mbps ≈ 12.5 MB/s. Thời gian = 10,485,760 / 12.5 ≈ 838,860 giây ≈ ~9.7 ngày. (Thực tế overhead ~20% → ~12 ngày).

6. **Có cần Agent khi copy dữ liệu từ S3 us-east-1 sang S3 ap-southeast-1 không?**
   - Không. Khi cả source lẫn destination đều là AWS storage, DataSync tự kết nối mà không cần agent.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
